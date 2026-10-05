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

## Fase 28 — catálogo de planes para el cliente y `plan_change_requests` (feedback de producción, migración `20260923150000_phase28_plan_catalog_and_change_requests.sql`)

Feedback: *"si excedo frecuencia de plan, ofrecer upgrade/downgrade de
planes y redireccionar a planes"*. Antes de esta fase, un rechazo
`OVER_PLAN_QUOTA`/`OUTSIDE_PLAN_QUOTA` sólo ofrecía "Ver mi plan"
(`/me/servicios`), que muestra el plan que ya tiene — justo el que no le
alcanza — y el único lugar donde existían los demás planes era
`/org/:slug/plans`, admin-only.

**La migración no toca el motor de reservas.** Ni una línea de
`evaluate_customer_booking()`, `evaluate_payment_coverage()`,
`payment_covers_slot()` o `assert_series_within_plan_quota()`. Es
aditiva: una función de lectura y una tabla que **ninguna función de
decisión lee**.

### `public_service_plans(p_organization_slug, p_service_id default null)`

La lista de precios que la organización publica: planes `is_active`, con
`name`, `description`, `price`, `currency` (de `organizations`, ADR-0024
resolución 2), `plan_kind`, `weekly_quota`, `quota_scope`,
`billing_type`/`billing_cycle`, y el conjunto cubierto ya resuelto
(`service_ids[]`, `service_names[]`).

RPC y no vista, por tres razones concretas:

1. `applies_to_all_services` (ADR-0029) se resuelve en vivo contra los
   servicios **activos**, y el helper que ya sabe hacerlo
   (`service_plan_covered_service_ids()`) está revocado de todos los roles
   a propósito (ADR-0026 resolución 7): resuelve contra todos los
   servicios sin mirar quién pregunta. La función nueva es `SECURITY
   DEFINER` y hace el mismo `union` con `s.is_active` adentro.
2. Una selección explícita (`service_plan_services`) puede incluir un
   servicio desactivado después. Ese servicio **no** sale al público —
   `services_public` tampoco lo muestra.
3. Un plan cuyos servicios cubiertos están todos inactivos se omite
   entero (`cardinality(covered.ids) > 0`): sería un precio sin nada que
   reservar detrás.

No amplía disclosure: `service_plans_select_public` (Fase 22) ya deja leer
los planes activos por PostgREST a `anon`. `grant execute ... to anon,
authenticated`.

`p_service_id` filtra a los planes que cubren ese servicio — la forma que
necesitan las pantallas de rechazo, donde la persona está parada frente a
un servicio.

### `plan_change_requests` — el pedido, no el cambio

El cambio de plan real sigue siendo **VOID + recargar** (ADR-0024
resolución 1), o sea una operación de dinero del mostrador. Sin cobro
online (ADR-0027, bloqueada por elección de pasarela), un cambio que se
aplicara solo tendría que crear cobertura que nadie cobró — exactamente
lo que "pago ≠ permiso" (ADR-0005/ADR-0013) existe para impedir. Así que
la tabla registra un **hecho accionable para el negocio**, con el patrón
de lead de `platform_contact_requests` (Fase 24) pero dentro del portal y
con identidad real: el autor es un `Customer` autenticado, así que no hace
falta rate limit por IP.

```
plan_change_requests(
  id, organization_id, customer_id,
  service_plan_id,            -- lo que pide (ON DELETE RESTRICT)
  current_service_plan_id,    -- lo que lo cubre hoy, resuelto en la base
  note, created_at,
  resolved_at, resolved_by, resolution  -- APPLIED | DISMISSED
)
```

- `current_service_plan_id` **nunca viene del caller**: lo resuelve
  `request_plan_change()` con `resolve_covering_service_plan()` sobre los
  servicios que cubre el plan pedido, en la **fecha local de la
  organización** (ADR-0014). Es lo que convierte la fila en "upgrade" o
  "downgrade" a ojos del mostrador. `null` = no tenía ninguno vigente, que
  también es información.
- Índice único parcial `(customer_id, service_plan_id) where resolved_at
  is null`: el doble click y el "insisto" no son dos ventas.
- `plan_change_requests_resolution_consistent`: nunca resuelta sin decir
  cómo, nunca un resultado sin fecha (mismo par que
  `cancelled_at`/`cancelled_by`).
- Trigger `check_plan_change_request_same_org()`: cliente y planes de la
  misma organización que la fila.
- RLS de dos capas (ADR-0006): `select` para el propio cliente o un
  miembro de la organización; **cero policies de escritura** — cada fila
  nace y se cierra por RPC `SECURITY DEFINER`, igual que `makeup_credits`.

### Las cuatro RPCs

| Función | Nivel | Qué hace |
|---|---|---|
| `request_plan_change(p_service_plan_id, p_note)` | CUSTOMER | Resuelve el `Customer` desde `auth.uid()` (ADR-0005, nunca un `customerId` del caller). Rechaza plan inactivo (`SERVICE_PLAN_NOT_AVAILABLE`), no-cliente (`NOT_A_CUSTOMER`), organización inactiva, nota > 500 y pedir el plan que ya lo cubre (`ALREADY_ON_PLAN`). **Idempotente**: repetir un pedido pendiente devuelve el mismo. Tope de 5 pendientes por cliente bajo `pg_advisory_xact_lock` (mismo patrón que `submit_platform_contact_request()`). |
| `my_plan_change_requests()` | CUSTOMER | Los propios, pendientes primero. Sin esto el botón es un agujero negro. |
| `organization_plan_change_requests(p_organization_id, p_include_resolved)` | ADMIN | Con el plan vigente al lado del pedido. `is_organization_member()` dentro del `WHERE`: un no-miembro recibe lista vacía, nunca la de otro. |
| `resolve_plan_change_request(p_request_id, p_resolution)` | ADMIN | `APPLIED`/`DISMISSED`. Idempotente: no pisa quién ni cuándo la cerró. Miembro (no OWNER-only): cerrar un pedido es trabajo de mostrador y no cambia ningún precio. |

### El pago del plan pedido cierra el pedido

Trigger `payments_close_plan_change_request` (`after insert on payments`,
función `close_plan_change_request_on_payment()`): si el pago no es `VOID`
y tiene `service_plan_id`, marca `APPLIED` el pedido **pendiente** de ese
`(customer_id, service_plan_id)`. Sin esto, vender el plan nuevo deja el
pedido abierto y la lista de pendientes se llena de trabajo ya hecho — el
mostrador dejaría de mirarla en una semana.

Sólo escribe en `plan_change_requests`: no valida el pago, no puede
rechazarlo y no participa de ninguna decisión de cobertura. Re-cobrar el
mismo plan más adelante no pisa una resolución anterior (el `WHERE` exige
pendiente).

Test: `test/phase28.plan-catalog-and-change-requests.test.ts` (19 casos:
catálogo público, filtro por servicio con `applies_to_all_services`,
aislamiento cross-tenant, `NOT_A_CUSTOMER`/`ALREADY_ON_PLAN`/plan
inactivo/plan ajeno, idempotencia, RLS self-or-staff, escritura directa
rechazada, cierre automático por pago, cierre manual idempotente, y que
pedir un cambio **no mueve** lo que el cliente puede reservar).
Verificado: `npx supabase db reset` limpio desde cero + suite de
integración completa (214 tests, 22 archivos).

## Fase 29 — `my_bookings()`: identidad de la ocurrencia y color de servicio (migración `20260923160000_phase29_my_bookings_slot_identity.sql`)

Cierre pendiente de la agenda real de `/me` (Fase 23-ish): `my_bookings()`
no traía `slot_occurrence_id` ni `service_id`, así que el frontend
deduplicaba una reserva propia contra `get_public_availability()`
comparando `startAt + serviceName` (`occurrenceKey()` en
`frontend/lib/my-agenda.ts`) — un heurístico documentado como aceptable
pero evitable. Tampoco traía `service_color`, así que no había forma de
pintar el bloque de la reserva propia con el mismo color que la
disponibilidad pública del mismo servicio (`services.color`, Fase 14/16).

`my_bookings(p_include_past boolean)` gana tres columnas —
`slot_occurrence_id`, `service_id`, `service_color` — sin tocar el `WHERE`
ni ninguna regla de negocio. `DROP FUNCTION` + `CREATE` (no `CREATE OR
REPLACE`): agrega columnas al resultado, y Postgres no permite cambiar los
`OUT` parameters de una función con `REPLACE` (mismo motivo ya documentado
en Fase 11 al agregar `not_generated_reason`).

**Nota de seguridad:** `DROP FUNCTION` borra el objeto y con él sus
privilegios — Postgres otorga `EXECUTE` a `PUBLIC` por default en una
función nueva. Fase 19 (security fix) había revocado explícitamente
`PUBLIC`/`anon` sobre `my_bookings(boolean)`; la migración repite ese
`revoke` después del `create` para no reabrir la función a un rol anónimo.
Verificado en local: `has_function_privilege('anon', ..., 'EXECUTE')` →
`false`, `has_function_privilege('authenticated', ..., 'EXECUTE')` →
`true`.

No hay migración de datos ni cambio de tabla — `slot_occurrence_id` y
`service_id` ya estaban disponibles vía los `join` existentes
(`slot_occurrences`, `services`); solo faltaba proyectarlos.

Verificado: `npx supabase db push --local` limpio + suite de integración
completa (214 tests, 22 archivos) contra la función recreada.

## Fase 30 — `audit_log` (ADR-0032, migración `20260923170000_phase30_audit_log.sql`)

Registro de hechos sensibles. Aditiva: **una tabla, un enum, cuatro
triggers de auditoría, un trigger de inmutabilidad, una policy de
`SELECT` y una función de lectura**. No toca el motor de reservas, ni el
de cobertura, ni ninguna firma existente.

### Por qué trigger y no un `insert` en cada RPC

Porque **la mitad de las escrituras del alcance no pasa por ninguna
RPC** (verificado en el código, no supuesto): el alta de pago es un
`INSERT` de PostgREST (`frontend/app/actions/billing.ts`), la anulación
es un `UPDATE payments set status='VOID'` directo, y los dos cambios de
plan (precio/nombre y activar/desactivar) son `UPDATE` directos
(`service-plans.ts`). Un insert explícito por RPC habría dejado afuera
justo eso. Es el mismo criterio del invariante 6 de `invariants.md` —
toda tabla es escribible por PostgREST, así que la defensa vive en la
base — y la lección de ADR-0028: lo que se confía a que cada llamador
recuerde hacer, en algún lado no se hace.

### El mapa: qué acción registra qué trigger

Esta tabla existe para que "¿esto se audita?" se conteste acá y no
leyendo ocho funciones.

| Acción | Cómo se escribe hoy | Trigger | `action` |
|---|---|---|---|
| Alta de pago | `INSERT payments` (PostgREST) | `payments_audit_insert` | `PAYMENT_CREATED` |
| Cambio de estado de pago, incluida la anulación | `UPDATE payments` (PostgREST) / `set_payment_status()` | `payments_audit_status` (`when old.status is distinct from new.status`) | `PAYMENT_STATUS_CHANGED` |
| Reserva creada por el mostrador | `admin_book_for_customer()` | `bookings_audit_insert` | `BOOKING_CREATED_BY_STAFF` |
| Cancelación hecha por staff | `cancel_booking()` con actor staff, `cancel_slot_occurrence()`, `discontinue_schedule_rule()`, cascada de serie | `bookings_audit_cancel` | `BOOKING_CANCELLED_BY_STAFF` |
| Alta de plan | `create_service_plan()` | `service_plans_audit_insert` | `SERVICE_PLAN_CREATED` |
| Cambio de precio / nombre / orden | `UPDATE service_plans` (PostgREST) | `service_plans_audit_update` | `SERVICE_PLAN_UPDATED` |
| Activar / desactivar plan | `UPDATE service_plans` (PostgREST) | `service_plans_audit_update` | `SERVICE_PLAN_DEACTIVATED` (metadata dice `from`/`to`) |
| Suspensión / cambio de plan SaaS | `set_organization_subscription()` | `organizations_audit_subscription` | `ORGANIZATION_SUBSCRIPTION_CHANGED` |

**Fuera de alcance, explícito**: asistencia (alto volumen: una clase de 3
personas son 3 filas por día), créditos de recupero (`makeup_credits`
**ya es su propio log**: `origin`, `issued_by`, `source_booking_id`,
`consumed_booking_id`, RLS `SELECT`-only — duplicarlo crearía dos
versiones de la misma verdad), acciones del propio cliente, altas de
cliente/servicio/horario/branding, logins y lecturas.

### Shape

`id`, `organization_id` (FK, nullable), `actor_id` (FK a `profiles`,
nullable), `action` (enum de 8 valores), `target_table` + `target_id`,
`metadata` jsonb, `created_at`. Dos índices:
`(organization_id, created_at desc)` — el de la pantalla, y el que un
borrado por retención necesitaría (ADR-0032 resolución 3: **sin política
de retención por ahora**, pero el índice queda para no agregarlo sobre
una tabla ya grande) — y `(target_table, target_id, created_at desc)`
para "¿qué pasó con este pago?".

- **`target_table` + `target_id` y no una FK por tabla.** Cuatro FK
  nullables para expresar "una de estas cuatro" no se consultan
  uniformemente y tres siempre están vacías. Se renuncia a la integridad
  referencial sobre el target **a propósito**: el log tiene que
  sobrevivir al borrado de lo que audita, y una FK con cascade borraría
  justo la evidencia.
- **`metadata` es el diff mínimo** (`{"status":{"from":"PENDING","to":"VOID"}}`,
  `{"price":{"from":2500,"to":3400}}`), armado por una función por tabla
  y **nunca** un `to_jsonb(new)` genérico — ese es el modo típico de
  fallar de estas tablas: termina duplicando datos personales fuera de su
  RLS. Ids, montos, estados y fechas sí; nombres, mails y teléfonos no.
- **Sin `updated_at` ni `cancelled_at`**: una fila de auditoría es
  inmutable por definición y darle columnas de ciclo de vida invita a
  editarla.

### Inmutabilidad

RLS habilitada, **cero policies de escritura** para ningún rol,
`revoke insert, update, delete ... from anon, authenticated` (+
`revoke select from anon`), y `reject_audit_log_mutation()` como trigger
`before update or delete` (y `before truncate`) que siempre lanza
`AUDIT_LOG_IMMUTABLE`. **`service_role` no se exceptúa**: verificado en
la suite, un cliente service-role puede leer pero no puede actualizar ni
borrar. Corregir una fila exige un superusuario desactivando el trigger —
una operación deliberada y fuera de banda, que es exactamente lo que
corresponde.

Dos consecuencias conocidas y aceptadas, documentadas en la migración:

- `organization_id` es `on delete cascade`, así que **borrar una
  organización a mano falla** con `AUDIT_LOG_IMMUTABLE`. Hoy no existe
  ninguna policy de `DELETE` sobre `organizations`, así que no rompe
  ningún camino del producto.
- `actor_id` no tiene acción de borrado (igual que `payments.created_by`
  desde la Fase 7): borrar un `auth.users` que dejó filas de auditoría
  falla. **Ya pasaba antes** por esas otras FK; no es nuevo.

### El actor, y la lógica de tres valores

`audit_write()` es el único punto de inserción: resuelve
`actor_id = auth.uid()` y **lo baja a `NULL` si ese uid no tiene
`profile`** — la auditoría nunca puede ser el motivo por el que un pago
no entra.

La condición "por staff, no por el propio cliente" se escribe
`profile_id is distinct from auth.uid()`, **nunca `<>`**: con un cliente
gestionado (ADR-0026) `profile_id` es `NULL`, `<>` daría `NULL`, el `if`
no entraría y la reserva que por definición hizo el mostrador quedaría
sin auditar. Es el mismo error que la Fase 19 encontró en
`cancel_booking()`, donde saltaba la autorización entera. Tiene su test.

**Dos exclusiones del trigger de `bookings`, con su motivo:**

1. `bookings_audit_insert` tiene `when (new.recurring_booking_id is
   null)`: las fechas de una serie las materializa
   `generate_recurring_booking()` (una por semana, por serie, por
   cliente) y son consecuencia de la serie, no un alta de mostrador —
   auditarlas multiplicaría el volumen del log por dos órdenes de
   magnitud y no hay valor de enum que las describa. El alta de la serie
   vive en `recurring_bookings` (`created_by`, `status`).
2. Un `INSERT` sin sesión (`auth.uid() is null`) es el job de horizonte
   (ADR-0009) y tampoco se registra. Una **cancelación** sin sesión sí:
   es una decisión que nadie reclama, justo lo que hay que poder
   rastrear.

### Contexto opcional desde la RPC

`set_config('app.audit_note', ..., true)` — local a la transacción. El
trigger la levanta si está (`metadata.note`, recortada a 500) y **la fila
se escribe igual si no está**: el contexto es opcional, el hecho no.
`set_organization_subscription()` es el primer consumidor real
(`'PLATFORM_CONSOLE'`), reescrita con `create or replace` (misma firma →
conserva los grants de la Fase 10 y los revoke de la Fase 19).

### Lectura

`audit_log_select`: `is_platform_admin()` ve todo; el `OWNER` ve el log
de su organización, **incluidas** las filas de
`ORGANIZATION_SUBSCRIPTION_CHANGED`; `STAFF` no ve nada (es el sujeto
mayoritario del log). El matiz de la resolución 2 —que la identidad del
actor de plataforma no se resuelva a un nombre del lado del cliente— lo
implementa `organization_audit_log()` (ver `docs/api.md` y
`docs/security.md`), que es la lectura que consume el frontend.

Test: `test/phase30.audit-log.test.ts` (12 casos). Verificado:
`npx supabase db push --local` limpio, `db reset` completo desde cero, y
la suite de integración completa (**226 tests, 23 archivos**) en verde
con `--no-file-parallelism`.

## Fase 31 — ciclos de facturación largos y prorrateo del primer período (ADR-0031, migración `20260923180000_phase31_long_billing_periods.sql`)

Aditiva y sin migración de datos: **dos valores de enum, dos columnas, dos
ramas nuevas en `billing_period_for()`, una función de cotización nueva y
dos funciones extendidas**. `billing_period_months` nulo se lee como 1, así
que ningún plan ni pago existente cambia de comportamiento el día del
deploy (mismo patrón que el `UNLIMITED` de ADR-0024 y el
`makeup_credits_enabled` de ADR-0025).

**No toca el motor de reservas.** `evaluate_payment_coverage()` sigue
preguntando "¿la fecha local del turno cae dentro del período pago?"
(ADR-0013) y un período de tres meses o de un año responde eso sin una
línea nueva. Todo lo de esta fase vive del lado de *escribir* un pago.

### `service_plans`: `billing_period_months` y `billing_anchor_month`

- `billing_cycle` gana `CALENDAR_PERIOD` y `ROLLING_PERIOD`.
  `CALENDAR_MONTH`/`ROLLING_MONTH` **no se tocan ni se deprecan**: son el
  caso `billing_period_months = 1` y el 100% de los datos existentes.
