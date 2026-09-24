# Contratos de API / Server Actions

Mantenido por: `backend-api-agent`. Aprobado por: Orchestrator.
Estado: **Draft de boundaries — no es obligatorio implementar REST literal
si el proyecto usa server actions; lo que importa es respetar estos tres
niveles de acceso.**

## Los tres niveles

### PUBLIC (sin autenticación)

```
GET /organizations/:slug/services
GET /organizations/:slug/availability
```

Rutas de UI conceptuales: `/[organizationSlug]` (perfil público) y
`/[organizationSlug]/reservar` (flujo de reserva).

Solo datos públicos: servicios, fechas, horarios, disponibilidad agregada.
Nunca nombres de clientes, bookings individuales, pagos, ni ninguna
escritura server-side (no se crea `Booking`/`Customer` en draft para un
anónimo).

**Excepción puntual (ADR-0030 resolución 2):**
`POST /contacto` (`submitContactRequest()` en
`frontend/app/actions/contact.ts`, RPC `submit_platform_contact_request`)
es la única escritura anónima permitida — un lead del formulario público
de la landing, sin relación con `Organization`/`Customer`/`Booking`. Solo
el platform admin lee esas filas (`GET /admin`, gateado por
`is_platform_admin()`), nunca es un dato público de lectura. Rate-limited
en la RPC (ver `docs/database.md`, Fase 24) en vez de un endpoint que
pueda floodearse sin control.

**Shape de disponibilidad — `publicAvailabilityDisplay` (ADR-0008):**
unión discriminada por `mode`, nunca incluye el conteo exacto salvo en
`EXACT`:

```jsonc
// EXACT
{ "availability": { "mode": "EXACT", "remaining": 4, "capacity": 12, "label": "4 lugares disponibles" } }
// LIMITED
{ "availability": { "mode": "LIMITED", "status": "AVAILABLE"|"LOW"|"FULL", "label": "..." } }
// BOOLEAN
{ "availability": { "mode": "BOOLEAN", "status": "AVAILABLE"|"FULL", "label": "..." } }
```

Modo efectivo = `service.publicAvailabilityDisplayOverride ??
organization.publicAvailabilityDisplay`. Aplica igual a un detalle de
slot puntual, no solo al listado agregado.

### CUSTOMER (autenticado)

```
GET    /organizations/:slug/bookings/check?serviceId=&slotOccurrenceId=
POST   /organizations/:slug/bookings
DELETE /organizations/:slug/bookings/:id
POST   /organizations/:slug/recurring-bookings/preview
POST   /organizations/:slug/recurring-bookings
GET    /me/bookings
GET    /me/services
GET    /me/payments
```

Los endpoints de booking se anidan bajo `/organizations/:slug/` (ADR-0015)
en vez de una ruta flat `/bookings`, para que `organizationId` se resuelva
sin ambigüedad desde el slug antes de tocar cualquier dato.

Todo endpoint de este nivel valida: identidad del `Profile`, que el recurso
pedido (`booking`, `payment`, etc.) le pertenezca, y las reglas de
`canCustomerBook()` cuando corresponda.

`POST .../bookings` **nunca** acepta `customerId` en el body como valor de
confianza — deriva `profileId` de la sesión y `organizationId` del slug,
y llama a `canCustomerBook(profileId, organizationId, serviceId,
slotOccurrenceId)` que resuelve el `Customer` internamente (ADR-0005).

`GET .../bookings/check` (ADR-0015): revalida `canCustomerBook()` de
solo-lectura, sin reservar — se usa al volver de un login durante el
flujo de reserva, para no mostrarle al usuario un botón "Confirmar" que
se sabe de antemano que va a fallar. **No es un gate de seguridad, es
UX**: `POST .../bookings` siempre vuelve a evaluar `canCustomerBook()` de
cero antes de invocar la RPC (ADR-0004), exacto mismo boundary que si
`check` no existiera.

`POST .../recurring-bookings/preview` (ADR-0012): dry-run de solo lectura
que devuelve, para las próximas N ocurrencias de un patrón recurrente,
cuáles tienen cupo y cuáles están en conflicto (con fecha exacta) — nunca
reserva. El admin/cliente confirma explícitamente antes de
`POST .../recurring-bookings`, que sí crea la serie y reserva cada
ocurrencia ya materializada vía la misma RPC atómica de siempre.

`GET /me/bookings` — **pendiente de definir** (Phase 1/2): un mismo
`Profile` puede ser `Customer` de varias `Organization`, así que hay que
decidir si este endpoint agrega across todas las organizaciones del
profile o requiere un parámetro de organización. No bloquea Phase 0.

### ADMIN (OWNER/STAFF de la organización)

```
POST  /services
POST  /schedule-rules
POST  /customers
POST  /payments
PATCH /bookings/:id
```

Todo endpoint de este nivel valida membership (`OrganizationMember`) sobre
el `organizationId` del recurso, no solo que el usuario esté autenticado.

## Reglas generales

- TypeScript estricto, validación de input con Zod en el borde de la
  aplicación.
- El endpoint/server action es una capa fina de transporte: valida input,
  llama a la lógica de dominio (`booking-engine-agent`,
  `payments-entitlements-agent`, etc.), traduce el resultado. La lógica de
  negocio no vive duplicada en el handler.
- `organizationId` siempre se deriva de la sesión/membership del usuario o
  del slug público, nunca se acepta como parámetro libre para operaciones
  privadas.
- Naming consistente entre lo que expone el backend y lo que consume el
  frontend (mismo vocabulario que `domain.md`).

## Pendiente de definir (Phase 0/2)

- Si se implementa como REST, RPC, o server actions puras (a confirmar
  como ADR una vez elegido el stack).
- Shapes exactos de request/response por endpoint (se agregan acá a medida
  que se implementan, fase por fase).

---

## Fase 25 — cambios de contrato en server actions (`frontend/app/actions/`)

Todos aditivos o compatibles hacia atrás. Lo que necesita trabajo de
`frontend-engineer` está marcado.

### `service-plans.ts` — `listPaymentPlanOptions(organizationSlug, month?)`

Parámetro nuevo **opcional** `month` (`"YYYY-MM"`). Sin él se comporta
exactamente como antes.

El motivo es el reporte 3 del cliente. La pantalla de pagos es una pantalla
*sobre un mes* (`/org/[slug]/payments/[customerId]?mes=YYYY-MM`) y el
formulario "Agregar pago" que vive abajo resolvía el período desde **hoy**,
sin importar qué mes estuviera en pantalla. Parado en octubre y registrando
un pago, el período que se mandaba era el de septiembre: con septiembre ya
`PAID` el `EXCLUDE` lo rechazaba ("ya hay un pago que cubre ese período" —
de un mes que el mostrador ni estaba mirando), y con `PENDING` entraba en un
mes que esa pantalla no lista, así que parecía que lo habían rechazado.

**Resuelto:** `app/org/[slug]/payments/[customerId]/page.tsx` pasa `mes`,
y `RegisterPaymentForm` lo usa como default en vez de
`firstOfMonth`/`lastOfMonth` calculados en el browser.

### `standing.ts` — `StandingReservation.upcomingBeyondPeriod`

Campo nuevo. Cambia además el **significado** de `upcomingUnpaid`: ahora son
sólo las fechas que se pueden cobrar hoy, no todas las de la ventana rodante
de 90 días (ver `database.md` §Fase 25.3).

**Resuelto:** la etiqueta roja "Falta el pago" de `standing-reservations.tsx`
ya se apaga sola para un cliente al día. Las fechas de `upcomingBeyondPeriod`
tienen su propio badge neutro ("N fuera del período") con una línea que
explica que se confirman solas cuando llegue ese pago — no una nueva alerta.

### `customer.ts` — `checkCanBookDetail(slotOccurrenceId)`

Función nueva; `checkCanBook` no se toca. Devuelve
`{ reason, makeupCreditId, makeupCreditExpiresOn }` sobre la RPC
`can_customer_book_detail()`.

**Resuelto:** `reservar/confirmar/page.tsx` muestra "esta reserva usa tu
crédito de recupero, vence el DD/MM" antes del botón de confirmar, sólo
cuando `makeupCreditExpiresOn` viene informado (sin ruido en el camino
normal).

`CanBookResult` suma `OUTSIDE_PLAN_QUOTA`, `OVER_PLAN_QUOTA` y
`SERVICE_HAS_NO_PLAN`: la RPC ya los devolvía desde ADR-0024 y el tipo
prometía menos motivos de los reales (`BOOKING_REASONS` ya sabía decirlos).

### `customer.ts` — `releaseMyBooking` deja de afirmar un éxito que no ocurrió

