# Base de datos

Mantenido por: `database-agent`. Aprobado por: Orchestrator, en coordinación
con `domain-architect` y `auth-security-agent`.
Estado: **Phase 0 completa** — estrategia de concurrencia, slots,
recurrencia y timezones resueltas (ver ADR-0002 a ADR-0015 en
`decisions.md`). El schema concreto (migraciones SQL completas) se
implementa en Phase 1.

## Motor y stack de acceso a datos (ADR-0002 — cerrado)

PostgreSQL vía Supabase. **Sin ORM adicional** — supabase-js + funciones
RPC de PostgreSQL alcanza para todo lo que este dominio necesita.
Confirmado explícitamente por `database-agent` en el cierre de Phase 0:
la operación crítica (ADR-0004) ya está forzada a vivir como función
PL/pgSQL sí o sí (PostgREST no permite `BEGIN`/`FOR UPDATE` en una
llamada e `INSERT` en la siguiente), y un ORM con conexión directa a
Postgres (bypaseando PostgREST) rompería o forzaría reimplementar a mano
el modelo de RLS de ADR-0006 (que depende de que cada query pase por
PostgREST con el JWT del usuario). `supabase gen types typescript`
genera tipos TS desde el schema SQL real — sin una segunda fuente de
verdad de schema que mantener sincronizada.

## Principios obligatorios

