# Autenticación, autorización y seguridad

Mantenido por: `auth-security-agent`. Aprobado por: Orchestrator.
Estado: **Phase 0 completa** (ver ADR-0005, ADR-0006, ADR-0008, ADR-0015).
Implementación concreta pendiente de Phase 1.

## Métodos de autenticación

- Email/password.
- Google OAuth.
- Recuperación de contraseña.

## Roles

- **OWNER** — dueño de la `Organization`. Acceso administrativo completo
  dentro de su tenant.
- **STAFF** — miembro operativo de la `Organization` (vía
  `OrganizationMember`). Acceso administrativo acotado según lo que se
  defina en Phase 1.
- **CUSTOMER** — cliente de una o más `Organization` (vía `Customer`).
  Acceso solo a su propia información y a la reserva pública.

## Público vs. privado — regla crítica

El **calendario es público**. Un usuario anónimo puede:

- Ver servicios de una `Organization`.
- Ver fechas y horarios disponibles.
- Ver capacidad/disponibilidad agregada (p. ej. "4 disponibles de 12").

Un usuario anónimo **no puede**:

- Ver clientes (nombres, emails).
- Ver bookings individuales de otra persona.
- Ver pagos.
- Reservar.

**Disclosure de disponibilidad — `publicAvailabilityDisplay` (ADR-0008,
reemplaza el umbral automático de ADR-0007):** "disponibilidad agregada"
no es automáticamente segura — en capacidades bajas (turnos individuales:
consultorio, sesión 1-a-1) un conteo exacto revela con certeza la
existencia de una reserva puntual. En vez de un umbral automático por
capacidad, `Organization`/`Service` **eligen explícitamente** el modo:
`EXACT` (conteo real), `LIMITED` (3 estados: disponible/últimos
lugares/completo) o `BOOLEAN` (2 estados: disponible/sin disponibilidad).
`Organization` define el default, `Service` puede hacer override (ej. un
gimnasio en `EXACT`, un servicio de psicólogo en `BOOLEAN`). El backend
**nunca** manda el conteo exacto (ni `capacity`) salvo en modo `EXACT` —
los campos numéricos se omiten del payload, no se mandan en `null`. La
regla aplica a todo endpoint público que exponga capacidad de un slot,
listado o detalle puntual. Rate limiting en el endpoint público de
disponibilidad para mitigar inferencia por polling repetido. Detalle
completo (shape de respuesta, umbral interno de `LIMITED`) en ADR-0008.

## Para reservar

```
authenticated
AND entitlement válido (ServiceEntitlement activo y vigente)
AND payment válido si el servicio lo requiere
AND cupo disponible
AND no duplicate booking
```

Estas cinco condiciones se validan **en el backend**, siempre. El frontend
puede reflejar el estado para UX, pero nunca es la fuente de verdad —
`payments-entitlements-agent` centraliza esta lógica en `canCustomerBook()`
para que no se duplique ni diverja entre endpoints.

**Contrato de `canCustomerBook()` (ADR-0005 — prevención de IDOR):** la
función recibe `(profileId, organizationId, serviceId, slotOccurrenceId)`
— **nunca** un `customerId` pasado por el caller como si fuera de
confianza — y resuelve/valida el `Customer` internamente a partir de
`profileId` + `organizationId`. Valida explícitamente que
`customer.organizationId == service.organizationId ==
slotOccurrence.organizationId` antes de evaluar el resto. Su resultado
nunca se reutiliza entre requests: se re-ejecuta inmediatamente antes de
cada intento de escritura, y el insert final siempre pasa por la RPC de
ADR-0004 (que es la que da la garantía de atomicidad — `true` acá no
reemplaza esa protección). Cuando un `STAFF`/`OWNER` reserva en nombre de
un customer, la capa de API valida membership del staff sobre la
organización (no ownership por `profileId`) antes de invocar la función.

## Multi-tenancy

- Todo acceso a datos de negocio se filtra por `organizationId`.
- RLS en base de datos como última línea de defensa (coordinado con
  `database-agent`), no solo filtros en la capa de aplicación.
- Ningún endpoint puede inferir el tenant desde un valor que el cliente
  controle sin validar membership (evitar IDOR: un `OrganizationMember`
  de la Org A no debe poder leer/escribir datos de la Org B nunca, ni
  por error de query, ni por parámetro manipulado).

**RLS de dos capas para tablas de customer (ADR-0006):** filtrar solo por
`organizationId` no alcanza para `Booking`, `Payment`,
`ServiceEntitlement` y `Customer`, porque un mismo `Profile` puede ser
`Customer` de varias `Organization` a la vez — un filtro de solo-tenant
dejaría que cualquier `CUSTOMER` de una organización viera los datos de
**todos** los customers de esa misma organización. Política de dos
condiciones combinadas con `OR`:
- **CUSTOMER**: acceso donde existe un `Customer` con
  `profileId = auth.uid()` dueño de la fila (+ chequeo redundante de
  `organizationId` como defensa en profundidad).
- **OWNER/STAFF**: acceso donde existe un `OrganizationMember` con ese
  `profileId` sobre el `organizationId` de la fila.

