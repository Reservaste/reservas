# Propuesta ADR-0032 — `audit_log`: registro de acciones sensibles (pagos, reservas de mostrador, suspensión, plan)

Estado: **Propuesta** (no implementada)
Propuesta por: `backend-engineer`
Fecha: 2026-09-23
Toca: base de datos, autorización, contratos → **decisión del Orchestrator** (CLAUDE.md)
Alcance ya acotado por el usuario: **no** todo el sistema. Sólo pagos (alta y
anulación), cambios de reserva hechos por staff (no por el propio cliente),
suspensión de organización y cambios de plan.

---

## 1. Problema

Hoy el sistema conserva *estados*, no *hechos*. Un `Payment` anulado dice
que está `VOID` pero no quién lo anuló ni cuándo; una `Booking` cancelada
guarda `cancelled_by` y `cancellation_reason` (ADR-0010) pero una reserva
*creada* por el mostrador en nombre de un cliente guarda `created_by` y
nada más; `organizations.subscription_status` cambia sin dejar rastro de
quién lo cambió; un `ServicePlan` desactivado guarda `cancelled_at` pero
un cambio de **precio** —que es libre por ADR-0024 resolución 5— no deja
absolutamente nada.

Las tres preguntas que hoy no se pueden contestar, y que son las que
aparecen en un mostrador real:

- "Este pago figuraba y ahora está anulado. ¿Quién lo anuló?"
- "A esta persona la anotaron en una clase que no pidió. ¿Quién la anotó?"
- "El plan salía 2500 y ahora 3400. ¿Desde cuándo y quién lo cambió?"

## 2. Un hallazgo que condiciona el diseño

**La mitad de las escrituras sensibles del alcance no pasan por ninguna
RPC.** Verificado en el código, no supuesto:

| Acción sensible | Cómo se escribe hoy |
|---|---|
| Alta de pago | `INSERT` PostgREST directo (`frontend/app/actions/billing.ts:79`) |
| Anulación de pago | `UPDATE payments SET status='VOID'` PostgREST directo (`billing.ts:154`) |
| Cambio de estado de pago | RPC `set_payment_status()` |
| Cambio de precio/nombre de plan | `UPDATE service_plans` PostgREST directo (`service-plans.ts:444`) |
| Activar/desactivar plan | `UPDATE service_plans` PostgREST directo (`service-plans.ts:491`) |
| Crear plan | RPC `create_service_plan()` |
| Reserva por mostrador | RPC `admin_book_for_customer()` |
| Suspensión de organización | RPC `set_organization_subscription()` |