- Aislamiento estricto por `organizationId` en toda tabla de negocio.
- Foreign keys explícitas entre entidades relacionadas (no "soft
  references" por convención de nombre).
- `UNIQUE` constraint que impida más de una `Booking` activa del mismo
  `Customer` para el mismo `SlotOccurrence` (a nivel de base de datos, no
  solo aplicación).
- Índices sobre las columnas usadas para filtrar disponibilidad
  (`organizationId`, `serviceId`/`resourceId`, rango de fecha, `status`).
- RLS (Row Level Security) activo en todas las tablas con datos de
  organización, coordinado con `auth-security-agent`.
- Separación explícita entre lo que es público (servicios, disponibilidad)
  y lo que requiere policy de acceso (clientes, bookings individuales,
  pagos).

## El problema de concurrencia (P0)

Escenario que **siempre** debe resolverse correctamente:

```
capacidad 10, reservas activas 9
dos usuarios reservan el último cupo al mismo tiempo
→ solo uno debe obtener la reserva
```

**No se acepta** una solución basada únicamente en:

```
if available > 0:
    insert booking
```

sin protección transaccional real, porque hay una condición de carrera
entre el `SELECT` y el `INSERT`.

### Estrategia elegida (ADR-0004 — Aceptada)

**RPC atómica en PostgreSQL** (`book_slot(slotOccurrenceId, customerId, ...)`
vía `supabase.rpc(...)`), que en una sola transacción server-side hace
`SELECT ... FOR UPDATE` sobre la fila de `SlotOccurrence` + verificación
de capacidad + `INSERT` del booking, devolviendo un resultado explícito
(`OK` / `SLOT_FULL` / `DUPLICATE`) en vez de una excepción genérica.
Respaldada por `UNIQUE (customerId, slotOccurrenceId) WHERE
status = 'CONFIRMED'` como defensa secundaria.

**Por qué no las otras opciones:** row locking "suelto" desde el backend
no es viable porque supabase-js habla con PostgREST y cada llamada
`.from(...)` es su propia transacción HTTP — el lock de un `SELECT FOR
UPDATE` en una llamada se libera antes de que llegue el `INSERT` de la
siguiente. Constraint+retry puro requeriría un contador denormalizado +
trigger para expresar "capacidad mutable", una versión peor documentada
del mismo problema. Advisory locks requieren una clave sintética (hash de
UUID a bigint) y son frágiles bajo el pooler transaccional de Supabase si
los statements no caen en la misma conexión física — sin ventaja sobre
lockear una fila real ya existente. Detalle completo en ADR-0004
(`decisions.md`).

`booking-engine-agent` consume esta RPC y maneja sus tres resultados
posibles; no reimplementa lógica de locking en la capa de aplicación.

## Estrategia de SlotOccurrence (ADR-0003, ADR-0009 — Aceptadas)

Materializada, horizonte rodante de **90 días**. Mecanismo de
mantenimiento de la ventana: **pg_cron diario** (función PL/pgSQL
idempotente, `INSERT...SELECT...ON CONFLICT DO NOTHING`, batcheada por
`ScheduleRule`, envuelta en `pg_try_advisory_lock` contra ejecuciones
solapadas) **+ fallback lazy** en el camino de lectura de disponibilidad
si se detecta un hueco de cobertura. Detalle completo en ADR-0009.

## Generación de Bookings recurrentes (ADR-0011)

RPC atómica hermana de `book_slot()` (ej.
`generate_recurring_booking(recurringBookingId, slotOccurrenceId)`), no
un `INSERT` de aplicación — porque un mismo customer puede estar en dos
`RecurringBooking` que coinciden en una misma ocurrencia, y el `UNIQUE`
de ADR-0004 rechazaría la segunda si se intentara como insert plano sin
manejo. Bajo el mismo `FOR UPDATE` que `book_slot`, hace
`INSERT ... ON CONFLICT (recurring_booking_id, slot_occurrence_id) DO NOTHING`
para idempotencia del job, e inserta `NOT_GENERATED` en vez de fallar si
no hay cupo. Dos `UNIQUE` conviven sin competir — ver el SQL completo en
ADR-0011.

## Timezones (ADR-0014)

`SlotOccurrence.startAt`/`endAt`: `timestamptz`. `Organization.timezone`:
string IANA. Conversión local→UTC en SQL vía `AT TIME ZONE`, recalculada
por fecha en cada generación (nunca offset cacheado) — así DST se
resuelve automáticamente. `SlotOccurrence.generatedTimezone` (text) como
columna de trazabilidad adicional. Detalle completo en ADR-0014.

## Auditoría — `updatedAt` vía trigger

`updatedAt` se mantiene con trigger `BEFORE UPDATE` genérico
(`set_updated_at()`), no por convención de aplicación — hay múltiples
caminos de escritura (server actions, RPCs, SQL editor) y el trigger es
el único punto que garantiza el invariante sin depender de disciplina.
`createdAt` usa `DEFAULT now()` como columna. `createdBy`/`cancelledAt`/
`cancelledBy`/`cancellationReason` se setean explícitamente por la RPC
que ejecuta la acción (nunca inferidos de `auth.uid()` en un trigger —
un STAFF puede cancelar en nombre de otro, o correr una cascada bajo
contexto privilegiado). Ver matriz de qué entidad necesita qué campos en
`domain.md`.

## Pendiente de definir (Phase 1)

- Tablas concretas y sus columnas (deriva de `domain.md`, ya cerrado en lo
  sustancial).
- Diseño exacto de políticas RLS por tabla (base ya definida en ADR-0006 —
  dos capas tenant + ownership; falta traducirlo a SQL por tabla).
- Estrategia de migraciones (herramienta, convención de nombres) — se
  gestionan como archivos SQL versionados de Supabase CLI, sin ORM.
- `EXCLUDE USING gist` sobre `daterange(periodStart, periodEnd)` sobre
  `payments` por `serviceEntitlementId` filtrado a `status='PAID'`, para
  prevenir solapamientos (no bloqueante para Phase 1, ver ADR-0013).

## `ServicePlan` y la cuota semanal (ADR-0024 — Fase H, migración `20260922120000_phase17_service_plans.sql`)

`service_plans` es la lista de precios que la organización ofrece a sus
clientes (no `plans`, que son los planes del SaaS): `plan_kind`
(`DROP_IN` | `WEEKLY_QUOTA` | `UNLIMITED`), `weekly_quota`, `price` y el
par `billing_type`/`billing_cycle`, que **suben del `Service` al plan**
(`services.billing_type`/`billing_cycle`/`price` quedan deprecadas;
`payment_required` no). `payments` gana `service_plan_id` (ancla de
registro, `NOT NULL`) y `slot_occurrence_id` (no nulo exactamente para
los pagos de clase suelta); `payments.service_id` se conserva como
columna derivada y verificada por trigger.

**La cuota no es un contador.** "N veces por semana" es "N
`RecurringBooking` en vigencia", porque cada `ScheduleRule` es semanal
por construcción. No se persiste ni se decrementa nada, y
`availableCapacity = maxCapacity - activeBookings` no se toca. Se mide
siempre contra la **fecha local del slot** (`ACTIVE` + `start_date <= D`
+ `end_date is null or >= D`), nunca contra `now()`, con desempate total
`created_at asc, id asc` para que preview y confirm no puedan discrepar.

**Dónde vive cada invariante** (el criterio, no la lista):

- `CHECK` — todo lo que se decide mirando una sola fila: `weekly_quota` ↔
  `plan_kind`, la combinación `plan_kind` ↔ `billing_type`/`billing_cycle`,
  `price >= 0`, coherencia de cancelación, formato de
  `organizations.currency`.
- **Índice único parcial / `EXCLUDE`** — todo lo que es ambigüedad o doble
  cobro: un solo `DROP_IN` activo y un solo `UNLIMITED` activo por
  servicio, nombre único entre activos, `EXCLUDE` de períodos solapados
  (ahora sólo para pagos por período) y `unique (customer_id,
  slot_occurrence_id) where status='PAID' and slot_occurrence_id is not
  null` para el pago suelto, que sin ese índice quedaba sin ninguna
  protección.
- **Trigger** — todo lo que cruza tablas y por lo tanto no puede ser
  `CHECK`, en una tabla escribible por PostgREST: plan ↔ servicio de la
  misma organización, `payments.service_id = service_plans.service_id`,
  `(plan_kind='DROP_IN') = (slot_occurrence_id is not null)`, la
  ocurrencia del pago suelto del mismo servicio y organización,
  inmutabilidad de `plan_kind`/`weekly_quota` con pagos no-`VOID`, y
  prohibir planes de cuota sobre un servicio sin `payment_required`.
- **RPC** — la cuota. Es un agregado que cruza `recurring_bookings →
  schedule_rules → services` contra un `weekly_quota` al que se llega por
  `payments → service_plans`: no hay constraint que lo exprese. Alcanza
  con la RPC porque la RLS de `recurring_bookings` es SELECT-only y toda
  escritura pasa por función; si alguna vez se abre un `INSERT` por
  PostgREST hace falta además un `BEFORE INSERT`.

Dos puertas, con pesos distintos: **blanda** al crear la serie (rechaza
la N+1 sólo si hay un pago en vigencia con plan de cuota, para no romper
el flujo de mostrador de ADR-0018/0019) y **dura por fecha** al generar
(`NOT_GENERATED` + `not_generated_reason = 'OVER_PLAN_QUOTA'`, que
ADR-0019 reconcilia solo cuando el cliente sube de plan).

Sigue habiendo **una sola** función de decisión: `payment_covers_slot()`
es literalmente `evaluate_payment_coverage(...) = 'OK'`, y
`evaluate_customer_booking()` recibe el contexto de serie en sus tres
formas (sin serie, serie existente, serie prospectiva). El lock de
`book_slot()` (ADR-0004) no se tocó.

RLS de `service_plans`: `SELECT` público (es una lista de precios, la
página pública la muestra), escritura sólo `is_organization_member`, sin
política de `DELETE` (un plan se desactiva; además `payments.service_plan_id`
es `ON DELETE RESTRICT`).

**Fase 18 (`20260922140000_phase18_drop_service_billing_check.sql`)** saca
el CHECK `services_payment_requires_priced_type` de Phase 14
(`not payment_required or billing_type <> 'FREE'`). Era correcto cuando
`services.billing_type` era el precio; con ADR-0024 el precio vive en el
plan y `billing_type` quedó en `'FREE'` por default, así que el CHECK
pasó a impedir algo legítimo: encender `payment_required` desde un
formulario que ya no pregunta modalidad de cobro. El invariante no se
pierde — "exige pago y no tiene ningún plan activo" es
`SERVICE_HAS_NO_PLAN`, evaluado al reservar, que además es más fuerte
(el CHECK aceptaba `MONTHLY` sin nada a la venta, el mismo callejón sin
salida que decía evitar). Los otros dos CHECK sobre columnas deprecadas
(`services_billing_cycle_matches_type`, `services_price_non_negative`) se
conservan: ninguno puede bloquear a un escritor que dejó de setear esas
columnas, porque los defaults los satisfacen, y se van con la migración
que finalmente dropee las columnas.

Los dos anclajes de `payments` son `ON DELETE RESTRICT`, incluido
`slot_occurrence_id` (corrige el DDL del proposal, que decía `CASCADE`):
un pago es plata y no se borra nunca, así que borrar una ocurrencia tiene
que fallar ruidoso en vez de llevarse el registro económico en silencio —
las ocurrencias se cancelan, no se borran.

"Este cliente ya tiene reserva fija en esta regla" tiene **una sola**
definición (`customer_standing_series_on_rule()`, mismo criterio de
vigencia que la cuota) y la usan los cuatro lugares que la necesitan: los
dos `create` (el de autoservicio la tenía faltando, y el argumento "k
series = k reservas por semana" se apoya en ella) y los dos `preview`, que
al detectar la serie existente reportan **su estado real** en vez de
evaluarla como prospectiva — si no, el mostrador leería `OVER_PLAN_QUOTA`
por contar dos veces la misma serie, mientras el confirm rechazaría por
`ALREADY_HAS_STANDING_RESERVATION`. Preview y confirm no pueden discrepar
(ADR-0018).

## Fase 19 — permisos de ejecución y autorización (`20260922160000_phase19_security_fixes.sql`)

Migración de seguridad sobre código ya existente. **No implementa ADR-0025
ni ADR-0026**: sólo las resoluciones 1 y 2 de ADR-0025, porque ambas son
precondición (sin ellas la política de aviso es decorativa y el motivo de
cancelación miente).

### El agujero de fondo: `EXECUTE` nunca se revocó de `PUBLIC`

Postgres otorga `EXECUTE` a `PUBLIC` por defecto en toda función nueva, y
Supabase además lo otorga explícitamente a `anon`/`authenticated`/
`service_role` por default privileges. Los `grant execute ... to
authenticated` repartidos por las Fases 1 a 18 **sumaban** un permiso
encima del default en vez de reemplazarlo: eran documentación, no una
restricción. Resultado: **las 57 funciones del schema `public` eran
invocables por PostgREST sin sesión**. La mayoría falla cerrado por su
propio chequeo de autorización; varias no tenían ninguno.

Regla que queda vigente desde ahora, y que toda función nueva debe seguir:

- **Helper interno** (sólo lo llaman funciones `security definer`):
  `revoke execute ... from public, anon, authenticated, service_role`.
  Adentro de un `security definer` los privilegios se chequean contra el
  dueño, así que revocar no cuesta nada y lo saca de la API.
- **RPC de negocio**: `revoke ... from public, anon` +
  `grant ... to authenticated`.
- **RPC pública** (`get_public_availability`, `public_slot_detail`):
  `grant ... to anon, authenticated`.
- **Helper de policy** (`is_organization_member`, `is_organization_owner`,
  `is_platform_admin`, `storage_object_organization`): conserva `PUBLIC`
  a propósito. Las expresiones de RLS se evalúan con el rol que consulta,
  y `anon` evalúa policies en cada lectura del calendario público. Sólo
  contestan sobre `auth.uid()`, así que no revelan nada de terceros.

## Fase 21 — clientes gestionados (ADR-0026, `20260922190000_phase21_managed_customers.sql`)

`customers.profile_id` pasa a nullable, `on delete set null` (era
`cascade` -- Hallazgo C). Columnas nuevas: `display_name`, `phone`
(E.164, `check` en la base), `claimed_at`, `merged_into_customer_id`.
Tabla nueva `customer_activations` (token de 256 bits hasheado con
sha256, un uso, 72h). Detalle completo de RLS, el trigger de guarda sobre
`profile_id` (Hallazgo E) y las RPC (`create_managed_customer`,
`issue_customer_activation`, `revoke_customer_activation`,
`claim_customer_activation`, `unlink_customer_profile`,
`merge_customers`) en `docs/security.md`, sección "Cliente gestionado +
activación por WhatsApp".

`organization_customers()`, `occurrence_bookings()`,
`organization_payment_summary()` y `schedule_rule_standing_reservations()`
pasan de `join public.profiles` a `left join`, con
`coalesce(nullif(trim(c.display_name), ''), p.full_name, 'Sin nombre')`
(Hallazgo B) -- si no, un cliente gestionado con reservas y pagos
desaparece de todo el panel. `customer_payment_detail()` no necesitaba el
mismo cambio: no joinea `profiles` en absoluto.

Pendiente antes de la demo: no se pudo correr `supabase db reset` ni la
suite de tests desde este agente (sin shell en esta sesión) -- ver el
comentario de verificación manual al final de la migración 21.

Después de la migración, `anon` sólo puede ejecutar esas seis funciones.

### `retry_not_generated_booking()` — lo más urgente

`security definer`, escribe en `bookings`, **cero autorización**, y sin un
solo `grant`/`revoke` en todo el repo. Cualquiera, sin sesión, podía pasar
una reserva `NOT_GENERATED` de cualquier organización a `CONFIRMED`.
Re-evalúa cobertura y cupo, así que no permite overbooking, pero sí sentar
a alguien que la organización dejó pendiente a propósito, y confirma por
la fila devuelta si un `booking_id` existe y en qué estado está.

Ahora: mismo criterio que sus hermanas (dueño de la reserva o miembro de
la organización), escrito en tres ramas, más una rama `auth.uid() is null`
para el llamador interno — `reconcile_pending_recurring_bookings()` es
`security definer` y puede correr sin JWT (job diario, cliente
service-role). Tras el `revoke` los únicos roles que llegan directo son
`authenticated` (que siempre trae `sub`) y el dueño, así que esa rama no
es una puerta.

### Lógica de tres valores en `cancel_booking()`

```sql
-- antes
if v_customer.profile_id <> auth.uid()
   and not public.is_organization_member(...) then
  raise exception 'NOT_AUTHORIZED';
end if;
```

Con `profile_id IS NULL` eso es `NULL and ...` = `NULL`, el `if` no entra
y la función sigue derecho al `UPDATE`: **la autorización se saltea
entera**. Hoy no es explotable sólo porque la columna es `NOT NULL`; la
Fase J la vuelve nullable y en ese momento cualquier autenticado podría
cancelar la reserva de cualquier cliente gestionado de cualquier
organización.

La forma correcta ya estaba en el repo, en `cancel_recurring_booking()`
(Fase 6): `if <es el dueño> / elsif <es miembro> / else raise`. Un `NULL`
hace falsas las dos ramas y cae en el `else`. Misma expresión,
comportamiento opuesto. Es el patrón obligatorio para toda comparación de
autorización contra una columna que pueda ser `NULL`.

Verificado con `profile_id` puesto en `NULL` (bajando el `NOT NULL` en una
prueba descartable): la forma vieja dejó que un tercero cancelara la
reserva **y** estampara `SLOT_CANCELLED`; la nueva devuelve
`NOT_AUTHORIZED` y no toca la fila.

### El motivo se deriva del actor (ADR-0025, resolución 1)

`cancel_booking(uuid, booking_cancellation_reason)` se **dropea**. Queda
`cancel_booking(uuid)`, que siempre estampa `CUSTOMER_REQUEST`. Es un
cambio de contrato de API. Los caminos de la organización
(`cancel_slot_occurrence`, `discontinue_schedule_rule`, cascada de serie)
siguen estampando el suyo desde adentro, nunca por parámetro. Staff
cancelando una reserva puntual desde el mostrador sigue siendo
`CUSTOMER_REQUEST` — es lo que es, el cliente pidió — y `cancelled_by`
(FK a `Profile`, no el enum que `domain.md` describía) registra qué perfil
lo hizo.

### `SERIES_CANCELLED` (ADR-0025, resolución 2)

Nuevo valor de `booking_cancellation_reason`. La cascada de
`cancel_recurring_booking()` lo estampa en **las dos ramas**: la de staff
decía `RULE_DISCONTINUED` (y la regla no se discontinuó) y la del cliente
decía `CUSTOMER_REQUEST` (y el cliente no pidió *esa fecha*). El estado de
la serie sigue registrando quién la dio de baja en
`recurring_bookings.cancellation_reason` (`CUSTOMER_REQUEST` |
`ORGANIZATION_REMOVED`). `discontinue_schedule_rule()` conserva
`RULE_DISCONTINUED`, que ahí sí es cierto.

Nota operativa: `ALTER TYPE ... ADD VALUE` no puede ser **usado** por DML
ni por un `CHECK` en la misma transacción. Mencionarlo en un cuerpo
PL/pgSQL sí se puede: la expresión recién se parsea en la primera
ejecución. Esta migración no escribe el valor nuevo en ningún lado.

### `generate_slot_occurrences_for_rule()` — chequeo de tenant y tope

También `security definer` sin autorización ni grant: escribía
`slot_occurrences` (y vía `generate_recurring_booking()`, `bookings`) para
la regla que le pasaran, de cualquier organización, con `p_horizon_days`
sin tope. **No se puede simplemente revocar de `authenticated`**: los
triggers `AFTER INSERT`/`AFTER UPDATE` de `schedule_rules` que la llaman
son `security invoker`, así que el staff que crea una regla por PostgREST
necesita el privilegio. El chequeo adentro es la frontera real: un
autenticado sólo puede generar para una regla de una organización a la
que pertenece; `auth.uid() is null` es el cron y el cliente service-role.
El horizonte se acota a 365 días.

**Bug preexistente que esto destapó y no se toca acá:**
`slot_occurrences` no tiene policy de `DELETE`, así que el `delete from
public.slot_occurrences` de `trigger_regenerate_on_schedule_rule_update()`
(que es `security invoker`) hoy **no borra nada** bajo RLS. Pasar ese
trigger a `security definer` — que sería el arreglo limpio del permiso —
lo haría borrar de verdad, incluidas ocurrencias con reservas. Es un
problema aparte.

### Pendiente explícito para la Fase J: `customers.profile_id`

Es `on delete cascade`, y `bookings.customer_id` / `payments.customer_id`
también. Borrar un `auth.users` se lleva el `Customer`, **sus reservas y
sus pagos** — contra `domain.md` ("una `Booking` cancelada nunca se
borra", "un `Payment` se anula con `VOID`, no se elimina"). El arreglo es
`on delete set null`, que Postgres **rechaza mientras la columna sea
`NOT NULL`**. Por eso tiene que viajar en la misma migración que vuelve
`profile_id` nullable (Fase J / ADR-0026), no antes.

## Fase 20 — `makeup_credits` (ADR-0025, migración `20260922180000_phase20_makeup_credits.sql`)

Aditiva, opt-in (`organizations.makeup_credits_enabled default false`). Shape
completo, las nueve resoluciones del Orchestrator y los 13 casos límite
contestados están en `docs/decisions.md` ADR-0025 y en
`docs/proposals/adr-0025-makeup-credits.md`; acá sólo lo específico de
implementación que no estaba ya escrito ahí.

### El lock del crédito es una UPDATE condicional, no un `FOR UPDATE` extra

`book_slot()`/`admin_book_for_customer()` resuelven el crédito usable con una
lectura sin lock (`resolve_usable_makeup_credit()`, dentro de
`evaluate_payment_coverage_with_credit()`) y **recién lo bloquean al
consumirlo**:

```sql
update public.makeup_credits set status = 'CONSUMED', consumed_booking_id = ..., consumed_at = now()
  where id = v_makeup_credit_id and status = 'AVAILABLE';
if not found then raise exception 'MAKEUP_CREDIT_RACE_LOST'; end if;
```

Dos reservas simultáneas del mismo cliente sobre dos `SlotOccurrence`
distintas no comparten ningún `FOR UPDATE` (son dos filas distintas, ADR-0025
§0.1) — lo único que las serializa es esta fila de `makeup_credits`. Postgres
resuelve la carrera solo: la segunda `UPDATE` en tocar la fila espera el lock,
lo obtiene después de que la primera comprometa, reevalúa `WHERE status =
'AVAILABLE'` contra el dato ya committeado, ve 0 filas y **aborta toda la
transacción** — incluida la `Booking` que esa misma llamada acababa de
insertar. Nunca un `status` explícito acá: es el único punto de este diseño
donde una excepción reemplaza a un resultado explícito (documentado también
en el comentario de la función). Test:
`test/phase20.makeup-credits.test.ts` → "two simultaneous bookings ... one
credit ... exactly one consumes it", sobre dos ocurrencias de dos reglas
distintas — sobre la misma ocurrencia el motivo de rechazo sería
`ALREADY_BOOKED`/`SLOT_FULL` y el crédito no se llegaría a mirar.

### `CREATE OR REPLACE` no puede agregarle un parámetro a una función existente

Se intentó primero agregar `p_use_makeup_credit boolean default true` como
parámetro final de `can_customer_book`, `evaluate_customer_booking`,
`book_slot`, `admin_book_for_customer` y `payment_covers_slot` vía `create or
replace function` directo, asumiendo que un parámetro nuevo con default es
compatible. **No lo es**: Postgres identifica una función por nombre + tipos
de argumentos, así que agregar uno crea un **overload nuevo y distinto**, y
dos overloads con "la vieja lista de argumentos es un prefijo de la nueva
+ default" quedan **ambiguos** para cualquier llamada que hoy pasa
exactamente la lista vieja de argumentos (`function ... is not unique`,
verificado en vivo — rompió los 5 archivos de test hasta corregirse). Cada
una de esas cinco funciones se **dropea explícitamente antes** de
recrearla con el parámetro nuevo, mismo patrón que usó Fase 17 para
`payment_covers_slot`. Efecto secundario a no olvidar: un `DROP` le
resetea los privilegios a `PUBLIC` (el default de Postgres) — cada una de
esas cinco funciones tiene su `revoke`/`grant` reescrito explícitamente
después del `create`, no asumido heredado del `create or replace` anterior.

### El veredicto compuesto es una función de tabla de una fila, nunca un valor de enum

`evaluate_payment_coverage_with_credit(...)` devuelve
`table (reason can_book_reason, makeup_credit_id uuid)`. `payment_covers_slot()`
sigue siendo booleano (`reason = 'OK'`); `evaluate_customer_booking()` sigue
devolviendo sólo `can_book_reason` (descarta el `makeup_credit_id`, que sólo
importa en el momento de consumir, bajo lock). El guard de "un crédito nunca
cubre una fecha de serie" vive **adentro** de esta única función
(`p_recurring_booking_id is null and not p_prospective_series`), no
repartido entre los tres o cuatro llamadores que podrían olvidarlo.

### Emisión: la `Booking` que recibe `issue_makeup_credit()` ya está `CANCELLED`

No `CONFIRMED`. Los tres llamadores (`cancel_booking`,
`cancel_slot_occurrence`, `discontinue_schedule_rule`) hacen primero la
`UPDATE ... WHERE status = 'CONFIRMED'` y le pasan a `issue_makeup_credit()`
la fila que esa misma `UPDATE`/`RETURNING` devolvió — ya con
`status = 'CANCELLED'`. Que "estaba `CONFIRMED`" (condición 2 de la emisión)
es cierto queda garantizado por construcción (el `WHERE` de esa `UPDATE`), no
por un chequeo de estado dentro de `issue_makeup_credit()`, que sólo lo
valida defensivamente. Esto importa para `cancel_slot_occurrence()` y
`discontinue_schedule_rule()`, que cancelan varias `Booking` en un solo
`UPDATE ... RETURNING` recorrido con `for ... in update ... returning ...
loop` (sintaxis de PL/pgSQL válida para `INSERT`/`UPDATE`/`DELETE` con
`RETURNING`, no sólo para `SELECT`) — cada fila que itera el loop ya viene
cancelada.

### Alcance no implementado en esta migración

`quote_booking()`/`book_slot_paying()` (ADR-0025 §2.8, el turno suelto
cobrado en el mismo hecho que la reserva) quedan pendientes — es la pieza
separable que la propia ADR marcó como tal (depende de decisiones de
ADR-0027 que no son de esta fase). Hoy "sin cupo, pagás" se sigue resolviendo
como antes de ADR-0025: carga manual del ADMIN.

## Fase 24 — `platform_contact_requests` (ADR-0030 resolución 2, migración `20260922220000_phase24_platform_contact_requests.sql`)

Tabla nueva, aislada, aditiva: captura los envíos del formulario público
`/contacto` de la landing (el CTA principal deja de prometer un alta
self-service que no existe — ADR-0017 — y pasa a generar un lead). No toca
`organizations`, `plans`, `services` ni ningún flujo de booking existente.

Columnas: `id`, `name`, `email` (`check` de formato + largo ≤320),
`phone` (opcional, mismo `check` E.164 que `customers.phone` de la Fase
21), `business_type` (texto libre, opcional, ≤120 — nunca un enum cerrado:
el producto es genérico), `message` (1-4000 caracteres), `created_at`,
`origin_ip` (derivado server-side de headers de request, ver abajo — nunca
un parámetro que mande el caller), `handled_at`/`handled_by` (par opcional
para que el platform admin marque un lead como atendido; `handled_by`
exige `handled_at` vía `check`).

**Mismo patrón de acceso que `customer_activations` (Fase 21):** RLS
habilitada sin ninguna policy — deniega todo acceso directo por PostgREST
(`select` devuelve `[]`, `insert`/`update` directos fallan) sin importar
privilegios ambiente (Supabase sí otorga grants de tabla por default a
`anon`/`authenticated`, que esta migración no revoca explícitamente — la
defensa real y suficiente es únicamente RLS sin policies, no la ausencia
de grants). Las únicas puertas son tres funciones `security definer`:

- `submit_platform_contact_request(p_name, p_email, p_message, p_phone
  default null, p_business_type default null)` — **pública, `grant ...
  to anon, authenticated`**, sin chequeo de autorización (es un
  formulario anónimo de landing). Valida formato/largo de cada campo
  (mismo criterio de normalización de teléfono que
  `create_managed_customer()`) y aplica un rate limit de **dos niveles por
  origen**, corregido en revisión de seguridad tras el primer borrador
  (que era un `count(*)` puramente global de 20/hora — un solo script
  anónimo agota esa cuota para cualquier visitante del planeta, y
  `/contacto` es la única puerta comercial pública tras ADR-0030
  resolución 2):
  - **Real: 5/hora por `origin_ip`** — derivado server-side de
    `current_setting('request.headers', true)::json`, header `x-real-ip`
    con fallback al último salto de `x-forwarded-for` (verificado contra
    `supabase start` local: el gateway sobreescribe `x-real-ip` con su
    propia vista del peer TCP y **apéndica** su vista del peer como
    último elemento de `x-forwarded-for` — un cliente puede falsear
    entradas anteriores de esa cadena pero no la última). `'unknown'` si
    ninguno de los dos headers está presente.
  - **Backstop: 200/hora global** — no es el límite real, protege contra
    un flood distribuido entre muchos orígenes.
  - Ambas ramas lanzan la **misma** excepción `RATE_LIMITED` (sin
    distinguir cuál se disparó — evita que un caller anónimo infiera
    volumen de leads probando cuál límite pisa primero).
  - Todo el bloque de chequeo-y-escritura corre bajo
    `pg_advisory_xact_lock(hashtext('platform_contact_requests_rate_limit'))`
    (mismo patrón que `book_slot()`, ADR-0004, adaptado a una tabla sin
    fila padre que lockear) — sin esto, una ráfaga concurrente puede leer
    el mismo conteo antes de que cualquier insert se vea y pasar de largo
    ambos umbrales (TOCTOU), como efectivamente pasaba en el primer
    borrador.
  - **Limitación conocida del deploy actual** (ADR-0021): `/contacto` se
    manda vía server action de Next.js
    (`frontend/app/actions/contact.ts`), que llama a esta RPC
    server-side desde el droplet — no desde el browser del visitante. El
    tráfico legítimo relayado por nuestro propio frontend comparte un
    único `origin_ip` aparente (el del droplet) ante Supabase, mientras
    que un script que llama a esta RPC directamente (es pública, alcanzable
    con la anon key embebida en el bundle del frontend) sí muestra su IP
    real. Esto cierra el ataque concreto descripto arriba sin tocar el
    frontend; aislar cada visitante del browser individualmente
    requeriría que el frontend reenvíe la IP real como parámetro explícito
    — fuera de alcance de esta ronda, marcado como pase futuro si esta
    tabla ve volumen orgánico real.
- `platform_contact_requests()` — lectura, gateada por
  `is_platform_admin()`, mismo patrón exacto que `platform_organizations()`/
  `platform_invites()` (Fase 10): sin chequeo explícito, el `where
  public.is_platform_admin()` filtra a cero filas para cualquier otro
  caller en vez de fallar.
- `mark_contact_request_handled(p_id)` — escritura, `NOT_AUTHORIZED` si
  no es platform admin, `CONTACT_REQUEST_NOT_FOUND` si el id no existe.
  Idempotente: un segundo llamado conserva el `handled_at`/`handled_by`
  original (`coalesce`) en vez de pisarlo.

Server actions en `frontend/app/actions/`: `submitContactRequest()` vive
en `app/actions/contact.ts` (archivo nuevo, separado de `platform.ts`
porque es la única acción de esta superficie sin `is_platform_admin()`
detrás — todo lo demás en `platform.ts` sigue gateado en SQL, como dice
su comentario de cabecera). `getPlatformContactRequests()` y
`markContactRequestHandled()` se agregaron a `platform.ts`, mismo shape
que `getPlatformOrganizations()`/`getPlatformInvites()`.

Test de integración: `test/phase24.platform-contact-requests.test.ts`.
Nota de diseño del test: como el rate limit ahora es por `origin_ip` y
todos los requests del archivo salen del mismo test runner (comparten el
mismo `origin_ip` ante la base), los dos tests de rate limit (umbral
simple + concurrencia) limpian la tabla al empezar cada uno — así quedan
deterministas y no dependen de cuántos `anon.rpc()` exitosos hicieron los
tests anteriores del archivo, ni interfieren entre sí. El test de
concurrencia (`Promise.all` con 8 envíos simultáneos contra un umbral de
5) verifica el fix de TOCTOU: sin el `pg_advisory_xact_lock`, más de 5
podrían colarse. El archivo entero también limpia la tabla en `afterAll`,
por la misma razón que antes (sin eso, sus propias filas seguirían dentro
de la ventana de 1h en la corrida siguiente). Verificado corriendo la
suite completa de integración (164 tests, 18 archivos) contra
`supabase db reset` local.

---

## Fase 25 — correcciones sobre el feedback del primer cliente en producción

Migración: `20260923120000_phase25_payment_duplicates_and_billing_horizon.sql`.
Ninguna decisión nueva: cada bloque repara una implementación que se había
apartado del ADR que la define. Test: `test/phase25.production-feedback.test.ts`.

### 1. `check_payment_same_org()` — un plan multi-servicio era invendible

ADR-0029 dejó `payments.service_id` como columna **derivada y nullable**:
`check_payment_plan_consistency()` la completa sólo cuando el plan cubre
exactamente un servicio, y la deja en `NULL` cuando cubre varios (la
cobertura real vive en `payment_service_coverage`).
`check_payment_same_org()` es de la Fase 7, de cuando `service_id` era
obligatorio, y hacía `if v_service_org is null ... raise`. Los dos triggers
son `BEFORE INSERT` y corren en orden alfabético
(`payments_plan_consistency` < `payments_same_org`), así que el segundo veía
el `NULL` que el primero acababa de poner **a propósito** y rechazaba el
insert. Resultado verificado contra la base local: **todo** pago de un plan
`applies_to_all_services` (o con dos o más servicios) fallaba con
`Payment organization_id must match its Customer and Service`, que en la
pantalla se lee como el genérico "No se pudo registrar el pago".

Ahora, con `service_id` nulo, se valida el conjunto cubierto por el plan
contra la organización del pago (`SERVICE_PLAN_SCOPE_EMPTY` si el plan no
cubre nada). La garantía de tenant es la misma; lo que cambia es sobre qué
se expresa.

### 2. `check_payment_no_duplicate()` — doble cobro sin depender de `PAID`

El `EXCLUDE` de ADR-0024 (hoy `payment_service_coverage_no_overlap`) y el
índice único por ocurrencia filtran por `status = 'PAID'`. Dos pagos
`PENDING` idénticos del mismo cliente, servicio y período entraban sin
ninguna protección, y dos pagos de turno suelto sobre la misma ocurrencia
también mientras no fueran `PAID` — el hueco que ADR-0027 ya tenía anotado.

Trigger nuevo `payments_no_duplicate` (`AFTER INSERT`), que rechaza con
`PAYMENT_DUPLICATE_PERIOD` / `PAYMENT_DUPLICATE_OCCURRENCE`. Dos decisiones
de implementación, con su motivo:

- **Trigger y no ampliar el `EXCLUDE` a `status <> 'VOID'`.** Ampliar el
  predicado obliga a validar todas las filas históricas en el deploy, y una
  organización que ya tenga dos `PENDING` solapados (exactamente el bug que
  se corrige) haría fallar la migración o forzaría a anular datos reales sin
  que nadie los mire. "Nada se borra" incluye "nada se anula solo". Los dos
  mecanismos de ADR-0024 quedan **intactos**: siguen siendo la defensa a
  prueba de carreras del caso que mueve plata.
- **Sólo `INSERT`.** En `UPDATE`, marcar pagado un `PENDING` que solapa con
  otro ya lo rechaza el `EXCLUDE`; agregar un rechazo nuevo sobre ese camino
  —cuya acción de frontend hoy ignora el error— convertiría un error visible
  en un botón que no hace nada.

Un pago `VOID` no cuenta: corregir un error de carga sigue siendo
anular y volver a cargar.

### 3. `customer_billing_horizon()` + `schedule_rule_standing_reservations()`

`upcoming_unpaid` contaba **todas** las fechas futuras `NOT_GENERATED /
PAYMENT_REQUIRED` de la ventana rodante (90 días, ADR-0009). Un pago mensual
nunca cubre noventa días, así que el contador era `> 0` para toda reserva
fija de todo cliente, siempre, y la etiqueta roja "Falta el pago" no tenía
forma de apagarse ni para alguien al día. Es el reporte 1 del cliente.

`customer_billing_horizon(customer, service, hoy)` devuelve hasta cuándo
alcanza lo que el cliente ya compró: el `period_end` de su cobertura `PAID`
vigente; si no tiene, el `period_end` que `billing_period_for()` da para el
plan de su último pago no anulado; y si nunca pagó nada de ese servicio, el
fin del mes calendario. `upcoming_unpaid` se corta ahí y la función devuelve
una columna nueva, **`upcoming_beyond_period`**, con las fechas de más
adelante: no son deuda, todavía no se facturan.

**El motor de decisión no tenía este problema y quedó verificado**: con un
pago que cubre 60 días, `evaluate_customer_booking()` responde `OK` para
cada fecha dentro del período, incluida una clase de las 21:00 del último
día del mes (00:00 UTC del día 1 siguiente, ADR-0014). El bug estaba en el
contador de la pantalla, no en la cobertura.

### 4. `customer_payment_detail()` — `LEFT JOIN`, no `INNER`

Joineaba `services` por `payments.service_id` con `INNER JOIN`, así que un
pago de plan multi-servicio (`service_id` nulo, §1) desaparecía de la
pantalla de pagos. Un pago invisible es indistinguible de uno que no se
registró. Ahora el nombre cae al del plan.

### 5. `can_customer_book_detail()` — el veredicto positivo, con su crédito

ADR-0025 §2.4.3 hace viajar el veredicto como par `(reason,
makeup_credit_id)` porque seis funciones comparan `= 'OK'`.
`can_customer_book()` devuelve sólo el enum, así que el portal mostraba
"Reservar" y **gastaba el crédito de recupero en silencio**. La RPC nueva
devuelve `(reason, makeup_credit_id, makeup_credit_expires_on)` y sólo
informa el crédito cuando la cobertura por sí sola no alcanzaba — nunca se
gasta un crédito si otra cobertura llegaba, así que un `OK` común no informa
ninguno. No reimplementa ninguna regla: llama a las mismas funciones que
`book_slot()`. `revoke ... from public, anon` + `grant to authenticated`
(ADR-0028).

### Revisión de `security-engineer` — cuatro fixes sobre la migración de arriba

`security-engineer` verificó **empíricamente** (transacciones `psql`
concurrentes reales) que la versión inicial de `check_payment_no_duplicate()`
(§2) no era atómica. Cuatro fixes, todos en la misma migración:

**Fix 1 — `check_payment_no_duplicate()` no era atómica.** Bajo Read
Committed, dos `INSERT` concurrentes del mismo pago `PENDING` pasaban los
dos: mismo TOCTOU que ADR-0004 cazó en `book_slot()`. El caso que mueve
plata (`PAID`) sigue protegido por el `EXCLUDE` declarativo de ADR-0024, sin
tocar.

Serializa con `pg_advisory_xact_lock(hashtext(new.customer_id::text))`. La
primera versión probada usaba `select ... from customers where id = ... for
update` (lock de fila real sobre el `Customer` padre, sin advisory lock
"porque hay fila padre real") — y **deadlockeaba** bajo el test de
concurrencia (`Promise.all` con 6 inserts): el trigger corre `AFTER INSERT`,
así que para cuando llega ahí el chequeo de FK de `payments.customer_id` ya
tomó un `FOR KEY SHARE` implícito sobre esa fila **en la misma transacción**;
pedir después un `FOR UPDATE` sobre la misma fila es un *upgrade* de lock, y
con varias transacciones concurrentes cada una ya sosteniendo su propio `FOR
KEY SHARE` y esperando que las demás lo suelten para subir a `FOR UPDATE`, la
espera es circular → `deadlock detected`, reproducido de verdad, no en
teoría. El advisory lock es una primitiva aparte que no interactúa con el
locking de fila/FK de Postgres — mismo patrón que
`generate_all_slot_occurrences()` (Fase 3) y
`submit_platform_contact_request()` (Fase 24), pero acotado por
`customer_id` en vez de global, para no serializar clientes sin relación
entre sí.

Test de concurrencia (mismo patrón que Fase 24): `Promise.all` con 6 inserts
simultáneos del mismo pago `PENDING`, verifica que entra exactamente 1 y el
resto rechaza con `PAYMENT_DUPLICATE_PERIOD`.

**Fix 2 — evasión por `VOID` → `PENDING`.** El trigger sólo corría en
`INSERT`, así que un pago se podía insertar como `VOID` (aceptado, correcto)
y después revivir a `PENDING` vía `set_payment_status()` o un `PATCH`
directo, dejando un duplicado sin que nada lo note. Dos triggers en vez de
uno con `INSERT OR UPDATE`: Postgres rechaza (`SQLSTATE 42P17`) un `WHEN` que
referencia `OLD` en un trigger cuyo conjunto de eventos incluye `INSERT`,
aunque el evento combinado también incluya `UPDATE` — confirmado corriendo la
migración. `payments_no_duplicate` sigue siendo `AFTER INSERT WHEN (new.status
<> 'VOID')`; `payments_no_duplicate_on_resurrect` es nuevo, `AFTER UPDATE
WHEN (old.status = 'VOID' and new.status <> 'VOID')` — no dispara en
transiciones normales (p. ej. `PENDING` → `PAID`) que no reviven nada.

**Fix 3 — `can_customer_book_detail()` re-derivaba la compuerta de
crédito.** Comparaba `evaluate_payment_coverage(...) <> 'OK'` a mano en vez
de leer el veredicto de `evaluate_payment_coverage_with_credit()`, el
resolutor canónico. No era explotable (la condición daba el mismo resultado),
pero era riesgo de divergencia silenciosa si algún día cambia la lista blanca
de motivos que habilitan el crédito. Ahora llama directo a
`evaluate_payment_coverage_with_credit(v_customer_id, v_service_id,
p_slot_occurrence_id, null, false, true)` y lee `makeup_credit_id` de su
resultado.

**Fix 4 — piso del `CHECK` de `release_deadline_hours`.** El action
`updateOrganizationSettings` valida 1–720, pero el `CHECK` de la tabla
(Fase 20) permitía 0–720, y `organizations` es editable directo por el OWNER
vía PostgREST (`organizations_update_owner`, ADR-0013) — el piso real era 0,
no 1: un OWNER podía poner `release_deadline_hours = 0` sin pasar por el
action, vaciando la política de ADR-0025 (el crédito se emitiría cancelando
en la puerta). `organizations_release_deadline_hours_range` ahora exige
`between 1 and 720`.

### Ronda final — el mismo hueco, un nivel más abajo: `services`

`security-engineer` verificó el estado real desplegado y encontró que el Fix 4
de arriba dejaba a mitad de camino el mismo problema en `services`: los cuatro
overrides de política de crédito de recupero (`makeup_credits_enabled_override`,
`release_deadline_hours_override`, `makeup_credit_expiry_override`,
`makeup_credit_expiry_days_override`, todos de Fase 20) heredan el `CHECK`
laxo y, peor, son escribibles por **cualquier miembro** de la organización, no
sólo el `OWNER`.

**Fix 5 — piso del `CHECK` de `services.release_deadline_hours_override`.**
Mismo problema que el Fix 4, un nivel más abajo:
`services_release_deadline_hours_override_range` (Fase 20) seguía en 0–720.
`resolve_makeup_credits_policy()` hace
`coalesce(s.release_deadline_hours_override, o.release_deadline_hours)`, así
que un override en `0` a nivel `Service` vacía la política exactamente igual
que `organizations.release_deadline_hours = 0` — el Fix 4 no lo cubría porque
es una columna distinta con su propio `CHECK`. Ahora exige `between 1 and 720`
igual que `organizations`.

**Fix 6 — los overrides de política de crédito de `services` pasan a ser de
`OWNER`, no de `STAFF`.** `services_write_staff` (Fase 2) es `for all using
(is_organization_member(organization_id))` — correcto para columnas
operativas (nombre, descripción, capacidad), pero esas cuatro columnas son
política de negocio: encienden/ajustan el crédito de recupero para **ese**
`Service` puntual, pisando el default de la organización (ADR-0025
resolución 3, "opt-in explícito del dueño"). Sin este fix, cualquier `STAFF`
podía hacer
`PATCH /rest/v1/services?id=eq.<X> {"makeup_credits_enabled_override": true, "release_deadline_hours_override": 1}`
directo por PostgREST con su propio JWT y encender el crédito para ese
servicio aunque el `OWNER` tuviera el interruptor general apagado, además de
vaciar la anticipación con `1` — exactamente lo que esa resolución vino a
impedir. La policy de `UPDATE` no puede pasar a OWNER-only completa (el resto
de columnas de `services` sigue necesitando ser editable por `STAFF`), así
que el chequeo va en un trigger nuevo, por columna, no en la policy:
`check_service_billing_override_owner()` (`BEFORE UPDATE`) compara `new` vs.
`old` de las cuatro columnas con `is distinct from` y, si alguna cambió,
exige `is_organization_owner(new.organization_id)` — misma función que ya usa
`organizations_update_owner` — o rechaza con `NOT_AUTHORIZED`. Cualquier otra
columna de `services` sigue sin este chequeo.

**Fix 3 (revisión), menor — comentario del test de concurrencia desactualizado.**
El comentario de `test/phase25.production-feedback.test.ts` decía que el fix
"lockea la fila del Customer (`for update`) antes del SELECT" — es la versión
que se probó primero y **deadlockeaba** (ver Fix 1 arriba); la que quedó es un
advisory lock por `customer_id`. Corregido para que el comentario del test
describa lo mismo que este documento.

No tocado (hallazgos preexistentes, anotados para una fase futura, evaluados
y aceptados como fuera de alcance de esta ronda por `security-engineer`):
`payments_no_duplicate_on_resurrect` (Fix 2) no cubre la edición de período de
un pago que ya está en un estado no-`VOID` (sólo dispara en la transición
`VOID` → no-`VOID`); staleness de `payment_service_coverage` cuando se edita
`payments.customer_id`/`service_plan_id` (Fase 22); y el `grant PUBLIC` inerte
en funciones de trigger (informativo).

Test nuevo para el Fix 5 (mismo patrón que el de `organizations.release_deadline_hours`,
sobre el override de `services`) y test nuevo para el Fix 6 (un `STAFF`
intentando tocar cualquiera de las cuatro columnas de override falla con
`NOT_AUTHORIZED`; un `OWNER` puede; una columna operativa como `name` sigue
editable por `STAFF`), ambos en
`test/phase25.production-feedback.test.ts`.

Verificado: `npx supabase db reset` completo aplica limpio desde cero (incluido
el `drop function schedule_rule_standing_reservations(uuid)` de §3), y la
suite de integración completa (182 tests, 19 archivos) pasa con
`--no-file-parallelism`.

## Fase 27 — `organizations_slug_format_check` (hallazgo de `security-engineer`, migración `20260923140000_phase27_organization_slug_format_check.sql`)

`organizations.slug` era `text not null unique` sin ninguna validación de
formato a nivel de base desde la Fase 1 — sólo `organizationSlugSchema`
(Zod, `backend/src/schemas.ts`) del lado del formulario, y
`create_organization_with_owner()` insertaba `p_slug` tal cual. Un usuario
autenticado podía llamar `rpc/create_organization_with_owner` directo por
PostgREST con un slug malicioso (`/evil.com`, `//evil.com`) saltándose Zod
por completo. Detalle del vector y por qué importa en `docs/security.md`.

**El `CHECK`, exacto espejo de `organizationSlugSchema`:**

```sql
alter table public.organizations
  add constraint organizations_slug_format_check
  check (
    char_length(slug) between 2 and 60
    and slug ~ '^[a-z0-9]+(-[a-z0-9]+)*$'
  );
```

Agregado como `CHECK` directo, no `NOT VALID` + `VALIDATE`: el único camino
de escritura de `slug` en las 27 migraciones de este repo es
`create_organization_with_owner()` (grepeado explícitamente antes de
decidir esto — no hay ningún `UPDATE ... set slug` en ninguna función), y
esa RPC hasta ahora sólo se invocó desde el formulario del frontend, que ya
corre este mismo patrón antes de llamarla. No hace falta backfill.

**`create_organization_with_owner()` (firma sin cambios, 4 args de la Fase
10) traduce la violación a `INVALID_SLUG`** en vez de dejar pasar el
mensaje crudo de Postgres (`new row for relation "organizations" violates
check constraint ...`), mismo criterio que cualquier otro RPC de este repo
(`SLOT_FULL`, `INVITE_NOT_FOUND`, etc. — nunca una excepción genérica sin
traducir):

```sql
begin
  insert into public.organizations (...) values (...) returning * into v_org;
exception
  when check_violation then
    get stacked diagnostics v_constraint_name = constraint_name;
    if v_constraint_name = 'organizations_slug_format_check' then
      raise exception 'INVALID_SLUG';
    end if;
    raise;
end;
```

`get stacked diagnostics ... = constraint_name` en vez de matchear
`sqlerrm` con `like` — evita atrapar por accidente una violación de un
`CHECK` distinto que algún día se agregue a la tabla y termine
reportándose también como `INVALID_SLUG`. Es `create or replace`, no
`drop`+`create`: los grants existentes (`grant ... to authenticated` de la
Fase 10, `revoke ... from public, anon` de la Fase 19/ADR-0028) se
preservan sin re-declararlos (`DROP` sí los resetea, `CREATE OR REPLACE`
no).

Test: `test/phase27.organization-slug-format.test.ts` (6 casos —
`/evil.com`, `//evil.com`, mayúsculas, vacío, guión al inicio/final/doble,
caso feliz). Verificado: `npx supabase db reset` limpio desde cero y la
suite de integración completa (194 tests, 21 archivos) pasa con
`--no-file-parallelism`.