No usar un claim de "organización actual de sesión" fijo para el rol
`CUSTOMER` — cada request valida contra la organización del recurso
puntual, no contra un tenant único de sesión.

## Checklist de revisión de seguridad (para cada feature)

- ¿Se puede acceder a este dato/acción sin pertenecer al tenant correcto?
- ¿Se puede acceder a este dato sin el rol adecuado?
- ¿Hay algún ID (booking, customer, payment) que se acepte del cliente sin
  verificar ownership/membership server-side?
- ¿La validación de negocio (cupo, entitlement, pago) se repite en backend
  o se confía en lo que mandó el frontend?
- ¿Esta ruta/acción debería ser pública, de customer autenticado, o de
  admin — y coincide con lo que realmente hace?

## Booking intent a través del login (ADR-0015)

Un usuario anónimo que arma un intent de reserva (organización, servicio,
fecha, slot) y va a login: el intent viaja en **query params sin firmar**
en la URL de retorno (no son datos de autorización, son los mismos datos
ya públicos — el servidor los revalida enteros en el submit, nunca los
toma como verdad). `returnTo` se valida contra un **allowlist
server-side** (path relativo, sin esquema/host) para evitar open
redirect. Al volver, la UI exige **confirmación explícita** del usuario
(mitiga un vector de confused deputy: alguien manda un link con un
intent que la víctima no eligió) antes de invocar el submit — nunca hay
reserva automática post-login. El submit final es siempre el mismo
`POST /bookings` de cualquier reserva normal, sin code path especial.
Ninguna escritura server-side (`Booking`/`Customer` en draft) se crea
para un usuario anónimo. Detalle completo en ADR-0015.

## Cliente gestionado + activación por WhatsApp (ADR-0026, Fase 21)

Migración: `20260922190000_phase21_managed_customers.sql`.

**El modelo.** `customers.profile_id` es nullable. Un `Customer` con
`profile_id is null` es un **cliente gestionado**: existe, se le agenda y
se le cobra, pero no tiene sesión y no puede ver nada — todas las policies
RLS de dos capas (ADR-0006) que comparan `profile_id = auth.uid()`
resuelven `NULL` para esa fila, que RLS trata como no visible, así que un
gestionado nunca aparece en `my_bookings()`/`my_payments()`/etc. de nadie.
`display_name` + `phone` (E.164) cubren su identidad visible para el
mostrador mientras no tiene `profile_id`; `claimed_at` marca cuándo pasó a
estar activado. `profile_id` es `on delete set null` (no `cascade`):
borrar la cuenta de alguien lo convierte en un cliente gestionado con su
historial intacto, nunca borra sus reservas/pagos.

**Regla transversal para todo código nuevo con `profile_id` nullable**:
en una policy RLS, `NULL` deniega (correcto por default). **Dentro de
`plpgsql`, un `if` con una expresión que da `NULL` no entra al `raise`** —
`profile_id <> auth.uid() and not es_miembro` da `NULL` cuando
`profile_id is null`, y salta la autorización. Escribir siempre
`if <es el dueño> elsif <es miembro> else raise`, nunca la forma negada.
(`cancel_booking()` tenía este bug exacto; se cerró en la migración 19,
antes de que esta ADR hiciera la columna nullable de verdad — el orden
importaba.)

**El token (`customer_activations`).** 256 bits de
`extensions.gen_random_bytes()` (nunca `random()` — no es un PRNG
criptográfico), guardado como `sha256(token)` en `token_hash`, nunca en
claro. El token en claro existe una sola vez, en el valor de retorno de
`issue_customer_activation()`. Un solo uso (`redeemed_at`, bajo `for
update`), 72 h de vencimiento fijo (no configurable — es una perilla que
degrada seguridad sin que quien la mueve entienda el costo), invalidación
del link anterior al reenviar (índice único parcial
`unique(customer_id) where redeemed_at is null and revoked_at is null`).
La tabla no tiene `grant select` para nadie — el panel lee el estado por
`customer_activation_status()`, que nunca devuelve `token_hash`.

**El canje (`claim_customer_activation(p_token text)`) — aplicación
literal de ADR-0005.** Un solo parámetro, el token; no nombra ninguna fila
de cliente. La identidad sale de `auth.uid()`, la fila destino sale
enteramente del token. **No hay IDOR posible porque no hay ningún input
que apunte a una fila.** `security definer`, `revoke ... from public,
anon` explícito (nunca a `anon`: vincular `auth.uid()` a una fila exige
que exista una sesión). Si el profile que canjea ya es cliente de esa
organización con cuenta propia, devuelve `ALREADY_CUSTOMER_OF_ORGANIZATION`
**sin marcar el token como canjeado** — el canje **nunca fusiona
automáticamente** dos filas; el dueño resuelve con `merge_customers()`
(OWNER-only, ver abajo) y el mismo link sigue sirviendo.