- `billing_period_months int` — cada cuántos meses cobra el plan. `NULL` = 1.
- `billing_anchor_month int` — mes (1..12) en que arranca el bloque
  calendario. `NULL` = arranca el mes de la compra (y entonces no hay nada
  que prorratear). Sólo para `CALENDAR_PERIOD`.

Tres `CHECK` nuevos, todos de una sola fila:

| Constraint | Qué impide |
|---|---|
| `service_plans_billing_period_months_matches_cycle` | Una segunda representación de lo mismo: los dos ciclos `_PERIOD` exigen la cantidad de meses, los `_MONTH` y el `ONE_TIME` la prohíben (bicondicional, con `coalesce(billing_cycle::text,'')` para que un `NULL` no lo vuelva trivialmente cierto). |
| `service_plans_billing_period_months_range` | 1..12, y para `CALENDAR_PERIOD` además **divisor de 12** (1, 2, 3, 4, 6, 12): si los bloques no tapizan el año exacto, el "trimestre" se corre de año en año y el anclaje deja de significar lo que la pantalla promete. Un ciclo rodante no tiene esa restricción porque no se ancla a nada. |
| `service_plans_billing_anchor_month_matches_cycle` | Anclar un ciclo que no se ancla (rodante) o cuyo bloque es de un mes (`CALENDAR_MONTH`). |

`service_plans_billing_matches_kind` (Fase 17) **no se tocó**: ya decía
"`MONTHLY` exige `billing_cycle` no nulo" sin enumerar cuáles.

`check_service_plan_terms_immutable()` suma las dos columnas a la lista de
términos congelados con pagos no-`VOID` (ADR-0024 resolución 5): la
cotización las lee **en vivo** desde el plan, nunca de una copia por pago,
así que cambiarlas reinterpretaría hacia atrás lo que alguien compró. Mover
el anclaje además desalinea de golpe a todos los clientes del plan. El
precio sigue libre.

> **Detalle de implementación que hay que conocer para leer el archivo**: un
> valor de enum agregado en la misma transacción no se puede usar como
> literal de ese tipo hasta que commitee, así que todas las comparaciones de
> ciclo van sobre `::text` (mismo recurso que el `OVER_PLAN_QUOTA` de la
> Fase 17).

### `billing_period_for()`: dos ramas nuevas, las tres viejas intactas

```
MONTHLY + CALENDAR_MONTH  -> [1 del mes de `from`, ultimo dia de ese mes]   (igual que antes)
MONTHLY + ROLLING_MONTH   -> [from, from + 1 mes - 1 dia]                    (igual que antes)
MONTHLY + CALENDAR_PERIOD -> el bloque de N meses anclado en billing_anchor_month que contiene a `from`
MONTHLY + ROLLING_PERIOD  -> [from, from + N meses - 1 dia]
ONE_TIME                  -> [from, from]                                    (igual que antes)
```

Se extendió, no se duplicó: una función "sólo para el primer pago de un
ciclo largo" obligaría a cada llamador a saber si el pago que está creando
es el primero — un dato que la base no tiene sin interpretar el historial —
y serían dos fuentes de verdad sobre qué período compra un pago.

**El período no se recorta** (resolución 2). Quien entra el 15 de septiembre
en un trimestral jul-sep compra el trimestre jul-sep y su cobertura empieza
el 1 de julio. Recortar `period_start` desalinearía el `EXCLUDE` de
solapamiento y rompería "todos los clientes del plan renuevan el mismo
día", que es el único motivo para elegir un ciclo calendario.

El módulo del anclaje se calcula sólo con el mes del año, y es correcto
porque el `CHECK` garantiza que N divide a 12. Un anual anclado en marzo
resuelve el 31-ene-2026 al bloque `2025-03-01 .. 2026-02-28`.

### `quote_service_plan_period(p_service_plan_id, p_from)` — la cotización

`stable security definer`, **sólo lectura**, `revoke` de `public`/`anon` +
`grant` a `authenticated`, con `is_organization_member()` adentro: el precio
de un plan es público, pero cotizar es una operación de mostrador. Devuelve
`period_start, period_end, full_price, prorated_price, prorated,
units_charged, units_total`.

- **Prorrateo por meses enteros, contando el mes de alta como completo**
  (resolución 1): trimestre jul-sep con alta el 15-sep → 1 de 3; el 2-ago →
  2 de 3; en julio → 3 de 3 (no prorratea). Es lo que el mostrador explica
  en una frase y no depende del largo del mes.
- **Sólo prorratea al entrar a mitad de un `CALENDAR_PERIOD` de más de un
  mes.** Un `ROLLING_PERIOD` arranca el día de la compra, así que no hay
  fracción que descontar — nunca prorratea. `CALENDAR_MONTH`,
  `ROLLING_MONTH` y `ONE_TIME` tampoco.
- Redondeo a la unidad entera de moneda, half-up (`round(numeric, 0)`).
  `organizations.currency_minor_units` queda diferido a ADR-0027
  (resolución 5). Cuando **no** prorratea devuelve el precio de lista *tal
  cual*, sin pasar por `round()`: si no, un plan de 1500.50 se cobraría 1501
  y aparecería marcado como prorrateado.
- `least(greatest(...))` y no un `CHECK`: el monto sugerido nunca puede
  pasar el precio de lista ni bajar de 0, pero **`payments.amount` sigue
  libre** a propósito (ADR-0024 resolución 5). Se acota la sugerencia, no el
  cobro.
- **Es función pura de (plan, fecha): no lee ni un pago.** Por eso no puede
  descontar un cambio de plan (resolución 3) ni reconocer lo no consumido de
  un plan anterior (resolución 4, sin nota de crédito). El cambio de plan a
  mitad de período sigue siendo VOID + recargar el **período completo**
  (ADR-0024 resolución 1): cotizado desde el inicio del período que se
  recarga, la respuesta es el precio completo. Si el negocio quiere
  reconocer algo, lo ajusta a mano en `amount`.

Separada de `billing_period_for()` a propósito: esa función es la que llaman
cuatro caminos de lectura del motor, y meterle precio la convertiría en la
función que todos llaman para todo. La envuelve, no la reimplementa.

### `create_service_plan()` y el crédito de recupero

- `create_service_plan()` gana `p_billing_period_months` y
  `p_billing_anchor_month`, **con default `NULL`**. `DROP` + `CREATE` (no un
  segundo `CREATE OR REPLACE`): dos firmas coexistiendo serían una sobrecarga
  que PostgREST tendría que desambiguar por nombre de argumento. Se repiten
  `revoke`/`grant` sobre la firma nueva — un `DROP FUNCTION` se lleva los
  privilegios y Postgres otorga `EXECUTE` a `PUBLIC` por default (misma nota
  de la Fase 29).
- `issue_makeup_credit()`: en un plan con `billing_period_months > 1`,
  `END_OF_BILLING_PERIOD` se **acota a `END_OF_MONTH`** (resolución 6). Un
  crédito vivo tres meses en un plan trimestral triplica el riesgo que
  ADR-0025 ya dejó anotado como abierto. Va acá y no en
  `resolve_makeup_credits_policy()` porque esa política se resuelve por
  `Service` (organización + override del servicio) y no sabe qué plan cubrió
  la reserva liberada; `issue_makeup_credit()` ya tiene el plan en la mano
  para decidir el alcance del crédito (ADR-0029). La fila del crédito guarda
  la base **efectiva** (`END_OF_MONTH`), no la de la política: `expires_on`
  es inmutable, así que la auditoría tiene que poder explicar la fecha. Para
  ciclos mensuales no cambia nada.

Test: `test/phase31.long-billing-periods.test.ts` (8 casos: cotización 1/3 y
2/3 de trimestre, ciclo rodante que nunca prorratea, borde de mes en los dos
sentidos, anual anclado que cruza el año, los tres `CHECK` nuevos,
inmutabilidad con pagos no-`VOID`, cambio de plan que se recarga completo,
crédito trimestral que vence a fin de mes con control mensual, y
autorización cross-tenant de la cotización). Verificado: `supabase migration
up --local` limpio y la suite de integración completa (**255 tests, 25
archivos**) en verde.

## Fase 32 — roles configurables por organización (ADR-0033, migración `20260923190000_phase32_configurable_roles.sql`)

Implementa ADR-0033 completa. Antes de esta fase el rol era binario
(`organization_member_role` = `OWNER | STAFF`) y **todo lo que no estaba
explícitamente cerrado a `OWNER` lo podía hacer cualquier `STAFF`** —
incluida la cobranza entera. Lo nuevo es una capa de roles configurables
**dentro** de `STAFF`. `OWNER` no cambia una línea: sigue siendo enum fijo,
nunca se evalúa contra un permiso y corta antes de mirar el rol.

### Orden de la migración (importa)

1. Enum `org_permission`, tabla `organization_roles`,
   `organization_members.role_id`.
2. Trigger de siembra + backfill del rol por defecto.
3. `has_org_permission()`.
4. Endurecimiento de policies y RPCs.

Al revés hay una ventana dentro de la transacción en la que
`has_org_permission()` devuelve `false` para todos, porque todavía no existe
el rol por defecto al que caen los `STAFF` sin `role_id`.

### `org_permission` (enum, no `text`)

`VIEW_PAYMENTS`, `MANAGE_PAYMENTS`, `MANAGE_BOOKINGS`, `MANAGE_CUSTOMERS`,
`MANAGE_ATTENDANCE`. Enum a propósito: un typo en una policy
(`'VIEW_PAYMENT'`) falla en la migración con `invalid input value for enum`,
no en producción con un permiso que silenciosamente deniega. El `else false`
del `CASE` dentro del helper es el segundo cinturón.

### `organization_roles`

| Columna | Notas |
|---|---|
| `organization_id` | FK a `organizations`, `on delete cascade` |
| `name` | **texto libre** elegido por la organización; `CHECK` 1..60 caracteres |
| `is_default` | el rol que recibe un `STAFF` sin `role_id` |
| `is_active` | `CHECK (not is_default or is_active)` |
| `can_view_payments` … `can_manage_attendance` | 5 columnas `boolean not null default true` |

Permisos como columnas booleanas y **no** `jsonb` (ADR-0033 §4.2):
`(permissions->>'view_payments')::boolean` con una clave ausente o mal
tipeada da `NULL`, y un `if NULL` no entra — que es exactamente la raíz del
bypass de autorización de ADR-0026/ADR-0028. Con `coalesce(..., false)` se
falla cerrado pero en silencio. Una columna booleana `not null` no tiene ese
estado. Bonus: agregar un permiso más adelante es
`add column can_x boolean not null default true` y ningún rol existente
cambia de comportamiento — el `DEFAULT` *es* el mecanismo de migración.

Constraints e índices:

- `organization_roles_manage_implies_view`:
  `check (not can_manage_payments or can_view_payments)`. Cobrar sin poder
  ver lo cobrado no es un rol, es un bug.
- `organization_roles_org_name_idx`: único por
  `(organization_id, lower(trim(name)))`.
- `organization_roles_one_default_idx`: único parcial por
  `(organization_id) where is_default` → **como máximo** un default. Es
  inmediato, así que mover el default siempre es "desmarcar y después
  marcar".
- `organization_roles_default_present`: **constraint trigger diferido**
  (`deferrable initially deferred`) → **como mínimo** un default. Diferido
  porque el swap son dos statements y un chequeo inmediato rechazaría el
  primero. Una organización ya borrada no se chequea (el cascade se lleva
  sus roles y no hay nada que exigir).
- `organization_roles_not_in_use` (BEFORE UPDATE OR DELETE): un rol con
  miembros activos no se desactiva ni se borra → `ROLE_IN_USE`. Si se
  pudiera, esos miembros caerían al rol por defecto de golpe, en silencio y
  posiblemente **ganando** permisos.

### `organization_members.role_id`

Nullable a propósito: `organization_members` es escribible por PostgREST
directo (`organization_members_write_owner` era `for all`; desde la Fase 36 son
`organization_members_insert_owner` + `organization_members_update_owner`, misma
expresión y sin `DELETE` — ADR-0037), así que un
`not null` rompería cualquier escritura que hoy no manda la columna — y
ADR-0028 ya dejó la lección de que lo que se confía a que cada llamador
recuerde hacer, en algún lado no se hace. El fallback al rol por defecto no
es fail-open: ese rol lo configura el `OWNER`.

Dos guardas nuevas, en la base y no en el action:

- `organization_members_owner_has_no_role`:
  `check (role <> 'OWNER' or role_id is null)`. La propuesta preveía "la
  pantalla no ofrece rol para un `OWNER`"; el `CHECK` mata la contradicción
  ("le puse el rol Profesor al dueño y sigue viendo todo") también para un
  `PATCH` directo.
