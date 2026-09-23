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