**Transporte del token a través del login — se aparta de ADR-0015 a
propósito.** En ADR-0015 el intent viaja sin firmar en query params porque
son datos ya públicos; acá el query param **es un secreto**. El route
handler `GET /activar/[token]` lo mueve a una cookie `httpOnly`, `Secure`,
`SameSite=Lax`, 15 min de vida, y redirige a `/activar/continuar` **sin el
token en la URL** — no queda en el historial del navegador ni en logs de
proxy. Ese primer GET manda `Cache-Control: no-store` y
`Referrer-Policy: no-referrer`. `returnTo` sigue el allowlist
server-side de ADR-0015 (`lib/return-to.ts`) — acá un open redirect sería
exfiltración directa del token, no solo phishing. La pantalla de
confirmación (`/activar/continuar`) **nunca vincula automáticamente**,
incluso con sesión ya abierta (mismo principio *confused deputy* de
ADR-0015): exige un clic explícito, y ofrece "usar otra cuenta" para el
caso más común, alguien abre el link con la sesión de otra persona en el
mismo teléfono.

**Gates de rol** (RPC `security definer`, no la server action — la RLS de
`customers` admite a cualquier miembro):

| RPC | Rol | Motivo |
|---|---|---|
| `create_managed_customer` | STAFF | mismo criterio que `enroll_customer_by_email` — alta de clientes es trabajo de mostrador |
| `issue_customer_activation` / `revoke_customer_activation` | STAFF | el link no da rol ni acceso a datos de la organización, da "ser este cliente"; gatearlo a OWNER forzaría a compartir su cuenta |
| `unlink_customer_profile` | OWNER | corta el acceso de una persona identificada, como `revoke_member` |
| `merge_customers` | OWNER | mueve pagos y reservas entre filas — es dinero |

**Hallazgo E, cerrado con un trigger, no con RLS.**
`customers_write_staff` (Phase 1) es `for all using
(is_organization_member(organization_id))`, sin `with check` separado, y
esa condición solo mira `organization_id` — nada impedía un
`PATCH /customers?id=eq.<fila_ajena> {profile_id: <uuid propio>}` directo
por PostgREST. Con clientes gestionados esperando ser reclamados, es el
camino obvio para saltear el token entero. Cerrado con
`customers_profile_link_guard` (`BEFORE UPDATE`): rechaza cualquier cambio
a `profile_id` salvo que la transacción haya hecho
`set_config('app.allow_profile_link', 'on', true)` — solo lo hacen
`claim_customer_activation()`, `unlink_customer_profile()` y
`merge_customers()`. Es un trigger y no una convención por el mismo motivo
que `set_updated_at`: hay varios caminos de escritura y el trigger es el
único punto que los cubre todos, incluido cualquiera futuro.

**`merge_customers(target, source)` — nunca reapunta lo que chocaría.**
Mueve `bookings`/`payments` fila por fila y deja que las restricciones
reales de la base (el unique de `CONFIRMED` por slot, el `EXCLUDE` de
ADR-0024 sobre períodos `PAID` solapados) decidan qué colisiona —
capturando la excepción por fila en vez de reimplementar esa misma regla
acá y arriesgar que las dos definiciones diverjan. Lo que no se puede
mover queda en la fila origen (ahora inactiva, `merged_into_customer_id`
apuntando al target) y se reporta, nunca se adivina ni se descarta.
Rechaza con `ALREADY_CUSTOMER_OF_ORGANIZATION` si la fila origen también
tiene `profile_id` — ese no es el caso que esta función resuelve.

**Interruptor por organización.** `organizations.customer_activation_enabled`
(default `true`) apaga el mecanismo completo para un rubro que no quiera
que circule un link de WhatsApp que diga "sos cliente de X" (ej. salud
mental). Chequeado dentro de `issue_customer_activation()`.

**Rate limiting.** Solo el de emisión se resolvió sin infraestructura
nueva: `count(*)` sobre `customer_activations.created_at` dentro de la
propia RPC (30/hora y 200/día por organización). El de canje (fuerza
bruta contra el token) queda como **riesgo residual aceptado por
escrito** — con 256 bits es computacionalmente inviable, y la
infraestructura general de rate limiting sigue pendiente desde ADR-0008.

**Verificación pendiente antes de demo** (no ejecutable por este agente en
esta sesión — sin acceso a shell/Supabase CLI): correr
`npx supabase db reset`, y el checklist de verificación manual que queda
como comentario al final de la migración 21 (Hallazgo A con `profile_id`
genuinamente `NULL`, Hallazgo E vía PostgREST directo, canje end-to-end).

## Pendiente de definir (Phase 1)

- Proveedor de auth concreto: **Supabase Auth** (ADR-0002, cerrado).
- Traducción de las políticas de ADR-0006 a SQL concreto por tabla (con
  `database-agent`).
- Alcance exacto de permisos de `STAFF` vs. `OWNER`.
- Si `GET /me/bookings` agrega across todas las organizaciones del
  profile o requiere parámetro de organización (con `backend-api-agent`,
  ver `api.md`).
- Shape público exacto a nivel de campo de `Organization`/`Service` (qué
  campos NUNCA van en la respuesta pública — facturación, config interna,
  referencias a `OrganizationMember`/`Profile`) — a definir con el schema
  de Phase 1.