Ignoraba el error de la RPC y redirigía igual con `?liberado=1`. Ahora, si
la llamada falla, redirige a `/me` sin ese parámetro.

**Resuelto:** `/me` lee `?liberar_error=1` y muestra un `Alert` de error de
verdad en vez de asumir éxito.

### `settings.ts` — configuración del crédito de recupero

`updateOrganizationSettings` acepta (opcionalmente, campo por campo)
`makeupCreditsEnabled`, `releaseDeadlineHours`, `makeupCreditExpiry` y
`makeupCreditExpiryDays`. Un formulario que no los manda deja la
configuración como está.

Es el reporte 4: ADR-0025 resolución 3 hizo el crédito **opt-in**
(`organizations.makeup_credits_enabled` default `false`) y **nada en el
producto lo escribía nunca** — ni una acción, ni un formulario. Así que para
toda organización real el flag está en `false`, `issue_makeup_credit()` sale
en su condición 1 y "liberar cupo" cancela la fecha sin emitir nada.

**Resuelto:** controles nuevos en `/org/[slug]/settings/settings-form.tsx`
(toggle + anticipación mínima + tipo de vencimiento), gateados a OWNER.

### `billing.ts` — mensajes para los rechazos nuevos

`PAYMENT_DUPLICATE_PERIOD`, `PAYMENT_DUPLICATE_OCCURRENCE` y
`SERVICE_PLAN_SCOPE_EMPTY` se traducen a texto de mostrador en vez de caer
en el genérico "No se pudo registrar el pago".

---

## Feedback de producción — "no veo la agenda para reservar"

Ningún shape de request/response cambia acá: lo que cambia es **a dónde
aterriza** cada acción. Se documenta igual porque el destino de un
`redirect()` es parte observable del contrato para `frontend-engineer`.

### `activation.ts` — `claimActivation()` aterriza en el negocio que invitó

Antes: `redirect("/me?activado=1")`. Ahora: `redirect("/{organization_slug}?activado=1")`,
con el slug que devuelve `claim_customer_activation()` (resuelto adentro de
la RPC desde el token — el caller nunca lo elige). Si el slug no viniera o
no tuviera forma válida, el fallback sigue siendo `/me`.

El motivo: `/me` para alguien **recién** activado está vacío por
definición (todavía no reservó nada) y no tiene un solo link a la agenda
del negocio — el nombre de la organización aparece en `/me` recién cuando
ya existe una `Booking`. La activación es la invitación de una
organización puntual y su página pública **es** su agenda (ADR-0023), así
que tirar ese contexto para caer en un portal genérico era terminar el
flujo en un callejón sin salida.

**Pendiente de `frontend-engineer` (no bloqueante):** `?activado=1` ya
viaja en la URL, pero `app/[organizationSlug]/page.tsx` todavía no lo lee.
Un `Alert` de éxito ("Listo, ya podés reservar en {negocio}") cierra el
flujo; sin él la confirmación es implícita (la agenda del negocio con su
marca).

### `auth.ts` — `signUpWithPassword()` sin `returnTo` va a `/dashboard`

El default era `/onboarding` ("creá tu negocio"): la misma puerta
equivocada que la Fase 9 ya había sacado de `/dashboard`, sobreviviendo en
la acción hermana. Hoy `/signup` siempre manda un `returnTo`, así que era
una trampa latente y no un bug en vivo, pero el default de "intención
desconocida" tiene que ser el mismo en las dos puertas: `/dashboard`
pregunta en vez de adivinar, y "Crear mi organización" sigue estando a un
tap.

### Helper nuevo: `frontend/lib/organization-path.ts`

`organizationPath(slug)` → `/slug` o `null`. Valida contra
`organizationSlugSchema` de `@reservaste/domain` (no una regex nueva) para
que un `redirect("/" + slug)` armado desde el valor de retorno de una RPC
no pueda salirse de la página que se quiso. Nunca "limpia" el slug:
devuelve `null` y el caller decide el fallback.

## Hotfix 2026-09-23 — el link de activación "vencido" en el primer intento

Ningún shape de request/response cambia. Lo que cambia es el **transporte**
del token y el destino del mail de confirmación.

### `frontend/lib/activation-cookie.ts` (nuevo) — la cookie dura lo que el token

`ACTIVATION_COOKIE_NAME`, `ACTIVATION_TOKEN_TTL_SECONDS` (72 h, espejo del
`interval '72 hours'` de `issue_customer_activation()`),
`activationCookieOptions()` y `activationCookieClearOptions()`. Lo usan el
route handler `GET /activar/[token]` y las server actions de
`activation.ts`, que antes repetían la política cada una por su cuenta —
con dos valores distintos de `path` al borrar y un `maxAge` de 15 min que
era, en los hechos, el vencimiento real del link. Invariante: **la cookie
nunca puede vencer antes que el token**.

### `activation.ts` — `claimActivation()` sin cookie ya no dice "venció"

El texto pasó de *"Este link ya no es válido. Pedile al negocio que te lo
reenvíe."* a *"No encontramos la invitación en este navegador. Volvé a
abrir el link de WhatsApp desde este mismo teléfono y seguí desde ahí."*.
No hay cookie ≠ link vencido: el token sigue pendiente en la base y pedir
uno nuevo no arregla nada. El campo sigue siendo `{ error: string | null }`.

