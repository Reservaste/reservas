# Registro de decisiones arquitectónicas (ADR)

Mantenido por: Orchestrator. Cualquier subagente puede **proponer** una
entrada nueva (vía `/propose-decision`), pero solo el Orchestrator la marca
como `Aceptada`.

Formato de cada entrada:

```
## ADR-000X — Título
Fecha: YYYY-MM-DD
Estado: Propuesta | Aceptada | Rechazada | Reemplazada por ADR-000Y
Propuesta por: <agente o usuario>

Problema:
Cambio:
Impacto:
Migración / compatibilidad:
Decisión:
```

---

## ADR-0001 — Sistema de agentes especializados para el desarrollo del proyecto

Fecha: 2026-09-14
Estado: Aceptada
Propuesta por: usuario / Orchestrator

**Problema:** desarrollar una plataforma multi-tenant compleja (agenda,
cupos, pagos, recurrencia) con un único agente monolítico genera
inconsistencias de arquitectura, modelo de datos, contratos de API y UI.

**Cambio:** se define un Orchestrator/Tech Lead (el hilo principal) que
coordina 12 subagentes especializados (`.claude/agents/`), con
documentación compartida en `/docs/*.md` como memoria de contexto
obligatoria, y un workflow de aprobación para cualquier decisión
estructural (dominio, schema, auth, contratos de API, estructura de
carpetas, estrategia de slots/pagos/recurrencia).

**Impacto:** ningún subagente puede tomar decisiones estructurales por su
cuenta; deben proponerlas al Orchestrator. Se prioriza consistencia sobre
velocidad de una sola tarea aislada.

**Migración / compatibilidad:** n/a (repo nuevo).

**Decisión:** aceptada. Ver `docs/agent-responsibilities.md` para el
catálogo completo y `CLAUDE.md` para el rol del Orchestrator.

---

## ADR-0002 — Stack técnico concreto

Fecha: 2026-09-14
Estado: **Aceptada**
Propuesta por: usuario, confirmada por Orchestrator (sin impedimento
técnico: el repo estaba vacío al momento de esta decisión, sin código
previo con el que colisionar) y validada por `database-agent` en el
cierre de Phase 0 (ver más abajo).

**Decisión:**

| Capa | Elección |
|---|---|
| Frontend / Backend | Next.js, App Router, TypeScript strict |
| Base de datos | PostgreSQL vía Supabase |
| Autenticación | Supabase Auth (email/password + Google OAuth) |
| Acceso a datos | supabase-js + RPC de PostgreSQL para operaciones transaccionales críticas |
| Validación | Zod |
| UI | TailwindCSS + shadcn/ui |
| Hosting | Vercel |

**Explícitamente sin ORM adicional** (nada de Prisma ni equivalente)
salvo que aparezca una razón técnica concreta y demostrable — se
aprovecha PostgreSQL/Supabase directamente: RLS, RPC, transactions,
database functions, indexes, constraints. `database-agent` confirmó en
el cierre de Phase 0 que supabase-js + RPC alcanza para lo que exige
ADR-0004 (RPC atómica) y ADR-0006 (RLS de dos capas) sin necesidad de un
ORM (ver detalle de esa confirmación en `database.md`).

**Impacto:** desbloquea Phase 1. `backend-api-agent` implementa server
actions/route handlers de Next.js consumiendo supabase-js;
`database-agent` escribe el schema y las funciones RPC directamente en
SQL/migraciones de Supabase, sin capa de ORM intermedia.

**Migración / compatibilidad:** n/a (repo nuevo, sin código previo).

---

## ADR-0003 — Persistencia de SlotOccurrence

Fecha: 2026-09-14
Estado: **Aceptada**
Propuesta por: `scheduling-agent` + `database-agent` + `domain-architect`
(revisión coordinada de Phase 0, tres agentes independientes convergieron
en la misma recomendación)

**Problema:** definir si las ocurrencias de horario se materializan como
filas en la base de datos o se calculan dinámicamente a partir de
`ScheduleRule` + `ScheduleException`.

**Decisión: `SlotOccurrence` se persiste como filas materializadas**,
generadas por un proceso de generación con **horizonte rodante** (rolling
window, ej. 60-90 días hacia adelante) a partir de `ScheduleRule` +
`ScheduleException`. `ScheduleException` se mantiene como registro
explícito de la desviación (auditoría + input del generador), no
desaparece por persistir las ocurrencias.

**Por qué (consolidado de los tres agentes):**
- `Booking` necesita una FK real y un `UNIQUE(customerId, slotOccurrenceId)`
  enforced por la base — esto exige una fila física con PK estable. Con
  cálculo dinámico no hay qué referenciar sin recurrir a claves sintéticas
  frágiles.
- El mecanismo de concurrencia elegido en ADR-0004 (`SELECT ... FOR
  UPDATE`) necesita una fila física para lockear. Calculado dinámicamente
  fuerza a caer en advisory locks sobre claves sintéticas, la opción menos
  robusta bajo el pooler de Supabase.
- Historial: una ocurrencia con `Booking` asociada debe quedar congelada
  (fecha/hora/recurso/capacidad en el momento de la reserva), igual que
  una `Booking` cancelada nunca se borra. Calcular dinámicamente arriesga
  reescribir retroactivamente el significado de ocurrencias pasadas si se
  edita la regla que las originó.
- `ScheduleException` (cancelar/modificar una ocurrencia puntual sin
  afectar el resto de la serie) se modela de forma natural como `UPDATE`
  sobre una fila existente, sin duplicar lógica de merge regla+excepción
  en cada punto de lectura.

**Política de horizonte (para no generar "hacia el infinito" con reglas
sin fecha de fin):**
- Al crear/editar una `ScheduleRule` se materializa inmediatamente su
  horizonte vigente; un job periódico extiende la ventana rodante.
- Ocurrencias futuras **sin** `Booking` son regenerables/descartables
  libremente si la regla cambia (son proyección de la regla, no fuente de
  verdad).
- Ocurrencias con **al menos una** `Booking` quedan "pineadas": no se
  regeneran ni se borran silenciosamente. Cualquier cambio pasa por
  cancelación explícita.
- Modificar una regla "desde una fecha en adelante": se cierra la
  vigencia de la regla vieja, se crea una regla nueva desde esa fecha, y
  solo se regenera la cola futura sin bookings de la regla vieja.

**Impacto:** `database-agent` diseña el schema y el job de generación;
`scheduling-agent` implementa la lógica de merge regla+excepción al
generar; `booking-engine-agent` reserva siempre contra una fila real de
`SlotOccurrence`.

---

## ADR-0004 — Estrategia de concurrencia para el último cupo

Fecha: 2026-09-14
Estado: **Aceptada**
Propuesta por: `database-agent` (revisión coordinada de Phase 0)

**Problema:** garantizar que, bajo reservas simultáneas por el último
cupo disponible, solo una tenga éxito, sin depender de
`if available > 0: insert` sin protección transaccional.

**Decisión: RPC atómica en PostgreSQL** (ej. `book_slot(slotOccurrenceId,
customerId, ...)`, invocada vía `supabase.rpc(...)`) que ejecuta en una
única transacción server-side:

1. `SELECT ... FROM slot_occurrences WHERE id = $1 FOR UPDATE` — lockea
   solo la fila de esa ocurrencia puntual (no la tabla completa).
2. Verifica `activeBookings < capacity`.
3. Si hay cupo: inserta la `Booking` en `CONFIRMED`.
4. Si no hay cupo: devuelve un resultado explícito (`SLOT_FULL`), no una
   excepción genérica.

Respaldado por `UNIQUE (customerId, slotOccurrenceId) WHERE
status = 'CONFIRMED'` como defensa secundaria contra duplicados del mismo
customer (doble click / doble request).

**Por qué esta y no las otras:**
- **Por qué RPC y no row locking "suelto" desde el backend**: supabase-js
  habla con PostgREST — cada llamada `.from(...)` es su propia transacción
  HTTP independiente. No se puede abrir `BEGIN`/`FOR UPDATE` en una
  llamada e insertar en la siguiente porque el lock ya se liberó. Una
  función PL/pgSQL invocada por RPC sí corre como una única transacción
  real en un solo round-trip.
- **Por qué no constraint+retry puro**: la capacidad no es un valor fijo
  por fila de booking, es un agregado contra un límite mutable — expresar
  eso en un `CHECK` sin lock termina necesitando un contador denormalizado
  + trigger, que es una forma peor documentada del mismo problema.
- **Por qué no advisory locks como mecanismo primario**: requieren derivar
  una clave sintética (hash de UUID a bigint) en vez de lockear una fila
  real ya existente, y son más frágiles bajo el pooler transaccional de
  Supabase si dos statements no caen en la misma conexión física.

**Impacto:** `database-agent` implementa la función SQL;
`booking-engine-agent` la consume y maneja sus tres resultados posibles
(éxito, sin cupo, duplicado) sin reimplementar lógica de locking en la
aplicación.

**Nota de seguridad (de `auth-security-agent`):** `canCustomerBook()` es
un gate de negocio, no el mecanismo de atomicidad — que devuelva `true` no
es autorización suficiente para el insert final; el insert siempre pasa
por esta RPC, nunca por un `insert` directo posterior a la validación.

---

## ADR-0005 — Contrato de `canCustomerBook()` (prevención de IDOR)

Fecha: 2026-09-14
Estado: **Aceptada**
Propuesta por: `auth-security-agent` (revisión coordinada de Phase 0)

**Problema:** `canCustomerBook()` (dueña: `payments-entitlements-agent`)
estaba descripta solo en términos de reglas de negocio, sin especificar de
dónde salen los IDs que recibe. Si un endpoint acepta `customerId` como
campo de request en vez de derivarlo de la sesión, un `Customer`
autenticado podría pasar el `customerId` de otra persona y la función,
validando solo reglas de negocio, terminaría operando sobre el
entitlement/pago de otro cliente (IDOR). Del mismo modo, sin validar
consistencia de `organizationId` entre `Customer`/`Service`/
`SlotOccurrence`, un customer de la Org A podría apuntar a un
`slotOccurrenceId` de la Org B.

**Decisión:** `canCustomerBook()` recibe `(profileId, organizationId,
serviceId, slotOccurrenceId)` — **nunca** un `customerId` de confianza
pasado por el caller — y resuelve/valida el `Customer` internamente a
partir de `profileId` + `organizationId`. Valida explícitamente que
`customer.organizationId == service.organizationId ==
slotOccurrence.organizationId` (vía su `Resource`) antes de evaluar el
resto de las condiciones. El resultado nunca se reutiliza entre requests:
se re-ejecuta inmediatamente antes de cada intento de escritura. Para el
flujo donde `STAFF`/`OWNER` reserva en nombre de un customer, la capa de
API valida membership del staff sobre la organización — no ownership por
`profileId` — antes de invocar la función con el `profileId` del customer
en cuyo nombre se reserva.

**Impacto:** `payments-entitlements-agent` ajusta la firma de
`canCustomerBook()`; `backend-api-agent` nunca acepta `customerId` como
input directo de un `Customer` autenticado reservando para sí mismo.

---

## ADR-0006 — RLS de dos capas para tablas con datos de customer

Fecha: 2026-09-14
Estado: **Aceptada**
Propuesta por: `auth-security-agent` (revisión coordinada de Phase 0)

**Problema:** filtrar RLS solo por `organizationId` no alcanza para
`Booking`, `Payment`, `ServiceEntitlement` y `Customer`, porque un mismo
`Profile` puede ser `Customer` de varias `Organization` — una policy de
solo-tenant dejaría que cualquier `CUSTOMER` autenticado en la Org A vea
los bookings/pagos de **todos** los customers de esa misma organización,
no solo los propios.

**Decisión:** políticas RLS de dos condiciones combinadas con `OR`:
- **CUSTOMER**: acceso donde existe un `Customer` con
  `profileId = auth.uid()` dueño de la fila, más chequeo redundante de
  `organizationId` como defensa en profundidad.
- **OWNER/STAFF**: acceso donde existe un `OrganizationMember` con ese
  `profileId` sobre el `organizationId` de la fila.

No usar un claim de "organización actual de sesión" único para el rol
`CUSTOMER` — cada request valida membership/ownership contra la
organización del recurso puntual, no contra un tenant fijo de sesión,
justamente porque un customer puede tener relación simultánea con varias
organizaciones.

**Impacto:** `database-agent` implementa estas policies al diseñar el
schema. Queda abierta para `backend-api-agent` la definición exacta de
`GET /me/bookings` (¿agrega across todas las organizaciones del profile,
o requiere parámetro de organización?) — se resuelve al implementar
Phase 1/2, no bloquea esta decisión de RLS.

---

## ADR-0007 — Límite de disclosure en disponibilidad pública de baja capacidad

Fecha: 2026-09-14
Estado: **Reemplazada por ADR-0008** (el usuario prefirió una configuración
explícita Organization/Service en vez de un umbral automático por
capacidad — ver ADR-0008. El problema de seguridad que motivó esta ADR
sigue vigente y resuelto, solo cambió el mecanismo.)
Propuesta por: `auth-security-agent` (revisión coordinada de Phase 0)

**Problema:** `docs/security.md` trataba "disponibilidad agregada" (ej.
"4 disponibles de 12") como públicamente segura por definición. No lo es
en capacidades bajas: para un `Service`/`SlotOccurrence` con
`maxCapacity` pequeña (ej. 1-2, típico de turnos individuales —
consultorio, sesión 1-a-1, cancha privada), pasar de "1 disponible" a "0
disponibles" revela con certeza que existe una reserva confirmada
específica. Como la plataforma es genérica por diseño, no se puede asumir
que siempre habrá capacidades altas tipo clase grupal de gimnasio.

**Decisión:** para `Service`/`Resource` con capacidad configurada por
debajo de un umbral (default: `maxCapacity <= 3`, configurable por
`Organization`), el endpoint público de disponibilidad expone un estado
booleano/bucket ("disponible" / "sin lugares" / "pocos lugares") en vez
del conteo exacto `X de Y`. Por encima del umbral se sigue exponiendo el
conteo agregado normal. Adicionalmente, se aplica rate limiting al
endpoint público de disponibilidad para mitigar inferencia por polling.

**Impacto:** `backend-api-agent` implementa el umbral al definir el shape
de respuesta de disponibilidad en `docs/api.md`; `scheduling-agent`
provee el dato de capacidad necesario para decidir el bucket.

---

## ADR-0008 — `publicAvailabilityDisplay` configurable (reemplaza ADR-0007)

Fecha: 2026-09-14
Estado: **Aceptada**
Propuesta por: usuario, diseñada por `backend-api-agent`, revisada por
`auth-security-agent` (cierre de Phase 0)

**Problema:** ADR-0007 fijaba un umbral automático (`maxCapacity <= 3`)
para decidir cuándo ocultar el conteo exacto. El usuario prefiere que la
propia `Organization`/`Service` elija explícitamente cómo mostrar
disponibilidad, en vez de que el sistema lo infiera de la capacidad.

**Decisión:** propiedad `publicAvailabilityDisplay` con tres modos:

- `EXACT` — "4 lugares disponibles" (conteo real).
- `LIMITED` — "Disponible" / "Últimos lugares" / "Completo" (3 estados).
- `BOOLEAN` — "Disponible" / "Sin disponibilidad" (2 estados).

Ubicación: `Organization.publicAvailabilityDisplay` (default de la
organización) + `Service.publicAvailabilityDisplayOverride` nullable
(override puntual). Modo efectivo = `service.override ?? organization.default`.
Ejemplo del propio usuario: un gimnasio usa `EXACT` a nivel Organization;
un `Service` de psicólogo hace override a `BOOLEAN`.

Es un **valor persistido y elegido explícitamente**, no recalculado
dinámicamente en base a `maxCapacity` — así cambiar la capacidad de un
servicio no cambia silenciosamente su política de disclosure.

**Garantía de seguridad que se mantiene de ADR-0007:** el backend nunca
manda el conteo exacto salvo en modo `EXACT`. En `LIMITED`/`BOOLEAN`, los
campos numéricos (`remaining`, `capacity`) **no se incluyen en el
payload** (no se mandan en `null` — se omiten del todo), para que no sea
ni siquiera técnicamente posible que el frontend los lea. La regla aplica
a **todo** endpoint público que exponga capacidad de un slot — listado de
disponibilidad y detalle de un slot puntual (gap señalado por
`auth-security-agent`: sin esto, alguien podría implementar el detalle de
un slot devolviendo el conteo "porque es uno solo, no una lista").

**Shape de respuesta** (`GET /organizations/:slug/availability`), unión
discriminada por `mode`:

```jsonc
// EXACT
{ "availability": { "mode": "EXACT", "remaining": 4, "capacity": 12, "label": "4 lugares disponibles" } }
// LIMITED
{ "availability": { "mode": "LIMITED", "status": "AVAILABLE"|"LOW"|"FULL", "label": "..." } }
// BOOLEAN
{ "availability": { "mode": "BOOLEAN", "status": "AVAILABLE"|"FULL", "label": "..." } }
```

Umbral interno de `LOW` en modo `LIMITED` (configurable por
`Organization`, no hardcodeado): `lowAvailabilityPercentage` (default 20%)
combinado con `lowAvailabilityFixedCap` (default 3) —
`threshold = max(1, min(fixedCap ?? ∞, ceil(capacity * percentage / 100)))`.
El cálculo del estado vive en la capa de dominio
(`booking-engine-agent`/`scheduling-agent`), la API solo serializa.

**Impacto:** reemplaza el umbral fijo de ADR-0007. `database-agent` agrega
las dos columnas a `Organization`/`Service`; `backend-api-agent` implementa
el shape de respuesta; `auth-security-agent` confirma que la garantía de
no-disclosure se mantiene igual que en ADR-0007, solo cambia quién decide
el modo.

---

## ADR-0009 — Mecanismo de generación del rolling window de SlotOccurrence

Fecha: 2026-09-14
Estado: **Aceptada** (extiende ADR-0003)
Propuesta por: `scheduling-agent`, confirmada por `database-agent`
(cierre de Phase 0)

**Problema:** ADR-0003 fijó que `SlotOccurrence` se persiste con horizonte
rodante, pero no definió qué proceso concreto mantiene esa ventana
(90 días, adoptado en esta revisión como horizonte inicial).

**Decisión:** **pg_cron (extensión nativa de Postgres en Supabase) como
mecanismo primario, corriendo diariamente**, invocando directamente una
función PL/pgSQL idempotente (`generate_slot_occurrences`), **más un
fallback lazy** en el camino de lectura de disponibilidad (si se detecta
un hueco de cobertura contra el horizonte esperado, se dispara la misma
función antes de responder) como red de seguridad si el cron falla
silenciosamente.

**Por qué:** pg_cron corre dentro de la instancia de Postgres — sin salto
de red, sin endpoint HTTP que proteger, consistente con el patrón ya
aceptado en ADR-0004 (correctness en la base, no en la capa de
aplicación). El fallback lazy solo no alcanza (latencia alta en la
primera consulta tras inactividad, cobertura no garantizada para vistas
que no disparan lectura de disponibilidad); el cron solo tampoco alcanza
(requiere que nadie note una falla silenciosa) — la combinación cubre
ambos huecos.

**Detalles de implementación (de `database-agent`, para que no se pierdan
al construir la migración):**
- La función hace `INSERT ... SELECT ... ON CONFLICT DO NOTHING`,
  **batcheada por `ScheduleRule`** (un `INSERT` por regla, no un mega-insert
  cruzando organizaciones), para que una regla no bloquee al resto.
- Envuelta en `pg_try_advisory_lock` para que una ejecución solapada
  (cron + lazy disparando a la vez, o dos runs de cron superpuestos) salga
  inmediatamente en vez de competir por los mismos inserts.
- Se invoca también de forma síncrona al crear/editar una `ScheduleRule`
  (ya establecido en ADR-0003).
- Riesgo operativo anotado: en el plan Free de Supabase el proyecto puede
  pausarse por inactividad y silenciar el cron — no aplica si Phase 1
  arranca en un plan pago, pero queda registrado como riesgo conocido.

**Impacto:** `database-agent` implementa la función y la extensión
`pg_cron`; no requiere infraestructura fuera de Supabase (sin Vercel Cron,
sin Edge Functions).

---

## ADR-0010 — Modelo de recurrencia: RecurringBooking, estados de SlotOccurrence y taxonomía de cancelación

Fecha: 2026-09-14
Estado: **Aceptada**
Propuesta por: `domain-architect`, validada operacionalmente por
`booking-engine-agent` (cierre de Phase 0)

**Problema:** el modelo de `RecurringBooking` ↔ `SlotOccurrence` ↔
`Booking` no tenía campos ni invariantes precisas; `SlotOccurrence` no
tenía estados definidos; `Booking.cancelledBy` no distinguía motivos.

**Decisión — modelo validado con el caso Mathias/CrossFit:**

`RecurringBooking { id, organizationId, customerId, scheduleRuleId (FK — nunca duplicar día/hora acá), status: ACTIVE|CANCELLED, startDate, endDate?, createdAt, updatedAt, createdBy, cancelledAt?, cancelledBy?, cancellationReason? }`.
Cada `Booking` generada por una recurrencia referencia **tanto**
`recurringBookingId` como `slotOccurrenceId`. Cancelar una fecha puntual
(ej. Mathias cancela el 28/09) toca **solo esa `Booking`**
(`status=CANCELLED`, `cancelledBy=CUSTOMER`) — `RecurringBooking` sigue
`ACTIVE` sin tocarse. Cancelar la serie completa "desde hoy en adelante"
toca `RecurringBooking` (`status=CANCELLED`) y cancela en cascada solo
las `Booking` futuras en `CONFIRMED`; el histórico pasado nunca se toca.
`RecurringBooking.status` permanece `ACTIVE` aunque **todas** sus
`Booking` actuales estén canceladas individualmente — expresa la
intención/suscripción del cliente, no un agregado de sus hijos; el job de
horizonte rodante sigue generando `Booking` nuevas mientras esté `ACTIVE`.

**Amplía el invariante ya aceptado en `domain.md`** (cascada al
discontinuar un `ScheduleRule`): la cascada cancela explícitamente
también toda `RecurringBooking` que seguía esa regla (`status=CANCELLED`,
`cancelledBy=ORGANIZATION`), no solo sus `Booking` hijas — de lo
contrario la serie queda `ACTIVE` apuntando a una regla que ya no existe.

**Estados de `SlotOccurrence`: `ACTIVE | BLOCKED | CANCELLED`** (tres
persistidos). `BLOCKED` es administrativo y reversible (mantenimiento,
feriado) sobre una ocurrencia sin reservas activas — distinto de
`CANCELLED`, que es terminal/histórico. **`COMPLETED` NO se persiste** —
es un derivado puramente temporal (`startAt < now()`), calculado en la
capa de lectura, para evitar un cuarto estado sin decisión de negocio
detrás.

**Taxonomía de cancelación de `Booking`** — `cancelledBy` (QUIÉN:
`CUSTOMER | ORGANIZATION`) + `cancellationReason` (POR QUÉ, texto/enum:
`CUSTOMER_REQUEST | SLOT_CANCELLED | RULE_DISCONTINUED`). Se eligen dos
campos ortogonales en vez de fusionar todo en un solo motivo — permite
que `notifications-agent`/reportes filtren por actor sin parsear el
motivo, y agregar nuevos motivos sin tocar el campo `cancelledBy`. Esto
resuelve el punto abierto que `booking-engine-agent` elevó al Orchestrator
(`SLOT_CANCELLED` como motivo propio, distinto del `ORGANIZATION`
genérico de discontinuar una regla completa) — la decisión final es esta
separación en dos campos, siguiendo la alternativa 2 que el propio
`booking-engine-agent` había esbozado.

**`Booking.status = NOT_GENERATED`** (ya en `domain.md`) es **terminal**,
no se reintenta automáticamente si se libera cupo después — decisión del
Orchestrator, aceptando la recomendación de `booking-engine-agent`: evita
una cola de reintentos asíncronos no pedida y mantiene la misma política
de "nunca reservas parciales silenciosas" también para la liberación de
cupo. Si se libera cupo, el cliente/admin debe reservar manualmente esa
fecha si la quiere.

