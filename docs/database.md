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