**Pendiente de `frontend-engineer` (misma causa, otro archivo):**
`app/activar/continuar/page.tsx` muestra, cuando no hay cookie, *"Este link
ya no es válido — Puede haber vencido o ya haberse usado"*. Es la pantalla
donde más se ve el mensaje equivocado y es UI, así que no la toqué: el
texto correcto es el mismo de arriba ("abrí de nuevo el link de WhatsApp
desde este teléfono").

### `auth.ts` — `signUpWithPassword()` pasa `emailRedirectTo`

`options.emailRedirectTo = ${siteUrl()}/auth/callback?next=<returnTo>`
(misma forma que el `redirectTo` de Google, ya en el allowlist de Supabase
Auth). Sin eso, con "Confirm email" prendido el link del mail caía en la
Site URL del proyecto y la intención con la que la persona se registró se
perdía ahí: el cliente gestionado que se registra **para** activar volvía a
la home y nunca a `/activar/continuar`. Aplica a todo signup con
`returnTo`, no sólo a la activación (ADR-0015 tenía el mismo agujero).

---

## Horario fijo — camino rápido sin preview (`standing.ts`)

Pedido del usuario: asignar un cupo fijo no tiene por qué pasar por la
lista de las próximas 8 fechas, y la serie "va a futuro
indeterminadamente".

**Nada de esto necesitó migración.** El estado real antes del cambio:

- `recurring_bookings.end_date` es `date` **nullable** desde Fase 6 y
  `admin_create_recurring_booking(p_schedule_rule_id, p_customer_id)` no
  lo recibe ni lo escribe: **toda serie nace sin fecha de fin**, y la
  ventana rodante de ADR-0009 le va agregando fechas mientras esté
  `ACTIVE` (`generate_slot_occurrences_for_rule()` genera el `Booking` de
  cada ocurrencia nueva para cada `recurring_booking` activo de la regla).
  La pantalla nunca ofreció una fecha de corte: lo que faltaba era
  **decirlo**, no soportarlo. Pinneado en
  `backend/test/phase11.standing-reservations.test.ts` ("the series has no
  end date…").
- El preview nunca fue una validación, sólo información:
  `createStandingReservation` sólo necesita `customerId`, y membresía,
  pertenencia del cliente, serie duplicada y cupo del plan se verifican
  dentro de la RPC. Saltearlo no abre ningún agujero.

### `StandingActionState.summary` (campo nuevo, opcional)

`createStandingReservation` devuelve ahora, cuando la serie se creó,
`summary: StandingCreateSummary`:

```ts
{ confirmed, pending, unpaid, overQuota, beyondPeriod, unavailable }
```

`unpaid + overQuota + beyondPeriod + unavailable === pending` (las cuatro
son disjuntas; `unavailable` se calcula por resta para que un
`not_generated_reason` nuevo no desaparezca de la cuenta). Sale de releer
`schedule_rule_standing_reservations()` después de crear — la RPC de
creación devuelve la fila de `recurring_bookings`, no el estado de los
`Booking` que generó, y cambiarle la firma rompería a sus otros callers.

`success` dejó de ser el fijo `"Horario fijo asignado"`: ahora es la frase
armada desde ese resumen ("Horario fijo asignado: 12 fechas confirmadas. Se
repite todas las semanas, sin fecha de fin, hasta que lo quites. Quedan 3
sin confirmar: …"). Es lo que reemplaza al preview obligatorio — sin eso,
el camino rápido no distingue al cliente con las doce fechas confirmadas
del que no confirmó ninguna porque debe el mes.

El campo es **opcional** a propósito: `{ error: null, success: null }`
como `initialState` sigue tipando. `previewStandingReservation` no cambió
de firma.

### Pendiente de `frontend-engineer` — `standing-reservations.tsx`

Es UI, así que no la toqué. Lo que falta:

1. **Botón directo "Asignar horario fijo"** en el mismo form donde se
   elige el cliente (hoy ese form sólo puede disparar el preview, y el
   submit de creación vive escondido detrás de `preview.dates.length > 0`).
   El preview queda como acción secundaria del mismo form —"Ver detalle
   antes de confirmar"— sin dejar de ser el camino que ya existe.
2. **Decir que no hay fecha de fin** junto al botón ("Se repite todas las
   semanas hasta que lo quites"), que es el otro medio pedido: la serie ya
   es indefinida, pero la pantalla nunca lo dijo.
3. Renderizar `created.summary` (o simplemente `created.success`, que ya
   trae la frase completa) después de confirmar.

**No agregar un selector de fecha de fin.** `generate_recurring_booking()`
no mira `end_date`, así que una serie con fecha de corte seguiría
generando `Booking` después de esa fecha: ofrecerlo hoy sería prometer
algo que la base no cumple (ver el reporte al Orchestrator).

## Fase 28 — catálogo de planes para el cliente y solicitud de cambio

Contratos **nuevos** (nada existente cambió de firma). Detalle de la base
en `docs/database.md`, Fase 28.

### `public.ts` — `listPublicServicePlans(organizationSlug, serviceId?)`

**Nivel PUBLIC.** Los planes activos que el negocio publica, con los
servicios que cubre cada uno ya resueltos (RPC `public_service_plans`).
Sin login, mismo nivel de disclosure que `service_plans_select_public`
(Fase 22): nombre, descripción, precio, moneda, tipo, frecuencia y
servicios cubiertos. Ningún dato de ningún `Customer`.

```ts
interface PublicServicePlan {
  id: string;
  name: string;
  description: string | null;
  price: number;
  currency: string;                 // ISO 4217 de la Organization
  planKind: ServicePlanKind;        // DROP_IN | WEEKLY_QUOTA | UNLIMITED
  weeklyQuota: number | null;       // sólo WEEKLY_QUOTA
  quotaScope: "PER_SERVICE" | "SHARED_ACROSS_SERVICES" | null;
  billingType: "ONE_TIME" | "MONTHLY";
  billingCycle: "CALENDAR_MONTH" | "ROLLING_MONTH" | null;
  appliesToAllServices: boolean;
  serviceIds: string[];
  serviceNames: string[];
}
```

`serviceId` la acota a los planes que cubren ese servicio — la forma que
necesitan las pantallas de rechazo `OVER_PLAN_QUOTA`/`OUTSIDE_PLAN_QUOTA`,
donde la persona está parada frente a un servicio concreto.

Los tipos viven en `frontend/app/actions/public.ts` y **no** en
`@reservaste/domain`, a propósito: el paquete es otro repo consumido por
git ref (CLAUDE.md), así que agregarlos allá bloquearía esta pantalla
detrás de un publish + bump. Mismo criterio que `PaymentPlanOption`.

### `plan-changes.ts` (archivo nuevo) — pedir y atender el cambio

El cambio de plan en sí sigue siendo **VOID + recargar** del mostrador
(ADR-0024 resolución 1). Estas acciones sólo registran el pedido: un
pedido pendiente **no habilita ni bloquea una sola reserva**.

| Acción | Nivel | Firma |
|---|---|---|
| `requestPlanChange` | CUSTOMER | `(servicePlanId: string, _prev: ActionState, formData: FormData) => Promise<ActionState>` — nota opcional en `formData.get("note")`, máx. 500. Idempotente: repetir el pedido pendiente devuelve éxito sin duplicar. |
| `getMyPlanChangeRequests` | CUSTOMER | `() => Promise<MyPlanChangeRequest[]>` — pendientes primero, con `currentPlanName` y `resolution`. |
| `listPlanChangeRequests` | ADMIN | `(organizationSlug: string, includeResolved = false) => Promise<PlanChangeRequest[]>` |
| `resolvePlanChangeRequest` | ADMIN | `(organizationSlug: string, requestId: string, resolution: "APPLIED" \| "DISMISSED") => Promise<void>` — target de `<form action>`. |

Errores traducidos por `requestPlanChange`: `NOT_A_CUSTOMER` ("todavía no
sos cliente de este negocio"), `ALREADY_ON_PLAN`,
`SERVICE_PLAN_NOT_AVAILABLE`, `ORGANIZATION_INACTIVE`,
`TOO_MANY_PENDING_PLAN_CHANGE_REQUESTS`, `NOTE_TOO_LONG`.

Registrar el pago del plan pedido (pantalla de pagos, sin cambios) cierra
el pedido como `APPLIED` por trigger, así que `resolvePlanChangeRequest`
es para los casos en que la venta no pasó por ahí.

### Pendiente de `frontend-engineer` (no toqué UI)

1. Pantalla de catálogo por organización — p. ej.
   `/[organizationSlug]/planes` (opcionalmente `?servicio=<serviceId>`),
   alimentada por `listPublicServicePlans`, con un botón por plan que
   dispare `requestPlanChange` (y el estado "ya lo pediste" desde
   `getMyPlanChangeRequests`).
2. **Redirigir ahí el rechazo**: en
   `app/[organizationSlug]/reservar/confirmar/page.tsx` el link de
   `OVER_PLAN_QUOTA`/`OUTSIDE_PLAN_QUOTA` hoy va a `/me/servicios` (el
   plan que ya tiene). Debería ir al catálogo con el servicio del slot.
   `SlotDetail` ya trae `organizationSlug` y `serviceId`.
3. `/me/servicios`: mostrar el pedido pendiente y un acceso al catálogo
   del negocio.
4. Admin: listar los pedidos pendientes (`listPlanChangeRequests`) donde
   el mostrador ya mira planes/pagos, con "marcar como atendido"
   (`resolvePlanChangeRequest`) — recordando que cobrar el plan pedido ya
   lo cierra solo.

## Fase 29 — cierres chicos de la agenda real de `/me`

Dos huecos que dejó la ronda que construyó la agenda con grilla en `/me`
(Fase 23-ish, `frontend/lib/my-agenda.ts` + `docs/database.md`).

### `customer.ts` — `MyBooking` trae identidad de la ocurrencia y color

**Contrato ampliado, no roto** (agrega campos, no quita ni renombra
ninguno). RPC `my_bookings()` (`docs/database.md`, Fase 29) ahora devuelve
`slot_occurrence_id`, `service_id` y `service_color` además de lo que ya
traía:

```ts
interface MyBooking {
  // ...igual que antes...
  slotOccurrenceId: string;
  serviceId: string;
  serviceColor: string | null;   // mismo color que get_public_availability() para el mismo servicio
}
```

Antes de esto, `/me` deduplicaba una reserva propia contra la
disponibilidad pública comparando `startAt + serviceName`
(`occurrenceKey()` en `frontend/lib/my-agenda.ts`) porque no había ningún
id en común entre `my_bookings()` y `get_public_availability()`. El
heurístico sigue funcionando (nunca dejó de ser correcto, solo evitable) —
queda **pendiente de `frontend-engineer`** migrar `occurrenceKey()`/el
componente que lo usa a comparar `slotOccurrenceId` directo y a pintar el
bloque propio con `serviceColor`, sin tocar la regla de negocio de ningún
lado. No lo hice yo: la consigna de esta fase fue no tocar `app/me/*`.

### `customer.ts` — `confirmBooking`/`releaseMyBooking` ahora redirigen con `org=`

Ambos volvían a `/me` (o `/me?liberado=1`, `/me?liberar_error=1`) sin el
`?org=<slug>` que `/me` usa para elegir qué organización mostrar
(`pickCustomerOrganization`, `frontend/lib/my-agenda.ts`). Sin él, alguien
cliente de varias organizaciones podía reservar/liberar en una y terminar
viendo la agenda de otra (la de la próxima clase más próxima, que no es
necesariamente la que acaba de tocar).

- `confirmBooking`: reusa `getSlotDetail(slotOccurrenceId)` (ya existía en
  el mismo archivo, RPC `public_slot_detail`, ADR-0015) para resolver el
  slug — no hizo falta ninguna consulta nueva.
- `releaseMyBooking`: reusa `getMyBookings()` — `my_bookings()` ya traía
  `organization_slug` (usado para `organizationName`) — resuelto *antes*
  de llamar a `release_my_booking()`, así el redirect de error
  (`?liberar_error=1`) también lleva `org=` cuando la reserva es del
  cliente. Una reserva ajena o inexistente no aparece en `getMyBookings()`
  y cae al fallback sin `org=` (mismo comportamiento que antes de esta
  fase, no un caso nuevo).

Ninguna firma de función cambió (siguen `(slotOccurrenceId, prevState)` y
`(bookingId)`); solo cambió el string al que redirigen.

## Fase 30 — `audit_log`: lectura para el OWNER (ADR-0032)

**Ninguna firma existente cambió.** `set_organization_subscription()`
conserva sus cuatro parámetros (sólo setea `app.audit_note` adentro), y
ningún action de pagos, reservas o planes cambió de shape: la auditoría
la escribe un trigger, así que el llamador no se entera. Detalle de la
base en `docs/database.md`, Fase 30.

### RPC nueva — `organization_audit_log(p_organization_id, p_limit, p_before)`

> **Firma reemplazada en la Fase 30b** (cursor compuesto `(created_at, id)`):
> ver "Fase 30b" al final de este documento. `p_before` ya no existe.

**Nivel ADMIN, restringido a OWNER.** `grant execute to authenticated` +
`revoke from public, anon`. Devuelve `NOT_AUTHORIZED` para `STAFF` y para
un `OWNER` de otra organización (no una lista vacía: acá "no sos dueño" y
"no hay nada registrado" tienen que distinguirse, porque la pantalla dice
"registro desde tal fecha", no "no pasó nada").

```ts
interface AuditLogEntry {          // ya vive en @reservaste/domain
  id: string;
  action: AuditAction;             // 8 valores, ver domain.md
  targetTable: string;             // "payments" | "bookings" | "service_plans" | "organizations"
  targetId: string;
  actorId: string | null;          // null = sistema, o actor de plataforma visto por el OWNER
  actorName: string | null;
  actorIsPlatform: boolean;        // true = la acción la hizo alguien que no es de esta organización
  metadata: Record<string, unknown>;  // diff mínimo: {"price":{"from":2500,"to":3400}}
  createdAt: string;
}
```

`p_limit` se acota en SQL a 1..200 (default 100) y `p_before` pagina hacia
atrás por `created_at` — el orden es `created_at desc, id desc`.

**El matiz de ADR-0032 resolución 2 está resuelto del lado de la lectura,
no de la UI**: cuando el actor no es miembro de la organización (es decir,
un platform admin suspendiendo o reactivando la cuenta), la función no
joinea `profiles` y devuelve `actorId = null`, `actorName = null` y
`actorIsPlatform = true`. El dueño ve **qué** pasó y **cuándo**, no quién
de nuestro lado lo hizo. Para un llamador `is_platform_admin()` devuelve
el actor completo.

### Pendiente de backend (no lo hice en esta ronda, por consigna)

`getOrganizationAuditLog()` en `frontend/app/actions/` (archivo nuevo
`audit.ts`, o dentro de `organization.ts` si se prefiere). Contrato
propuesto, sin sorpresas respecto de la RPC:

```ts
export async function getOrganizationAuditLog(
  organizationSlug: string,
  options?: { limit?: number; before?: string },
): Promise<AuditLogEntry[]>
```

Resuelve el `organizationId` desde el slug + membership como ya hacen las
demás acciones ADMIN (nunca un `organizationId` de parámetro libre) y
traduce `NOT_AUTHORIZED` a "sólo el dueño puede ver el registro".

### Pendiente de `frontend-engineer` — la pantalla

Solo lectura, para el `OWNER`, en `/org/[slug]/configuracion` (es
configuración del negocio, no una operación de mostrador; y ahí ya está
el resto de lo que sólo el dueño toca). Nada de filtros elaborados en el
primer corte: una lista cronológica descendente con

- **qué pasó**, en castellano y por acción (no el enum crudo): "Se
  registró un pago", "Se anuló un pago", "Se anotó a un cliente", "Se
  canceló una reserva", "Se cambió el precio de un plan", "Se suspendió
  la cuenta"...
- **quién**: `actorName`, o "Reservaste" / "Soporte de la plataforma"
  cuando `actorIsPlatform` es `true` (**nunca** intentar resolver ese
  actor a un nombre, y no mostrar el uuid: viene `null` a propósito), o
  "Sistema" cuando `actorId` es `null` y `actorIsPlatform` es `false`.
- **cuándo**, en la zona de la organización (ADR-0014).
- el diff de `metadata` cuando aporta ("de $2500 a $3400", "de PENDIENTE
  a ANULADO"). `metadata.note` es texto de contexto que puso la RPC.

Dos cosas que la pantalla debe decir explícitamente, porque el dato no
las tiene:

1. **"Registro desde DD/MM/AAAA"** (la fecha del deploy). No hay backfill:
   no se puede inventar quién hizo qué antes de que el log existiera, y un
   vacío sin esa leyenda se lee como "no pasó nada".
2. El registro **no** incluye asistencia, créditos de recupero (que tienen
   su propia vista) ni lo que hace cada cliente por su cuenta.

`STAFF` no debe ver ni la entrada de menú: la RPC le responde
`NOT_AUTHORIZED`, así que una pantalla visible sería un botón que siempre
falla.

## Fase 31 — ciclos de cobro largos y prorrateo (ADR-0031)

Todo aditivo y compatible hacia atrás. Lo que necesita trabajo de
`frontend-engineer` está al final, marcado.

### `service-plans.ts` — `PaymentPlanOption` gana la cotización

`listPaymentPlanOptions(organizationSlug, month?)` **no cambia de firma**.
Internamente pasa de llamar `billing_period_for()` a
`quote_service_plan_period()` (que la envuelve), y la fila que devuelve gana
seis campos:

| Campo | Qué es |
|---|---|
| `billingPeriodMonths: number \| null` | Cada cuántos meses cobra el plan. `null` = 1, igual que en la base. |
| `billingAnchorMonth: number \| null` | Mes en que arranca el bloque de un ciclo calendario largo. `null` = el mes de la compra. |
| `proratedPrice: number` | Monto **sugerido** para el período. Igual a `price` salvo que el alta caiga a mitad de un ciclo calendario largo. |
| `prorated: boolean` | Si `proratedPrice` difiere de `price`. |
| `unitsCharged` / `unitsTotal: number` | La cuenta a la vista: "1 de 3 meses". |

`periodStart`/`periodEnd` siguen existiendo con el mismo significado, y
**siguen sin recortarse**: un plan trimestral jul-sep cotizado el 15 de
septiembre devuelve `2026-07-01`/`2026-09-30`, no un período recortado al
día del alta (ADR-0031 resolución 2).

El prorrateo se calcula **en SQL**, no en el browser: "un mes de un
trimestre, contando el mes de alta como completo, redondeado a la unidad
entera de moneda" es una regla de negocio, y una segunda implementación en
el cliente derivaría (CLAUDE.md). Es una **sugerencia**: el campo de monto
sigue editable y `Payment.amount` sigue libre (ADR-0024 resolución 5).

### Fase 25/38 — `registerPayment()` gana un `viewedMonth` opcional y el front recalcula el monto por días (ADR-0038)

`registerPayment(organizationSlug, customerId, prev, formData)` sigue sin
tocar su firma, pero el `FormData` admite un campo nuevo, opcional:
`viewedMonth` (`"YYYY-MM"`). Solo lo manda `RegisterPaymentForm` cuando la
pantalla está parada sobre un mes (`/payments/[customerId]?mes=`), como un
input hidden. Si el período que se acaba de guardar no se solapa con ese
mes, el mensaje de éxito lo dice explícitamente (`"... · no vas a verlo en
este mes: cubre oct–dic 2026"`) en vez del genérico "Pago registrado" —
evita que un pago de un período distinto al que se está mirando parezca
que no se guardó. `registerPayment()` sigue grabando `amount` tal cual
llega, sin recalcularlo ni validarlo contra el período.

Del lado de `RegisterPaymentForm` (sin cambio de contrato con el server,
solo de UI): para un plan de **un mes** (fuera del territorio de
`LongPeriodQuote`/ADR-0031), editar a mano "Período desde/hasta" recalcula
el monto sugerido proporcional a los días, con el período completo que el
plan sugirió al elegirlo como denominador fijo (`lib/billing-period.ts`:
`inclusiveDays()`, `round2()`). Sigue siendo una sugerencia editable —
igual que el prorrateo de ADR-0031, nunca algo que el backend imponga.

### `service-plans.ts` — `createServicePlan` acepta el ciclo largo

El `FormData` admite dos campos nuevos, los dos opcionales:

- `billingPeriodMonths` (1..12) — obligatorio en la práctica cuando
  `billingCycle` es `CALENDAR_PERIOD` o `ROLLING_PERIOD`; si no viene, la
  acción manda 1.
- `billingAnchorMonth` (1..12) — sólo para `CALENDAR_PERIOD`. Sin él, el
  ciclo arranca el mes de la compra (y entonces nunca hay nada que
  prorratear).

`billingCycle` acepta ahora cuatro valores (`CALENDAR_MONTH`,
`ROLLING_MONTH`, `CALENDAR_PERIOD`, `ROLLING_PERIOD`). Un formulario que no
mande ninguno de los campos nuevos crea exactamente el plan que creaba
antes. La acción limpia los dos campos cuando el ciclo no los admite, y
traduce los tres `CHECK` nuevos a mensajes en castellano
(`service_plans_billing_period_months_matches_cycle`,
`..._range`, `service_plans_billing_anchor_month_matches_cycle`).

`updateServicePlan` **no** los acepta: son términos congelados en la
edición, como `plan_kind`/`weekly_quota`/alcance (y además inmutables por
trigger una vez que el plan tiene pagos no-`VOID`).

### `public.ts` — `PublicServicePlan.billingCycle` se ensancha

Pasa de `"CALENDAR_MONTH" | "ROLLING_MONTH" | null` a `BillingCycle | null`:
el tipo mentía en cuanto existiera un plan de ciclo largo. El catálogo
público **todavía no trae** `billing_period_months` (`public_service_plans()`
no cambió), así que un plan trimestral se ve con su precio de lista y sin el
"cada N meses" — pendiente de un corte futuro, no de esta fase.

### `lib/plan-labels.ts` — etiquetas

`BILLING_CYCLE_LABEL` gana las dos entradas nuevas.
`planBillingLabel(billingType, billingCycle, billingPeriodMonths?)` y
`planPriceSuffix(billingType, billingPeriodMonths?)` ganan un parámetro
**opcional**: sin él dicen exactamente lo que decían antes; con él, un plan
trimestral se rotula "Cada 3 meses" y su precio " / 3 meses" en vez de " /
mes". La copy definitiva es de `frontend-engineer`.

### Pendiente de `frontend-engineer` (no toqué UI)

1. **Pantalla de planes** (`/org/[slug]/plans`): el formulario de alta gana
   "cada cuántos meses se cobra" y, cuando el ciclo es de bloque fijo del
   año, el **mes de anclaje**. Riesgo conocido (ADR-0031 riesgo 2): un
   anclaje mal elegido desalinea de golpe a todos los clientes del plan y
   después es inmutable — **mostrar los bloques resultantes antes de
   guardar** ("ene-mar, abr-jun, jul-sep, oct-dic"). Los dos campos
   desaparecen en la edición: están congelados.
2. **Formulario de registrar pago**: al elegir un plan de ciclo largo,
   mostrar el **período completo**, el **precio completo** y —si
   `prorated`— el **monto sugerido con la cuenta a la vista** ("1 de 3
   meses del trimestre jul-sep"). El campo de monto **sigue editable**: el
   prorrateo es una sugerencia, no un cobro, y un descuento de mostrador
   tiene que poder escribirse encima.
3. **Cambio de plan a mitad de período**: sigue siendo anular + recargar el
   período completo (ADR-0024 resolución 1), y **no se prorratea**.
   `quote_service_plan_period()` es función pura de (plan, fecha) y no mira
   ningún pago, así que no puede distinguir un alta de un cambio de plan:
   cuando la pantalla está recargando un período que el cliente ya tenía
   cubierto, tiene que cotizar **desde el inicio de ese período** (con lo
   cual la respuesta es el precio completo) o usar `price` en vez de
   `proratedPrice`. No hay nota de crédito por lo no consumido del plan
   anterior: si el negocio quiere reconocer algo, lo escribe a mano en el
   monto.

## Fase 32 — roles configurables por organización (ADR-0033)

Backend completo (migración `20260923190000_phase32_configurable_roles.sql`
+ paquete `@reservaste/domain` + server actions). **UI pendiente de
`frontend-engineer`** — esta sección es la especificación completa, no hace
falta volver a leer la migración.

**Regla de oro de esta feature**: la UI **esconde y deshabilita**, nunca
autoriza. La autorización está en RLS, en 17 RPCs y en un trigger. Pero
tampoco vale mostrar todo y fallar al guardar: cada pantalla recibe los
permisos resueltos antes de renderizar, así que puede no ofrecer lo que la
base va a rechazar.

### Cambio de contrato 1 — `requireOrganizationMembership(slug)`

`frontend/app/actions/organizations.ts`. El shape de retorno pasa de
`{ organization, membership }` a:

```ts
{
  organization: Organization,
  membership: OrganizationMember,   // ahora con `roleId: string | null`
  roleName: string | null,          // nombre del rol efectivo; null para un OWNER
  permissions: {
    canViewPayments: boolean,
    canManagePayments: boolean,
    canManageBookings: boolean,
    canManageCustomers: boolean,
    canManageAttendance: boolean,
  },
}
```

Aditivo: ninguna pantalla existente se rompe. `permissions` viene de la RPC
`my_organization_permissions()`, con los cinco booleanos **ya resueltos**
(rol propio, o rol por defecto, o todo en `true` si es `OWNER`). Si la RPC
fallara se devuelve `NO_ORG_PERMISSIONS` (todo en `false`): se falla cerrado.

Para leer un permiso usar el helper del paquete de dominio, no el booleano
suelto:

```ts
import { hasOrgPermission } from "@reservaste/domain";
if (hasOrgPermission(permissions, "VIEW_PAYMENTS")) { /* ... */ }
```

### Cambio de contrato 2 — `getTeam(slug)` → `TeamMember`

`frontend/app/actions/admin.ts`. Dos campos nuevos:

```ts
interface TeamMember {
  memberId: string; profileId: string; fullName: string;
  role: "OWNER" | "STAFF"; isActive: boolean;
  roleId: string | null;    // rol EFECTIVO (el asignado, o el por defecto)
  roleName: string | null;  // null para un OWNER
}
```

`roleId` es el **efectivo**, no el asignado: si el miembro no tiene rol
propio, viene el id del rol por defecto. Eso es justo lo que el `<select>`
de la pantalla de equipo tiene que mostrar preseleccionado.

### Cambio de contrato 3 — `inviteMember` acepta `roleId`

`frontend/app/actions/admin.ts`. El `FormData` admite un campo opcional
`roleId`. Vacío = rol por defecto (el comportamiento de antes). Si
`role === "OWNER"` el `roleId` se ignora: un `OWNER` no lleva rol.

### Acciones nuevas — `frontend/app/actions/roles.ts`

Todas **OWNER-only** (chequeado también en la base):

| Acción | Firma | Notas |
|---|---|---|
| `getOrganizationRoles(slug)` | `→ OrganizationRole[]` | lectura para cualquier miembro; ordenado con el default primero |
| `createOrganizationRole(slug, prev, formData)` | `→ ActionState` | campos: `name`, y checkboxes `canViewPayments`, `canManagePayments`, `canManageBookings`, `canManageCustomers`, `canManageAttendance` |
| `updateOrganizationRole(slug, prev, formData)` | `→ ActionState` | campos: `roleId`, `name` y los cinco checkboxes. **Un checkbox ausente significa "apagado"**, no "no lo toques" |
| `setOrganizationRoleDefault(slug, roleId)` | `→ ActionState` | mueve el default; atómico (una RPC, no dos updates) |
| `deactivateOrganizationRole(slug, roleId)` | `→ ActionState` | falla con `ROLE_IN_USE` si todavía lo tiene alguien |
| `setMemberRole(slug, prev, formData)` | `→ ActionState` | campos: `memberId`, `roleId` (vacío = volver al rol por defecto) |

`ActionState` es el `{ error, success }` de siempre, y los errores ya vienen
traducidos a español por `describeRoleError()`.

### Pantalla de roles — `/org/[slug]/team`

Va **en la misma pantalla de Equipo**, no en una ruta nueva: la pregunta
"¿qué puede esta persona?" y "¿qué puede este rol?" se responden juntas, y
`/team` ya es OWNER-gated para escribir.

**Sección 1 — Roles** (visible para cualquier miembro, editable sólo por el
`OWNER`):

- Lista de `getOrganizationRoles(slug)`. Por rol: nombre, badge "Por
  defecto" en el que lo sea, y un resumen legible de los permisos (algo como
  "Pagos: ver y registrar · Reservas · Clientes · Asistencia"). No mostrar
  cinco íconos crudos sin texto: el dueño tiene que poder leer de un vistazo
  qué firmó.
- Formulario de alta con el nombre libre y los cinco checkboxes, **los cinco
  tildados por defecto** (así crear un rol sin pensarlo reproduce el
  comportamiento actual y nunca deja a alguien sin poder trabajar).
- `MANAGE_PAYMENTS` implica `VIEW_PAYMENTS`: al tildar "registrar pagos",
  tildar y **deshabilitar** "ver pagos". Si se destilda "ver pagos",
  destildar "registrar pagos". Hay `CHECK` en la base y `refine` en Zod, pero
  el formulario no debería poder llegar a ese estado.
- Acciones por rol: editar, "usar como rol por defecto"
  (`setOrganizationRoleDefault`) y desactivar
  (`deactivateOrganizationRole`). **Nunca ofrecer desactivar el rol por
  defecto ni un rol con miembros** — se puede saber sin pedir nada al
  backend: el default trae `isDefault: true`, y los miembros de cada rol
  salen de contar `team.filter(m => m.roleId === role.id)`.
- No hay tope de roles (ADR-0033 resolución 5).

**Sección 2 — Equipo** (la lista que ya existe):

- Cada fila muestra hoy un `StatusBadge` con `roleLabel(member.role)`. Para
  un `STAFF`, mostrar además (o en su lugar) el **nombre del rol**
  (`member.roleName`), que es el dato que le importa al dueño — "Profesor"
  dice más que "Equipo".
- Para un `OWNER`: **no** ofrecer selector de rol, y decir por qué en una
  línea ("El dueño siempre puede todo"). La RPC responde
  `OWNER_HAS_NO_ROLE` y hay un `CHECK` en la base, así que ofrecerlo sería
  ofrecer un error.
- Para un `STAFF` activo y si quien mira es `OWNER`: un `<select>` con los
  roles activos + `setMemberRole`, preseleccionado en `member.roleId`.
- El formulario de invitar (`InviteForm`) gana el mismo `<select>` de rol,
  con `name="roleId"`, deshabilitado cuando se elige `OWNER`.

### Esconder según permiso — el mapa completo

| Permiso ausente | Qué hay que esconder / deshabilitar |
|---|---|
| `canViewPayments` | El ítem "Pagos" de la navegación y toda `/org/[slug]/payments/*`; el bloque de deuda/estado de pago en la ficha de cliente; la cola de solicitudes de cambio de plan. La pantalla de planes (`/org/[slug]/plans`) **no** se esconde: es la lista de precios, que es pública |
| `canManagePayments` | Botón "Registrar pago", anular pago (`VOID`), y resolver una solicitud de cambio de plan. La lectura queda |
| `canManageBookings` | "Anotar cliente" en la agenda, cancelar la reserva **de otra persona**, cancelar una ocurrencia, crear/cancelar un horario fijo. Ver la agenda y la lista de anotados **no** se esconde |
| `canManageCustomers` | "Nuevo cliente", "Cliente sin cuenta", enrolar por email, dar de baja, y emitir/revocar el link de activación de WhatsApp. El padrón se sigue viendo |
| `canManageAttendance` | Los controles de presente/ausente. El resumen de asistencia se sigue viendo |

No hay permiso para "ver el calendario", "ver la agenda" ni "ver el padrón":
son de todo miembro activo a propósito (un rol que no ve a los clientes no
puede pasar lista).

**Fuga aceptada, no la esconda**: un rol sin `canViewPayments` sigue viendo
la señal `upcoming_unpaid` en el horario fijo y el motivo
`PAYMENT_REQUIRED` cuando no puede anotar a alguien (ADR-0033 resolución 2).
Es deliberado y está documentado en `security.md`: sin ese dato el rol no
entiende por qué el sistema lo rechaza. Redactarlo en términos de acción
("no se puede anotar: falta el pago del período") y no de monto.

### Lo que sigue siendo OWNER-only (y ahora también en la base)

Planes y precios (`/org/[slug]/plans`): los gates
`membership.role !== "OWNER"` de `service-plans.ts` **se quedan** — ahora
están respaldados por policies y un trigger, así que ya no son la única
defensa, pero siguen siendo el que da el mensaje decente. Igual
configuración/branding, invitar y revocar equipo, administrar roles,
créditos manuales de cortesía, merge/unlink de clientes.

### Errores nuevos ya traducidos

`ROLE_NAME_TAKEN`, `ROLE_NAME_TOO_LONG`, `ROLE_NAME_REQUIRED`,
`MANAGE_PAYMENTS_REQUIRES_VIEW`, `ROLE_IN_USE`, `DEFAULT_ROLE_REQUIRED`,
`ROLE_INACTIVE`, `ROLE_OTHER_ORGANIZATION`, `OWNER_HAS_NO_ROLE`,
`ROLE_NOT_FOUND`, `MEMBER_NOT_FOUND`. `NOT_AUTHORIZED` ya estaba y sigue
mapeando a "No tenés permiso para hacer esto" — es lo que devuelve cada RPC
cuando el permiso falta, así que **no hace falta un código nuevo por
permiso**.

### Tipos del paquete de dominio

`OrganizationRole`, `OrgPermission`, `OrganizationPermissions`,
`MyOrganizationPermissions`, `OrganizationTeamMember`, los mappers
(`mapOrganizationRole`, `mapMyOrganizationPermissions`,
`mapOrganizationTeamMember`), los schemas Zod
(`createOrganizationRoleSchema`, `updateOrganizationRoleSchema`,
`setMemberRoleSchema`) y los helpers `hasOrgPermission`,
`ALL_ORG_PERMISSIONS`, `NO_ORG_PERMISSIONS`, `ORG_PERMISSION_KEYS`.
**No duplicar nada de esto del lado del frontend.**

Requiere que `frontend/package.json` apunte a una versión de
`@reservaste/domain` que incluya estos tipos (commit de `backend#main`
posterior a la Fase 32) — orden de publicación en `CHARTER.md`.

## Fase 33 — invitaciones de equipo sin registro previo (ADR-0034)

Backend completo: migración `20260923200000_phase33_team_invitations.sql` +
tipos/schemas/mappers en `@reservaste/domain`. **Server actions y UI pendientes
— esta sección es la especificación completa, no hace falta volver a leer la
migración.** Por consigna de esta ronda no se tocó nada de `frontend/`, ni
siquiera `app/actions/`.

El problema que resuelve: hoy `inviteMember` traduce `PROFILE_NOT_FOUND` a "No
existe una cuenta con ese email. Si todavía no se registró..." — o sea, el
dueño no puede dar de alta a un profesor hasta que el profesor se registre
solo. Es exactamente el problema que ADR-0026 resolvió para clientes.

**`inviteMember` no cambia.** Sigue siendo el camino rápido para quien ya tiene
cuenta (ADR-0034 resolución 1), elegido **explícitamente** por el dueño. No
hacerlo automático es deliberado: elegir el camino según si el email ya tiene
cuenta reintroduciría el oráculo de "¿tal email está registrado en la
plataforma?".

### RPCs nuevas (contrato con la base)

| RPC | Gate | Parámetros | Devuelve |
|---|---|---|---|
| `issue_team_invitation` | **OWNER** | `p_organization_id`, `p_email`, `p_display_name`, `p_phone`, `p_role_id` | `setof (invitation_id uuid, token text, expires_at timestamptz)` — **una fila** |
| `revoke_team_invitation` | **OWNER** | `p_invitation_id` | `void`, idempotente |
| `organization_team_invitations` | **OWNER** | `p_organization_id`, `p_include_history boolean default false` | `setof` (ver abajo) |
| `claim_team_invitation` | `authenticated` | `p_token` | `jsonb`: `{ status, organization_slug, organization_name, member_id }` |

`organization_team_invitations()` devuelve, por fila: `invitation_id`, `email`,
`display_name`, `phone`, `role_id`, `role_name` (el rol **efectivo**),
`status` (`PENDING | REDEEMED | REVOKED | EXPIRED`, derivado), `created_at`,
`expires_at`, `redeemed_at`, `redeemed_profile_id`, `revoked_at`, `created_by`.
**Nunca `token_hash`.** Para un no-`OWNER` devuelve **cero filas**, no un error.

`issue_team_invitation()` devuelve el token claro **exactamente una vez**. No se
puede recuperar (en la base sólo vive su `sha256`): si se pierde antes de mandar
el WhatsApp, hay que **reemitir** — lo que revoca el anterior.

### Server actions a escribir — `frontend/app/actions/team-invitations.ts`

Archivo nuevo, no dentro de `admin.ts`: `admin.ts` ya pasa las 700 líneas y este
es un flujo con su propio vocabulario de errores. Todas `OWNER`-only (y
chequeado en la base, así que el gate en TS es el que da el mensaje decente, no
la defensa).

```ts
export interface IssuedTeamInvitation {
  invitationId: string;
  expiresAt: string;
  /**
   * Deep link de api.whatsapp.com ya armado en el servidor (nombre de la
   * organización y siteUrl() nunca desde el cliente). El token existe en esta
   * URL exactamente una vez: no se devuelve por separado, no se persiste, no
   * se loguea.
   */
  whatsappUrl: string;
  /** El link pelado, para "copiar link" cuando no hay teléfono. */
  invitationUrl: string;
}

// Campos del FormData: email, displayName, phone (opcional), roleId (vacío = rol por defecto)
export async function issueTeamInvitation(
  organizationSlug: string,
  prev: ActionState,
  formData: FormData,
): Promise<ActionState & { invitation: IssuedTeamInvitation | null }>;

export async function revokeTeamInvitation(
  organizationSlug: string,
  invitationId: string,
): Promise<ActionState>;

export async function getTeamInvitations(
  organizationSlug: string,
  includeHistory?: boolean,
): Promise<OrganizationTeamInvitation[]>;
```

Notas de implementación que no son opcionales:

- Validar con `inviteTeamMemberSchema` de `@reservaste/domain` **antes** de la
  RPC. El schema ya normaliza el email a minúsculas, igual que la RPC: si el
  borde y la base normalizaran distinto, "reenviar" crearía una segunda
  invitación viva en vez de reemplazar la primera.
- `invitationUrl = `${siteUrl()}/equipo/${token}`` y el mensaje de WhatsApp
  armado con un helper propio, **`buildWhatsAppTeamInvitationLink()`** en
  `frontend/lib/whatsapp-team-invitation-link.ts`. No reusar
  `buildWhatsAppActivationLink()`: el texto del mensaje es otro ("te invita a
  sumarse al equipo", no "a activar tu cuenta para gestionar tus reservas") y
  el path es otro. Igual que el de clientes, **el mensaje nombra a la
  organización y nunca a la persona**: mandarlo a un número mal tipeado no
  puede filtrar el nombre de un tercero.
- El token **no** se devuelve como campo suelto ni se guarda en estado de
  cliente. Sólo viaja dentro de `whatsappUrl`/`invitationUrl`.
- `revalidatePath(`/org/${slug}/team`)` después de emitir y de revocar.
- Si no hay `phone`, `whatsappUrl` va `null`/vacío y la UI ofrece sólo "copiar
  link". Una invitación sin teléfono es válida.

### El canje — `frontend/app/actions/team-invitations.ts` + ruta propia

**Cookie y ruta propias, no las de `/activar`.** Alguien puede ser cliente del
negocio **y** haber sido invitado al equipo (la recepcionista que además
entrena ahí es el caso normal, no el raro): si el flujo de equipo reusara
`/activar/[token]` o la cookie `activation_token`, un token pisaría al otro y la
persona perdería uno de los dos sin forma de recuperarlo (el claro no existe en
la base).

| Pieza | Clientes (ADR-0026) | Equipo (ADR-0034) |
|---|---|---|
| Ruta que recibe el link | `/activar/[token]/route.ts` | `/equipo/[token]/route.ts` |
| Pantalla de confirmación | `/activar/continuar` | `/equipo/continuar` |
| Cookie | `activation_token`, `path: "/activar"` | `team_invitation_token`, `path: "/equipo"` |
| TTL de la cookie | 72 h | **24 h** |
| Helper | `lib/activation-cookie.ts` | `lib/team-invitation-cookie.ts` (nuevo) |

La ruta `/equipo/[token]/route.ts` es un calco del handler de clientes, y por
las mismas razones (que están escritas en el archivo existente y siguen
valiendo):

- `Cache-Control: no-store`, `Referrer-Policy: no-referrer`,
  `X-Robots-Tag: noindex`, cookie `httpOnly` + `Secure` + `sameSite: "lax"`.
- **El GET no toca la base**: WhatsApp (y todo cliente de chat) hace fetch del
  link para armar la tarjeta de preview antes de que la persona lo toque, así
  que cualquier cosa que se consumiera acá la consumiría un crawler y el primer
  click humano encontraría el link usado.
- **`maxAge` de la cookie = TTL del token (24 h), no unos minutos.** Ésta es la
  lección que ya rompió el flujo de clientes en producción (ver el hotfix de
  2026-09-23 más arriba): la cookie es la **única copia** del token que tiene la
  persona, y el camino normal es link → `/login` → `/signup` → confirmar email →
  volver, que tarda más que unos minutos. La invariante es *la cookie nunca
  puede vencer antes que el token*, y `TEAM_INVITATION_TOKEN_TTL_SECONDS` tiene
  que espejar el `interval '24 hours'` de `issue_team_invitation()`
  (`backend/test/phase33.team-invitations.test.ts` fija el lado SQL, así que los
  dos no pueden divergir en silencio).
- Al limpiar la cookie, **repetir el `path`**: `jar.delete(name)` apunta a la
  cookie de path `/`, que es otra cookie distinta.

```ts
// Server component helper -- desde el hardening de 2026-09-24 vive en
// `frontend/lib/server-cookies.ts` (`import "server-only"`), NO en el módulo
// "use server" de las actions: ver "Hardening 2026-09-24" al final.
export async function readTeamInvitationToken(): Promise<string | null>;

/** El canje. La RPC recibe SÓLO el token, leído de la cookie httpOnly. */
export async function claimTeamInvitation(): Promise<{ error: string | null }>;
```

En éxito: limpiar la cookie y redirigir a `/org/${organization_slug}` (el panel
del negocio al que se acaba de sumar), no a `/dashboard`. La persona hizo click
para entrar a **ese** negocio.

Sin cookie, **no decir "venció"** — mismo error que el hotfix de clientes tuvo
que arreglar. El texto correcto es: *"No encontramos la invitación en este
navegador. Volvé a abrir el link desde este mismo teléfono y seguí desde ahí."*

### Errores a traducir

| Código | Mensaje sugerido |
|---|---|
| `NOT_AUTHORIZED` | "No tenés permiso para hacer esto" (ya existe) |
| `SUBSCRIPTION_INACTIVE` | "La suscripción de la organización está suspendida" (ya existe) |
| `EMAIL_REQUIRED` / `INVALID_EMAIL` | "Revisá el email: es el que la persona va a tener que usar para entrar" |
| `INVALID_PHONE` | "Revisá el teléfono: tiene que incluir el código de país (ej: +598 99 123 456)" |
| `ROLE_NOT_FOUND` / `ROLE_OTHER_ORGANIZATION` | "Ese rol ya no está disponible. Elegí otro" (ya existen) |
| `RATE_LIMITED_HOURLY` / `RATE_LIMITED_DAILY` | "Mandaste muchas invitaciones seguidas. Probá de nuevo en un rato" |
| `PLAN_LIMIT_REACHED` | "Tu plan no tiene más lugares de equipo. Contá también las invitaciones pendientes: revocá una o cambiá de plan" — **el mensaje tiene que mencionar las pendientes**, porque es la diferencia con el error que ya existía |
| `INVALID_TOKEN` | "Este link no es válido. Pedile al negocio que te lo reenvíe" |
| `INVITATION_REVOKED` | "Este link fue desactivado. Pedile al negocio uno nuevo" |
| `INVITATION_EXPIRED` | "Este link venció (duran 24 horas). Pedile al negocio que te lo reenvíe" |
| `ALREADY_REDEEMED` | "Este link ya fue usado por otra cuenta. Pedile al negocio uno nuevo" |
| `INVITE_WRONG_EMAIL` | **El mensaje más importante del flujo.** "Esta invitación es para otro email. Entrá con la casilla a la que te la mandaron, o pedile al negocio que te la reenvíe a `<el email con el que estás>`." Hay que decir **con qué email está la sesión**, o la persona no tiene forma de entender qué salió mal (el caso típico: se registró con Google y su cuenta de Google es otra). |
| `INVITATION_ROLE_UNAVAILABLE` | "El rol de esta invitación ya no existe. Pedile al negocio que te la reenvíe" |
| `ORGANIZATION_UNAVAILABLE` | "Este negocio no está disponible en este momento" |
| `AUTH_REQUIRED` | "Iniciá sesión para continuar" (ya existe) |

### UI pendiente de `frontend-engineer`

**En `/org/[slug]/team`**, junto a la sección de Roles de la Fase 32. Todo
`OWNER`-only.

**Sección "Invitar al equipo" — dos caminos, elegidos por el dueño:**

1. *"Ya tiene cuenta"* → el `InviteForm` que ya existe (`inviteMember`).
2. *"Todavía no tiene cuenta"* → formulario nuevo: **nombre + teléfono +
   email** + el mismo `<select>` de rol de la Fase 32. Lo que el usuario pidió
   textualmente es el alta por nombre y teléfono; el email va igual y **no es
   opcional**, porque es lo único que el canje puede verificar — decirlo en el
   campo ("con este email va a tener que entrar") en vez de dejarlo como un dato
   burocrático más.

No unificar los dos en un solo formulario que decida solo: eso reintroduce el
oráculo de emails (ADR-0034 §5.6) y además borra la diferencia real entre "ya
está adentro" y "le mandé un link que puede no llegar".

**Al emitir**, mostrar el resultado en el acto y **una sola vez**:

- Botón primario "Enviar por WhatsApp" → `invitation.whatsappUrl`
  (`target="_blank"`).
- Botón secundario "Copiar link" → `invitation.invitationUrl`.
- Aviso explícito: *"Este link se muestra una sola vez y vence en 24 horas. Si
  lo perdés, reenvialo."* No es decoración: el token no se puede recuperar.
- **Nunca** renderizar el token como texto suelto ni ponerlo en un `input`
  visible, ni loguearlo.

**Lista de invitaciones** (`getTeamInvitations`), separada de la lista de
miembros — una invitación pendiente **no es** un miembro y ponerlas juntas
miente sobre el padrón. Por fila: nombre (o el email si no hay nombre), email,
teléfono, rol, y un badge por `status`:

| `status` | Badge | Acciones |
|---|---|---|
| `PENDING` | "Enviada" + "vence en N h" | Reenviar, Revocar |
| `EXPIRED` | "Vencida" (warning, no error: nadie hizo nada mal) | Reenviar, Revocar |
| `REVOKED` | "Cancelada" (sólo con historial) | — |
| `REDEEMED` | "Activada" + fecha (sólo con historial) | — |

- **"Reenviar" es emitir de nuevo con los mismos datos**, no un "resend" que
  recicle el link: el token anterior se revoca y sale uno nuevo. Decirlo en el
  botón o en un tooltip ("el link anterior deja de funcionar").
- Un toggle "ver historial" que pasa `includeHistory: true`.
- Estado vacío con el porqué: "Todavía no invitaste a nadie que no tenga
  cuenta."

**Cuando el plan está lleno**: deshabilitar el formulario de invitar y decir
cuántos lugares hay y cuántos están tomados **contando las pendientes**. El
dato ya está: miembros activos de `getTeam()` + invitaciones con `status` en
`PENDING` (las vencidas no cuentan, igual que en la base). Es el error más
confuso de la feature si se lo deja aparecer recién al apretar el botón.

### Tipos del paquete de dominio

`TeamInvitation`, `TeamInvitationStatus`, `OrganizationTeamInvitation`, los
mappers `mapTeamInvitation` / `mapOrganizationTeamInvitation` (con sus filas
`TeamInvitationRow` / `OrganizationTeamInvitationRow`) y los schemas Zod
`inviteTeamMemberSchema`, `revokeTeamInvitationSchema`,
`claimTeamInvitationSchema`, `teamInvitationEmailSchema`,
`teamInvitationPhoneSchema`. **No duplicar nada de esto del lado del
frontend.**

Requiere que `frontend/package.json` apunte a una versión de
`@reservaste/domain` que incluya estos tipos (commit de `backend#main`
posterior a la Fase 33) — orden de publicación en `CHARTER.md`.

## Fase 30b — `organization_audit_log()`: cursor compuesto (CAMBIO DE CONTRATO)

Migración `20260924100000_phase30b_audit_log_composite_cursor.sql`. Hallazgo de
`qa-engineer`: el corte `created_at < p_before` perdía filas. `audit_log.created_at`
es `default now()` (inicio de la transacción), y `cancel_slot_occurrence()` /
`discontinue_schedule_rule()` cancelan N reservas en un solo `UPDATE`: N filas con
el **mismo timestamp exacto**. Si el límite de página caía dentro de ese grupo, el
resto del grupo no aparecía en ninguna página, sin error.

**Firma nueva** (la vieja se dropea; no queda como overload):

```sql
organization_audit_log(
  p_organization_id   uuid,
  p_limit             int         default 100,   -- acotado 1..200
  p_before_created_at timestamptz default null,  -- created_at de la ÚLTIMA fila de la página anterior
  p_before_id         uuid        default null   -- id de esa misma fila
)
```

- Keyset por la clave total: `where (created_at, id) < (p_before_created_at, p_before_id)
  order by created_at desc, id desc`. Cada fila cae en exactamente una página.
- **Los dos o ninguno.** Uno solo → `INVALID_CURSOR` (con sólo la fecha volvería el
  bug). Primera página: los dos `null`.
- Shape de respuesta **sin cambios** (`AuditLogEntry`). Tipo nuevo en
  `@reservaste/domain`: `AuditLogCursor { beforeCreatedAt: string; beforeId: string }`.
- **El cursor se pasa tal cual vino de la RPC (string).** `created_at` tiene
  microsegundos; pasarlo por `new Date()` / `Date.parse` / `toISOString()` lo trunca
  a milisegundos, el cursor queda antes del grupo real y se vuelven a perder filas.

### Qué tiene que cambiar en `frontend/` (pendiente de `frontend-engineer`)

La base ya tiene la firma nueva: **hasta que esto se haga, `/org/[slug]/settings/registro`
muestra "No pudimos cargar el registro"** (PostgREST no encuentra una función con
`p_before`). Deploy coordinado: backend y frontend juntos.

1. `frontend/app/actions/audit.ts` — `getOrganizationAuditLog`:
   ```ts
   export async function getOrganizationAuditLog(
     organizationSlug: string,
     options?: { limit?: number; cursor?: AuditLogCursor },   // AuditLogCursor de @reservaste/domain
   ): Promise<AuditLogResult>
   // ...
   supabase.rpc("organization_audit_log", {
     p_organization_id: organization.id,
     p_limit: options?.limit ?? 50,
     p_before_created_at: options?.cursor?.beforeCreatedAt ?? null,
     p_before_id: options?.cursor?.beforeId ?? null,
   });
   ```
   Traducir `INVALID_CURSOR` a volver a la primera página (o a un mensaje neutro),
   no a "sólo el dueño puede ver".
2. `frontend/app/org/[slug]/settings/registro/page.tsx` — hoy lee `?antes=<fecha>` y lo
   valida con `Date.parse`. Pasa a necesitar **dos** parámetros (p. ej.
   `?antes=<createdAt>&antesId=<id>`), armados con `last.createdAt` y `last.id` **sin
   transformar** (`encodeURIComponent` sí, `Date` no). Si falta uno, o el id no es un
   uuid, se ignora el cursor y se muestra la primera página. `Date.parse` puede quedar
   solo como validación, nunca para reconstruir el valor.
3. Bump de `@reservaste/domain` a un commit de `backend` que incluya `AuditLogCursor`
   (orden de publicación de `CHARTER.md`).

## Fase 27b — slugs reservados

Migración `20260924100100_phase27b_reserved_organization_slugs.sql`.
`create_organization_with_owner()` mantiene su firma; **código de error nuevo**:
`SLUG_RESERVED` cuando el slug es un segmento top-level de `frontend/app/` (`_next`,
`activar`, `admin`, `api`, `auth`, `contacto`, `dashboard`, `equipo`, `login`, `me`,
`onboarding`, `org`, `signup`). `INVALID_SLUG` sigue siendo el de formato.

`createOrganization` (`frontend/app/actions/organizations.ts`) ahora traduce
`SLUG_RESERVED` → "Ese nombre está reservado, elegí otro" e `INVALID_SLUG` → el mismo
texto de formato que Zod (antes los dos caían en "No se pudo crear la organización").
Firma y shape de la acción sin cambios. `organizationSlugSchema` (Zod) rechaza la misma
lista (`RESERVED_ORGANIZATION_SLUGS`, exportada), así que con el paquete de dominio
actualizado el formulario lo corta antes de llegar a la base.

**Regla de mantenimiento:** una ruta top-level nueva en `frontend/app/` = una migración
que la agrega al `CHECK` + el valor en `RESERVED_ORGANIZATION_SLUGS`, **antes** de
publicar la ruta.

## Hardening 2026-09-24 — lectores de cookie fuera de módulos `"use server"`

`readActivationToken()` (antes en `app/actions/activation.ts`) y
`readTeamInvitationToken()` (antes en `app/actions/team-invitations.ts`) se mudaron a
`frontend/lib/server-cookies.ts` (sin `"use server"`, con `import "server-only"`;
dependencia `server-only` agregada a `frontend/package.json`). Misma firma
(`() => Promise<string | null>`). Motivo: todo export de un módulo `"use server"` es una
server action invocable por `POST` — una que devuelve el valor de una cookie `httpOnly`
deshace el `httpOnly`. **Regla:** un helper que devuelve un secreto nunca vive en un
módulo `"use server"`. Consumidores actualizados: `app/activar/continuar/page.tsx` y
`app/equipo/continuar/page.tsx` (sólo el import).