- `organization_members_role_same_org` (trigger): un rol de **otra**
  organización no es un rol de este miembro → `ROLE_OTHER_ORGANIZATION`. La
  FK sola no lo impide (apunta a `organization_roles`, no a "los roles de
  esta organización") y el resultado sería un cross-tenant de autorización.
  Mismo precedente que `check_payment_same_org()`.

### Siembra + backfill: nadie pierde acceso el día del deploy

`organizations_seed_default_role` (AFTER INSERT on `organizations`) crea el
rol `'Equipo'` con los cinco booleanos en `true` y `is_default`. Va como
**trigger y no como línea dentro de `create_organization_with_owner()`**:
una organización sin rol por defecto es una organización donde todo `STAFF`
futuro queda sin permisos, y ADR-0028 ya demostró que lo que depende de que
cada llamador se acuerde, en algún camino no pasa (esto cubre además el alta
de ADR-0034 y cualquier inserción por `service_role` en tests).

El backfill hace lo mismo para las organizaciones existentes y asigna
`role_id` a los miembros **`STAFF`** (los `OWNER` quedan en null por el
`CHECK`). Comportamiento **idéntico, bit a bit**, al del día anterior.

### `has_org_permission(organization_id, org_permission)`

Única puerta de permisos. `security definer` (rompe la recursión de RLS que
documenta `phase1:204-213`), `stable`, y **con `EXECUTE` para `PUBLIC` a
propósito** — es una de las funciones "correctas por construcción que RLS
necesita evaluar" que ADR-0028 dejó exceptuadas del `revoke from public`;
sólo contesta sobre `auth.uid()`.

```
OWNER                  -> true, sin consultar organization_roles
STAFF con role_id      -> la columna del permiso en ese rol
STAFF con role_id null -> la columna del permiso en el rol is_default
rol inactivo / ausente -> NULL -> el EXISTS no devuelve fila -> false
```

`my_organization_permissions(organization_id)` es el read model para el
panel: devuelve `role`, `role_id`, `role_name` y los cinco booleanos ya
resueltos. Sin filas para un no-miembro.

### Dónde se aplica: policies **y** RPCs

Policies endurecidas (la **rama del cliente no se toca** — ADR-0006 de dos
capas sigue igual; sólo la rama de staff pasa de "es miembro" a "es miembro
con el permiso"):

| Policy | Permiso |
|---|---|
| `payments_select_self_or_staff` | `VIEW_PAYMENTS` |
| `payments_insert_staff`, `payments_update_staff` | `MANAGE_PAYMENTS` |
| `payment_service_coverage_select_self_or_staff` | `VIEW_PAYMENTS` |
| `plan_change_requests_select_self_or_staff` | `VIEW_PAYMENTS` |
| `customers_write_staff` | `MANAGE_CUSTOMERS` |

⚠️ **Renombrada en la Fase 35 (ADR-0036):** `customers_write_staff` ya no existe.
La misma expresión `has_org_permission(organization_id, 'MANAGE_CUSTOMERS')` vive
ahora en `customers_insert_staff` + `customers_update_staff` — la policy se partió
para quitarle el `DELETE`. El permiso no cambió.

**17 RPCs `security definer` cambiaron su chequeo interno.** Esto es lo que
decide si la feature sirve o es decorado: una función `security definer`
corre como dueña de la tabla y **bypassea RLS por completo**, así que
endurecer la policy de `payments` no la toca — el dato seguiría saliendo por
`POST /rest/v1/rpc/organization_payment_summary` con la pestaña escondida.
Ninguna cambió de firma.

| Permiso | RPCs |
|---|---|
| `VIEW_PAYMENTS` | `organization_payment_summary()`, `customer_payment_detail()`, `organization_plan_change_requests()` |
| `MANAGE_PAYMENTS` | `set_payment_status()`, `resolve_plan_change_request()`, `book_slot_paying()` (Fase 44, ADR-0046 — además exige `MANAGE_BOOKINGS` si hace falta crear la `Booking`) |
| `MANAGE_BOOKINGS` | `admin_book_for_customer()`, `admin_create_recurring_booking()`, `admin_preview_recurring_booking()`, `cancel_booking()` (rama de staff), `cancel_recurring_booking()` (rama de staff), `cancel_slot_occurrence()`, `retry_not_generated_booking()` (rama de staff) |
| `MANAGE_CUSTOMERS` | `create_managed_customer()`, `enroll_customer_by_email()`, `issue_customer_activation()`, `revoke_customer_activation()` |
| `MANAGE_ATTENDANCE` | `mark_attendance()` |

`customer_billing_horizon()` estaba en la lista de la propuesta y **no
lleva chequeo, deliberadamente**: no está otorgada a nadie (sólo al dueño de
la función). Es un helper interno que `evaluate_payment_coverage()` y
compañía llaman dentro del camino de reserva de un cliente, donde
`auth.uid()` es el cliente y no un miembro — un `has_org_permission()` ahí
rompería el booking de cualquier cliente. La puerta es el `REVOKE`, que esta
migración deja explícito (`from public, anon, authenticated, service_role`)
para que un `grant` futuro tenga que ser una decisión y no un descuido.

### RPCs de administración de roles (OWNER)

`create_organization_role()`, `update_organization_role()`,
`set_organization_role_default()`, `set_member_role()`. Todas
`is_organization_owner()`. Las policies
(`organization_roles_select_members` para lectura,
`organization_roles_write_owner` para escritura — partida en la Fase 36 en
`organization_roles_insert_owner` + `organization_roles_update_owner`, ADR-0037)
ya alcanzarían para un
`PATCH` directo, pero mover el rol por defecto son **dos statements** y dos
llamadas de PostgREST son dos transacciones (mismo razonamiento de ADR-0004
que obligó a `book_slot()`); además los errores traducidos
(`ROLE_NAME_TAKEN`, `ROLE_IN_USE`, `DEFAULT_ROLE_REQUIRED`,
`MANAGE_PAYMENTS_REQUIRES_VIEW`, `OWNER_HAS_NO_ROLE`) son mejores que una
violación de índice cruda.

**Administrar roles no es un permiso configurable**: si lo fuera, existiría
un rol capaz de ampliarse a sí mismo y los otros cinco no significarían
nada.

### Cambios de contrato

- `organization_team()` (drop + create, dos columnas nuevas): gana `role_id`
  y `role_name`, que son el rol **EFECTIVO** — el asignado, o el por defecto
  cuando `role_id` es null. Null en las dos para un `OWNER`.
- `invite_member_by_email()` (drop + create): gana
  `p_role_id uuid default null` **al final**. Por el `DEFAULT`, la llamada de
  3 argumentos que ya existía sigue resolviendo y deja al miembro en el rol
  por defecto — es aditivo, no un contrato roto (la propuesta suponía lo
  contrario).

### El hueco preexistente que se cerró (ADR-0033 resolución 3)

El precio y el nombre de un `ServicePlan` estaban protegidos a nivel `OWNER`
**sólo en TypeScript** (`frontend/app/actions/service-plans.ts`), mientras
`create_service_plan()` y las policies `service_plans_*_staff` pedían nada
más `is_organization_member()`: cualquier `STAFF` cambiaba un precio con un
`PATCH /rest/v1/service_plans` sin pasar por el panel. Ahora:

- `service_plans_insert_owner` / `service_plans_update_owner` (renombradas
  desde `*_staff`) y `service_plan_services_write_owner` (partida en la Fase 36
  en `service_plan_services_insert_owner` + `service_plan_services_update_owner`,
  ADR-0037).
- `service_plans_write_owner`, trigger `before insert or update` con
  `check_service_plan_write_owner()`. Va como **trigger y no reescribiendo
  `create_service_plan()`**: esa función es `security definer`, o sea que
  bypassea RLS y las policies no la alcanzan, mientras que un trigger se
  dispara siempre — y así no se duplica el cuerpo de una función que otras
  migraciones siguen evolucionando (la Fase 31 la rehizo). Mismo patrón que
  `check_service_billing_override_owner()` de la Fase 25.
- `auth.uid() is null` no se bloquea: ése es el camino sin JWT (backfills de
  migración, el job diario, un cliente `service_role` — que ya bypassea RLS
  por definición). Toda request real de un panel lleva su `sub`.

`service_plans_select_public` **no** cambia: sigue siendo
`is_active or is_organization_member(...)`. La lista de precios de los planes
activos es pública — "no ver pagos" no es "no ver precios".

### Lo que a propósito quedó como está

- `makeup_credits` y `organization_customer_makeup_credits()` siguen siendo
  de todo miembro: un crédito de recupero es un derecho a reprogramar, no
  dinero cobrado, y tiene su propio registro (ADR-0025).
- `organization_customers()`, `occurrence_bookings()`,
  `occurrence_attendance_summary()`, `service_attendance_history()`,
  `agenda_occurrences()`, `customer_activation_status()` y la **edición** de
  horarios (`create_schedule_rule_group()`, modificar una regla…) siguen
  siendo de todo miembro activo. Sin eso no hay trabajo que hacer.
  ⚠️ **Corregido en la Fase 34:** `discontinue_schedule_rule()` estaba en
  esta lista por error — cancela en masa reservas de clientes, así que pasó
  a exigir `MANAGE_BOOKINGS`.
- `services.price` (columna legada de pre-ADR-0024) sigue escribible por
  cualquier miembro vía `services_write_staff`. Los cuatro overrides de
  política de recupero ya eran OWNER-only por trigger desde la Fase 25.
  ⚠️ **Cerrado en la Fase 34:** `price` pasó a OWNER-only por el mismo
  trigger.

Test: `test/phase32.configurable-roles.test.ts` (21 casos: el rol por
defecto sembrado, un `STAFF` sin `role_id` que se comporta como antes, un
`OWNER` que no se filtra aunque el rol por defecto no pueda nada,
`VIEW_PAYMENTS` atacado por PostgREST **y** por las cuatro RPCs,
`MANAGE_PAYMENTS` ver-sí/escribir-no por las tres puertas, el `CHECK` de
manage-implica-view por RPC y por insert directo, `MANAGE_BOOKINGS` sobre
las cinco RPCs más la rama "mi propia reserva" que no se toca,
`MANAGE_CUSTOMERS` sobre alta/enrolamiento/activación/tabla directa,
`MANAGE_ATTENDANCE`, precios de plan cerrados a `OWNER` por RPC y por
`PATCH`, administrar roles cerrado a `OWNER`, `ROLE_IN_USE`,
`DEFAULT_ROLE_REQUIRED` y el swap atómico del default, cross-tenant de roles
por RPC y por trigger, `OWNER_HAS_NO_ROLE`, `LAST_OWNER` todavía protegido,
`organization_team()` con el rol efectivo, `invite_member_by_email()` con 3
y con 4 argumentos, y un no-miembro que no obtiene nada). Verificado:
`npx supabase db reset` limpio y la suite de integración completa
(**255 tests, 25 archivos**) en verde, más 54 unitarios del paquete de
dominio.

## Fase 33 — invitaciones de equipo (ADR-0034, migración `20260923200000_phase33_team_invitations.sql`)

Implementa ADR-0034 completa. Antes de esta fase sumar a alguien al equipo
exigía que **ya tuviera cuenta**: `invite_member_by_email()` hace
`select id from auth.users where lower(email) = ...` y levanta
`PROFILE_NOT_FOUND` si no la encuentra, así que el dueño no podía dar de alta
a un profesor — tenía que pedirle que se registrara primero, esperar, y
recién ahí invitarlo con el email exacto con el que se registró. Es el mismo
problema que ADR-0026 resolvió para clientes, del otro lado del mostrador.

`invite_member_by_email()` **no se toca**: sigue siendo el camino rápido
cuando la persona ya tiene cuenta (ADR-0034 resolución 1), elegido
explícitamente por quien invita y nunca automático — automático reintroduciría
el oráculo de `PROFILE_NOT_FOUND`.

### `new_activation_token()` — el acuñado, una sola definición

`returns table (token text, token_hash bytea)`. Las cuatro decisiones de
ADR-0026 sobre el token en un solo lugar: 256 bits de
`extensions.gen_random_bytes` (CSPRNG real, no `random()`), base64url sin
padding (43 caracteres de `[A-Za-z0-9_-]`, entra en un path segment sin
re-encodear), `sha256` en reposo, todo schema-calificado para no depender del
`search_path`.

**Sin `EXECUTE` para nadie** — ni `authenticated`, ni `service_role`. Es un
helper interno al que sólo llegan las funciones `security definer`, que corren
como su dueño. Mismo patrón (y mismo motivo) que el `REVOKE` de
`customer_billing_horizon()` en la Fase 32.

`issue_customer_activation()` se rehace para usarlo. Es lo único que esta
migración le cambia: el cuerpo es el de la Fase 32 (con su gate
`has_org_permission(..., 'MANAGE_CUSTOMERS')`, no el de la Fase 21) salvo las
dos líneas del acuñado. El punto de extraer el helper era justamente que esas
dos líneas no puedan divergir entre los dos flujos (ADR-0034 §2.4).

> Trampa comprobada: escribir un `create or replace` de una función viva
> partiendo de la versión de la migración que la **creó** en vez de la que la
> modificó **revierte** el endurecimiento posterior en silencio. Pasó acá con
> el gate de `MANAGE_CUSTOMERS` de la Fase 32 y lo cazó
> `test/phase32.configurable-roles.test.ts`. Al rehacer una función, la
> referencia es la última migración que la tocó, no la primera.

### `team_invitations`

Mecanismo **separado** de `customer_activations` (ADR-0034 decisión 1), no una
columna nueva en ella: el destino del token todavía no existe como fila (la de
`organization_members` **nace en el canje**), lo que otorga es acceso a datos
de **terceros** en todo el tenant y no "sos vos mismo", y la unicidad de "un
link vivo" es por `(organización, email)` y no por fila.

| Columna | Notas |
|---|---|
| `organization_id` | FK a `organizations`, `on delete cascade` |
| `email` | el **vínculo**; normalizado por `CHECK`, no por confianza en el llamador |
| `phone` | sólo **canal** de envío (E.164). Se guarda para poder reenviar y para saber a dónde se mandó |
| `display_name` | el nombre con el que el dueño dio el alta. No es identidad |
| `role` | `organization_member_role not null default 'STAFF'` + `CHECK (role = 'STAFF')` |
| `role_id` | FK a `organization_roles`, **nullable = el rol `is_default`** de la organización |
| `token_hash` | `bytea not null unique`, `sha256` del token. El claro nunca se guarda |
| `expires_at` | `now() + 24 h` |
| `created_by` / `redeemed_at` / `redeemed_profile_id` / `revoked_at` / `revoked_by` | ciclo de vida |

Constraints:

- `team_invitations_never_owner`: `check (role = 'STAFF')`. ADR-0034
  resolución 5 — una invitación **nunca** puede crear un `OWNER`, y es un
  `CHECK` y no una convención que la RPC deba recordar. Cierra incluso el
  camino de `service_role` (que bypassea RLS por definición). La columna existe
  en vez de asumir `'STAFF'` para que el día que el enum crezca, el `CHECK`
  siga siendo la frontera explícita.
- `team_invitations_email_lower`: `check (email = lower(btrim(email)))`. Sin
  esto, `Profe@X.com` y `profe@x.com` son dos invitaciones vivas para la misma
  persona y el índice parcial no los ve como el mismo destino.
- `team_invitations_email_shape` y `team_invitations_phone_e164`
  (`^\+[1-9][0-9]{6,14}$`, igual que `customers`).
- `team_invitations_redeemed_shape`:
  `check ((redeemed_at is null) = (redeemed_profile_id is null))`. "Canjeada" y
  "canjeada por alguien" son el mismo hecho; el estado mixto es justo el que
  haría ambigua la idempotencia del doble click.

Índices:

- `team_invitations_one_live_idx`: único parcial por
  `(organization_id, email) where redeemed_at is null and revoked_at is null`.
  **Un solo link vivo por (organización, email)**; reenviar revoca el anterior
  adentro de la RPC — un link viejo es justo el que pudo haber ido al lugar
  equivocado, no algo que deba seguir sirviendo al lado del nuevo.
- `team_invitations_org_created_idx` `(organization_id, created_at)`: respalda
  los dos `count(*)` del rate limit y el orden del read model.
- `team_invitations_role_idx`.

Trigger `team_invitations_role_same_org` → `ROLE_OTHER_ORGANIZATION`: un rol de
**otra** organización no es un rol de esta invitación. La FK sola no lo impide
(apunta a `organization_roles`, no a "los roles de esta organización") y el
resultado sería un cross-tenant de autorización **acuñado en un token**. Mismo
precedente que `check_member_role_same_org()`.

RLS habilitada **sin una sola policy**, y además
`revoke all on public.team_invitations from anon, authenticated` (más estricto
que `customer_activations`, que se apoya sólo en RLS-sin-policies). Esta tabla
tiene emails y teléfonos de gente que todavía no aceptó nada: un
`grant select` sería un padrón de contactos expuesto. La única puerta son las
RPCs `security definer`.

### `assert_team_seat_available(organization_id, include_live_invitations)`

ADR-0034 resolución 9. `enforce_plan_limit()` es un
`before insert on organization_members`, y con este flujo el `INSERT` ocurre en
el **canje**: sin un chequeo en la emisión, un `OWNER` con un lugar libre emite
cinco invitaciones y cuatro personas se comen `PLAN_LIMIT_REACHED` al hacer
click en un link que ya tenían — un error incomprensible, y del lado equivocado
del mostrador.

El trigger sigue siendo **la regla**; esto es el mensaje temprano. Por eso la
excepción tiene exactamente la forma que levanta `enforce_plan_limit()`
(`PLAN_LIMIT_REACHED: personas en el equipo (n/m)`): si el borde aprendió a
leer una, lee las dos. Mismo reparto que el proyecto ya usa (la base manda, el
borde explica).

- `include_live_invitations = true` → emisión: miembros activos **+**
  invitaciones vivas (no canjeadas, no revocadas, no vencidas).
- `include_live_invitations = false` → la rama de **reactivación** del canje,
  que es un `UPDATE` y por lo tanto **no** pasa por el trigger (es INSERT-only).
  Sin esto, un link viejo reabriría una plaza que el plan ya no tiene — un
  agujero que hoy comparte `invite_member_by_email()` y que acá queda cerrado
  porque el que lo atraviesa es un token portador.

Sin `EXECUTE` para nadie, igual que el helper de acuñado.

### Las cuatro RPCs

| RPC | Gate | Qué hace |
|---|---|---|
| `issue_team_invitation(org, email, display_name, phone, role_id)` → `(invitation_id, token, expires_at)` | **OWNER** | acuña, revoca el link vivo anterior para ese email, devuelve el token claro **una sola vez** |
| `revoke_team_invitation(invitation_id)` | **OWNER** | idempotente; no falla si ya estaba revocada o canjeada |
| `organization_team_invitations(org, include_history)` | **OWNER** | read model; nunca devuelve `token_hash` |
| `claim_team_invitation(token)` → `jsonb` | `authenticated` | el canje |

Las cuatro con `revoke execute from public, anon` antes del
`grant to authenticated` (ADR-0028).

**Quién emite: `OWNER`**, igual que `invite_member_by_email()`. Asimetría
deliberada con ADR-0026 (donde un `STAFF` emite activaciones de cliente): allá
el link otorga "ser vos mismo", acá otorga acceso a los datos de todos. El
argumento de ADR-0026 resolución 2 para gatear a `STAFF` ("gatearlo a `OWNER`
forzaría a compartir su cuenta") no aplica al alta de personal, que es
ocasional y del dueño.

**`issue_team_invitation()` no consulta `auth.users` en ningún momento**, a
propósito (ADR-0034 §5.6). `invite_member_by_email()` levanta
`PROFILE_NOT_FOUND` y con eso convierte al panel en un oráculo de "¿tal email
tiene cuenta en la plataforma?". Este camino no responde esa pregunta — ni
siquiera para decir "ya es miembro": si la persona ya lo es, el canje devuelve
`OK` sin tocarle nada.

Orden de chequeos al emitir: `OWNER` → `organization_can_operate()` →
email (requerido + forma) → nombre → teléfono normalizado → rol vigente de esta
organización → rate limit (**10/h, 30/día** por organización) → **revocar el
link vivo de ese email** → cupo de plan → acuñar → insertar.

El revoke va **antes** del chequeo de cupo a propósito: un reenvío a la misma
persona no puede contar dos plazas, o "no le llegó, mandale de nuevo" sería un
callejón sin salida en el plan más chico.

Números del rate limit: 10/h y 30/día, no los 30/h y 200/día de
`issue_customer_activation()` — un equipo son 2-20 personas, no 200 clientes.
Como en ADR-0026, son propuestas razonadas, no medidas. La fuerza bruta del
**canje** sigue siendo riesgo residual aceptado (256 bits lo vuelven
computacionalmente irrelevante; el rate limiting general es deuda de ADR-0008).

Diferencia deliberada con `create_managed_customer()`: ante un teléfono
impresentable (`"no-es-un-telefono"`), aquélla lo deja en `null` sin decir nada
y ésta levanta `INVALID_PHONE`. Acá el teléfono **es** el canal por el que va a
viajar el link; dejarlo en null en silencio haría que el dueño crea que ya
mandó el WhatsApp.

### `claim_team_invitation(token)` — el canje

ADR-0005 aplicado literalmente, igual que `claim_customer_activation()`: **un
solo parámetro, el token**. No nombra ninguna organización ni ninguna fila de
miembro. El destino sale entero del token; la identidad, entera de
`auth.uid()`. No hay superficie de IDOR porque **no hay input que apunte a una
fila**.

1. `auth.uid() is null` → `AUTH_REQUIRED`.
2. `token_hash = digest(token,'sha256')` **`for update`** → `INVALID_TOKEN`.
3. `revoked_at` → `INVITATION_REVOKED`.
4. `redeemed_at`: si `redeemed_profile_id = auth.uid()` → `OK` (idempotencia
   del doble click); si es otro → `ALREADY_REDEEMED`.
5. `expires_at <= now()` → `INVITATION_EXPIRED`.
6. **El email de la sesión tiene que coincidir** → `INVITE_WRONG_EMAIL`.
   Precedente exacto: `create_organization_with_owner()` ya hace esta
   comparación. ADR-0034 resolución 3, **sin excepción**.
7. Organización activa y `organization_can_operate()` →
   `ORGANIZATION_UNAVAILABLE`.
8. Rol vigente → `INVITATION_ROLE_UNAVAILABLE`. **Falla cerrado**: no se cae al
   rol por defecto ni se adivina. Un rol desactivado suele ser una decisión de
   seguridad reciente; entrar "con otra cosa" sería justo lo que esa decisión
   quería evitar.
9. Si ya es miembro: **activo** → `OK` sin tocar nada; **inactivo** →
   `assert_team_seat_available(..., false)` y reactivar. En los dos casos
   **nunca se le pisa el rol** (ADR-0034 resolución 8 / riesgo 3).
   `invite_member_by_email()` sí lo pisa (`set role = p_role, role_id = ...`) y
   ahí es correcto porque es sincrónico y `OWNER`-gated; en un canje sería un
   camino de escalada — un `STAFF` que consigue una invitación a un rol mayor,
   o peor, un link que **degrada** a alguien que ya trabaja. Un `OWNER` que se
   invita a sí mismo por torpeza sigue siendo `OWNER`.
10. Si no es miembro: `insert` con `role = 'STAFF'`, `role_id` de la invitación
    y **`created_by` = el `created_by` de la invitación** — el alta la decidió
    quien invitó, no quien hizo click. Acá corre `enforce_plan_limit()`, que es
    la regla real.
11. `update ... set redeemed_at = now(), redeemed_profile_id = auth.uid() where
    id = ... and redeemed_at is null`. El `and redeemed_at is null` sobre el
    `for update` del paso 2 da **exactamente un ganador** entre dos canjes
    concurrentes del mismo token, incluso si el lock de fila no los hubiera
    serializado.

### `organization_team_invitations(org, include_history)` — el read model que faltaba

ADR-0034 resolución 10 / riesgo 8. `organization_team()` hace
`join public.profiles p on p.id = om.profile_id`, y una invitación pendiente
**ni siquiera tiene fila de miembro**: sin este read model el dueño no tiene
forma de ver a quién invitó, ni de reenviar, ni de revocar. Es el eco exacto
del hallazgo B de ADR-0026, y por eso entra en el alcance y no en "después".

`status` es **derivado**, no una columna: `REDEEMED` → `REVOKED` → `EXPIRED` →
`PENDING`, en ese orden de precedencia. `role_name` es el rol **EFECTIVO** (el
elegido, o el por defecto de la organización si se invitó sin elegir), igual
que en `organization_team()`.

Con `include_history = false` (default) muestra sólo lo no canjeado y no
revocado — **incluidas las vencidas**, que son justo las que hay que reenviar.

**Gate: `OWNER`, más estricto que el "miembro" que sugería la propuesta
(§4.2).** Motivo: cada fila es el email y el teléfono de alguien que todavía no
aceptó nada, y la emisión, la revocación y la pantalla que lo muestra ya son
`OWNER`-only. Para un no-`OWNER` devuelve **vacío**, no una excepción (igual
que `customer_activation_status()`): una lista sin filas es algo que la UI ya
sabe dibujar. Nunca devuelve `token_hash`, y el token claro no se puede
recuperar — para volver a mandarlo hay que reemitir.

### Errores

`NOT_AUTHORIZED`, `SUBSCRIPTION_INACTIVE`, `EMAIL_REQUIRED`, `INVALID_EMAIL`,
`INVALID_PHONE`, `ROLE_NOT_FOUND`, `ROLE_OTHER_ORGANIZATION`,
`RATE_LIMITED_HOURLY`, `RATE_LIMITED_DAILY`,
`PLAN_LIMIT_REACHED: personas en el equipo (n/m)` (emisión y canje),
`AUTH_REQUIRED`, `INVALID_TOKEN`, `INVITATION_REVOKED`, `ALREADY_REDEEMED`,
`INVITATION_EXPIRED`, `INVITE_WRONG_EMAIL`, `ORGANIZATION_UNAVAILABLE`,
`INVITATION_ROLE_UNAVAILABLE`.

Test: `test/phase33.team-invitations.test.ts` (21 casos: el camino feliz
completo con el `role_id` elegido y los permisos resueltos, TTL de 24 h,
normalización de teléfono, email distinto rechazado **sin consumir el token**,
vencido, un solo uso con idempotencia del doble click, revocado, reenvío que
revoca el anterior, el canje que no le pisa el rol a un miembro existente ni
degrada a un `OWNER`, `OWNER` no fabricable por RPC ni por `service_role`,
cupo de plan en la **emisión** y también en el canje, reenvío que no cuenta dos
plazas, rate limit por organización y no global, `OWNER`-only para emitir /
revocar / listar, los cuatro estados del read model, la tabla invisible por
PostgREST, rol de otra organización, rol desactivado entre emisión y click,
token inexistente / vacío / anónimo, invitación de otro tenant, organización
suspendida). Verificado: `npx supabase db reset` limpio y la suite completa en
verde (**276 tests de integración en 26 archivos** + **60 unitarios** del
paquete de dominio).

## Fase 34 — límites de permiso: dónde va la autorización (migración `20260923210000_phase34_permission_boundary_fixes.sql`)

Cuatro hallazgos de la auditoría de `security-engineer` sobre ADR-0031/0032/0033
(~110 sondas de explotación reales contra la base). Tres eran **bloqueantes de
ADR-0033**; el cuarto era el residuo de `services.price` que la Fase 32 había
dejado anotado como "lo que a propósito quedó como está" y el Orchestrator
decidió cerrar acá. No hay tablas, columnas ni enums nuevos: la migración sólo
mueve chequeos de autorización y agrega un trigger.

### La regla que la migración deja escrita

> **La autorización de una operación va en el punto de entrada público, nunca
> en un helper compartido que también corre como efecto colateral interno de
> otra escritura.**

Ponerlo adentro del helper produce los dos errores opuestos, y la auditoría
encontró uno de cada uno:

| | Síntoma | Caso |
|---|---|---|
| Falso negativo | el helper corre como consecuencia de una escritura legítima y exige un permiso que no le corresponde al actor de *esa* escritura | Fix 1 — el cajero no podía cobrar |
| Falso positivo | otra entrada pública comparte el helper y hereda un chequeo más laxo que el suyo | Fix 2 — `discontinue_schedule_rule()` se quedó en `is_organization_member()` |

### Fix 1 — `MANAGE_PAYMENTS` sin `MANAGE_BOOKINGS` no podía cobrar

La Fase 32 le agregó a `retry_not_generated_booking()` un chequeo de
`MANAGE_BOOKINGS` (correcto para la RPC: reintentar una fecha ajena a mano *es*
operar sobre la reserva de otro). Pero esa función también corre como efecto
colateral interno:

```
insert payments (status = 'PAID')
  -> trigger payments_reconcile_pending            (Fase 12)
    -> reconcile_after_payment()
      -> reconcile_pending_recurring_bookings()
        -> retry_not_generated_booking()           <-- chequeaba MANAGE_BOOKINGS
          -> raise NOT_AUTHORIZED
=> el INSERT del pago entero se abortaba
```

Un rol "Recepción" (`can_manage_payments = true`, `can_manage_bookings = false`
a propósito) registrando el pago de un cliente con fechas
`NOT_GENERATED`/`PAYMENT_REQUIRED` pendientes recibía `NOT_AUTHORIZED` y el pago
no entraba. La misma cascada existía por
`services_reconcile_on_payment_not_required` (Fase 15): apagar
`payment_required` es edición de servicio de cualquier miembro y también moría
en `NOT_AUTHORIZED`.

Se partió en dos:

| Función | Autoriza | Grants |
|---|---|---|
| `internal_retry_not_generated_booking(uuid)` | **nada** — el cuerpo real | `revoke execute ... from public, anon, authenticated, service_role` (mismo patrón que `audit_write()`, Fase 30) |
| `retry_not_generated_booking(uuid)` | cliente dueño de la reserva · miembro con `MANAGE_BOOKINGS` · `auth.uid()` null (job) · y **después** delega | sin cambios: mismo nombre y firma, conserva el `revoke`/`grant` de la Fase 19 |

`reconcile_pending_recurring_bookings()` pasa a llamar al **helper interno**.
Reconciliar es la consecuencia automática del pago, no una acción del cajero
sobre la reserva. El lock `for update` de ADR-0004 sigue viviendo en el helper,
así que la protección de concurrencia no se movió.

Barrido del resto del código de la Fase 32 buscando la misma forma: **no hay
otro caso**. `cancel_booking()` la llama `release_my_booking()`, que es un punto
de entrada público (la cancelación del propio cliente), no un trigger;
`can_customer_book()` la llama `book_slot()`, idem. Las demás funciones
invocadas internamente (`evaluate_payment_coverage`, `generate_recurring_booking`,
`issue_makeup_credit`, `assert_*`…) no tienen chequeos propios por diseño.

### Fix 2 — `discontinue_schedule_rule()` evadía `MANAGE_BOOKINGS`

Seguía gateada en `is_organization_member()` desde la Fase 3, pero cancela en
masa **todas** las `slot_occurrences` futuras de la regla, **todas** las
`bookings` `CONFIRMED` de esas ocurrencias (con `MakeupCredit` cada una) y la
serie recurrente entera. Su prima `cancel_slot_occurrence()`, que hace lo mismo
para **una** fecha, ya exigía `MANAGE_BOOKINGS` desde la Fase 32. Resultado: un
`STAFF` con los cinco permisos en `false` no podía cancelar una clase y sí podía
cancelarle las reservas a toda la organización.

Ahora exige `has_org_permission(organization_id, 'MANAGE_BOOKINGS')`, exactamente
el mismo patrón que `cancel_slot_occurrence()`.

Decisión de producto del Orchestrator, y es la frontera exacta:

- **cancelación en masa** → `MANAGE_BOOKINGS`;
- **edición** de un horario (crear la regla, cambiarle capacidad/hora, sin
  cancelar nada) → cualquier miembro activo, sin gate nuevo.

`discontinue_schedule_rule_group()` **hereda** el gate: llama a la función una
vez por regla y el `raise` aborta la transacción completa, así que no existe
cancelación parcial.

Residuo conocido y aceptado: un `PATCH schedule_rules.is_active = false` directo
por PostgREST sigue siendo de cualquier miembro. Apaga el horario a futuro pero
**no cancela ninguna reserva ni ocurrencia** y no estampa `cancelled_at` — es
edición, no cancelación, del lado correcto de la frontera de arriba.

### Fix 3 — el `OWNER` se auto-reactivaba y se subía de plan

`organizations_update_owner` es `for update using is_organization_owner(id)`, sin
`WITH CHECK` y sin restricción de columnas. Un `PATCH /rest/v1/organizations` del
propio `OWNER` podía escribir `subscription_status`, `plan_code`,
`trial_ends_at` y `current_period_end`: se saltaba la suspensión que un platform
admin le hubiera puesto y se subía de plan sin pasar por
`set_organization_subscription()` (que sí está cerrada a `is_platform_admin()`).
Preexistente a ADR-0033, pero real.

Igual que en `services` (Fase 25), la policy de UPDATE de la tabla **no** puede
pasar a platform-admin-only: el `OWNER` sigue editando nombre, timezone,
branding y política de recupero. El chequeo tiene que ser **por columna**, así
que va en un trigger:

```sql
create trigger organizations_subscription_platform_only
  before update on public.organizations
  for each row
  execute function public.check_organization_subscription_platform_only();
```

`check_organization_subscription_platform_only()` levanta `NOT_AUTHORIZED` cuando
cambia alguna de las cuatro columnas y `auth.uid() is not null and not
is_platform_admin()`. Dos ramas pasan:

- **platform admin real** — `set_organization_subscription()` es `SECURITY
  DEFINER`, pero eso cambia privilegios, **no la sesión**: el trigger la ve con
  el `auth.uid()` del admin, `is_platform_admin()` da `true` y pasa (verificado
  con un platform admin real en el test, no razonado).
- **`auth.uid()` null** — migraciones de datos (el backfill de la Fase 10 hace
  `update organizations set plan_code = 'full'`), jobs internos. No hay actor al
  que autorizar y ese camino no es alcanzable por PostgREST.

Un `PATCH` que escribe el **mismo** valor no es un cambio (`is distinct from` da
`false`) y pasa: no es una escalada.

### Fix 4 — `services.price` sólo lo cambia el `OWNER`

Columna legada pre-ADR-0024: ninguna decisión de cobertura la lee desde la Fase
17 (el precio real vive en `service_plans.price`), pero seguía escribible por
cualquier miembro vía `services_write_staff` mientras el precio del `ServicePlan`
ya estaba cerrado al `OWNER` (`check_service_plan_write_owner`). Es un número que
la organización publica: mismo criterio.

Se **extendió** el trigger que ya existe sobre esa tabla,
`check_service_billing_override_owner()` (Fase 25), en vez de agregar un segundo
`before update` que haría el mismo `raise`. Ahora exige `is_organization_owner()`
para los cuatro overrides de recupero **y** para `price`. El resto de columnas de
`services` (nombre, descripción, `is_active`, `payment_required`…) sigue siendo
de cualquier miembro.

La columna **no se borra**: sacarla es un ADR aparte (hay lecturas históricas y
el backfill de la Fase 17 la usó como origen). Su `comment on column` ahora dice
quién puede escribirla. El gate es sólo en `UPDATE`: crear un servicio con
`price` en el `INSERT` sigue siendo de cualquier miembro, igual que los cuatro
overrides desde la Fase 25.

### Cambios de contrato

Ninguno. Ninguna firma de RPC cambió (la partición del Fix 1 conserva nombre y
firma de la RPC pública a propósito) y ningún shape de respuesta se movió. Lo que
cambia es **quién** puede llamar a `discontinue_schedule_rule()` /
`discontinue_schedule_rule_group()` y quién puede escribir cuatro columnas de
`organizations` y una de `services`. El panel ya sabe esconder/deshabilitar con
`my_organization_permissions()` (Fase 32); `MANAGE_BOOKINGS` es el permiso que
ahora también gobierna el botón "discontinuar horario".

Test: `test/phase34.permission-boundary-fixes.test.ts` (14 casos: el rol
cobra-pero-no-gestiona registrando un pago `PAID` de un cliente con fechas
pendientes y la reconciliación confirmándolas, el mismo rol pasando un `PENDING`
a `PAID`, la cascada de `payment_required` apagado por un rol sin ningún permiso,
la RPC pública que **sigue** exigiendo `MANAGE_BOOKINGS` y el `OWNER` que sí
puede, el helper interno inalcanzable como RPC, `discontinue_schedule_rule()` sin
permiso → `NOT_AUTHORIZED` con regla/ocurrencia/reserva intactas, el grupo que
hereda el gate sin cancelar ninguna de sus dos reglas, el `STAFF` **con**
`MANAGE_BOOKINGS` que sí discontinúa con `RULE_DISCONTINUED`, la edición pura de
un horario que sigue abierta, las cuatro columnas de suscripción rechazadas al
`OWNER` una por una con la suspensión intacta, el `OWNER` que sigue editando
nombre y timezone, un platform admin real cambiando plan y estado por
`set_organization_subscription()`, `services.price` rechazado al `STAFF` con y
sin permisos, y el `OWNER` que sí puede mientras `description` sigue siendo de
cualquier miembro). Verificado: `npx supabase db reset` limpio y la suite de
integración completa en verde (**290 tests en 27 archivos**) + **60 unitarios**
del paquete de dominio. Además se re-corrieron los dos scripts de explotación de
`security-engineer` (`r5.mjs` 6/6, `r6.mjs` 10/10).

## Fase 35 — se cierra el `DELETE` de la Data API (ADR-0036, migración `20260923220000_phase35_close_data_api_delete.sql`)

Hallazgo de `security-engineer`, reproducido contra la base real, en una línea:

> **La cascada de FK no evalúa RLS.** Postgres aplica las policies a la fila que
> el comando nombra; las filas que la integridad referencial arrastra detrás se
> van sin que ninguna policy las mire.

Las dos reproducciones concretas:

| Ataque | Cascada | Daño |
|---|---|---|
| `DELETE /rest/v1/schedule_rules?id=eq.<X>` | `schedule_rules` → `slot_occurrences` → `bookings` | Borra `Booking`s `CONFIRMED`. **No las cancela: las borra** — sin `cancelled_at`/`cancelled_by`, sin `MakeupCredit`, sin fila en `audit_log` |
| `DELETE /rest/v1/services?id=eq.<X>` | `services` → `payments` (`payments.service_id`, Fase 14) | Borra pagos `PAID`. Historial financiero destruido |

Lo hacía un `STAFF` con los cinco permisos de ADR-0033 en `false`: exactamente el
efecto que `MANAGE_BOOKINGS` y `MANAGE_PAYMENTS` existen para negarle. No es un
bug introducido por ADR-0033 — es preexistente desde que estas tablas tienen
policy `ALL using is_organization_member()` (Fases 1/2/3) — pero ADR-0033 lo
vuelve urgente: **un rol restringido no vale nada si hay una puerta lateral que
lo ignora.** Ningún gate de la Fase 32 ni de la Fase 34 lo cubría porque todos
viven en policies de `INSERT`/`UPDATE` o en chequeos dentro de RPCs, y este
camino no pasa por ninguno de los dos.

### El fix: quitar el comando de la policy, no agregar un trigger

Bajo RLS, la **ausencia** de policy de `DELETE` deniega por default. Las siete
policies `ALL` pasan a `INSERT` + `UPDATE` explícitos con la **misma** expresión
que ya tenían:

| Tabla | Policy que se reemplaza | Policies nuevas | Expresión (sin cambios) |
|---|---|---|---|
| `schedule_rules` | `schedule_rules_write_staff` | `schedule_rules_insert_staff`, `schedule_rules_update_staff` | `is_organization_member(organization_id)` |
| `services` | `services_write_staff` | `services_insert_staff`, `services_update_staff` | `is_organization_member(organization_id)` |
| `resources` | `resources_write_staff` | `resources_insert_staff`, `resources_update_staff` | `is_organization_member(organization_id)` |
| `customers` | `customers_write_staff` | `customers_insert_staff`, `customers_update_staff` | `has_org_permission(organization_id, 'MANAGE_CUSTOMERS')` (Fase 32) |
| `schedule_exceptions` | `schedule_exceptions_write_staff` | `schedule_exceptions_insert_staff`, `schedule_exceptions_update_staff` | `is_organization_member(organization_id)` |
| `service_entitlements` | `service_entitlements_write_staff` | `service_entitlements_insert_staff`, `service_entitlements_update_staff` | `is_organization_member(organization_id)` |
| `service_resources` | `service_resources_write_staff` | `service_resources_insert_staff`, `service_resources_update_staff` | `exists (select 1 from services s where s.id = service_id and is_organization_member(s.organization_id))` |

**El `OWNER` tampoco tiene `DELETE` directo.** Si algún día hace falta borrar de
verdad (no cancelar ni desactivar), es una RPC nueva con su propia decisión
explícita, nunca un `DELETE` genérico.

### Por qué `ALL → INSERT + UPDATE` y no `ALL → SELECT + INSERT + UPDATE`

Una policy `FOR ALL` también cubre `SELECT`, así que quitarla podría cerrar
lectura sin querer. No pasa: **las siete tablas ya tienen su policy de lectura
dedicada** (`*_select_*`, Fases 1/2/3), con una expresión idéntica o más amplia
que la de la policy `ALL` que se reemplaza. El caso menos obvio es `customers`:
`has_org_permission(..., 'MANAGE_CUSTOMERS')` exige membresía activa, así que es
un subconjunto estricto de `is_organization_member()`, y la policy de lectura
(`customers_select_self_or_staff`) es `profile_id = auth.uid() or
is_organization_member(...)`. Agregar una policy de `SELECT` duplicada no
cambiaría ni una fila visible; sólo sumaría una expresión más a evaluar por fila
leída.

### Qué NO se toca

- **Ninguna FK `on delete cascade`.** Siguen siendo correctas para cuando la fila
  padre sí se borre por una vía legítima futura. El problema nunca fue la
  cascada: era la puerta que la disparaba.
- **Ninguna RPC.** Verificado antes de escribir la migración, no asumido: en todo
  `backend/supabase/migrations/` el único `delete from` es el de
  `slot_occurrences` dentro de `regenerate_occurrences_for_rule()` (Fase 3) —
  tabla que **no** está en esta lista y cuyo borrado de ocurrencias futuras sin
  `Booking` es el mecanismo mismo de ADR-0003. En `frontend/app/actions/` no hay
  un solo `.delete()` contra PostgREST (el único `.delete()` del repo es
  `jar.delete()` sobre una cookie en `activation.ts`). Las funciones `security
  definer`, además, no evalúan RLS, así que ningún camino interno se ve afectado
  ni siquiera en principio.
- **Los caminos legítimos de baja**, que ya existían y ninguno usa `DELETE`:
  `discontinue_schedule_rule()` (`UPDATE`: corta la regla y cancela las
  ocurrencias futuras liberando cupo), `is_active = false` para
  `Service`/`Resource`, `revoke_member()` para equipo.

### Cómo se ve un `DELETE` denegado por RLS

**No da error.** Postgres no encuentra filas que el actor pueda borrar, PostgREST
responde `204` con `[]` y la fila sigue ahí. El test afirma las dos mitades (la
respuesta no borró nada **y** la fila sobrevive, mirada con el cliente de service
role): afirmar sólo un código de error daría un falso verde si mañana alguien
reabre el `DELETE`.

### Cambios de contrato

Ninguno. Ninguna firma de RPC cambió, ningún shape de respuesta se movió y
ninguna pantalla del panel ofrece hoy un botón "eliminar" sobre estas tablas
(confirmado contra `docs/api.md` antes de aceptar el ADR). El único cambio
observable es que un `DELETE` directo por PostgREST sobre esas siete tablas deja
de borrar.

Test: `test/phase35.data-api-delete-closed.test.ts` (11 casos: los dos escenarios
exactos de `security-engineer` — `DELETE schedule_rules` con una `Booking`
`CONFIRMED` abajo y `DELETE services` con un pago `PAID` abajo, ambos rechazados
con la reserva todavía `CONFIRMED` y `cancelled_at` en `null` y el pago todavía
`PAID`; las siete tablas una por una con el **`OWNER`** como actor, que es el rol
más privilegiado fuera del platform admin; el contrapeso de que `SELECT`,
`INSERT` y `UPDATE` siguen funcionando en las siete; y `discontinue_schedule_rule()`
todavía cancelando y liberando en vez de borrar). El test se validó **al revés**
antes de darlo por bueno: con una migración temporal que restauraba las dos
policies `ALL` originales, 8 de los 11 casos fallan — o sea que el test prueba lo
que dice probar y no es un verde estructural. Verificado: `npx supabase db reset`
limpio, `npm run typecheck` sin errores, **301 tests de integración en 28
archivos** en verde (eran 290 en 27) + **60 unitarios** del paquete de dominio.

## Fase 36 — las vistas públicas dejan de ser escribibles (ADR-0037, migraciones `20260923230000_phase36_public_views_read_only.sql` + `20260923240000_phase36b_role_and_plan_scope_delete_closed.sql`)

> **Por qué ADR-0037 son dos migraciones y no una.** Nació como una sola y se
> partió después de que CI la volteara. CI construye la base **desde cero con lo
> que hay en git**, y las secciones de `organization_roles` y
> `service_plan_services` dependen de la Fase 32, que todavía no está commiteada:
> el `drop policy ... on public.organization_roles` abortaba la migración entera
> con `relation "public.organization_roles" does not exist`. El caso de
> `service_plan_services` era peor porque no fallaba: la tabla sí existe desde la
> Fase 22, pero la policy que hay que reemplazar,
> `service_plan_services_write_owner`, la crea la **Fase 32** (la Fase 22 la había
> llamado `service_plan_services_write_staff`, con `is_organization_member`), así
> que el `drop policy if exists` era un no-op silencioso que dejaba viva la policy
> `FOR ALL` real — con el `DELETE` todavía abierto y el test en verde por el motivo
> equivocado.
>
> Reparto: `..._phase36_public_views_read_only.sql` (el fix de la
> vulnerabilidad crítica: las dos vistas + `organization_members` +
> `audit_public_view_write_grants()`) no depende de nada posterior a la Fase 13 y
> **va sola**. `..._phase36b_role_and_plan_scope_delete_closed.sql` se aplica
> después de la Fase 32 y **se commitea junto con ella**.
>
> Regla general que deja este incidente: una migración sólo puede referenciar
> objetos creados por migraciones **ya commiteadas**. "Anda en mi working tree" no
> es una verificación — la verificación es aplicar desde cero sobre el estado real
> de git.

Hallazgo de `security-engineer` en el barrido que pidió ADR-0036, reproducido
contra la base real con el actor `anon` **sin sesión**. **Vulnerabilidad crítica
preexistente en producción desde la Fase 4 (ADR-0008)**, sin relación con ninguna
ADR del mismo día.

La causa son dos mitades que por separado parecían inofensivas:

> 1. Una vista **sin `security_invoker`** corre con los privilegios de su dueño.
>    Eso es justo lo que el calendario público necesita para leer
>    `organizations`/`services` salteando la RLS de miembro — pero el bypass **no
>    es sólo de lectura**: alcanza los cuatro comandos. Las dos vistas son
>    auto-updatable (un solo `FROM`, sin agregación, sin `DISTINCT`), así que un
>    `INSERT`/`UPDATE`/`DELETE` sobre la vista se reescribe como el mismo comando
>    sobre la tabla base, con los privilegios del dueño y sin evaluar su RLS.
> 2. Los **default privileges de Supabase** en `public` entregan *todos* los
>    privilegios a `anon`/`authenticated` sobre cada tabla y vista nueva. El
>    `grant select` explícito de la Fase 4 era decorativo: `anon` ya tenía
>    `INSERT`, `UPDATE`, `DELETE` y `TRUNCATE` antes de que esa línea corriera.
>    Verificado en `information_schema.role_table_grants`, no deducido.

Lo que lograba un visitante anónimo con la anon key (pública por diseño) y nada
más:

| Ataque | Cascada / gate salteado | Daño |
|---|---|---|
| `DELETE /rest/v1/services_public?id=eq.<X>` (servicio **sin plan**) | `services → schedule_rules → slot_occurrences → bookings` | Borra `Booking`s `CONFIRMED`. **No las cancela: las borra** — sin `cancelled_at`, sin `MakeupCredit`, sin `audit_log` |
| `DELETE /rest/v1/services_public?id=eq.<X>` (servicio cubierto **sólo** por un plan `applies_to_all_services`) | `services → payments` (`payments.service_id`, Fase 14) | Borra pagos `PAID` |
| `PATCH /rest/v1/organizations_public?id=eq.<otro tenant>` | — | Reescribe `slug`/`name`/`timezone` de una organización ajena |
| `POST /rest/v1/organizations_public` | `create_organization_with_owner()`, gate de invitación (ADR-0017), `enforce_plan_limit()` | Crea una `Organization` sin `OWNER` y sin invitación. La ausencia deliberada de policy de `INSERT` en `organizations` (Fase 1) no servía: la vista no la evalúa |
| `PATCH` (libera el slug) + `POST` (lo reclama) | — | **Secuestro del slug**: la URL pública de un negocio real resuelve a la organización falsa del atacante |

### El fix: cerrar la escritura, no tocar la lectura

```sql
revoke insert, update, delete, truncate on public.organizations_public from anon, authenticated;
revoke insert, update, delete, truncate on public.services_public from anon, authenticated;
```

**`security_invoker = on` es el fix equivocado y no se aplicó.** Haría que el
`SELECT` evaluara RLS con los privilegios del llamador, y un visitante anónimo no
es miembro de ninguna organización: el calendario público — todo el punto de
ADR-0008 — devolvería cero filas para todo el mundo. La lectura anónima de estas
dos vistas es legítima e intencional; lo que nunca debió existir es la escritura.

`relacl` antes: `anon=arwdDxtm/postgres`. Después: `anon=rxtm/postgres`.
`REFERENCES`/`TRIGGER`/`MAINTAIN` quedan concedidos y **fuera del alcance de
ADR-0037**: no son alcanzables por la Data API (PostgREST no emite DDL ni comandos
de mantenimiento, y `anon` no tiene `CREATE` en `public`). Anotado para
`security-engineer` en vez de ampliar el ADR sin decisión.

`TRUNCATE` va en el `revoke` aunque no se pueda truncar una vista: el bit de
privilegio existe igual, y el objetivo es que el test genérico de abajo no tenga
excepciones que explicar.

### Las tres policies `FOR ALL` que quedaban

Mismo patrón que ADR-0036 (`ALL → INSERT` + `UPDATE`, **misma** expresión, sin
`DELETE`). Bajo RLS la ausencia de policy de `DELETE` deniega por default.

| Tabla | Migración | Policy que se reemplaza | Policies nuevas | Expresión (sin cambios) | Por qué |
|---|---|---|---|---|---|
| `organization_members` | Fase 36 | `organization_members_write_owner` (Fase 1) | `organization_members_insert_owner`, `organization_members_update_owner` | `is_organization_owner(organization_id)` | **El caso real**: un `OWNER` podía `DELETE` su propia fila y saltear el guard `LAST_OWNER` de `revoke_member()`, dejando la organización sin ningún `OWNER` activo — inadministrable para siempre, porque agregar un `OWNER` es a su vez `OWNER`-only |
| `organization_roles` | Fase 36b | `organization_roles_write_owner` (Fase 32) | `organization_roles_insert_owner`, `organization_roles_update_owner` | `is_organization_owner(organization_id)` | Defensa en profundidad: hoy lo protegían triggers (`ROLE_IN_USE`, `DEFAULT_ROLE_REQUIRED`), no la ausencia de la policy |
| `service_plan_services` | Fase 36b | `service_plan_services_write_owner` (Fase 32) | `service_plan_services_insert_owner`, `service_plan_services_update_owner` | `exists (select 1 from service_plans sp where sp.id = service_plan_id and is_organization_owner(sp.organization_id))` | Defensa en profundidad: `check_service_plan_services_immutable()` sólo cubría los planes con pagos no `VOID` |

La columna "Migración" es la que hace que esto funcione desde cero: las dos de la
Fase 36b tocan policies que **crea la Fase 32**, así que no pueden vivir en la
migración que se commitea sin ella.

La policy original de `organization_members` **no tenía `with check`**, así que
Postgres usaba su `using` también como check de `INSERT`: la policy nueva de
`INSERT` lleva exactamente esa expresión y el reparto de permisos no cambia.

Verificado antes de escribir la migración, no asumido: ningún `delete from` en
`backend/supabase/migrations/` toca estas tres tablas, y el producto nunca borró
ninguna de las tres — el equipo se da de baja con `revoke_member()` (`UPDATE`), un
rol se desactiva con `update_organization_role(p_is_active => false)`, y el
alcance de un plan se fija al crearlo (`create_service_plan(p_service_ids)`) y el
plan se desactiva en vez de rescopearse.

### `audit_public_view_write_grants()` — el guardarraíl

Función nueva, `stable`, `EXECUTE` **sólo** para `service_role` (`revoke all ...
from public` primero: toda función nueva nace con `EXECUTE` para `PUBLIC`).
Devuelve una fila por cada `(vista de public, rol de request, privilegio de
escritura)` que siga concedido. **El resultado correcto es el conjunto vacío.**

Usa `has_table_privilege` y no `information_schema.role_table_grants` a propósito:
resuelve también los privilegios concedidos a `PUBLIC` y los heredados por
pertenencia a otro rol, que no aparecen como fila de `anon` en el catálogo pero
`anon` los tiene igual.

**Regla nueva del repo (ADR-0037)**: toda vista de `public` se cierra con un
`REVOKE` explícito de escritura para `anon`/`authenticated` **en la misma
migración que la crea**. El default de Postgres/Supabase es entregar los cuatro
comandos, y "hicimos `grant select`" no implica que el resto esté cerrado. Esta
función existe para que olvidarse no sea silencioso.

### Dos formas distintas de "denegado"

| Mecanismo | Qué responde PostgREST | Qué afirma el test |
|---|---|---|
| Falta el privilegio de tabla (las dos vistas) | Error `42501` (`insufficient_privilege`) | el código de error **y** que la fila sobrevive |
| Falta la policy de RLS (las tres tablas) | `204` / `[]`, **sin error** | `error === null`, cero filas afectadas **y** que la fila sobrevive |

Conviven en ADR-0037, así que un test que afirmara sólo una de las dos formas
daría un falso verde el día que alguien reabra la puerta por la otra.

### Cambios de contrato

Ninguno. Ninguna firma de RPC cambió y ningún shape de respuesta se movió. El
único uso de las dos vistas en todo `frontend/` es `.select()` en
`frontend/app/actions/public.ts`, y no hay un solo `.delete()` contra PostgREST en
`frontend/`. Se agrega una RPC nueva (`audit_public_view_write_grants()`) que
**no** es parte de ningún contrato de producto: sólo `service_role` la ejecuta.

Tests: `test/phase36.public-views-read-only.test.ts` (**17 casos**, se commitea con
la Fase 36) y `test/phase36b.role-and-plan-scope-delete-closed.test.ts` (**3
casos**, se commitea con la Fase 32 + 36b: cubren `organization_roles`,
`service_plan_services` y el `UPDATE` de `organization_members.role_id`, que es
una columna de la Fase 32). Lo primero que
corre no es un ataque: son los tres casos que afirman que `anon` **sigue** leyendo
`organizations_public`, `services_public` y `get_public_availability()` — si eso se
pone rojo, el fix está mal (es el síntoma exacto de haber puesto
`security_invoker = on`) y hay que volver atrás, no ajustar el test. Después los
seis vectores anónimos, el mismo intento con un usuario autenticado **sin
membresía**, el test genérico sobre `pg_views`, el `OWNER` intentando borrar su
propia fila de `organization_members` (con `revoke_member()` todavía devolviendo
`LAST_OWNER` y todavía funcionando sobre un `STAFF`, con su rastro
`cancelled_at`/`cancellation_reason`), y el contrapeso de que
`SELECT`/`INSERT`/`UPDATE` siguen funcionando en las tres tablas.

El fixture usa **dos servicios distintos** para los dos `DELETE` críticos, y eso
no es decorativo: un servicio cubierto por un plan `PER_SERVICE` sobrevivía al
`DELETE` anónimo **por accidente** (`validate_service_plan_scope` /
`check_service_plan_services_immutable` abortaban la transacción al quedar el plan
sin servicios). Un test que sólo mirara ese caso estaría midiendo la casualidad y
no el agujero, así que hay un servicio **sin ningún plan** (con una `Booking`
`CONFIRMED` abajo) y uno cubierto **sólo** por un plan `applies_to_all_services`
(con un `Payment` `PAID` abajo).

Validado **al revés** antes de darlo por bueno: con una migración temporal que
devolvía los `GRANT` de escritura y las tres policies `FOR ALL`, **16 de los 19
casos fallan** (contados sobre el archivo único, antes del split), y los tres que
pasan son justamente los de lectura anónima, que tienen que pasar en los dos
estados. En ese estado vulnerable los dos `DELETE` anónimos críticos devuelven
`error === null`: borran de verdad.

Un test preexistente cambió: `phase32.configurable-roles.test.ts` >
*"un rol con miembros activos no se desactiva ni se borra"* afirmaba
`ROLE_IN_USE` sobre un `DELETE` directo. Ese `DELETE` ya no llega al trigger — no
hay policy — así que ahora afirma cero filas afectadas, sin error, y que el rol
sigue en la tabla. La intención del caso no cambia; la razón por la que pasa es
más fuerte.

Verificado, en los **dos** estados que importan:

| Estado | `db reset` | `typecheck` | Unit | Integración |
|---|---|---|---|---|
| Sólo lo commiteado + la Fase 36 (lo que arma CI: Fases 30-35 y 36b **fuera** de `supabase/migrations/`) | limpio, 30 migraciones | sin errores | 43 | **231 en 23 archivos** |
| Working tree completo (Fases 30-35 + 36 + 36b) | limpio, 37 migraciones | sin errores | 60 | **321 en 30 archivos** |

La primera fila es la que faltó la primera vez: correr la suite contra el working
tree completo no prueba nada sobre lo que CI va a aplicar.

Además de la dependencia faltante, esa corrida destapó un segundo bloqueante de CI
**preexistente** e independiente (viene de antes del fix de seguridad):
`phase25.production-feedback.test.ts` > *"reporte 1: el motor no es el problema"*
filtraba ocurrencias con techo `lastOfMonth(0)T23:59:59Z`, y la clase de las 21:00
del último día del mes tiene `start_at` a las 00:00 UTC del **1 del mes siguiente**
(ADR-0014) — justo el caso que el test existe para probar. El filtro la excluía y
el test pasaba sólo mientras quedara otra fecha del mes por delante: fallaba según
la hora del día en que corriera la suite. Techo corregido a `firstOfMonth(1)`.

**Nota operativa (de ADR-0037)**: como la vulnerabilidad ya estaba en producción,
corresponde revisar `audit_log` y los conteos de `organizations`/`services` contra
lo esperado una vez aplicado el fix, para descartar que haya sido explotada antes
de encontrarla. La `anon key` es pública por diseño: no hay secreto que rotar.

## Fase 30b — cursor compuesto de `organization_audit_log()` (migración `20260924100000_phase30b_audit_log_composite_cursor.sql`)

Hallazgo de `qa-engineer`. `audit_log.created_at default now()` = hora de **inicio de
la transacción**, así que un `UPDATE` que cancela N reservas (`cancel_slot_occurrence()`,
`discontinue_schedule_rule()`) deja N filas con el mismo `created_at`. El corte
`created_at < p_before` perdía el resto del grupo cuando el límite de página caía
adentro.

- `drop function organization_audit_log(uuid, int, timestamptz)` + `create` con
  `(p_organization_id uuid, p_limit int, p_before_created_at timestamptz, p_before_id uuid)`.
  Drop y no overload: dejar la firma vieja invocable dejaba vivo el camino con el bug.
  Grants re-declarados (`revoke from public, anon`; `grant to authenticated`).
- Corte `(a.created_at, a.id) < (p_before_created_at, p_before_id)`, orden
  `created_at desc, id desc` (misma clave, mismo sentido). Cursor a medias →
  `INVALID_CURSOR`.
- `audit_log_org_time_idx` se recrea como `(organization_id, created_at desc, id desc)`.
- Resto del cuerpo (autorización, enmascarado del actor de plataforma) idéntico a la Fase 30.

**No** se cambió `created_at` a `clock_timestamp()`: distinguiría las filas de una misma
transacción pero no arregla el caso general (dos transacciones distintas también pueden
coincidir al microsegundo). El cursor compuesto es el arreglo correcto; el timestamp
compartido es un dato verdadero ("pasó todo junto").

Test: `backend/test/phase30b.audit-log-cursor.test.ts` (4 casos: precondición de
timestamp compartido entre 7 cancelaciones, paginación con límite 3 sin faltantes ni
duplicados y en orden total, `INVALID_CURSOR`, firma vieja inexistente).

## Fase 27b — slugs reservados (migración `20260924100100_phase27b_reserved_organization_slugs.sql`)

Hallazgo de `security-engineer`. Constraint nuevo, **aparte** del de formato:

```sql
alter table public.organizations
  add constraint organizations_slug_not_reserved_check
  check (slug not in ('_next','activar','admin','api','auth','contacto','dashboard',
                      'equipo','login','me','onboarding','org','signup'));
```

- Lista = árbol top-level real de `frontend/app/` al 2026-09-24 (verificado: `activar`,
  `admin`, `auth`, `contacto`, `dashboard`, `equipo`, `login`, `me`, `onboarding`, `org`,
  `signup`, más `[organizationSlug]`) + `api` + `_next`. `_next` ya lo rechaza el regex
  de formato; se lista para que el `CHECK` sea la referencia completa.
- `CHECK` y no sólo validación en la RPC: no se saltea con ningún camino de escritura
  futuro. Aparte del de formato para que la RPC distinga los dos casos.
- `create_organization_with_owner()`: `create or replace`, misma firma y cuerpo de la
  Fase 27 + `organizations_slug_not_reserved_check` → `SLUG_RESERVED`.
- Validado en el acto (no `NOT VALID`): si una organización existente tuviera uno de
  estos slugs la migración falla — preferible a que capture una ruta en silencio.
- Espejo exacto: `RESERVED_ORGANIZATION_SLUGS` en `backend/src/schemas.ts`.
  **Ruta top-level nueva = migración nueva que la agrega acá.**

Test: `backend/test/phase27b.reserved-organization-slugs.test.ts` (3 casos: cada slug
reservado rechazado por la RPC, `INSERT` directo con `service_role` rechazado por el
`CHECK`, slug que sólo *contiene* una palabra reservada aceptado).

## Fase 37 — `my_payments()`: `LEFT JOIN` + descripción del plan (ADR-0029, migración `20260925100000_phase37_my_payments_plan_scope.sql`)

Pedido: en `/me/pagos` (portal del cliente) el pago sólo mostraba el monto, sin decir
qué compró. Al investigar apareció el mismo bug que ya se había corregido del lado
admin en la Fase 25 (§4, `customer_payment_detail()`), pero nunca replicado acá:
`my_payments()` (Fase 15, `phase15:574`) joineaba `services` por `payments.service_id`
con `INNER JOIN`. Desde ADR-0029 (Fase 22) `payments.service_id` es `NULL` cuando el
pago está anclado a un plan que cubre varios servicios (o todos,
`applies_to_all_services`) — ese `INNER JOIN` descartaba la fila entera. Un pago real,
`PAID`, de un plan multi-servicio simplemente no aparecía en la pantalla de pagos del
propio cliente — sin error visible, sólo una fila que nunca llegaba.

`services` pasa a `LEFT JOIN` y `service_name` cae a `coalesce(s.name, sp.name)`
(mismo criterio que `customer_payment_detail()`). Se agregan cuatro columnas para que
el frontend pueda armar "de qué plan vino y qué da" incluso cuando no hay un único
servicio: `plan_name`, `plan_kind`, `weekly_quota` (entrada de `planSummary()`,
`frontend/lib/plan-labels.ts`) y `plan_applies_to_all_services` (alcance, relevante
cuando `service_name` cae al plan porque no hay un único servicio). `service_plans` se
joinea también por `LEFT JOIN`, defensivo: `payments.service_plan_id` es `NOT NULL`
desde la Fase 7, así que en la práctica nunca pierde filas.

**Cambio de contrato — `returns table` cambia de forma.** `CREATE OR REPLACE` no
alcanza (Postgres no permite cambiar el tipo de retorno de una función existente): la
migración hace `DROP FUNCTION` + `CREATE FUNCTION`, igual que la Fase 15 ya había hecho
con esta misma función. Eso resetea los grants a los default de Postgres (`PUBLIC`
tiene `EXECUTE` por default en funciones) — `revoke ... from public, anon` + `grant
... to authenticated` están en la misma migración, sin depender de que el revoke de la
Fase 19 (que apuntaba a un objeto con OID distinto) siga aplicando.

`service_name` pasa a ser nullable — el único consumidor hoy (`getMyPayments()`,
`frontend/app/actions/customer.ts`) ya lo tipa `string | null`. El JSX de
`/me/pagos` queda **pendiente de `frontend-engineer`**: hoy sólo muestra el monto, no
se tocó esa pantalla en este cambio.

Test: `backend/test/phase37.my-payments-plan-scope.test.ts` — un pago `PAID` anclado a
un plan `UNLIMITED` `applies_to_all_services` (dos servicios cubiertos, sin service
único) confirma que la fila aparece con `plan_name`/`plan_kind`/
`plan_applies_to_all_services` correctos (este es exactamente el caso que el `INNER
JOIN` viejo descartaba en silencio); un pago normal de un plan `WEEKLY_QUOTA` de un
único servicio confirma `service_name`/`plan_name`/`plan_kind`/`weekly_quota`; un
no-cliente sigue recibiendo `[]` (ADR-0006, sin cambios).

**Fase 37b — `currency` (migración `20260925120000_phase37b_my_payments_currency.sql`).**
Mismo día, cambio chico y aditivo: `/me/pagos` necesita formatear cada monto en la
moneda real de la organización que emitió el pago — un mismo cliente puede tener pagos
de varias organizaciones distintas, cada una con su propia `organizations.currency`
(ISO 4217, default `UYU`, ADR-0024), así que no hay forma segura de asumir una moneda
desde el frontend sin este dato viajando junto al monto. `my_payments()` ya hacía
`join public.organizations o`; se agrega `o.currency` al `select` y `currency text` al
final del `returns table` (mismo patrón que `my_services()`, Fase 23,
`20260922210000:26,45`). Contrato aditivo — agrega una columna, no quita ni renombra
ninguna existente.

Otro `DROP FUNCTION` + `CREATE FUNCTION` (el `returns table` cambia de forma otra vez;
`CREATE OR REPLACE` no alcanza) con `revoke`/`grant` repetidos en la misma migración —
mismo motivo que la Fase 37: el objeto queda con OID nuevo, los grants anteriores no le
aplican. No se editó la migración de la Fase 37 (`20260925100000`), ya revisada y
probada — esta es una migración nueva, separada.

Test: se extendieron los dos casos con pago (`plan applies_to_all_services` y `plan`
de servicio único) de `backend/test/phase37.my-payments-plan-scope.test.ts` para
afirmar `currency === "UYU"` (default de la organización de test, sin override).

## Fase 38 — `recurring_booking_occurrences()`: detalle fecha por fecha de un horario fijo activo (migración `20260925110000_phase38_recurring_booking_occurrence_detail.sql`)

Pedido del dueño (captura de pantalla): en la fila de un horario fijo, el badge rojo
"Falta el pago" (de `upcoming_unpaid`) quedaba pegado visualmente a la oración de
`upcoming_beyond_period` ("N caen más adelante que el período que ya pagó..."), la
única que tenía texto propio. Leído en conjunto se lee como contradictorio. Dos
partes: el fix de mensajería simétrica es de frontend puro (ver `docs/api.md` §
`standing-reservations.tsx`); esta migración resuelve la otra mitad — "ver claramente
qué está agendado y qué no", fecha por fecha, no sólo el conteo agregado que ya da
`schedule_rule_standing_reservations()` (Fase 25).

**RPC nueva** (no se toca `schedule_rule_standing_reservations()`, que sigue siendo la
fuente del agregado):

```sql
recurring_booking_occurrences(p_recurring_booking_id uuid)
  returns table (
    slot_occurrence_id uuid,
    start_at timestamptz,
    display_status text,                          -- 'CONFIRMED' | 'UNPAID' | 'OVER_QUOTA'
                                                    -- | 'BEYOND_PERIOD' | 'UNAVAILABLE'
    booking_status public.booking_status,          -- 'CONFIRMED' | 'NOT_GENERATED', real
    not_generated_reason public.not_generated_reason  -- real, null si CONFIRMED
  )
```

`security definer`, `stable`; `revoke ... from public, anon` + `grant ... to
authenticated` (mismo patrón que `schedule_rule_standing_reservations()`).
Autorización: filtra por `public.is_organization_member(rb.organization_id)` en el
`WHERE` — un no-miembro recibe `[]`, no una excepción (mismo criterio que el agregado).

**Por qué no reusar `admin_preview_recurring_booking()`.** Esa RPC (Fase 11/32) evalúa
una serie **prospectiva**: `evaluate_customer_booking(occurrence, customer,
v_existing_series, v_existing_series is null)` — cuando la serie ya existe
(`v_existing_series` no nulo) sigue pasando `p_prospective_series = false`, pero el
punto es que fue diseñada para el caso "todavía no la creé, ¿qué pasaría?" (dry-run
antes de confirmar, ADR-0012). Reusarla contra una serie que **ya está activa**
arriesga evaluarla como si fuera una serie adicional hipotética en vez de la que ya
ocupa su lugar en la cuota del plan (`customer_series_in_force_count()` vs.
`customer_series_quota_position()`, Fase 22) — el mismo tipo de error que ya se vio
antes en esta tabla (Fase 12/17: reescribir sin cuidado la generación pierde el
descuento de cuota). `recurring_booking_occurrences()` en cambio no evalúa nada: lee
el `not_generated_reason` real que `generate_recurring_booking()` /
`reconcile_pending_recurring_bookings()` ya dejaron guardado en `bookings` la última
vez que corrieron.

**`display_status` desdobla `PAYMENT_REQUIRED` en dos**, con la misma comparación de
fechas que el agregado ya usa para separar `upcoming_unpaid` de
`upcoming_beyond_period`:

```sql
case
  when b.status = 'CONFIRMED' then 'CONFIRMED'
  when b.not_generated_reason = 'PAYMENT_REQUIRED' then
    case
      when public.slot_local_date(so.id) <= public.customer_billing_horizon(
             rb.customer_id, sr.service_id, (now() at time zone org.timezone)::date)
      then 'UNPAID'        -- cobrable hoy
      else 'BEYOND_PERIOD' -- no es deuda, todavía no se factura
    end
  when b.not_generated_reason::text = 'OVER_PLAN_QUOTA' then 'OVER_QUOTA'
  else 'UNAVAILABLE'        -- SLOT_FULL / DUPLICATE
end
```

Esto **no** es una evaluación prospectiva: es la misma comparación de fecha contra
`customer_billing_horizon()` (Fase 25) que ya corre dentro de
`schedule_rule_standing_reservations()`, aplicada fila por fila en vez de contada.
Sin este desdoble el `not_generated_reason` crudo no alcanza — `UNPAID` y
`BEYOND_PERIOD` comparten el mismo valor guardado (`PAYMENT_REQUIRED`) y sólo se
distinguen por la fecha de la ocurrencia contra el horizonte de facturación.

`booking_status` y `not_generated_reason` viajan también en crudo (sin traducir) para
quien prefiera leer el dato real en vez de confiar en `display_status`.

**Acción de frontend nueva**: `listStandingReservationOccurrences(organizationSlug,
recurringBookingId)` en `frontend/app/actions/standing.ts` (mapea
`display_status` → `StandingOccurrenceStatus`, mismo vocabulario que
`StandingCreateSummary`). El consumo en UI (detalle expandible) queda para
`frontend-engineer`.

Test: `backend/test/phase38.recurring-booking-occurrence-detail.test.ts` — una serie
real con `CONFIRMED`, `UNPAID` y `BEYOND_PERIOD` simultáneos (un único pago que cubre
un tramo intermedio, sin cubrir hoy, para que `customer_billing_horizon()` caiga en la
rama de fallback) verificando que el detalle suma exactamente igual que
`schedule_rule_standing_reservations()` para esa misma serie; una segunda serie sobre
otro `ScheduleRule` del mismo cliente que excede la cuota del plan (`OVER_QUOTA`,
mismas fechas pagas que la primera, para confirmar que lo que la bloquea es la cuota y
no el pago); un no-miembro de la organización recibe `[]`; `anon` no puede ejecutar la
RPC.

## Fase 39 — ficha de cliente: `customer_standing_reservations()` y `customer_service_plan_quotas()` (migración `20260928120000_phase39_customer_standing_reservations.sql`)

Pedido del dueño (con captura): la ficha de un cliente
(`/org/[slug]/customers/[customerId]`) mostraba "Pagos" y "Créditos de recupero" pero
nada decía qué horarios fijos tiene agendados ni si le faltan para completar la cuota
semanal de su plan. Caso real: cliente con plan "Pilates Reformer 2 x S"
(`weekly_quota=2`) sin forma de ver si tenía 0, 1 o 2 horarios fijos asignados.

### `recurring_booking_upcoming_counts()`: los 5 conteos, extraídos a una función compartida

`schedule_rule_standing_reservations()` (Fase 11/25) ya resolvía correctamente los 5
contadores de "próximas fechas" de una `RecurringBooking` (`upcoming_confirmed` /
`upcoming_not_generated` / `upcoming_unpaid` / `upcoming_over_quota` /
`upcoming_beyond_period`, este último desdoblado de `PAYMENT_REQUIRED` contra
`customer_billing_horizon()`), pero inline dentro de esa función, escaneada por
`ScheduleRule`. Esta fase pedía lo inverso — por cliente, sin importar a qué regla
pertenezca cada serie — así que en vez de repetir las cinco subconsultas se extraen a
un helper interno nuevo:

```sql
recurring_booking_upcoming_counts(
  p_recurring_booking_id uuid,
  p_customer_id uuid,
  p_service_id uuid,
  p_organization_id uuid
) returns table (
  upcoming_confirmed int, upcoming_not_generated int,
  upcoming_unpaid int, upcoming_over_quota int, upcoming_beyond_period int
)
```

`security definer`, `stable`, `revoke ... from public, anon, authenticated,
service_role` (Fase 19: helper interno, sólo lo llaman otras funciones
`security definer` de este archivo, nunca invocable directo por PostgREST).
`schedule_rule_standing_reservations()` se reescribe para llamarlo vía
`cross join lateral` en vez de repetir las subconsultas — mismo `returns table`
exacto, así que `CREATE OR REPLACE` alcanza y los grants de la Fase 25 siguen
vigentes (el OID no cambia). El test de la Fase 25/38 (que compara este agregado
contra el detalle fecha por fecha de `recurring_booking_occurrences()`) es la prueba
de que el refactor no cambió el resultado — sigue pasando tal cual, sin tocarlo.

### `customer_standing_reservations(p_customer_id uuid)`: el inverso de `schedule_rule_standing_reservations()`

Todas las `RecurringBooking` `status = 'ACTIVE'` de un cliente puntual, con el
`ScheduleRule`/servicio de cada una y los mismos 5 conteos de arriba:

```sql
returns table (
  recurring_booking_id uuid, schedule_rule_id uuid, service_id uuid, service_name text,
  weekday smallint, local_start_time time, duration_minutes int,
  status recurring_booking_status, created_at timestamptz,
  upcoming_confirmed int, upcoming_not_generated int,
  upcoming_unpaid int, upcoming_over_quota int, upcoming_beyond_period int
)
```

`security definer`, `stable`; `revoke ... from public, anon` + `grant ... to
authenticated` (mismo patrón que `schedule_rule_standing_reservations()`).
Autorización: `where ... and public.is_organization_member(rb.organization_id)` — un
no-miembro de la organización del cliente recibe `[]`, no una excepción (mismo
criterio que el resto de esta familia). Filtra a `status = 'ACTIVE'` a propósito
(distinto de `schedule_rule_standing_reservations()`, que muestra también las
`CANCELLED` de esa regla): la ficha pregunta "qué tiene agendado hoy", no el
histórico de series de ese cliente.

### `customer_service_plan_quotas(p_customer_id uuid)`: cuánto le corresponde, por servicio

La segunda mitad del pedido — no sólo qué tiene agendado, sino cuánto le da derecho
su plan. Una fila por servicio donde el cliente tiene al menos un horario fijo
`ACTIVE`:

```sql
returns table (
  service_id uuid, service_name text,
  service_plan_id uuid, plan_name text, plan_kind service_plan_kind,
  weekly_quota int, quota_scope plan_quota_scope, assigned_count int
)
```

**No reinventa "cuál es el plan vigente de este cliente para este servicio"**: llama
a `resolve_covering_service_plan(customer_id, service_id, hoy)` (ADR-0029, Fase 22),
la misma función que usa `evaluate_payment_coverage()` en el camino de reserva — un
`Payment PAID` cuyo período cubre la fecha local de hoy. `assigned_count` tampoco
reinventa "cuántas series ya ocupan esa cuota": es `customer_series_in_force_count()`
con el mismo conjunto de servicios que resuelve `service_plan_quota_service_ids()` —
bajo un plan `SHARED_ACROSS_SERVICES` (ADR-0029) cuenta el pool compartido entre los
servicios que el plan cubre, no sólo el servicio de esa fila, aunque la respuesta
siga viniendo una fila por servicio (pedido explícito: "cuota por servicio, no una
sola cuota global" — un cliente puede tener horarios fijos en varios servicios con
planes de cuota distintos).

`weekly_quota`/`plan_kind`/`quota_scope`/`service_plan_id` salen `null` cuando: (a) el
servicio no tiene hoy ningún `Payment PAID` vigente para este cliente (plan lapsado o
nunca pagado), o (b) el plan vigente es `UNLIMITED`/`DROP_IN` — ninguno de los dos
tiene un tope de series que mostrar. `assigned_count` viaja **siempre**, con o sin
plan vigente: son las `RecurringBooking ACTIVE` de este cliente en vigencia hoy para
ese servicio (o el pool completo bajo `SHARED_ACROSS_SERVICES`), el mismo "N" que la
ficha necesita mostrar aunque hoy no haya "M" contra qué compararlo.

`language plpgsql` (no `sql`, a diferencia de las otras funciones de esta familia):
necesita ramificar por fila entre "hay plan vigente" / "no hay" / "el plan tiene
tope" antes de resolver `assigned_count`, mismo estilo `return next` por fila que
`admin_preview_recurring_booking()` (Fase 11). `security definer`, `stable`;
`revoke ... from public, anon` + `grant ... to authenticated`. Autorización explícita
al principio de la función (`is_organization_member` sobre la organización del
cliente resuelto por `p_customer_id`, no por un `organization_id` que pase el
caller) — un cliente inexistente o de una organización ajena hace `return;` sin
filas, mismo criterio que el resto.

Test: `backend/test/phase39.customer-standing-reservations.test.ts` — un cliente con
horarios fijos en 3 servicios (`WEEKLY_QUOTA` de cuotas 2/1/2, pagos vigentes hoy en
los tres) donde dos están completos y uno tiene menos series que su cuota (`1 de 2`,
para que el frontend pueda calcular "le falta 1"), verificando que
`customer_standing_reservations()` trae las 4 series con su servicio correcto y que
su agregado coincide con el que ya da `schedule_rule_standing_reservations()` para la
misma serie; un servicio con plan de cuota creado pero nunca pagado
(`weekly_quota`/`plan_kind` `null`, `assigned_count` viaja igual); un no-miembro de la
organización recibe `[]` en ambas RPC; `anon` no puede ejecutar ninguna de las dos.

## Fase 40 — nonce de continuación de activación (ADR-0040, migración `20260928130000_phase40_activation_continuation.sql`)

Problema (reporte real de producción, diagnosticado en vivo, no solo por lectura de
código — ver ADR-0040): la cookie httpOnly `activation_token` (ADR-0026 Sec 2.4) la
escribe `/activar/[token]` en el contexto de navegador donde la persona tocó el link
de WhatsApp. Confirmar el email o terminar el login de Google puede aterrizarla en
OTRO contexto (Mail, el navegador del sistema — Google bloquea OAuth dentro de
WebViews embebidos desde 2021), con un cookie jar distinto: `/activar/continuar` en
ese segundo contexto nunca encuentra la cookie, aunque el token siga vigente por sus
72h completas.

### `customer_activation_continuations`

```sql
create table public.customer_activation_continuations (
  id uuid primary key default gen_random_uuid(),
  activation_id uuid not null references public.customer_activations (id) on delete cascade,
  activation_token_enc bytea,      -- nullable a propósito, ver más abajo
  nonce_hash bytea not null unique,
  expires_at timestamptz not null,
  created_at timestamptz not null default now(),
  used_at timestamptz,
  constraint customer_activation_continuations_token_matches_used check (
    (used_at is null and activation_token_enc is not null)
    or (used_at is not null and activation_token_enc is null)
  )
);
```

RLS habilitada, **cero policies y cero grants** a `anon`/`authenticated`/`service_role`
— mismo patrón que `customer_activations` (Fase 21): la única puerta son las dos RPC
`security definer` de abajo. Además, al final de la migración: `revoke all on table
customer_activation_continuations from anon, authenticated` explícito — defensa en
profundidad redundante con la RLS sin policies, agregada a pedido del gate de
seguridad (no bloqueante, pero de costo casi nulo).

**Corrección post-review de seguridad (2026-09-28, ver ADR-0040 "Corrección
post-review de seguridad"): la primera versión de esta tabla guardaba el token real en
claro (`activation_token text`)**, bajo la teoría de que el TTL de 30 min acotaba la
exposición igual que `customer_activations` acota su propio token. Esa teoría se
rompía en un camino real: si el nonce nunca se redimía (ej. la persona vuelve a
loguearse con contraseña en el MISMO contexto de navegador, que nunca pasa por
`/auth/callback`), nada ponía la columna en `null` — la fila (y el token en claro
adentro) quedaba ahí indefinidamente, y el propio `check` constraint (que exige token
no-nulo mientras `used_at` sea nulo) hacía imposible limpiar la columna sin borrar la
fila entera.

**Fix, verificado por `security-engineer` contra la base local antes de aceptarse**:
cifrar en reposo con el nonce mismo como clave simétrica (`pgcrypto`, ya instalado
desde la Fase 21 — `pgp_sym_encrypt`/`pgp_sym_decrypt`, AES-256) en vez del nonce en
claro. El nonce en claro nunca se persiste en ningún lado (`issue_activation_
continuation()` sólo lo tiene en una variable local que devuelve al caller;
`nonce_hash` es su sha256, igual que antes) — así que ninguna fila, por sí sola,
reconstruye el secreto, misma garantía que ya daba el hash de `customer_activations.
token_hash`. Se completa con dos medidas adicionales, ambas dentro de
`issue_activation_continuation()`:

- **Purga de vencidos sin usar** para esa `activation_id`, antes de contar o insertar
  — cierra exactamente el mismo agujero que motivó el cifrado (un nonce abandonado ya
  no deja un blob cifrado sentado para siempre, se borra en el próximo `issue`).
- **Límite de 10 nonces vivos por activación** (`TOO_MANY_CONTINUATIONS` si ya hay 10
  sin usar) — sin este límite, nada impedía acumular nonces indefinidamente para la
  misma activación.

Ambas corren bajo el mismo `select ... for update` sobre la fila de
`customer_activations` que ya validaba vigencia (revocada/reclamada/vencida), para que
el conteo sea consistente bajo llamadas concurrentes.

Exposición acotada igual que antes: TTL de 30 min, un solo uso, y la columna
`activation_token_enc` se pone en `null` en el mismo `update` que marca `used_at` — no
queda un secreto cifrado vivo sentado en una fila ya usada, y ahora tampoco en una fila
vencida sin usar (la purga se encarga). El `check` constraint sigue haciendo que las
dos columnas nunca puedan discrepar (mismo estilo que
`customers_claimed_requires_profile`, Fase 21).

### `issue_activation_continuation(p_token text) returns text`

Recibe el token real (leído server-side de la cookie, nunca expuesto al cliente),
hace `select ... for update` sobre la `customer_activations` que resuelve el hash del
token, y valida que esté vigente y no reclamada — mismo vocabulario de error que ya
usa `claim_customer_activation()`: `INVALID_TOKEN` (no existe), `ACTIVATION_REVOKED`,
`ALREADY_REDEEMED`, `ACTIVATION_EXPIRED`. Bajo ese mismo lock: borra las continuaciones
vencidas sin usar de esa activación, cuenta las que quedan sin usar y rechaza con
`TOO_MANY_CONTINUATIONS` si ya hay 10. Si pasa todo lo anterior, acuña un nonce nuevo
con `new_activation_token()` (el mismo helper compartido de la Fase 34, sin grants a
nadie salvo estas RPC `security definer`), inserta la fila con
`activation_token_enc = pgp_sym_encrypt(token, nonce, 'cipher-algo=aes256')` y
`expires_at = now() + interval '30 minutes'`, y devuelve el nonce **en claro, sólo
esta vez** (igual que `issue_customer_activation()` con el token real).

`security definer`, `set search_path = public`. **`grant execute ... to anon,
authenticated`** — a propósito: esta RPC corre desde `/activar/continuar` exactamente
en la rama donde **no hay sesión de Supabase todavía** (es la razón de ser de esta
fase), así que el caller real llega con la anon key. No hay ninguna rama por
`auth.uid()` — poseer el token de 256 bits es la autorización, mismo modelo de amenaza
que `claim_customer_activation()` documenta para el token original (ADR-0026 Sec 2.2).

### `redeem_activation_continuation(p_nonce text) returns text`

Recibe el nonce, hace `select ... for update` por `nonce_hash`, y en una sola
transacción: si la fila no existe, o `used_at is not null`, o `expires_at <= now()`,
levanta `INVALID_CONTINUATION` — **un solo error genérico para esas tres razones a
propósito** (a diferencia de `claim_customer_activation()`, que sí puede permitirse
ser específica porque llegar hasta ahí ya exigió poseer el token real). Además,
verifica que la `customer_activations` referenciada siga viva (no revocada, no
reclamada, no vencida) — si no lo está, levanta el mismo `INVALID_CONTINUATION`
genérico, sin distinguir cuál de los casos aplica (agregado post-review: el nonce
puede ser válido aunque la activación haya cambiado de estado después de emitirlo). Si
todo es válido, descifra con `pgp_sym_decrypt(activation_token_enc, nonce)` (el nonce
recién validado hace de clave), hace
`update ... set used_at = now(), activation_token_enc = null where id = ... and
used_at is null`, y devuelve el token real descifrado. El `for update` ya serializa
dos redenciones concurrentes del mismo nonce (la segunda transacción bloquea, y al
desbloquear relee la fila ya marcada `used_at`); el `and used_at is null` del `update`
es defensa en profundidad encima de eso, no la única guardia.

`security definer`, `set search_path = public`. `grant execute ... to anon,
authenticated` — corre desde `/auth/callback` justo después de
`exchangeCodeForSession()`, en el contexto de navegador que recién terminó de
autenticarse; no depende de `auth.uid()` por el mismo motivo que la RPC anterior, así
que exigir sesión acá sólo agregaría un modo de falla si la sesión recién intercambiada
todavía no se propagó al cliente de Supabase de ese request exacto.

**Redimir no activa nada por sí solo.** Sólo le devuelve al caller el token real para
que lo replante como cookie en el contexto nuevo; el resto del flujo (cookie + sesión
→ `ConfirmActivationForm` → clic explícito → `claim_customer_activation()`) sigue
exactamente igual, sin tocarse.

**Corrección post-review de seguridad, importante**: a diferencia de lo que decía la
versión original de esta sección, `claim_customer_activation()` (Fase 21) **no**
exige que el email de la sesión coincida con el cliente invitado — sólo exige
`auth.uid() is not null` (los clientes gestionados ni siquiera tienen columna de
email para comparar). Esto significa que quien posea el nonce (o el token
subyacente) puede reclamar la activación con **cualquier** cuenta autenticada, no
sólo con la del destinatario real. Se acepta este diseño explícitamente en ADR-0040:
el nonce equivale de hecho al token completo durante su vida útil, mismo modelo de
amenaza ya aceptado para el token original (ADR-0026) — no es una regresión
introducida por esta fase, es el comportamiento preexistente de
`claim_customer_activation()`, ahora ejercitado también a través del nonce.

Test: `backend/test/phase40.activation-continuation.test.ts` (9 casos) — un nonce
válido (emitido con el cliente `anon`, reproduciendo la falta de sesión real del
flujo) se redime una sola vez y un segundo intento inmediato falla con
`INVALID_CONTINUATION`; un nonce vencido (backdateado directamente en la fila vía
`admin`, ubicada por `activation_id`) falla igual; un nonce que nunca existió falla
con el mismo error genérico; `redeem_activation_continuation()` devuelve el token
real exacto, verificado comparándolo contra el token que emitió
`issue_customer_activation()`; emitir una continuación para un token inválido o para
una activación revocada reusa el vocabulario de error existente (`INVALID_TOKEN`,
`ACTIVATION_REVOKED`); redimir un nonce y reclamar con una cuenta sin ninguna relación
con el cliente invitado funciona (documentado en el test como comportamiento
intencional, no un bug); la columna `activation_token_enc`, leída cruda vía `admin`
justo después de emitir (antes de redimir), nunca contiene el token en claro bajo
ninguna decodificación; pedir un onceavo nonce vivo para la misma activación falla
con `TOO_MANY_CONTINUATIONS`; redimir un nonce cuya activación fue revocada después de
emitido falla con `INVALID_CONTINUATION`.

**Alcance de esta fase**: sólo la activación de clientes (ADR-0026). ADR-0040 señala
que `team_invitations` (ADR-0034, `/equipo/[token]`) tiene la misma cookie httpOnly y
podría sufrir el mismo síntoma — no se toca acá; sería una extensión directa del mismo
mecanismo si se confirma, a evaluar por separado.

**Estado**: gate de `security-engineer` corrió dos veces (2026-09-28) — la primera
devolvió NO LISTO (token en claro sin límite de vida real, sin límite de nonces vivos,
sin verificación de vigencia de la activación en `redeem`); los tres hallazgos están
incorporados arriba y verificados con la suite de integración completa (`npx supabase
db reset` + `npm run test:integration`, 9/9 tests de esta fase en verde). Falta que
`security-engineer` vuelva a correr el gate sobre esta versión corregida antes de
desplegar.

## Fase 41 — corrección post-review de seguridad de ADR-0043 (migración `20260929140000_phase41_adr0043_post_review_hardening.sql`)

Implementa los puntos 1 y 2 de la sección "Corrección post-review de seguridad" de
ADR-0043 (`docs/decisions.md`): el gate de `security-engineer` sobre esa ADR encontró
tres caminos reales hacia acceso de `OWNER` en un negocio ajeno, habilitados por
`enable_confirmations = false` (ADR-0043 base). Esta fase cierra los caminos 1 y 2. El
camino 3 (usuarios viejos sin confirmar) es una auditoría manual, documentada en la ADR
para copiar/pegar antes de apagar el toggle en producción — no es una migración.

### 1. `EXECUTE` revocado de `invite_member_by_email()` y `enroll_customer_by_email()`

```sql
revoke execute on function public.invite_member_by_email(
  uuid, text, public.organization_member_role, uuid
) from anon, authenticated, service_role;

revoke execute on function public.enroll_customer_by_email(uuid, text)
  from anon, authenticated, service_role;
```

Las dos funciones **no se borran** — siguen definidas en `20260914184943_phase8_admin_operations.sql` /
`20260923230300_phase32_configurable_roles.sql`, sin cambios de cuerpo — sólo pierden
todo camino de invocación. Investigado antes de decidir el alcance del revoke (incluir
`service_role`, no sólo `anon`/`authenticated`): ningún caller de este repo (tests,
`frontend/app/actions/admin.ts`) las invoca con el cliente `service_role`; todos usan la
sesión del usuario autenticado. Y de hecho `service_role` **ya no tenía** `EXECUTE`
sobre ninguna de las dos desde la Fase 19 (`20260922160000_phase19_security_fixes.sql:526-527`
revocó el grant heredado del default de Supabase para `public`/`anon`, y ningún
`create or replace`/`drop + create` posterior de las dos, en la Fase 32, le devolvió
nada a `service_role`) — el `revoke` explícito de esta fase documenta ese estado y evita
que una migración futura se lo devuelva por accidente (mismo patrón que
`retry_not_generated_booking()`, Fase 19).

Reemplazos ya existentes, sin depender de "el email prueba identidad":
`invite_member_by_email()` → invitación de equipo por token (ADR-0034,
`claim_team_invitation()`, `/equipo/[token]`); `enroll_customer_by_email()` →
activación de cliente gestionado por link de WhatsApp (ADR-0026,
`issue_customer_activation()` / `claim_customer_activation()`).

**Cambio de contrato, coordinado con `frontend-engineer`**: `frontend/app/actions/admin.ts`
(`enrollCustomer()`, `inviteMember()`) llama a las dos RPC con la sesión del usuario —
después de esta migración ambas devuelven `42501 permission denied` en vez del
comportamiento anterior. La UI que las invoca se saca o se redirige al flujo por token
(trabajo de `frontend-engineer`, no cubierto por esta fase).

Los ~8 tests de integración que antes ejercitaban estas dos RPC como fixture (crear un
`Customer`/`STAFF` de prueba) pasaron a escribir directo en `customers`/
`organization_members` con el cliente del `OWNER` (misma política
`organization_members_write_owner` que ya usaba `test/phase32.configurable-roles.test.ts`)
o con `admin` (`service_role`); los que probaban el comportamiento propio de las RPC
(reactivación por email, chequeo `NOT_AUTHORIZED`/`PROFILE_NOT_FOUND` interno, soporte de
3 vs. 4 argumentos) se reemplazaron por una aserción de `error.code === '42501'`, porque
esa lógica interna quedó inalcanzable para cualquier caller.

### 2. Trigger sobre `auth.identities` contra el secuestro vía Google OAuth

El problema (verificado en vivo por `security-engineer`, ver ADR-0043): GoTrue sólo
purga identidades sin confirmar al vincular una identidad OAuth nueva a un email ya
registrado cuando la identidad existente está sin confirmar. Con
`enable_confirmations = false` toda cuenta por contraseña queda confirmada al crearse,
así que esa purga nunca se dispara — GoTrue en cambio hace su vinculación normal
"por email ya verificado", atando la identidad Google nueva al mismo `user_id` que ya
tenía la contraseña. Si esa cuenta la creó un atacante ocupando el Gmail de una futura
dueña, la dueña real termina autenticada, vía "Continuar con Google", dentro de la
cuenta del atacante.

```sql
create or replace function public.check_identity_link_not_oauth_hijack()
returns trigger
language plpgsql
set search_path = ''
as $$
begin
  if new.provider <> 'email' and exists (
    select 1 from auth.identities
    where user_id = new.user_id and provider = 'email'
  ) then
    raise exception 'OAUTH_LINK_BLOCKED_EXISTING_PASSWORD_IDENTITY' using errcode = 'P0001', ...;
  end if;
  return new;
end;
$$;

create trigger auth_identities_block_oauth_hijack
  before insert on auth.identities
  for each row execute function public.check_identity_link_not_oauth_hijack();
```

Bloquea **todo** `insert` de una identidad no-`email` sobre un `user_id` que ya tiene una
identidad `email`, sin distinguir el origen de la cuenta (signup público vs.
invitación/activación administrada): este proyecto tiene `enable_manual_linking = false`
(`supabase/config.toml`, `[auth]`) — sin manual linking no existe ningún flujo soportado
por el que una persona ya autenticada por contraseña agregue Google a su propia cuenta de
forma deliberada, así que el único origen posible de este `insert` es el camino
automático y vulnerable de "vincular por email ya verificado" durante un
`signInWithOAuth()` sin sesión previa. No hace falta (ni sería confiable) distinguir el
origen de la cuenta, porque ninguna la prueba posesión de email al crearse.

Efecto: se bloquea el `insert` (no se borra la identidad vieja ni se toca `auth.users`) —
fail-closed deliberado, más seguro que replicar el comportamiento viejo de GoTrue
(purgar y dejar crear cuenta nueva), que exigiría mutar `auth.users`/`auth.identities` a
mano en medio de la propia transacción de GoTrue. El soporte resuelve a mano vía la
auditoría del punto 3 de ADR-0043.

Sin `comment on trigger ... on auth.identities`: exige ser dueño de la relación
(`must be owner of relation identities`, confirmado en vivo — la dueña es
`supabase_auth_admin`, el rol de migración `postgres` puede crear el trigger pero no es
dueño de la tabla). La documentación vive en el comentario de la función.

**Seguro triggerear sobre `auth.identities`**: mismo patrón que ya usa este repo desde la
Fase 1 (`public.handle_new_user()`, `after insert on auth.users`) — ambas tablas viven en
el mismo schema `auth` gestionado por Supabase, mismo dueño (`supabase_auth_admin`) y
mismo rol de migración (`postgres`) con privilegio suficiente. Confirmado en vivo contra
Supabase local (CLI 2.118.0): `npx supabase db reset` aplica la migración limpio, y un
insert directo de una identidad `google` para un `user_id` con identidad `email`
preexistente falla con `OAUTH_LINK_BLOCKED_EXISTING_PASSWORD_IDENTITY`; el mismo insert
para un `user_id` recién creado (sin identidad `email` previa, el caso normal de "Google
por primera vez") pasa sin error.

### 3. Auditoría manual antes de producción (no es una migración)

Documentada en `docs/decisions.md` ADR-0043, sección "Corrección post-review de
seguridad" — lista para copiar/pegar en el SQL Editor del dashboard de producción antes
de apagar `enable_confirmations`.

### 5. Turnstile (`[auth.captcha]`, `supabase/config.toml`)

Habilitado con `provider = "turnstile"` y el secreto de prueba público de Cloudflare
("always passes", documentado y pensado para automatizar tests sin navegador —
`https://developers.cloudflare.com/turnstile/troubleshooting/testing/`, no es un secreto
real). **Producción necesita reemplazarlo** por un Site Key + Secret Key reales,
generados gratis en <https://dash.cloudflare.com/?to=/:account/turnstile>, cargando el
Secret Key en el dashboard de Supabase (Authentication → Attack Protection o la sección
equivalente según la versión) — mismo patrón que este repo ya usa para toggles que sólo
se pueden aplicar a mano en el proyecto administrado (ADR-0017, ADR-0041, ADR-0043 base).

Confirmado en vivo que GoTrue exige `captcha_token` (`gotrue_meta_security.captcha_token`)
tanto en `/signup` como en `/token?grant_type=password` — **no sólo en signup**. Con esto
habilitado, `frontend/app/actions/auth.ts` (`signUp()`, `signInWithPassword()`) necesita
mandar un token de Turnstile real (widget de Cloudflare en el formulario) o toda cuenta
nueva y todo login por contraseña empieza a fallar con `captcha_failed` — **trabajo de
`frontend-engineer`, no cubierto por esta fase**. `backend/test/helpers.ts`
(`createSignedInUser()`) ya pasa un `captchaToken` fijo en `signInWithPassword()` porque
la clave de prueba acepta cualquier token no vacío; eso alcanza para los tests de
integración pero no reemplaza el widget real que necesita producción.

## Fase 42 — recursos exclusivos: anti-solapamiento a nivel de base de datos (ADR-0044, migración `20260930100000_phase42_exclusive_resources.sql`)

Cierra el gap encontrado en la auditoría de generalización (ver `docs/decisions.md`,
sección posterior a ADR-0043): nada, a ningún nivel, impedía que el mismo `Resource`
(ej. un profesional) quedara reservado dos veces a la misma hora en servicios distintos.

- `resources.is_exclusive boolean not null default false` — "se ocupa de a uno, no admite
  turnos superpuestos". Genérico, sin ningún término de rubro.
- `slot_occurrences.resource_is_exclusive boolean not null default false`, denormalizado
  desde `resources.is_exclusive` porque un `EXCLUDE` no puede mirar otra tabla:
  - trigger `slot_occurrences_set_resource_is_exclusive` (`before insert or update of
    resource_id`) lo copia al crear/mover una ocurrencia;
  - trigger `resources_propagate_is_exclusive` (`after update of is_exclusive on
    resources`) lo propaga sólo a ocurrencias futuras `status='ACTIVE'`
    (`start_at >= now()`) — el pasado nunca se toca.
- Constraint `slot_occurrences_exclusive_resource_no_overlap`:
  `exclude using gist (resource_id with =, tstzrange(start_at, end_at, '[)') with &&)
  where (status = 'ACTIVE' and resource_is_exclusive)` (`btree_gist`, ya instalado desde
  Fase 7). Con `'[)'`, dos turnos pegados (10:00–10:30 y 10:30–11:00) no chocan.
- `generate_slot_occurrences_for_rule()` (recreada desde la versión vigente de Fase 19)
  envuelve el insert en `begin...exception when exclusion_violation then
  v_new_occurrence_id := null; end;` — salta sólo la fecha en conflicto, nunca aborta el
  resto de la regla ni de la organización. Crítico: sin esto, una colisión en un tenant
  frenaría `generate_all_slot_occurrences()` (el cron diario) para todos.
- `check_schedule_rule_conflicts(p_resource_id, p_weekday[], p_local_start_time[],
  p_duration_minutes, p_exclude_rule_id default null)` — proyecta 90 días y devuelve los
  choques contra ocurrencias `ACTIVE` del mismo recurso exclusivo. `create_schedule_rule_group()`
  (recreada) la llama **antes** de insertar cualquier `ScheduleRule`, `raise exception
  'RESOURCE_SCHEDULE_CONFLICT: ...'` si hay choque — atómico, nada queda a medias.
- Activar `is_exclusive=true` sobre un recurso con solapamientos existentes: el propio
  constraint rechaza el `UPDATE` (`exclusion_violation`, SQLSTATE `23P01`) — queda como
  error crudo de Postgres a propósito; mapearlo a `RESOURCE_HAS_OVERLAPS` es tarea de
  `frontend-engineer` (ver ADR-0044 "Impacto" en `docs/decisions.md`).

**Tests** (`backend/test/phase42.exclusive-resources.test.ts`, 5 casos): solapamiento
bloqueado / turnos pegados sin bloquear; el generador salta sólo la fecha en conflicto sin
abortar el resto; `check_schedule_rule_conflicts` rechaza antes de insertar; activar
`is_exclusive` con solapamientos falla y revierte; un recurso NO exclusivo sigue
permitiendo solapamiento libre (comportamiento del gimnasio, sin cambios). **Verificado en
vivo contra `reservaste-stg`**: 366/366 tests de la suite completa en verde, sin
regresiones. No requirió gate de `security-engineer` (no toca auth/RLS/multi-tenant,
hereda la RLS de `resources`) — sí pasó `reviewer`, LISTO.

## Fase 43 — generador de horarios por franja horaria (ADR-0045, migración `20260930120000_phase43_schedule_rule_spans.sql`)

Extiende `create_schedule_rule_group()` (ADR-0022, producto cartesiano `weekdays[] x UN
local_start_time`) al producto cartesiano `weekdays[] x [range_start, range_end)` expandido
por `step_minutes` — sin cambiar el modelo de `ScheduleRule` (sigue siendo una fila por
día+hora, nunca un rango).

- **`create_schedule_rules_batch(p_organization_id, p_service_id, p_resource_id,
  p_weekdays int[], p_local_start_times time[], p_duration_minutes, p_capacity) returns
  setof schedule_rules`** — rutina interna nueva, **no expuesta como RPC** (`revoke execute
  ... from public, anon, authenticated`, sin `grant`). Llama `check_schedule_rule_conflicts()`
  (ADR-0044) **una sola vez** para toda la grilla expandida antes de insertar cualquier fila
  — atómico: si hay choque, no se inserta nada. Único punto compartido de
  expansión+chequeo+inserción entre los dos RPC públicos de abajo.
- **`create_schedule_rule_group()` recreada** con la misma firma y comportamiento observable
  de Fase 16/ADR-0044 (tests de `phase16`/`phase34` sin cambios) — ahora delega en
  `create_schedule_rules_batch()` en vez de duplicar el insert.
- **`create_schedule_rule_span(p_service_id, p_resource_id, p_weekdays int[], p_range_start
  time, p_range_end time, p_step_minutes int, p_duration_minutes int, p_capacity int)
  returns uuid`** (RPC nueva, `grant ... to authenticated`) — mismo chequeo de permiso que
  `create_schedule_rule_group` (`is_organization_member`, nada más granular). Devuelve sólo
  el `group_id` compartido (no las filas). Expande `local_start_time` con
  `generate_series(date '2000-01-01' + range_start, date '2000-01-01' + range_end -
  duration_minutes, step_minutes)` (Postgres no tiene `generate_series` sobre `time` puro,
  de ahí el ancla de fecha fija — es aritmética de hora del día, nunca una fecha real) y
  llama a `create_schedule_rules_batch()` con el grid completo vía `array_agg(group_id)`
  (nunca `limit 1` sobre una set-returning function, que cortaría la ejecución tras la
  primera fila e insertaría sólo una regla del lote).
- **Topes defensivos** (antes de tocar la base): `step_minutes >= 5` (si no,
  `STEP_TOO_SHORT`); máximo 96 inicios por día (`TOO_MANY_START_TIMES`); máximo 7×96=672
  reglas por llamada (`TOO_MANY_SCHEDULE_RULES`) — sin esto, un solo request podría generar
  decenas de miles de `SlotOccurrence` en la ventana de 90 días.
- **Recurso exclusivo (ADR-0044)**: `step_minutes < duration_minutes` (los turnos generados
  se pisarían entre sí) rechaza con `SPAN_SELF_OVERLAP_ON_EXCLUSIVE_RESOURCE` antes de
  insertar nada; `p_capacity > 1` rechaza con `EXCLUSIVE_RESOURCE_CAPACITY_MUST_BE_ONE`.
  Ambos chequeos corren antes del chequeo de rango/weekdays, apenas resuelto el `Resource`.

**Tests** (`backend/test/phase43.schedule-rule-spans.test.ts`, 7 casos): franja simple
genera N reglas con `group_id` compartido; rechaza `step_minutes < 5`; rechaza al exceder el
tope de reglas por llamada; recurso exclusivo + `step_minutes < duration_minutes` →
`SPAN_SELF_OVERLAP_ON_EXCLUSIVE_RESOURCE` sin insertar nada; recurso exclusivo +
`capacity > 1` → `EXCLUSIVE_RESOURCE_CAPACITY_MUST_BE_ONE`; conflicto real contra una regla
existente en el mismo recurso exclusivo → `RESOURCE_SCHEDULE_CONFLICT`, atómico (nada
insertado); caso feliz en recurso exclusivo sin conflictos con `step_minutes ==
duration_minutes` (turnos pegados sin pisarse). **Pendiente de correr contra
`reservaste-stg`** (requiere confirmación explícita del usuario antes de tocar la base
compartida — no corrido todavía a la fecha de este resumen). No requiere gate de
`security-engineer` (mismo perímetro de permisos que `create_schedule_rule_group`, hereda la
RLS existente) — `reviewer` debe confirmar los topes defensivos explícitamente antes de
mergear.

## Fase 44 — cobrar un turno suelto desde el mostrador (ADR-0046, migración `20260930140000_phase44_drop_in_booking.sql`)

Implementa `docs/proposals/adr-0025-makeup-credits.md` §2.8 (diseño original nunca
construido) tal como lo cerró ADR-0046: dos RPC nuevas (`quote_booking()` /
`book_slot_paying()`) más una extensión aditiva de `agenda_occurrences()` y
`occurrence_bookings()`. Ninguna reimplementa `evaluate_payment_coverage()` /
`evaluate_payment_coverage_with_credit()` / `payment_covers_slot()` (ADR-0018): las
envuelve. **Pendiente de gate de `security-engineer`** (toca pagos y la frontera
multi-tenant) antes de mergear.

### `resolve_active_drop_in_plan(p_service_id)` — helper interno

El plan `DROP_IN` activo de un servicio, resuelto vía `service_plan_services` /
`applies_to_all_services` (ADR-0029 movió `service_plans.service_id` a esa tabla —
`service_plans` ya no tiene esa columna). A lo sumo una fila, garantizado por el trigger
`check_one_active_drop_in_per_service()` (`phase22`). `revoke` de todos los roles, mismo
patrón que `resolve_covering_service_plan()`.

### `quote_booking(p_slot_occurrence_id, p_customer_id)` — lectura, sin side-effects

Devuelve una fila `{can_book, reason, coverage_path, price, currency, makeup_credit_id,
makeup_credit_expires_on}`. `reason` es el `can_book_reason` de siempre (nunca un valor
nuevo). `coverage_path` es un enum nuevo, **de lectura únicamente** (ADR-0025 §2.4.4):
`FREE_SERVICE | DROP_IN_PAID | PLAN_UNLIMITED | PLAN_QUOTA | MAKEUP_CREDIT | NONE`.

Autorización resuelta **antes** de tocar la fila de `customers`, para que la RPC no sirva
para sondear si un `customer_id` ajeno existe en otra organización: `has_org_permission(
org, 'MANAGE_BOOKINGS')` (staff), o un `exists` independiente contra `customers` que
compara `profile_id = auth.uid()` directamente (el propio cliente consultando su propia
cobertura). Sólo después de pasar ese gate se hace el `select * into v_customer` que
puede devolver `NOT_A_CUSTOMER`.

### `book_slot_paying(p_slot_occurrence_id, p_customer_id, p_amount default null)` — escritura atómica, sólo-mostrador

`security definer`, exige **`MANAGE_PAYMENTS`** siempre (escribe un `Payment`, mismo
permiso que la policy `payments_insert_staff` de Fase 32 ya exige para insertar
cualquier pago) y **`MANAGE_BOOKINGS`** además, sólo si hace falta crear la `Booking`
(Fase 34: un cajero con `MANAGE_PAYMENTS` puede cobrar un turno ya anotado, pero no
anotar uno nuevo). **Corrección del gate de seguridad (2026-09-30)**: el diseño original
exigía sólo `MANAGE_BOOKINGS` — con eso, un rol configurado a propósito sin permiso de
cobro (ADR-0033) podía registrar un `Payment PAID` con `p_amount=0`, esquivando el
`PAYMENT_REQUIRED` que `admin_book_for_customer()` le devuelve al mismo rol (verificado
en vivo contra `reservaste-stg`). **El cliente no puede invocarla** (sin pasarela real,
ADR-0027 sigue postergada; dejar que se autodeclare pagado sin verificación sería peor
que no tener la RPC). Mismo orden de locks que `book_slot()` /
`admin_book_for_customer()`: `select ... for update` sobre la ocurrencia primero.

Estados posibles en `status`: `OCCURRENCE_NOT_AVAILABLE`, `CUSTOMER_NOT_IN_ORG` (la
frontera multi-tenant principal: el `customer_id` tiene que pertenecer a la organización
de la ocurrencia ya lockeada, que nunca sale de un input libre), `ALREADY_COVERED`,
`SLOT_FULL`, `NO_DROP_IN_PLAN`, `INVALID_AMOUNT`, `ALREADY_PAID` (vía `unique_violation`
sobre `payments_one_paid_per_occurrence_idx`, `phase17`), `OK` (con `booking` y
`payment` serializados).

**Decisión de implementación que no estaba escrita mecánicamente en ADR-0046, confirmada
por `security-engineer` en el gate (2026-09-30) como la lectura correcta**: el re-chequeo de
"¿ya está cubierto?" sólo corre cuando `services.payment_required = true`. Motivo: el
paso 1 de `evaluate_payment_coverage()` devuelve `'OK'` para **todo** servicio con
`payment_required = false`, sin haber mirado plan/pago/crédito — si esa vía contara como
"ya cubierto", sería imposible cumplir el caso de negocio que la propia ADR-0046
resolución 2 cita explícitamente ("un negocio con `payment_required=false` que igual
quiere dejar registro de un cobro hecho en efectivo después del servicio"): la función
jamás llegaría a cobrar nada, siempre devolvería `ALREADY_COVERED`. Con el chequeo
condicionado a `payment_required`, un servicio gratuito con un `DROP_IN` configurado
permite registrar el cobro igual, y la protección contra doble cobro en ese caso la da
únicamente el índice único (`ALREADY_PAID`) — cubierto explícitamente por el test (d),
que documenta por qué usa un servicio gratuito en vez de uno pago (con
`payment_required=true` el mismo lock de la ocurrencia hace que el segundo llamador,
tras esperar, vea el pago recién comprometido del primero y salga por `ALREADY_COVERED`
en vez de `ALREADY_PAID` — el índice único casi nunca se ejercita en ese camino).

"Ya anotado, falta cobrar" (resolución 2): si la `Booking` ya existe, la función sólo
inserta el `Payment` — nunca vuelve a chequear cupo ni intenta reservar de nuevo. Si no
existe, capacidad se chequea **antes** de intentar cobrar (si está llena, `SLOT_FULL` sin
haber tocado `payments`); recién después se resuelve el `DROP_IN` activo y se cobra.
Desde el insert del `Payment` en adelante, cualquier fallo (`BOOKING_RACE_LOST`, la
carrera de `on conflict ... do nothing` sobre `bookings_customer_occurrence_confirmed_idx`)
aborta con excepción, no con un status — mismo criterio ya documentado en la Fase 20 para
`MAKEUP_CREDIT_RACE_LOST`: "cobrar y reservar son el mismo hecho" (ADR-0025 §2.8.1) exige
que un fallo posterior al cobro deshaga también el cobro.

El plan y el precio nunca son input del cliente: el plan sale de
`resolve_active_drop_in_plan()`; `p_amount` sólo permite el descuento de mostrador que
decide el staff con `MANAGE_PAYMENTS` que ya pasó el gate de autorización — nunca lo fija
quien reserva. `p_amount = 'NaN'::numeric` no es `< 0` en Postgres y `numeric(12,2)` lo
acepta sin queja; sin el chequeo explícito (`v_amount is null or v_amount = 'NaN' or
v_amount < 0` → `INVALID_AMOUNT`) quedaba un `PAID` con `amount NaN`, envenenando
cualquier `sum()` de reportes — hallazgo MEDIO del gate, también corregido.

`resolve_active_drop_in_plan()` filtra explícitamente `sp.organization_id =
s.organization_id` — **hallazgo ALTO del gate de seguridad, corregido**: la versión
original no filtraba por organización, así que un plan `DROP_IN` con
`applies_to_all_services=true` de CUALQUIER organización matcheaba todos los servicios de
la plataforma (verificado en vivo: el precio de un tenant ajeno aparecía en
`agenda_occurrences`/`quote_booking` de otro tenant, y `book_slot_paying` abortaba con un
error de integridad — un DoS del cobro para todos los tenants). El `order by
applies_to_all_services asc, created_at asc, id asc` desempata de forma determinista el
caso (no cubierto por `check_one_active_drop_in_per_service()`) de un `DROP_IN` explícito
y uno `applies_to_all` en la misma organización: gana el vínculo explícito.

**Gate de `security-engineer`: LISTO** (segunda pasada, tras los tres fixes de arriba).
Dos hallazgos menores quedaron como decisión del Orchestrator, documentados en ADR-0046
("Corrección post-gate") y en `docs/security.md`: un `reason` (no dato) que puede
filtrarse entre tenants en `evaluate_payment_coverage()` — pre-existente, deuda técnica
separada — y un edge case de doble cobro deliberado del staff en un servicio gratuito con
plan `UNLIMITED` vigente, aceptado por baja severidad.

### `agenda_occurrences()` / `occurrence_bookings()` — extensión aditiva

`agenda_occurrences()` (Fase 8) agrega `drop_in_plan_id, drop_in_price, drop_in_currency`
al final (`null` si el servicio no tiene un `DROP_IN` activo), vía
`left join lateral (select (resolve_active_drop_in_plan(so.service_id)).*) as dip on
true` — la expansión de un valor compuesto posiblemente `NULL` produce siempre una fila
(con columnas `NULL`), así que el `LEFT JOIN ... ON TRUE` nunca descarta la ocurrencia.
`occurrence_bookings()` (Fase 8/16/21) agrega `is_covered` (reusa `payment_covers_slot()`,
nunca reimplementado) y `paid_payment_id` por asistente. Las dos usan `drop function` +
`create function` (cambia el `returns table`, `CREATE OR REPLACE` no alcanza) y
re-emiten sus grants a `authenticated` — aprovechando el `drop`, también se les agregó un
`revoke` explícito de `public`/`anon` que la versión original (Fase 8, anterior a
ADR-0028) nunca tuvo — mismo endurecimiento sin cambio de comportamiento para un caller
legítimo, a confirmar en el gate de seguridad.

### Tests (`backend/test/phase44.drop-in-booking.test.ts`)

Los ocho casos que pidió el Orchestrator: (a) alta atómica Payment+Booking para un
cliente nuevo; (b) sólo Payment cuando la Booking ya existía (servicio gratis + `DROP_IN`
opcional, el caso de negocio de la resolución 2); (c) `ALREADY_COVERED` sin cobrar de
nuevo; (d) doble cobro concurrente → uno `OK`, uno `ALREADY_PAID` (con el comentario
explicando por qué ese test necesita un servicio gratuito, ver arriba); (e)
`CUSTOMER_NOT_IN_ORG` cross-tenant; (f) un cliente no puede invocar `book_slot_paying`;
(g) `quote_booking` sí lo puede invocar el propio cliente, nunca para otro `customer_id`;
(h) `NO_DROP_IN_PLAN`. **Verificado en vivo contra `reservaste-stg`**: los 8 casos pasan,
incluidos en los 366/366 de la suite completa. Migración sin
aplicar contra `reservaste-stg` todavía.

## Fase 54 — `customer_contact()` (ADR-0050)

Función nueva `customer_contact(p_customer_id uuid) returns table (email
text, phone text)`: `security definer`, `stable`, `search_path = public`.
La organización se deriva de `customers.organization_id`; exige
`is_organization_member()` (OWNER/STAFF activos). Cliente inexistente,
de otra organización, o llamador CUSTOMER → 0 filas. `email` viene de
`auth.users` vía `customers.profile_id` (null en clientes gestionados);
`phone` es `customers.phone`. `revoke` a `public, anon`; `grant` a
`authenticated`. `organization_customers()` no cambia. Migración aditiva:
`20261005120000_phase54_customer_contact.sql`.