Esto decide la pregunta que el pedido plantea como abierta ("¿insert
explícito en cada RPC, o trigger transversal?"): **un insert explícito en
cada RPC deja fuera el alta de pago, la anulación de pago y todos los
cambios de plan.** Es decir, deja fuera la mayoría de lo que se pidió
auditar. No es una preferencia de estilo: es una cobertura incompleta.

Y es coherente con el invariante 6 de `invariants.md` ("toda tabla es
escribible por PostgREST directo, así que la defensa es un
CHECK/trigger/índice") y con la lección de ADR-0028 (lo que se confía a
que cada llamador recuerde hacer, en algún lado no se hace).

## 3. Recomendación: **trigger para el hecho, contexto opcional desde la RPC**

Híbrido, con el reparto claro:

- **El trigger es la fuente del hecho.** `AFTER INSERT OR UPDATE` sobre
  `payments`, `service_plans`, `bookings` y `organizations`, con un
  predicado `WHEN` que lo limita a lo que se pidió auditar. Escribe una
  fila de `audit_log` con el *qué*, el *quién* (`auth.uid()`) y el
  *cuándo*. No se puede evadir: si la fila cambió, la auditoría existe.
- **La RPC sólo agrega intención**, cuando la tiene y el trigger no la
  puede deducir: "esto fue una suspensión por falta de pago", "esta
  cancelación fue pedida por teléfono". Se pasa por
  `set_config('app.audit_note', ..., true)` (local a la transacción) y el
  trigger la levanta si está. **Si nadie la setea, la fila igual se
  escribe.** El contexto es opcional; el hecho no.

La objeción legítima al trigger es la "magia": alguien lee
`admin_book_for_customer()` y no ve que audita. Se mitiga con dos cosas
concretas, no con disciplina:

1. Un `comment on trigger` en cada uno que diga qué audita y por qué.
2. Una sección propia en `docs/database.md` con la tabla de arriba —qué
   acción, qué trigger la registra— para que "¿esto se audita?" se
   conteste en un archivo y no leyendo ocho funciones.

El costo del camino alternativo es peor y es concreto: son ~8 inserts
explícitos repartidos en RPCs, más 3 acciones que no tienen RPC y
habría que **convertir en RPC sólo para poder auditarlas** — reescribir
`registerPayment`, `voidPayment` y los dos updates de plan, con su
migración, su manejo de errores y sus tests, para obtener menos cobertura
que un trigger. Eso sí es trabajo, y no compra legibilidad: compra ocho
lugares donde olvidarse.

## 4. Shape

```sql
create type public.audit_action as enum (
  'PAYMENT_CREATED',
  'PAYMENT_STATUS_CHANGED',   -- incluye la anulación (VOID)
  'BOOKING_CREATED_BY_STAFF',
  'BOOKING_CANCELLED_BY_STAFF',
  'SERVICE_PLAN_CREATED',
  'SERVICE_PLAN_UPDATED',     -- precio, nombre, orden
  'SERVICE_PLAN_DEACTIVATED',
  'ORGANIZATION_SUBSCRIPTION_CHANGED'
);

create table public.audit_log (
  id              uuid primary key default gen_random_uuid(),
  -- Nullable: la suspensión de una organización la hace un platform admin
  -- que no es miembro de ella. Igual se completa siempre que haya una
  -- organización involucrada, que es todos los casos del alcance.
  organization_id uuid references public.organizations (id) on delete cascade,
  actor_id        uuid references public.profiles (id),   -- auth.uid(); null = job/cron
  action          public.audit_action not null,
  target_table    text not null,
  target_id       uuid not null,
  -- Sólo los campos que cambiaron y que importan, nunca la fila entera:
  -- una copia completa de `payments` duplica datos personales en una
  -- tabla con reglas de acceso distintas.
  metadata        jsonb not null default '{}'::jsonb,
  created_at      timestamptz not null default now()
);

create index audit_log_org_time_idx on public.audit_log (organization_id, created_at desc);
create index audit_log_target_idx   on public.audit_log (target_table, target_id, created_at desc);
```

Notas de diseño, cada una con su motivo:

- **`action` es un enum, no texto libre.** Un texto libre convierte cada
  consulta en un `like` y cada typo en una fila que no se encuentra
  nunca. El alcance está acotado a propósito: ocho valores, y agregar uno
  es una migración de una línea.
- **`target_table` + `target_id` en vez de una FK por tabla.** Cuatro
  columnas FK nullables para expresar "una de estas cuatro" es peor: no
  se puede consultar uniformemente y tres siempre están vacías. Se acepta
  no tener integridad referencial sobre el target — **a propósito**: un
  log de auditoría tiene que sobrevivir al borrado de lo que audita, y
  una FK con `ON DELETE CASCADE` borraría justo la evidencia.
- **`actor_id` es FK a `profiles` y nullable.** Nullable no es descuido:
  ADR-0019 reevalúa fechas desde jobs, y una fila con actor nulo dice "lo
  hizo el sistema", que es información. Escribir un UUID sintético de
  "sistema" sería una mentira con forma de dato.
- **`metadata` con el diff mínimo**, p. ej.
  `{"from":"PENDING","to":"VOID"}` o `{"price":{"from":2500,"to":3400}}`.
  Nunca nombres, mails ni teléfonos: esos ya viven en sus tablas con su
  RLS y copiarlos acá los saca de ese control.
- **Sin `updated_at`, sin `cancelled_at`.** Una fila de auditoría es
  inmutable por definición; darle columnas de ciclo de vida invita a
  editarla.

## 5. Inmutabilidad: es la mitad del valor

Un log que el auditado puede editar no es un log. Se aplica el mismo
patrón que `makeup_credits` de ADR-0025 (que ya es `SELECT`-only, cero
policies de escritura), reforzado:

```sql
alter table public.audit_log enable row level security;
-- Ninguna policy de INSERT/UPDATE/DELETE, para ningún rol. Las filas
-- nacen sólo desde triggers SECURITY DEFINER.
revoke insert, update, delete on public.audit_log from authenticated, anon;
```

más un trigger `before update or delete` que hace `raise exception
'AUDIT_LOG_IMMUTABLE'`, para que tampoco un `SECURITY DEFINER` futuro
pueda tocarlo por accidente. `service_role` **no** se exceptúa: si hace
falta corregir algo, se corrige en la base con un superusuario y queda
registrado fuera del producto, que es exactamente lo que corresponde.

## 6. Quién lee

**Recomendación: `is_platform_admin()` ve todo, y el `OWNER` de una
organización ve el log de su propia organización. `STAFF` no ve nada.**

```sql
create policy audit_log_select on public.audit_log for select using (
  public.is_platform_admin()
  or (organization_id is not null and public.is_organization_owner(organization_id))
);
```

El argumento a favor del OWNER es el que motivó el pedido: las tres
preguntas de §1 son preguntas del dueño del negocio sobre su propio
personal, no de la plataforma. Un log que sólo puede leer el proveedor del
SaaS obliga al dueño a abrir un ticket para saber quién anuló un pago en
su propio mostrador, que es absurdo y además nos convierte en el soporte
de primera línea de cada disputa interna de cada cliente.

El argumento en contra —"el OWNER puede usarlo para vigilar a su
personal"— es real pero es una decisión del negocio sobre su propia
operación, no un problema de la plataforma; y los datos involucrados
(montos, estados, IDs) ya son todos visibles para el OWNER en las
pantallas que tiene. La auditoría no le muestra nada nuevo: le muestra
*cuándo y quién*, que es justamente lo que pide.

`STAFF` queda afuera porque es el sujeto mayoritario del log. Con
`is_organization_owner()` alcanza, y esa función ya existe (es la que usa
la policy de `organizations`).

**Queda como decisión del Orchestrator**: si el `OWNER` debería ver
también las filas de `ORGANIZATION_SUBSCRIPTION_CHANGED` (es decir,
"quién de la plataforma me suspendió"). Mi recomendación es **sí**: una
suspensión que el dueño no puede rastrear es exactamente el tipo de cosa
que genera desconfianza en un SaaS. Pero `actor_id` de un platform admin
no debería resolverse a un nombre en la UI del cliente.

## 7. Alcance inicial, con precisión

**Entra:**

| Acción | Origen | Disparador |
|---|---|---|
| Alta de pago | `INSERT payments` (PostgREST) | trigger `payments_audit` (INSERT) |
| Cambio de estado de pago, incluida la anulación | `UPDATE payments`, `set_payment_status()` | trigger `payments_audit` (UPDATE WHEN status cambió) |
| Reserva creada por staff en nombre de un cliente | `admin_book_for_customer()` | trigger `bookings_audit` (INSERT WHEN el actor no es el `profile_id` del customer) |
| Cancelación de reserva hecha por staff | `cancel_booking()` con actor staff, `cancel_slot_occurrence()`, `discontinue_schedule_rule()`, `cancel_recurring_booking()` con actor staff | trigger `bookings_audit` (UPDATE a `CANCELLED` WHEN el actor no es el `profile_id` del customer) |
| Creación de plan | `create_service_plan()` | trigger `service_plans_audit` (INSERT) |
| Cambio de precio/nombre de plan | `UPDATE service_plans` (PostgREST) | trigger `service_plans_audit` (UPDATE) |
| Desactivación/reactivación de plan | `UPDATE service_plans` (PostgREST) | trigger `service_plans_audit` (UPDATE WHEN `is_active` cambió) |
| Suspensión / cambio de plan SaaS de la organización | `set_organization_subscription()` | trigger `organizations_audit` (UPDATE WHEN `subscription_status` o `plan_code` cambió) |

**La condición "por staff, no por el propio cliente"** se evalúa en el
trigger comparando `auth.uid()` contra
`customers.profile_id` de la reserva. Es la misma comparación que
`cancel_booking()` ya hace para autorizar, y escribirla en un solo lugar
evita el problema de tres valores que documenta `invariants.md`: con un
cliente gestionado `profile_id` es `NULL` (ADR-0026), y
`auth.uid() <> profile_id` da `NULL`, que en un `if` **no entra**. Se
escribe `profile_id is distinct from auth.uid()` — así una reserva de un
cliente gestionado, que por definición la hizo el mostrador, se audita.

**NO entra** (explícito, para que nadie lo agregue "de paso"):

- Asistencia (`mark_attendance`) — es un hecho operativo de alto volumen,
  una clase de 3 personas genera 3 filas por día.
- Créditos de recupero — `makeup_credits` **ya es su propio log**:
  `origin`, `issued_by`, `issued_at`, `source_booking_id`,
  `consumed_booking_id` y RLS `SELECT`-only. Duplicarlo en `audit_log`
  crearía dos versiones de la misma verdad. `grant_manual_makeup_credit()`
  (OWNER-only, con nota obligatoria) ya cumple lo que ADR-0025
  resolución 5 pedía por "auditado".
- Reservas y cancelaciones hechas por el propio cliente.
- Altas/bajas de cliente, servicios, horarios, branding, configuración.
- Logins y lecturas. No se pidió y multiplicaría el volumen por dos
  órdenes de magnitud.

## 8. Volumen y retención

Estimación con el cliente actual (un estudio de pilates): pagos ~40/mes,
reservas de mostrador ~100/mes, planes ~5/mes → **< 2.000 filas al año por
organización**. No hace falta particionar ni archivar en el MVP.

**Queda como decisión del Orchestrator** si se define una retención (p.
ej. 24 meses) desde ahora. Mi recomendación: **no todavía**, pero dejar el
índice `(organization_id, created_at desc)` que un borrado por fecha
necesitaría, para no tener que agregarlo sobre una tabla grande después.

## 9. Impacto

| Área | Qué cambia |
|---|---|
| Schema | 1 tabla, 1 enum, 4 triggers, 1 trigger de inmutabilidad, 1 policy. Todo aditivo. |
| RPCs existentes | **Ninguna cambia de firma.** Opcionalmente algunas setean `app.audit_note`. |
| Motor de reservas / pagos | Nada. La auditoría es AFTER y no puede rechazar nada. |
| Seguridad | Revisión de `security-engineer` obligatoria: tabla sin policies de escritura, `SECURITY DEFINER` en los triggers, y el `revoke ... from public, anon` de ADR-0028. |
| Frontend | Una pantalla de sólo lectura para el OWNER. No bloquea la migración. |
| Tests | Que el hecho se registre aunque la escritura no pase por RPC; que el actor sea el correcto con cliente gestionado (`profile_id` nulo); que `authenticated` no pueda insertar ni borrar; que un `STAFF` no lea; que un OWNER no lea el log de otra organización. |

## 10. Migración / compatibilidad

Tabla nueva, sin backfill: **no se puede inventar quién hizo qué antes de
que existiera el log**. El log arranca el día del deploy y eso se dice en
la pantalla ("registro desde DD/MM/AAAA") en vez de mostrar un vacío que
se lea como "no pasó nada".

## 11. Riesgos

1. **La "magia" del trigger.** §3 — mitigada con comentarios y una tabla
   en `docs/database.md`, no con disciplina.
2. **Un `metadata` que crezca hacia "la fila entera".** Es el modo típico
   de fallar de estas tablas y termina duplicando datos personales fuera
   de su RLS. Mitigación: el diff lo arma una función por tabla, no un
   `to_jsonb(new)` genérico.
3. **La regla "por staff, no por el cliente" mal escrita** es lógica de
   tres valores otra vez (§7). Es el mismo error que ADR-0028 documentó y
   tiene que tener su test.
