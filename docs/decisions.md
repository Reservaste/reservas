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

**Implementación**: pendiente, próxima ronda (backend: schema + función
de cotización; frontend: formulario de plan + formulario de pago).

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

**Implementación**: pendiente, próxima ronda (backend: schema + 4
triggers + policy, con revisión obligatoria de `security-engineer`;
frontend: pantalla de solo lectura para el `OWNER`).

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

**Implementación**: pendiente, con revisión obligatoria de
`security-engineer` antes de cerrarse (token portador + acceso a datos
de terceros).

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