**Matriz de auditoría por entidad** (de `domain-architect`, aplica el
criterio "solo si hay lifecycle real de baja + importa saber quién/por
qué"): llevan `cancelledAt/cancelledBy/cancellationReason` además de
`createdAt/updatedAt/createdBy` → `OrganizationMember`, `Customer`,
`Service`, `ServiceEntitlement`, `Resource`, `ScheduleRule`,
`SlotOccurrence` (sin `createdBy`, se genera por job), `Booking`,
`RecurringBooking`. Solo `createdAt/updatedAt` (sin cancelación) →
`Organization`, `Profile`, `ScheduleException` (la excepción en sí ya es
el registro de la desviación). `Payment` tiene su propio concepto de baja
(`VOID`, ver ADR-0013), no reusa la tripleta genérica.

**Impacto:** `database-agent` agrega `scheduleRuleId` como FK en
`RecurringBooking` (no duplicar día/hora), tercer valor de `status` en
`SlotOccurrence`, columnas `cancelledBy`/`cancellationReason` separadas en
`Booking`/`RecurringBooking`, y la matriz de auditoría por tabla.

---

## ADR-0011 — Generación idempotente de Bookings recurrentes

Fecha: 2026-09-14
Estado: **Aceptada** (extiende ADR-0004)
Propuesta por: `booking-engine-agent`, extendida por `database-agent`
(cierre de Phase 0)

**Problema:** al extender el rolling window, hay que crear
automáticamente las `Booking` de las `RecurringBooking ACTIVE` que
matchean las nuevas `SlotOccurrence`, sin duplicar y sin asumir que la
recurrencia tiene prioridad ilimitada sobre el cupo.

**Decisión:** dos `UNIQUE` distintos, en granos distintos, que conviven
sin competir:

```sql
-- Idempotencia del job: nunca más de una Booking por (serie, ocurrencia),
-- sin importar el status — protege contra doble-ejecución/retry/overlap.
CREATE UNIQUE INDEX uq_booking_recurring_occurrence
  ON bookings (recurring_booking_id, slot_occurrence_id)
  WHERE recurring_booking_id IS NOT NULL;

-- Ya aceptado en ADR-0004: nunca más de una Booking CONFIRMED del mismo
-- customer sobre la misma ocurrencia, sin importar el origen.
CREATE UNIQUE INDEX uq_booking_customer_occurrence_confirmed
  ON bookings (customer_id, slot_occurrence_id)
  WHERE status = 'CONFIRMED';
```

**Hallazgo importante de `database-agent`** (no estaba contemplado
originalmente): un mismo customer puede estar en **dos `RecurringBooking`
distintas** que un día coinciden generando sobre la misma
`SlotOccurrence` — el segundo constraint rechazaría esa segunda fila como
`CONFIRMED`. Por eso **la generación recurrente no puede ser un `INSERT`
de aplicación**: necesita ser una **RPC atómica hermana de `book_slot()`**
(ej. `generate_recurring_booking(recurringBookingId, slotOccurrenceId)`),
que bajo el mismo `FOR UPDATE` sobre la ocurrencia hace
`INSERT ... ON CONFLICT (recurring_booking_id, slot_occurrence_id) DO NOTHING`
para la idempotencia del job, y si no hay cupo o el customer ya tiene otra
`CONFIRMED` ahí, inserta directamente con `status='NOT_GENERATED'` en vez
de dejar que la violación de constraint burbujee sin manejar.

**Impacto:** `booking-engine-agent` y `database-agent` coordinan una
segunda función RPC (no reutiliza `book_slot` literal, pero comparte su
patrón de "resultado explícito, nunca excepción genérica"). El job de
ADR-0009 llama a esta RPC por cada par (`RecurringBooking ACTIVA`,
`SlotOccurrence` nueva que matchea su patrón).

---

## ADR-0012 — Política de cupo + recurrencia (conflictos al crear una serie)

Fecha: 2026-09-14
Estado: **Aceptada**
Propuesta por: `booking-engine-agent` (cierre de Phase 0), preferencia
inicial del usuario

**Problema:** al crear una `RecurringBooking` (ej. semanal), algunas de
las próximas ocurrencias ya materializadas pueden estar llenas. La
recurrencia no tiene prioridad ilimitada sobre el cupo — no se acepta
overbooking ni reserva parcial silenciosa.

**Decisión — flujo de preview + confirmación explícita:**

1. **Preview de solo lectura** (`POST /recurring-bookings/preview` o
   `dryRun:true`): para las próximas N ocurrencias (ej. 12) ya
   materializadas que matchean el patrón, calcula disponibilidad sin
   `FOR UPDATE` (informativo, no reserva nada) y devuelve fechas con cupo
   vs. fechas en conflicto, con la fecha exacta de cada una. Ejemplo
   exacto del usuario: "no se puede asignar este horario recurrente
   porque 2 de las próximas 12 ocurrencias están completas" +
   detalle de fechas.
2. El admin confirma explícitamente que quiere crear la serie de todos
   modos (o cancela y ajusta).
3. Al confirmar: se crea `RecurringBooking (ACTIVE)`; para cada ocurrencia
   ya materializada se invoca la **misma RPC atómica `book_slot`** de
   ADR-0004 (extendida con `recurringBookingId` opcional) — la recurrencia
   nunca bypasea el `FOR UPDATE` + check de capacidad. `OK → CONFIRMED`,
   `SLOT_FULL → NOT_GENERATED` (no se omite la fila).

**Alternativas evaluadas y descartadas** (de `booking-engine-agent`):
bloqueo duro (no crear la serie si cualquier ocurrencia futura está
llena — demasiado rígido, castiga por algo transitorio); prioridad de
recurrencia/overbooking silencioso (rompe el invariante central de cupo
sin excepción, descartado de plano). Waitlist automática queda fuera de
alcance de Phase 0 (no hay entidad `Waitlist` en el dominio) — mejora
futura, no bloqueante.

**Caso sin aviso previo posible** (ocurrencia todavía no materializada al
crear la serie, se llena antes de que el job de ADR-0009/0011 la genere):
mismo resultado `NOT_GENERATED`, mismo evento de notificación — simétrico
con el caso ya avisado en el preview, un solo mecanismo para ambos.

**Impacto:** `backend-api-agent` implementa el endpoint de preview;
`booking-engine-agent` implementa la RPC de generación inicial reusando
`book_slot`; UI admin debe mostrar la lista de fechas en conflicto antes
de permitir confirmar.

---

## ADR-0013 — `requiresActivePayment`, modelo de Payment y definición de "pago válido"

Fecha: 2026-09-14
Estado: **Aceptada** (extiende ADR-0005/`canCustomerBook()`)
Propuesta por: `domain-architect` + `payments-entitlements-agent`
(cierre de Phase 0) — con una contradicción entre ambos resuelta por el
Orchestrator (ver "Resolución de conflicto" abajo)

**Resolución de conflicto:** `domain-architect` propuso `requiresActivePayment`
en `ServiceEntitlement`; `payments-entitlements-agent` lo propuso en
`Service` (con un workaround de "Payment de cortesía a monto $0" para el
caso de becas). **El Orchestrator adopta la propuesta de
`domain-architect`: `ServiceEntitlement.requiresActivePayment`, no
`Service`.** Motivo: el mismo `Service` puede tener algunos
`ServiceEntitlement` que requieren pago y otros de cortesía/beca (caso ya
listado como válido en `domain.md`) — un flag a nivel `Service` no puede
expresar esa variación por cliente sin un mecanismo de excepción aparte.
El workaround de "Payment ficticio a $0" además contamina `Payment`, que
`domain.md` define como "el movimiento/estado económico" real — un pago
de monto 0 no es un movimiento económico, es una excepción disfrazada de
transacción. La solución de `domain-architect` resuelve el caso de
cortesía de forma directa: ese `ServiceEntitlement` simplemente tiene
`requiresActivePayment = false`, sin tocar `Payment` en absoluto.

**Decisión — semántica del flag** (paso 7 de `canCustomerBook()`,
actualizado en `.claude/agents/payments-entitlements-agent.md`):
- `ServiceEntitlement.requiresActivePayment = false` → el paso 7 se omite
  completamente, el entitlement `ACTIVE` del paso 6 alcanza.
- `= true` y el entitlement es por **vigencia de tiempo** → exige un
  `Payment` válido para el período (ver definición abajo).
- `= true` y el entitlement es por **créditos de uso** → el paso 7 se
  omite igual: el crédito ya implica que el paquete se pagó al comprarse,
  no hay "período de reserva" contra el cual chequear un pago recurrente.

**Decisión — qué significa "pago válido" (lo más importante de esta ADR):**
la validación se hace **contra `SlotOccurrence.startAt`, nunca contra la
fecha en la que se ejecuta la reserva**. Ejemplo del usuario, confirmado
exacto: `Payment{periodStart: 2026-09-01, periodEnd: 2026-09-30, status: PAID}`
permite reservar un slot del 25/09; no existe cobertura para un slot del
05/10 → rechaza, aunque la reserva se haga hoy (14/09, dentro del período
pago de septiembre). El período relevante es el del turno, no el del
momento de reservar — se marca explícitamente porque es el bug más fácil
de cometer al implementar.

```
existe Payment donde:
  Payment.customerId = customer.id
  Payment.serviceEntitlementId = <entitlement resuelto>
  Payment.status = 'PAID'
  Payment.periodStart <= date(slotOccurrence.startAt, org.timezone)
  Payment.periodEnd   >= date(slotOccurrence.startAt, org.timezone)
```

**Modelo de `Payment`:** `{ id, organizationId, customerId, serviceEntitlementId (no serviceId — un ServiceEntitlement puede cubrir varios Service, un pago cubre el entitlement completo), periodStart, periodEnd, status: PAID|PENDING|OVERDUE|VOID, amount?, createdBy, createdAt }`.
`VOID` (no `DELETE`) para un pago cargado por error — mismo criterio que
`Booking` cancelada: nunca se borra, se marca. Gaps entre períodos de
distintos `Payment` (cliente pagó enero, se dio de baja, volvió en abril)
son **estado válido y esperado**, no un error — es lo que hace que el
paso 7 rechace correctamente ese hueco. Se recomienda (no bloqueante,
`database-agent` a implementar si Phase 1 lo prioriza) una
`EXCLUDE USING gist` sobre `daterange(periodStart, periodEnd)` por
`serviceEntitlementId` filtrado a `status='PAID'`, para prevenir
**solapamientos** (indicio de doble carga), distinto de los gaps que sí
son válidos.

**Impacto:** `payments-entitlements-agent` actualiza la firma completa de
`canCustomerBook()` en su archivo de persona; `database-agent` agrega
`ServiceEntitlement.requiresActivePayment boolean not null default true`
y la tabla `payments` con el shape de arriba.

---

## ADR-0014 — Estrategia de timezones

Fecha: 2026-09-14
Estado: **Aceptada**
Propuesta por: `scheduling-agent`, confirmada por `database-agent`
(cierre de Phase 0)

**Problema:** la plataforma es genérica y debe soportar organizaciones en
países con DST, aunque Uruguay no lo tenga. Definir cómo conviven
`ScheduleRule` (hora local) y `SlotOccurrence` (instante real) sin
aritmética de fechas ingenua.

**Decisión:**
- `Organization.timezone`: string IANA (ej. `America/Montevideo`), nunca
  un offset fijo.
- `ScheduleRule.weekday` + `localStartTime`: se mantienen como hora de
  pared local, sin zona — es correcto que sea así, la conversión ocurre
  recién al generar.
- `SlotOccurrence.startAt`/`endAt`: **`timestamptz`**, sin excepción.
  Postgres lo normaliza a instante UTC internamente, permitiendo
  comparar/ordenar/indexar rangos de fecha directamente (ya exigido en
  `database.md`), sin joins ni aritmética manual contra la timezone de la
  organización en cada query.
- **Conversión local→UTC, siempre en SQL, en el momento de generación**,
  vía `(occurrence_date + rule.local_start_time) AT TIME ZONE org.timezone`.
  **Nunca se cachea un offset** — se recalcula por cada fecha puntual de
  la ventana rodante, consultando tzdata para esa fecha específica. Esto
  resuelve DST automáticamente (offsets distintos antes/después de una
  transición, sin lógica especial de `scheduling-agent`) y es la razón
  por la que no se implementa esta conversión en TypeScript en paralelo —
  evita dos implementaciones DST-sensibles que puedan divergir.
- Edge cases de DST (hora local inexistente en el salto de resorte, hora
  ambigua en el retroceso de otoño) se documentan como comportamiento por
  defecto de Postgres/tzdata, sin lógica custom en MVP — no bloqueante
  dado que el primer cliente (Uruguay) no tiene DST.
- `SlotOccurrence.generatedTimezone` (text, nombre IANA): columna
  adicional de trazabilidad, poblada en generación desde
  `Organization.timezone` — no participa del índice/hot path, permite
  reconstruir con qué zona se generó una ocurrencia si la organización
  cambia de timezone después.
- **Renderizado en UI**: siempre en la timezone de la `Organization`, con
  etiqueta explícita (ej. "hora de Montevideo"), tanto en admin como en
  público/cliente — **no** se convierte a la timezone del browser del
  usuario en MVP (el caso dominante es reserva presencial local; convertir
  agrega una clase de bugs peor que el problema que resuelve). Queda como
  mejora futura explícita para servicios remotos cross-timezone.
- Librería TS para formateo/preview (no para la conversión de generación,
  que vive en SQL): `luxon` (o `date-fns-tz` si el resto del stack ya usa
  `date-fns`) — sin necesidad de `Temporal` (no estable en Node LTS aún).

**Impacto:** `database-agent` implementa la conversión `AT TIME ZONE`
dentro de la función de generación de ADR-0009; `scheduling-agent` y
`booking-engine-agent` reusan la misma función/patrón en vez de
reimplementar la conversión (incluyendo la generación de `RecurringBooking`
de ADR-0011).

---

## ADR-0015 — Preservación de booking intent a través del login

Fecha: 2026-09-14
Estado: **Aceptada**
Propuesta por: `auth-security-agent` + `backend-api-agent` (cierre de
Phase 0)

**Problema:** un usuario anónimo arma un intent de reserva
(organization+service+date+slot), tiene que autenticarse, y al volver no
debe reservarse automáticamente — debe revalidarse todo y requerir
confirmación explícita.

**Decisión:**
- **Transporte del intent**: query params planos en la URL de retorno del
  login (`organizationSlug`, `serviceId`, `date`, `slotOccurrenceId`) —
  **sin firmar/cifrar**. No hace falta: son los mismos datos ya públicos
  (servicio, fecha, horario), y el servidor nunca los toma como verdad —
  los vuelve a resolver y validar enteros en el submit. Nunca viaja
  `customerId` ni ningún dato de sesión en el intent (consistente con
  ADR-0005).
- **`returnTo` se valida como destino** (allowlist server-side: path
  relativo, un solo `/` inicial, sin esquema/host, matcheando el patrón
  esperado) — mitiga **open redirect**, el riesgo de seguridad real de
  este mecanismo (distinto del que se preguntó originalmente), no el
  contenido del intent en sí.
- **Confirmación explícita post-login, obligatoria**: la UI re-renderiza
  el resumen del intent (qué se va a reservar, qué consume) y exige un
  clic explícito antes de invocar el submit — mitiga un vector de
  *confused deputy* (alguien manda un link de "continuar tu reserva" con
  un intent que la víctima no eligió), que la sola revalidación de
  backend no cierra por sí misma.
- **Contrato de API — dos endpoints, no uno**:
  ```
  GET  /organizations/:slug/bookings/check?serviceId=&slotOccurrenceId=   [CUSTOMER]
  POST /organizations/:slug/bookings                                      [CUSTOMER]
  ```
  `check` solo invoca `canCustomerBook()` de lectura (sin comprometerse a
  nada) para poder decirle al usuario *antes* de mostrarle "Confirmar" si
  el cupo ya se llenó o si no tiene entitlement, evitando un botón que se
  sabe de antemano que va a fallar. **No es un gate de seguridad, es UX**:
  `POST /bookings` **siempre** vuelve a llamar `canCustomerBook()` de cero
  (consistente con ADR-0005: su resultado nunca se reutiliza entre
  requests) inmediatamente antes de invocar la RPC de ADR-0004 — exacto
  mismo boundary que si `check` no existiera. Ningún endpoint acepta un
  flag "vengo de post-login" que salte o relaje algún paso.
- **Ninguna escritura server-side para usuarios anónimos**: navegar
  `/[organizationSlug]/reservar` o clickear "Reservar" sin estar
  autenticado nunca crea una fila `Booking`/`Customer` en estado
  draft/pending — evita una superficie nueva de datos huérfanos o
  reclamables por terceros.

**Confirmado explícitamente**: no hay forma de bypassear la revalidación
de ADR-0004/ADR-0005 a través de este mecanismo — el peor caso con datos
de intent manipulados es el mismo rechazo (`SLOT_FULL`, cross-org,
sin entitlement) que cualquier intento normal.

**Impacto:** `backend-api-agent` implementa `GET .../bookings/check` +
anida `POST .../bookings` bajo `/organizations/:slug/` (en vez de la ruta
flat `/bookings` que tenía `api.md`, para que `organizationId` se resuelva
sin ambigüedad desde el slug); `auth-security-agent` implementa el
allowlist de `returnTo` en la capa de login.

---

## ADR-0016 — Dos repos separados: `Reservaste/frontend` y `Reservaste/backend`

Fecha: 2026-09-14
Estado: **Aceptada**
Propuesta por: usuario

**Problema:** el código se había scaffoldeado todo en un único directorio
sin separación clara entre frontend y backend, y el usuario pidió
explícitamente repos separados en GitHub bajo el org `Reservaste`.

**Decisión:**
- **`Reservaste/backend`** — Supabase (`supabase/migrations`, config) +
  paquete `@reservaste/domain` (tipos TS, esquemas Zod, invariantes puras
  y mappers snake_case→camelCase). **No es un servicio HTTP separado** —
  el usuario confirmó explícitamente que el modelo de backend sigue siendo
  "Supabase + dominio compartido", sin agregar un servicio API nuevo. Esto
  no cambia ADR-0002: Next.js (en `Reservaste/frontend`) sigue siendo la
  capa de API vía server actions/route handlers, llamando a Supabase
  directamente con supabase-js + las RPC ya definidas.
- **`Reservaste/frontend`** — la app Next.js completa (UI + server
  actions + route handlers), consumiendo `@reservaste/domain` como
  dependencia git (`github:Reservaste/backend`) para no necesitar
  publicar a un registry npm en el MVP.
- El workspace local (`docs/`, `.claude/`, `CLAUDE.md`) queda en un
  tercer directorio de coordinación, **no publicado como repo propio** —
  es el espacio de trabajo de los agentes, referenciado por ambos repos
  pero sin código de producto.

**Impacto:** `frontend/package.json` depende de `@reservaste/domain` vía
`"github:Reservaste/backend"` (o un tag/commit fijado una vez que el
dominio esté más estable, para no romper el frontend con cada push al
backend). CI de cada repo es independiente.

**Migración / compatibilidad:** n/a — reorganización antes de cualquier
release.

---

## ADR-0017 — Proyecto Supabase real conectado

Fecha: 2026-09-14
Estado: **Aceptada**
Propuesta por: usuario

**Decisión:** proyecto `reservaste` (ref `wgdlflhdjpqcxykblqme`,
`https://wgdlflhdjpqcxykblqme.supabase.co`) es el Supabase de referencia.
La migración de Phase 1 ya está aplicada ahí (`supabase db push` vía el
connection pooler — la conexión directa (`db.<ref>.supabase.co`) solo
resuelve por IPv6 y esta red de desarrollo no tiene salida IPv6; el
pooler (`aws-0-us-east-2.pooler.supabase.com`) sí resuelve por IPv4 y es
el que hay que usar para cualquier operación de CLI/CI futura).
`frontend/.env.local` (gitignored, nunca commiteado) apunta a este
proyecto. Verificado con smoke test de solo lectura: `GET
/rest/v1/organizations` sin auth devuelve `[]` (RLS bloqueando
correctamente acceso anónimo) y `/auth/v1/health` responde 200.

**Pendiente:** configurar credenciales de Google OAuth en el dashboard
(Authentication → Providers → Google) — requiere un client ID/secret de
Google Cloud que el usuario tiene que generar, no es algo que se pueda
hacer por CLI. Hasta entonces, el botón "Continuar con Google" del
frontend fallará contra este proyecto (el signup con email/password sí
funciona ya).

---

## ADR-0018 — Reserva fija (standing reservation) asignada desde el mostrador

Fecha: 2026-09-15
Estado: **Aceptada**
Propuesta por: usuario (caso "un cliente me paga mensualmente por todos
los lunes de 09 a 10 de Pilates")

**Problema:** el modelo de recurrencia de Phase 6 (ADR-0010/0011) ya
cubría exactamente ese caso —`ScheduleRule` lunes 09:00 → un
`RecurringBooking` → un `Booking` por ocurrencia, condicionado por
`ServiceEntitlement` + `Payment`— pero sólo como autoservicio del
cliente: `create_recurring_booking()` resuelve el customer desde
`auth.uid()`. Un negocio que le vende un cupo fijo a alguien necesita
armárselo él, y ninguna pantalla llamaba a esas RPC.

**Decisión:**

- **`admin_create_recurring_booking(p_schedule_rule_id, p_customer_id)`**,
  con la misma forma que `admin_book_for_customer` de Phase 8: la
  autorización es **membresía en la organización de la regla**, no
  propiedad de la reserva. ADR-0005 sigue intacto para el camino del
  cliente — lo que cambia no es que el customerId venga del caller, sino
  quién está autorizado a pedirlo.
- **Una reserva fija activa por (cliente, regla).** Una segunda generaría
  un `Booking` duplicado por ocurrencia, que el índice único de Phase 5
  convertiría en una cadena de `NOT_GENERATED`. Se rechaza con
  `ALREADY_HAS_STANDING_RESERVATION`.
- **`evaluate_customer_booking(occurrence, customer)`**: la regla de "¿se
  puede reservar?" se extrae de `can_customer_book()`, que queda como
  "resolvé quién soy y aplicá la regla". El preview del mostrador y el
  del cliente comparten la misma función; duplicarla es precisamente
  cómo se llega a "el preview decía que sí y el confirm dijo que no".
- **Preview obligatorio antes de confirmar** (ADR-0012), ahora también en
  la variante admin: el mostrador ve qué fechas se reservan y por qué las
  otras no, antes de escribir nada.
- **`not_generated_reason` en `bookings`.** `NOT_GENERATED` mezclaba dos
  situaciones muy distintas: la clase estaba llena, o el mes no está
  pago. El portal mostraba "sin lugar" para ambas, que es falso en el
  segundo caso — y el segundo caso es todo el sentido de una reserva
  fija. Se guarda el motivo (`SLOT_FULL` / `NO_ENTITLEMENT` /
  `DUPLICATE`) y se propaga a `my_bookings()`.
- **El cliente ve las fechas que no se confirmaron.** El portal las
  filtraba de "próximas"; para alguien que paga un cupo fijo, que la
  clase desaparezca de la lista es peor que ver por qué no salió.

**Consecuencia deseada:** la serie es autolimitada. Si el mes no se
paga, `resolve_bookable_entitlement()` no devuelve nada y cada lunes
siguiente queda `NOT_GENERATED` — nadie tiene que acordarse de apagar la
reserva fija, y cuando el pago se registra las fechas vuelven a
confirmarse solas. No se agrega ningún estado nuevo de "suspendida".

**Nota de implementación:** al reescribir `generate_recurring_booking()`
para agregar el motivo se perdió el descuento de créditos de Phase 7. Lo
detectó el test de integración de Phase 7 (`13` fechas confirmadas con
`2` créditos), no la revisión de código — el mismo patrón que en casi
todas las fases anteriores.

---

## ADR-0019 — Una fecha no confirmada es una fecha *pendiente*, no decidida

Fecha: 2026-09-15
Estado: **Aceptada**
Propuesta por: Orchestrator (agujero detectado al cerrar ADR-0018)

**Problema:** ADR-0018 dejó la serie autolimitada — si el mes no está
pago, cada fecha futura queda `NOT_GENERATED` en vez de confirmarse.
Faltaba la otra mitad: **nada reevaluaba esas fechas**. Al registrar el
pago seguían pendientes para siempre, y la única salida operativa era
borrar la reserva fija y volver a crearla. El mismo problema aparecía con
la ventana móvil de ADR-0009: las fechas que pg_cron genera se evalúan
con el estado de pago **de ese momento**.

**Decisión:** cualquier escritura que **amplíe** lo que el cliente puede
reservar reevalúa todas sus fechas pendientes futuras.

- **`retry_not_generated_booking(booking)`** hace `UPDATE` de la fila
  existente, no un `INSERT` nuevo. `generate_recurring_booking()` es
  idempotente vía `ON CONFLICT DO NOTHING`, así que volver a correrlo
  sobre una fecha ya registrada es un no-op y nunca revisaría su estado.
  Además la fecha conserva su identidad: el cliente ve **el mismo lunes**
  pasar de pendiente a confirmado.
- **Toma el mismo lock que `book_slot`** (`FOR UPDATE` sobre la
  ocurrencia, ADR-0004): decide si hay cupo, así que no puede competir
  con una reserva simultánea.
- **`reconcile_pending_recurring_bookings(customer, service?)`** recorre
  las fechas **en orden cronológico**. Importa para `CREDITS`: con dos
  créditos y cinco lunes pendientes se confirman los dos próximos, no un
  par arbitrario.
- **Disparadores:** `payments` (insert o update a `PAID`),
  `service_entitlements` insert (alta del permiso) y update — este último
  **sólo** cuando el cambio amplía (`is_active` de false a true, créditos
  que **suben**, `valid_from`/`valid_until` que cambian). Gastar un
  crédito **baja** `credits_remaining` y por eso no dispara: es lo que
  evita que `retry_not_generated_booking()` se reentre a sí mismo a
  través de su propio descuento.
- **El motivo se mantiene honesto aunque siga bloqueada.** Si el mes se
  pagó pero mientras tanto la clase se llenó, la fecha pasa de
  `NO_ENTITLEMENT` a `SLOT_FULL` en vez de quedar con un motivo viejo.
- **Nunca se toca el pasado.** Una fecha ya ocurrida no se revive aunque
  el pago la cubra retroactivamente.
- **Una serie `CANCELLED` no resucita.**

**Lo que deliberadamente NO hace:**

- **Cupo liberado.** Cuando alguien cancela y se abre un lugar, decidir
  quién se lo queda —la reserva fija que quedó afuera, o el primero que
  reserve— es una **política de equidad**, no un bug. Hoy el lugar vuelve
  al calendario público por orden de llegada, que es el comportamiento
  del motor desde Phase 5. **Queda abierto** para decidirlo explícito.
- **Anular un pago no cancela clases ya confirmadas.** Una corrección
  contable no debería sacarle a alguien las reservas que ya tiene; eso lo
  decide una persona.

**Consecuencia para el mostrador:** `registerPayment` cuenta las fechas
pendientes antes y después del insert y dice cuántas se confirmaron. Sin
eso las clases se confirman en silencio en una pantalla que nadie está
mirando.

---

## ADR-0020 — Identidad propia por organización: color y logo

Fecha: 2026-09-15
Estado: **Aceptada**
Propuesta por: usuario ("que cada organización pueda cargar sus propios
colores de templates y poner logos de empresa")

**Problema:** el negocio le muestra su página de reservas a sus propios
clientes. Hoy esa página dice "Reservaste" en todo: el mismo azul y una
inicial en un cuadrado. Para un producto que se cobra, la página tiene
que poder parecerse al negocio.

**Decisión: dos cosas y ninguna más — un color de acento y un logo.**

No un editor de temas. Un tenant que puede tocar toda la paleta es cómo
un producto termina con pantallas ilegibles que después tiene que
soportar. Un color y un logo cubren el 95% de lo que un negocio quiere y
mantienen el resto del diseño bajo control del producto.

- **Los colores semánticos no se tocan.** `success`, `warning` y
  `destructive` significan lo mismo en todos los tenants. Un botón
  "Cancelar" que se vuelve amarillo-marca porque al gimnasio le gusta el
  amarillo es un peligro, no un tema.
- **Dónde aplica:** la superficie pública de reservas (página del
  negocio, flujo de reserva, confirmación) y el panel de la
  organización. La consola de plataforma (`/admin`) queda con la marca
  del producto — es nuestra, no de ellos.
- **El contraste se deriva, no se elige.** `brandTheme()` calcula el
  color del texto sobre el acento por luminancia (WCAG). Si el dueño
  elige un amarillo pálido, el texto sale oscuro solo. La UI avisa
  (`isAccessibleAccent`) pero **no bloquea**: un color incómodo es
  decisión del negocio, lo que no puede pasar es que se entere por sus
  clientes.
- **El formato lo valida la base, no la UI.** El color se interpola en
  una custom property de CSS, así que un string libre ahí es inyección de
  estilos. Un CHECK (`^#[0-9a-f]{6}$`) lo rechaza, con un trigger BEFORE
  que normaliza a minúsculas primero para que pegar "#FFAA00" guarde en
  vez de fallar con un error de constraint ilegible. La columna es
  escribible por PostgREST directo: Zod en el frontend no es una defensa.
- **El logo se guarda como path, no como URL.** Guardar la URL completa
  hornea el host del proyecto en cada fila. Un CHECK obliga a que el path
  empiece con el id de la propia organización, así que un update armado a
  mano no puede apuntar la página de un negocio al objeto de otro.
- **Bucket público, escritura solo del OWNER.** La identidad es una
  decisión de nivel dueño, igual que el nombre. Los límites de tamaño
  (512 KB) y tipo viven en el bucket, donde los aplica el servicio de
  storage — un upload directo con content-type falsificado nunca llega a
  código nuestro.
- **SVG excluido a propósito.** Puede llevar script y la URL del objeto
  se abre directo en el navegador. Permitirlo pondría una superficie de
  XSS almacenado en el host de storage a cambio de un formato que un logo
  no necesita.
- **Nombre de archivo con timestamp**, no `logo.png` fijo: reemplazar el
  logo en el mismo path deja el viejo cacheado en el CDN y el dueño ve
  que su cambio "no tomó".

**Modo oscuro:** hoy nada activa `.dark` — los tokens oscuros son código
muerto del scaffold. Los derivados usan `color-mix` contra `--background`,
así que `--primary-subtle` seguiría funcionando si se enciende; lo único
que habría que resolver es levantar el acento en sí para una marca muy
oscura sobre lienzo oscuro.

**Hallazgo del test de contraste:** el barrido por el círculo de tonos
encontró una banda real (luminancia relativa ~0.18 a 0.24) donde **ni el
blanco ni el casi-negro** (`#111827`) llegan a AA. El negro puro sí
siempre (el cruce está en 4.58). Quedó como fallback: el casi-negro es
más lindo, el negro puro es el que hace que la garantía sea cierta.

---

## ADR-0021 — Autohospedaje en DigitalOcean en vez de Vercel

Fecha: 2026-09-16
Estado: **Aceptada** — reemplaza la parte de hosting de ADR-0002
Propuesta por: usuario (restricción de costo real)

**Problema:** ADR-0002 eligió Vercel cuando no había modelo de negocio.
Hoy lo hay: USD 20/mes por cliente, y **un** cliente. Vercel Pro (USD 20)
más Supabase Pro (USD 25) son USD 45/mes contra USD 20 de ingreso. El
plan Hobby de Vercel no es alternativa: es explícitamente no comercial,
así que usarlo con un cliente que paga es operar fuera de los términos.

**Decisión:** droplet propio en DigitalOcean (USD 6/mes,
`s-1vcpu-1gb`, nyc1) con el **mismo proyecto Supabase en plan free**
(ADR-0017 sigue vigente). Margen positivo desde el primer cliente.

- **TLS no es opcional.** Las cookies de sesión de Supabase Auth viajan
  ahí; sobre HTTP van en texto plano. Por eso el sitio vive en
  `161-35-63-60.sslip.io` y no en una IP pelada: `sslip.io` es DNS
  comodín, resuelve el hostname a esa misma IP sin registrar nada ni
  crear cuenta, y Let's Encrypt le emite certificado como a cualquier
  dominio. Caddy lo saca y lo renueva solo. Es explícitamente para la
  demo: atado a la IP, así que recrear el droplet cambia la URL. Con
  dominio propio es un registro A y un rebuild.
- **Backups propios.** El plan free de Supabase no hace backups, y acá
  viven las reservas y los pagos de los clientes de otro negocio. `pg_dump`
  nocturno con 30 días de retención, escrito a `.tmp` y movido recién al
  completarse: un dump truncado que parece un backup es peor que ninguno.
- **La pausa por inactividad del plan free** no aplica con un cliente
  usándolo a diario. Es un riesgo aceptado, no ignorado.
- **La imagen se construye afuera y se envía.** Las deploy keys están
  deshabilitadas por política de la organización en GitHub, así que el
  droplet no puede clonar el repo privado del paquete de dominio durante
  un build. Construir donde las credenciales ya existen lo evita — y deja
  al droplet sin ninguna credencial de git, que es mejor postura igual.
- **`mem_limit` en el contenedor de la app.** En 1 GB, un contenedor
  desbocado se lleva puesto el sistema entero, `sshd` incluido, y te
  quedás sin forma de entrar a arreglarlo.
- **El droplet no guarda estado.** La base está en Supabase; la imagen se
  reconstruye. Lo único que vive solo ahí son los dumps, que es la
  limitación conocida (ver `frontend/deploy/README.md`).

**Cuándo revisar esto:** con 3 clientes (USD 60/mes), Supabase Pro a USD
25 es el 40% del ingreso y conviene pagarlo para dejar de administrar
backups y pausas. El hosting propio puede seguir siendo el más barato
bastante más allá de eso.

---

## ADR-0022 — El Service pasa a ser la unidad de cobro; se elimina la habilitación manual

Fecha: 2026-09-21
Estado: **Aceptada** — reemplaza a ADR-0013 en su mayor parte
Propuesta por: usuario (iteración funcional: puntos 1, 8, 9, 13, 14, 15, 18)

**El hallazgo:** los puntos 8, 9, 13, 14 y 15 del pedido parecen cinco
cambios y son uno solo. Tomados por separado se contradicen.

Hoy la cadena es `Payment → ServiceEntitlement → (customer, service)`. El
`ServiceEntitlement` es lo que ata un pago a una persona y un servicio, y
además carga `requires_active_payment`. Si se borra sin reemplazo —punto
8— los pagos quedan sin a qué servicio pertenecen: el punto 15
(validar cobertura contra la fecha del slot) se vuelve **imposible de
expresar**, y el punto 9 (mostrar a qué servicio corresponde cada pago)
tampoco se puede responder.

**Decisión:** el pago se ancla directo a `(customer_id, service_id)` y la
configuración económica sube al `Service`.

- **`payments.service_id`**, con backfill desde el entitlement. Es lo que
  hace contestable "¿de qué servicio es este pago?".
- **`services.payment_required`** reemplaza a
  `service_entitlements.requires_active_payment`.
- **`services.billing_type`** (`FREE` | `ONE_TIME` | `MONTHLY`) y
  **`services.billing_cycle`** (`CALENDAR_MONTH` | `ROLLING_MONTH`). Los
  dos ciclos no son una preferencia estética: un gimnasio cobra por mes
  calendario y el entrenamiento personalizado por mes corrido desde el
  pago. Ninguno es "el correcto", por eso es configuración del servicio y
  no una constante del producto.
- **La regla anti doble cobro de ADR-0013 sobrevive, reanclada**: el
  `EXCLUDE` pasa de `(service_entitlement_id, rango)` a
  `(customer_id, service_id, rango)`. Los huecos entre períodos siguen
  siendo válidos a propósito.
- **Lo que de ADR-0013 no se toca y es lo importante**: un pago se valida
  contra **la fecha del slot**, nunca contra "hoy". Eso es exactamente el
  punto 15 y ya estaba probado.

**Quién puede reservar ahora:** cualquier `Customer` activo de la
organización. Sigue haciendo falta ser cliente de ese negocio (alta por
email), así que no es abierto a cualquiera con cuenta — lo que
desaparece es el segundo permiso, por servicio, que un admin tenía que
dar a mano.

**Se eliminan los bonos (`entitlement_type = 'CREDITS'`).** Decisión
explícita del usuario: el modelo del primer cliente es mensual. La tabla
`service_entitlements` queda **deprecada, no borrada**: conserva la
procedencia de los datos históricos y deja de consultarse al reservar.
Si en el futuro hacen falta paquetes prepagos, se diseñan como producto
propio en vez de heredar este modelo.

**Asistencia separada de la reserva (punto 18/22):**
`bookings.attendance_status` (`PENDING` | `PRESENT` | `ABSENT`) es una
columna aparte del `status`. "Reservó y no vino" es una combinación
normal y válida —`CONFIRMED` + `ABSENT`— que un solo campo de estado no
puede expresar. `COMPLETED` **no** implica `PRESENT`: asistir es un hecho
que alguien registra, no una consecuencia de que la clase haya pasado.
Una reserva `CANCELLED` no aparece en la lista para pasar lista, pero
conserva la asistencia que se le haya marcado antes.

**Horarios multi-día (punto 1): una `ScheduleRule` por día, agrupadas.**
Un `weekdays[]` rompería `ScheduleException` —que se identifica por regla
+ fecha—, la generación de ocurrencias y la cascada de
`discontinue_schedule_rule`, y obligaría a repensar las reservas fijas de
ADR-0018. `schedule_rules.group_id` da la vista agrupada ("Lun/Mié/Vie
09:00" como una sola configuración) sin que el modelo finja que es una
sola fila. Todas las reglas existentes recibieron su propio grupo: un
caso "sin agrupar" nulo obligaría a ramificar cada lectura.

**Migración en dos pasos, a propósito.** Esta fase es puramente aditiva:
agrega columnas y las rellena con lo que el entitlement significa hoy, y
no cambia ninguna decisión de reserva. Un trigger puente completa
`payments.service_id` desde el entitlement para que los llamadores viejos
sigan funcionando; muere en la fase que saca el entitlement del camino.
Sin ese puente, poner `service_id` en `NOT NULL` rompía 16 tests de una
vez, que es exactamente lo contrario de una migración en dos pasos.

**Hallazgo lateral, fuera de alcance por ahora:** `book_slot()` no
valida que la ocurrencia no sea pasada — hoy se puede reservar una clase
que ya ocurrió. No es regresión de este cambio, pero con asistencia
(puntos 16-22) pasa a ser visible y conviene cerrarlo.

---

## ADR-0023 — Un solo calendario, tres pantallas; asistencia como hecho aparte

Fecha: 2026-09-21
Estado: **Aceptada**
Propuesta por: usuario (iteración funcional: puntos 1–7, 16–28)

**Decisión: un componente de calendario, no tres.**

`ScheduleCalendar` recibe **datos planos serializables**, no callbacks de
render. Las páginas que lo usan son Server Components, así que cualquier
cosa que cruce el límite tiene que ser serializable — y además obliga a
la pregunta correcta: admin y público difieren en **qué dice cada
etiqueta y a dónde lleva cada link**, no en cómo se dibuja una semana.

- `lib/calendar.ts` concentra la aritmética de fechas, con 15 tests. Los
  que importan son los de timezone: **01:30 UTC del 22 sigue siendo el 21
  en Montevideo**, y equivocarse ahí pone una clase nocturna en el día de
  semana equivocado (ADR-0014).
- La vista Mes abarca semanas completas a propósito, para que la grilla
  sea rectangular: un mes que empieza jueves igual dibuja la primera fila
  entera.
- El rango se trae completo (los 90 días de la ventana rodante, ADR-0009)
  en vez de pedir por vista. Son unos cientos de filas; paginarlo
  costaría un viaje de red por cada flecha para ahorrar bytes que nadie
  está contando.

**El público NO es el calendario de admin achicado.** En un teléfono es
una tira de días más una lista; la grilla semanal aparece solo donde hay
lugar. Siete columnas de una grilla horaria en 390px no se usan. Lo que
un visitante necesita es "qué puedo tomar y si hay lugar", no una
reproducción fiel de una app de calendario. Por eso en cada tarjeta **la
hora es lo más grande**: elegir un turno es elegir una hora, el nombre
del servicio confirma.

**El filtro por servicio es solo presentación.** Nunca cambia una
consulta: un servicio oculto no puede confundirse con uno cancelado. Se
guarda en `sessionStorage`, y si el almacenamiento falla el filtro se
olvida en vez de romper la pantalla.

**Dos destinos en un mismo bloque.** El título lleva al servicio, el
bloque lleva al turno (punto 7). El link del título corta la propagación
del click; sin eso uno de los dos gestos queda inalcanzable.

**Pasar lista es una pantalla, no un diálogo.** Objetivos de 56px de
alto, dos por persona, ocupando media fila cada uno. Esto se hace con el
pulgar, de pie, en la puerta de una clase. Es optimista a propósito:
esperar un viaje de ida y vuelta entre un nombre y el siguiente se siente
roto aunque funcione.

**La asistencia no se deduce.** Una reserva `CANCELLED` no aparece en la
lista pero conserva la asistencia que se le haya marcado antes; "nadie
pasó lista" (`PENDING`) se muestra distinto de "no vino" (`ABSENT`)
porque no son lo mismo. `mark_attendance()` rechaza una reserva que no
esté `CONFIRMED`: marcar asistencia de alguien que canceló es un error de
carga, no un flujo.

**Horarios multi-día: una acción, N reglas.** Ver ADR-0022 — el modelo
sigue siendo una `ScheduleRule` por día de semana. Un grupo de Lun/Mié/Vie
tiene **tres** reservas fijas posibles, y eso es correcto: un lunes fijo
no es un miércoles fijo.

**Pagos: la pantalla responde una pregunta.** Quién pagó y quién no, por
mes. El filtrado y la búsqueda son del lado del cliente porque la lista
es "los clientes de un negocio" y un viaje de red por tecla se sentiría
más lento de lo que es. El estado `NO_PAYMENTS` existe justamente para el
caso que la pantalla viene a destapar: alguien sin ningún pago en el
período.

**Lo que quedó sin cubrir:** las pantallas nuevas no tienen tests de
integración propios — lo probado son las RPC que las alimentan. Tampoco
pude verlas renderizadas con sesión iniciada: no tengo credenciales de
ninguna cuenta del proyecto real.

---

## ADR-0024 — `ServicePlan`: un pago mensual compra cupos, no acceso ilimitado

Fecha: 2026-09-21
Estado: **Aceptada**
Propuesta por: usuario (feedback del primer cliente real: planes con precio y
veces por semana, cupo fijo, "sin cupo siempre pagás")
Diseño: `domain-architect`, propuesta completa en
`proposals/adr-0024-service-plan.md`. Aprobada por el Orchestrator con las
siete resoluciones de más abajo.

**El hallazgo:** los pedidos de planes, frecuencia semanal, cupo fijo, recupero
y "sin cupo pagás" parecen cinco cambios y son **uno**.

Hoy `payment_covers_slot()` dice: si existe un `Payment` PAID de este cliente
para este servicio cuyo período cubra la fecha del turno, puede reservar
**cuantos turnos quiera** de ese servicio en ese período. El primer cliente
describe otra cosa: el pago mensual compra **N cupos fijos por semana**, y lo
que exceda se paga o se recupera (ADR-0025). Sin esa inversión los pedidos se
contradicen entre sí: un plan de 1x semana no significa nada si el mismo pago
ya habilitaba los 5 días, y un crédito de recupero no vale nada si reservar de
más nunca costó.

### Decisión: `service_plans` es la lista de precios que la organización ofrece a sus clientes

No `plans`, que ya son los planes del SaaS que paga la organización (Phase 10).

- `plan_kind` = **`DROP_IN`** (clase suelta) | **`WEEKLY_QUOTA`** (N cupos
  fijos por semana) | **`UNLIMITED`** (el comportamiento de hoy).
- `billing_type`/`billing_cycle` **suben del `Service` al plan**. Es la
  corrección más importante del diseño: un mismo servicio tiene ahora un plan
  `ONE_TIME` y tres `MONTHLY`, que una sola columna por servicio no puede
  expresar. Sin esto quedan dos fuentes de verdad para "qué período compra un
  pago" y divergen el primer día que un servicio tenga un plan calendario y
  otro corrido. `services.billing_type`/`billing_cycle`/`price` quedan
  **deprecadas**; `services.payment_required` **no** se toca — es el
  interruptor de "este servicio exige pago", no un precio.
- Caso pilates completo: **un solo** `Service` con capacidad 3 y cuatro planes
  (DROP_IN 800; WEEKLY_QUOTA 1/2/3 → 1800/2500/3400). Modelarlo como cuatro
  servicios partiría la capacidad de 3 en cupos separados, que es exactamente
  lo que el cliente no quiere.
- `UNLIMITED` existe para que **nadie cambie de comportamiento el día del
  deploy**: el backfill le da a cada servicio con pagos un plan `UNLIMITED` con
  su configuración actual. Mismo patrón aditivo que ADR-0022, que es el que no
  rompió 16 tests de una vez.

### "Veces por semana" **es** "cantidad de series", no una aproximación

Cada `ScheduleRule` es semanal por construcción (`weekday` +
`local_start_time`), cada `RecurringBooking` es la suscripción a **una** regla,
y `ALREADY_HAS_STANDING_RESERVATION` impide dos series sobre la misma regla.
Entonces *k* series producen exactamente *k* reservas por semana.

Consecuencias, y son la mitad del valor de esta decisión:

- **No se persiste ningún contador.** Nada que decrementar, restituir al
  cancelar ni reconciliar — toda la clase de bugs que ADR-0022 eliminó al
  borrar `CREDITS` no vuelve a entrar por esta puerta.
- `availableCapacity = maxCapacity - activeBookings` no se toca.
- **La pregunta "semana calendario o semana corrida" se disuelve**: nunca se
  calcula una ventana semanal, así que no hay nada que anclar.

Tres precisiones sin las cuales la implementación sale mal:

1. **`status = 'ACTIVE'` sobrecuenta.** ADR-0010 dejó `RecurringBooking.status`
   como expresión de intención, no agregado de sus hijas: una serie con
   `end_date` en junio sigue `ACTIVE` en septiembre. La cuota se mide contra
   las series **en vigencia para la fecha evaluada** (`ACTIVE` +
   `start_date <= D` + `end_date is null or >= D`).
2. **`D` es la fecha local del slot, nunca `now()`** — la lección de ADR-0013
   aplicada a la cuota. Una serie que arranca el mes que viene no consume cuota
   hoy, pero sí para un turno del mes que viene.
3. **Desempate determinístico** `created_at asc, id asc` cuando hay más series
   que cuota: entran las primeras que el cliente contrató. Estable, explicable
   en el mostrador, y no obliga a nadie a elegir cuál se cae.

`RecurringBooking` **no gana ninguna columna**: el servicio se alcanza por
`schedule_rule_id → schedule_rules.service_id`. Copiarlo sería la segunda
fuente de verdad que ADR-0010 prohibió para día/hora.

### Dos puertas para la cuota, con pesos distintos

- **Blanda, al crear la serie**: se rechaza la N+1 con `OVER_PLAN_QUOTA` sólo
  si el cliente **tiene** un pago en vigencia con plan de cuota. Si no tiene
  ninguno, la creación **no** se rechaza: ese es el flujo de mostrador de
  ADR-0018/0019 (se arma la reserva fija y se cobra después) y romperlo sería
  una regresión.
- **Dura, por fecha, al generar**: es la única que puede ser exacta, porque
  recién ahí "qué plan pagó ese mes" tiene respuesta definida. La fecha fuera
  de cuota queda `NOT_GENERATED` con
  `not_generated_reason = 'OVER_PLAN_QUOTA'`.

No puede ser un constraint: es un agregado que cruza `recurring_bookings →
schedule_rules → services` contra un `weekly_quota` al que se llega por
`payments → service_plans`. Hoy toda escritura de `recurring_bookings` pasa por
RPC (la RLS es SELECT-only), así que el chequeo en la RPC alcanza.

### Sigue habiendo **una sola** función de decisión

`payment_covers_slot()` y `evaluate_customer_booking()` ganan contexto de serie
como cuarto parámetro opcional (`default null` significa literalmente "esta
reserva no viene de una serie"), con tres formas: sin serie, serie existente y
**serie prospectiva** (la que `preview_recurring_booking()` necesita antes de
que la serie exista, evaluada como la k+1-ésima).

> **Regresión concreta que esto evita, verificada en el código:**
> `admin_preview_recurring_booking` llama a `evaluate_customer_booking()`
> ocurrencia por ocurrencia en forma de reserva puntual
> (`phase11_standing_reservations.sql:329`). Agregarle la cuota sin hilar el
> contexto de serie haría que **el preview de una reserva fija diga "fuera de
> tu frecuencia" en todas las fechas**, justo al cliente que tiene el plan que
> la permite.

Un pago de clase suelta se consulta **antes** que todo lo demás: una clase
pagada está pagada, independientemente del estado de cualquier plan.

**Motivos nuevos de `can_book_reason`** — `PAYMENT_REQUIRED` no alcanza porque
**le diría "tenés que pagar" a alguien que acaba de pagar**, el mismo error que
ADR-0018 y Phase 12 ya tuvieron que corregir:

| Valor | Significa | Remedio |
|---|---|---|
| `PAYMENT_REQUIRED` (existente) | Ningún pago cubre esa fecha | Cobrale el mes |
| **`OUTSIDE_PLAN_QUOTA`** | El mes está pago, pero esta reserva no viene de uno de sus horarios fijos | Clase suelta, crédito de recupero (ADR-0025), o esperar |
| **`OVER_PLAN_QUOTA`** | Esta **serie** excede la frecuencia del plan | Subir de plan o dar de baja una serie |
| **`SERVICE_HAS_NO_PLAN`** | El servicio exige pago y no tiene plan activo | Error de configuración del dueño, no del cliente |

`OUTSIDE_PLAN_QUOTA` y `OVER_PLAN_QUOTA` no son el mismo mensaje: cambian de
actor (el cliente intentando de más vs. el mostrador habiendo vendido de más),
de pantalla y de remedio. `not_generated_reason` suma sólo `OVER_PLAN_QUOTA`
(una fecha `NOT_GENERATED` siempre viene de una serie).

### El `EXCLUDE` se re-ancla, y le faltaba una mitad

Hoy el `EXCLUDE` impide dos `Payment` PAID con períodos solapados del mismo
`(customer, service)`. Alguien con plan mensual que compra una clase suelta cae
dentro de su propio período y el insert se rechaza.

Se resuelve anclando el pago suelto a la `SlotOccurrence`
(`payments.slot_occurrence_id`) en vez de a un período — más fuerte que un
período de un día, porque dice **qué clase** se pagó. Pero excluirlos del
`EXCLUDE` los deja sin **ninguna** protección de doble cobro: hace falta además
`unique index (customer_id, slot_occurrence_id) where status = 'PAID' and
slot_occurrence_id is not null`.

`payments.service_id` **se mantiene** como columna derivada y verificada por
trigger (no segunda fuente de verdad): el `EXCLUDE`, sus índices y cuatro
funciones de lectura están cableados por servicio, y sigue siendo el grano
correcto de "quién pagó este mes", que es todo el punto de ADR-0022.

### Las siete resoluciones del Orchestrator

1. **Cambio de plan a mitad de período: VOID + recargar** (con el usuario).
   Queda **un solo pago vigente** por `(customer, service)`, así que nunca hay
   que inventar una regla de "cuál de los dos planes manda" dentro del motor de
   reservas. Una regla así se equivoca en silencio. El `EXCLUDE` no se relaja.
2. **Moneda: `organizations.currency`** (ISO 4217, default `UYU`, con el
   usuario). Cierra también el hueco preexistente de `services.price`.
3. **`MONTHLY_QUOTA` / paquete por cantidad queda fuera de alcance.** Es el
   prepago que ADR-0022 eliminó por decisión explícita; reabrirlo metería un
   segundo camino de cuota en la única función de decisión, por un caso que
   ningún cliente pidió. (De paso corrige el ejemplo del consultorio que yo
   había usado en `iteration-3-plan.md` como prueba de genericidad: "4
   consultas al mes" es un contador sobre un período, no un conjunto de
   horarios. Lo genérico es turno fijo semanal + consulta suelta.)
4. **Bajar de plan NO cancela series.** Las excedentes se autolimitan
   (`NOT_GENERATED` / `OVER_PLAN_QUOTA`) y ADR-0019 las reconstituye si el
   cliente vuelve a subir. Cancelar series por un cambio de plan destruiría la
   agenda del cliente de forma irreversible a partir de un hecho
   administrativo.
5. **`plan_kind` y `weekly_quota` son inmutables** una vez que el plan tiene
   pagos no-`VOID`; **`price` es libre**. Ese corte es el criterio: el precio
   sólo afecta cobros futuros y cada `Payment` guarda su propio monto, mientras
   que los términos tienen que poder resolverse hacia atrás desde un pago
   vigente. Sin `payments.weekly_quota_snapshot`: duplicaría la verdad justo
   para las lecturas que ya joinean contra el plan. Corregir un error de carga
   = desactivar el plan y crear otro, que no corta cobertura.
6. **Un servicio con `payment_required = false` no puede tener planes de
   cuota** (trigger). Sin pago vigente no hay de dónde resolver la cuota:
   aparentaría estar vigente y sería inaplicable.
7. **La firma exacta del contexto de serie es detalle de `database-agent`.** Lo
   no negociable es que siga habiendo una sola función de decisión (ADR-0018).

### Invariantes nuevos

- **Desactivar un plan no invalida un pago ya hecho.** Un
  `join service_plans ... and is_active` le cortaría la cobertura a todos el
  día que el dueño ordena su lista de precios. `payment_covers_slot()` lee
  `plan_kind` **sin** filtrar por `is_active`.
- La cuota se mide en series en vigencia para la fecha del slot, nunca en una
  ventana semanal.
- Una reserva extra no la cubre un pago por período de un plan de cuota.
- Un solo plan `DROP_IN` activo y un solo `UNLIMITED` activo por servicio: el
  primero es el precio que el flujo "pagá esta clase" cobra solo, el segundo es
  a lo que el puente de migración ancla un pago sin plan explícito.

### Restricción de secuenciación (vinculante para la Fase H)

La migración y la pantalla de pagos con selector de plan **salen en la misma
fase**. El formulario actual inserta en `payments` por PostgREST directo sin
saber de planes; el puente de compatibilidad lo sostiene sólo mientras exista
un `UNLIMITED` activo, así que en cuanto el dueño desactive ese plan para
dejar sólo los de cuota, el mostrador no puede registrar un pago. El puente
falla ruidoso (`PAYMENT_REQUIRES_PLAN`) en vez de dejar un pago sin plan: un
pago que el camino de decisión tuviera que interpretar como "ilimitado por las
dudas" es la peor de las dos opciones.

### Alternativas descartadas

- **Un `Service` por plan** ("Pilates 1x", "Pilates 2x"): parte la capacidad
  de 3 personas en paralelo en cupos separados, que es justo lo que el cliente
  pidió que no pase.
- **`weekly_quota` como contador persistido y decrementado por reserva**:
  reintroduce toda la clase de bugs de `CREDITS` (decrementar, restituir,
  reconciliar) para expresar algo que las series ya expresan exactamente.
- **Relajar el `EXCLUDE` para pagos de cuota** (en vez de VOID + recargar):
  obliga a una regla de precedencia entre dos pagos vigentes dentro del motor
  de reservas.
- **`payments.weekly_quota_snapshot`**: segunda fuente de verdad para lecturas
  que ya joinean contra el plan.

### Riesgo principal

El camino de decisión es el código más cargado del producto y esta migración
lo toca en ocho funciones. La mitigación no es revisión: son los tests de
concurrencia y de fecha que ya existen (`book_slot` con capacidad 1, pago
validado contra la fecha del slot y no contra hoy) más los que la Fase H tiene
que sumar para cuota fuera de vigencia, preview de serie prospectiva y pago
suelto con plan mensual activo simultáneo.

---

## ADR-0025 — Crédito de recupero: liberar un cupo y recuperarlo dentro del mes

Fecha: 2026-09-22
Estado: **Aceptada**
Propuesta por: usuario (feedback del primer cliente real)
Diseño: `domain-architect`, propuesta completa en
`proposals/adr-0025-makeup-credits.md`. Aprobada por el Orchestrator con las
nueve resoluciones de más abajo, **y con cinco correcciones al encuadre que
yo mismo había escrito mal** — las tres primeras verificadas contra el schema
antes de aceptarlas.

**El pedido, textual del cliente:** quien tiene cupo fijo y avisa con
anticipación liberando su lugar puede recuperarlo dentro del mismo mes; el
cupo liberado se ve en la agenda pública; quien reserva se valida como
candidato; y el que no tiene cupo siempre paga.

### Lo que yo tenía mal, y por qué importa

**1. "Se consume bajo el mismo lock que el cupo" era falso.** El `FOR UPDATE`
de ADR-0004 lockea **la fila de la ocurrencia**. No serializa nada cuando el
**mismo cliente reserva dos ocurrencias distintas a la vez**, que es
exactamente el caso que gastaría el crédito dos veces: dos filas, dos locks,
cero conflicto. El crédito necesita su propia exclusión (`UPDATE ... WHERE
status = 'AVAILABLE'` verificando filas afectadas). Peor: **el test que yo
declaré obligatorio no probaba lo que decía** — dos reservas simultáneas
sobre la *misma* ocurrencia las rechaza `ALREADY_BOOKED` o la capacidad, y el
crédito no se mira nunca. El test tiene que ser sobre **dos ocurrencias
distintas**.

**2. `cancel_booking()` deja que el cliente elija el motivo de su propia
cancelación** (verificado: `p_reason` con default, granted a `authenticated`,
guardado tal cual). Hoy eso es un dato sucio. Con créditos, es **un crédito
gratis**: cancelo diez minutos antes pasando `'SLOT_CANCELLED'` y me llevo el
crédito que la anticipación existía para negarme. Sin arreglar esto, la
política de aviso es voluntaria.

**3. La regla de emisión que yo describí es una máquina de imprimir
créditos.** `cancel_recurring_booking()` cascadea a las hijas con
`CUSTOMER_REQUEST` (verificado). Cada fecha futura cumple literalmente "vino
de serie activa, dentro de cuota, con anticipación". El exploit completo: el
día 1 cancelo la serie y cobro cuatro créditos; recreo la serie sobre la
misma regla (la vieja quedó `CANCELLED`, así que no cuenta para la cuota);
repito. Plan de 1×semana, clases ilimitadas — justo lo que ADR-0024 vino a
impedir.

**4. Los motivos nuevos que pedí ya existen.** `PAYMENT_REQUIRED` vs.
`OUTSIDE_PLAN_QUOTA` los creó ADR-0024. **ADR-0025 no agrega ningún motivo de
rechazo.** Lo que sí falta es un **veredicto positivo que diga con qué entra**
("con tu crédito, vence el 30/09").

**5. Faltaba el interruptor.** Sin él, todas las organizaciones empiezan a
emitir créditos el día del deploy y se les afloja sola la cuota que acaban de
comprar con ADR-0024.

### Decisión

**`makeup_credits`**: `origin` (`CUSTOMER_RELEASE | ORGANIZATION_CANCELLED |
MANUAL`), `expires_on` **date congelada al emitir**, `status`
(`AVAILABLE | CONSUMED | REVOKED`). RLS **`SELECT`-only, cero policies de
escritura**: un crédito es dinero, y sólo nace de una RPC.

- **`unique(source_booking_id)`** es el invariante anti-inflación, y es
  estructural en vez de una validación que alguien recuerde: una reserva
  liberada emite **un** crédito, para siempre.
- **`unique(consumed_booking_id) where status = 'CONSUMED'`** (parcial) para
  que un crédito devuelto conserve el rastro y pueda reusarse.
- **El vencimiento es derivado, no un job y no un estado persistido** — mismo
  criterio que `COMPLETED` de `SlotOccurrence` (ADR-0010). Y simplifica la
  devolución: el crédito **vuelve siempre**, sin la condición "si no venció".
- **`expires_on` se ancla a la fecha de la ocurrencia liberada, no a
  `issued_at`**: con `DAYS_AFTER`, avisar tres semanas antes te quemaría la
  ventana antes de que la clase ocurra.
- **Se compara contra la fecha local del turno, nunca contra `now()`** —
  "recuperar dentro del mes" es que la clase *ocurra* dentro del mes. Es la
  lección de ADR-0013 otra vez.

**Emisión — la regla es más simple que la que yo había escrito, y por eso es
segura:** sólo emiten la cancelación de **una fecha puntual** y las
cancelaciones **originadas por la organización**. Una cascada de serie **no
emite nunca, la pida quien la pida**. "Vino de una serie dentro de cuota" no
se reimplementa: se re-evalúa la cobertura de ADR-0024 y tiene que dar `OK`,
lo que además cierra gratis el caso "libero un mes que no pagué" — `CONFIRMED`
ya implicaba que estaba pago.

**Corrección del agente a mi punto 4, aceptada:** "no se emite para una
reserva suelta" vale para `CUSTOMER_REQUEST` y **no** vale cuando cancela la
organización. Si no, el que pagó 800 por una clase que el estudio canceló
queda sin remedio y con un `Payment PAID` anclado a una ocurrencia cancelada
que el índice único de ADR-0024 le impide reusar.

**Consumo — última compuerta, después de toda la cadena de ADR-0024.**
Invariante: **nunca se gasta un crédito si otra cobertura alcanzaba**. Rescata
`OUTSIDE_PLAN_QUOTA`, `PAYMENT_REQUIRED` y `SERVICE_HAS_NO_PLAN`; elige el de
vencimiento más próximo con orden total; y **los jobs nunca consumen**
(`generate_recurring_booking` / `retry_...` operan sobre fechas que ADR-0019
define como pendientes — un job no gasta el crédito de nadie). Atómico: si el
`UPDATE` condicional afecta cero filas se aborta todo, nunca `OK` después de
una escritura parcial.

**El veredicto positivo no puede ser un valor del enum.** Seis funciones
comparan `= 'OK'`, así que un `OK_WITH_MAKEUP_CREDIT` sería un "sí" que media
docena de caminos leería como "no", en silencio. Va como veredicto compuesto
`(reason, makeup_credit_id)`. El criterio: tiene que romper en `db reset`, no
en producción.

**Agenda pública:** un booleano `recently_released`, **sin número, sin actor,
sin timestamp y sin reordenar la lista**, suprimido en modo `BOOLEAN` y con
capacidad 1 (ADR-0007 en su forma pura). Que *fulano* liberó su lugar es dato
privado.

**Turno suelto:** `quote_booking()` (lectura, envuelve el mismo resolutor) y
`book_slot_paying()` (atómica, bajo el mismo lock, con el plan y el precio
resueltos en la base y **nunca** tomados del cliente, y que **no cobra si ya
está cubierto**). Funciona hoy con carga manual del ADMIN y el webhook de
ADR-0027 se enchufa encima sin tocar el motor.

**Portal:** `my_services()` deja de devolver `price`/`billing_*` deprecados
—que después de ADR-0024 pueden ser información falsa— y pasa a exponer el
plan vigente, cupos usados y comprados, y los planes activos.
`my_makeup_credits()` con `is_expired` calculado en SQL.

### Las nueve resoluciones del Orchestrator

1. **`cancel_booking()` cambia de contrato: el motivo se deriva del actor**, no
   se acepta de un caller `authenticated`. Es un cambio de contrato de API y
   por eso es mío. Sin esto la política de aviso es decorativa.
2. **Se agrega `SERIES_CANCELLED`** a `booking_cancellation_reason`. Hoy una
   serie cancelada por staff estampa `RULE_DISCONTINUED` y la regla no se
   discontinuó: el motivo miente, y ADR-0010 creó ese campo justamente para
   poder notificar y reportar con precisión.
3. **`makeup_credits_enabled`, default `false`**, opt-in del dueño. Los
   defaults de política (12 h, `END_OF_MONTH`) siguen siendo los del usuario y
   aplican una vez encendido. Mismo patrón que el `UNLIMITED` de ADR-0024:
   el día del deploy no cambia el comportamiento de nadie.
4. **Tercer valor `END_OF_BILLING_PERIOD`** para el vencimiento. ADR-0022
   soporta mes calendario y mes corrido a propósito porque ninguno es "el
   correcto"; forzar el crédito a mes calendario reintroduciría justo el
   desajuste que esa decisión evitó.
5. **`origin = 'MANUAL'` entra en la Fase I**, OWNER-only y auditado. Es el
   único remedio de media docena de casos límite, y sin él la salida es
   editar la base a mano.
6. **Contratos de API aprobados**: `my_services()` sin columnas deprecadas, y
   `get_public_availability()` con la columna `recently_released` y sus dos
   supresiones.
7. **Firma del veredicto compuesto: detalle de `database-agent`**, con lo no
   negociable escrito arriba (una sola función de decisión; el qualifier no es
   un valor del enum).
8. **La pregunta del cupo que se llena entre checkout y webhook se difiere a
   ADR-0027**, con una restricción ya fijada: si la solución introduce un
   estado nuevo de `Booking`, no puede romper
   `availableCapacity = maxCapacity - activeBookings`. Hoy no hay hueco
   porque el pago lo carga el admin.
9. **La ventana de 72 h de `recently_released` la revisa
   `auth-security-agent`** contra ADR-0008 antes de cerrar la fase.

### Corrección preexistente que este trabajo destapó

`domain.md` describe `Booking.cancelledBy` como un enum `CUSTOMER |
ORGANIZATION`. El schema lo tiene como **FK a `Profile`** desde Phase 5, que
además documenta por qué: el actor categórico ya lo carga
`cancellation_reason`. El documento estaba desactualizado, no el schema. Se
corrige en `domain.md`.

### Secuenciación (vinculante)

Migración, `quote_booking()` y la pantalla "pagá este turno" salen **juntas**.
Si no, el cliente ve `OUTSIDE_PLAN_QUOTA` sin forma de resolverlo, con una
tabla de créditos vacía al lado prometiéndole otra cosa. Mismo criterio que la
Fase H.

### Riesgo abierto, no técnico

Cuántos créditos vivos tolera la capacidad real de un negocio: 20 créditos
contra 3 lugares por clase es un reclamo en el mostrador, no un bug. Se
mitiga mostrando "créditos vivos por servicio y mes" en el panel, para que el
dueño lo vea venir. Si con uso real resulta insuficiente, la palanca siguiente
es un tope de créditos por cliente y mes — no se construye todavía.

---

## ADR-0028 — `GRANT EXECUTE` es aditivo: PUBLIC nunca se revocó

Fecha: 2026-09-22
Estado: **Aceptada** — corrección de seguridad, ya desplegada
Encontrado por: `auth-security-agent`, al diseñar ADR-0026. Implementado por
`database-agent` en `20260922160000_phase19_security_fixes.sql`.

**El problema, verificado en vivo contra producción antes de aceptar el
reporte:** un request sin ninguna sesión —solo la clave pública `anon`—
invocó `generate_all_slot_occurrences()` contra la base real y recibió
`204`. La causa: **`GRANT EXECUTE ... TO authenticated` suma un permiso, no
reemplaza el default.** Postgres/Supabase deja `EXECUTE` en PUBLIC salvo que
se revoque explícitamente, y en ninguna migración de las Fases 1 a 18 se
revocó. Los ~38 `grant` que sí se escribieron eran documentación de intención,
no una restricción — **toda RPC de negocio del proyecto era invocable por
PostgREST sin sesión**, incluida `book_slot`.

**Por qué no se vio en los tests hasta ahora:** los tests llaman a las RPC
autenticados, como se supone que se usan. El agujero solo aparece contra el
rol `anon` real, que ningún test ejercitaba sobre estas funciones — al revés
de las RLS de tabla, donde sí hay un test con cliente anónimo desde Phase 4.

### Decisión

Regla nueva y no negociable para toda función `SECURITY DEFINER` de acá en
adelante: **`REVOKE EXECUTE ... FROM PUBLIC, anon` explícito, antes o en el
mismo `GRANT` que la habilita.** No alcanza con "esta función ya chequea
autorización adentro" — la falla tiene que ser en dos capas.

Tres categorías, con trato distinto:

- **Helpers internos** (solo los llama otra función `security definer`):
  revocados de todo rol externo. Adentro de un `security definer` el
  privilegio se resuelve contra el dueño de la función, así que revocarlos no
  rompe nada y los saca de la superficie de PostgREST.
- **Funciones correctas por construcción** (`is_organization_member`,
  `is_organization_owner`, `is_platform_admin`): **quedan con PUBLIC a
  propósito**. Las expresiones de RLS se evalúan con el rol que consulta, y
  `anon` las evalúa en cada lectura del calendario público — revocarlas
  rompe la página pública. Solo contestan sobre `auth.uid()`, nunca revelan
  datos de terceros.
- **RPC de negocio**: `REVOKE FROM PUBLIC, anon` + `GRANT TO authenticated`
  explícito. `get_public_availability` y `public_slot_detail` son la
  excepción declarada: el calendario público es sin login por diseño
  (ADR-0008), así que mantienen `anon`.

**Estado verificado en `pg_proc` después del fix:** `anon` solo puede
ejecutar 6 funciones — los 4 helpers de policy y las 2 del calendario
público. `authenticated` mantiene exactamente las RPC de negocio.

### Corrección de contrato que viajó en la misma migración

Mientras se tocaba `cancel_booking()` para blindar la autorización con
`profile_id` potencialmente `NULL` (resolución 1 de ADR-0025, ya aceptada),
se cerró su contrato: **el motivo de cancelación deja de ser un parámetro del
caller**. Aceptarlo era, con ADR-0025 encima, un crédito de recupero gratis
(cancelar tarde pasando `SLOT_CANCELLED`). `cancel_booking(uuid)` sin
segundo parámetro; el motivo se deriva del actor dentro de la función.

**Dos llamadores del frontend usaban la firma vieja y fallaban en silencio**
(`app/actions/customer.ts`, `app/actions/admin.ts`) — ninguno revisaba el
error de la RPC, así que ya era un bug preexistente independiente de esta
migración. Se corrigió el parámetro; el manejo de error sigue pendiente y
queda anotado como `TODO(Fase L)` en el propio código, porque agregarlo ahora
sin un `error.tsx` en el árbol cambiaría el comportamiento de fallo sin
haberlo verificado — eso es trabajo de UI, no de un parche de seguridad.

### Riesgo que queda abierto, con decisión pendiente del Orchestrator

`slot_occurrences` **no tiene policy de `DELETE`**. El trigger que debería
borrar ocurrencias futuras al editar una `ScheduleRule`
(`trigger_regenerate_on_schedule_rule_update`) es `SECURITY INVOKER`, así que
bajo RLS **no borra nada** cuando el staff edita una regla por PostgREST —
solo se agregan ocurrencias nuevas por `ON CONFLICT DO NOTHING`. Pasarlo a
`SECURITY DEFINER` sería el arreglo de permiso correcto, pero entonces sí
borraría de verdad, **incluidas ocurrencias con reservas activas** — que es
exactamente lo que un comentario de Phase 3 ya marcaba como pendiente de
revisión y ninguna fase revisó. No se toca en esta migración: necesita una
decisión de producto (¿qué pasa con las reservas de una ocurrencia que el
cambio de horario deja huérfana?), no solo un cambio de permiso.

### Migración

`20260922160000_phase19_security_fixes.sql`. 19 migraciones aplican desde
cero. 139 tests de integración (8 nuevos, uno de ellos reproduce el bug con
`profile_id = NULL` contra una base con el `NOT NULL` bajado temporalmente),
35 unitarios de dominio, 36 de frontend. Desplegada y verificada en vivo
contra el proyecto real antes y después (`204` sin auth → `401 permission
denied`).

---

## ADR-0026 — Cliente gestionado + activación por WhatsApp

Fecha: 2026-09-22
Estado: **Aceptada**
Propuesta por: usuario (feedback del primer cliente real: alta de clientes
que no pueden auto-registrarse, activación por teléfono vía WhatsApp)
Diseño: `auth-security-agent`, propuesta completa en
`proposals/adr-0026-managed-customers.md`. Aprobada con las ocho
resoluciones de más abajo.

**El pedido:** el dueño da de alta a un cliente por nombre y teléfono sin que
tenga cuenta; un link de WhatsApp arma la activación; el que activa queda
vinculado a su historial. Hoy es imposible: `customers.profile_id` es
`NOT NULL` y `enroll_customer_by_email()` falla si la persona no tiene cuenta.

### El repaso de RLS que pedí, hecho — el problema no estaba donde yo temía

Encargué explícitamente un repaso **policy por policy** de las 29 policies del
proyecto contra `profile_id` nullable, no un razonamiento en abstracto.
Resultado: **el `NULL` no abre ninguna policy** — las cinco de dos capas de
ADR-0006 usan `= auth.uid()` en un `USING` o un `EXISTS`, y `NULL` ahí da
`NULL`, que RLS trata como no visible. El problema estaba en **plpgsql**, no
en RLS, y es más grave de lo que yo esperaba: `cancel_booking()`
(`v_customer.profile_id <> auth.uid() and not is_organization_member(...)`)
da `NULL` con `profile_id IS NULL`, y un `if NULL` **no entra al raise** —
saltea la autorización entera. Hoy no era explotable porque la columna es
`NOT NULL`; **esta ADR era exactamente lo que lo iba a activar**. Ya se
cerró en la migración 19 (ADR-0028), antes de que `profile_id` se vuelva
nullable — el orden importaba.

### Decisión: shape y canje

- `customers` gana `display_name`, `phone` (E.164, validado en la base),
  `claimed_at`, `merged_into_customer_id`. `profile_id` pasa a nullable.
  `unique (organization_id, phone) where phone is not null`.
- **El canje NUNCA fusiona automáticamente.** Si la persona que activa ya es
  cliente de esa organización con cuenta propia, la RPC devuelve
  `ALREADY_CUSTOMER_OF_ORGANIZATION` sin consumir el token. Fusionar mueve
  plata (pagos, planes) y puede chocar contra el `EXCLUDE` de ADR-0024 a
  mitad de camino — es una operación separada, `merge_customers()`,
  **OWNER-only**, que nunca reapunta lo que chocaría: adivinar cuál de dos
  pagos solapados vale es equivocarse en silencio con la plata de un cliente.
- **Token: 256 bits de `gen_random_bytes`, hash SHA-256 en reposo, un solo
  uso, revocación al reenviar por índice único parcial.** No `random()`
  (el precedente de `organization_invites` lo usa y es un PRNG no
  criptográfico) y no en claro (a diferencia de `organization_invites`, acá
  existe un actor cercano —STAFF— que no debe poder robar el link de otro).
  La RPC de canje recibe **un solo parámetro, el token**, y no nombra
  ninguna fila de cliente — aplicación literal de ADR-0005: no hay IDOR
  posible porque no hay input que apunte a una fila.
- **Sin confirmación por últimos 4 dígitos.** No defiende el caso real (si el
  dueño tipeó mal el número, el sistema compararía contra el número
  tipeado, que es el que tiene la persona equivocada) y solo grava de
  fricción un flujo cuyo único objetivo es no caerse.
- El link lleva **el nombre del negocio, nunca el del cliente**: ponerlo en
  el `text=` del deep link haría que un link mal enviado revele a un
  desconocido el nombre de otra persona. El nombre del cliente aparece
  recién en la pantalla de activación, detrás del token.
- Roles: **STAFF** para alta, emisión y revocación (mismo criterio que
  enrolar clientes hoy); **OWNER** para unlink y merge (mismo criterio que
  `revoke_member`). Gatear la emisión a OWNER tendría el efecto perverso de
  que el mostrador comparta la cuenta del dueño.
- Un cliente gestionado **puede** tener pagos, planes, reservas fijas y
  créditos de recupero (ADR-0025) — el mostrador cancela en su nombre, y la
  auditoría de `cancelled_by` degrada de "quién decidió" a "quién ejecutó +
  qué dijo que era el motivo". Ese es el costo real de no exigir cuenta.

### Las ocho resoluciones del Orchestrator

1. **Shape aprobado** tal como está arriba.
2. **Gates de rol aprobados**: STAFF alta/emisión/revocación, OWNER
   unlink/merge.
3. **`merge_customers()` entra en la Fase J**, aunque sea sin pantalla propia.
   Sin ella el remedio a "teléfono repetido dentro de una organización" es
   SQL a mano en producción.
4. **Rate limiting: el de emisión se resuelve dentro de la RPC** (contar
   activaciones recientes por cliente/organización, sin infraestructura
   nueva). **El de canje queda como riesgo residual aceptado por escrito**:
   la infraestructura general de rate limiting la pidió ADR-0008 para
   disponibilidad pública y sigue pendiente; no se construye dos veces la
   misma pieza de infraestructura por partes. Se revisita cuando esa deuda
   se pague.
5. **Sin confirmación por últimos dígitos.**
6. **Hallazgos de la propuesta**: **A ya está cerrado** (migración 19,
   ADR-0028 — el bypass de `cancel_booking` con `profile_id NULL`).
   **B (los `INNER JOIN` a `profiles` que hacen invisible al cliente
   gestionado en `organization_customers`, `occurrence_bookings`,
   `organization_payment_summary`, `schedule_rule_standing_reservations`) y
   E (la policy `customers_write_staff` solo mira `organization_id`, así que
   un STAFF podría apuntar el `profile_id` de un cliente ajeno por PATCH
   directo) son bloqueantes de la migración de la Fase J.** C
   (`profile_id on delete cascade` → `set null`) ya estaba confirmado
   pendiente para esta fase desde la migración 19.
7. **`service_plans_select_public using (true)` se acota, pero no en la Fase
   J.** El usuario pidió el mismo día planes que cubran varios servicios
   (ver ADR-0029): esa tabla va a cambiar de forma pronto. Acotar su RLS
   ahora y volver a tocarla en un par de migraciones es hacer el mismo
   trabajo dos veces — se resuelve en la misma migración que implemente
   ADR-0029.
8. **TTL del token: 72 h.**

### Resuelto fuera de la propuesta

**Interruptor por organización para desactivar el mecanismo entero:** se
agrega (`customer_activation_enabled`, default `true`). Es un caso de
genericidad real que el agente levantó sin diseñar — un rubro sensible
(salud mental, adicciones) puede no querer que exista un link circulando por
WhatsApp que diga "sos cliente de X". Es barato (un booleano que oculta el
botón) y reversible, así que no bloquea la fase esperando la decisión del
usuario.

### Lo que sigue sin resolver

Rate limits numéricos exactos (30/h, 200/día, 10/15 min son propuestas
razonadas, no medidas — se ajustan con uso real) y `customer_payment_detail()`
sin confirmar si comparte el mismo `INNER JOIN` que sus hermanas (se verifica
al implementar).

---

## ADR-0029 — Alcance configurable de `ServicePlan`: uno, varios o todos los servicios

Fecha: 2026-09-22
Estado: **Aceptada**
Propuesta por: usuario en producción, con ADR-0024 ya desplegada: *"que
pasa si quiero crear planes que habiliten a todos los servicios? [...]
para mí los planes pueden o no ser por servicio, ahora están 100%
sujetos."* Diseño: `domain-architect`, propuesta completa en
`proposals/adr-0029-plan-scope.md`.

**Nota de secuenciación que cambió a mitad de camino:** la propuesta se
escribió asumiendo que `makeup_credits` (ADR-0025) todavía no existía —
"este ADR llega a tiempo de fijar la forma antes de que se construya".
Entre que se escribió y se aprobó, la Fase I se implementó y desplegó
completa. Esta ADR ya no fija una forma sobre una tabla vacía: **retrofitea
una tabla en producción**, potencialmente con créditos reales emitidos
durante la demo del usuario. No cambia la decisión, cambia el rigor de
verificación exigido antes de tocar producción: contra una copia de los
datos reales, no solo `db reset` sobre una base vacía.

### Decisión: dos mecanismos que conviven, no uno que se elige

`service_plans.service_id` (1:N) se **elimina**, no se depreca — se
reemplaza por dos mecanismos que resuelven intenciones distintas y no
chocan gracias a la regla de inmutabilidad (más abajo):

- **`service_plan_services`** (tabla puente): "este plan cubre exactamente
  estos N servicios" — una selección deliberada y cerrada que nunca crece
  sola.
- **`service_plans.applies_to_all_services`** (flag, resuelto dinámicamente
  contra los servicios activos de la organización en el momento de
  evaluar): "todos, incluidos los que se creen mañana". Se resuelve en
  vivo a propósito — materializarlo por trigger chocaría contra la
  inmutabilidad de alcance.

**`DROP_IN` queda fuera de esta ADR, a propósito**: multi-servicio en un
turno suelto es el paquete prepago/contador que ADR-0022 y ADR-0024 ya
excluyeron por decisión explícita del usuario, disfrazado de variante de
turno suelto. No se reabre esa puerta.

### La cuota, configurable como pidió el usuario

`quota_scope: PER_SERVICE | SHARED_ACROSS_SERVICES`, obligatorio solo en
`WEEKLY_QUOTA`. Bajo `SHARED_ACROSS_SERVICES` un cliente **sí** puede tener
una serie en cada servicio (total = cuota) — "veces por semana" ya era
"cantidad de `RecurringBooking`" desde ADR-0024 y nunca dependió de que
todas apuntaran al mismo servicio; generalizar el filtro de
`customer_series_in_force_count()` de un `service_id` a un `service_id[]`
extiende el invariante, no lo rompe. **Un solo `weekly_quota` por plan**,
nunca heterogéneo por servicio dentro del mismo plan — decisión explícita
para no reabrir la puerta de `MONTHLY_QUOTA` que ADR-0024 cerró.

### `Payment` se ancla a `(customer, plan)`; el servicio se vuelve derivado

`payments.service_id` pasa a nullable (poblada solo cuando el plan cubre un
único servicio). Se agrega `payment_service_coverage` (una fila por
servicio cubierto por el pago, con su propio rango y estado), que
**reemplaza** el `EXCLUDE` viejo de `payments` — dos mecanismos de
exclusión separados no detectarían el choque cruzado entre un pago de un
solo servicio y uno multi-servicio que también lo cubre. El índice de "un
solo `UNLIMITED` activo por servicio" se retira: era scaffolding del puente
de migración de ADR-0024, ya muerto en la práctica (la Fase H manda
`service_plan_id` explícito desde el formulario de pagos) — dos
`UNLIMITED` activos que se solapan en cobertura pasan a ser catálogo
legítimo.

### Impacto en créditos de recupero (ADR-0025), ahora sobre datos reales

Un crédito originado en un plan `PER_SERVICE` queda anclado a ese servicio
exacto. Uno originado en `SHARED_ACROSS_SERVICES` queda utilizable en
cualquiera de los servicios del plan (`service_id null`, resuelto contra
`service_plan_covered_service_ids()` al consumir) — porque el pool nunca
fue "de" un servicio en particular. No es simétrico y no debía serlo:
restringir el crédito compartido a un solo servicio regalaría menos de lo
que el cliente tenía; hacer fungible un crédito `PER_SERVICE` regalaría
más.

### Inmutabilidad del alcance, con una distinción explícita

El *valor* de `applies_to_all_services` es inmutable con pagos (igual que
`plan_kind`/`weekly_quota`), pero lo que ese flag *resuelve* no lo es — un
plan "todos los servicios" con pagos activos cubre automáticamente un
servicio creado después, por diseño: es lo que un pase VIP significa.
`service_plan_services` (selección explícita) sí es inmutable fila por
fila una vez que el plan tiene pagos.

### Las cinco resoluciones del Orchestrator

1. **`fill_payment_service_plan()` y el índice
   `service_plans_one_active_unlimited_idx` se retiran** en la misma
   migración de esta ADR — confirmado muerto en la práctica, no hace falta
   un puente de compatibilidad de dos fases para código que nada externo
   escribe ya.
2. **El nombre de la sección pública "planes combinados" lo decide
   `ui-ux-agent`**, en el vocabulario genérico del producto. No lo fijo
   desde acá.
3. **La sintaxis exacta del `EXCLUDE` sobre `payment_service_coverage`
   (orden de triggers, `FOR EACH ROW WHEN`) es de `database-agent`** al
   implementar — el patrón ya funciona hoy sobre `payments`, el riesgo
   técnico es bajo.
4. **La laguna de reporte** (pantallas que filtran pagos por `service_id`
   no van a mostrar pagos de planes multi-servicio hasta actualizarse) se
   **acepta como deuda conocida**, no bloquea esta fase. No es un agujero
   de seguridad ni de cobertura de reserva — la cadena de reserva ve todo.
   Se resuelve en un pase posterior de `frontend-admin-agent`.
5. **La migración de los tests existentes es responsabilidad de quien
   implemente**, con revisión de `qa-testing-agent` antes de cerrar la
   fase — sin estimar de antemano cuántos rompen.

### Restricción de secuenciación (nueva, por el retrofit sobre producción)

**Antes de tocar la base real**: dump de `makeup_credits` y `payments` del
proyecto de producción, y verificación de que la migración de retrofit
preserva cada fila existente con su significado exacto (un crédito viejo
con `service_id` no nulo tiene que seguir siendo `PER_SERVICE` después de
la migración, nunca reinterpretarse como compartido). Esta migración se
prueba contra una copia de los datos reales, no solo contra `db reset`
sobre una base vacía — es la primera de esta iteración con ese requisito,
porque es la primera que retrofitea una tabla que ya tiene filas de un
cliente real.

## ADR-0030 — Refresh visual: marca de plataforma (violeta/cian), landing tipo funnel

Fecha: 2026-09-22
Estado: **Aceptada**
Propuesta por: `ux-ui-designer`, a pedido del usuario ("mejorar el diseño y
la UX de todo el producto" tomando `https://turnito.app/uy/` como
referencia de nivel). Propuesta completa en
`docs/proposals/adr-0030-visual-refresh.md` — este ADR registra la
decisión, no la repite.

**El hallazgo que reordenó la propuesta**: `--primary` significaba a la vez
"acento de esta pantalla" y "color de Reservaste", así que el logo de la
plataforma se pintaba con el `accent_color` de cada organización dentro de
`[data-brand]` — lo opuesto a lo que ADR-0020 decidió ("la consola de
plataforma es nuestra marca, no la del cliente"). Se separa en tres capas:
**P** (`--brand-violet`/`-strong`/`-deep`, `--brand-cyan`/`-deep`, fijos,
nunca sobreescribibles por un tenant — logo, landing, `/admin`), **S**
(semánticos success/warning/destructive, sin cambios), **T** (`--primary`,
default = violeta, pisado por `[data-brand]` exactamente igual que hoy). Una
organización con acento configurado no ve cambiar su página; una sin acento
pasa de índigo a violeta junto con el resto del producto sin marca de
tenant.

### Resoluciones a las preguntas abiertas de la propuesta (decididas con el usuario, 2026-09-22)

1. **Violeta `#7c3aed`: aprobado tal cual propuesto.**
2. **CTA principal de la landing → página `/contacto` con formulario**, no
   el link directo a WhatsApp que recomendaba la propuesta por ser el de
   menor esfuerzo. Es la opción de mayor alcance de las tres — genera un
   lead real, no solo abre un chat — y el usuario la eligió a sabiendas de
   que implica capturar el envío en algún lado, no solo maquetar un
   formulario. Alcance mínimo: una tabla nueva y chica para las
   submissions (`backend-engineer` define el shape exacto — algo en la
   línea de `platform_contact_requests`, `INSERT` público vía RPC o policy
   estricta, `SELECT` solo `is_platform_admin()`), visible desde `/admin`.
   No es alta de organización ni toca `ServicePlan`/booking — no arrastra
   ninguna otra decisión estructural.
3. **Precios: USD tal cual**, sin aclaración de IVA ni conversión a UYU.
4. **Dominio: todavía no hay uno real.** La landing y el copy de onboarding
   no deben prometer `reservaste.app` — ajustar para no citar un dominio
   que no resuelve, hasta que haya uno comprado.

### Resoluciones del Orchestrator sobre las preguntas menores (no bloqueantes, criterio de bajo riesgo)

5. **Testimonios: sin fabricar ninguno**, tal como ya proponía §3.6 —
   señales factuales verificables hasta que el cliente real autorice ser
   nombrado.
6. **`.brand-band` (banda oscura acotada de la landing): aprobada tal como
   está escrita en §1.6** — no es dark mode (sin toggle, sin persistencia,
   sin inputs adentro), es una sección con fondo literal de capa P.
7. **`font-optical-sizing: auto` global: no por ahora.** Queda confinado a
   la landing (la escala `.display-*`/`.lead` ya vive solo en
   `components/marketing/`) para no tocar la tipografía de las 60
   pantallas de producto en un pase que ya es grande.
8. **Vocabulario de rubro filtrado ("la clase" en copy de `/me` y
   asistencia): corrección mínima ahora** (neutralizar a "la fecha"/"el
   turno"), **no** la columna configurable por `Organization` que
   proponía la propuesta como alternativa — eso es un cambio de dominio
   nuevo y por lo tanto su propio ADR si se decide encararlo, no algo que
   se cuela dentro de un pase de diseño visual.

### Plan de fases (de la propuesta, sin cambios)

Fase 1 (tokens, aditiva, riesgo nulo) → Fase 2 (landing + `/contacto`) →
Fase 3 (de-fuga de `Brand`/`BrandMark` a capa P — **visible para tenants
con acento configurado, avisar al cliente actual antes de desplegar**) →
Fase 4 (flip de `--primary` + neutros, requiere pasada visual con
`next dev`) → Fase 5 (superficies de cliente, verificar con ≥3 acentos de
tenant distintos) → Fase 6 (deuda de densidad del panel admin, no arranca
sin `next dev`) → Fase 7 (`/admin`). Se arranca por la Fase 1 y 2 ahora;
3-7 quedan para las próximas rondas de este mismo ADR, no para fases
nuevas del roadmap — es un solo cambio de identidad visual ejecutado en
etapas por riesgo, no siete features distintas.

## ADR-0031 — Prorrateo del primer período en ciclos de facturación largos

Fecha: 2026-09-23
Estado: **Aceptada**
Propuesta por: `backend-engineer`, a pedido del usuario (feedback de
producción). Propuesta completa en
`docs/proposals/adr-0031-prorrateo-ciclos-largos.md` — este ADR registra
la decisión, no la repite.

Extiende `billing_period_for()` (no la duplica) con `CALENDAR_PERIOD`/
`ROLLING_PERIOD` + `billing_period_months`/`billing_anchor_month` en
`service_plans`, aditivo, sin tocar `CALENDAR_MONTH`/`ROLLING_MONTH`
existentes. `quote_service_plan_period()`, función nueva de solo lectura,
cotiza — el mostrador sigue registrando el pago con `amount` libre.

### Resoluciones del Orchestrator a las preguntas abiertas

1. **Prorrateo por meses enteros, no por días.** Es lo que el mostrador
   puede explicar en una frase y no depende de cuántos días tiene el mes.
2. **Se prorratea el precio, no el período.** La cobertura del cliente
   arranca al principio del ciclo calendario, no el día que se hizo
   cliente — la asimetría es real pero explicable, y recortar
   `period_start` rompe el `EXCLUDE` y la alineación de renovación de
   todo el plan.
3. **El prorrateo aplica solo al alta inicial en un ciclo largo, nunca al
   cambio de plan a mitad de período.** ADR-0024 resolución 1 (VOID +
   recargar) es una regla de resolución, no de cobro — mezclar las dos
   metería una jerarquía entre planes en la función de cotización, que es
   justo lo que esa resolución evitó del lado del motor.
4. **Sin nota de crédito por lo no consumido de un plan anterior.** No es
   parte de lo que se pidió (el pedido era sobre ciclos largos, no sobre
   cambios de plan) — si en algún momento se necesita, es una entidad
   nueva y su propio ADR, no un campo escondido en `quote_service_plan_period()`.
5. **`organizations.currency_minor_units` se difiere a ADR-0027.** Hoy no
   hay ningún consumidor que necesite el centavo; redondeo a unidad
   entera de moneda alcanza.
6. **El default de vencimiento del crédito de recupero en planes de ciclo
   largo pasa a `END_OF_MONTH`, no `END_OF_BILLING_PERIOD`.** Un crédito
   vivo tres meses en un plan trimestral triplica el riesgo que ADR-0025
   ya había dejado anotado como abierto (cuántos créditos vivos tolera la
   capacidad real). `END_OF_BILLING_PERIOD` sigue siendo válido para
   ciclos mensuales, donde no cambia nada de lo ya aceptado.

**Implementación**: backend completo (2026-09-23) — Fase 31,
`20260923180000_phase31_long_billing_periods.sql`. Pendiente: revisión
de `security-engineer`, formulario de plan + formulario de pago
(`frontend-engineer`, especificado en `docs/api.md` §Fase 31).

## ADR-0032 — `audit_log`: registro de acciones sensibles

Fecha: 2026-09-23
Estado: **Aceptada**
Propuesta por: `backend-engineer`, a pedido del usuario (feedback de
producción, alcance acotado explícitamente por el usuario a acciones
sensibles/administrativas, no todo el sistema). Propuesta completa en
`docs/proposals/adr-0032-audit-log.md`.

**El hallazgo que decide el diseño**: la mitad de las escrituras del
alcance (alta y anulación de pago, cambios de precio/nombre de plan) no
pasan por ninguna RPC — son `INSERT`/`UPDATE` de PostgREST directo. Un
insert explícito por RPC dejaría afuera la mayoría de lo que se pidió
auditar. Por eso el diseño es por **trigger** (`AFTER INSERT/UPDATE` con
predicado `WHEN`, no evadible) con contexto opcional desde la RPC vía
`set_config('app.audit_note', ...)`. Tabla `audit_log` con `action` enum
(8 valores), `target_table`+`target_id` (sin FK al target, a propósito:
el log tiene que sobrevivir al borrado de lo que audita), `metadata`
jsonb con el diff mínimo (nunca la fila entera ni datos personales),
inmutable (RLS sin policies de escritura + trigger que rechaza
`UPDATE`/`DELETE`, sin excepción para `service_role`).

### Resoluciones del Orchestrator a las preguntas abiertas

1. **Lectura**: `is_platform_admin()` ve todo; `OWNER` ve el log de su
   propia organización; `STAFF` no ve nada. Aprobado tal cual propuesto.
2. **El `OWNER` sí ve las filas de `ORGANIZATION_SUBSCRIPTION_CHANGED`**
   (que la plataforma le suspendió/reactivó la cuenta) — una suspensión
   que el dueño no puede rastrear genera desconfianza. **Con un matiz**:
   la identidad del actor de plataforma (`actor_id` de un platform admin)
   **no se resuelve a un nombre en la UI del cliente** — el dueño ve que
   pasó y cuándo, no quién de nuestro lado lo hizo.
3. **Sin política de retención por ahora.** Mantener el índice
   `(organization_id, created_at desc)` listo para cuando haga falta, sin
   implementar el borrado todavía.

**Alcance confirmado, sin cambios sobre lo propuesto**: pagos (alta,
cambio de estado incluida anulación), reservas de mostrador (alta y
cancelación cuando el actor no es el propio cliente), planes (alta,
cambio de precio/nombre, activar/desactivar), suspensión/cambio de plan
SaaS de la organización. Fuera de alcance, explícito: asistencia,
créditos de recupero (ya tienen su propio log en `makeup_credits`),
acciones del propio cliente, altas de cliente/servicio/horario/branding,
logins y lecturas.

**Implementación**: backend completo (Fase 30,
`20260923170000_phase30_audit_log.sql`, 226/226 tests de integración) —
tabla + enum + 4 triggers de auditoría + trigger de inmutabilidad +
policy de `SELECT` + `organization_audit_log()` como lectura que
enmascara al actor de plataforma (resolución 2). Detalle en
`docs/database.md` (Fase 30), `docs/security.md` y `docs/api.md`.
Pendiente: revisión de `security-engineer` (obligatoria por el propio
ADR), la server action `getOrganizationAuditLog()` y la pantalla de solo
lectura del `OWNER` en `/org/[slug]/configuracion` (especificada en
`docs/api.md`).

## ADR-0033 — Roles configurables por organización

Fecha: 2026-09-23
Estado: **Aceptada**
Propuesta por: `backend-engineer`, a pedido del usuario (feedback de
producción — un rol de "profesor" no debería ver pagos, configurable por
organización). Propuesta completa en
`docs/proposals/adr-0033-roles-configurables.md`.

`OWNER` se mantiene como enum fijo, no configurable — es la raíz de
confianza (evita el ciclo "me edito el rol para poder editar roles",
corta antes de mirar permisos así que un rol mal configurado nunca deja
a la organización sin quien lo arregle, y cero riesgo de migración sobre
las policies/RPCs que ya usan `is_organization_owner()`). Lo nuevo es una
capa de roles configurables **dentro** de `STAFF`: tabla
`organization_roles` (por organización, nombre libre, uno default) +
`organization_members.role_id` nullable con fallback al default.
Permisos como columnas booleanas (no jsonb, para no reintroducir lógica
de tres valores — la causa de un bypass de autorización ya documentado
en ADR-0026/ADR-0028) con clave de permiso en enum.

### Resoluciones del Orchestrator a las preguntas abiertas

1. **`VIEW_PAYMENTS` y `MANAGE_PAYMENTS` separados**, no un solo permiso
   — con `CHECK` de que `MANAGE` implica `VIEW`.
2. **Se acepta la fuga residual de `upcoming_unpaid`/`PAYMENT_REQUIRED`**
   en pantallas operativas (asistencia, horario fijo) para un rol sin
   `VIEW_PAYMENTS` — documentada, no oculta: sin ese dato un rol no
   entiende por qué no puede anotar a alguien. "No ver pagos" no es "no
   ver que algo depende de un pago".
3. **Se cierra ahora, en la misma migración, el hueco ya existente** de
   que el precio/nombre de un `ServicePlan` solo está protegido a nivel
   `OWNER` en TypeScript (`frontend/app/actions/service-plans.ts`), no en
   la base — cualquier `STAFF` puede cambiarlo hoy por PostgREST directo.
   No es parte del pedido original pero la propuesta no se puede
   construir con ese hueco abierto debajo.
4. **Asistencia (`MANAGE_ATTENDANCE`) es un permiso configurable.**
5. **Sin tope de cantidad de roles por organización.**

Alcance del primer corte, sin más: `VIEW_PAYMENTS`, `MANAGE_PAYMENTS`,
`MANAGE_BOOKINGS`, `MANAGE_CUSTOMERS`, `MANAGE_ATTENDANCE`. Fuera:
invitar equipo/administrar roles (`OWNER` únicamente — un rol no puede
ampliarse a sí mismo), configuración/branding, planes y precios,
suscripción SaaS, créditos manuales.

**Migración**: un rol "Equipo" por organización con los cinco booleanos
en `true` + backfill de `role_id` — comportamiento idéntico el día del
deploy para todo `STAFF` existente.

**Implementación**: backend completo (2026-09-23) — Fase 32,
`20260923190000_phase32_configurable_roles.sql`, 255/255 tests de
integración, 17 RPCs endurecidas (más allá de las 8 mínimas de la
propuesta) y el hueco preexistente de precio de plan editable por
`STAFF` cerrado en la misma migración. Pendiente: revisión de
`security-engineer` (obligatoria), pantalla de roles + esconder por
permiso (`frontend-engineer`, especificado en `docs/api.md` §Fase 32).

**Implementación**: pendiente, con revisión obligatoria de
`security-engineer` antes de cerrarse (toca auth y roles).

## ADR-0034 — Alta de equipo (STAFF) sin registro previo

Fecha: 2026-09-23
Estado: **Aceptada**
Propuesta por: `backend-engineer`, a pedido del usuario (feedback de
producción — invitar a alguien al equipo hoy exige que ya tenga cuenta;
extender el patrón de activación por WhatsApp de ADR-0026 al personal).
Propuesta completa en `docs/proposals/adr-0034-team-invitations.md`.

Mecanismo **separado** de `customer_activations` (tabla propia
`team_invitations`), aunque comparte el acuñado del token: el destino
del token no existe todavía como fila operable (a diferencia de un
cliente gestionado), otorga acceso a datos de terceros con mayor radio
de explosión, y la unicidad de "un link vivo" es por `(organización,
email)`, no por fila. **Link de activación, no contraseña temporal**: una
contraseña obligaría a usar la Admin API de Supabase con la
`service_role` key fuera de todo camino de request normal, seguiría
viva después de la ventana de 24h salvo rotación forzada, y no sirve si
la persona entra por Google OAuth. El canje exige que el email de la
sesión coincida con el de la invitación (mismo precedente que
`INVITE_WRONG_EMAIL` de `create_organization_with_owner()`) — el canal
es WhatsApp, el vínculo real es el email.

### Resoluciones del Orchestrator a las preguntas abiertas

1. **Se conserva `invite_member_by_email()`** como camino rápido cuando
   la persona ya tiene cuenta — elegido explícitamente por quien invita,
   nunca automático (automático reintroduciría el oráculo de
   `PROFILE_NOT_FOUND` que ya existe).
2. **El canje exige coincidencia de email**, sin excepción.
3. **TTL de 24 horas** (el pedido original del usuario, textual).
4. **Se guarda el teléfono en la invitación**, para el link de WhatsApp.
5. **Una invitación nunca puede crear un `OWNER`** — `CHECK` en la tabla,
   no una convención que la RPC deba recordar.

Riesgos ya identificados y aceptados con su mitigación: `enforce_plan_limit()`
se chequea también al **emitir** la invitación además de al canjearla (si
no, se pueden emitir más invitaciones que cupo libre); el canje **nunca**
pisa el rol de un miembro que ya existe (a diferencia de
`invite_member_by_email()`, que si lo hace, y ahí es correcto porque es
sincrónico y `OWNER`-gated); `organization_team_invitations()` como read
model nuevo para que el dueño vea a quién invitó, no solo a quién ya se
sumó; cookie/ruta propia (`/equipo/[token]`) para no pisar el token de
activación de un cliente que además fue invitado al equipo.

**Orden de implementación: ADR-0033 primero, después ADR-0034** — si se
implementan las dos, conviene que `team_invitations` nazca con
`role_id` en vez de una segunda migración sobre una tabla con tokens
vivos. Si ADR-0033 nunca se implementara, ADR-0034 es igual de
implementable con el enum `STAFF`/`OWNER` liso.

**Implementación**: backend completo (2026-09-23) — Fase 33,
`20260923200000_phase33_team_invitations.sql`, 276/276 tests de
integración (21 nuevos), nace con `role_id` sobre ADR-0033 ya aplicada.
Pendiente: revisión de `security-engineer` (obligatoria por este mismo
ADR), server actions y UI (`frontend-engineer`, especificado en
`docs/api.md` §Fase 33).

## ADR-0035 — La solicitud de cambio de plan no es el cambio

Fecha: 2026-09-23
Estado: **Aceptada**
Propuesta por: `backend-engineer`, a pedido del usuario ("si excedo
frecuencia de plan, ofrecer upgrade/downgrade de planes y redireccionar
a planes"). Implementación completa en la migración Fase 28
(`docs/database.md`).

**El problema**: sin cobro online (ADR-0027 bloqueada por elección de
pasarela), dejar que un cliente cambie de plan por su cuenta implicaría
crear cobertura (`Payment` PAID + `payment_service_coverage`) que nadie
cobró — exactamente lo que "pago ≠ permiso" (ADR-0005/ADR-0013) existe
para impedir. Y relajar el `EXCLUDE` de `payment_service_coverage` para
que el cliente pueda anular su propio pago ya fue descartado como
alternativa en ADR-0024.

**Decisión**: `plan_change_requests` registra un **hecho accionable**
("este cliente pidió cambiarse a este plan"), no una cobertura. **Ninguna
función de decisión de reserva la lee** — un pedido pendiente no habilita
ni bloquea ninguna reserva. El cliente ve el catálogo público de planes
de un servicio y pide el cambio; el mostrador lo cobra como ya cobra
cualquier alta/cambio de plan hoy (`RegisterPaymentForm`, VOID +
recargar de ADR-0024), y ese pago cierra el pedido solo (trigger
`after insert on payments`). Mismo patrón que el lead de `/contacto`
(ADR-0030, Fase 24) pero dentro del portal, con identidad real (autor
= `Customer` autenticado, no anónimo — alcanza un índice único parcial
+ tope de 5 pendientes, sin necesitar rate limit por IP).

Sin este ADR, la alternativa "el cambio se aplica solo, sin cobro" habría
sido más rápida de construir pero rompía dos invariantes ya cerradas del
producto — no era una opción real, era la misma pregunta de ADR-0024
resolución 1 vuelta a abrir desde otro ángulo.

**Implementación**: backend completo (Fase 28, 214/214 tests). Pendiente:
`frontend-engineer` (pantalla de catálogo + redirigir el rechazo de
cuota ahí, en vez de a "ver mi plan actual" como está hoy) y
`security-engineer` (RLS de la tabla nueva, disclosure del catálogo
público, el trigger sobre `payments`).

## ADR-0036 — `DELETE` por la Data API queda cerrado en tablas con hijos en cascada

Fecha: 2026-09-23
Estado: **Aceptada**
Propuesta por: `security-engineer`, verificación independiente de la
Fase 34 (ADR-0033). No es un hallazgo de ADR-0033 — es preexistente
desde que esas tablas tienen policy `ALL using is_organization_member()`
— pero ADR-0033 lo vuelve urgente: el valor entero de un rol restringido
(`MANAGE_BOOKINGS`/`MANAGE_PAYMENTS` en `false`) queda vacío si el mismo
actor puede lograr el mismo efecto con un `DELETE` directo por
PostgREST, que **no evalúa RLS en la cascada de FK**.

**El problema, verificado con reproducción real**: `schedule_rules`,
`services`, `resources` (y probablemente `customers`) tienen policy
`ALL` para cualquier miembro activo, con hijos `ON DELETE CASCADE`
(`schedule_rules → slot_occurrences → bookings`,
`services → payments`). Un `DELETE /rest/v1/schedule_rules?id=eq.<X>`
por un STAFF con los cinco permisos de ADR-0033 en `false` borra en
cascada `Booking`s `CONFIRMED` — no las cancela, las **borra**: sin
`cancelled_at`/`cancelled_by`, sin `MakeupCredit`, sin fila en
`audit_log`. Rompe directamente el invariante de negocio ya escrito en
`CLAUDE.md` ("las reservas canceladas nunca se borran"). Un
`DELETE /rest/v1/services` con un `ServicePlan` de alcance global borra
en cascada pagos `PAID` — historial financiero destruido por un rol sin
`VIEW_PAYMENTS` ni `MANAGE_PAYMENTS`.

### Decisión

**Ninguna de estas tablas tiene un caso de uso legítimo para `DELETE`
por la Data API.** Todo lo que hoy se "borra" en el producto ya tiene su
camino correcto: `discontinue_schedule_rule()` (cancela y libera, no
borra), desactivar (`is_active = false`) para `Service`/`Resource`,
`revoke_member()` para equipo. Ninguna pantalla del panel ofrece un
botón "eliminar" sobre estas tablas — lo confirmé revisando `docs/api.md`
antes de aceptar esto como decisión, no lo asumí.

**Se cierra el `DELETE` de la Data API para `schedule_rules`,
`services`, `resources`, `customers`, `schedule_exceptions`,
`service_entitlements`, `service_resources`** — las policies `ALL`
pasan a `SELECT`/`INSERT`/`UPDATE` explícitos, sin `DELETE`. Bajo RLS,
la ausencia de policy de `DELETE` deniega por default: no hace falta
ningún trigger nuevo, es remover el comando de la policy existente.
`OWNER` tampoco tiene `DELETE` directo — si algún día se necesita borrar
de verdad (no cancelar/desactivar), es una RPC nueva con su propia
decisión explícita, no un `DELETE` genérico.

**No se toca** ninguna FK `ON DELETE CASCADE` existente (siguen siendo
correctas para cuando la fila padre sí se borra por una vía legítima
futura) ni ninguna RPC ya gateada (`discontinue_schedule_rule()` sigue
usando `UPDATE`, no `DELETE`, así que no se ve afectada).

**Implementación**: backend completo (2026-09-23) — Fase 35,
`20260923220000_phase35_close_data_api_delete.sql`, 301/301 tests de
integración (11 nuevos). Las 7 policies `ALL` pasan a `INSERT`/`UPDATE`
explícitos, sin `SELECT` duplicado (las tablas ya tenían policy de
lectura propia, subconjunto o igual a la que se retiró — verificado
tabla por tabla, no se agregó una policy redundante). Confirmado que
ninguna de las 7 tenía un `DELETE` legítimo en todo el código (grep
completo en `backend/` y `frontend/`) y que ningún RPC interno usa
`DELETE FROM` sobre ellas. El test se validó al revés (restaurando las
policies viejas temporalmente): sin el fix, 8 de los 11 casos fallan,
incluidos los dos escenarios exactos que encontró `security-engineer`.
Pendiente: barrido de `security-engineer` sobre el resto del schema por
otras policies `FOR ALL` con hijos en cascada fuera de estas 7 tablas.
**Hecho, ver ADR-0037 — encontró algo peor.**

## ADR-0037 — Las vistas públicas dejan de ser escribibles por `anon`

Fecha: 2026-09-23
Estado: **Aceptada, urgente**
Propuesta por: `security-engineer`, en el barrido que pidió ADR-0036.
**Vulnerabilidad crítica preexistente, no introducida por ninguna ADR de
hoy** — existe desde la Fase 4 (ADR-0008, hace meses), recién detectada.

**El hallazgo, reproducido contra la base real, actor `anon` sin
sesión**: `organizations_public`/`services_public` se crearon sin
`security_invoker` a propósito (el calendario público necesita saltear
RLS en **lectura**), pero el bypass alcanza los cuatro comandos, y
`anon`/`authenticated` tienen `INSERT`/`UPDATE`/`DELETE` sobre las dos
vistas por default privileges de Postgres (el `grant select` explícito
de la migración es decorativo, ya tenían todo). Confirmado con requests
reales, sin autenticar:

- `DELETE services_public` sobre un servicio sin plan → **borra en
  cascada una `Booking` `CONFIRMED`** (sin `cancelled_at`, sin
  `MakeupCredit`, sin auditoría).
- Lo mismo sobre un servicio cubierto por un plan
  `applies_to_all_services` → **borra un `Payment` `PAID`**.
- `PATCH organizations_public` sobre otro tenant → reescribe
  `slug`/`name`/`timezone` de una organización ajena.
- `POST organizations_public` → **crea una organización saltando
  `create_organization_with_owner()`**, el gate de invitación
  (ADR-0017) y `enforce_plan_limit()`.
- Encadenado (`PATCH` para liberar un slug + `POST` para reclamarlo) →
  **secuestro completo del slug de un negocio real**: su URL pública
  pasa a resolver a la organización falsa del atacante.
- `GET services_public` sin filtro → enumera servicios de **154
  organizaciones** en un solo request, sin necesidad de adivinar nada.

### Decisión

```sql
revoke insert, update, delete, truncate on public.organizations_public from anon, authenticated;
revoke insert, update, delete, truncate on public.services_public  from anon, authenticated;
```

**`security_invoker = on` es el fix equivocado** — haría que el
`SELECT` evaluara RLS con los privilegios del llamador y el calendario
público (todo el punto de ADR-0008) dejaría de funcionar para un
visitante anónimo. Se cierra la escritura, se conserva la lectura
intacta.

**De paso, mismo barrido, dos hallazgos menores en el mismo lote**:
`organization_members_write_owner` (policy `ALL`) permite que el propio
`OWNER` se borre a sí mismo por `DELETE` directo saltando el guard
`LAST_OWNER` de `revoke_member()` — deja la organización sin ningún
`OWNER`, inadministrable para siempre salvo por un platform admin.
Mismo fix que ADR-0036 (`ALL` → `INSERT`+`UPDATE`, misma expresión). Y,
como defensa en profundidad de bajo riesgo, la misma partición en
`organization_roles`/`service_plan_services` (hoy protegidas solo por
trigger, no por ausencia de policy).

**Regla nueva para el repo**: toda vista de `public` se cierra con
`REVOKE` explícito de escritura para `anon`/`authenticated` en el mismo
momento en que se crea — el default de Postgres/Supabase es entregar
los cuatro comandos, no solo `SELECT`, y "hicimos `grant select`" no
implica que el resto esté cerrado.

**Implementación**: completa (2026-09-23) — Fase 36, en **dos
migraciones** que se commitean por separado:

- `20260923230000_phase36_public_views_read_only.sql` — el fix de la
  vulnerabilidad crítica: las dos vistas, `organization_members` y
  `audit_public_view_write_grants()`. No depende de nada posterior a la
  Fase 13, así que **se puede commitear sola**.
- `20260923240000_phase36b_role_and_plan_scope_delete_closed.sql` — las
  dos defensas en profundidad (`organization_roles`,
  `service_plan_services`). **Se commitea junto con la Fase 32**, de la
  que dependen las dos.

El split no es cosmético: la versión original, en un solo archivo,
volteó el PR en CI. CI aplica las migraciones **desde cero sobre lo que
hay en git**, y `organization_roles` es una tabla de la Fase 32, que
todavía no está commiteada → `relation "public.organization_roles" does
not exist` y la migración entera aborta. El caso de
`service_plan_services` era más peligroso por silencioso: la tabla
existe desde la Fase 22, pero la policy que se reemplaza
(`service_plan_services_write_owner`) la crea la Fase 32 — la Fase 22 la
había llamado `service_plan_services_write_staff` —, así que el
`drop policy if exists` no borraba nada y la policy `FOR ALL` con
`DELETE` seguía viva, con los tests en verde por el motivo equivocado.

**Regla que deja el incidente**: una migración sólo puede referenciar
objetos creados por migraciones **ya commiteadas**, y la verificación de
"aplica desde cero" se corre contra el estado real de git, no contra el
working tree. Correr la suite completa con todo el WIP presente no dice
nada sobre lo que CI va a aplicar.

Verificado en los dos estados: sólo-commiteado + Fase 36 → 231/231
integración en 23 archivos + 43 unitarios, `db reset` limpio; working
tree completo (Fases 30-35 + 36 + 36b) → 321/321 en 30 archivos + 60
unitarios. Validado al revés: restaurando temporalmente
los grants/policies viejos, 16 de 19 fallan, incluidos los seis
vectores anónimos y el secuestro de slug encadenado. `anon` sigue
pudiendo `SELECT` ambas vistas (el calendario público no se rompió).
`REFERENCES`/`TRIGGER`/`MAINTAIN` quedan sin revocar a propósito
(inalcanzables por la Data API — PostgREST no emite DDL — y ADR-0037
especificaba los cuatro verbos de escritura, no estos tres): deuda
aceptada de riesgo nulo, no bloqueante.

**Nota operativa**: como esta vulnerabilidad ya estaba en producción,
corresponde revisar `audit_log` y los conteos de `organizations`/
`services` contra lo esperado una vez aplicado el fix, para descartar
que haya sido explotada antes de encontrarla hoy. La `anon key` es
pública por diseño (no hay secreto que rotar).

## ADR-0038 — El monto sugerido se recalcula por días cuando se edita el período de un pago

Fecha: 2026-09-24
Estado: **Aceptada**
Propuesta por: usuario en producción, reportando una captura de
`/payments/[customerId]`: un plan mensual (Pilates, $1.700) con el
período acortado a mano de 03-01→03-31 a 03-01→03-10 seguía mostrando
$1.700 al registrar el pago. Decisión tomada por el Orchestrator
después de presentar tres opciones (recalcular por días, sólo avisar,
dejarlo manual) — el usuario eligió recalcular.

**El hallazgo**: en `RegisterPaymentForm`, "Período desde/hasta" y
"Monto" son dos inputs independientes que sólo se precargan **una vez**
al elegir el plan (`selectedPlan.periodStart/periodEnd` y
`suggestion.amount`). Después de eso no hay ningún vínculo entre
ellos — es a propósito, el hint dice literalmente "podés ajustarlos" —
pero nada distingue "edité el monto a propósito" de "acorté el período
y me olvidé de tocar el monto", y el segundo caso queda en pantalla
como una inconsistencia que parece un bug.

### Decisión

El monto sugerido se **recalcula en el cliente, proporcional a los
días**, cada vez que se edita "Período desde" o "Período hasta" —
**solo para planes de un mes** (`billingPeriodMonths` ausente o `1`,
es decir, fuera del flujo de `LongPeriodQuote`):

```
tarifaDiaria = selectedPlan.price / díasEnElPeríodoCompletoDelPlan
montoSugerido = round2(tarifaDiaria × díasEnElPeríodoEditado)
```

- `díasEnElPeríodoCompletoDelPlan` se congela al elegir el plan (el
  `periodStart`/`periodEnd` que ya resuelve `billing_period_for()` en
  el servidor), no se recalcula con cada tecla — es el denominador
  estable de la proporción.
- Sigue siendo **una sugerencia editable**, nunca algo que el backend
  imponga: el campo `amount` se puede pisar a mano después, igual que
  hoy. `registerPayment()` no cambia — sigue grabando el `amount` que
  llegue en el `FormData`, sin volver a calcularlo ni validarlo contra
  el período.
- **No toca el flujo de ciclos largos de ADR-0031.** Ese prorrateo es
  por meses enteros y sólo aplica al alta inicial — mezclarlo con un
  prorrateo por días aquí contradiría esa decisión ya cerrada. Un plan
  con `billingPeriodMonths > 1` sigue mostrando su `LongPeriodQuote`
  sin este recálculo.
- No requiere cambios de schema, RPC ni server action: es lógica
  puramente de presentación, igual que `suggestedAmount()`
  (ADR-0031) — vive en el mismo componente cliente.

**Implementación**: completa (2026-09-24) — `frontend-engineer` la hizo en
`lib/billing-period.ts` (`inclusiveDays()`, `round2()`) y
`RegisterPaymentForm`; `qa-engineer` agregó `lib/billing-period.test.ts`
(13 casos) tras un primer NO LISTO del reviewer por falta de cobertura;
segunda pasada, LISTO. Commit `6ff1902` (frontend), mergeado a `main` vía
PR #6. Verificado además a mano contra producción el mismo día (ver nota
de Fase de verificación visual, más abajo): plan de $1.700/mes, período
editado a 10 días → sugiere $566,67 exacto.

**Bug real encontrado en producción (2026-09-28), con dinero real de por
medio** — reportado por el dueño: un plan `WEEKLY_QUOTA` de $2.500
("Pilates Reformer 2 x S", organización `mathias-gym`), pago de un
cliente real registrado para 2026-10-01→2026-10-31 (octubre completo,
31 días, sin que el dueño sintiera que estaba "editando" nada raro) quedó
guardado en **$2.583,33** en vez de $2.500 — exactamente
`2500 / 30 × 31`.

Causa confirmada por `frontend-engineer` (sin necesitar acceso a
producción — el mismatch está forzado por la lógica del código, no por la
configuración particular del plan): `RegisterPaymentForm` se usa en dos
pantallas. Desde `/org/[slug]/payments/[customerId]?mes=` el período
completo del plan (el denominador del prorrateo) se resuelve anclado al
mes que dice la URL — correcto. Desde `/org/[slug]/customers/[customerId]`
(la ficha del cliente, sin selector de mes) se resuelve anclado a **hoy**
(`listPaymentPlanOptions(slug)` sin `month`, cae a la fecha actual en
`app/actions/service-plans.ts`). Si el mostrador registra un pago desde la
ficha del cliente y edita el período a mano a un mes que NO es el actual
(un caso legítimo y común: cargar un pago por adelantado, o de un mes que
ya pasó), el denominador queda congelado en los días del mes de "hoy"
mientras el numerador son los días del mes que se tipeó — dos meses sin
relación entre sí. Septiembre (denominador, 30 días) ÷ octubre (numerador
tipeado, 31 días) reproduce exacto los $2.583,33 reportados.

**Fix** (mismo día, `frontend/app/org/[slug]/customers/[customerId]/customer-forms.tsx`):
el prorrateo por días de ADR-0038 sólo tenía sentido para su caso
original — acortar/alargar el período *dentro* del mismo mes que ya
sugirió el plan. Nunca debió aplicarse cuando el período editado es un
mes completamente distinto (eso no es un prorrateo, es cobrar otro
período). Se agregó un chequeo de solapamiento (`overlaps()`, ya
existente en `lib/billing-blocks.ts`): si el período editado no se
solapa con el período completo que resolvió el servidor, el prorrateo
por días se desactiva y el monto sugerido cae al precio de lista del
plan (o al prorrateo del servidor si aplica, igual que antes de ADR-0038).
El caso original (acortar dentro del mismo mes) sigue funcionando igual.

**No se tocó el pago ya registrado de María Hornos** — corregirlo o no es
una decisión de negocio del dueño, no algo que el código deba decidir por
su cuenta.

**Auditoría completa (2026-09-28, corrida por el usuario contra
producción)**: un solo pago afectado en todo el sistema —
`fa018d14-f687-4a6e-b5c9-e882151436a4`, organización `kaimovete`, María
Hornos, $2.583,33 cobrados vs. $2.500 de lista, diferencia $83,33. Sin
otros casos. Corrección delegada al dueño vía el flujo normal de la app
(anular ese pago + registrar uno nuevo por el monto correcto) — no hizo
falta ninguna intervención directa sobre la base.

## ADR-0039 — Suite E2E de humo contra producción, después de cada deploy de frontend

Fecha: 2026-09-24
Estado: **Aceptada**
Propuesta por: usuario, después de una sesión de verificación visual
manual (Playwright ad-hoc, agente Orchestrator operando el navegador con
las credenciales de un owner real) que encontró todo el trabajo del día
funcionando correctamente en producción — pidió que esa verificación deje
de ser manual y corra sola en cada deploy futuro.

**Decisión, con las dos preguntas que el Orchestrator le hizo al usuario
ya resueltas:**

1. **Cuándo corre**: después del deploy, como *smoke test* contra el
   sitio real (`https://161-35-63-60.sslip.io`) — no como gate de CI
   antes de mergear. Un job nuevo en `frontend/.github/workflows/ci.yml`,
   `needs: deploy`, que corre Playwright contra producción una vez que el
   healthcheck del contenedor ya dio verde. Si falla, no bloquea nada
   (el deploy ya pasó) pero el workflow queda en rojo — visible, no
   silencioso.
2. **Rigor de "consistencia visual"**: **sin capturas de referencia**
   (no hay pixel-diff ni baseline que aprobar a mano en cada cambio
   visual intencional). En su lugar, reglas concretas y automatizables:
   escaneo de accesibilidad (`axe-core`, que entre otras cosas mide
   contraste de color) en cada pantalla clave, sin scroll horizontal a
   390px de ancho, y presencia de los componentes esperados
   (`EmptyState`, `FormError`/`FormSuccess`) donde el patrón del resto
   del producto los exige.

### Organización de prueba: `redentor`, permanente

La organización `redentor` (dueño real del usuario, antes vacía) se usó
hoy para crear datos de prueba a mano vía la UI real (no SQL directo) y
verificar visualmente: horario, agenda con tres estados de ocupación,
plan mensual, prorrateo por días (ADR-0038), roles, invitación de equipo,
auditoría. El usuario pidió explícitamente **dejar esos datos, no
limpiarlos** — pasa a ser la organización de pruebas permanente de la
suite, con datos con prefijo/nombre reconocible (`Cliente Uno`..,
`Pilates`, plan `Mensual`) para no confundirse con una organización real.

**Regla no negociable para la suite (la razón de este párrafo es un
hallazgo de hoy, no hipotético)**: cada test que cree estado en
`redentor` tiene que **limpiarlo al final** o **reusar lo que ya existe**
en vez de crear de nuevo, porque:

- **Los cupos de equipo son finitos** (2 en el plan actual) — armar una
  invitación de prueba sin revocarla al terminar deja el cupo ocupado
  para siempre; a la segunda corrida la suite ya no puede probar el
  flujo de invitación (se topa con "Tu plan no tiene más lugares de
  equipo"). **Toda invitación de prueba se revoca en el mismo test.**
- **Una reserva duplicada del mismo cliente en el mismo turno rechaza**
  (invariante del dominio, no negociable) — si un test reserva y no
  cancela, la corrida de la semana que viene sobre el mismo horario
  fijo falla por una razón que no tiene nada que ver con lo que se
  quiere probar. **Toda reserva de prueba se cancela (`Quitar`, staff)
  al final del test que la creó.**
- **No hay `DELETE`** para casi nada de esto (ADR-0036/0037): "limpiar"
  significa anular el pago, cancelar la reserva, revocar la invitación —
  nunca borrar filas.
- Recursos/servicios/planes/clientes de prueba se crean **una sola vez**
  (buscar por nombre antes de crear) — no hace falta recrearlos en cada
  corrida, y `ScheduleRule` sigue generando `SlotOccurrence` futuras solas
  (ventana rodante de 90 días, ADR-0009), así que un horario fijo creado
  una vez alcanza para siempre.

### Credenciales

Cuenta QA: la del dueño real de `redentor` que el usuario ya usa. Viven
como secrets de GitHub Actions del repo `frontend` (`QA_EMAIL`,
`QA_PASSWORD`) — el Orchestrator no tiene forma de crear secrets de
GitHub por su cuenta, así que **queda a cargo del usuario** cargarlos
antes de que el job corra por primera vez.

**Implementación**: delegada a `qa-engineer`.

### Addendum (2026-09-30) — Turnstile rompe la suite de humo en los flujos con login

ADR-0043 (desactivar confirmación por email) activó el widget real de
Cloudflare Turnstile en el login de producción. Un navegador operado por
Playwright nunca pasa el challenge ("Checking your Browser…", cuelgue
indefinido) — investigado a fondo, sin bypass legítimo disponible:

- No existe whitelist de IP para un sitio que no está proxyado por
  Cloudflare.
- Servir una site key de test (vía header/detección de entorno) tampoco
  funciona: Supabase Auth valida el captcha contra **una sola** secret key
  a nivel de proyecto — un token de la test key nunca validaría contra
  ella, y poner la secret key de test en producción dejaría a cualquiera
  bypasear el captcha con el keypair público de test de Cloudflare.

**Decisión**: la suite de humo de ADR-0039 **deja de cubrir flujos que
requieren login**. Se mantiene automatizada solo para lo que ya es público
sin autenticación (calendario, disponibilidad). Los flujos gateados por
login (agenda interna, pagos, clientes, equipo) vuelven a verificación
manual — el usuario acepta este límite como permanente, no como algo a
resolver con Turnstile Enterprise u otra alternativa por ahora.

## ADR-0040 — Activación por WhatsApp sobrevive un cambio de contexto de navegador

Fecha: 2026-09-28
Estado: **Aceptada**
Propuesta por: usuario en producción — reporte: "cuando un nuevo usuario
quiere loguear o registrarse [para activar su cuenta] siempre aparece
[el error 'no encontramos la invitación'], luego al refrescar queda como
activo". Diagnóstico de `frontend-engineer`, con evidencia real contra
producción (no sólo lectura de código): descartadas dos veces la hipótesis
de un TTL de cookie vencido (ya se había arreglado el 2026-09-23, commit
`3d865ea`, verificado en vivo con `curl` que la cookie de hoy dura 72h) y
la de una carrera Set-Cookie/redirect (viajan en la misma respuesta HTTP,
sin ventana). Causa real, por descarte y coherente con que el propio
mensaje de error ya la anticipa ("volvé a abrir el link de WhatsApp desde
este mismo navegador"): el cliente toca el link de WhatsApp en el
navegador embebido de WhatsApp (ahí queda la cookie httpOnly del token),
pero confirmar el email o completar el login de Google lo saca a OTRA
app/contexto (Mail, o el navegador del sistema — Google bloquea el login
dentro de WebViews embebidos desde 2021). Ese segundo contexto tiene un
cookie jar distinto: la cookie de activación nunca llegó ahí, así que
`/activar/continuar` no la encuentra aunque el token siga válido por sus
72h. No es un bug de un TTL ni de una carrera — es que el diseño actual
depende por completo de que la activación termine en el MISMO contexto de
navegador donde empezó, y un teléfono real no garantiza eso.

### Decisión

En vez de depender sólo de la cookie httpOnly para reencontrar la
activación después del desvío por login/signup/OAuth, se emite un
**nonce de continuación** — corto, opaco, de un solo uso, vida corta
(30 min) — que viaja por el único canal que sí sobrevive el cambio de
contexto: el `emailRedirectTo`/`next` que Supabase Auth ya reenvía a
través del link de confirmación de email o del callback de OAuth. Este
nonce **nunca es el token de activación real** (eso sigue sin salir de
la cookie httpOnly, ADR-0026 Sec 2.4 sigue vigente para el token en sí) —
sólo le permite al contexto de navegador que SÍ terminó la autenticación
volver a plantar la cookie de activación ahí, para que el resto del flujo
(cookie + sesión → mostrar el form → clic explícito en "Confirmar y
activar" → `claim_customer_activation()`) siga funcionando exactamente
igual que hoy, sin tocar esa parte.

**Mecánica**:

1. `/activar/continuar`, cuando encuentra la cookie pero no hay sesión,
   ya no arma `returnTo=/activar/continuar` a secas: primero llama a una
   RPC nueva `issue_activation_continuation()` (lee el token de la cookie
   server-side, nunca lo expone al cliente) que devuelve un nonce nuevo,
   y arma `returnTo=/activar/continuar?c=<nonce>`. Ese querystring viaja
   tal cual por todo el resto de la cadena que ya existe hoy
   (`login-form.tsx` → `/signup?returnTo=...` → `emailRedirectTo`/OAuth
   `next` → `/auth/callback`), sin tocar esos archivos más que para
   preservar el nuevo param igual que ya preservan `returnTo`.
2. `/auth/callback`, después de `exchangeCodeForSession()` (sesión ya
   creada en ESTE contexto), si el `next` trae `?c=<nonce>`, llama a
   `redeem_activation_continuation(p_nonce)` — valida sin usar y sin
   vencer, lo marca usado, devuelve el token real (sigue sin tocar al
   cliente) — y el propio route handler pone la cookie `activation_token`
   en ESTA respuesta (mismo mecanismo que `/activar/[token]/route.ts`)
   antes de redirigir a `/activar/continuar`, ahora sin el `?c=` (ya
   cumplió su función).
3. `/activar/continuar` corre exactamente como hoy: cookie + sesión →
   confirma el form → clic explícito → RPC de siempre. Nada de esta ADR
   toca esa mitad del flujo.

**Corrección post-review de seguridad (2026-09-28) — el párrafo original
de esta sección era falso, se deja tachado en el historial de git y
reemplazado por lo que sigue.** La premisa "la activación real sigue
exigiendo que la sesión coincida con el email del cliente invitado" **no
es cierta**: `claim_customer_activation()` (Phase 21) sólo exige
`auth.uid() is not null`, nunca compara email — un `Customer` gestionado
ni siquiera tiene columna de email para comparar. `security-engineer`
reprodujo en vivo que, con el nonce en mano, cualquier cuenta ajena
puede canjearlo, recibir el token real, y llamar a
`claim_customer_activation()` con éxito, quedando como dueña del
`Customer` de otra persona (ve sus reservas y pagos, reserva con su
plan).

**Por qué se acepta igual el diseño, con esto por escrito**: el nonce
equivale de hecho al token completo durante su vida útil — no es una
capa adicional de seguridad, es el mismo secreto viajando por un canal
más. Eso ya era cierto del token original (ADR-0026: quien lo intercepta
en el link de WhatsApp/email ya podía hacer exactamente esto). El nonce
viaja por canales de exposición equivalente (URL, logs de Vercel/Supabase,
el email de la propia persona) y con ventana más corta (30 min ilustrado
vs. 72h del token). **Riesgo residual nuevo y real, aceptado
explícitamente**: si alguien tipea mal su propio email al registrarse, el
mail de confirmación (con `?c=` embebido) le llega a un tercero, que con
un solo clic hereda la sesión y el cliente de la víctima — sin el nonce,
ese tercero nunca tenía la cookie httpOnly y no podía hacer nada. Se
acepta este riesgo por ser de exposición baja (typo de email + tercero
que efectivamente abre y hace clic, dentro de una ventana de 30 min) y
consistente con el modelo de amenaza ya aceptado del token base, no por
estar mitigado por un chequeo de email que no existe.

**Cambios de diseño exigidos por el gate de seguridad antes de LISTO**
(no opcionales, `security-engineer` los verificó funcionando contra la
base local antes de proponerlos):

1. **El token no se guarda en claro ni siquiera 30 minutos.** El diseño
   original dejaba `activation_token` en texto plano en la fila mientras
   el nonce no se usara — y si el nonce vencía sin canjearse (login con
   contraseña en el mismo navegador nunca pasa por `/auth/callback`), el
   texto plano quedaba para siempre, no 30 minutos. Se reemplaza por
   `activation_token_enc bytea`, cifrado con `pgp_sym_encrypt(token,
   nonce)` (pgcrypto, ya instalado) en `issue`, descifrado con
   `pgp_sym_decrypt` en `redeem` usando el nonce recién validado como
   clave — la base nunca tiene, en reposo, ninguna combinación de datos
   que por sí sola reconstruya el token (igual garantía que el hash en
   `customer_activations`).
2. **Límite de nonces vivos por activación** (`issue`, `for update` sobre
   la activación): antes de emitir uno nuevo, se borran los vencidos sin
   usar de esa activación (limpieza, además cierra el resto del punto 1
   si por algún motivo quedaran filas viejas) y se rechaza con
   `TOO_MANY_CONTINUATIONS` si ya hay 10 sin usar.
3. **`redeem_activation_continuation` verifica que la activación siga
   viva** (no revocada/ya reclamada/vencida) antes de devolver el token —
   mismo `INVALID_CONTINUATION` genérico si no lo está, para no filtrar
   cuál de los tres casos aplica.
4. **Defensa en profundidad, no bloqueante**: `revoke all on table
   customer_activation_continuations from anon, authenticated` explícito
   (además de RLS sin policies, que ya cierra el acceso vía PostgREST).

**Frontend, a implementar junto con lo ya delegado**: `Referrer-Policy:
no-referrer` en `/activar/continuar`, `/login` y `/signup` cuando la URL
trae `?c=`; nunca loguear `next` ni `c`; `/auth/callback` saca `c` de la
URL al redirigir; el allowlist `safeReturnTo` acepta `?c=` sin abrir un
open redirect.

**Alcance**: sólo la activación de clientes (ADR-0026). El mismo problema
podría existir en la invitación de equipo (ADR-0034, `/equipo/[token]`,
mismo patrón de cookie httpOnly) — no se toca en esta ADR; si se confirma
el mismo síntoma ahí, es una extensión directa del mismo mecanismo, a
evaluar por separado.

**Implementación**: delegada a `backend-engineer` (migración: tabla
`customer_activation_continuations` + las dos RPC, ahora con los 4 puntos
de arriba) y `frontend-engineer` (construcción del `returnTo` con `?c=`,
`/auth/callback`, más los puntos de frontend de arriba). Gate obligatorio
de `security-engineer` antes de desplegar — toca autenticación. Corrió
tres veces: 2026-09-28 NO LISTO (hallazgo de fondo sobre la premisa de
seguridad, corregido arriba), 2026-09-28 LISTO sobre backend corregido,
2026-09-29 LISTO sobre backend+frontend — con un hallazgo funcional que
se documenta y resuelve en ADR-0041.

---

## ADR-0041 — Confirmación de email por `token_hash`/`verifyOtp` (reemplaza el link PKCE para ese camino)

Fecha: 2026-09-29
Estado: **Aceptada**
Propuesta por: `security-engineer`, en el tercer pase del gate de
ADR-0040 (hallazgo funcional "F1"), decisión de diseño tomada por el
usuario entre las alternativas planteadas.

**Problema:** el gate de seguridad de ADR-0040 encontró que su mecanismo
de nonce, aun estando LISTO en seguridad, probablemente **no resuelve el
escenario más común** del bug original que motivó toda la ADR. Causa:
`@supabase/ssr` usa PKCE por defecto — `signUpWithPassword()`
(`app/actions/auth.ts`) genera un link de confirmación cuyo canje
(`exchangeCodeForSession(code)` en `/auth/callback`) exige una cookie
`code_verifier` que sólo existe en el navegador donde arrancó el signup.
Si el cliente toca el link de WhatsApp en el navegador embebido (contexto
A) y después abre el link de confirmación de email desde la app de Mail
(contexto B, sin relación de cookies con A), el intercambio de código
**falla antes de llegar a usar el nonce** — nunca se ejecuta el canje que
diseñó ADR-0040, y el usuario cae al mismo error de siempre. El nonce sí
funciona para el camino de Google OAuth cuando el login arranca de cero
en un único contexto (confirmado en vivo por `security-engineer`), pero
no para un link de email clickeado en un contexto distinto al que lo
generó — eso es estructural a PKCE, no algo que el nonce pueda arreglar
por sí solo.

**Alcance real de este problema**: no es exclusivo de la activación de
clientes (ADR-0026/0040). `signUpWithPassword()` es el único camino de
signup por contraseña de toda la plataforma — lo usa cualquier
`OrganizationMember` que se registra igual que un `Customer` gestionado.
El fix, por lo tanto, es una decisión de autenticación general, no un
parche acotado a la activación.

**Decisión**: reemplazar el link de confirmación de email basado en PKCE
por uno basado en **`token_hash` + `supabase.auth.verifyOtp()`**, que no
depende de ninguna cookie del navegador que originó el signup — el link
mismo (más el `token_hash` que trae) es autosuficiente para crear sesión
en cualquier navegador que lo abra, exactamente el caso de uso de un
link de confirmación por email.

**Mecánica**:
1. Template de confirmación de email de Supabase Auth pasa de
   `{{ .ConfirmationURL }}` (PKCE) a un link armado a mano:
   `{{ .SiteURL }}/auth/confirm?token_hash={{ .TokenHash }}&type=email&next={{ .RedirectTo }}`.
   `{{ .RedirectTo }}` sigue siendo el mismo `emailRedirectTo` que ya
   arma `signUpWithPassword()` hoy (incluye `next=<returnTo o
   returnTo+?c=nonce>` sin cambios).
2. Ruta nueva `frontend/app/auth/confirm/route.ts`: lee `token_hash` +
   `type` + `next`, llama `supabase.auth.verifyOtp({ token_hash, type })`.
   Si hay sesión, aplica **la misma lógica de nonce que ya vive en
   `/auth/callback`** (extraer y quitar `?c=` de `next` antes de construir
   cualquier redirect, canjear con `redeem_activation_continuation` si
   está presente, plantar la cookie de activación con
   `activationCookieOptions()`) — se factoriza en un helper compartido en
   vez de duplicar el bloque, ya que la lógica es idéntica.
3. `/auth/callback` (PKCE) queda intacto para Google OAuth, que no pasa
   por un link de email y no tiene esta limitación de la misma forma
   (confirmado en vivo: funciona si el login arranca de cero en un
   único contexto, que es el caso que cubre el nonce de ADR-0040).

**Verificado en vivo contra Supabase local** (`backend-engineer`,
2026-09-29, CLI 2.118.0, Postgres 17): `supabase/config.toml` define
`[auth.email.template.confirmation]` con `content_path =
"./supabase/templates/confirmation.html"` conteniendo exactamente el link
de la sección **Mecánica** arriba. Con `enable_confirmations = true`
temporalmente (local vive con `false` por defecto — ver nota debajo) y un
signup real contra `POST /auth/v1/signup?redirect_to=<url>` (`redirect_to`
va como **query param**, no en el body JSON — es lo que `supabase-js`
arma internamente a partir de `options.emailRedirectTo`), el mail que
aparece en Mailpit (`http://127.0.0.1:54324`, no Inbucket — el proyecto ya
migró a Mailpit aunque el env var viejo `INBUCKET_URL` se siga exponiendo
por compatibilidad) trae:

```
http://127.0.0.1:3000/auth/confirm?token_hash=<hash>&type=email&next=http%3a%2f%2flocalhost%3a3000%2fauth%2fcallback%3fnext%3d%2factivar%2fcontinuar%3fc%3dabc123nonce
```

Es decir: **`type` es literalmente `email`**, tal como asume el borrador
de la ADR — confirmado contra el comportamiento real, no asumido.
`frontend-engineer` puede usar `verifyOtp({ token_hash, type: "email" })`
sin ambigüedad. Se probó además el canje real: `POST /auth/v1/verify` con
`{"type":"email","token_hash":"<hash>"}` devuelve sesión completa
(`access_token`/`refresh_token`, `user_metadata.email_verified: true`) —
el mecanismo funciona de punta a punta, no sólo el shape del link.

**Hallazgo adicional (no introducido por esta ADR, pero bloqueaba poder
probarla): `additional_redirect_urls` con match exacto de URL completa.**
GoTrue arma `{{ .RedirectTo }}` a partir del `redirect_to` de la request,
pero **sólo si esa URL está en la lista de redirects permitidos**; si no,
cae en silencio al `site_url` pelado — sin `next`, sin el `?c=<nonce>` de
ADR-0040. El `config.toml` que ya estaba commiteado sólo tenía
`["https://127.0.0.1:3000"]` (ni siquiera el mismo esquema que
`site_url = "http://127.0.0.1:3000"`, y sin el `/auth/callback` real que
arma `signUpWithPassword()`), así que **este problema ya existía para el
link PKCE viejo también** — nadie lo había notado porque local corre con
`enable_confirmations = false` por defecto y nunca se ejerce este camino.
Se corrigió agregando patrones glob del mismo origen:
`additional_redirect_urls = ["https://127.0.0.1:3000",
"http://127.0.0.1:3000/**", "http://localhost:3000/**"]` — confirmado con
el mismo signup de arriba que con esto el `next` sobrevive completo
(incluye `/auth/callback?next=/activar/continuar?c=...`). **Esto hay que
verificarlo también en el dashboard de producción** (Authentication → URL
Configuration → Redirect URLs) — si esa lista no incluye el equivalente
del `/auth/callback` (y ahora `/auth/confirm`) reales de producción con
wildcard de query, el `next=` se pierde en producción igual que acá,
haciendo que el nonce de ADR-0040 nunca llegue aunque el resto de esta ADR
esté bien implementado. Queda como parte del mismo pendiente manual de
abajo.

`enable_confirmations` quedó **repuesto a `false`** tras la verificación —
no se cambió el default de desarrollo local, sólo se usó `true`
temporalmente para poder observar el mail real en Mailpit.

**Pendiente manual del usuario, no resoluble por CLI/migración** (mismo
patrón que las credenciales de Google OAuth en ADR-0017), proyecto Supabase
de producción `wgdlflhdjpqcxykblqme`:

1. **Dashboard → Authentication → Email Templates → "Confirm signup"** →
   reemplazar el HTML del cuerpo por (mecánica verificada en vivo con el
   template original en inglés; el copy se tradujo después, sin tocar
   ninguno de los tres parámetros — `token_hash`, `type=email`, `next` —
   ni la lógica que `/auth/confirm` espera, consistente con que el resto
   del producto está en español):
   ```html
   <h2>Confirmá tu cuenta</h2>

   <p>Hacé clic en el siguiente link para confirmar tu cuenta:</p>
   <p><a href="{{ .SiteURL }}/auth/confirm?token_hash={{ .TokenHash }}&type=email&next={{ .RedirectTo }}">Confirmar mi email</a></p>
   ```
   Sólo se toca **este** template. "Magic Link", "Reset Password", "Change
   Email Address" y los demás no participan del flujo de signup con
   contraseña y quedan intactos — el `type=email` de `verifyOtp` es
   específico a confirmación de signup; los otros templates de Supabase
   usan (y necesitan) otros valores de `type` (`magiclink`, `recovery`,
   `email_change`) si el día de mañana se migran por la misma razón, pero
   eso no es parte de esta ADR.
2. **Dashboard → Authentication → URL Configuration → Redirect URLs**:
   confirmar que la lista incluye el origen real de producción con
   wildcard de path/query (equivalente a lo que se agregó en
   `supabase/config.toml` para local), p. ej. `https://<dominio-prod>/**`.
   Sin esto, `{{ .RedirectTo }}` cae al `Site URL` pelado y el `next=`
   (incluido el nonce `?c=` de ADR-0040) se pierde — el hallazgo de arriba
   aplica igual en producción.
3. Hasta que el paso 1 se aplique, el link que reciben los usuarios sigue
   siendo el viejo (PKCE) aunque el código ya soporte `/auth/confirm` — no
   hay forma de que la app fuerce ese cambio de template de un proyecto
   Supabase administrado.

**Impacto**: `backend-engineer` actualizó `supabase/config.toml` (template
local vía `[auth.email.template.confirmation]` +
`supabase/templates/confirmation.html`, y el fix de
`additional_redirect_urls` de arriba) y redactó las instrucciones exactas
del cambio manual de dashboard para producción. `frontend-engineer`
implementa `/auth/confirm` reusando la lógica de nonce ya construida para
`/auth/callback`, con `type: "email"` confirmado (no `"signup"` ni otro
valor). Mismo gate obligatorio de `security-engineer` antes de desplegar
(es autenticación) — puntual sobre esta ruta nueva, no repite lo ya
revisado de ADR-0040. **Nota de seguridad para ese gate:** el riesgo
residual #8 de ADR-0040 (email mal tipeado → el nonce viaja en
`redirect_to` a un tercero) asumía que PKCE por sí solo frenaba el "click
directo" porque el tercero no tiene la cookie `code_verifier`; con
`token_hash`/`verifyOtp` esa fricción adicional desaparece — el link ya
no depende de ninguna cookie del navegador que lo generó, así que el
tercero que reciba el mail por typo **sí puede** confirmarlo con un solo
click. El TTL de 30 min, el single-use y el "Confirmar y activar" siguen
vigentes, pero ya no hay una segunda barrera accidental encima; vale que
`security-engineer` lo evalúe explícitamente en el gate, no asumir que
sigue cubierto por el mismo razonamiento de ADR-0040.

---

## ADR-0042 — Pipeline autónomo de bugs: GitHub Issues → fix → deploy, sin aprobación manual del usuario

Fecha: 2026-09-29
Estado: **Aceptada**
Propuesta por: usuario ("hoy me levantan los bugs a mí y yo te los escalo
a vos, quiero dejar de ser cuello de botella... llevarlos a prod y
atajarlos completamente vos sin depender de mí, backend front, todo").

**Problema:** hasta ahora, todo bug reportado en producción llegaba a
través del usuario (screenshot o descripción pegada en el chat), y toda
migración de schema quedaba esperando su merge + aprobación manual del
deploy en el GitHub Environment "production" — el patrón usado sin
excepción en ADR-0038, 0040 y 0041 de esta misma sesión. El usuario pasó
a ser, él mismo lo dice, el cuello de botella de todo el ciclo.

**Decisión:**

1. **Canal de entrada: GitHub Issues**, en `Reservaste/reservas` (el repo
   raíz de coordinación, no `backend`/`frontend` — el Orchestrator hace la
   triage y decide a qué repo(s) toca el fix, evitando que quien reporta
   tenga que adivinar si es un bug de front, de back, o de los dos).
   Reemplaza que el usuario pegue el reporte a mano en el chat.
2. **Autonomía completa de deploy, sin excepción de schema**: el
   Orchestrator mergea a `main` y aprueba el deploy de producción él
   mismo — incluidas migraciones de base de datos — para cualquier bug
   que entre por este pipeline. Reemplaza el patrón de "el usuario mergea
   + aprueba manualmente" que regía hasta ADR-0041 inclusive. **Decisión
   explícita del usuario, tomada con el trade-off dicho en estos
   términos**: la única red de seguridad real contra un cambio automático
   mal hecho llegando a producción sin revisión humana deja de existir a
   cambio de velocidad — el usuario la aceptó a sabiendas, eligiendo
   "sacarlo del todo" sobre la alternativa de mantenerlo sólo para
   schema.
3. **Lo que NO se relaja, porque es un gate entre agentes y no una
   dependencia del usuario** (`CLAUDE.md`, sección "Decisiones que
   SIEMPRE pasan por el Orchestrator" — esto sigue vigente, esta ADR sólo
   quita al usuario del loop, no las reglas del proyecto):
   - Gate obligatorio de `security-engineer` (LISTO/NO LISTO) para
     cualquier cambio de auth, roles, RLS, aislamiento multi-tenant, IDOR,
     datos públicos/privados o pagos — igual que en toda esta sesión.
   - Gate de `reviewer` antes de cualquier commit.
   - `qa-engineer` corre la suite de integración completa y tiene que dar
     verde antes de mergear — no sólo el archivo tocado.
   - Todo bug + fix se documenta en `docs/decisions.md` (si toca algo
     estructural) o en `.claude/knowledge/curation-inbox.md` (si es un
     bug puntual sin decisión de diseño detrás) — la autonomía no
     significa perder el registro histórico que el resto de esta sesión
     mantuvo sin excepción.
   - El Orchestrator sigue siendo quien decide si un bug reportado implica
     en realidad un cambio estructural (modelo de dominio, contratos de
     API, estrategia de slots/pagos/recurrencia) — en ese caso, aunque el
     deploy ya no espere aprobación humana, la decisión de diseño sigue
     necesitando su propia entrada de ADR antes de implementarse, igual
     que siempre.
4. **Mecánica de intake** (a definir/documentar en la siguiente entrada de
   este archivo una vez armada, ver "Pendiente" abajo): el Orchestrator
   necesita algún mecanismo de despertar periódico (cron/wakeup) para
   revisar issues nuevos sin que el usuario tenga que iniciar la
   conversación — a diferencia de todo lo anterior en esta sesión, que
   dependía de que el usuario abriera el chat y pegara el reporte.

**Riesgo aceptado explícitamente:** un bug mal diagnosticado, o un fix
con un error que ningún gate automático detecta, puede llegar a
producción (incluido un cambio de schema) sin que ningún humano lo haya
visto antes. Los gates de `security-engineer`/`reviewer`/`qa-engineer`
siguen siendo la única defensa — se vuelven, de hecho, más importantes
que antes, porque ya no hay una revisión humana de respaldo detrás
esperando al final de la cadena.

**Impacto:** cambia el rol operativo del Orchestrator de "coordina cuando
el usuario abre una conversación con un reporte" a "vigila un canal de
entrada y opera de punta a punta sin intervención". No cambia ninguna
regla de `CLAUDE.md` sobre decisiones estructurales ni sobre los gates
obligatorios entre agentes — sólo elimina al usuario como aprobador
humano del deploy.

**Pendiente:** definir y documentar el mecanismo concreto de polling/cron
para GitHub Issues (qué lo dispara, con qué frecuencia, qué pasa si dos
issues llegan a la vez) antes de considerar este pipeline operativo.

**Seguimiento 2026-09-29 — mecanismo elegido y primer paso ejecutado:**
en vez de un cron atado a esta sesión de chat (se corta al cerrar la
ventana, expira solo a los 7 días — no cumple "sin depender de mí"), se
opta por **GitHub Actions con `anthropics/claude-code-action`**, el
integration oficial de Anthropic para correr Claude Code disparado por
eventos de GitHub. Decisión tomada conscientemente con el trade-off de
costo explícito: a diferencia del cron de sesión (gratis), esto consume
API de Anthropic por token en cada corrida — el usuario lo aceptó sabiendo
que no es gratis, priorizando autonomía real sobre costo cero.

Como parte de esto, se ejecutó **ya** la mitad de esta ADR que no dependía
de infraestructura nueva: se sacó el `required_reviewers` del GitHub
Environment `production` de `Reservaste/backend` (antes exigía la
aprobación manual de `mathiasfernandez`, ver comentario en
`.github/workflows/ci.yml` sobre por qué existía ese gate desde Phase 22).
Efecto inmediato: el deploy de la migración de ADR-0040/0041 (PR #12/#13,
run `36588865913`), que llevaba varias horas esperando esa aprobación,
se destrabó solo y corrió — backup pre-migración tomado como siempre,
`deploy-migrations` en verde. Es la primera migración de este proyecto
desplegada a producción sin que ningún humano apruebe el paso final.

Falta todavía: que el usuario corra `/install-github-app` (requiere su
propia autorización de OAuth, no delegable) para instalar la GitHub App
de Claude a nivel organización sobre los 3 repos y cargar
`ANTHROPIC_API_KEY` como secret compartido; y que el Orchestrator escriba
el workflow específico de este proyecto (no el genérico que
`/install-github-app` propone por defecto) — checkout de los 3 repos,
prompt consciente de `CLAUDE.md`/`docs/agent-responsibilities.md`, y
disparo por label `bug` en vez de cualquier issue nuevo.

**Seguimiento 2026-09-29 — pipeline operativo.** El usuario completó
`/install-github-app` (GitHub App instalada a nivel organización
`Reservaste`, autenticando con el token de su propia suscripción —
`CLAUDE_CODE_OAUTH_TOKEN`, no una API key separada, decisión consciente
de costo: usa el plan que ya paga en vez de facturación nueva). El
Orchestrator escribió `.github/workflows/claude.yml` en
`Reservaste/reservas` (rama `main`): dispara sólo con la etiqueta `bug`
en un Issue (nunca en cualquier Issue nuevo — la etiqueta es la válvula
manual de "esto entra al pipeline", aplicable sólo por alguien con
permiso de escritura en el repo, lo que además resuelve sin configuración
extra el caso de un vendedor/tercero sin acceso reportando bugs: puede
abrir el Issue en el repo público, pero no puede etiquetarlo él mismo).
El prompt de ese workflow reproduce el rol de Orchestrator completo
(lee `CLAUDE.md`, delega a subagentes, exige los mismos gates de
`qa-engineer`/`reviewer`/`security-engineer`, mergea y determina si hay
deploy automático sin aprobación humana, documenta el resultado).
Dos pushes/merges de infraestructura (el workflow en sí y el PR que lo
llevó a `main`) fueron bloqueados por el clasificador de auto mode de
Claude Code del lado del Orchestrator ("Create Unsafe Agents") — un
control de seguridad del propio entorno, no de GitHub. No se intentó
rodear: el usuario ejecutó esos dos pasos (push + merge) directamente
desde su propia terminal. Secret cross-repo (`CROSS_REPO_PAT`, un fine-
grained PAT con permisos de Contents/Issues/Pull requests/Workflows
sobre los 3 repos) cargado por el usuario — con un incidente menor en el
camino: el primer valor del token se pegó en el chat (por lo tanto
expuesto) y se revocó/regeneró antes de cargarlo, mismo criterio que la
regla de credenciales de ADR-0039. Pipeline confirmado operativo (los 2
secrets existen, el workflow está en `main`, la etiqueta `bug` existe por
default en GitHub) pero **todavía no procesó ningún bug real** al momento
de este seguimiento.

---

## ADR-0043 — Se elimina la confirmación de email del signup

Fecha: 2026-09-29
Estado: **Aceptada**
Propuesta por: usuario ("no quiero ningún mecanismo que dependa del envío
de emails porque ahí se me va a ir mucho costo"), alcance confirmado
explícitamente como "sacar la confirmación de email del todo" (no sólo
evitar SMTP pago) tras pregunta directa del Orchestrator sobre qué,
puntualmente, le preocupaba.

**Problema:** ADR-0041 (confirmación por `token_hash`/`verifyOtp`)
resolvía que el link de confirmación de email sobreviviera un cambio de
navegador, pero seguía dependiendo de que Supabase Auth mande un email
por cada signup — con el servicio por defecto de Supabase (gratis pero
con un rate limit bajo, no apto para producción según su propia
documentación) o con SMTP propio (Resend u otro, con costo real más allá
del tier gratis si el volumen crece). El usuario, vendiendo el proyecto a
un precio bajo, no quiere que ningún costo de infraestructura escale con
la cantidad de signups — prefiere sacar la dependencia de raíz antes que
optimizar el proveedor de email.

**Decisión:** se deshabilita "Confirm email" en Supabase Auth. `signUp()`
devuelve sesión inmediatamente, sin mandar ningún email ni esperar
ninguna confirmación — ni para clientes gestionados (ADR-0026) ni para
`OrganizationMember` (dueños/staff), porque `signUpWithPassword()` es el
mismo código compartido para los dos casos (ya señalado en ADR-0041).

**Efecto en cadena sobre ADR-0040/0041:**
- **ADR-0041 queda sin objeto y se da de baja.** Sin ningún email de
  confirmación, no existe ningún link que canjear — `/auth/confirm`,
  `frontend/lib/confirm-next.ts` y el template
  `supabase/templates/confirmation.html` quedan sin ningún caller posible
  y se eliminan (no se dejan como código muerto).
- **ADR-0040 (nonce de continuación) se ACOTA, no se elimina.** Seguía
  necesitándose para el camino de Google OAuth (Google fuerza salir del
  WebView embebido de WhatsApp a un navegador del sistema, cambio de
  contexto real, sin relación con email). Para el camino de
  email/contraseña, el problema que motivó ADR-0040 desaparece por
  completo con esta ADR: sin redirección a ningún lado (ni a Mail, ni a
  un link externo), la cuenta queda activa en el mismo navegador/contexto
  donde se completó el formulario de signup — no hay salto de contexto
  que perder. El mecanismo de nonce sigue viviendo (RPC
  `issue_activation_continuation`/`redeem_activation_continuation`,
  helper `activation-continuation.ts`) pero ejercitado sólo por
  `/auth/callback` (OAuth), nunca por `/auth/confirm` (que deja de
  existir).

**Trade-off de seguridad, aceptado explícitamente por el usuario** (se le
explicó antes de decidir, ver pregunta del Orchestrator en esta misma
conversación): sin confirmación de email, cualquiera puede registrarse
con un email que no le pertenece (typo propio o ajeno, o a propósito) sin
que el dueño real de esa casilla se entere ni pueda impedirlo. Dos
poblaciones distintas, con exposición real distinta:
- **`OrganizationMember` (dueños/staff)**: es donde el riesgo pega más —
  alguien podría "ocupar" el email de otra persona antes de que esa
  persona intente registrarse ahí, o registrarse con un email inventado
  sin límite. Sin mitigación nueva en esta ADR más allá de lo que ya
  existía (unicidad de email a nivel de Supabase Auth evita que dos
  cuentas compartan el mismo email, así que no hay *account takeover* de
  una cuenta ya existente — el riesgo es "alguien se adelanta a crear la
  cuenta con tu email", no "alguien entra a tu cuenta ya creada").
- **Clientes gestionados (ADR-0026)**: la exposición real es menor de lo
  que parece a primera vista — `claim_customer_activation()` (Phase 21,
  sin cambios en esta ADR) **nunca comparó email** como parte de su
  chequeo de seguridad (hallazgo del gate de ADR-0040, ya corregido en la
  ADR misma) — la autorización real siempre fue "tener el token del link
  de WhatsApp" + "clic explícito en Confirmar y activar", nunca la
  confirmación de email. Sacar la confirmación de email no cambia la
  superficie de seguridad de este flujo particular, sólo la vuelve
  explícita en vez de una barrera accidental que ya se había documentado
  como no-existente en ADR-0040.

**Implementación**: `backend-engineer` deshabilita "Confirm email" en
`supabase/config.toml` (local) y redacta las instrucciones del cambio
manual equivalente en el dashboard de producción (Authentication →
Providers → Email → "Confirm email", toggle off) — mismo patrón que ya
usa este proyecto para cambios que sólo se pueden hacer a mano en el
proyecto Supabase administrado (ADR-0017, ADR-0041). `frontend-engineer`
elimina `/auth/confirm`, `confirm-next.ts` y su test, y ajusta
`signUpWithPassword()` si el `notice` de "revisá tu email" deja de
aplicarse (redirige directo, como cualquier login exitoso). Gate
obligatorio de `security-engineer` antes de desplegar — toca
autenticación, mismo criterio que toda esta sesión, aunque el trade-off
principal ya fue explicado y aceptado por el usuario antes de esta
decisión.

**Implementado y verificado en vivo (`backend-engineer`, 2026-09-29, CLI
2.118.0)**: `enable_confirmations = false` en `[auth.email]` de
`supabase/config.toml` ya era el default de desarrollo local desde
ADR-0041 (se usaba `true` sólo temporalmente para observar el mail en
Mailpit y se revertía después de cada prueba) — con esta ADR pasa a ser
el estado **permanente y definitivo** del proyecto, documentado como tal
inline. Se eliminó `[auth.email.template.confirmation]` (la sección que
ADR-0041 había agregado) junto con `supabase/templates/confirmation.html`
— sin ningún email de confirmación, no queda ningún caller posible del
template. El comentario de `additional_redirect_urls` se reescribió: la
lista de patrones (`http://127.0.0.1:3000/**`, `http://localhost:3000/**`)
sigue haciendo falta porque `signInWithOAuth()` arma el mismo
`redirect_to=/auth/callback?next=<...>` que necesitaba el link de email
viejo — sin la lista, GoTrue pierde el `?c=<nonce>` de ADR-0040 igual para
Google OAuth — pero el motivo documentado pasa a ser exclusivamente OAuth,
no el flujo de email que ya no existe. Las RPC
`issue_activation_continuation`/`redeem_activation_continuation` y la
tabla `customer_activation_continuations` (ADR-0040) no se tocaron.

Verificado en vivo contra Supabase local reiniciado con el config nuevo
(`supabase stop && supabase start`, sin esto GoTrue sigue el proceso
viejo en memoria): `POST /auth/v1/signup` real (vía `supabase-js`,
`auth.signUp({ email, password })`, sin `admin.createUser`) devuelve
`session`/`access_token` no nulos en la misma respuesta, sin generar
ningún mail en Mailpit. Se ejecutó además el flujo completo de activación
de cliente gestionado (ADR-0026) de punta a punta contra RPCs reales,
en el mismo cliente/contexto que hizo el `signUp()`: `create_managed_customer`
→ `issue_customer_activation` (como owner) → `signUp()` del cliente
(sesión inmediata, sin mail) → `claim_customer_activation(token)` con esa
misma sesión → `status: "OK"` y `customers.profile_id` apuntando al
usuario recién creado — nunca se pasó por `/auth/confirm` (ya no existe)
ni por `issue_activation_continuation`/`redeem_activation_continuation`
(el nonce de ADR-0040), confirmando que ese mecanismo queda acotado a
Google OAuth como preveía esta ADR.

**Tests**: no se encontró ningún test de integración de `backend/` que
llame a `auth.signUp()` directamente ni que asuma `data.session === null`
tras un signup — todos los fixtures de usuario (`test/helpers.ts`,
`createSignedInUser`) usan `admin.auth.admin.createUser({ email_confirm:
true })` + `signInWithPassword()`, que ya bypaseaba la confirmación por
vía de service role independientemente de esta ADR. No hizo falta ajustar
ningún test existente. Suite completa corrida contra una base reseteada
(`npx supabase db reset`, 46 migraciones aplicadas limpio): unit
(`npm test`) **62/62 verdes**. Integración (`npm run test:integration`,
**350 tests en total**) corrida dos veces en paralelo (modo por default):
la primera con **349/350** verdes (1 falla,
`phase34.permission-boundary-fixes.test.ts`, `Hook timed out in 30000ms`
en el `afterAll` de limpieza), la segunda con **349/350** verdes (1 falla
distinta, `phase32.configurable-roles.test.ts`, `Test timed out in
20000ms`). Ambos archivos, corridos en aislado inmediatamente después,
pasan limpio (`phase34`: 14/14 en 88s, un tercio del tiempo que bajo
contención) — es la misma contención de GoTrue por hashing de contraseñas
concurrente entre archivos que ya documenta el comentario de
`hookTimeout`/`testTimeout` en `vitest.integration.config.ts`
(preexistente a esta ADR, no introducida por ella: cambiar
`enable_confirmations` no agrega ningún hash ni ninguna llamada de red
nueva al signup, si acaso quita una). Confirmado eliminando la variable
de contención: una tercera corrida con `--no-file-parallelism` (sin
paralelismo entre archivos) dio **350/350 verdes, 36/36 archivos**, en
704s — cero fallas relacionadas con esta ADR. `npm run typecheck` verde.

**Corrección post-review de seguridad (2026-09-29) — el trade-off
explicado al usuario antes de decidir era más chico que el real, se deja
por escrito acá para que el historial de la decisión sea preciso.** El
razonamiento original de esta ADR decía que sin confirmación de email "no
hay *account takeover* de una cuenta ya existente — el riesgo es que
alguien se adelanta a crear la cuenta con tu email". El gate de
`security-engineer` encontró, y verificó en vivo contra Supabase local,
que **sí hay una forma real de terminar con acceso de OWNER a un negocio
ajeno**, por tres caminos distintos:

1. **Ocupación de email + invitación de equipo**: `invite_member_by_email()`
   (Phase 32) le da membresía — incluido rol OWNER — a quien tenga
   registrado ese email en `auth.users`, sin ninguna prueba de que sea la
   persona real. Un atacante que se registra primero con el email
   previsible de un futuro dueño/empleado queda con acceso apenas alguien
   lo invita por ese email.
2. **Secuestro vía Google OAuth**: GoTrue sólo borra identidades no
   confirmadas al vincular una cuenta nueva de OAuth a un email ya
   registrado quen no está confirmado — con `enable_confirmations=false`
   toda cuenta por contraseña queda confirmada de una, así que esa
   protección deja de aplicar. Un atacante que se registra por contraseña
   con el Gmail de una futura dueña queda con la cuenta vinculada cuando
   ella entra por primera vez con "Continuar con Google" — ella no ve
   ningún error, termina dentro de la cuenta del atacante.
3. **Usuarios viejos sin confirmar, en el instante de apagar el toggle en
   producción**: cualquiera que haga `signUp()` con el mismo email de una
   cuenta que ya existía sin confirmar recibe sesión como ESE `user.id`
   (mismo UUID) — si esa cuenta ya tenía membresías/clientes vinculados,
   quedan expuestos apenas se apaga el toggle.

Adicional: `claim_team_invitation()` (ADR-0034) pierde su segunda barrera
— la invitación por email ya no prueba posesión de la casilla, sólo
conocimiento del email (rompe la regla 1 del gate de seguridad de
ADR-0034, documentada en `docs/security.md`).

**Decisión, presentado el riesgo real al usuario: se aplican las
mitigaciones sin costo de email** (ninguna reintroduce envío de mails ni
SMTP):

1. Se revoca el `grant execute` de `invite_member_by_email()` y
   `enroll_customer_by_email()` — quedan sin camino de invocación desde
   `anon`/`authenticated`. Ya existen reemplazos que no dependen de
   "email = identidad verificada": la invitación de equipo por token de
   ADR-0034 (`/equipo/[token]`) y la activación de cliente gestionado por
   link de WhatsApp de ADR-0026. La UI que llamaba a las versiones por
   email se saca o se redirige al flujo por token.
2. Trigger nuevo sobre `auth.identities`: cuando se vincula una identidad
   no-email (Google) a un usuario que ya tiene una identidad de
   email/contraseña, si esa cuenta fue creada por signup público (no por
   invitación/activación administrada) se rechaza la vinculación o se
   fuerza a tratar la cuenta como si no tuviera contraseña utilizable —
   mismo comportamiento que GoTrue ya aplica hoy para cuentas no
   confirmadas, replicado a mano porque `enable_confirmations=false` lo
   desactiva.
3. **Antes de apagar el toggle en producción**: auditar
   `auth.users where email_confirmed_at is null`. Cuentas sin ningún dato
   vinculado (membership/customer) se pueden borrar sin más; cuentas con
   datos vinculados se resuelven a mano (confirmarlas explícitamente
   corta el vector, porque `signUp()` vuelve a dar `user_already_exists`
   para un email ya confirmado).
4. `docs/security.md` (regla 1 del gate de ADR-0034) se actualiza:
   la invitación de equipo por email pasa a ser un factor de
   *conocimiento*, no de *posesión* — documentado como cambio de modelo
   de amenaza, no como bug.
5. No bloqueante, aplicado igual por ser gratis: mensaje de error
   genérico en español para `user_already_exists` (evita enumeración de
   emails registrados), y Turnstile (`[auth.captcha]`, gratis, soportado
   nativamente por Supabase) como mitigación de creación masiva de
   cuentas.

**Riesgo que queda abierto y se acepta explícitamente**: el mensaje "Tu
cuenta todavía no está confirmada" en `signInWithPassword()` no es código
muerto — lo siguen recibiendo cuentas viejas sin confirmar hasta que se
complete la auditoría del punto 3. El texto se actualiza para no decirle
a esa persona que busque un mail que ya no se va a reenviar.

**Implementación**: delegada a `backend-engineer` (revocar los dos
`grant execute`, trigger de `auth.identities`, query de auditoría para
producción, captcha) y `frontend-engineer` (sacar/redirigir la UI de
invitación directa por email, mensaje de error genérico). Segundo pase de
`security-engineer` obligatorio sobre estos cambios antes de release —
el primer gate ya dio la mecánica base por buena, esto es incremental.

**Implementado (`backend-engineer`, 2026-09-29, puntos 1, 2, 3 y 5 de la
lista de arriba)**: migración `backend/supabase/migrations/20260929140000_phase41_adr0043_post_review_hardening.sql`.
Detalle técnico completo en `docs/database.md` ("Fase 41"); acá sólo lo
que hace falta para decidir y para que `frontend-engineer`/`security-engineer`
sepan qué esperar.

1. `revoke execute ... from anon, authenticated, service_role` sobre
   `invite_member_by_email()` y `enroll_customer_by_email()` — no se
   borraron. Investigado antes: ningún caller de este repo las invocaba
   vía `service_role` (todos usan la sesión del usuario), y de hecho
   `service_role` ya no tenía `EXECUTE` sobre ninguna de las dos desde la
   Fase 19 (revocó el grant heredado del default de Supabase); el revoke
   explícito documenta ese estado y evita que una migración futura se lo
   devuelva por accidente.

   **Rompe `frontend/app/actions/admin.ts` (`enrollCustomer()`,
   `inviteMember()`) de inmediato** — las dos pasan a devolver
   `42501 permission denied` en vez de su comportamiento actual. Es la
   consecuencia esperada y ya prevista arriba ("la UI que llamaba a las
   versiones por email se saca o se redirige al flujo por token"), pero
   **no desplegar esta migración sin coordinar el orden con ese cambio de
   frontend** — si se despliega primero, esos dos botones del panel
   empiezan a fallar en producción hasta que `frontend-engineer` los saque
   o los redirija.

2. Trigger `auth_identities_block_oauth_hijack` sobre `auth.identities`
   (`before insert`): bloquea vincular una identidad no-`email` (Google) a
   una cuenta que ya tiene una identidad `email`/contraseña. No distingue
   "signup público" de "invitación/activación administrada" — con
   `enable_manual_linking = false` (ya así en `supabase/config.toml`) no
   existe ningún flujo soportado por el que alguien agregue Google a su
   propia cuenta de forma deliberada, así que cualquier insert de este
   tipo es, por definición, el camino automático vulnerable. Verificado en
   vivo contra Supabase local: un insert simulando el secuestro (identidad
   `google` para un `user_id` con contraseña preexistente) falla con
   `OAUTH_LINK_BLOCKED_EXISTING_PASSWORD_IDENTITY`; un signup nuevo por
   Google (sin identidad previa) no se ve afectado.

   **Riesgo residual aceptado, documentado explícitamente**: esto es
   fail-closed, no fail-silent — la persona real recibe un error genérico
   de Postgres al hacer "Continuar con Google" por primera vez con un
   email ya ocupado, en vez de quedar vinculada en silencio a la cuenta
   ajena (que era el bug). El texto de error que ve el usuario depende de
   cómo GoTrue/`frontend/app/actions/auth.ts` traduzcan ese fallo — no
   evaluado en esta fase porque el flujo de Google real (con credenciales
   de Cloudflare/Google de verdad) no se puede ejercitar completo contra
   Supabase local; sólo se verificó el `insert` a nivel de base. Si
   `security-engineer` o `frontend-engineer` observan un mensaje crudo de
   Postgres llegando al usuario en este camino, es un seguimiento de UI,
   no una regresión de este trigger.

3. **Query de auditoría para producción, lista para copiar/pegar** (no
   corrida contra producción por `backend-engineer` — la corrida real la
   hace el usuario o el Orchestrator, con supervisión, antes de apagar
   `enable_confirmations` en el dashboard):

   ```sql
   -- 1. Todas las cuentas sin confirmar (candidatas al problema del
   --    camino 3: mismo email, mismo user_id, expuestas apenas se apaga
   --    el toggle).
   select id, email, created_at
   from auth.users
   where email_confirmed_at is null
   order by created_at asc;

   -- 2. De esas, cuáles tienen algo vinculado (membership u/o Customer) --
   --    esas NO se pueden borrar sin más, hay que resolverlas a mano
   --    (p.ej. confirmarlas explícitamente, lo que corta el vector porque
   --    signUp() vuelve a dar user_already_exists para un email
   --    confirmado). Las que no aparecen en ningún lado de este segundo
   --    resultado se pueden borrar directo.
   select
     u.id,
     u.email,
     u.created_at,
     (select count(*) from public.organization_members om where om.profile_id = u.id) as membership_count,
     (select count(*) from public.customers c where c.profile_id = u.id) as customer_count
   from auth.users u
   where u.email_confirmed_at is null
     and (
       exists (select 1 from public.organization_members om where om.profile_id = u.id)
       or exists (select 1 from public.customers c where c.profile_id = u.id)
     )
   order by u.created_at asc;
   ```

5. `[auth.captcha]` habilitado en `supabase/config.toml` con
   `provider = "turnstile"` y el secreto de prueba público que Cloudflare
   documenta para automatizar tests sin navegador ("always passes", no es
   un secreto real). **Confirmado en vivo, hallazgo importante**: GoTrue
   exige el `captcha_token` tanto en `/signup` como en
   `/token?grant_type=password` — **no sólo en signup**. Esto significa
   que habilitar este toggle en el dashboard de producción, sin que
   `frontend/app/actions/auth.ts` mande un token real de Turnstile en
   `signUp()` **y** en `signInWithPassword()`, deja **todo login por
   contraseña roto**, no sólo el alta de cuentas nuevas.

   **Aviso explícito para el Orchestrator, tal como pedía la tarea**: esto
   es trabajo de `frontend-engineer` — agregar el widget de Turnstile
   (Cloudflare, Site Key público) al formulario de signup/login y pasar el
   token resultante como `options.captchaToken` en `signUp()` y
   `signInWithPassword()`. Hasta que eso exista, **no tocar el toggle
   equivalente en el dashboard de producción** (Authentication → Attack
   Protection o la sección equivalente) — local sí quedó con el toggle
   activo porque `backend/test/helpers.ts` ya manda un `captchaToken` fijo
   (la clave de prueba acepta cualquier valor no vacío), pero un usuario
   real de producción no tiene ese atajo.

   Producción además necesita un Site Key + Secret Key reales (gratis) de
   <https://dash.cloudflare.com/?to=/:account/turnstile> — el Secret Key
   se carga en el dashboard de Supabase, el Site Key lo necesita
   `frontend-engineer` para el widget.

**Verificación en vivo (`backend-engineer`, 2026-09-29)**: ataques
reproducidos y confirmados bloqueados contra Supabase local reseteado
(`npx supabase db reset` con la migración aplicada) —
*email-squatting + invitación*: `enroll_customer_by_email()`/
`invite_member_by_email()` devuelven `42501` para un `OWNER` autenticado
real (antes hubieran dado membresía/acceso a Customer sin más). *Robo de
cuenta vieja sin confirmar vía Google*: un insert de identidad `google`
sobre un `user_id` con contraseña preexistente fue rechazado por el
trigger; el mismo insert para una cuenta Google nueva (sin contraseña
previa) no se vio afectado.

**Tests**: la migración obligó a actualizar 5 archivos de
`backend/test/` que usaban las dos RPC revocadas como fixture o como
sujeto de prueba (`phase8.admin-operations.test.ts`,
`phase10.plans.test.ts`, `phase13.branding.test.ts`,
`phase32.configurable-roles.test.ts`,
`phase33.team-invitations.test.ts`) — los que las usaban sólo para armar
un `Customer`/`STAFF` de prueba pasaron a un insert directo con el cliente
del `OWNER` (misma política `organization_members_write_owner` que ya
probaba `phase32`); los que probaban el comportamiento propio de las RPC
pasaron a afirmar `error.code === '42501'`. `npm run typecheck` verde.
Suite completa (`npx supabase db reset` + `npx vitest run --config
vitest.integration.config.ts --no-file-parallelism`, sin paralelismo entre
archivos por la misma contención de GoTrue que ya documenta ADR-0043 base):
**350/350 tests en verde, 36/36 archivos** (mismo total que antes de esta
fase — los cambios en `phase8`/`phase32` se compensan: `phase8` pasa de 3 a
2 tests, `phase32` pasa de 1 a 2). `npm test` (unit, paquete de dominio):
**62/62 verdes**, sin cambios — ningún test unitario toca RPCs ni
`auth.*`.

**Pendiente, no cubierto por esta fase**: `frontend-engineer` — sacar o
redirigir la UI de `enrollCustomer()`/`inviteMember()` en
`frontend/app/actions/admin.ts` (coordinar el orden de deploy con la
migración de arriba), agregar el widget de Turnstile + `captchaToken` a
`signUp()`/`signInWithPassword()` antes de habilitar captcha en
producción, y el mensaje de error genérico de `user_already_exists`
(BAJO-1) si todavía no está — `frontend/app/actions/auth.ts` ya tenía un
comentario citando ADR-0043 punto 5 para ese mensaje al momento de este
trabajo, no verificado en detalle por no ser parte del alcance de
`backend-engineer`. Segundo pase de `security-engineer` obligatorio sobre
todo lo de arriba antes de release.

**Checklist único de deploy (2026-09-30, consolidado por pedido de
`reviewer` en el review final — la información ya estaba correcta pero
repartida en 6 lugares distintos: el comentario de la migración,
`docs/security.md`, `.github/workflows/ci.yml`, `deploy/README.md`; esta
es la versión canónica, copiable tal cual).** Todo lo de código ya pasó
dos gates de `security-engineer` (LISTO) y `reviewer` (LISTO). Lo que
sigue son pasos de ejecución, varios manuales:

**Paso 0 — antes de tocar cualquier repo (manual, usuario):**
1. Crear el widget en [Cloudflare Turnstile](https://dash.cloudflare.com/?to=/:account/turnstile)
   (gratis) con los hostnames reales de producción. Guardar Site Key +
   Secret Key.
2. Cargar la Site Key como secret `NEXT_PUBLIC_TURNSTILE_SITE_KEY` en
   GitHub → `Reservaste/frontend` → Settings → Secrets → Actions. (Sirve
   provisoriamente la key de test `1x00000000000000000000AA` acá también
   — ambas funcionan mientras el captcha de Supabase siga apagado en
   producción; sólo hace falta la real antes del Paso 6.)
3. Confirmar en el dashboard de Supabase de producción (proyecto
   `wgdlflhdjpqcxykblqme`) → Authentication → Providers que "manual
   linking" sigue apagado — el diseño del trigger anti-secuestro depende
   de eso.

**Paso 1 — backend (mixto, yo abro el PR/migración, usuario aprueba el deploy):**
4. Release del repo `backend/` (migración `20260929140000_phase41_...`):
   PR development→main, CI en verde, y el usuario aprueba manualmente el
   job `deploy-migrations` (Environment "production", como toda migración
   de schema en este proyecto).
5. Verificar en producción: `select has_function_privilege('anon',
   'invite_member_by_email(uuid,text,text)', 'EXECUTE')` da `false` (o el
   nombre exacto de firma que tenga en ese momento), y que existe el
   trigger `auth_identities_block_oauth_hijack` sobre `auth.identities`
   (`select tgname from pg_trigger where tgrelid =
   'auth.identities'::regclass`).
   
   **Ventana funcional esperada y aceptada** entre este paso y el Paso 3:
   los botones viejos de "Ya tiene cuenta"/"Cliente con cuenta existente"
   ya no existen en el código nuevo de frontend, pero ese código todavía
   no está desplegado — cualquiera que siga en la versión vieja del
   frontend va a ver esos botones fallar con error genérico si los usa.
   No es un problema de seguridad, es cosmético, y se cierra en el Paso 3.

**Paso 2 — auditoría de cuentas viejas (manual, usuario, con mi ayuda si la pide):**
6. Correr en producción la query de auditoría (ver sección
   "Implementado (backend-engineer...)" de esta misma ADR más arriba,
   punto 3, para el SQL exacto): `auth.users` sin `email_confirmed_at`,
   cruzado contra `organization_members`/`customers`. Cuentas sin ningún
   dato vinculado se borran directo; cuentas con datos vinculados se
   resuelven a mano (confirmarlas explícitamente corta cualquier vector,
   porque `signUp()` vuelve a dar `user_already_exists` para un email ya
   confirmado). **Este paso borra/modifica datos reales de producción —
   no lo automatizo sin que el usuario lo vea y decida caso por caso.**

**Paso 3 — apagar confirmación de email (manual, usuario):**
7. Dashboard de producción → Authentication → Providers → Email → apagar
   "Confirm email". **No antes del Paso 2** (cuentas viejas sin resolver
   quedarían expuestas al vector ALTO-3 apenas se apaga) **ni después del
   Paso 4** (el frontend nuevo asume sesión inmediata tras signup; si el
   toggle sigue prendido, el signup se rompe porque no hay ningún link de
   confirmación al que volver — `/auth/confirm` ya no existe en el código
   nuevo).

**Paso 4 — frontend (mixto, yo mergeo, despliega solo):**
8. Inmediatamente después del Paso 3: merge de `frontend/` a `main` (yo
   lo hago, sin aprobación manual — no tiene schema, mismo patrón ya
   usado en esta sesión). Despliega automático.

**Paso 5 — verificación conjunta en producción (usuario + yo):**
9. Signup por contraseña real → sesión inmediata, sin pantalla de
   "revisá tu email".
10. Login por contraseña real.
11. "Continuar con Google" con una cuenta de Google que nunca se registró
    antes en la plataforma → funciona normal.
12. "Continuar con Google" con el Gmail de una cuenta que YA tiene
    contraseña en la plataforma (probar a propósito el caso que el
    trigger tiene que bloquear) → vuelve a `/login` sin crear sesión ni
    vincular nada (no hay mensaje de error visible, es el comportamiento
    esperado — `/login` no expone el motivo).

**Paso 6 — captcha real (manual, usuario, sólo al final):**
13. Recién acá, con la Site Key real ya desplegada (si en el Paso 0 se
    usó la de test, esto implica un redeploy del frontend con la key
    real primero) y el M1 (reset del widget) ya verificado en vivo (hecho,
    `reviewer` lo confirmó en el review final de esta corrección):
    habilitar `[auth.captcha]` en el dashboard de producción con el
    Secret real de Turnstile — nunca el de prueba. Probar login y signup
    a mano de inmediato. Plan de rollback si algo falla: apagar el
    toggle, vuelve al estado sin captcha (mismo riesgo residual de
    creación masiva que ya está aceptado hasta este paso, nada peor).

**Hasta que se complete el Paso 6, no hay protección contra creación
masiva de cuentas — conviene no demorarlo mucho, pero no bloquea nada de
lo anterior.**

**Cerrado — 2026-09-30, los 7 pasos completos.** Auditoría (Paso 2): cero
cuentas sin confirmar en producción, sin nada que resolver. "Confirm
email" apagado (Paso 3). Frontend desplegado con las mitigaciones
completas (Paso 4) y verificado en producción real (Paso 5: signup sin
confirmación, login, mensaje genérico de email duplicado, todo en verde
contra `https://161-35-63-60.sslip.io` con Playwright). Turnstile real
(Site Key + Secret Key del usuario) cargado y activado (Paso 6),
verificado con un signup+login manual real del usuario tras el deploy —
el widget real bloqueaba automatización estándar de Playwright (esperado,
es el captcha funcionando), así que la confirmación final la hizo el
usuario a mano. Ajuste cosmético adicional el mismo día: el widget se
fijó en `theme: "light"` (antes seguía el tema del sistema).

ADR-0043 queda completamente implementada, desplegada y verificada de
punta a punta, sin pasos pendientes.

---

## ADR-0044 — Recursos exclusivos: anti-solapamiento a nivel de base de datos

Fecha: 2026-09-30
Estado: **Aceptada**
Propuesta por: usuario ("necesito revisar toda la lógica que hay y
cuánta es custom y cuánta no" — auditoría de qué tan genérico es el
producto de cara a vender a una barbería), diseñada por `Plan` con tres
rondas de investigación previa (modelo `ScheduleRule`/`SlotOccurrence`,
modelo `Resource`, patrón de generación).

**Problema:** una auditoría completa del sistema (ver resumen ejecutivo
en la conversación, no repetido acá) encontró que el gap más riesgoso
operativamente para un negocio de turno individual (ej. una barbería) es
que **no existe ningún chequeo, a ningún nivel, que impida que el mismo
`Resource` (ej. un barbero) quede reservado dos veces a la misma hora en
servicios distintos**. El único `EXCLUDE USING gist` de todo el proyecto
protege pagos (`daterange` de períodos), no reservas. `book_slot()` y
`can_customer_book()` sólo validan capacidad de la ocurrencia puntual, y
`generate_slot_occurrences_for_rule()` no consulta otras reglas del mismo
recurso al generar.

**Decisión — campo genérico en `Resource`, respaldado por un exclusion
constraint en base de datos** (no sólo una validación en la RPC, porque
el generador, el cron y los triggers insertan por su cuenta sin pasar por
un único punto de entrada):

- `resources.is_exclusive boolean not null default false` — "se ocupa de
  a uno, no admite turnos superpuestos". Genérico, sin ninguna palabra de
  rubro (sirve igual para un profesional, una camilla, una cancha de
  1-a-1).
- `slot_occurrences.resource_is_exclusive boolean not null default
  false` — denormalizado desde `resources.is_exclusive` vía trigger
  (`BEFORE INSERT OR UPDATE OF resource_id`, y un segundo trigger `AFTER
  UPDATE OF is_exclusive` en `resources` que propaga sólo a ocurrencias
  futuras `ACTIVE` — el pasado no se revalida). Necesario porque un
  `EXCLUDE` no puede mirar otra tabla.
- Constraint:
  ```sql
  alter table slot_occurrences add constraint slot_occurrences_exclusive_resource_no_overlap
    exclude using gist (resource_id with =, tstzrange(start_at, end_at, '[)') with &&)
    where (status = 'ACTIVE' and resource_is_exclusive);
  ```
  (`btree_gist` ya está instalado desde `phase7_payments.sql`). Con
  `'[)'`, dos turnos pegados (10:00–10:30 y 10:30–11:00) no chocan.
  `BLOCKED`/`CANCELLED` quedan afuera del constraint, así que cancelar
  libera el hueco.
- `generate_slot_occurrences_for_rule()` se recrea para envolver cada
  insert en `begin ... exception when exclusion_violation then ... end`
  y saltear esa ocurrencia puntual en vez de abortar toda la regeneración
  del tenant (sin esto, un conflicto en una organización frenaría el
  `pg_cron` diario para todas).
- Nueva función `check_schedule_rule_conflicts(...)` que proyecta la
  ventana de 90 días y devuelve los choques ANTES de crear/editar una
  regla — el constraint es el respaldo final, esta función da un mensaje
  útil (`RESOURCE_SCHEDULE_CONFLICT`) en vez de dejar que el usuario se
  entere por un error de Postgres.
- Activar el flag en un recurso que ya tiene solapamientos: el propio
  constraint rechaza el `update` (`RESOURCE_HAS_OVERLAPS`), hay que
  resolver los choques antes de marcarlo exclusivo.

**Alternativas descartadas** (evaluadas explícitamente por el diseño):
- **Inferir el conflicto sólo por solapamiento, sin campo nuevo**:
  descartada — rompería datos hoy válidos (ej. un gimnasio con una sala
  compartida entre dos clases a la misma hora pasaría a ser un error),
  cambiando comportamiento en el deploy para tenants existentes. Además
  deja el modelado ambiguo (la única forma de "compartir de verdad"
  sería crear recursos fantasma).
- **`concurrency_limit int` en vez de `boolean`**: descartada por ahora
  — con N>1 el constraint deja de ser un `EXCLUDE` declarativo y pasa a
  requerir un conteo con trigger y lock, más complejo y con riesgo de
  carreras, para un caso de uso ("cancha que admite 2 servicios
  simultáneos a la vez") que no tiene demanda real todavía. El boolean
  migra a int sin romper nada si aparece la necesidad.

**Impacto:** `backend-engineer` implementa la migración
`phase42_exclusive_resources.sql`. `frontend-engineer` agrega el
checkbox "Se ocupa de a uno" al formulario de recursos, y mapea
`RESOURCE_SCHEDULE_CONFLICT`/`RESOURCE_HAS_OVERLAPS`. No requiere gate de
`security-engineer` (no toca auth/RLS/fronteras de tenant, hereda la RLS
existente de `resources`) — sí pasa por `reviewer`, con foco en el manejo
de excepciones del generador y en que el trigger de propagación respete
`organization_id`. Es la base de ADR-0045 (generador por franja) y deja
lista la capacidad=1 implícita que ADR-0046 (cobrar turno) asume para
recursos exclusivos.

---

## ADR-0045 — Generador de horarios por franja horaria

Fecha: 2026-09-30
Estado: **Aceptada**
Propuesta por: usuario (auditoría de generalización), diseñada por
`Plan`.

**Problema:** cargar la agenda de un negocio de turno individual hoy
significa crear una `ScheduleRule` por cada horario de inicio, uno a la
vez — para una barbería con 2 profesionales, lunes a sábado de 10 a 20
con turnos de 30 minutos, son unos 40 envíos de formulario. El formulario
actual (`schedule-rule-form.tsx`) sólo acepta un único `localStartTime`
por regla.

**Decisión — extender el precedente ya existente de ADR-0022
(`create_schedule_rule_group()`, que itera `weekdays[]`) al producto
cartesiano `weekdays[] × local_start_times[]`**, sin cambiar el modelo de
`ScheduleRule` (sigue siendo una fila por día+hora, nunca un rango):

- Nueva RPC `create_schedule_rule_span(p_service_id, p_resource_id,
  p_weekdays int[], p_range_start time, p_range_end time, p_step_minutes
  int, p_duration_minutes int, p_capacity int) returns uuid` (el
  `group_id`), mismo chequeo de permiso que `create_schedule_rule_group`
  vigente.
- Expansión de `local_start_time` en SQL con `generate_series(range_start,
  range_end - duration, step)`, incluyendo sólo inicios donde `start +
  duration <= range_end`.
- Topes defensivos obligatorios: `step_minutes >= 5`, máximo 96 inicios
  por día, máximo 7×96 reglas por llamada — sin esto, un solo request
  podría generar decenas de miles de `SlotOccurrence` en la ventana de 90
  días (vector de DoS de almacenamiento).
- Si el recurso es exclusivo (ADR-0044) y `step_minutes < duration_minutes`
  (turnos que se pisarían entre sí dentro de la misma franja), se
  rechaza de entrada con `SPAN_SELF_OVERLAP_ON_EXCLUSIVE_RESOURCE`. Si es
  exclusivo y `p_capacity > 1`, se rechaza con
  `EXCLUSIVE_RESOURCE_CAPACITY_MUST_BE_ONE` (regla genérica: un recurso
  exclusivo atiende de a uno).
- Llama a `check_schedule_rule_conflicts()` (ADR-0044) una vez para todo
  el lote; la operación es atómica (todo o nada).

**Por qué no un `ScheduleRule` con rango propio** (alternativa
descartada, mismo motivo que ADR-0022 ya documentó en
`docs/decisions.md:1154-1161`): rompería `ScheduleException` (clave
regla+fecha puntual), la cascada de `discontinue_schedule_rule`, y
`RecurringBooking.schedule_rule_id` — los tres asumen una regla puntual,
no un rango expandible.

**Impacto:** `backend-engineer` implementa la migración
`phase43_schedule_rule_spans.sql` (idealmente refactorizando
`create_schedule_rule_group` para compartir la rutina interna de
expansión+inserción). `frontend-engineer` agrega un toggle "Un
horario / Franja" en `schedule-rule-form.tsx`, con vista previa de los N
horarios resultantes (sólo informativa, calculada en cliente) y
agrupación por `group_id` en el listado. No requiere gate de
`security-engineer` — mismo perímetro de permisos que
`create_schedule_rule_group`. `reviewer` debe confirmar los topes
defensivos explícitamente. Depende de que ADR-0044 esté mergeada primero
(sin el constraint, una franja sobre un recurso exclusivo puede generar
dobles turnos silenciosamente) aunque el código puede escribirse en
paralelo.

**Review (2026-09-30): LISTO**, con dos precisiones no bloqueantes que
`reviewer` encontró al refactorizar `create_schedule_rule_group()` para
compartir rutina con la RPC nueva: (1) el orden entre `INVALID_WEEKDAY` y
`RESOURCE_SCHEDULE_CONFLICT` cambió respecto a la versión anterior (antes
el conflicto de recurso se evaluaba primero; ahora la validación de
weekday va primero) — no rompe ningún invariante ni es alcanzable desde
la UI actual (que ya restringe weekday a 0-6), sólo afecta el código de
error exacto en una combinación de inputs inválidos simultáneos, sin
test que lo cubra; (2) el comentario sobre por qué se usa `array_agg()`
en vez de `select ... limit 1` sobreestimaba el riesgo — PL/pgSQL con
`RETURN NEXT` siempre materializa el resultado completo antes de volver
al llamador (a diferencia de funciones en C como `generate_series`), así
que un `LIMIT 1` no habría cortado la inserción a mitad de camino en este
caso puntual; `array_agg()` sigue siendo la forma más clara, sólo se
corrigió la justificación en el comentario. Verificado en vivo contra
`reservaste-stg` por el Orchestrator (385/385, suite completa).

---

## ADR-0046 — `book_slot_paying()`: cobrar un turno suelto, versión sólo-mostrador

Fecha: 2026-09-30
Estado: **Aceptada**
Propuesta por: usuario, diseñada por `Plan`, implementa el §2.8 de
`docs/proposals/adr-0025-makeup-credits.md` (documento de diseño nunca
implementado, escrito junto con ADR-0025 original).

**Problema:** el modelo `DROP_IN` (turno suelto, un pago por ocurrencia
puntual) ya existe completo en el schema desde ADR-0024/ADR-0022
(`service_plan_kind`, `payments.slot_occurrence_id`, el trigger
`PAYMENT_PLAN_KIND_REQUIRES_OCCURRENCE`, el índice anti-doble-cobro) —
pero nunca se construyó la pieza que lo cobra. `RegisterPaymentForm`
excluye `DROP_IN` a propósito ("eso se cobra desde el turno, no desde
acá"), y esa pantalla del turno nunca se construyó. Consecuencia real: un
servicio configurado con `DROP_IN` + `payment_required=true` queda con
reservas bloqueadas sin ninguna salida — no es un bug de lógica de
`evaluate_payment_coverage()` (que hace exactamente lo que tiene que
hacer: un `DROP_IN` nunca cubre por período), es una pieza de UI/RPC que
falta.

**Decisión — implementar `quote_booking()`/`book_slot_paying()` tal como
especifica §2.8 de la propuesta original, con dos precisiones nuevas**:

1. **Por ahora, `book_slot_paying()` lo invoca sólo el staff** (mismo
   permiso que `admin_book_for_customer`, ej. `MANAGE_BOOKINGS`) — sin
   una pasarela de pago real conectada (ADR-0027, postergada, sin fecha),
   dejar que el cliente se auto-marque como `PAID` sería autodeclararse
   pagado sin ninguna verificación. La firma queda lista para engancharse
   con `payment_intents` (§2.8.2) el día que ADR-0027 se implemente,
   pero eso es explícitamente fuera de este alcance. **Confirmado con el
   usuario**: cobro por mostrador (efectivo/tarjeta en el local,
   registrado en el sistema) alcanza para la primera venta — no se
   necesita pago online para vender.
2. **Cubre también "ya anotado, falta cobrar"**: si la `Booking` ya
   existe y la cobertura no está resuelta, sólo crea el `Payment`. Si ya
   está cubierta, `ALREADY_COVERED` sin cobrar de nuevo — cubre el caso
   de un negocio con `payment_required=false` que igual quiere dejar
   registro de un cobro hecho en efectivo después del servicio.

**Mecánica** (dos RPC nuevas, `phase44_drop_in_booking.sql`):
- `quote_booking(p_slot_occurrence_id, p_customer_id)` — `stable`, sin
  side-effects, devuelve `{can_book, reason, coverage_path, price,
  currency, makeup_credit_id?, makeup_credit_expires_on?}`. Envuelve
  `evaluate_payment_coverage()`/`evaluate_customer_booking()` ya
  existentes. Invocable por staff con permiso, o por el propio cliente
  consultando su propia cobertura (sólo lectura, sin riesgo).
- `book_slot_paying(p_slot_occurrence_id, p_customer_id, p_amount?)` —
  `security definer`, mismo orden de locks que `book_slot()`
  (`FOR UPDATE` sobre la ocurrencia primero, para no generar deadlocks
  con reservas concurrentes). Exige permiso de staff. Valida que
  `customer_id` pertenece a la misma organización que la ocurrencia
  (`CUSTOMER_NOT_IN_ORG` si no — es la frontera multi-tenant principal de
  esta pieza). Re-evalúa cobertura antes de cobrar. Resuelve el `DROP_IN`
  activo del servicio (`NO_DROP_IN_PLAN` si no hay uno); `amount =
  coalesce(p_amount, plan.price)`, nunca negativo. El índice
  `payments_one_paid_per_occurrence_idx` ya existente es la red contra
  doble cobro concurrente (`unique_violation` → `ALREADY_PAID`). Si no
  existe `Booking` todavía, la crea en la misma transacción (mismo camino
  interno que `admin_book_for_customer`) — o quedan `Payment PAID` +
  `Booking CONFIRMED` juntos, o no queda nada.
- `agenda_occurrences()`/el detalle de asistentes se extienden
  (patrón aditivo, `drop function` + `create function`) con
  `drop_in_plan_id, drop_in_price, drop_in_currency` por ocurrencia y
  `is_covered, paid_payment_id` por asistente.

**Impacto**: `backend-engineer` implementa la migración.
`frontend-engineer` agrega "Cobrar $X" (o badge "Pagado") junto a cada
asistente en `occurrence-actions.tsx`, y "Anotar y cobrar" en la sección
de anotar cliente; nueva server action en `app/actions/`. **Gate
obligatorio de `security-engineer`** — toca pagos y fronteras
multi-tenant: foco en (a) pertenencia de `customer_id`/ocurrencia a la
misma organización, (b) que el cliente nunca pueda fijar `amount` ni
plan y que la RPC no sea invocable sin el permiso de staff, (c) orden de
locks bajo concurrencia (doble cobro, sobre-reserva), (d) que
`quote_booking` no filtre precios/cobertura de otro cliente o otra
organización. Independiente de ADR-0044/0045, pero se recomienda
mergearla antes de ADR-0047 (ambas tocan `book_slot()`/el árbol de
reserva y un merge en paralelo garantiza conflicto).

**Corrección post-gate de seguridad (2026-09-30).** El gate encontró dos
hallazgos ALTO reales, ya corregidos y verificados en vivo contra
`reservaste-stg` (11/11 tests en verde, incluidos 3 tests de regresión
nuevos):

1. **Fuga cross-tenant en `resolve_active_drop_in_plan()`**: la función
   no filtraba por `organization_id` — un plan `DROP_IN` con
   `applies_to_all_services=true` de CUALQUIER organización matcheaba
   todos los servicios de la plataforma (verificado en vivo: el precio
   de un tenant ajeno aparecía en `agenda_occurrences`/`quote_booking` de
   otro, y `book_slot_paying` abortaba — un DoS del cobro para todos los
   tenants). Fix: `join services` + filtro explícito por
   `sp.organization_id = s.organization_id`.
2. **El permiso de esta ADR estaba mal** — no es `MANAGE_BOOKINGS` como
   decía el punto 1 más arriba, es **`MANAGE_PAYMENTS`** (el mismo que ya
   exige la policy `payments_insert_staff` de Fase 32 para insertar
   cualquier pago). Con sólo `MANAGE_BOOKINGS`, un rol configurado a
   propósito sin permiso de cobro (ADR-0033) podía registrar un
   `Payment PAID` con `p_amount=0`, esquivando el `PAYMENT_REQUIRED` que
   `admin_book_for_customer()` le devuelve al mismo rol — verificado en
   vivo. `book_slot_paying()` ahora exige `MANAGE_PAYMENTS` siempre (por
   escribir un `Payment`) y `MANAGE_BOOKINGS` adicionalmente sólo si hace
   falta crear la `Booking` (mismo criterio de Fase 34: un cajero con
   `MANAGE_PAYMENTS` puede cobrar un turno ya anotado, pero no anotar uno
   nuevo).
3. **MEDIO, también corregido**: `p_amount = 'NaN'::numeric` no es `< 0`
   en Postgres y `numeric(12,2)` lo acepta — sin chequeo explícito
   quedaba un `PAID` con `amount NaN`, envenenando cualquier `sum()` de
   reportes. Fix: `v_amount is null or v_amount = 'NaN' or v_amount < 0`
   → `INVALID_AMOUNT`.

**Dos hallazgos menores, decisión del Orchestrator, ninguno bloqueante**:
- **Pre-existente, no introducido por esta ADR**: `evaluate_payment_coverage()`
  (Fase 22) usa un `exists` sobre `service_plans` sin filtro de
  organización para decidir entre devolver `SERVICE_HAS_NO_PLAN` o
  `PAYMENT_REQUIRED` — un plan global de otra organización puede cambiar
  cuál de los dos `reason` ve un tenant ajeno. No filtra datos ni
  habilita ninguna reserva/cobro indebido, sólo el texto del motivo
  mostrado. **Aceptado como deuda técnica separada**, a resolver en una
  fase propia (no forma parte del alcance de ADR-0046) — anotado en
  `docs/security.md`.
- **Edge case de negocio, severidad baja, aceptado**: un cliente con un
  plan `UNLIMITED` vigente en un servicio **gratuito** (`payment_required
  = false`) puede igual recibir un cobro `DROP_IN` si el staff invoca
  `book_slot_paying` a propósito — el re-chequeo de cobertura está
  gateado por `payment_required`, no por "¿tiene algún plan vigente?".
  Es un doble cobro iniciado deliberadamente por el staff (no un bypass
  de seguridad ni algo que un cliente pueda gatillar), y requeriría que
  el negocio tenga simultáneamente un servicio gratuito Y un `DROP_IN`
  activo sobre ese mismo servicio — configuración rara. Se acepta el
  riesgo tal cual por ahora; si aparece en producción, se resuelve
  extendiendo el re-chequeo de cobertura a mirar planes de período
  incluso en servicios gratuitos.

`docs/database.md`/`docs/security.md` actualizados con el detalle
completo y las reglas de la Fase 44 corregida.

---

## ADR-0047 — Reserva abierta: alta de `Customer` en el momento de reservar, detrás de un flag

Fecha: 2026-09-30
Estado: **Aceptada**
Propuesta por: usuario, diseñada por `Plan`. **Priorizada explícitamente
para la primera semana de trabajo** (no la segunda, como recomendaba el
plan original) — decisión del usuario tras confirmar que es el
bloqueante comercial más visible (una auditoría previa ya lo había
marcado como el punto #1: hoy ningún cliente nuevo puede reservar sin
que el dueño lo dé de alta a mano primero, lo cual además contradice el
FAQ de la propia landing page, que dice "sólo necesitan una cuenta
simple").

**Problema:** `can_customer_book()` devuelve `NOT_A_CUSTOMER` para
cualquier cuenta autenticada que no sea ya `Customer` activo de esa
organización — no existe ningún camino de autoservicio, ni ningún flag
para relajarlo. Confirmado que esto es así por diseño desde ADR-0005/
ADR-0022, no un bug.

**Decisión — columna `organizations.open_booking_enabled boolean not
null default false`**, mismo patrón ya establecido 3 veces en este
proyecto (`makeup_credits_enabled`, `customer_activation_enabled`,
`public_availability_display`): flag opt-in por organización, default
que no cambia comportamiento existente, editable desde Configuración con
el mismo mecanismo de checkbox+hidden-input+`formData.has()` que ya usa
`settings-form.tsx`/`actions/settings.ts` — la policy
`organizations_update_owner` ya alcanza como control de acceso, sin RPC
nueva para el flag en sí.

**El alta on-the-fly vive DENTRO de `book_slot()`, nunca dentro de
`can_customer_book()`** — decisión de diseño explícita y no trivial: 
`can_customer_book()`/`evaluate_customer_booking()` se invocan también
desde `preview_recurring_booking()` y `can_customer_book_detail()` en
contextos de **preview de solo lectura**, sin intención real de reservar.
Si el alta viviera ahí, un preview inocente crearía `Customer`s reales
como side-effect no deseado. `book_slot()` ya toma `FOR UPDATE` sobre la
ocurrencia en el intento real de reserva — es el único lugar seguro.
Mecánica exacta: después del lock de la ocurrencia y antes de invocar
`can_customer_book()`, si no hay `Customer` activo para `(org,
auth.uid())`, el flag está prendido y pasa el rate limit (ver abajo), se
inserta `customers(organization_id, profile_id=auth.uid(),
display_name=profiles.full_name, is_active=true, source='SELF_SERVICE')`
dentro de la misma transacción que la reserva — si la reserva falla
después (sin cupo, sin cobertura), el rollback revierte también el alta,
así que nunca quedan `Customer`s huérfanos. Concurrencia (dos pestañas):
maneja `unique_violation` sobre `(organization_id, profile_id)` con
re-select. **Si existe un `Customer` inactivo para ese `profile_id`, NO
se reactiva** — `NOT_A_CUSTOMER` igual; si el staff lo dio de baja, la
reserva abierta no puede pasar por encima de esa decisión.

`can_customer_book()` cambia sólo su código de retorno en este caso: con
el flag prendido y sin `Customer`, devuelve un reason nuevo (ej.
`OK_OPEN_BOOKING`) en vez de `NOT_A_CUSTOMER`, para que la UI pública
muestre "Reservar" en vez del mensaje de "escribile al negocio". La
cobertura de pago se evalúa como si fuera un `Customer` nuevo, sin
planes ni créditos previos — si el servicio exige pago, el resultado es
`PAYMENT_REQUIRED` normalmente.

**Regla de seguridad citada explícitamente, por ser la más relevante de
esta zona** (`docs/security.md`, regla ya establecida en el gate de
ADR-0043): *"El email de `auth.users` es un factor de conocimiento,
nunca de posesión."* El alta on-the-fly se basa exclusivamente en
`auth.uid()` de la sesión ya autenticada (mismo patrón IDOR-proof de
ADR-0005) — **nunca** busca ni vincula un `Customer` gestionado
preexistente por email (ADR-0026). Si ya existe un `Customer` gestionado
con el mismo email sin `profile_id`, se crea uno nuevo (duplicado a
fusionar por el staff más adelante) en vez de vincular por email — un
duplicado es preferible a un vínculo no verificado.

**Riesgo heredado de ADR-0043, nombrado explícitamente y mitigado**:
desde que el signup no tiene confirmación de email, "reservar" con este
flag prendido baja a "pasar el captcha de Turnstile". Mitigaciones
obligatorias en esta misma fase:
1. Rate limit dentro de la RPC, mismo patrón que
   `issue_customer_activation()` (`phase21:335-353`): tope de altas
   on-the-fly por `auth.uid()` en 24h, y tope por organización por hora.
2. El `Customer` creado lleva `source='SELF_SERVICE'` para que el staff
   pueda filtrarlo/limpiarlo.
3. El flag es opt-in — la organización acepta el riesgo residual al
   prenderlo, con el texto de advertencia correspondiente en Settings.

**Impacto**: `backend-engineer` implementa
`phase45_open_booking.sql` (recrea `book_slot()` desde la versión
vigente de `phase20`). `frontend-engineer` agrega el checkbox en
Settings y ajusta el flujo público de reserva para tratar
`OK_OPEN_BOOKING` como reservable. **Gate obligatorio de
`security-engineer`, el más importante de todo este lote** — foco en:
(a) que `organization_id` salga siempre de la ocurrencia, nunca de un
input del caller; (b) el rate limit; (c) la no-reactivación de cuentas
inactivas; (d) la ausencia total de vínculo por email; (e) un test
explícito de que los caminos de preview NO crean `Customer`s con el flag
prendido; (f) si hace falta un tope de reservas futuras activas por
`Customer` `SELF_SERVICE` para acotar el abuso de cupo con cuentas
descartables. Se recomienda mergear después de ADR-0046 (ambas tocan
`book_slot()`) para evitar conflicto de merge garantizado si van en
paralelo.

**Corrección post-gate de seguridad (2026-09-30).** El gate confirmó que
la reestructuración de `book_slot()` es correcta (verificada en vivo
contra `reservaste-stg`, comparando 19 escenarios entre la versión vieja
y la nueva, resultado idéntico salvo un caso no alcanzable desde la API),
que el alta on-the-fly nunca se dispara desde ningún camino de sólo
lectura, y que no reactiva cuentas inactivas. **Veredicto: LISTO para
mergear y desplegar con el flag apagado (default `false`, cero cambio de
comportamiento para el gimnasio actual) — NO LISTO para activar el flag
en ningún negocio real todavía.** Dos hallazgos ALTO, confirmados en
vivo, bloquean activarlo:

1. **Sin tope de reservas para un `Customer` `SELF_SERVICE`.** Nada
   impide que una cuenta descartable (sólo necesita pasar el captcha)
   reserve todas las ocurrencias futuras de un servicio sin pago, e
   incluso arme una reserva fija recurrente (`create_recurring_booking`)
   que genera reservas para todas las ocurrencias futuras de una regla —
   vaciando la agenda con una sola cuenta. **Fix decidido**: tope de 2
   reservas confirmadas futuras mientras `customers.source =
   'SELF_SERVICE'` (`FOR UPDATE` sobre la fila del cliente para que no se
   pueda saltear en paralelo), y vedar `create_recurring_booking()` a
   clientes `SELF_SERVICE`. El staff "verifica" a un cliente real
   pasando `source` a `STAFF` (la policy `customers_update_staff` ya lo
   permite, sin RPC nueva) — desde ahí el tope deja de aplicarle.
2. **Las altas `SELF_SERVICE` consumen `max_customers` del plan del
   tenant, y al agotarse filtran el conteo de clientes en un error
   crudo** (`PLAN_LIMIT_REACHED: clientes (50/50)`) a cualquier usuario
   autenticado — reproducido en vivo. Con el tope original de 50
   altas/organización/hora, una hora agota el plan starter completo
   (`max_customers=50`) y bloquea al staff de dar de alta clientes reales
   hasta limpiar las cuentas basura a mano. **Fix decidido**: atrapar
   `PLAN_LIMIT_REACHED`/`SUBSCRIPTION_INACTIVE` en el insert de
   `book_slot()` y devolver un status propio sin filtrar el detalle
   crudo; bajar el tope por organización de 50/hora a **10/hora y
   30/día**; bajar el tope por `auth.uid()` de 5/24h a algo efectivo — el
   gate señaló que 5 "no frena a nadie" porque crear cuentas sólo cuesta
   el captcha, así que el tope real de contención pasa a ser el de
   reservas por `Customer` (punto 1), no el de altas de cuenta.

Un hallazgo MEDIO también se corrige en el mismo pase: los rate limits no
estaban serializados (7 altas concurrentes pasaron contra un tope de 5,
verificado en vivo) — fix: `pg_advisory_xact_lock` por `auth.uid()` y por
organización antes de contar. Un segundo MEDIO (`can_customer_book`
devuelve `OK_OPEN_BOOKING` sin validar ocurrencia cancelada/pasada o
servicio inactivo) **se acepta sin fix**: no es explotable, porque
`book_slot()` vuelve a validar todo desde cero antes de escribir nada —
es un detalle de UI (la pantalla pública podría mostrar "Reservar" en un
caso que después rebota), no de seguridad.

**Implementación de los fixes**: delegada de nuevo a `backend-engineer`,
mismo archivo (`phase45_open_booking.sql`, sin migración nueva —
se edita antes de mergear, todavía no está en `main`). Requiere un
segundo pase corto de `security-engineer` confirmando específicamente
los dos ALTO antes de dar LISTO definitivo para activar el flag.

---

## ADR-0048 — Disponibilidad pública por recurso (opt-in)

Fecha: 2026-09-30
Estado: **Aceptada**
Propuesta por: usuario, diseñada por `Plan`.

**Problema:** `get_public_availability()` nunca devolvió `resource_id`
ni el nombre del recurso — un cliente que reserva no puede elegir "con
quién" (ej. qué barbero). El RPC equivalente de staff
(`agenda_occurrences()`) sí lo expone; sólo falta en el camino público.

**Decisión**: agregar al final del `returns table` (patrón aditivo ya
usado 3 veces: fase 5, 16, 20) `resource_id uuid, resource_name text`.
`resource_id` siempre se devuelve (es opaco, sirve para agrupar/filtrar
sin revelar nada). `resource_name` **sólo se devuelve si
`organizations.public_resource_names boolean not null default false`
está prendido**; si no, `null`. Es una regla de disclosure nueva,
análoga en espíritu a ADR-0008 pero sobre identidad en vez de cupo: si el
recurso es una persona, su nombre es un dato que hoy nunca salió por
`anon`, y no debería empezar a salir por default silenciosamente.

**Impacto**: `backend-engineer` implementa
`phase46_public_availability_resource.sql` (`drop function` + `create
function`, grants re-emitidos a `anon, authenticated`).
`frontend-engineer` mapea las columnas nuevas a mano en
`app/actions/public.ts` (el paquete `@reservaste/domain` está fijado a
un commit SHA, no a HEAD — mismo patrón ya usado 2 veces cuando esto
pasa) y agrega un selector "Con quién" en la reserva pública cuando haya
más de un recurso y el flag esté prendido, con "Cualquiera" como
comportamiento por default. **Gate liviano de `security-engineer`** (es
una RPC `anon`): confirmar que no se filtra nada más de `resources`
(ej. `description`) y que el flag se lee de la organización correcta.
Tiene más sentido después de ADR-0044 (recursos exclusivos) pero es
técnicamente independiente — no bloqueante para la primera venta (con un
servicio por profesional, ej. "Corte con Juan", se puede operar sin
esto).

---

## Plan de generalización a turno individual — orden y alcance mínimo

Fecha: 2026-09-30

Registro de coordinación para ADR-0044 a ADR-0048 (más dos ajustes
menores sin ADR propia, ver abajo) — no es una ADR nueva, es el mapa de
secuenciación acordado con el usuario tras la auditoría de qué tan
genérico es el sistema.

**Orden acordado** (reordenado tras la decisión explícita del usuario de
priorizar ADR-0047 — reserva abierta — en la primera semana, no la
segunda como recomendaba el plan original, aceptando que el gate de
seguridad puede estirar el tiempo):

1. ADR-0044 (recursos exclusivos) — base de todo lo demás.
2. ADR-0045 (generador por franja) — depende de 0044 mergeada.
3. ADR-0046 (cobrar turno, sólo mostrador) — independiente, en paralelo
   con 1-2.
4. ADR-0047 (reserva abierta) — después de 0046 (ambas tocan
   `book_slot()`).
5. ADR-0048 (con quién) — no bloqueante, puede ir después o en paralelo
   si sobra tiempo.

**Dos ajustes menores, sin migración ni ADR propia, sólo `reviewer`**:
- Ocultar la pestaña "Créditos" del portal del cliente
  (`frontend/components/me-nav.tsx`) cuando `makeup_credits_enabled` esté
  apagado para todas las organizaciones del usuario — el flag ya existe
  en el backend, sólo falta que el frontend lo respete visualmente.
  Revisar también el copy de "clase"/"liberar cupo" en `app/me/*`.
- ~20 strings de copy con "clase"/"CrossFit"/"Iron Gym"/"socio" en
  placeholders y empty states pasan a términos neutros.

**Confirmado con el usuario**: cobro sólo por mostrador (sin pasarela de
pago online) alcanza para la primera venta — ADR-0027 (integrar una
pasarela real) sigue fuera de alcance y no se menciona a la barbería como
disponible.

**Alcance mínimo honesto para la primera semana**: recurso exclusivo por
profesional, agenda armada por franja horaria en vez de 40 formularios,
cobro registrado por mostrador, y reserva abierta para que un cliente
nuevo pueda reservar por link sin alta manual previa — con el gate de
seguridad de ADR-0047 corrido con el mismo rigor de siempre, sin
recortar pasos aunque apriete el tiempo.

---

## ADR-0049 — `Organization.industry`: rubro declarado, puramente informativo

Fecha: 2026-09-30
Estado: **Propuesta** (documentada para revisión, no implementada —
sesión en fase de revisión de plan, sin luz verde de implementación
todavía)
Propuesta por: usuario ("quiero entender el rubro de mi cliente, eso
debe estar documentado").

**Problema:** el usuario, como dueño del SaaS, quiere poder identificar
el rubro de cada `Organization` que usa la plataforma (gimnasio,
barbería, consultorio, cancha, etc.) — para su propio entendimiento de
negocio, analytics, y poder filtrar/segmentar sus clientes. Hoy no existe
ningún campo así.

**Tensión con la regla no-negociable de `CLAUDE.md`** ("nunca introducir
Gym/Member/Trainer/Class como modelo central"): esa regla prohíbe que el
rubro determine COMPORTAMIENTO del sistema (ninguna rama de código del
tipo "si es gimnasio, hacé X"), pero no prohíbe un campo puramente
descriptivo que nunca participa de ninguna decisión de lógica de negocio
— exactamente como `Organization.name` no es "modelo central" aunque
contenga texto libre elegido por el negocio.

**Decisión: `organizations.industry text` nullable, sin ningún CHECK que
lo restrinja a una lista cerrada** (texto libre, con una lista de
sugerencias en el frontend tipo autocomplete/datalist, no un enum de
base de datos) — para que un rubro nuevo (ej. "estudio de tatuajes") no
requiera una migración. Comentario explícito en la columna, citando
`CLAUDE.md`, dejando por escrito que **ninguna función, policy, ni
componente del frontend puede leer esta columna para cambiar
comportamiento** — sólo se lee para mostrarla (en el panel de admin del
propio negocio, y en cualquier vista interna/analítica que el usuario
quiera armar a futuro, ej. un dashboard de "mis clientes por rubro").

**Por qué texto libre y no enum**: un enum fijo (`GYM | BARBERSHOP |
CLINIC | ...`) reintroduce exactamente la taxonomía de rubros que el
proyecto evita a propósito — aunque sea sólo para mostrar, un enum que
crece requiere tocar el schema cada vez que aparece un rubro nuevo, y
empuja a alguien, en el futuro, a la tentación de hacer `if industry ===
'GYM'`. Texto libre sin CHECK no tiene ese imán.

**Impacto (cuando se implemente)**: `database-agent`/`backend-engineer`
agrega la columna (migración chica, sin RPC nueva — un `select`/`update`
directo alcanza, mismo criterio que otros campos simples de
`Organization`). `frontend-engineer` agrega el campo al onboarding
(opcional, no bloqueante) y a Configuración. No requiere gate de
`security-engineer` (dato no sensible, mismo nivel que el nombre del
negocio). Sin dependencias con ADR-0044 a ADR-0048.

**Pendiente**: el usuario todavía no dio luz verde para implementar esto
— queda documentado como próximo ítem del backlog de generalización,
a la espera de que se confirme cuándo entra en la cola de trabajo.

---

## ADR-0050 — `schedule_rule_groups()` expone múltiples horarios por grupo

Fecha: 2026-09-30
Estado: **Aceptada** (2026-09-30, Orchestrator) — cambio de contrato
necesario, no había forma aditiva de corregir el bug sin dejar el shape
viejo silenciosamente incorrecto para franjas; pendiente coordinar el
deploy con el rework de `frontend-engineer` en la misma ventana.
Propuesta por: `frontend-engineer` (bug real encontrado implementando la
UI de ADR-0045), fix diseñado e implementado por `backend-engineer`.

**Problema:** `schedule_rule_groups()` (Phase 16, ADR-0022) agrupa
`schedule_rules` por `group_id` y colapsa las columnas no-agrupadas con
`min(sr.local_start_time)`, `min(sr.duration_minutes)`, `min(sr.capacity)`
— válido en su momento porque `create_schedule_rule_group()` sólo podía
producir grupos donde cada fila tiene un `weekday` distinto y **todas**
comparten el mismo `local_start_time`. `create_schedule_rule_span()`
(Phase 43, ADR-0045) rompe esa invariante a propósito: un mismo
`group_id` puede tener varias filas con el **mismo** `weekday` y
`local_start_time` **distinto** (ej. una franja 09:00–11:00 cada 30 min,
duración 30, en un solo día, genera 4 `ScheduleRule` 09:00/09:30/10:00/
10:30 bajo un único grupo). `min(local_start_time)` colapsaba las 4 a
"09:00", y el frontend (`frontend/app/org/[slug]/services/[serviceId]/
schedule/page.tsx`) asumía además que `group.weekdays[i]` correspondía
1:1 con `group.ruleIds[i]` — válido sólo cuando cada fila tenía un
`weekday` distinto, no cuando varias filas comparten `weekday` y sólo se
distinguen por hora. Resultado real: las 4 filas de `StandingReservations`
de esa franja se etiquetaban todas "09:00", perdiendo toda distinción
entre ellas.

**Decisión:** `schedule_rule_groups(p_service_id)` deja de devolver
`local_start_time`, `weekdays[]` y `rule_ids[]` colapsados/zippeados, y
devuelve en su lugar un array `items jsonb` con un elemento por fila real
de `schedule_rules`: `{ ruleId, weekday, localStartTime }`, ordenado por
`(weekday, localStartTime, ruleId)`. `duration_minutes`, `capacity`,
`resource_id` y `resource_name` **siguen colapsados** con
`min()`/primer valor — esos sí son constantes garantizadas dentro de un
mismo `group_id` en todo camino de inserción existente:
`create_schedule_rules_batch()` (rutina interna compartida desde Phase
43) recibe un único `p_duration_minutes`/`p_capacity`/`p_resource_id` por
llamada y estampa todas las filas del lote bajo un `group_id` recién
generado; el insert manual de una sola regla
(`frontend/app/actions/schedule.ts::createScheduleRule`) nunca fija
`group_id` explícito, así que siempre recibe un grupo propio de una sola
fila vía el default de columna. Ningún camino agrega una fila a un
`group_id` **existente** con un recurso/duración/capacidad distintos —
sólo `weekday` y `local_start_time` varían dentro de un grupo, que es
exactamente lo que `items` expresa por fila en vez de colapsar.

**Es un cambio de contrato, no aditivo**: se quitan tres columnas del
shape de retorno de la RPC (`local_start_time`, `weekdays`, `rule_ids`) y
se agrega `items`; el único consumidor hoy
(`frontend/app/actions/schedule.ts::listScheduleRuleGroups()` →
`frontend/app/org/[slug]/services/[serviceId]/schedule/page.tsx`) necesita
reescribirse para leer `items` en vez de zippear los dos arrays viejos —
no había forma de mantener el shape viejo utilizable de forma aditiva
sin dejarlo silenciosamente incorrecto para el caso de franja.

**Migración:** `backend/supabase/migrations/
20260930160000_phase46_schedule_rule_groups_multi_time.sql` — `drop
function` + `create function` (cambia el tipo de retorno, no alcanza con
`create or replace`), mismo permiso (`grant execute ... to authenticated`)
que la versión anterior. Sin cambios de RLS ni de permisos — hereda el
mismo perímetro (`is_organization_member`) que ya tenía.

**Tests:** `backend/test/phase43.schedule-rule-spans.test.ts` — caso
nuevo (h) crea una franja real de un día/4 horas con
`create_schedule_rule_span()` y confirma que `schedule_rule_groups()`
devuelve las 4 combinaciones reales (`items` con 4 `ruleId` distintos,
cada uno con su propio `localStartTime`, verificado contra
`schedule_rules` directamente) en vez de una colapsada; caso nuevo (i)
repite el caso original de ADR-0022 (un grupo de weekdays distintos,
mismo horario) y confirma que sigue devolviendo el mismo resultado
correcto con el shape nuevo (no regresión). `backend/test/
phase16.groups-attendance-payments.test.ts` se actualizó para leer
`items` en el único assert que dependía del shape viejo
(`weekdays`/`rule_ids`) — no se agregó lógica nueva ahí, sólo se adaptó
al contrato nuevo.

**Impacto:** `frontend-engineer` reescribe
`ScheduleRuleGroup`/`listScheduleRuleGroups()` en
`frontend/app/actions/schedule.ts` para consumir `items`, y
`schedule/page.tsx` para derivar de `items` tanto los badges de día
(weekdays únicos presentes) como la etiqueta real de cada fila de
`StandingReservations` (ya no puede asumir que hay una sola hora por
grupo). No requiere gate de `security-engineer` (mismo perímetro de
lectura ya existente, ninguna fuga cross-tenant nueva). Pendiente:
Orchestrator confirma esta ADR como Aceptada y coordina con
`frontend-engineer` el rework del consumidor antes de desplegar contra
`reservaste-stg` (la migración en sí es segura de aplicar sola: sólo
afecta lectura, pero el frontend actual dejaría de compilar/funcionar
contra el shape viejo hasta que se actualice).

---

## ADR-0051 — Disponibilidad dinámica para recursos exclusivos (opt-in por `Resource`)

Fecha: 2026-10-02
Estado: **Aceptada**
Propuesta por: usuario (caso real: una barbería con un recurso exclusivo
que ofrece varios servicios de distinta duración), diseñada por `Plan`
en dos vueltas (diseño base + mecanismo de hold agregado a pedido
explícito del usuario), decisiones estructurales confirmadas por el
usuario en el chat antes de registrar esta ADR.

**Problema:** `ScheduleRule` ata siempre un `Service` a una duración fija
y genera una grilla de `SlotOccurrence` pre-generada (ADR-0009/ADR-0045).
Un `Resource` exclusivo (ADR-0044) que ofrece varios servicios de
distinta duración (ej. un barbero con "Corte" de 30 min y "Combo" de 60
min) arma una grilla independiente por servicio — cuando el generador
choca con una ocurrencia ya existente de otro servicio en el mismo
recurso, la restricción `EXCLUDE` de ADR-0044 rechaza el insert
(`exclusion_violation`) y `generate_slot_occurrences_for_rule()` salta
esa fecha **en silencio**. No hay riesgo de doble-reserva (el `EXCLUDE`
ya lo impide), pero el servicio de mayor duración termina con huecos de
disponibilidad inexplicables, sin aviso al dueño ni al cliente.

**Decisión:** segundo flag opt-in a nivel `Resource`,
**`dynamicAvailability: boolean`** (default `false`, sólo válido si
`isExclusive = true` — `CHECK (not dynamic_availability or is_exclusive)`
a nivel de base, no una convención de aplicación). Con el flag apagado
(todo dato existente, hoy, sin excepción), cero cambio de comportamiento
en ningún camino — el modelo de gimnasio (`isExclusive = false`, cupo
compartido, `ScheduleRule` de duración fija) queda exactamente igual. Con
el flag prendido en un recurso exclusivo, ese recurso deja de usar la
grilla pre-generada: la disponibilidad se calcula al momento de reservar,
contra los huecos reales del recurso (de cualquier servicio que atienda).

### Modelo nuevo

- **`resource_availability_windows`** (nueva, no reusa `ScheduleRule`):
  `id, organization_id, resource_id, weekday, local_start_time,
  local_end_time, valid_from, valid_until, is_active, created_at/by,
  cancelled_at/by`. Es la "apertura general" de un recurso dinámico (ej.
  "atiende lunes a viernes 9 a 17"), desacoplada de cualquier `Service` —
  a diferencia de `ScheduleRule`, una sola fila cubre un rango horario
  completo, no un punto de inicio. Elegida en vez de reusar
  `schedule_rules.service_id` como nullable porque: (a) es puramente
  aditiva, cero auditoría de los muchos consumidores existentes que
  asumen `service_id not null`; (b) un recurso dinámico no tiene
  **ninguna** fila en `schedule_rules`, así que `RecurringBooking`
  (atada a `schedule_rule_id`) no puede referenciarlo por construcción —
  la incompatibilidad con reservas fijas semanales se resuelve sola, sin
  validación nueva.
- **`resource_availability_exceptions`** (Fase 2, no bloqueante para el
  mínimo viable): feriados/bloqueos puntuales, keyed a
  `(resource_availability_window_id, exception_date)` — mismo rol que
  `ScheduleException` pero por recurso, no por regla.
- **`get_dynamic_availability(p_organization_slug, p_service_id, p_date,
  p_resource_id default null)`**: `stable`, `security definer`, grant a
  `anon, authenticated` (mismo perímetro que `get_public_availability()`,
  mismas reglas de disclosure de ADR-0008/ADR-0048). Calcula los
  horarios de inicio donde `[start, start+service.duration)` entra en
  alguna ventana abierta, sin excepción que la bloquee, y sin solapar
  ninguna `SlotOccurrence` `ACTIVE` **o `HELD` no vencida** del recurso —
  de cualquier servicio.
- **`SlotOccurrence` no cambia de rol**: sigue siendo la ocurrencia
  reservable real, con el mismo `Booking`, `attendance_status`,
  `agenda_occurrences()`, `book_slot_paying()` (ADR-0046). Lo único que
  cambia es *cuándo* se inserta la fila: el cron de 90 días (modelo de
  hoy) vs. una RPC de reserva en el instante (sólo para recursos con el
  flag).

### Mecanismo de hold (decisión explícita del usuario, no el default
### recomendado por `Plan` — ver razonamiento)

El usuario pidió explícitamente que, mientras alguien completa la
reserva, ese horario deje de verse disponible para otra persona — no
sólo que el doble-booking real sea imposible (eso ya lo garantiza el
`EXCLUDE` de ADR-0044 con o sin hold).

- **`slot_occurrence_status` gana el valor `'HELD'`** (hoy `ACTIVE |
  BLOCKED | CANCELLED`), con columnas nuevas `held_until timestamptz`,
  `held_by uuid references profiles(id)`. Es la misma fila/tabla, no una
  tabla de holds aparte, para que el `EXCLUDE` de ADR-0044 proteja
  también al hold con el mismo mecanismo (una tabla separada necesitaría
  su propio `EXCLUDE` duplicado).
- **El `EXCLUDE` de `slot_occurrences_exclusive_resource_no_overlap` se
  amplía** de `where (status = 'ACTIVE' and resource_is_exclusive)` a
  `where (status in ('ACTIVE', 'HELD') and resource_is_exclusive)` — no
  rompe nada existente: ningún recurso no-dinámico escribe jamás
  `status='HELD'`.
- **TTL: 5 minutos.** Vencimiento resuelto en dos lugares distintos (el
  predicado de un `EXCLUDE`/índice parcial no puede evaluar `now()`, así
  que el constraint no puede saber solo si un `HELD` venció):
  - **Lectura** (`get_dynamic_availability()`): un `HELD` vencido no
    cuenta como ocupado (`status='ACTIVE' or (status='HELD' and
    held_until > now())`) — sin side-effects, sigue `stable`.
  - **Escritura** (`hold_dynamic_slot()`, antes de su propio insert):
    limpia (`CANCELLED`) los `HELD` vencidos de **ese recurso** —
    acotado, indexado, sin job nuevo. Mismo patrón de limpieza perezosa
    ya usado por `customer_activations` (ADR-0026): sin cron de
    limpieza, cada escritor resuelve lo vencido antes de escribir lo
    propio. El cron diario existente (ADR-0009) suma una sola sentencia
    extra de housekeeping (holds de más de un día) — no es un job
    nuevo, es higiene de almacenamiento, no de corrección.
- **`hold_dynamic_slot(p_resource_id, p_service_id, p_start_at)`**:
  valida ventana/excepción, limpia holds vencidos del recurso, inserta
  `status='HELD'`. **Exige `auth.uid()` no nulo** — desviación deliberada
  del patrón público actual (`/reservar/confirmar` resuelve login después
  de elegir, porque elegir hoy es sólo lectura sobre una fila que ya
  existe). Holdear es un `insert` real sobre un recurso escaso,
  invocable directo por PostgREST con la `anon key` sin pasar por
  ningún formulario — sin `auth.uid()` no hay con qué limitar cuántos
  horarios puede trabar la misma persona. Rate limit: máximo **3 holds
  vivos simultáneos por perfil** (mismo mecanismo
  `pg_advisory_xact_lock` + `count(*)` que ya usa `book_slot()` para
  altas `SELF_SERVICE`, ADR-0047), rechazo `TOO_MANY_ACTIVE_HOLDS`.
  **No requiere ser `Customer` activo de la organización** — sólo una
  cuenta real de la plataforma (`auth.uid()`), igual que la reserva
  abierta de ADR-0047 da de alta el `Customer` on-the-fly recién al
  confirmar.
- **`book_dynamic_slot(p_slot_occurrence_id, p_use_makeup_credit default
  true)`**: wrapper delgado, no reimplementa nada. Toma `for update`
  sobre la fila (re-entrante dentro de la misma transacción, ADR-0004),
  rechaza con `HOLD_NOT_FOUND`/`HOLD_EXPIRED` si no está `HELD` y
  vigente, promueve a `ACTIVE`, y delega el 100% de cobertura/pago/alta
  de `Customer`/creación de `Booking` a `book_slot()` ya existente — sin
  duplicar una sola línea de esa lógica ya revisada. Si `book_slot()` no
  devuelve `OK`, revierte explícitamente la promoción a `CANCELLED` (sin
  este paso quedaría una `SlotOccurrence` `ACTIVE` fantasma, sin
  `Booking`, ocupando el horario para siempre).
- **`release_dynamic_hold(p_slot_occurrence_id)`** (opcional, Fase 1 si
  el tiempo alcanza): cancela el propio hold antes de que venza, para
  cuando el cliente cambia de horario sin esperar los 5 minutos.

### Decisiones estructurales confirmadas (no delegadas a ningún subagente)

1. **Entidad nueva `resource_availability_windows`**, no reusar
   `ScheduleRule` con `service_id` nullable — confirmado.
2. **Hold en la primera fase**, no diferido — confirmado explícitamente
   por el usuario, en contra de la recomendación inicial de `Plan` de
   dejarlo para después; el usuario entendió y aceptó el costo (login
   requerido para holdear, antes de lo que el flujo público de hoy
   pide) a cambio de que un horario no se vea disponible mientras otra
   persona lo está completando.
3. **Un `Service` que requiera más de un `Resource` simultáneo queda
   bloqueado** (error explícito) cuando alguno de ellos es dinámico — el
   caso real del usuario es "cada profesional es un recurso exclusivo
   independiente, con su propia agenda y sus propios servicios" (ej. 2
   barberos = 2 recursos, nunca un servicio que cruce ambos a la vez),
   confirmado explícitamente. Revisitar sólo si aparece demanda real de
   un servicio que combine dos recursos a la vez.

### Riesgos identificados

- **"Hold squatting"**: mitigado por el tope de 3 holds vivos/perfil +
  requerir `auth.uid()` — sin esto sería el riesgo más serio del
  mecanismo.
- **RLS existente** (`slot_occurrences_toggle_active_blocked`, Fase 3):
  su `with check` sólo excluye `status <> 'CANCELLED'`, así que un STAFF
  con escritura directa podría tocar una fila `HELD` por ese camino en
  vez de por las RPCs nuevas — riesgo bajo (requiere ser staff de la
  propia organización), a confirmar explícitamente en el gate de
  seguridad obligatorio de esta fase.
- **Límite técnico del `EXCLUDE`**: su predicado tiene que ser una
  expresión inmutable, no puede evaluar `held_until > now()` — por eso
  la limpieza de vencidos es un `UPDATE` físico en el camino de
  escritura (`hold_dynamic_slot()`), nunca "algo que el constraint
  resuelve solo". Documentar esto en la migración para que nadie intente
  "simplificarlo" moviendo la condición de vencimiento al `where` del
  constraint.
- **Gate de seguridad obligatorio**: `hold_dynamic_slot()`/
  `book_dynamic_slot()` tocan el mismo perímetro crítico que
  `book_slot()` (multi-tenant, cobertura, pago, alta de `Customer`) —
  mismo rigor que tuvo ADR-0046/ADR-0047, con foco extra en el rate
  limit de holds y en que la revalidación de ventana/excepción ocurra
  dentro de la misma transacción que el insert.

### Fases

**Fase 1 (mínimo viable, incluye el hold por decisión del usuario):**
`resources.dynamic_availability` + `CHECK`; `resource_availability_windows`
+ RPC de alta simple (una fila por día, sin constructor de franjas
sofisticado); filtro de una línea en el generador para saltear recursos
dinámicos; `slot_occurrence_status` gana `HELD` + columnas +
`EXCLUDE` ampliado; `hold_dynamic_slot()`/`book_dynamic_slot()`/
`release_dynamic_hold()` (opcional); `get_dynamic_availability()` a
granularidad de un día, sin excepciones todavía; bloqueo explícito de
`RecurringBooking`/`create_schedule_rule_*` sobre un recurso dinámico
(mensaje de error claro, aunque ya cae solo por construcción); acción
explícita de discontinuar `ScheduleRule` viejas al prender el flag sobre
un recurso que ya tenía grilla (reusa la cascada existente de ADR-0010,
`discontinue_schedule_rule`, que cancela `Booking`/`RecurringBooking`
afectadas); contrato de API para que `frontend-engineer` construya el
flujo "elegí servicio → elegí día → elegí horario de la lista →
(login si falta) → confirmá".

**Fase 2 (fast follow, no bloqueante):**
`resource_availability_exceptions` (feriados/bloqueos puntuales); mejora
de `agenda_occurrences()` para mostrar ventanas abiertas sin reservas;
constructor de franjas para `resource_availability_windows` (reusar la
UX de ADR-0045); extraer una rutina interna compartida entre
`book_slot()`/`book_dynamic_slot()` si la duplicación resulta real en la
práctica (mismo criterio ya aplicado en ADR-0045 para
`create_schedule_rules_batch`); soporte N:M si aparece demanda real;
`pg_advisory_xact_lock` adicional para mejorar UX bajo alta contención
(la corrección ya está garantizada sin esto).

**Impacto:** `backend-engineer` implementa Fase 1 completa (migración +
tests). Gate de `security-engineer` obligatorio antes de cerrar (toca el
mismo perímetro crítico que ADR-0046/ADR-0047). `frontend-engineer`
construye el flujo nuevo recién después de que el backend esté verificado
en vivo contra `reservaste-stg` y gateado.

### Addendum (2026-10-02) — Fase 1 implementada, gate de seguridad cerrado en dos vueltas

Implementación completa en `backend/supabase/migrations/20261002100000_phase50a_...sql`
/ `20261002100001_phase50b_...sql` / `20261002110000_phase50c_...sql` —
detalle técnico completo en `docs/database.md`, sección "Fase 50".
**Corrección sobre el texto original de este ADR, arriba**: el primer
pase de `book_dynamic_slot()` devolvía `HOLD_NOT_FOUND`/`HOLD_EXPIRED`
como dos resultados distintos; el gate de seguridad (ver
`docs/security.md`, sección "Fase 50") encontró que esa distinción
funcionaba como oráculo y que la función no validaba `held_by`,
permitiendo robar/sabotear el hold de otra persona si se conseguía el
`slot_occurrence_id` (filtrable por URL, igual que `/reservar/confirmar?slot=`).
**El contrato real, ya implementado, unifica todo en un único
`HOLD_NOT_FOUND`** (no existe, no es tuyo, venció, o es una ocurrencia de
grilla) — `frontend-engineer` tiene que construir el flujo contra este
contrato, no el que describe el cuerpo original del ADR arriba.

El gate de `security-engineer` encontró, en su primer pase, dos
hallazgos bloqueantes reales (no en el texto original de este ADR, que
sólo anticipaba el rate limit y el límite técnico del `EXCLUDE`):

1. **Cancelar una reserva dinámica nunca liberaba el horario** — rompía
   el invariante no negociable de CLAUDE.md "cancelar libera el cupo
   inmediatamente", permitiendo vaciar la agenda de un recurso dinámico
   de forma permanente con una sola cuenta (hold → confirmar → cancelar,
   repetido). Cerrado con un trigger nuevo sobre `bookings`.
2. **`book_dynamic_slot()` no validaba `held_by`** (el robo/sabotaje
   descrito arriba). Cerrado agregando esa validación, con la
   unificación de contrato ya descripta.

Más un hallazgo MEDIO (un STAFF podía escribir `HELD` directo vía
PostgREST, afectando el tope global de holds de clientes de OTRAS
organizaciones) y dos BAJO (inserts directos a `schedule_rules`
esquivando `RESOURCE_IS_DYNAMIC`; `resource_availability_windows` con
`DELETE` habilitado), todos cerrados en la misma pasada (migración 50c).
Segunda vuelta del gate: **LISTO**, con 3 hallazgos BAJO residuales
registrados como deuda técnica no bloqueante (ver `docs/database.md`
"Fase 50" y `docs/security.md` "Fase 50" para el detalle completo).
Gate de `reviewer`: **LISTO**.

Verificado en vivo contra `reservaste-stg`: 15/15 en
`test/phase50.dynamic-availability.test.ts`, regresión limpia en el
resto de la suite afectada.

**Próximo paso**: `frontend-engineer` construye el flujo nuevo (elegir
servicio → día → horario → hold → confirmar), contra el contrato real
ya cerrado arriba.
