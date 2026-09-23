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
