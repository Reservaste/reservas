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
