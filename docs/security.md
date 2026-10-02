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
  `OrganizationMember`). Desde ADR-0033 su alcance lo define un
  **`OrganizationRole` configurable por la propia organización** (nombre
  libre + cinco permisos booleanos: `VIEW_PAYMENTS`, `MANAGE_PAYMENTS`,
  `MANAGE_BOOKINGS`, `MANAGE_CUSTOMERS`, `MANAGE_ATTENDANCE`). Sin rol
  asignado cae en el rol por defecto de la organización, que el día del
  deploy puede todo lo que un `STAFF` podía antes. Ver "Fase 32" más abajo.
  Desde ADR-0033 hay **dos** caminos para llegar a `STAFF`, los dos
  `OWNER`-gated: `invite_member_by_email()` (sincrónico, exige cuenta previa) y
  el canje de una `TeamInvitation` (ADR-0034, link de 24 h vinculado al email,
  para quien todavía no tiene cuenta). **Ninguno de los dos puede producir un
  `OWNER` por invitación**: el segundo lo tiene prohibido por `CHECK`. Ver
  "Fase 33" más abajo.
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
`SameSite=Lax`, y redirige a `/activar/continuar` **sin el
token en la URL** — no queda en el historial del navegador ni en logs de
proxy. Ese primer GET manda `Cache-Control: no-store` y
`Referrer-Policy: no-referrer`, y **no toca la base**: WhatsApp le pega a
todo link compartido para armar el preview, así que cualquier cosa que ese
GET consumiera la consumiría un crawler antes que la persona.

**La cookie dura lo mismo que el token (72 h), no menos** — corregido el
2026-09-23 tras el bug en producción "el primer intento de abrir el link ya
dice que está expirado". La cookie es *la única copia* del token en claro
(la base guarda sólo el SHA-256), y el flujo de ADR-0026 es por definición
el de alguien que **todavía no tiene cuenta**: abre el link, se registra,
confirma por mail y recién ahí vuelve. Con 15 min de vida, la cookie —y no
el token— era el verdadero vencimiento del link, y al volver
`/activar/continuar` no encontraba nada y decía "venció" con el token
todavía pendiente en la base por tres días. El invariante quedó escrito y
testeado en `frontend/lib/activation-cookie.ts`: *la cookie nunca puede
vencer antes que el token*, con el TTL de SQL fijado desde
`backend/test/phase21.managed-customers.test.ts` y el de la cookie desde
`frontend/app/activar/[token]/route.test.ts`. `returnTo` sigue el allowlist
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

### Residuales de la ventana de 72 h (revisión de seguridad 2026-09-23)

Alargar la cookie de 15 min a 72 h es correcto —la cookie **no puede** ser
el vencimiento real del link, ver arriba— pero mueve dos cosas, y quedan
escritas para no volver a discutirlas:

- **Dispositivo compartido.** El token ahora sobrevive hasta 72 h en el
  navegador donde se abrió el link. Si esa persona no termina el canje,
  cualquiera que use ese mismo navegador, inicie sesión con su cuenta y
  entre a `/activar/continuar` puede quedarse con la invitación (la ficha
  del cliente: su nombre, teléfono, pagos y reservas de ahí en adelante).
  Mitigado, no cerrado: la pantalla **nunca vincula automáticamente**,
  exige un clic explícito y muestra con qué cuenta se va a vincular. Se
  acepta porque el link sigue estando en el WhatsApp de esa misma persona
  igual, y porque acortar la cookie reintroduce el bug que ya llegó a
  producción. **Regla:** un secreto que sólo vive en una cookie se borra en
  cuanto se consume — `claimActivation()` la borra **con su `path`**
  (`/activar`), no con `cookies().delete(name)`, que apunta a la cookie de
  `path=/`, que es otra. Fijado en `frontend/lib/activation-cookie.test.ts`.
- **`Referrer-Policy: no-referrer` de esa ruta lo pisaba el proxy.** El
  bloque `header` de `frontend/deploy/Caddyfile` es un *set*, no un
  *default*: reemplazaba el `no-referrer` que manda `GET /activar/[token]`
  por `strict-origin-when-cross-origin`. Impacto real hoy: despreciable —
  esa respuesta es un `303` sin cuerpo, no hay documento que dispare un
  `Referer`, y en un redirect el navegador no usa la URL del redirect como
  referrer. Se corrigió igual (`?Referrer-Policy`, o sea "sólo si la app no
  mandó uno") porque la única defensa que le queda a esa URL, si alguna vez
  renderiza un documento (una página de error de Next sobre la misma URL,
  por ejemplo), es ese header. **Regla: el proxy no baja headers de
  seguridad que la app sube; los headers globales del edge van como
  default (`?`), nunca como override.**
- **Logs.** El token viaja en la URL exactamente una vez, y hoy Caddy
  corre **sin `log`** (los access logs de Caddy 2 son opt-in), así que no
  queda en disco. Si alguna vez se habilita access logging, ese `GET` deja
  el token en claro en el log — hay que excluir `/activar/*` o redactar el
  URI en la misma tarea, no después.

## `my_customer_organizations()` (Fase 26) — portal del cliente

RPC nueva (`20260923130000_phase26_my_customer_organizations.sql`) que
lista los negocios donde el profile autenticado ya es `Customer`, para que
alguien recién activado tenga un link a la agenda en vez de un portal
vacío. Cumple el patrón completo de las RPC de portal (`my_bookings()`,
`my_services()`):

- **Identidad sólo desde `auth.uid()`.** Cero parámetros: no hay input que
  apunte a una fila, así que no hay IDOR posible. Sin sesión, `auth.uid()`
  es `null` y la comparación no devuelve filas (defensa además del grant).
- **`where c.is_active`**: un cliente dado de baja deja de ver el negocio.
- **ADR-0028**: `revoke execute ... from public, anon` + `grant ... to
  authenticated` explícitos.
- **Shape mínimo**: `slug` y `name` de la organización, ambos datos ya
  públicos (son la página pública, ADR-0023). No expone `organization_id`,
  ni la fila de `customers`, ni nada de otros clientes.

El `security definer` es necesario y está acotado: `organizations_select_members`
(Fase 1) limita el `SELECT` directo de `organizations` a miembros, así que
un `Customer` sin membership no puede leer ni el nombre del negocio al que
pertenece por PostgREST.

**Resuelto (Fase 27, `20260923140000_phase27_organization_slug_format_check.sql`)
— `organizations.slug` sin `CHECK` de formato.** Era `text not null unique`
y `create_organization_with_owner()` lo insertaba tal cual, sin validar — la
forma la garantizaba hasta ahora sólo Zod en el formulario. Un usuario
autenticado podía llamar la RPC por PostgREST con `p_slug = '/evil.com'`;
cualquiera de los ~8 `href={\`/${slug}\`}` del frontend (`/me`,
`/me/servicios`, `/admin`, `/org/[slug]/layout`, confirmación de reserva)
producía entonces `//evil.com`, que el navegador lee como URL
protocol-relative: un link externo con la estética del producto, dentro de
la sesión de la víctima. Explotarlo exigía además que la víctima fuera
`Customer` activo de esa organización (o sea, que hubiera canjeado una
invitación del atacante), así que la severidad era baja, pero el arreglo
correcto es **un `CHECK` en la columna** (mismo patrón que
`organizationSlugSchema`), no parchear cada `href`. `frontend/lib/organization-path.ts`
—que sí valida— cubre sólo el `redirect()` del canje, y sigue sin tocarse:
el dato pasa a ser imposible de guardar mal, así que no hace falta.

`organizations_slug_format_check` espeja exactamente `organizationSlugSchema`
(`backend/src/schemas.ts`): `char_length(slug) between 2 and 60` y
`slug ~ '^[a-z0-9]+(-[a-z0-9]+)*$'` (minúsculas/dígitos separados por un
solo guión, nunca al principio/final, nunca doble). Se agregó como `CHECK`
directo (no `NOT VALID` + `VALIDATE`): el único camino de escritura de
`slug` en todo el repo es `create_organization_with_owner()` (verificado
grepeando las 27 migraciones — no existe ningún `UPDATE ... set slug`), y
esa RPC sólo se llamó hasta ahora desde el formulario, que ya corre este
mismo patrón de Zod antes de invocarla — las filas existentes lo cumplen
por construcción. `create_organization_with_owner()` traduce la violación
del `CHECK` a `INVALID_SLUG` (vía `get stacked diagnostics ... =
constraint_name`, comparando contra el nombre exacto de la constraint) en
vez de dejar pasar el mensaje crudo de Postgres — mismo criterio que
`SLOT_FULL`/`INVITE_NOT_FOUND`: ningún RPC de este repo devuelve una
excepción genérica sin traducir al caller. 6 tests de integración en
`test/phase27.organization-slug-format.test.ts`: `/evil.com`, `//evil.com`,
mayúsculas, vacío, guión al inicio/final/doble, y el caso feliz.

## Escritura pública anónima — `platform_contact_requests` (ADR-0030, Fase 24)

Primera y **única** superficie del producto donde un anónimo sin sesión
escribe una fila. Regla general que queda establecida para cualquier
tabla futura de esta clase:

1. **RLS habilitada con cero policies.** Es la única defensa real: en
   Supabase las *default privileges* de `public` ya otorgan
   `select/insert/update/delete` a `anon`/`authenticated` sobre toda tabla
   nueva, y este repo nunca las revoca. "No hay grant" **no** es un
   argumento válido — el que deniega es RLS. El único acceso es vía RPC
   `security definer`. Mismo patrón que `customer_activations` (Fase 21).
2. **Toda validación de formato y largo vive en SQL**, no solo en Zod: la
   RPC es invocable directo por PostgREST sin pasar por el Server Action.
   Zod en `schemas.ts` es solo mensaje de error de borde y sus bounds
   deben espejar los `CHECK` exactamente.
3. **Lectura y marcado solo con `is_platform_admin()`**, mismo shape que
   `platform_organizations()`/`platform_invites()` (Fase 10): la RPC de
   listado filtra con `where public.is_platform_admin()` (devuelve lista
   vacía al no-admin, no error), la de escritura hace
   `if not public.is_platform_admin() then raise 'NOT_AUTHORIZED'`.
   `revoke execute ... from public, anon` + `grant ... to authenticated`.
4. **El texto libre se guarda crudo y se escapa en la salida**, nunca al
   revés. `name`/`message`/`business_type` son contenido controlado por
   un atacante: prohibido `dangerouslySetInnerHTML`, `innerHTML`,
   interpolación en HTML de email de notificación, o export a CSV sin
   prefijar `'` a celdas que empiecen con `= + - @` (formula injection).
   React escapa por default; el riesgo aparece cuando el lead sale de
   React.
5. **Rate limiting global = DoS barato.** Un contador `count(*)` global
   sin dimensión de origen convierte a cualquiera en capaz de agotar el
   cupo y bloquear envíos legítimos. En un formulario que es el tope del
   embudo comercial, el default correcto es **nunca perder un lead**: el
   límite estrecho debe ser por origen (IP vía
   `current_setting('request.headers', true)::json`), y el global solo un
   circuit breaker holgado.

   **Resuelto** en la migración: `origin_ip` (columna nueva) se deriva de
   `x-real-ip` con fallback al último salto de `x-forwarded-for` —
   verificado contra `supabase start` local que el gateway sobreescribe
   `x-real-ip` con su propia vista del peer TCP y apéndica esa misma
   vista como último elemento de `x-forwarded-for` (un cliente puede
   falsear entradas previas de la cadena pero no la última). Umbral real:
   5/hora por `origin_ip`; el global sube a 200/hora como circuit
   breaker holgado. Detalle completo en `docs/database.md`, Fase 24.

   **Limitación conocida, no resuelta en esta ronda:** `/contacto` se
   manda vía server action de Next.js (`frontend/app/actions/contact.ts`),
   que llama a la RPC server-side desde el droplet (ADR-0021) — no desde
   el browser del visitante. Todo el tráfico legítimo relayado por
   nuestro propio frontend comparte un único `origin_ip` aparente (el del
   droplet) ante Supabase; un script que llama a la RPC directamente sí
   muestra su IP real, y es exactamente el ataque que esto cierra. Aislar
   cada visitante del browser individualmente requeriría que el frontend
   reenvíe la IP real como parámetro explícito — fuera de alcance de esa
   ronda (backend-only).

   **Revisión de la Fase 2 de ADR-0030 (frontend): la limitación dejó de
   ser teórica y pasa a hallazgo abierto (severidad media,
   disponibilidad).** Con `/contacto` ya construido sobre el server
   action, el camino del relay no es solo el camino legítimo: es también
   el camino *más barato para el atacante*, porque le presta el
   `origin_ip` del droplet en vez de exponer el suyo. 5 POST a `/contacto`
   desde cualquier IP agotan el bucket por-origen del droplet y durante el
   resto de la hora **todo visitante legítimo del sitio ve
   `RATE_LIMITED`** — el "nunca perder un lead" del punto 5 invertido,
   con la única entrada comercial pública del producto caída. El límite
   por-origen solo protege contra el atacante que elige la ruta peor para
   él (PostgREST directo).

   Regla que queda establecida para cualquier escritura anónima
   relayada por un server action: **si el rate limit se mide en la base,
   la dimensión de origen tiene que llegar hasta ahí; si no, el límite
   hay que aplicarlo también en el proceso que sí ve al visitante.** El
   droplet tiene la IP real (Caddy setea `x-forwarded-for`) y es un
   proceso único, así que el límite por-visitante es implementable en el
   server action. **No** se resuelve pasando la IP como parámetro de la
   RPC a secas: la RPC es `grant execute to anon` e invocable directo, y
   un parámetro de origen que la función acepte como verdad es
   spoofeable por el atacante que hoy queda fuera.

   **Resuelto** en `frontend/lib/rate-limit.ts`, consumido desde
   `app/actions/contact.ts` **antes** de llamar a la RPC (protege también
   el cupo compartido del droplet, no solo agrega una capa redundante
   después). `getVisitorIp()` toma el último salto de `X-Forwarded-For`
   que agrega Caddy — verificado contra `frontend/deploy/Caddyfile` real
   (único reverse proxy, directo frente a Internet, ADR-0021): un
   visitante puede falsear entradas previas de la cadena pero no la que
   Caddy agrega él mismo, mismo criterio que ya usa `origin_ip` del lado
   de la RPC. Sin IP identificable, el límite local no bloquea — el
   backstop de la RPC sigue aplicando. Contador `Map` en memoria a nivel
   de módulo: válido porque el deploy es un único proceso de larga vida
   (no serverless); se resetea en cada restart/redeploy y no escala a
   múltiples instancias, límite aceptado al volumen actual. 7 tests en
   `frontend/lib/rate-limit.test.ts`.
6. **Chequeo y escritura del rate limit deben ser atómicos.** `count(*)`
   seguido de `insert` sin lock es el mismo TOCTOU que ADR-0004 cazó en
   `book_slot()`: bajo Read Committed, N requests concurrentes leen el
   mismo conteo y pasan todos. Donde no hay fila padre para tomar
   `for update`, el equivalente es `pg_advisory_xact_lock(<constante>)`
   antes del conteo.

   **Resuelto** en `submit_platform_contact_request()`: todo el bloque de
   chequeo-y-escritura corre bajo un único
   `pg_advisory_xact_lock(hashtext('platform_contact_requests_rate_limit'))`
   (clave global, no por `origin_ip` — al volumen esperado de este
   formulario la serialización cruzada entre orígenes no relacionados es
   irrelevante, y una clave solo-por-origen dejaría el umbral *global*
   racy bajo una ráfaga repartida entre muchos orígenes distintos).
   Cubierto por un test de concurrencia nuevo (`Promise.all` de envíos
   simultáneos) en `test/phase24.platform-contact-requests.test.ts`.
   **Sigue pendiente, fuera de alcance de esta ronda:**
   `issue_customer_activation()` (Fase 21) tiene el mismo patrón no
   atómico y no se tocó acá. **Y desde la Fase 25,
   `check_payment_no_duplicate()`** — ver abajo.

   **La regla no es solo del rate limit.** Vale para *cualquier*
   invariante de unicidad que se implemente como `select ... where
   <existe otro>` + `raise` dentro de un trigger o una función, en vez
   de como índice único / `EXCLUDE`. Bajo Read Committed una transacción
   no ve la fila no commiteada de la otra, así que dos requests
   simultáneos idénticos pasan los dos. Si el invariante no puede ser
   una restricción declarativa, el chequeo necesita
   `pg_advisory_xact_lock()` o un `select ... for update` sobre una fila
   padre común.

   **Resuelto en `check_payment_no_duplicate()` (Fase 25), y con una
   corrección al párrafo anterior:** la fila padre común de un `Payment`
   es su `Customer`, pero `select ... for update` sobre ella
   **deadlockea** — el trigger corre `AFTER INSERT`, para entonces el
   chequeo de FK de `payments.customer_id` ya dejó a esa misma
   transacción sosteniendo un `FOR KEY SHARE` sobre la fila, y pedir
   después `FOR UPDATE` es un upgrade de lock que con N inserts
   concurrentes se vuelve espera circular (reproducido, no teórico; ver
   `docs/database.md` § Fase 25). Se usa
   `pg_advisory_xact_lock(hashtext(customer_id::text))` **antes** del
   `select` que detecta el duplicado y dentro de la misma transacción.
   **Regla general que deja escrita:** para serializar un chequeo dentro
   de un trigger `AFTER INSERT/UPDATE`, el lock de fila sobre un padre
   referenciado por FK desde la propia fila insertada no es una opción —
   usar advisory lock. La colisión de `hashtext` (32 bits) entre dos
   `customer_id` distintos es sólo contención: dos filas sólo pueden ser
   duplicadas entre sí si comparten `customer_id`, así que el par en
   conflicto siempre cae en la misma clave y una colisión nunca deja
   pasar un duplicado, sólo serializa de más a dos clientes que no se
   pisaban.

## Fase 25 — niveles de acceso de las funciones nuevas

Ninguna decisión nueva; se registra el nivel que quedó verificado en
base local (`pg_proc.proacl`), para que no haya que re-derivarlo.

| Función | Nivel | Cómo se resuelve la identidad |
|---|---|---|
| `customer_billing_horizon(uuid,uuid,date)` | **interna** — `execute` solo para el owner (`postgres`); revocada de `public, anon, authenticated, service_role` | no resuelve identidad: es un helper puro, invocado owner-a-owner desde `schedule_rule_standing_reservations()`, que sí gatea con `is_organization_member()`. Mismo patrón que `service_plan_covered_service_ids()` / `resolve_usable_makeup_credit()`. No debe volverse alcanzable desde PostgREST. |
| `can_customer_book_detail(uuid)` | **CUSTOMER** — `revoke from public, anon` + `grant to authenticated` (ADR-0028) | **solo `auth.uid()`**, igual que `can_customer_book()` (ADR-0005). El único parámetro es el `slot_occurrence_id`, que no nombra ninguna fila de cliente: la organización sale de la ocurrencia y el `Customer` sale de `(organization_id, auth.uid(), is_active)`. Con una ocurrencia de otra organización devuelve `NOT_A_CUSTOMER` y nunca llega al bloque de crédito. |
| `schedule_rule_standing_reservations(uuid)` | **ADMIN** — `is_organization_member()` dentro del `where`; el `drop`+`create` por cambio de firma re-aplica `revoke from public, anon` + `grant to authenticated` (ADR-0028: `DROP` resetea privilegios) | membership sobre `schedule_rules.organization_id`. |

**Configuración de crédito de recupero (`organizations.makeup_credits_enabled`,
`release_deadline_hours`, `makeup_credit_expiry`,
`makeup_credit_expiry_days`): OWNER.** El gate real no es el `if
membership.role !== "OWNER"` del server action sino
`organizations_update_owner` (`for update using
is_organization_owner(id)`, sin `with check` separado, así que el `using`
oficia de check) — un `STAFF` no puede escribir esas columnas ni por
PostgREST directo. El server action es el mensaje de error, no la
frontera.

**Límite conocido (parcialmente resuelto en Fase 25):** los rangos que
valida `updateOrganizationSettings` eran más estrictos que los `CHECK` de
la tabla, y `organizations` tiene policy de UPDATE directa, así que un
OWNER podía saltear el action por PostgREST y poner
`release_deadline_hours = 0` —exactamente lo que el comentario del código
llama "la política entera"—. Aplica la regla de ADR-0030 §2 (los bounds
de la capa de aplicación espejan el `CHECK`, porque la tabla/RPC es
alcanzable sin pasar por el Server Action).

- **Resuelto:** `organizations_release_deadline_hours_range` pasó a
  `between 1 and 720` (migración de Fase 25), igualando el action.
- **Abierto:** `makeup_credit_expiry_days` sigue ≤ 365 en el action y sin
  tope en el `CHECK` (sólo `>= 1`).
- **Resuelto (ronda final de Fase 25):** los overrides por servicio de
  esta misma política viven en `services`
  (`makeup_credits_enabled_override`, `release_deadline_hours_override`,
  `makeup_credit_expiry_override`, `makeup_credit_expiry_days_override`)
  y `resolve_makeup_credits_policy()` los resuelve con `coalesce(override,
  organización)`. Como `services_write_staff` es `for all using
  is_organization_member(...)`, un STAFF podía antes `PATCH
  /rest/v1/services?id=eq.<X>` con `{"makeup_credits_enabled_override":
  true, "release_deadline_hours_override": 0}` y encender créditos de
  recupero —opt-in del OWNER, ADR-0025 resolución 3— con anticipación
  cero para ese servicio. **Regla que esto dejó escrita: un override por
  servicio hereda el nivel de acceso de `services` salvo que algo lo
  corrija explícitamente.** Cerrado con dos fixes en la misma migración
  de Fase 25: `services_release_deadline_hours_override_range` pasó a
  `between 1 and 720` (igual que `organizations`), y el trigger
  `services_billing_override_owner` (`check_service_billing_override_owner()`,
  `BEFORE UPDATE`) exige `is_organization_owner(organization_id)` cuando
  cualquiera de las 4 columnas de override cambia — el resto de columnas
  de `services` sigue editable por cualquier member. Con test dedicado:
  STAFF rechazado en las 4 columnas, OWNER puede, `name` (columna
  operativa) sigue editable por STAFF. Residual, sin cerrar a propósito
  (alcance pedido era `BEFORE UPDATE`): un `INSERT` de `Service` no pasa
  por este trigger — hoy no explotable porque ningún RPC/action inserta
  un `Service` con estas columnas ya pobladas, pero si alguna vez existe
  ese camino, el trigger necesita extenderse a `BEFORE INSERT OR UPDATE`.

## Fase 28 — catálogo público de planes y `plan_change_requests` (ADR-0035)

Migración: `20260923150000_phase28_plan_catalog_and_change_requests.sql`.
Revisado y verificado contra base local (`npx supabase db reset` +
`pg_policy`/`has_function_privilege`/sondas con clientes reales), no contra
el reporte del agente.

**Niveles de acceso que quedan fijados.**

| Función | Nivel | Verificado |
|---|---|---|
| `public_service_plans(text,uuid)` | **PUBLIC** | `anon=t, authenticated=t, PUBLIC=f` |
| `request_plan_change(uuid,text)` | **CUSTOMER** | `anon=f, authenticated=t, PUBLIC=f` |
| `my_plan_change_requests()` | **CUSTOMER** | ídem |
| `organization_plan_change_requests(uuid,boolean)` | **ADMIN** | ídem; `is_organization_member()` dentro del `WHERE` (no-miembro → lista vacía) |
| `resolve_plan_change_request(uuid,enum)` | **ADMIN** | ídem; gate por `is_organization_member(v_row.organization_id)`, o sea por la organización **de la fila**, nunca por un parámetro del caller |

**`organizations.currency` pasa a ser un campo PUBLIC.** Es el único campo
que `public_service_plans()` expone y que `anon` no podía leer antes:
`service_plans_select_public` (Fase 22, `using (is_active or member)`) ya
daba a `anon` todas las columnas de un plan activo —`price` incluido—, pero
`organizations_public` (Fase 4) es `id, slug, name, timezone, brand_color,
logo_path` y la RLS de `organizations` bloquea el `SELECT` directo.
Se acepta: `currency` es la unidad de un número que ya era público, y una
lista de precios sin moneda no es una lista de precios. **Queda escrito
para que haya una sola respuesta**: `currency` es público; si otra
superficie pública lo necesita, va en `organizations_public`, no en una RPC
nueva. El resto de la configuración de `organizations` sigue siendo
member-only.

**`plan_change_requests` — RLS de dos capas (ADR-0006), idéntica al resto.**
`plan_change_requests_select_self_or_staff` es, expresión por expresión, la
misma que `bookings_select_self_or_staff`, `payments_select_self_or_staff`,
`makeup_credits_select_self_or_staff` y
`service_entitlements_select_self_or_staff` (comparadas con
`pg_get_expr(polqual)` en base real). **Cero policies de escritura**: como
en `platform_contact_requests` (ADR-0030 §1), las *default privileges* de
Supabase sí otorgan `INSERT/UPDATE/DELETE` a `anon` y `authenticated` sobre
la tabla nueva —verificado en `information_schema.role_table_grants`— y el
que deniega es RLS, no la ausencia de grant. Comprobado con clientes
reales: `anon` insert → *"new row violates row-level security policy"*; un
cliente resolviéndose su propio pedido por `PATCH` directo → 0 filas
afectadas.

**`request_plan_change()` aplica ADR-0005 al pie.** No recibe ningún
`customerId`: resuelve el `Customer` desde `(plan.organization_id,
auth.uid(), is_active)`, y `current_service_plan_id` lo calcula la base con
`resolve_covering_service_plan()` en la fecha local de la organización. El
único input que nombra una fila es el `service_plan_id`, que no pertenece a
ningún cliente. El tope de 5 pendientes corre bajo
`pg_advisory_xact_lock(hashtext('plan_change_request:' || customer_id))` —
**clave por-cliente, no global**, así que dos clientes distintos no se
serializan entre sí. Probado con 8 pedidos simultáneos del mismo cliente:
5 aceptados, 3 con `TOO_MANY_PENDING_PLAN_CHANGE_REQUESTS`, 5 filas
pendientes en la tabla.

**Regla nueva — un trigger de integridad no puede abortar la escritura de
otra tabla.** `payments_close_plan_change_request` (`AFTER INSERT ON
payments`) hace un `UPDATE` de `plan_change_requests` **dentro de la
transacción del pago**. Con `plan_change_requests_same_org` declarado como
`BEFORE INSERT OR UPDATE` a secas, cualquier `raise` de
`check_plan_change_request_same_org()` sube hasta el `INSERT` del pago y lo
aborta: reproducido en base local (fila derivada a otra organización → el
`UPDATE` de resolución falla con
`PLAN_CHANGE_REQUEST_CUSTOMER_ORG_MISMATCH`). Llegar a ese estado exige que
alguien con membresía en dos organizaciones mueva el `Customer` y el
`ServicePlan` a la vez (`customers_write_staff`/`service_plans_update_staff`
son `for all using is_organization_member(...)`, sin `with check` aparte),
así que la severidad real era baja — pero el costo del arreglo es una
cláusula. **Corregido en la misma migración**: el trigger pasa a
`before insert or update of organization_id, customer_id, service_plan_id,
current_service_plan_id`, con lo que el `UPDATE` de resolución
(`resolved_at`/`resolved_by`/`resolution`) ni siquiera lo dispara y la
integridad que protege queda igual de cubierta. **Regla general: si un
trigger escribe en la tabla B desde una escritura en la tabla A, ningún
`BEFORE` de B puede quedar habilitado para columnas que A toca — o el
invariante de B se vuelve un modo de falla de A.**

**El texto libre de `note` hereda ADR-0030 §4**: se guarda crudo
(`CHECK length(trim(note)) between 1 and 500`, espejado en la RPC y en el
server action) y se escapa en la salida. Hoy sólo lo renderiza React en
`/org/:slug/plans`; si alguna vez sale de React (mail al mostrador, export
CSV), aplica la misma prohibición de `dangerouslySetInnerHTML` /
interpolación en HTML / celda CSV sin prefijar `'`.

**Riesgo residual aceptado: `request_plan_change()` no tiene rate limit.**
El tope de 5 pendientes limita las *filas*, no las *llamadas*: cada
invocación —incluida la idempotente, que devuelve el pedido ya existente—
recorre los servicios cubiertos por el plan resolviendo cobertura, y un
cliente autenticado puede repetirla sin costo. Es abuso de CPU, no de
datos, y requiere ser `Customer` activo de la organización. Queda dentro de
la infraestructura de rate limiting que sigue pendiente desde ADR-0008.

**Oráculo de existencia (informativo).** `resolve_plan_change_request()`
distingue `PLAN_CHANGE_REQUEST_NOT_FOUND` de `NOT_AUTHORIZED`, o sea que un
autenticado puede saber si un UUID existe. Con UUID v4 no es explotable; se
deja anotado por si alguna vez se decide unificar el mensaje.

## Fase 29 — `my_bookings()` recreada con `DROP` + `CREATE`

Migración: `20260923160000_phase29_my_bookings_slot_identity.sql`.

`DROP FUNCTION` borra el objeto **y sus privilegios**, y Postgres otorga
`EXECUTE` a `PUBLIC` por default en la función nueva — la trampa que
ADR-0028 ya documentó. La migración repite el `revoke execute ... from
public, anon` + `grant ... to authenticated` de Fase 19 después del
`create`. **Verificado en base real tras `npx supabase db reset` completo**
(no en el reporte del agente): `has_function_privilege` da `anon=false`,
`PUBLIC=false`, `authenticated=true`, y una llamada real con la clave
`anon` responde `permission denied for function my_bookings`.

Las tres columnas nuevas (`slot_occurrence_id`, `service_id`,
`service_color`) no amplían disclosure: el `WHERE` no cambia
(`join customers c on c.id = b.customer_id and c.profile_id = auth.uid()`,
que además deniega correctamente a un cliente gestionado con `profile_id
is null`), y `service_color` ya viaja al anónimo en
`get_public_availability()`.

**Regla que se repite y conviene automatizar**: todo `DROP FUNCTION` +
`CREATE` sobre una función que tenía `revoke` explícito debe reaplicarlo en
la misma migración, y la verificación que vale es
`has_function_privilege()` contra la base reseteada, no la lectura del SQL.

## Fase 30 — `audit_log`: una tabla que nadie puede escribir ni editar (ADR-0032)

Migración: `20260923170000_phase30_audit_log.sql`. **Pendiente de revisión
de `security-engineer`** (el propio ADR la declara obligatoria: tabla sin
policies de escritura, cuatro triggers `SECURITY DEFINER` y una lectura
nueva).

### Niveles de acceso

| Rol | `audit_log` (PostgREST) | `organization_audit_log()` |
|---|---|---|
| `anon` | Sin `SELECT` (revocado) y sin policy | Sin `EXECUTE` (revocado) |
| CUSTOMER / `STAFF` | Lista vacía (la policy no los alcanza) | `NOT_AUTHORIZED` |
| `OWNER` | Sólo las filas de **su** organización | Su organización; otra → `NOT_AUTHORIZED` |
| platform admin | Todo | Todo, con el actor sin enmascarar |
| `service_role` | Lee todo (BYPASSRLS) y puede insertar; **no** puede actualizar ni borrar | — |

`STAFF` queda afuera a propósito: es el sujeto mayoritario del log. El
`OWNER` entra porque las preguntas que motivaron el ADR ("¿quién anuló
este pago de mi mostrador?") son del dueño sobre su propia operación, y
todos los datos involucrados (montos, estados, ids) ya le son visibles en
las pantallas que tiene — la auditoría le agrega *cuándo y quién*, no
información nueva sobre terceros.

### Inmutabilidad: qué la garantiza, y qué no

1. RLS habilitada con **una sola policy, de `SELECT`**. Ninguna de
   `INSERT`/`UPDATE`/`DELETE`, para ningún rol.
2. `revoke insert, update, delete on public.audit_log from anon,
   authenticated` (+ `revoke select` de `anon`). Verificado con
   `has_table_privilege`: `authenticated` tiene `select` y nada más.
3. `reject_audit_log_mutation()` como trigger `before update or delete`
   (y `before truncate`), que **siempre** lanza `AUDIT_LOG_IMMUTABLE`.
   Cubre lo que los grants no cubren: un `SECURITY DEFINER` futuro, el
   dueño de la tabla y `service_role`. Tiene test.

Residuo conocido y aceptado (es lo que el ADR §5 especifica): el
`revoke` de escritura no incluye `service_role`, así que un proceso con
la clave secreta puede **insertar** filas forjadas (no editar ni borrar).
Hoy ningún código del producto usa esa clave — grepeado: `frontend/` no
referencia `SERVICE_ROLE` en ningún lado, sólo la usan los tests de
integración. Si algún día una server action la necesita, este revoke
debería extenderse.

### Los triggers son `SECURITY DEFINER` porque la tabla no acepta escrituras

Las filas nacen sólo desde `audit_write()`, `SECURITY DEFINER` propiedad
de `postgres`, revocada de `public, anon, authenticated, service_role`
(regla de helper interno de la Fase 19/ADR-0028): adentro de un definer
los privilegios se chequean contra el dueño, así que revocarla no le
quita nada y la saca de la API. Los cuatro triggers de auditoría son
`SECURITY DEFINER` por el mismo motivo — un trigger `SECURITY INVOKER`
correría con el rol `authenticated`, que justamente no tiene `INSERT`.

`auth.uid()` sigue funcionando dentro de un `SECURITY DEFINER`: lee el
GUC de la request, no el rol de ejecución. Por eso el actor registrado es
el staff real aunque la escritura venga de una RPC definer como
`admin_book_for_customer()`.

### La auditoría no puede tumbar una escritura de negocio

`actor_id` es FK a `profiles`. Si `auth.uid()` no tuviera perfil, el
`INSERT` de auditoría fallaría y **abortaría la transacción del pago**
(los triggers son `AFTER`, misma transacción). `audit_write()` baja el
actor a `NULL` en ese caso. Los triggers tampoco pueden rechazar nada:
son `AFTER` y devuelven `null`.

### Lo que el log **no** guarda, por diseño de privacidad

`metadata` es el diff mínimo y lo arma una función por tabla, nunca un
`to_jsonb(new)`: ids, montos, estados, fechas y el `note` opcional de la
RPC. **Nunca** nombres, mails ni teléfonos — esos viven en `customers` /
`profiles` con su propia RLS, y copiarlos a una tabla con reglas de acceso
distintas (legible por el `OWNER` y por la plataforma) los saca de ese
control. Es el modo típico de fallar de estas tablas y está anotado como
riesgo 2 del ADR.

### El enmascarado del actor de plataforma (ADR-0032 resolución 2)

El `OWNER` **sí** ve las filas de `ORGANIZATION_SUBSCRIPTION_CHANGED`
(una suspensión que el dueño no puede rastrear genera desconfianza), pero
`organization_audit_log()` no resuelve la identidad del actor cuando no es
miembro de la organización: devuelve `actor_id`/`actor_name` en `null` y
`actor_is_platform = true`. El `actor_id` crudo sigue en la tabla y un
`OWNER` puede leerlo por PostgREST — es un uuid que la RLS de `profiles`
(`select` sólo de la propia fila) no le deja convertir en un nombre, y el
ADR acepta explícitamente ese dato crudo. La defensa real de "no exponer
al equipo de la plataforma" está en la lectura que consume el frontend,
que es la única que la UI debe usar.

La resolución de nombre **no** filtra por `is_active`: un `STAFF` dado de
baja después sigue teniendo nombre. Sin eso, sus acciones se leerían como
"plataforma", que sería falso y justamente la información que el dueño
busca.

### Cobertura de tests (`test/phase30.audit-log.test.ts`, 12 casos)

Los cinco que el ADR pedía, más siete: el hecho se registra con `INSERT`/
`UPDATE` directo de PostgREST (pago y plan); el actor es el staff con un
cliente gestionado (`profile_id is null`); `authenticated` no puede
insertar, actualizar ni borrar; `service_role` tampoco actualizar ni
borrar; `STAFF` no lee ni la tabla ni la función; un `OWNER` no lee el log
de otra organización (ni por tabla ni por RPC); la cancelación hecha por
el propio cliente **no** genera `BOOKING_CANCELLED_BY_STAFF`; el actor de
plataforma llega enmascarado al `OWNER` y completo al platform admin; y
`app.audit_note` no se filtra a la transacción siguiente.

## Fase 32 — roles configurables dentro de `STAFF` (ADR-0033)

Esta sección **cierra la pregunta abierta desde la Fase 1** que estaba al
final de este documento ("alcance exacto de permisos de `STAFF` vs.
`OWNER`"). El diagnóstico era concreto: darle acceso al panel a un profesor
era darle acceso a la cobranza completa de la organización, y no había forma
de no hacerlo salvo no darle acceso.

### El modelo, en tres líneas

```
OWNER                      -> todo, sin mirar role_id ni permisos
STAFF con role_id          -> los cinco booleanos de ese OrganizationRole
STAFF con role_id null     -> los del rol is_default de la organización
```

`OWNER` **no** es configurable, y es una decisión de seguridad, no de
compatibilidad: quien administra roles no puede tener un rol administrable,
o existiría el ciclo "me edito el rol para poder editar roles". Como `OWNER`
corta antes de evaluar cualquier permiso, un rol mal configurado nunca puede
dejar a una organización sin nadie que pueda entrar a arreglarlo — el mismo
razonamiento que ya estaba escrito en `revoke_member()` / `LAST_OWNER`.
Por lo mismo, **administrar roles no es un permiso configurable**: si lo
fuera, existiría un rol capaz de ampliarse a sí mismo.

Un `OWNER` tampoco puede llevar rol: hay un `CHECK`
(`organization_members_owner_has_no_role`), no una convención de la UI, así
que no existe el estado confuso "le puse el rol Profesor al dueño y sigue
viendo todo".

### Los cinco permisos y qué habilitan exactamente

| Permiso | Habilita |
|---|---|
| `VIEW_PAYMENTS` | `payments` SELECT, `payment_service_coverage` SELECT, `plan_change_requests` SELECT, `organization_payment_summary()`, `customer_payment_detail()`, `organization_plan_change_requests()` |
| `MANAGE_PAYMENTS` | `payments` INSERT/UPDATE, `set_payment_status()`, `resolve_plan_change_request()` |
| `MANAGE_BOOKINGS` | `admin_book_for_customer()`, `admin_create_recurring_booking()`, `admin_preview_recurring_booking()`, `cancel_slot_occurrence()`, y la **rama de staff** de `cancel_booking()` / `cancel_recurring_booking()` / `retry_not_generated_booking()` |
| `MANAGE_CUSTOMERS` | `customers` INSERT/UPDATE/DELETE, `create_managed_customer()`, `enroll_customer_by_email()`, `issue_customer_activation()`, `revoke_customer_activation()` |
| `MANAGE_ATTENDANCE` | `mark_attendance()` |

`MANAGE_PAYMENTS` implica `VIEW_PAYMENTS` por `CHECK` de tabla (ADR-0033
resolución 1). Un rol que cobra sin poder ver lo cobrado no es un rol.

**La rama del cliente de cada policy de dos capas no se tocó.** ADR-0006
sigue igual: un `Customer` sigue viendo sus pagos, su cobertura y sus
pedidos de cambio de plan por `profile_id = auth.uid()`, sin pasar por
ningún permiso. Un `STAFF` que además es cliente de la misma organización
sigue pudiendo soltar **su propia** reserva sin `MANAGE_BOOKINGS` (hay test).

### Las dos puertas: RLS **y** RPC

Éste es el punto que decide si la feature es real o es decorado, y es el
modo de fallar más probable de este tipo de cambio:

- Una policy protege la **tabla** vía PostgREST.
- Una función `SECURITY DEFINER` corre como dueña de la tabla y **bypassea
  RLS por completo**. Endurecer `payments_select_self_or_staff` no cambia
  absolutamente nada para `organization_payment_summary()`.

Si se hubiera endurecido sólo la policy, el resultado habría sido una falsa
sensación de seguridad: la pestaña escondida en el panel y la cobranza
entera saliendo por `POST /rest/v1/rpc/organization_payment_summary` con el
JWT del profesor. **17 RPCs cambiaron su chequeo interno** (lista completa en
`database.md`, Fase 32) y cada permiso se testea por las dos puertas, no
"una vez por permiso".

La UI sólo esconde. La frase ya estaba escrita en `enforce_plan_limit()`:
*"Hiding the button is not enforcement: the RPCs and PostgREST are callable
directly."*

### Fuga residual aceptada y documentada (ADR-0033 resolución 2)

Un rol **sin** `VIEW_PAYMENTS` sigue viendo:

- `upcoming_unpaid` en `schedule_rule_standing_reservations()` (pantalla de
  horario fijo).
- `PAYMENT_REQUIRED` / `OVER_PLAN_QUOTA` como razón de rechazo en
  `evaluate_customer_booking()` y `can_customer_book_detail()`.

Es información de pago, aunque no sea un monto, y **se acepta a propósito**:
sin ella el rol no entendería por qué no puede anotar a alguien y el
producto quedaría inusable para el caso pedido. "No ver pagos" no es "no ver
que algo depende de un pago". Lo que **no** ve: montos, estados de pago,
fechas de pago, quién debe cuánto, ni la cobertura por período.

Precisión para el usuario: **"no ver pagos" tampoco es "no ver precios"**.
`service_plans_select_public` sigue siendo
`is_active or is_organization_member(...)` — la lista de precios de los
planes activos es pública, la ve cualquier visitante del calendario.
`VIEW_PAYMENTS` oculta *quién pagó, cuánto, cuándo y quién debe*.

### Hueco preexistente cerrado: precios de `ServicePlan` (ADR-0033 resolución 3)

Antes de esta fase, el gate de `OWNER` sobre planes y precios vivía **sólo
en TypeScript** (`frontend/app/actions/service-plans.ts`), mientras
`create_service_plan()` y las policies `service_plans_*_staff` pedían nada
más `is_organization_member()`. Un `STAFF` autenticado con su propio JWT
podía hacer `PATCH /rest/v1/service_plans?id=eq.<X> {"price": 1}` sin tocar
el panel. No lo causaba ADR-0033, pero no se podía construir encima:
habríamos entregado un producto donde el dueño cree que configuró quién toca
la plata y el rol restringido cambia precios igual.

Ahora: policies `service_plans_insert_owner` / `service_plans_update_owner`
/ `service_plan_services_write_owner`, **más** el trigger
`service_plans_write_owner` que es el que alcanza a la RPC `security
definer` (una policy no puede). Mismo patrón que
`check_service_billing_override_owner()` de la Fase 25.

### Multi-tenancy

- Un rol pertenece a una organización y **nunca cruza**: el trigger
  `organization_members_role_same_org` rechaza asignar un `role_id` de otra
  organización con `ROLE_OTHER_ORGANIZATION`, por RPC y por `PATCH` directo.
  La FK sola no alcanzaba (apunta a `organization_roles`, no a "los roles de
  esta organización") y el agujero habría sido un cross-tenant de
  autorización, no sólo de datos.
- `organization_roles_select_members` limita la lectura a miembros de la
  misma organización; un tercero no ve ni los nombres de los roles.
- `has_org_permission()` resuelve la membresía desde `auth.uid()`, nunca
  desde un parámetro del llamador — mismo criterio que `canCustomerBook()`
  (ADR-0005).

### Fail-closed, verificado

- Un rol inactivo o inexistente hace que el `EXISTS` de
  `has_org_permission()` no devuelva fila → `false`. No hay `coalesce` que
  convierta un `NULL` en un permiso.
- El `CASE` del helper termina en `else false`: un valor agregado al enum
  sin su columna deniega, no otorga.
- Un miembro revocado (`is_active = false`) pierde todos los permisos en el
  acto, sin tocar su `role_id`.
- El `OWNER` es el único camino que no consulta nada, y es el único que
  garantiza que una configuración de permisos rota sea reparable.

### Lo que sigue siendo de todo miembro activo (decisión, no olvido)

Ver el calendario, la agenda, el padrón de clientes, la lista de reservas de
una ocurrencia, el resumen de asistencia, el estado de activación de un
cliente y la **edición** de horarios. Un rol que no ve a los clientes es un rol
que no puede pasar lista; hacerlo opcional duplicaba el tamaño de la matriz
sin resolver ningún caso pedido. `makeup_credits` también: un crédito de
recupero es un derecho a reprogramar, no dinero cobrado, y tiene su propio
registro (ADR-0025).

⚠️ **Corregido en la Fase 34:** "la gestión de horarios" incluía por error
`discontinue_schedule_rule()`, que cancela reservas de clientes en masa. Pasó a
exigir `MANAGE_BOOKINGS`; **editar** una regla sigue siendo de todo miembro.

Residuo conocido, fuera del alcance de ADR-0033: `services.price` (columna
legada de pre-ADR-0024) sigue escribible por cualquier miembro vía
`services_write_staff`. Ya no la lee ninguna decisión de cobertura, pero si
alguna pantalla todavía la muestra conviene cerrarla en su propia migración.
→ Verificado explotable y con recomendación de cerrarlo ya: ver "Revisión de
`security-engineer` de las Fases 30/31/32", sección `services.price`.
✅ **Cerrado en la Fase 34:** `price` es OWNER-only por trigger.

### Checklist para `security-engineer`

1. Que no exista ninguna RPC `security definer` nueva que lea `payments`,
   `payment_service_coverage` o `plan_change_requests` sin
   `has_org_permission(..., 'VIEW_PAYMENTS')`.
2. Que ninguna policy permisiva nueva sobre esas tablas reintroduzca
   `is_organization_member()` en la rama de staff (las policies se combinan
   con `OR`: una sola alcanza para abrir todo).
3. Que `has_org_permission()` conserve `EXECUTE` para `PUBLIC` (lo necesitan
   las policies) y que nada más de esta familia lo tenga.
4. Que `customer_billing_horizon()` siga sin `grant` a `authenticated`.
5. Que el trigger de siembra siga creando el rol por defecto para toda
   organización nueva — sin él, todo `STAFF` futuro de esa organización
   queda sin permisos (fail-closed, pero roto).

## Revisión de `security-engineer` de las Fases 30/31/32 (2026-09-23)

Revisión obligatoria pedida por ADR-0032 y ADR-0033. Verificada **contra la
base reseteada** (`supabase db reset` hasta la Fase 32 + ~110 pruebas de
explotación, por PostgREST con JWT real y por SQL), no leyendo el SQL.
Resultado: `255/255` integración, `54/54` unitarios, typecheck limpio, y
**dos hallazgos bloqueantes sobre ADR-0033** más un hallazgo preexistente de
suscripción SaaS. La Fase 33 (ADR-0034) llegó después de esta corrida y
**no está cubierta acá**.

### Confirmado correcto (no volver a discutirlo sin una prueba nueva)

- `has_org_permission()` resuelve la membresía desde `auth.uid()` y la
  organización desde el parámetro: pasar el `organization_id` de otra
  organización devuelve `false` siempre (probado como `STAFF` y como `OWNER`
  de otro tenant). No hay camino de inyección de tenant.
- Los chequeos de permiso están **antes de cualquier escritura** en las RPCs
  endurecidas: en los intentos denegados no se creó ninguna `Booking`, la
  ocurrencia siguió `ACTIVE`, la asistencia siguió `PENDING`, la serie siguió
  `ACTIVE` y la reserva siguió `CONFIRMED`. Único matiz:
  `admin_book_for_customer()` toma el `select ... for update` de la
  ocurrencia **antes** del chequeo y devuelve `OCCURRENCE_NOT_AVAILABLE` para
  un id inexistente — es un oráculo de existencia sobre datos que el
  calendario público ya expone, y el lock dura un statement. No se cambia.
- `customer_billing_horizon()`: ACL real `{postgres=X/postgres}`.
  `authenticated` y `anon` reciben `permission denied for function`. El
  argumento del agente se sostiene contra la base.
- El trigger `service_plans_write_owner` cierra el hueco de precio por los
  cuatro caminos (`PATCH` de precio, de nombre y de `is_active`, `INSERT`
  directo, y `create_service_plan()` que es `security definer`). El escape
  `auth.uid() is null` **no es alcanzable** por un `authenticated`: un JWT de
  Supabase siempre trae `sub`, y `anon` —que sí tiene `uid` nulo— no pasa las
  policies `service_plans_insert_owner` / `_update_owner`, que se evalúan
  antes que el trigger. `service_plan_services` también cerrado.
- `organization_members_owner_has_no_role` y el trigger cross-tenant
  `organization_members_role_same_org`: un `role_id` de la Org B es rechazado
  en un miembro de la Org A por `set_member_role()` (`ROLE_NOT_FOUND`), por
  `PATCH` directo (`ROLE_OTHER_ORGANIZATION`) y por
  `invite_member_by_email()`. Un `OWNER` de la Org A tampoco puede *mover* un
  rol suyo a la Org B (lo impide el `WITH CHECK` de
  `organization_roles_write_owner`).
- Fuga residual de ADR-0033 resolución 2: es **exactamente** eso y nada más.
  `schedule_rule_standing_reservations()` devuelve sólo **contadores**
  (`upcoming_unpaid`, `upcoming_over_quota`, `upcoming_beyond_period`) —
  ningún monto, ninguna fecha de pago, ningún nombre de plan.
  `can_customer_book_detail()` resuelve al `Customer` del propio `auth.uid()`,
  así que no es una superficie de staff.
- `audit_log` inmutable, verificado por prueba directa: `authenticated` tiene
  `select` y **nada más** (`has_table_privilege`), un `OWNER` no puede
  `UPDATE`/`DELETE`/`INSERT` ni su propia fila (`permission denied for
  table`), `service_role` recibe `AUDIT_LOG_IMMUTABLE` en `UPDATE`/`DELETE`,
  y `TRUNCATE` falla **incluso como superusuario**.
- `organization_audit_log()` enmascara al actor de plataforma de verdad: para
  el `OWNER` devuelve `actor_id = null`, `actor_name = null`,
  `actor_is_platform = true`; para el platform admin devuelve el uuid y el
  nombre. **Precisión sobre `docs/database.md`**: la función *sí* joinea
  `profiles` siempre — el enmascarado vive en la proyección
  (`case when v_is_platform_admin or m.profile_id is not null then ...`), no
  en el join. El efecto es el mismo, pero la frase "no joinea `profiles` para
  ese caso" es inexacta y conviene no repetirla.
- Privacidad del log: 0 de 270 filas de `audit_log` contenían `@`, teléfono o
  nombre de cliente. `app.audit_note` no se filtra a la transacción siguiente
  (probado con `commit` + lectura posterior).
- `quote_service_plan_period()` (ADR-0031) no cotiza planes ajenos:
  `NOT_AUTHORIZED` para el `OWNER` de otro tenant, para un no-miembro y para
  una sesión sin `auth.uid()`; `anon` no tiene `EXECUTE`.
- Los tres `CHECK` de ADR-0031 no dejan estado inconsistente insertable.
  Probados y rechazados: `CALENDAR_PERIOD` sin meses, `CALENDAR_MONTH` con
  meses, `CALENDAR_PERIOD` con 5 y con 0 meses, `ONE_TIME` con meses,
  **`ROLLING_PERIOD` con `billing_anchor_month`**, `CALENDAR_MONTH` con
  anclaje, y anclaje 13. `ROLLING_PERIOD` con 5 meses entra, como se diseñó.
- El acotamiento del vencimiento del crédito de recupero vive en
  `issue_makeup_credit()` y **no hay segundo camino automático de emisión**:
  el único otro `insert into makeup_credits` es `grant_manual_makeup_credit()`,
  `OWNER`-only, con `expires_on` explícito y base `DAYS_AFTER` (nunca
  `END_OF_BILLING_PERIOD`, así que no hay nada que acotar). `makeup_credits`
  no tiene ninguna policy de escritura.
- Ninguna función `security definer` **no-trigger** del producto tiene
  `EXECUTE` para `PUBLIC` fuera de la lista ya documentada
  (`is_organization_member`, `is_organization_owner`, `is_platform_admin`,
  `get_public_availability`, `public_slot_detail`) + `has_org_permission`.
  Todas las nuevas llevan `set search_path = public`. Los `DROP`+`CREATE` de
  las Fases 31/32 (`create_service_plan`, `organization_team`,
  `invite_member_by_email`, `my_bookings`) conservaron sus `revoke`: ninguna
  quedó con `anon` ni con `PUBLIC`.
- Convivencia de las tres fases en `service_plans`: los triggers `BEFORE`
  (`require_paid_service`, `set_updated_at`, `terms_immutable`,
  `write_owner`) corren antes que los `AFTER` de auditoría, así que una
  escritura rechazada **no deja fila de auditoría**. Efecto colateral menor:
  `write_owner` es el último `BEFORE` por orden alfabético, así que un `STAFF`
  que toca un término inmutable recibe `SERVICE_PLAN_TERMS_IMMUTABLE` antes
  que `NOT_AUTHORIZED` — oráculo de "este plan tiene pagos vivos" para
  alguien que ya ve el plan. Aceptado.

### Hallazgo 1 (ALTO, bloqueante) — `MANAGE_PAYMENTS` sin `MANAGE_BOOKINGS` no puede cobrar

> ✅ **CERRADO en la Fase 34** (`20260923210000_phase34_permission_boundary_fixes.sql`),
> con el fix recomendado tal cual. Ver "Fase 34" más abajo.

Un rol "Recepción" (`VIEW_PAYMENTS` + `MANAGE_PAYMENTS` en `true`,
`MANAGE_BOOKINGS` en `false`) **no puede registrar un pago** ni pasar un pago
a `PAID` para un cliente que tenga fechas de horario fijo pendientes. La
transacción entera aborta con `NOT_AUTHORIZED`.

Cadena: `INSERT/UPDATE payments (status='PAID')` → trigger
`payments_reconcile_pending` → `reconcile_after_payment()` →
`reconcile_pending_recurring_bookings()` → `retry_not_generated_booking()`,
que desde la Fase 32 exige `MANAGE_BOOKINGS` cuando `auth.uid()` no es nulo y
no es el propio cliente. La reconciliación es una **consecuencia del
sistema**, no una acción del cajero sobre la reserva de otro, pero corre con
el `auth.uid()` del cajero.

Reproducido: cliente con 13 fechas `NOT_GENERATED/PAYMENT_REQUIRED`;
`insert payments` por Recepción → `NOT_AUTHORIZED`, 0 pagos registrados; el
mismo `insert` por el `OWNER` → OK. `set_payment_status(PENDING→PAID)` por
Recepción → `NOT_AUTHORIZED`. La suite no lo detecta porque su caso de
`MANAGE_PAYMENTS` usa un cliente sin fechas pendientes.

Por qué es de seguridad y no sólo un bug: el arreglo natural del dueño en
producción es darle `MANAGE_BOOKINGS` al rol de caja, que es justo la
restricción que ADR-0033 existe para permitir.

**Regla que queda establecida**: la autorización de una operación de
mostrador va en el **punto de entrada**, nunca en un helper compartido que
también se ejecuta como consecuencia interna de otra escritura. Fix
recomendado (mínimo): mover el cuerpo de `retry_not_generated_booking()` a un
helper interno sin chequeo (`revoke` de `public, anon, authenticated,
service_role`, igual que `audit_write()`), dejar la RPC pública como
"gate + llamada al helper", y hacer que
`reconcile_pending_recurring_bookings()` llame al helper. **No** agregar un
parámetro `p_enforce_authorization` a la RPC: sería un interruptor de bypass
invocable por PostgREST.

### Hallazgo 2 (ALTO, bloqueante) — `discontinue_schedule_rule()` evade `MANAGE_BOOKINGS`

> ✅ **CERRADO en la Fase 34.** El Orchestrator cerró la decisión de producto
> por la segunda opción: **discontinuar** exige `MANAGE_BOOKINGS`, **editar**
> un horario sigue siendo de cualquier miembro. Ver "Fase 34" más abajo.

Un `STAFF` con los cinco permisos en `false` llama
`POST /rest/v1/rpc/discontinue_schedule_rule` con una regla de su propia
organización y **cancela todas las ocurrencias futuras y todas las reservas
`CONFIRMED` futuras** de ese horario (`cancellation_reason =
RULE_DISCONTINUED`, `cancelled_by` = ese `STAFF`). El mismo actor recibe
`NOT_AUTHORIZED` de `cancel_slot_occurrence()`, que hace exactamente eso para
**una** fecha.

Reproducido de punta a punta. `discontinue_schedule_rule()` sigue gateada en
`is_organization_member()` porque "la gestión de horarios es de todo miembro
activo" — pero esa decisión se tomó mirando *crear* horarios, no la
cancelación en masa que esta función arrastra. La auditoría de ADR-0032 lo
**registra** (`BOOKING_CANCELLED_BY_STAFF`), así que es detectable, no
prevenible.

**Regla que queda establecida**: si una operación cancela reservas de
terceros, exige `MANAGE_BOOKINGS`, sin importar en qué pantalla viva. Fix
recomendado: agregar `has_org_permission(v_rule.organization_id,
'MANAGE_BOOKINGS')` a `discontinue_schedule_rule()`. Decisión de producto que
el Orchestrator tiene que cerrar: o la gestión de horarios pasa a requerir
`MANAGE_BOOKINGS` entera, o se parte en "editar el horario" (miembro) vs.
"discontinuarlo" (`MANAGE_BOOKINGS`).

Nota del mismo barrido: un miembro sin permisos igual puede poner
`schedule_rules.is_active = false` y `services.is_active = false` por `PATCH`
(no cancela reservas, pero apaga el horario y saca el servicio del calendario
público). Es la misma decisión de producto, un nivel más suave.

### Hallazgo 3 (ALTO, preexistente, no lo causan estas tres ADRs) — el `OWNER` se auto-reactiva la suscripción SaaS

> ✅ **CERRADO en la Fase 34**, con el fix recomendado tal cual. Ver "Fase 34"
> más abajo.

`organizations_update_owner` es `FOR UPDATE USING is_organization_owner(id)`
sin restricción de columnas. Con la organización en `SUSPENDED` y
`plan_code = 'starter'`, un `PATCH /rest/v1/organizations?id=eq.<propia>` del
`OWNER`:

- `subscription_status: SUSPENDED → ACTIVE` (se salta `enforce_plan_limit()`,
  que es lo que corta cuando la cuenta no está al día),
- `plan_code: starter → full` (se quita todos los límites de servicios,
  recursos, clientes y miembros),
- y mueve `trial_ends_at` / `current_period_end`.

`set_organization_subscription()` sí está cerrada a `is_platform_admin()`, y
un `STAFF` u otro tenant no pueden tocar nada: el agujero es sólo por `PATCH`
del propio dueño. Es **la misma forma exacta** del hueco que ADR-0033
resolución 3 acaba de cerrar una tabla más allá (gate sólo en el server
action de TypeScript). Mitigación que ya existe gracias a ADR-0032: el
trigger `organizations_audit_subscription` **registra** el cambio con
`actor_id` = el dueño (verificado), así que la plataforma puede detectarlo.

Fix recomendado: trigger `BEFORE UPDATE ON organizations` que rechace cambios
de `plan_code`, `subscription_status`, `trial_ends_at` y `current_period_end`
cuando `auth.uid() is not null and not is_platform_admin()` — mismo patrón
que `check_service_plan_write_owner()`.

### `services.price` — opinión pedida por el Orchestrator

> ✅ **CERRADO en la Fase 34** por la vía del trigger (no el `DROP COLUMN`,
> que sigue necesitando su propio ADR). Ver "Fase 34" más abajo.

**Cerrarla ahora, en la misma migración que los hallazgos 1 y 2, no como
deuda.** Motivos: (a) el argumento de ADR-0033 resolución 3 fue "no se puede
construir la feature con ese hueco abierto debajo", y `services.price` es
literalmente el mismo hueco en la tabla de al lado — dejar uno cerrado y el
otro abierto es peor que cerrar los dos o ninguno, porque el dueño ya cree
que configuró quién toca la plata; (b) el costo es una línea (sumar `price` a
un trigger `BEFORE UPDATE ON services` con el patrón
`check_service_plan_write_owner()`), y hoy nadie la escribe en el camino
normal, así que el riesgo de regresión es prácticamente nulo; (c) si queda
como deuda, el próximo que la lea va a asumir que se revisó y se aceptó.
Alternativa igual de válida y más limpia a futuro: **borrar la columna**, ya
que ninguna decisión de cobertura la lee desde ADR-0024 — pero eso es un
`DROP COLUMN` y necesita su propio ADR, mientras el trigger no.

### Residuos aceptados (documentados, no ocultos)

- `service_role` puede **insertar** filas forjadas en `audit_log` (no editar
  ni borrar). Confirmado: `frontend/` no referencia la service role key en
  ningún archivo — sólo usa `NEXT_PUBLIC_SUPABASE_ANON_KEY` en los tres
  clientes (`lib/supabase/client.ts`, `server.ts`, `middleware.ts`), y la
  clave secreta vive únicamente en los tests de integración. **No amerita
  cerrarse ahora**: quien tiene esa clave ya bypassea toda la RLS del
  producto, así que el `revoke` no agregaría una frontera real. La regla que
  sí hay que sostener: si alguna server action llega a necesitar la service
  role key, ese `revoke` se extiende **en el mismo cambio**.
- El `OWNER` puede leer el `actor_id` crudo de un actor de plataforma por
  PostgREST (no por `organization_audit_log()`). Verificado que **no** lo
  puede resolver a un nombre: la RLS de `profiles` le devuelve 0 filas.
- Borrar una organización a mano falla mientras tenga filas de auditoría
  (`on delete cascade` + trigger de inmutabilidad). Hoy no hay policy de
  `DELETE` sobre `organizations`, así que no rompe ningún camino; queda
  anotado para el día que se necesite dar de baja un tenant.

## Fase 33 — invitaciones de equipo: un token portador que da acceso a datos de terceros (ADR-0034)

Migración: `20260923200000_phase33_team_invitations.sql`.
**Revisión de `security-engineer` obligatoria por el propio ADR-0034**
(token portador + acceso a datos de terceros) — **hecha el 2026-09-23 contra
la base real** (`supabase db reset` + sondas propias, no sólo lectura del
SQL). Veredicto: **LISTO CON RESERVAS**. Lo verificado con sondas, y no
supuesto: privilegios reales de `new_activation_token()` /
`assert_team_seat_available()` en `pg_proc` (sin `EXECUTE` para `anon`,
`authenticated` **ni** `service_role`), `team_invitations` sin un solo grant
para `anon`/`authenticated` (SELECT/INSERT/UPDATE/DELETE por PostgREST →
`permission denied`, también para el `OWNER`), el `CHECK`
`team_invitations_never_owner` rechazando un `INSERT` **hecho como
`service_role` dentro de la base** (que tiene `bypassrls`), el canje con
otro email (`INVITE_WRONG_EMAIL`, sin consumir el token), el doble canje,
el no-pisado de rol, la rama de reactivación con cupo lleno
(`PLAN_LIMIT_REACHED`), el rate limit (10 pasan, la 11.ª corta, y otro
tenant no lo hereda) y que `issue_customer_activation()` conserva hoy el
gate `MANAGE_CUSTOMERS` de la Fase 32 (un `STAFF` con un rol sin ese
permiso recibe `NOT_AUTHORIZED`). Las dos reservas están abajo, en
**"El email es un segundo factor"** y en **"El cupo de plan"**.

Esto es el mecanismo de ADR-0026 aplicado al personal, y la diferencia de
seguridad es la que manda todas las decisiones de abajo:

| | Cliente (ADR-0026) | Equipo (ADR-0034) |
|---|---|---|
| Lo que otorga el token | *"sos este cliente"* — sus propias reservas y pagos | acceso a **los datos de otras personas** en todo el tenant: padrón, teléfonos, cobranza |
| Quién emite | `STAFF` (`MANAGE_CUSTOMERS`) | **`OWNER`** |
| TTL | 72 h | **24 h** |
| Rate limit | 30/h, 200/día | **10/h, 30/día** |
| Destino del token | una fila que **ya existe** (`customers.id`) | una fila que **nace en el canje** |
| Unicidad de "un link vivo" | por fila (`customer_id`) | por `(organización, email)` |

Mismo patrón, radio de explosión mucho mayor. Tablas y RPCs separadas
precisamente para poder endurecer el de mayor riesgo sin volver a testear el
flujo de clientes, que ya está en producción.

**Emisión `OWNER`-only, y es una asimetría deliberada.** ADR-0026 resolución 2
gateó la emisión de activaciones de cliente a `STAFF` con el argumento de que
"gatearlo a `OWNER` forzaría a compartir su cuenta". Ese argumento no aplica al
alta de personal: es ocasional y es del dueño. Sumar equipo ya era `OWNER`-only
(`invite_member_by_email()`), y ADR-0033 dejó explícito que **invitar equipo y
administrar roles no son permisos configurables** — si lo fueran, existiría un
rol capaz de ampliarse a sí mismo.

**Una invitación nunca puede fabricar un `OWNER`.** `CHECK
team_invitations_never_owner (role = 'STAFF')`, no una convención de la RPC. Es
un `CHECK` y no un `if` porque el `if` sólo protege el camino que pasa por la
función: el `CHECK` cierra también `service_role`, que bypassea RLS por
definición. Promover a dueño sigue siendo un acto deliberado sobre un miembro
que ya existe y ya se autenticó. `claim_team_invitation()` inserta con
`role = 'STAFF'` literal, no con un valor que venga de ningún parámetro.

**El canje no puede apuntar a otra fila: está garantizado estructuralmente.**
`claim_team_invitation(p_token text)` recibe **un solo parámetro** (ADR-0005
aplicado literalmente, igual que `claim_customer_activation()`). No nombra
organización, ni miembro, ni rol. El destino sale entero del token y la
identidad entera de `auth.uid()`; **no hay input que apunte a una fila, así que
no hay superficie de IDOR**.

**El canje nunca le pisa el rol a un miembro que ya existe.** Es la regla de
autorización más importante de la fase, y merece subrayarse porque
`invite_member_by_email()` **sí** lo pisa (`set role = p_role, role_id = ...`).
Ahí es correcto: es sincrónico y lo ejecuta el `OWNER` en ese instante. En un
canje sería un camino de escalada — un `STAFF` que consigue una invitación a un
rol mayor — y también de sabotaje: un link que **degrada** a alguien que ya
trabaja, o que convierte a un `OWNER` en `STAFF`. Si `auth.uid()` ya es miembro
activo, el canje devuelve `OK` y **no toca nada**; si es un miembro dado de
baja, lo reactiva sin tocarle el rol.

**El email es un segundo factor, no un detalle.** `INVITE_WRONG_EMAIL` sin
excepción (ADR-0034 resolución 3), con el precedente exacto de
`create_organization_with_owner()`. El canal es WhatsApp y el vínculo es el
email: son cosas distintas a propósito. Eso convierte un secreto de un factor
en uno de dos y mitiga el caso de falla realmente probable — el número mal
tipeado. Un link que cae en el teléfono equivocado **no sirve** salvo que el que
lo reciba también controle esa casilla. El costo es que quien se registró con
otro email (típico con Google) recibe un error y el dueño reemite: fricción
visible y arreglable, no un acceso silencioso al tenant equivocado.

> **Reserva del `security-engineer` (2026-09-23) — el segundo factor no es
> gratis: depende de dos cosas que hoy no están garantizadas.** Verificado
> contra la instancia real, no razonado: con la configuración de auth que el
> repo commitea (`backend/supabase/config.toml`,
> `[auth.email] enable_confirmations = false`), `signUp` con el email invitado
> devuelve **sesión inmediata** y Supabase escribe `email_confirmed_at` solo.
> En esa configuración, quien tenga el link **y sepa a qué email fue emitido**
> se registra con ese email sin tener la casilla y canjea: entra al tenant como
> `STAFF`. Por lo tanto:
>
> 1. **Regla nueva (auth, producción):** la confirmación de email tiene que
>    quedar **activada** en el proyecto de producción. Es lo que hace que
>    `signUp` no entregue sesión hasta confirmar, y es lo único que convierte a
>    `INVITE_WRONG_EMAIL` en un factor real. **No se puede arreglar en SQL:**
>    con confirmaciones apagadas `email_confirmed_at` viene igual, así que un
>    `and email_confirmed_at is not null` en `claim_team_invitation()` no
>    distingue una casilla verificada de una auto-confirmada. Verificar esta
>    setting en el dashboard **antes** de publicar la pantalla de equipo.
> 2. **Regla nueva (frontend, `/equipo/[token]`):** ni la página de canje, ni
>    el mensaje de WhatsApp, ni ningún error pueden **nombrar el email ni el
>    `display_name` de la invitación**. Es la misma regla que ADR-0026 §3 para
>    el nombre del cliente, y acá es la que sostiene el factor: si la pantalla
>    dice "Invitación para juan@gimnasio.com", el número mal tipeado —el caso
>    de falla que todo este diseño dice mitigar— vuelve a alcanzar solo. Hoy el
>    backend no lo filtra (`claim_team_invitation()` devuelve nombre y slug de
>    la organización y nada más, y `organization_team_invitations()` es
>    `OWNER`-only), así que el riesgo es enteramente de la UI que falta.
>
> **Conflicto abierto con ADR-0043 (gate `security-engineer`, 2026-09-29).**
> ADR-0043 apaga "Confirm email" en forma permanente, lo que **deja sin efecto la
> regla 1 de arriba**. Verificado en vivo contra Supabase local (GoTrue
> v2.197.0, `GOTRUE_MAILER_AUTOCONFIRM=true`): `signUp()` público con el email
> invitado devuelve sesión inmediata, y `claim_team_invitation(token)` responde
> `OK` → el que tenga el link y sepa (o adivine) el email entra como miembro.
> `INVITE_WRONG_EMAIL` deja de ser un factor de posesión y pasa a ser, como
> mucho, un factor de *conocimiento* del email (lo único que queda en pie es la
> regla 2). El mismo supuesto de "email = casilla verificada" también lo usan
> `invite_member_by_email()`, `enroll_customer_by_email()` y el chequeo de email
> de `create_organization_with_owner()`, y la vinculación automática de
> identidades de Supabase con Google OAuth. Hasta que el Orchestrator decida
> (mitigar o aceptar por escrito cada uno), **no asumir en ningún diseño nuevo
> que el email de `auth.users` pertenece a quien tiene la sesión**.
>
> **Resuelto (ADR-0043, corrección post-review, Fase 41).** Ver la sección
> "Fase 41 — ADR-0043 sin confirmación de email: reglas" al final de este
> documento. La regla "el email de `auth.users` NO prueba posesión de la
> casilla" pasa a ser permanente, no transitoria.

**El token.** 256 bits de `extensions.gen_random_bytes()` (nunca `random()`),
base64url sin padding, `sha256` en reposo, un solo uso bajo `for update` con un
`and redeemed_at is null` en el `update` final (dos canjes concurrentes del
mismo token: exactamente un ganador), 24 h de vencimiento fijo **no
configurable** — es una perilla que degrada seguridad sin que quien la mueve
entienda el costo. El acuñado vive ahora en `new_activation_token()`, sin
`EXECUTE` para nadie (ni `authenticated` ni `service_role`): sólo lo alcanzan
las funciones `security definer`, que corren como su dueño.

**La tabla es más cerrada que `customer_activations`.** RLS habilitada sin una
sola policy **y además** `revoke all from anon, authenticated`. `customers` es
escribible por staff, pero `team_invitations` tiene emails y teléfonos de gente
que todavía no aceptó nada: un `grant select` sería un padrón de contactos
expuesto a cualquier miembro. La única puerta son las cuatro RPCs
`security definer`, todas con `revoke execute from public, anon` antes del
`grant to authenticated` (ADR-0028).

**El read model es `OWNER`-only, más estricto que la propuesta.**
`organization_team_invitations()` nunca devuelve `token_hash`, y para un
no-`OWNER` devuelve cero filas en vez de una excepción. Aunque la propuesta
(§4.2) lo dejaba en "miembro", cada fila es el email y el teléfono de un tercero
que no aceptó nada, y la emisión, la revocación y la pantalla ya son
`OWNER`-only.

**Enumeración de emails: esta fase la *reduce*.** `invite_member_by_email()`
levanta `PROFILE_NOT_FOUND`, lo que convierte al panel en un oráculo de "¿tal
email tiene cuenta en la plataforma?". `issue_team_invitation()` **no consulta
`auth.users` en ningún momento** — ni siquiera para decir "esa persona ya es
miembro" —, así que no responde esa pregunta. El caso "ya era miembro" se
resuelve en el canje devolviendo `OK` sin tocar nada.

**Cross-tenant.** `team_invitations_role_same_org` (trigger) rechaza adjuntar un
rol de otra organización a una invitación → `ROLE_OTHER_ORGANIZATION`. La FK
sola no alcanza (apunta a `organization_roles`, no a "los roles de esta
organización") y el resultado sería un cross-tenant de autorización **acuñado en
un token portador**. El canje, además, sólo puede llevar a la organización que
está en la fila del token.

**Fail-closed en el rol.** Si el rol de la invitación fue **desactivado** entre
la emisión y el click → `INVITATION_ROLE_UNAVAILABLE`. No se cae al rol por
defecto ni se adivina: desactivar un rol suele ser una decisión de seguridad
reciente, y entrar "con otra cosa" sería exactamente lo que esa decisión quería
evitar.

**El cupo de plan se exige en los dos lados.** `enforce_plan_limit()` es
INSERT-only sobre `organization_members` y sigue siendo la regla. Además:
`assert_team_seat_available()` corre al **emitir** (contando miembros activos +
invitaciones vivas) para que nadie reciba un `PLAN_LIMIT_REACHED` al hacer click
en un link que ya tenía, y corre también en la rama de **reactivación** del canje
— que es un `UPDATE` y por lo tanto **no** pasa por el trigger. Sin ese segundo
chequeo, un link viejo reabriría una plaza que el plan ya no tiene.
(`invite_member_by_email()` comparte ese hueco en su propia rama de
reactivación; acá queda cerrado porque el que lo atraviesa es un token portador.
Vale revisarlo para el camino sincrónico en una próxima pasada.)

> **Reserva del `security-engineer` (2026-09-23) — el hueco del camino
> sincrónico está *demostrado*, no supuesto.** Contra la base real, plan
> `starter` (`max_team_members = 2`): `OWNER` + `staffA` ocupan las dos plazas
> → `revoke_member(staffA)` → `invite_member_by_email(staffC)` vuelve a llenar
> → `invite_member_by_email(staffA)` **pasa sin error** y la organización queda
> con **3 miembros activos sobre un tope de 2**. La rama de reactivación
> (`phase32:748-756`) es un `UPDATE ... set is_active = true` y
> `enforce_plan_limit()` es `before insert`, así que nadie mira el cupo; el
> ciclo es repetible y el tope deja de existir. **Decisión: no bloquea
> ADR-0034.** No es escalada ni cross-tenant —es `OWNER`-gated, sobre una
> persona que el dueño nombra, con el rol que el dueño elige— y es
> **preexistente desde la Fase 8**, no algo que introduzca esta fase; el daño es
> de facturación (un plan que no limita), no de datos. **Deuda nombrada, con el
> arreglo ya escrito:** un `perform public.assert_team_seat_available(
> p_organization_id, false);` antes del `update` de esa rama, exactamente como
> lo hace hoy `claim_team_invitation()`. `assert_team_seat_available()` es
> `security definer` y no tiene `EXECUTE` para nadie, así que
> `invite_member_by_email()` (también `security definer`, mismo dueño) puede
> llamarla sin abrir nada.

**Pendiente menor de endurecimiento, detectado en esta revisión.**
`team_invitations` quedó con `revoke all ... from anon, authenticated`; verificado
que `customer_activations` (Fase 21) **no** lo tiene: conserva los grants
completos que Supabase da por defecto a `anon`/`authenticated` y se apoya sólo en
"RLS con cero policies". Hoy eso alcanza (sin policy no hay fila que devolver),
pero es un cinturón menos: el día que alguien le agregue una policy a esa tabla
por cualquier motivo, los grants ya están puestos. Recomendado replicarle el
`revoke all` en la próxima migración — una línea, sin cambio de comportamiento.

**Transporte del token.** Igual que ADR-0026 y por los mismos motivos, con una
precaución propia: **ruta y cookie separadas** (`/equipo/[token]` →
`team_invitation_token`, `path: "/equipo"`). Alguien puede ser cliente del
negocio **y** haber sido invitado al equipo — la recepcionista que además
entrena ahí es el caso normal, no el raro. Si los dos flujos compartieran la
cookie, un token pisaría al otro, y el claro no se puede recuperar de la base.
El `maxAge` de la cookie tiene que ser el TTL del token (24 h): la cookie es la
única copia que tiene la persona, y el camino normal incluye registrarse y
confirmar el email (fue exactamente el bug de producción del flujo de clientes).

**Riesgos residuales aceptados, explícitos:**

1. **El link es un secreto portador.** Quien lo tenga **y** controle la casilla
   entra al tenant. Mitigaciones acumuladas: 256 bits de CSPRNG, `sha256` en
   reposo, un solo uso, 24 h, revocable, vinculado al email, nunca `OWNER`, y el
   rol acotado por `OrganizationRole`.
2. **El token viaja en el `text=` del deep link de WhatsApp.** Inevitable con el
   diseño "sin Twilio, el dueño lo manda desde su propio WhatsApp" (igual que
   ADR-0026 §5.3). Lo que contiene el riesgo es el TTL corto y el único uso, no
   el transporte.
3. **Fuerza bruta del canje.** Aceptado, como en ADR-0026 resolución 4: 256 bits
   lo vuelven computacionalmente irrelevante, y la infraestructura general de
   rate limiting sigue siendo deuda de ADR-0008 — no se construye dos veces la
   misma pieza por partes. El rate limit **de emisión** sí está, adentro de la
   RPC (10/h, 30/día por organización, números razonados y no medidos).
4. **`phone` es un dato personal más en la base.** Se guarda a propósito
   (ADR-0034 resolución 4) para poder reenviar sin retipear y para saber a dónde
   se mandó el link — el dato que hace falta cuando alguien dice "no me llegó".
   Es sólo canal: ninguna decisión de autorización lo mira.
5. **La invitación revela, a quien la reciba, el nombre de la organización que
   invita** (en el canje y en el mensaje). Es inevitable y es el punto. Lo que
   el mensaje **no** puede nombrar es a la persona invitada, para que un número
   mal tipeado no filtre el nombre de un tercero — misma regla que ADR-0026 §3.

**Trampa de mantenimiento encontrada en esta fase, vale para toda la base.**
Rehacer una función viva con `create or replace` partiendo de la migración que
la **creó** en vez de la última que la **modificó** revierte el endurecimiento
posterior **en silencio**. Pasó acá: al extraer el acuñado del token,
`issue_customer_activation()` se reescribió sobre la versión de la Fase 21
(`is_organization_member()`) y perdió el gate `MANAGE_CUSTOMERS` que la Fase 32
le había puesto horas antes. Lo cazó `test/phase32.configurable-roles.test.ts`,
que es exactamente para lo que existe. **Regla: antes de un `create or replace`
sobre una función existente, buscar todas las migraciones que la tocan y partir
de la última.**

### Revisión del frontend de ADR-0031/0032/0033/0034 (security-engineer, 2026-09-24)

Verificado sobre el código (y contra el `server-reference-manifest.json` del
build), no supuesto: cookie `team_invitation_token` con `path: "/equipo"`,
`maxAge` 24 h = `interval '24 hours'` de `issue_team_invitation()`
(`phase33:470`), `httpOnly`, `sameSite: lax`, `secure` en producción, nombre
y path distintos de `activation_token`/`/activar`; el `GET /equipo/[token]`
no toca la base; `INVITE_WRONG_EMAIL` muestra sólo el email de la sesión; el
mensaje de WhatsApp nombra sólo a la organización; el token no se loguea ni
vuelve en ningún error. Ocultar por permiso no es la defensa: `mark_attendance`
(`MANAGE_ATTENDANCE`), `payments` (RLS `VIEW_PAYMENTS`/`MANAGE_PAYMENTS`),
`discontinue_schedule_rule[_group]` (`MANAGE_BOOKINGS`, Fase 34), las RPCs de
roles (`is_organization_owner()` sobre la organización **del rol/miembro**, no
la del slug) y `organization_audit_log()` rechazan en la base.

**Regla nueva — un helper que devuelve un secreto no vive en un módulo
`"use server"`.** Toda función `export async` de un archivo `"use server"` se
registra como server action invocable por `POST` con su id, aunque sólo la
llame un server component. `readTeamInvitationToken()` (y
`readActivationToken()`, preexistente) figuran en el manifest: devuelven al
navegador el valor de una cookie `httpOnly`, o sea que deshacen el `httpOnly`
para cualquier JS del mismo origen que conozca el id (hoy el id no está en
ningún bundle de `.next/static`, por eso es **bajo** y no bloquea). Arreglo:
mover esos lectores a un módulo sin `"use server"` con `import "server-only"`.
**Resuelto (2026-09-24):** los dos viven ahora en `frontend/lib/server-cookies.ts`
(`import "server-only"`, sin `"use server"`); ya no figuran en el manifest de actions.

**Deuda (bajo): slugs reservados.** `organizations_slug_format_check` no
excluye los segmentos de primer nivel de la app (`equipo`, `activar`, `login`,
`signup`, `dashboard`, `org`, `me`, `auth`, `onboarding`, ...). Una organización
con slug `equipo` tendría su `/equipo/planes` capturado por
`/equipo/[token]`, que pisaría la cookie de una invitación en vuelo con
`"planes"`. Requiere un alta aprobada por la plataforma, así que es sabotaje
de baja probabilidad, no escalada.
**Resuelto (Fase 27b, `20260924100100_phase27b_reserved_organization_slugs.sql`):**
`organizations_slug_not_reserved_check` rechaza los segmentos top-level de
`frontend/app/` (+ `api`, `_next`); la RPC lo traduce a `SLUG_RESERVED`. Ruta
top-level nueva = migración nueva que la agrega al `CHECK`.

## Fase 34 — dónde va la autorización (cierre de la revisión de las Fases 30/31/32)

Migración: `20260923210000_phase34_permission_boundary_fixes.sql`.
Tests: `backend/test/phase34.permission-boundary-fixes.test.ts` (14 casos).

Cierra los tres hallazgos ALTOS de la revisión anterior y el residuo de
`services.price`. No agrega tablas, columnas ni enums: **sólo mueve chequeos de
autorización y agrega un trigger**. Ninguna firma de RPC cambió.

### La regla, escrita para que no se vuelva a romper

> **La autorización de una operación va en el punto de entrada público, nunca
> en un helper compartido que también corre como efecto colateral interno de
> otra escritura.**

Es la misma regla que el hallazgo 1 dejó enunciada, pero vista entera: poner el
chequeo adentro del helper falla de las **dos** maneras, y la auditoría encontró
una de cada una.

- **Falso negativo** — el helper corre como consecuencia de una escritura
  legítima y le exige al actor de *esa* escritura un permiso que no le
  corresponde. El cajero con `MANAGE_PAYMENTS` no podía cobrar (hallazgo 1).
- **Falso positivo** — otra entrada pública comparte el helper y hereda un
  chequeo más laxo que el suyo. `discontinue_schedule_rule()` cancelaba en masa
  con `is_organization_member()` mientras su prima de una sola fecha pedía
  `MANAGE_BOOKINGS` (hallazgo 2).

**Corolario operativo para cualquier RPC nueva:** antes de agregarle un
`has_org_permission()` a una función, preguntarse *quién más la llama*. Si la
llama un trigger o una función `SECURITY DEFINER` como efecto colateral, el
chequeo no va ahí: va en el punto de entrada y el cuerpo se extrae a un helper
revocado de todos los roles.

### Fix 1 — la partición de `retry_not_generated_booking()`

| | Autoriza | Alcanzable por |
|---|---|---|
| `internal_retry_not_generated_booking(uuid)` | **nada** | nadie: `revoke execute from public, anon, authenticated, service_role` (patrón `audit_write()`, Fase 30). Sólo funciones `SECURITY DEFINER` del esquema |
| `retry_not_generated_booking(uuid)` | cliente dueño · miembro con `MANAGE_BOOKINGS` · `auth.uid()` null | `authenticated` (grants de la Fase 19 intactos: mismo nombre y misma firma) |

`reconcile_pending_recurring_bookings()` llama al **helper**. Las dos cascadas
que pasaban por acá quedaron desbloqueadas: el trigger de `payments` (`PAID`) y
el de `services` (`payment_required` apagado).

Lo que **no** se hizo, a propósito y como pedía el hallazgo: no se agregó un
parámetro `p_enforce_authorization`. Sería un interruptor de bypass invocable
por PostgREST.

Se barrió el resto del código de la Fase 32 buscando la misma forma: **no hay
otro caso**. `cancel_booking()` la llama `release_my_booking()` (punto de entrada
del propio cliente, no un trigger) y `can_customer_book()` la llama `book_slot()`
(idem). El resto de funciones llamadas internamente no tiene chequeos propios por
diseño.

Matizado por la verificación independiente de abajo: en funciones el barrido
cierra (el único otro gated-llamado-por-trigger es
`generate_slot_occurrences_for_rule()`, y su gate es `is_organization_member()`,
que el actor de la escritura que la dispara ya cumple por RLS — mismo patrón,
inofensivo). Lo que **no** cierra es el mismo permiso por otra vía: el `DELETE`
por PostgREST con cascada de FK. Ver el hallazgo ALTO abierto al final de esta
sección.

Verificación de que el chequeo público **no** se aflojó: un `STAFF` sin permisos
sigue recibiendo `NOT_AUTHORIZED` de la RPC, el `OWNER` sigue pudiendo, y el
helper interno devuelve error al intentarlo como RPC.

### Fix 2 — la frontera "editar" vs. "cancelar"

`discontinue_schedule_rule()` exige `has_org_permission(..., 'MANAGE_BOOKINGS')`.
Decisión de producto cerrada por el Orchestrator:

| Operación | Permiso |
|---|---|
| `cancel_slot_occurrence()` — una fecha | `MANAGE_BOOKINGS` (ya lo era) |
| `discontinue_schedule_rule()` / `..._group()` — la regla entera | `MANAGE_BOOKINGS` (**nuevo**) |
| crear / modificar una `ScheduleRule` sin cancelar nada | cualquier miembro activo |

`discontinue_schedule_rule_group()` hereda el gate porque llama a la función una
vez por regla, y el `raise` aborta la transacción entera: no hay cancelación
parcial (verificado con un grupo de dos reglas, las dos siguen activas).

**Residuo aceptado, del lado correcto de la frontera:** un miembro sin permisos
sigue pudiendo `PATCH schedule_rules.is_active = false` y
`services.is_active = false`. Apagan el horario/servicio a futuro pero **no
cancelan ninguna reserva ni ocurrencia** y no estampan `cancelled_at` — es
edición. El caso que importaba (cancelarle las reservas a todo el mundo) está
cerrado.

### Fix 3 — el estado de suscripción no es configuración de la organización

Trigger `organizations_subscription_platform_only` (`BEFORE UPDATE ON
organizations`): `NOT_AUTHORIZED` si cambia `subscription_status`, `plan_code`,
`trial_ends_at` o `current_period_end` y `auth.uid() is not null and not
is_platform_admin()`.

Por qué trigger y no policy: `organizations_update_owner` tiene que seguir
dejando al `OWNER` editar nombre, timezone, branding y política de recupero. El
chequeo es **por columna**, igual que el de `services` en la Fase 25.

Los dos caminos que siguen abiertos, los dos verificados:

- **platform admin real** vía `set_organization_subscription()`.
  `SECURITY DEFINER` cambia privilegios, **no la sesión**: el trigger ve el
  `auth.uid()` del admin y `is_platform_admin()` da `true`. Probado con un
  platform admin de verdad (fila en `platform_admins`), no razonado.
- **`auth.uid()` null** — migraciones de datos (el backfill de la Fase 10) y
  jobs internos. No hay actor al que autorizar y no es alcanzable por PostgREST.

Un `PATCH` que escribe el mismo valor no es un cambio y pasa: no es escalada.
La mitigación de ADR-0032 (`organizations_audit_subscription` registra el
cambio) sigue ahí y ahora es redundante para este vector, no la única defensa.

### Fix 4 — `services.price`

`check_service_billing_override_owner()` (Fase 25) se **extendió** con `price`
en vez de agregar un segundo `BEFORE UPDATE` sobre la misma tabla. Cambiar
`price` exige `is_organization_owner()`, igual que `service_plans.price`
(`check_service_plan_write_owner`, ADR-0033 resolución 3). El resto de columnas
de `services` sigue siendo de cualquier miembro.

Se eligió el trigger y **no** el `DROP COLUMN`: borrar la columna sigue
necesitando su propio ADR. El gate es sólo en `UPDATE` — crear un servicio con
`price` en el `INSERT` sigue abierto, exactamente igual que los cuatro overrides
desde la Fase 25.

### Verificación

`npx supabase db reset` sobre el tip completo (Fases 1→34) + suite de
integración entera: **290/290 en 27 archivos**, **60/60 unitarios**, typecheck
limpio. Además se re-corrieron los dos scripts de explotación de la revisión
anterior contra la base arreglada: `r5.mjs` (regresión de reconciliación) 6/6 y
`r6.mjs` (bypass de `MANAGE_BOOKINGS`) 10/10 — incluida la línea que antes
informaba `services.price` editable por cualquier miembro, que ahora devuelve
`NOT_AUTHORIZED`.

### Checklist para `security-engineer` (suma a la de la Fase 32)

1. Que ninguna RPC nueva con `has_org_permission()` sea llamada además desde un
   trigger o desde otra `SECURITY DEFINER` como efecto colateral. Si lo es:
   partir en helper interno (revocado) + entrada pública gateada.
2. Que `internal_retry_not_generated_booking()` siga sin `EXECUTE` para
   `public, anon, authenticated, service_role`.
3. Que toda función que cancele reservas de terceros exija `MANAGE_BOOKINGS`,
   sin importar en qué pantalla viva.
4. Que ninguna policy ni RPC nueva permita escribir las cuatro columnas de
   suscripción de `organizations` fuera de `set_organization_subscription()`.
5. **Nuevo (ver hallazgo de abajo):** que ninguna tabla cuya policy de escritura
   sea `is_organization_member()` tenga hijos con `ON DELETE CASCADE` que sólo se
   puedan tocar con `MANAGE_BOOKINGS` / `MANAGE_PAYMENTS`. La cascada de una FK
   **no evalúa RLS**: el permiso efectivo sobre el hijo es el del padre.

### Verificación independiente de la Fase 34 (security-engineer)

Re-auditoría de los cuatro fixes, sin apoyarse en el reporte de quien los
implementó: ACLs leídos de `pg_proc`/`pg_policies`/`pg_trigger` sobre una base
recién reseteada, más sondas propias de explotación (`s1`–`s8`) además de
re-correr `r5`/`r6`. Resultado: **los cuatro fixes cierran lo que la ronda
anterior encontró y no abren nada nuevo.** Detalle de lo que quedó comprobado
contra la base y no sólo razonado:

- `internal_retry_not_generated_booking(uuid)` tiene `proacl` = `postgres=X`
  únicamente; `has_function_privilege` da `false` para `anon`, `authenticated`,
  `service_role` y `authenticator`, y los cuatro reciben `permission denied for
  function` al invocarla por PostgREST.
- La reconciliación **ocurre de verdad** con el cajero (`MANAGE_PAYMENTS` sin
  `MANAGE_BOOKINGS`): 13/13 fechas `NOT_GENERATED` pasaron a `CONFIRMED`. Ídem
  la cascada de `payment_required` apagado por un rol con los cinco permisos en
  `false` (12/12). La RPC pública sigue negando a ese mismo cajero y a un
  miembro de otra organización.
- `discontinue_schedule_rule_group()` con un grupo de dos reglas y tres reservas
  `CONFIRMED`: `NOT_AUTHORIZED`, las tres reservas intactas, las reglas activas y
  sin `cancelled_at`. El rol con `MANAGE_BOOKINGS` sí puede (control positivo).
- Fix 3 probado con una suspensión **real** de plataforma (`SUSPENDED`/`starter`)
  y valores distintos a los actuales: el `OWNER` no se reactiva, no se sube de
  plan, no se estira el trial, y el `PATCH` mixto nombre+plan se rechaza entero.
  El platform admin reactiva por RPC sin chocar con el trigger y
  `ORGANIZATION_SUBSCRIPTION_CHANGED` queda en `audit_log`. Un `PATCH` con el
  mismo valor pasa y no es escalada (no cambia nada).
- Fix 4: es el **mismo** trigger `services_billing_override_owner` extendido (un
  solo `BEFORE UPDATE` sobre `services` con esa función, no hay lógica
  duplicada); `service_plans.price` y los cuatro overrides de la Fase 25 siguen
  cerrados al `OWNER`.

Dos notas menores, sin impacto de seguridad: el comentario de la RPC pública
dice que la rama sin JWT es "el cliente service-role de los tests", pero
`service_role` **no** tiene `EXECUTE` sobre `retry_not_generated_booking()`
desde la Fase 19 (es más estricto que el comentario); y
`check_service_billing_override_owner()` es el único de los tres triggers de
autorización que **no** tiene el escape `auth.uid() is not null`, así que un
backfill futuro que toque `services.price` sin JWT fallaría.

#### Hallazgo ALTO abierto (preexistente, fuera del alcance de la Fase 34): destrucción por cascada de FK

El barrido de "otro caso con la misma forma" **no** cierra en cero. La Fase 34
corrigió la entrada por RPC (`discontinue_schedule_rule()`), pero la misma
capacidad sigue disponible por `DELETE` directo a PostgREST, y en peor versión:
borra filas en vez de cancelarlas.

`schedule_rules`, `services` y `resources` tienen policy `ALL using
is_organization_member(organization_id)` — cualquier miembro puede `DELETE`. Sus
hijos son `ON DELETE CASCADE` (`schedule_rules → slot_occurrences → bookings`, y
`services → payments`), y **la acción de una FK no evalúa RLS ni dispara ningún
gate de permiso**.

Reproducido (rol `STAFF` con los cinco permisos de ADR-0033 en `false`):

| Vector | Resultado medido |
|---|---|
| `DELETE /rest/v1/schedule_rules?id=eq.<X>` | 3 reservas `CONFIRMED` **borradas** (no canceladas): 0 filas restantes, sin `cancelled_at`/`cancelled_by`, sin `MakeupCredit`, sin fila de `audit_log` |
| `DELETE /rest/v1/services?id=eq.<X>` (plan `applies_to_all_services`) | el pago `PAID` del cliente **borrado**: historial financiero destruido por un rol sin `VIEW_PAYMENTS` ni `MANAGE_PAYMENTS` |

Rompe dos cosas escritas: la invariante de dominio "las reservas canceladas
**nunca se borran**" y la frontera de ADR-0033 (cancelar reservas de terceros
exige `MANAGE_BOOKINGS`; tocar pagos, `MANAGE_PAYMENTS`). Con un plan de alcance
acotado el `DELETE` de `services` queda tapado por accidente
(`SERVICE_PLAN_SCOPE_EMPTY` / `SERVICE_PLAN_TERMS_IMMUTABLE`), que es una
defensa de consistencia, no de autorización, y desaparece cuando el plan es
global.

No se corrige acá: partir `ALL` en `INSERT`/`UPDATE`/`DELETE` separados, o pasar
el borrado a un `BEFORE DELETE` que lo rechace cuando existan ocurrencias
futuras con reservas `CONFIRMED` o pagos asociados (y obligue a ir por
`discontinue_schedule_rule()`), es un cambio de multi-tenancy/autorización:
**requiere ADR del Orchestrator**. Lo verificado por ahora es el alcance exacto
del vector, para que la decisión se tome sobre datos.

> ✅ **Cerrado por ADR-0036 / Fase 35** — ver la sección siguiente. La resolución
> fue la primera de las dos opciones (partir la policy), sin trigger nuevo.

## Fase 35 — el `DELETE` de la Data API queda cerrado (ADR-0036)

Cierre del hallazgo ALTO de arriba. Migración
`20260923220000_phase35_close_data_api_delete.sql`.

### La regla, escrita para que no se vuelva a romper

> **Una policy de RLS protege la fila que el comando nombra, no las filas que la
> integridad referencial arrastra detrás.** Una tabla con hijos `ON DELETE
> CASCADE` no tiene forma de autorizar el efecto de su propio borrado: el único
> lugar donde ese efecto se puede negar es en el `DELETE` del padre.

Corolario práctico, y el criterio con el que se revisa una tabla nueva: **si una
tabla de negocio no tiene un caso de uso concreto y nombrado de `DELETE`, su
policy no debe incluir `DELETE`.** Una policy `FOR ALL` concede `DELETE` en
silencio, y ese es el único de los cuatro comandos cuyo daño no es reversible ni
auditable.

### Lo que se cerró

Siete tablas. Las policies `ALL` pasan a `INSERT` + `UPDATE` explícitos, con la
**misma** expresión que ya tenían (nada se abre ni se cierra además del
`DELETE`):

| Tabla | Antes | Ahora | Expresión |
|---|---|---|---|
| `schedule_rules` | `schedule_rules_write_staff` (`ALL`) | `*_insert_staff` + `*_update_staff` | `is_organization_member()` |
| `services` | `services_write_staff` (`ALL`) | `*_insert_staff` + `*_update_staff` | `is_organization_member()` |
| `resources` | `resources_write_staff` (`ALL`) | `*_insert_staff` + `*_update_staff` | `is_organization_member()` |
| `customers` | `customers_write_staff` (`ALL`) | `*_insert_staff` + `*_update_staff` | `has_org_permission(..., 'MANAGE_CUSTOMERS')` |
| `schedule_exceptions` | `schedule_exceptions_write_staff` (`ALL`) | `*_insert_staff` + `*_update_staff` | `is_organization_member()` |
| `service_entitlements` | `service_entitlements_write_staff` (`ALL`) | `*_insert_staff` + `*_update_staff` | `is_organization_member()` |
| `service_resources` | `service_resources_write_staff` (`ALL`) | `*_insert_staff` + `*_update_staff` | `exists (... services s ... is_organization_member(s.organization_id))` |

**El `OWNER` tampoco tiene `DELETE`.** No es un permiso más de la matriz de
ADR-0033: es una capacidad que la Data API deja de exponer para toda la
organización. Si algún día hace falta borrar de verdad, es una RPC nueva con su
propio gate y su propia decisión, no un `DELETE` genérico.

Sin trigger nuevo: bajo RLS la ausencia de policy de `DELETE` **deniega por
default**. Es la superficie más chica posible para el mismo resultado — un
`BEFORE DELETE` habría que mantenerlo, se puede desactivar, y habría que
razonarlo tabla por tabla contra siete conjuntos distintos de hijos.

### Fail-closed, verificado

- Un `DELETE` denegado por RLS **no devuelve error**: PostgREST responde `204`
  con `[]` y la fila sigue ahí. Quien escriba una sonda de explotación para esto
  tiene que mirar la fila, no el status code. El test afirma las dos mitades.
- El actor de los tests es el **`OWNER`**, deliberadamente: es el rol más
  privilegiado fuera del platform admin, así que no hace falta repetir la matriz
  de ADR-0033 tabla por tabla. Si el `OWNER` no puede, nadie de la organización
  puede.
- Los dos vectores de la tabla del hallazgo, re-probados como test permanente:
  `DELETE schedule_rules` con una `Booking` `CONFIRMED` abajo → rechazado, la
  reserva sigue `CONFIRMED` con `cancelled_at` en `null`; `DELETE services` con un
  pago `PAID` abajo → rechazado, el pago sigue `PAID`. También se comprueba que la
  cascada `customers → bookings`/`payments` no corre.
- El test se validó **al revés** antes de darlo por bueno: con una migración
  temporal que restauraba las dos policies `ALL` originales, 8 de los 11 casos
  fallan. Un test de "esto no se puede hacer" que nunca se vio fallar no prueba
  nada.

### Caminos que siguen vivos (confirmado, no asumido)

- `discontinue_schedule_rule()` — `UPDATE`, no `DELETE`: corta la regla, cancela
  las ocurrencias futuras y libera cupo, con gate `MANAGE_BOOKINGS` desde la Fase
  34. El test lo ejerce para que el ADR no se apoye en una suposición.
- `is_active = false` para `Service`/`Resource`; `revoke_member()` para equipo.
- `regenerate_occurrences_for_rule()` (Fase 3) sigue haciendo `delete from
  slot_occurrences` de ocurrencias futuras sin `Booking`: es el mecanismo de
  ADR-0003, `slot_occurrences` **no** está en la lista de siete, y además corre
  como `SECURITY DEFINER` (no evalúa RLS). Es el único `delete from` de todo
  `backend/supabase/migrations/`.
- En `frontend/app/actions/` no hay un solo `.delete()` contra PostgREST — el
  único `.delete()` del repo es `jar.delete()` sobre una cookie en
  `activation.ts`.

### Lo que **no** se tocó, a propósito

Ninguna FK `ON DELETE CASCADE`. El problema nunca fue la cascada — es correcta
para cuando la fila padre sí se borre por una vía legítima futura — sino la
puerta que la disparaba sin autorización. Dejar la FK y cerrar la puerta mantiene
el modelo de integridad intacto.

### Checklist para `security-engineer` (suma a las de las Fases 32 y 34)

- [ ] ¿Alguna tabla de negocio tiene policy `FOR ALL`? Si sí: ¿hay un caso de uso
      nombrado de `DELETE`? Si no lo hay, la policy debe partirse.
- [ ] Para cada tabla nueva con hijos `ON DELETE CASCADE`: ¿qué se borra en
      cascada, y ese borrado sería aceptable viniendo de cualquier actor que pase
      la policy del padre?
- [ ] Una sonda de `DELETE` denegado mira **la fila**, no el status code.

## Barrido completo del `DELETE` del schema (2026-09-23) — hallazgo CRÍTICO **CERRADO**

> ✅ **Cerrado por ADR-0037 / Fase 36** (`20260923230000_phase36_public_views_read_only.sql`) —
> ver la sección "Fase 36" más abajo. El relato del hallazgo se conserva tal
> cual porque es la mejor descripción de por qué la regla nueva existe.

Barrido de `security-engineer` sobre **las 27 tablas y las 2 vistas** del schema
(no sólo las de la ronda ADR-0031…0036), buscando el patrón de ADR-0036: una
puerta de escritura amplia + hijos `ON DELETE CASCADE` que lleven a datos que el
producto trata como "nunca se borran".

### Regla nueva, y el agujero que la motiva

> **Una vista sin `security_invoker` es un `SECURITY DEFINER` con forma de tabla:
> no evalúa la RLS de las tablas base. `GRANT SELECT` sobre ella no limita nada,
> porque en Supabase las default privileges del rol `postgres` en `public`
> (`pg_default_acl`, objtype `r`) ya le dieron `arwdDxtm` a `anon` y
> `authenticated` en el momento del `CREATE VIEW`.** Una vista de lectura pública
> se cierra con un `REVOKE INSERT, UPDATE, DELETE, TRUNCATE` explícito. El
> `GRANT SELECT` del `CREATE VIEW` es decorativo.

`organizations_public` y `services_public` (Fase 4, ADR-0008; `organizations_public`
recreada en la Fase 13) se crearon **a propósito** sin `security_invoker`: el
bypass de RLS en lectura *es* el diseño del calendario público, y el comentario de
la migración lo dice. Lo que no se vio es que el bypass aplica a los cuatro
comandos, y que `anon` tenía los cuatro. `relacl` de ambas vistas:
`anon=arwdDxtm/postgres`.

**No se arregla con `security_invoker = on`**: eso haría que el `SELECT` evaluara
`organizations_select_members` / `services_select_members` como el llamador y el
calendario público dejaría de existir. El fix es el `REVOKE`, dejando
`security_invoker` apagado.

### Reproducido contra la base real (no inferido)

Actor: `anon`, la clave publishable que viaja al navegador. Sin sesión, sin JWT.

| Vector | Resultado |
|---|---|
| `DELETE /rest/v1/services_public?id=eq.<X>` (servicio sin plan) | **Borra.** Cascada `services → schedule_rules → slot_occurrences → bookings`: `Booking` `CONFIRMED` destruida, sin `cancelled_at`, sin `MakeupCredit`, sin `audit_log` |
| `DELETE /rest/v1/services_public?id=eq.<X>` (plan `applies_to_all_services`) | **Borra.** `payments` `PAID` + `payment_service_coverage` destruidos vía `payments.service_id → services CASCADE` |
| `PATCH /rest/v1/organizations_public?id=eq.<otro tenant>` | **Reescribe** `slug`, `name`, `timezone` de una organización ajena |
| `PATCH /rest/v1/services_public` con `organization_id` | **Muda** un `Service` de un tenant a otro |
| `POST /rest/v1/organizations_public` | **Crea una `Organization`**, saltando `create_organization_with_owner()`, el gate de invitación de ADR-0010 y `enforce_plan_limit()` |
| `POST /rest/v1/services_public` | **Crea un `Service`** dentro de cualquier organización |
| `GET /rest/v1/services_public` sin filtro | Enumera los servicios de **todas** las organizaciones: la vista no está scopeada por tenant |
| `DELETE /rest/v1/organizations_public` | Frenado sólo por la FK `organization_invites.redeemed_organization_id` (`NO ACTION`), **no por RLS**. Una org creada por el vector anterior no tiene invite y sí se borra (cascada a todo, `audit_log` incluido) |

Secuestro de slug, encadenando dos de los anteriores: `PATCH` que libera el `slug`
de la víctima + `POST` que lo reclama para una organización nueva del atacante ⇒
`/[slug]` del negocio real resuelve a la organización falsa. Verificado.

Dónde los triggers de negocio frenaron el borrado **por accidente, no por
control**: `SERVICE_PLAN_SCOPE_EMPTY` (`validate_service_plan_scope`, diferido) y
`SERVICE_PLAN_TERMS_IMMUTABLE` (`check_service_plan_services_immutable`) abortan
el `DELETE` cuando la cascada toca `service_plan_services`. Son las dos razones
por las que el borrado masivo en un solo request aborta y por las que un servicio
con plan `PER_SERVICE` y pago sobrevive. No cubren el servicio sin plan, ni el
plan `applies_to_all_services`, ni el `UPDATE`, ni el `INSERT`.

### Fix aplicado (ADR-0037, Fase 36)

```sql
revoke insert, update, delete, truncate on public.organizations_public from anon, authenticated;
revoke insert, update, delete, truncate on public.services_public  from anon, authenticated;
```

Verificado antes de proponerlo, con el mismo método de la Fase 35: el único uso de
ambas vistas en todo `frontend/` es `.select()` en `frontend/app/actions/public.ts`;
no hay un solo `.delete()` contra PostgREST en `frontend/`, y en
`backend/supabase/migrations/` el único `delete from` sigue siendo el de
`slot_occurrences`. El `REVOKE` no rompe ningún camino existente.

Regresión pedida, en dos niveles: (1) los vectores de la tabla, afirmando que la
fila sobrevive y que `anon` **sigue** pudiendo `SELECT` (el comportamiento de
ADR-0008 no se puede perder); (2) un test genérico sobre `pg_views` que afirme,
para **toda** vista de `public`, que `has_table_privilege('anon', v, 'INSERT' |
'UPDATE' | 'DELETE')` es `false` — así la próxima vista pública no repite el
agujero. Ambos validados al revés antes de darlos por buenos.

### Segundo hallazgo: `organization_members` — `DELETE` saltea `LAST_OWNER`

`organization_members_write_owner` es `FOR ALL` con `is_organization_owner()`, la
tabla **no tiene trigger de `DELETE`**, y `revoke_member()` existe justamente para
impedir esto (su comentario: *"An organization with no active OWNER is
unadministrable: nobody could ever add one back, since adding owners is itself
OWNER-gated"*).

Reproducido: `revoke_member()` sobre el único `OWNER` → `LAST_OWNER`.
`DELETE /rest/v1/organization_members?id=eq.<misma fila>` → **204, fila borrada,
organización con cero miembros**. Además destruye el rastro de la baja
(`cancelled_at`/`cancelled_by`/`cancellation_reason`) que el `UPDATE` conserva.

Fix aplicado: `ALL → INSERT` + `UPDATE` con la misma expresión, igual que las
siete tablas de ADR-0036.

### Las otras dos policies `FOR ALL` del schema: seguras, con fundamento

- **`organization_roles`** — `is_organization_owner()`. Hijos `NO ACTION`
  (`organization_members.role_id`, `team_invitations.role_id`), más
  `check_organization_role_not_in_use` (`BEFORE DELETE` → `ROLE_IN_USE`) y
  `check_organization_default_role_present` (diferido → `DEFAULT_ROLE_REQUIRED`).
  Un `DELETE` sólo pasa sobre un rol no-default y sin **ninguna** fila que lo
  referencie: una fila descartable de verdad. Nada en cascada.
- **`service_plan_services`** — `OWNER` vía `service_plans`. Sin hijos.
  `check_service_plan_services_immutable` (`BEFORE DELETE`) bloquea cualquier
  cambio si el plan tiene un pago no-`VOID`. Editar la composición de un plan sin
  pagos es el caso de uso legítimo.

Se recomienda partirlas igual (`ALL → INSERT` + `UPDATE`) como defensa en
profundidad — hoy dependen de triggers, no de la ausencia de la policy — pero es
**bajo**, no bloqueante. **Hecho en el mismo commit** (Fase 36).

### El resto del schema: por qué no hay más nada

- Las 27 tablas tienen RLS habilitada. Ninguna otra tiene policy de `DELETE` ni
  `ALL`: `bookings`, `payments`, `slot_occurrences`, `makeup_credits`,
  `recurring_bookings`, `payment_service_coverage`, `plan_change_requests`,
  `organizations`, `profiles`, `service_plans`, `organization_invites`,
  `platform_admins`, `plans` sólo tienen `SELECT` (± `INSERT`/`UPDATE`).
  `customer_activations`, `platform_contact_requests` y `team_invitations` tienen
  RLS y **cero** policies: todo pasa por RPC `SECURITY DEFINER`.
- `audit_log` y `team_invitations` además no tienen ni el `GRANT DELETE`:
  `revoke insert, update, delete ... on audit_log` (Fase 30, ADR-0032) y
  `revoke all on team_invitations` (Fase 33, ADR-0034). Es el patrón correcto y el
  que la vista pública nunca aplicó.
- `organizations` no tiene policy de `DELETE`, que es lo que importa: su cascada
  llega a **todo**, `audit_log` incluido. El agujero era la vista, no la tabla.
- En las tablas base el `GRANT DELETE` amplio a `anon`/`authenticated` (default
  privileges de Supabase) es inofensivo: RLS es el gate y deniega por default.
  En una **vista** sin `security_invoker` no hay gate. Esa es toda la diferencia.

### Checklist para `security-engineer` (suma a las de las Fases 32, 34 y 35)

- [ ] ¿La feature crea una **vista** en un schema expuesto? Entonces:
      `REVOKE INSERT, UPDATE, DELETE, TRUNCATE` explícito. El `GRANT SELECT` no
      alcanza, y `security_invoker` no siempre se puede encender.
- [ ] ¿Hay un invariante que una RPC defiende con una excepción (`LAST_OWNER`,
      `ROLE_IN_USE`, …)? ¿Se puede llegar al mismo estado con un `DELETE`,
      `UPDATE` o `INSERT` directo por la Data API?
- [ ] Un trigger de negocio que aborta un borrado **no es un control de acceso**:
      frena algunos casos y da falsa sensación de cierre.

## Fase 36 — las vistas públicas dejan de ser escribibles (ADR-0037)

Implementación del barrido de arriba. Migración
`20260923230000_phase36_public_views_read_only.sql`. **Vulnerabilidad crítica
preexistente en producción desde la Fase 4 (ADR-0008, hace meses)**, sin relación
con ninguna ADR del 2026-09-23.

> **Se aplica en dos migraciones.** El cierre de las dos vistas y de
> `organization_members` va en la migración de arriba, que no depende de nada
> posterior a la Fase 13 y se commitea sola. Las dos defensas en profundidad
> (`organization_roles`, `service_plan_services`) viven en
> `20260923240000_phase36b_role_and_plan_scope_delete_closed.sql`, que depende de
> la Fase 32 y se commitea con ella. Detalle del porqué y de los dos modos de
> falla (uno ruidoso, uno silencioso) en `docs/database.md` § Fase 36. La
> superficie cerrada es la misma; cambia en qué orden se aplica.

### Lo que se cerró

```sql
revoke insert, update, delete, truncate on public.organizations_public from anon, authenticated;
revoke insert, update, delete, truncate on public.services_public from anon, authenticated;
```

`SELECT` **no** se toca y `security_invoker` **no** se enciende: la lectura
anónima de esas dos vistas es el calendario público de ADR-0008. `relacl` antes del
fix: `anon=arwdDxtm/postgres`. Después: `anon=rxtm/postgres` — quedan `SELECT`,
`REFERENCES`, `TRIGGER` y `MAINTAIN`, ninguno alcanzable por la Data API (ver
"Fuera de alcance" al final de esta sección).

Las tres policies `FOR ALL` restantes del schema pasan a `INSERT` + `UPDATE` con
la misma expresión, sin `DELETE` — mismo patrón que ADR-0036:

| Tabla | Policy que se reemplaza | Policies nuevas | Expresión (sin cambios) |
|---|---|---|---|
| `organization_members` | `organization_members_write_owner` | `organization_members_insert_owner`, `organization_members_update_owner` | `is_organization_owner(organization_id)` |
| `organization_roles` | `organization_roles_write_owner` | `organization_roles_insert_owner`, `organization_roles_update_owner` | `is_organization_owner(organization_id)` |
| `service_plan_services` | `service_plan_services_write_owner` | `service_plan_services_insert_owner`, `service_plan_services_update_owner` | `exists (select 1 from service_plans sp where sp.id = service_plan_id and is_organization_owner(sp.organization_id))` |

La policy original de `organization_members` no tenía `with check`, así que
Postgres usaba su `using` también como check de `INSERT`: la policy nueva de
`INSERT` lleva exactamente esa expresión y el reparto de permisos no cambia.

### El guardarraíl: `audit_public_view_write_grants()`

Función nueva (`stable`, `EXECUTE` **sólo** para `service_role`). Devuelve una
fila por cada `(vista de public, rol de request, privilegio de escritura)` que
siga concedido; **el resultado correcto es vacío**. Usa `has_table_privilege` y
no `information_schema.role_table_grants` a propósito: resuelve también los
privilegios concedidos a `PUBLIC` y los heredados por pertenencia a otro rol, que
no aparecen como fila de `anon` en el catálogo pero `anon` tiene igual.

Existe porque el default de Supabase va a volver a entregar los cuatro comandos a
la próxima vista que alguien cree en `public`, y la única forma de que eso no pase
otros meses desapercibido es que un test falle solo.

### Dos formas distintas de "denegado", y por qué los tests afirman las dos

| Mecanismo | Qué responde PostgREST | Cómo se ve en el test |
|---|---|---|
| Falta el privilegio de tabla (las dos vistas) | Error `42501` (`insufficient_privilege`) | `expect(error!.code).toBe("42501")` **y** la fila sobrevive |
| Falta la policy de RLS (las tres tablas) | `204` / `[]`, **sin error** | `expect(error).toBeNull()`, cero filas afectadas **y** la fila sobrevive |

En los dos casos el test afirma que la fila sigue existiendo, mirada con el
cliente de `service_role`. Afirmar sólo el código de error daría un falso verde el
día que alguien reabra la puerta por el otro mecanismo.

### Regresión

`backend/test/phase36.public-views-read-only.test.ts`, **17 casos**, más
`backend/test/phase36b.role-and-plan-scope-delete-closed.test.ts`, **3 casos**
(los de `organization_roles` y `service_plan_services`, que se commitean con la
Fase 32). Lo primero
que corre no es un ataque: son los tres casos que afirman que `anon` **sigue**
leyendo `organizations_public`, `services_public` y `get_public_availability()`.
Si eso se pone rojo, el fix está mal (es exactamente el síntoma de haber puesto
`security_invoker = on`) y hay que volver atrás, no ajustar el test.

Después: los seis vectores anónimos (`DELETE` de un servicio **sin plan** con una
`Booking` `CONFIRMED` abajo; `DELETE` de un servicio cubierto **sólo** por un plan
`applies_to_all_services` con un `Payment` `PAID` abajo; `PATCH` y `POST` sobre las
dos vistas; el secuestro de slug encadenado), el mismo intento con un usuario
**autenticado sin membresía**, el test genérico sobre `pg_views`, el `OWNER`
intentando borrar su propia fila de `organization_members` (con `revoke_member()`
todavía devolviendo `LAST_OWNER` y todavía funcionando sobre un `STAFF`), y el
contrapeso de que `SELECT`/`INSERT`/`UPDATE` siguen funcionando en las tres
tablas — los dos últimos casos, sobre `organization_roles` y
`service_plan_services`, en el archivo de la Fase 36b.

El detalle de los dos servicios distintos no es decorativo: un servicio cubierto
por un plan `PER_SERVICE` sobrevivía al `DELETE` anónimo **por accidente**
(`validate_service_plan_scope` / `check_service_plan_services_immutable` abortaban
la transacción). Un test que sólo mirara ese caso mediría la casualidad y no el
agujero.

**Validación al revés** (mismo método que la Fase 35): con una migración temporal
que devolvía los `GRANT` de escritura y las tres policies `FOR ALL`, **16 de los
19 casos fallan** (contados sobre el archivo único, antes de partirlo en 36 + 36b)
— y los tres que pasan son justamente los de lectura anónima,
que tienen que pasar en los dos estados. En ese estado vulnerable, los dos
`DELETE` anónimos críticos devuelven `error === null`: borran de verdad.

### Fuera de alcance, anotado

`REFERENCES`, `TRIGGER` y `MAINTAIN` siguen concedidos a `anon`/`authenticated`
sobre las dos vistas (también vienen de los default privileges). No son
alcanzables por la Data API — PostgREST no emite DDL ni comandos de
mantenimiento, y `anon` no tiene `CREATE` en `public` — así que quedan fuera de
ADR-0037 en vez de ampliarlo sin decisión. Pendiente de una mirada de
`security-engineer`: si se decide cerrarlos, el statement es
`revoke references, trigger, maintain on ... from anon, authenticated` y no
necesita tocar nada más.

### Nota operativa (de ADR-0037)

Como la vulnerabilidad ya estaba en producción, corresponde revisar `audit_log` y
los conteos de `organizations`/`services` contra lo esperado **una vez aplicado el
fix**, para descartar que haya sido explotada antes de encontrarla. La `anon key`
es pública por diseño: no hay secreto que rotar.

## Fases 37/37b/38 — niveles de acceso verificados (2026-09-25)

Ninguna policy nueva ni tocada (las tres migraciones no contienen `create/alter/drop
policy` ni cambios de `row level security`). Niveles verificados contra base local
(`pg_proc.proacl` = `{postgres, authenticated, service_role}`, sin `PUBLIC` ni `anon`;
`anon` recibe `42501`), y con una prueba cross-org con ids reales:

| Función | Nivel | Cómo se resuelve la identidad |
|---|---|---|
| `my_payments()` (reescrita 2 veces por `drop`+`create`) | **CUSTOMER** — cada migración re-aplica `revoke from public, anon` + `grant to authenticated` (ADR-0028) | **solo `auth.uid()`**: `join customers c on c.id = p.customer_id and c.profile_id = auth.uid()`, idéntico a la Fase 15. Un OWNER que no es `Customer` recibe `[]`. Columnas nuevas (`plan_name`, `plan_kind`, `weekly_quota`, `plan_applies_to_all_services`, `currency`) son del plan que el propio cliente compró; mismo nivel de disclosure que `public_service_plans()` (anon). `payments_same_org`/`payments_plan_consistency` garantizan que el plan joineado es de la misma organización que el pago. |
| `recurring_booking_occurrences(uuid)` | **ADMIN** — `is_organization_member(rb.organization_id)` dentro del `where`; `revoke from public, anon` + `grant to authenticated` | membership sobre `recurring_bookings.organization_id` (el trigger `recurring_bookings_same_org` ata ese valor al del `Customer` y la `ScheduleRule`). OWNER y STAFF de otra organización con el id real de una serie ajena reciben `[]`; el propio `Customer` de la serie también recibe `[]` (no es vista de cliente). |

**Regla que queda escrita:** una RPC ADMIN que recibe un id de fila hija (serie,
booking, pago) gatea con `is_organization_member()` sobre el `organization_id` de
**esa fila** (no sobre el slug que manda el server action). El
`requireOrganizationMembership(slug)` del server action es UX, no el gate: un
miembro de A y B que manda slug A con una serie de B ve la serie de B porque es
miembro legítimo de B — no es fuga, pero el frontend no debe asumir que el
resultado pertenece al slug.

## Suite E2E contra producción (ADR-0039) — reglas de credenciales

La suite Playwright de `frontend/e2e/` opera producción real con una sesión real.
Reglas (auditoría security-engineer, 2026-09-25):

1. **La contraseña QA nunca se tipea dentro de un test reportado.** Verificado
   empíricamente con Playwright 1.63: `locator.fill(value)` deja `value` en claro en
   (a) el título del step del reporte HTML (`Fill "<valor>"`), **para tests que
   pasan también**; (b) los params de la acción y los snapshots DOM
   (`__playwright_value_`, incluso en `type="password"`) del `trace.zip`; y (c) el
   snapshot ARIA de `error-context.md` (`textbox "Contraseña": <valor>`). El
   reporte se sube como artifact de CI. El login con la credencial real se hace
   una sola vez en `globalSetup` (fuera del reporter y sin trace) y los tests
   reusan `storageState`; el archivo de `storageState` es un token de sesión: va
   a una ruta gitignoreada y **nunca** dentro del artifact subido.
2. **Secrets a nivel de step, no de job**: `QA_EMAIL`/`QA_PASSWORD` solo en el
   `env:` del step que corre Playwright; el job con `permissions: contents: read`.
3. **Cuenta QA dedicada, de mínimo privilegio — propuesta, evaluada y
   rechazada por el usuario (Orchestrator, 2026-09-25)**: se le presentó la
   opción de crear una cuenta que sea OWNER únicamente de `redentor`, sin ser
   platform admin ni pertenecer a otras organizaciones, para acotar el radio
   de impacto de una fuga de credenciales a solo datos de prueba. El usuario
   eligió explícitamente seguir usando su cuenta personal real
   (`credendor@gmail.com`, dueño real de `redentor`) como cuenta QA. **Riesgo
   aceptado**: si `QA_PASSWORD` se filtra por cualquier vector (este ya
   cerrado por la regla 1, u otro futuro), el radio de impacto es el de esa
   cuenta completa — no solo `redentor` — y depende de a qué otras
   organizaciones pertenezca o si tiene privilegios de plataforma. Esto hace
   que la regla 1 (nunca tipear la contraseña real en un test reportado) sea
   todavía más importante de mantener, no menos: es la única barrera real
   que queda contra ese escenario.
4. Si se sospecha que un artifact con la credencial llegó a subirse: borrar el
   artifact y **rotar la contraseña**.

## Fase 40 — nonce de continuación de activación (ADR-0040) — reglas

Migración: `20260928130000_phase40_activation_continuation.sql`. Gate de
security-engineer del 2026-09-28: primera pasada **NO LISTO** (reglas 2 a 4);
segunda pasada (mismo día, sobre la migración corregida, `db reset` + 9/9
tests + ataque manual con psql) **LISTO**.

1. **El nonce es un portador equivalente al token de activación durante su
   vida (30 min).** `claim_customer_activation()` **no** compara el email de
   la sesión con nada: un cliente gestionado sólo tiene `display_name` +
   `phone`, y el token es "ser este cliente" para cualquier cuenta que lo
   presente (ver la sección de ADR-0026). Verificado en vivo: nonce →
   `redeem_activation_continuation()` → token → una cuenta **ajena** canjea
   y queda como `customers.profile_id`. Todo razonamiento de la forma "el
   nonce no alcanza porque el claim exige el email del invitado" es falso.
   El nonce se protege igual que el token: nunca en logs propios, nunca en
   `Referer` (`Referrer-Policy: no-referrer` en las páginas que lo llevan en
   la URL), y `/auth/callback` lo saca de la URL en el redirect.
2. **El token nunca se guarda en claro en `customer_activation_continuations`.**
   Se guarda cifrado con el propio nonce como clave
   (`extensions.pgp_sym_encrypt(token, nonce, 'cipher-algo=aes256')`); la
   base sólo tiene `sha256(nonce)`, así que un dump, backup o un grant/policy
   futuro mal puesto no entrega tokens vivos. `redeem_*` descifra con
   `p_nonce` y pone el cifrado en `null` en el mismo `update` que `used_at`.
3. **Nonces vivos acotados por activación**: `issue_*` bloquea la fila de
   `customer_activations` (`for update`), borra las continuaciones vencidas
   sin usar de esa activación y rechaza con `TOO_MANY_CONTINUATIONS` si ya
   hay 10 sin usar.
4. **`redeem_*` no devuelve el token de una activación que ya no está viva**
   (revocada, canjeada o vencida): `INVALID_CONTINUATION` genérico.
5. **Grants**: `anon`/`authenticated` sólo tienen `EXECUTE` en las dos RPC
   (`security definer`, owner `postgres`). Sobre la tabla, `revoke all ...
   from anon, authenticated` explícito además de RLS con cero policies
   (verificado: `set role anon`/`authenticated` → `permission denied`; las
   RPC siguen funcionando para `anon` vía PostgREST). `service_role` conserva
   `ALL` por default privileges de Supabase y hace bypass de RLS — aceptable
   porque la key nunca sale del servidor, y el contenido está cifrado.
6. **Chequeo de "activación viva" en `redeem_*` sin `for update`: correcto,
   no es carrera explotable.** Si la activación se revoca/canjea entre esa
   lectura y el commit de `redeem_*`, lo único que sale es el token, que por
   sí solo no hace nada: `claim_customer_activation()` vuelve a leer la
   activación con `for update` y re-chequea `revoked_at`/`redeemed_at`/
   `expires_at` antes de escribir. Regla: **la autoridad sobre el estado de
   la activación es siempre `claim_customer_activation()` bajo lock**; los
   chequeos previos (issue/redeem) son fail-fast, no la garantía. Si algún
   día un camino devuelve algo más que el token (p. ej. crea sesión o
   vincula el customer directamente), ese camino sí necesita `for update`.
7. **Cifrado verificado**: paquete OpenPGP SKESK con AES-256 y S2K iterado
   con sal (`c3 0d 04 09 03 02 …`). Con 10 nonces emitidos y vencidos sin
   canjear: ninguna fila contiene el token en claro, y `pgp_sym_decrypt` con
   `nonce_hash` (hex/base64) o clave vacía falla. PostgREST pasa `p_token`
   como `$1`, así que `pg_stat_statements` no captura el literal. Las filas
   vencidas de una activación que nunca vuelve a emitir quedan como
   ciphertext irreversible (sólo se purgan en el próximo `issue_*` de la
   misma activación o por `on delete cascade`).
8. Riesgo residual aceptado: si la persona tipea mal su email al
   registrarse, el mail de confirmación (con el nonce en `redirect_to`) llega
   a un tercero, que durante 30 min puede leer el nonce del link. Con PKCE
   (default de `@supabase/ssr`) el clic directo del tercero **falla** en
   `exchangeCodeForSession` (no tiene el `code_verifier`), así que no es
   "un clic": tiene que extraer el nonce y armar a mano su propio login
   (p. ej. Google) con `returnTo=/activar/continuar?c=<nonce>`. Mitigan el
   TTL, el single-use y el clic explícito de "Confirmar y activar".
   **Superado por ADR-0041** (ver sección siguiente): con `token_hash` la
   fricción de PKCE desaparece y el tercero sí llega con un clic hasta la
   pantalla "Confirmar y activar".
9. **Frontend (tercera pasada del gate, 2026-09-28, LISTO en seguridad):**
   - `/auth/callback` calcula el destino sin `c` **antes** de cualquier
     llamada de red (`exchangeCodeForSession`/`redeem_*`); ninguna salida
     (éxito, nonce inválido, error de auth, excepción → 500 sin `Location`)
     puede llevar `c=`. El destino se re-valida con `safeReturnTo()` después
     de pasar por `new URL()` (la normalización de dot-segments puede
     producir `//host`), y se sigue prefijando con `siteUrl()` como string.
   - La cookie replantada usa literalmente `activationCookieOptions()` (misma
     que `/activar/[token]`), con `Cache-Control: no-store` en esa respuesta.
   - `safeReturnTo()` no cambió: `?c=` no amplía la superficie (probado con
     `//`, `\`, `%5C`, `%2F%2F`, dot-segments, `@`, `javascript:`, tab/LF,
     `∕`/`／`, fragmento): todo lo que se rechazaba sin `c` se sigue rechazando.
   - `Referrer-Policy: no-referrer` en `/activar/continuar`, `/login` y
     `/signup` sólo con `?c=` (directo o en `returnTo` con un nivel de
     encoding, que es lo único que la app genera). Falsos negativos con
     `returnTo` doble-encodeado o anidado no son alcanzables por la app, y
     aun así el default de Caddy (`strict-origin-when-cross-origin`) no manda
     path ni query cross-origin. Sin `?c=` el header de esas rutas no cambia.
   - Regla: el nonce sólo se canjea en `/auth/callback` tras una sesión
     creada en ESE request. Si se agrega otro punto de canje (p. ej.
     `/activar/continuar` con sesión ya existente), tiene que exigir sesión,
     sacar `c` de la URL y replantar con `activationCookieOptions()`.
   - Desde ADR-0041 el cálculo del destino sin `c` vive en
     `lib/activation-continuation.ts` y corre **después** de crear la sesión,
     no antes. El invariante que importa se mantiene (ninguna salida lleva
     `c=`: el camino de error redirige a `/login?error=...` sin `next`, y una
     excepción da 500 sin `Location`), pero cualquier cambio futuro que
     agregue `next`/`returnTo` a la redirección de error tiene que pasar
     primero por el helper.

## Confirmación de email por `token_hash` (ADR-0041) — reglas

Gate de security-engineer del 2026-09-29. Verificado en vivo contra Supabase
local + `next build`/`next start` apuntando a local (scripts en scratchpad,
no en el repo).

1. **`/auth/confirm` es un segundo punto de canje del nonce** (el primero es
   `/auth/callback`). Ambos usan `redeemActivationContinuationIfPresent()`
   y sólo después de crear sesión en ESE request (`verifyOtp` /
   `exchangeCodeForSession` sin error). Mismas reglas que la Fase 40 §9:
   `c` nunca en el `Location`, cookie con `activationCookieOptions()`,
   `Cache-Control: no-store`, nonce/token_hash nunca logueados.
2. **`next` en `/auth/confirm` llega ABSOLUTO.** El template usa
   `next={{ .RedirectTo }}`, y `RedirectTo` es el `emailRedirectTo` que arma
   `signUpWithPassword()`: `${siteUrl()}/auth/callback?next=<returnTo>`.
   `safeReturnTo()` lo rechaza (no empieza con `/`) y cae a `/dashboard`.
   Regla: `/auth/confirm` sólo puede desenvolverlo si `new URL(next).origin
   === new URL(siteUrl()).origin` **y** `pathname === "/auth/callback"`,
   tomando su `next` interno y pasándolo por `safeReturnTo()`; cualquier
   otra forma absoluta cae al fallback. Nunca se redirige a una URL absoluta
   recibida por query, ni se relaja `safeReturnTo()` para aceptar absolutas.
3. **`type` con allowlist** (implementado): `/auth/confirm` sólo acepta
   `type === "email"` estricto (el único que genera el template de
   confirmación) — cualquier otro valor (`recovery`, `magiclink`,
   `email_change`, `invite`, ausente) cae al mismo fallback de error sin
   llamar a `verifyOtp`, así que la ruta ya no funciona como verificador
   OTP genérico. Verificado en el código (`app/auth/confirm/route.ts`) y
   con `type=recovery` tampeado sobre un `token_hash` real: rechaza sin
   consumir el token.
4. **TTL real del link**: `otp_expiry` (`GOTRUE_MAILER_OTP_EXP`), 3600 s en
   local — verificado en vivo (`confirmation_sent_at` retrasado 59 min →
   sesión; 61 min → `403 otp_expired`). Single-use verificado (segundo clic
   → `/login?error`). **Producción: confirmar en el dashboard** (Auth →
   Providers → Email → "Email OTP Expiration"); no debe superar 3600 s.
5. **Riesgo residual aceptado (reemplaza el #8 de la Fase 40): email mal
   tipeado.** El tercero que recibe el mail entra con un clic, con sesión,
   y aterriza en `/activar/continuar` con la cookie de activación ya
   plantada; si toca "Confirmar y activar" queda como dueño del `Customer`
   ajeno. Lo que NO es riesgo nuevo: la cuenta en sí (el dueño del buzón ya
   controla esa cuenta vía recuperación de contraseña, con o sin PKCE). La
   ventana efectiva es la del **nonce (30 min desde que se emitió en
   `/activar/continuar`)**, no la del OTP: pasado ese lapso el link sigue
   confirmando la cuenta pero ya no planta la cookie. Mitigan: TTL 30 min,
   single-use del nonce y del `token_hash`, clic explícito de "Confirmar y
   activar" con el email de la sesión a la vista, y que la persona real ve la
   invitación como ya usada. Es el riesgo que la decisión de ADR-0040 ya
   aceptó por escrito ("con un solo clic").
6. **Login-CSRF (nuevo con `token_hash`, aceptado):** un atacante puede
   mandarle a la víctima el link de confirmación de SU PROPIA cuenta sin
   confirmar; al abrirlo, la víctima queda logueada como el atacante (PKCE lo
   impedía). Si la víctima después abre su link de activación en ese
   navegador, `/activar/continuar` muestra "Vas a activar esta invitación con
   la cuenta <email del atacante>" y "No soy yo — usar otra cuenta": la
   pantalla de confirmación explícita (ADR-0026, confused-deputy) es la
   mitigación y **no se puede quitar ni automatizar**.
7. **Prefetch de links (recomendado, no bloqueante):** `/auth/confirm`
   consume `token_hash` Y nonce en un GET. Un escáner de links (Outlook Safe
   Links, gateways corporativos) que siga el link quema los dos: la persona
   real recibe error y pierde la continuidad. Con PKCE el escáner quemaba el
   token de confirmación pero no el nonce (el canje de código fallaba antes).
   Mitigación mínima: que el GET de `/auth/confirm` renderice una página con
   un botón que haga POST (server action) y recién ahí llame a `verifyOtp`.
8. **Orden de despliegue**: (a) migración de la Fase 40; (b) frontend con
   `/auth/confirm` (con la regla 2 implementada); (c) recién después, el
   template en el dashboard. El `Site URL` de Supabase tiene que ser
   idéntico a `NEXT_PUBLIC_SITE_URL` (el link usa `{{ .SiteURL }}` y la
   sesión se planta en ese host; si difiere, p. ej. `www` vs. apex, la
   cookie queda en otro host que el del redirect). **Nunca `supabase config
   push`** contra producción: subiría `site_url`/redirects de local.

## Fase 41 — ADR-0043 sin confirmación de email: reglas

Gate de `security-engineer` (segundo pase, 2026-09-29), verificado en vivo
contra Supabase local (GoTrue con `MAILER_AUTOCONFIRM=true`,
`SECURITY_CAPTCHA_ENABLED=true`, `SECURITY_MANUAL_LINKING_ENABLED=false`).

1. **El email de `auth.users` es un factor de *conocimiento*, nunca de
   posesión.** Ninguna RPC, policy ni flujo nuevo puede otorgar membresía,
   vínculo de `Customer` ni ningún acceso por "la sesión tiene el email X".
   La autorización para sumar a alguien a un tenant es **tener un token de
   un solo uso** emitido por el tenant (`claim_team_invitation()`,
   `claim_customer_activation()`, código de `organization_invites`). Los
   chequeos de email que quedan (`INVITE_WRONG_EMAIL` en
   `claim_team_invitation()` y en `create_organization_with_owner()`) son
   defensa en profundidad contra un link mal enviado, no una barrera: se
   aceptan así porque el token/código sigue siendo el factor de posesión.
2. **`invite_member_by_email()` y `enroll_customer_by_email()` sin
   `EXECUTE` para nadie** salvo su dueño (`postgres`). Verificado:
   `proacl = {postgres=X/postgres}`, sin `PUBLIC`. Ninguna función de la
   base las invoca (el único match en `pg_proc.prosrc` es un comentario de
   `claim_team_invitation()`), ni `cron.job`, ni `frontend/`. Reotorgarlas
   requiere ADR.
3. **Trigger `auth_identities_block_oauth_hijack`** (`before insert on
   auth.identities`, `public.check_identity_link_not_oauth_hijack()`,
   `security invoker`, `search_path = ''`): rechaza insertar una identidad
   no-`email` para un `user_id` que ya tiene identidad `email`. Corre como
   `supabase_auth_admin` (dueño de `auth.identities`) — verificado con
   `set role supabase_auth_admin`: el insert de secuestro falla con
   `OAUTH_LINK_BLOCKED_EXISTING_PASSWORD_IDENTITY`; alta Google nueva y el
   `UPDATE` de identidad de logins Google posteriores pasan. No afecta a
   cuentas Google-only preexistentes (sólo `INSERT`, y no tienen identidad
   `email`). El sentido inverso ya lo cubre GoTrue: `signUp()` por
   contraseña sobre un email de cuenta Google-only devuelve
   `user_already_exists` (422) sin crear identidad ni contraseña
   (verificado). **Precondición:** "manual linking" apagado en producción —
   si alguna vez se habilita `linkIdentity()`, este trigger lo bloquea para
   cuentas con contraseña y hay que rediseñarlo (ADR). Falta un test de
   regresión en `backend/test/` que afirme el bloqueo.
4. **Captcha (Turnstile) es enforcement de GoTrue, no del frontend.** El
   widget y el chequeo de `captchaToken` vacío en `app/actions/auth.ts` son
   UX; la barrera real es `[auth.captcha]`. GoTrue lo exige en `/signup` y
   en `/token?grant_type=password` (verificado: `captcha_failed` en ambos
   sin token). Reglas: (a) nunca cargar en producción el secreto de prueba
   `1x0000000000000000000000000000000AA` — convierte el captcha en no-op;
   (b) nunca habilitar captcha en Supabase de producción mientras el bundle
   desplegado tenga la Site Key de prueba (`1x00000000000000000000AA`) — sus
   tokens dummy no validan contra un secreto real y rompe todo login por
   contraseña; (c) la Site Key real tiene que tener cargados en Cloudflare
   todos los hostnames de producción, o el widget no emite token y el botón
   queda deshabilitado; (d) un token de Turnstile es de un solo uso: el
   widget tiene que resetearse después de cada submit fallido.
5. **Sin captcha:** `/authorize` (Google OAuth) y `updateUser()`. El cambio
   de email vía `updateUser({ email })` sigue exigiendo confirmación
   (`new_email` queda pendiente, verificado) — no es un camino de ocupación
   de email, pero sí manda mails: cualquier UI futura de cambio de email
   entra en conflicto con ADR-0043 y requiere decisión.
6. **Riesgo residual aceptado:** ocupación de email (DoS): quien se registra
   primero con el email de otra persona la deja sin poder usar ese email
   (ni por contraseña ni por Google — el trigger la bloquea). No hay flujo
   de recuperación; se resuelve por soporte a mano (auditoría de ADR-0043
   punto 3). El callback OAuth descarta el error de GoTrue y redirige a
   `/login` sin mostrar texto crudo de Postgres (verificado en el código).

## Fase 44 — cobrar un turno suelto (ADR-0046): reglas

Gate de `security-engineer` sobre `quote_booking()` / `book_slot_paying()`
(`backend/supabase/migrations/20260930140000_phase44_drop_in_booking.sql`).
Tres hallazgos reproducidos contra `reservaste-stg` y corregidos en la
migración (sin commitear al momento del gate; hay que re-aplicar las dos
funciones en stg):

1. **Resolver un plan siempre filtra por la organización del servicio.**
   `resolve_active_drop_in_plan()` no filtraba `organization_id`: un DROP_IN
   `applies_to_all_services=true` de *cualquier* tenant matcheaba todos los
   servicios de la plataforma. Reproducido: el precio del plan de la org B
   aparecía en `agenda_occurrences()`/`quote_booking()` de la org A, y
   `book_slot_paying()` de la org A abortaba con `Payment organization_id
   must match its ServicePlan` (DoS del cobro para todos los tenants,
   disparable por cualquier org). **Regla:** todo helper que resuelva
   `service_plans` por `applies_to_all_services` tiene que acotar
   `sp.organization_id` al del servicio. El mismo defecto existe
   *preexistente* en el `exists` de `SERVICE_HAS_NO_PLAN` de
   `evaluate_payment_coverage()` (Fase 22) — abierto, ver abajo.
2. **Una RPC `security definer` que escribe un `Payment` exige
   `MANAGE_PAYMENTS`** — el mismo permiso que la policy
   `payments_insert_staff` (Fase 32). `book_slot_paying()` sólo pedía
   `MANAGE_BOOKINGS`: reproducido, un rol configurado sin permiso de cobro
   registraba un `PAID` con `p_amount=0` y esquivaba el `PAYMENT_REQUIRED`
   que `admin_book_for_customer()` le devuelve al mismo rol. Ahora:
   `MANAGE_PAYMENTS` siempre, y además `MANAGE_BOOKINGS` sólo si la función
   tiene que *crear* la `Booking` (regla de la Fase 34: el cajero cobra lo
   ya anotado, no anota).
3. **`amount` nunca es `NaN`.** `'NaN'::numeric < 0` es falso y
   `numeric(12,2)` acepta `NaN`; reproducido un `PAID` con amount `NaN`.
   `book_slot_paying()` devuelve `INVALID_AMOUNT` para `NaN`/negativos.

Verificado sin cambios (con sondas reales contra stg): anon no ejecuta
ninguna de las cuatro RPC (`permission denied`); el cliente no puede
invocar `book_slot_paying()`; `quote_booking()` resuelve la autorización
antes de leer la fila del `Customer` (un `customer_id` ajeno y uno
inexistente dan la misma respuesta, sin oráculo); el staff de A contra una
ocurrencia de B da `NOT_AUTHORIZED`, y con ocurrencia propia + cliente
ajeno da `CUSTOMER_NOT_IN_ORG`; concurrencia: 8 cobros simultáneos del
mismo cliente → 1 `OK` + 7 `ALREADY_PAID`, 6 clientes distintos sobre
capacidad 1 → 1 `OK` + 5 `SLOT_FULL`, sin sobre-reserva ni doble cobro.

**Doble cobro en servicio gratuito:** `book_slot_paying()` sólo re-evalúa
cobertura si `payment_required=true`. En un servicio gratuito la defensa es
triple, no sólo el índice: el `FOR UPDATE` de la ocurrencia serializa las
llamadas concurrentes, `payments_one_paid_per_occurrence_idx` rechaza un
segundo `PAID` (`ALREADY_PAID`) y `check_payment_no_duplicate()` rechaza
cualquier otro pago no-`VOID` de la misma ocurrencia.

**Abierto:** `evaluate_payment_coverage()` (Fase 22) decide
`SERVICE_HAS_NO_PLAN` vs `PAYMENT_REQUIRED` con un `exists` sobre
`service_plans` sin filtro de organización: un plan `applies_to_all_services`
activo de otra org cambia el `reason` que ven los tenants sin planes. No
filtra datos ni habilita reservas (ambos son "no"), pero es influencia
cross-tenant sobre el motor de cobertura; requiere fix en la cadena
ADR-0018 (decisión del Orchestrator).

## Fase 45 — reserva abierta (ADR-0047): reglas

Gate de `security-engineer` (2026-09-30) sobre
`backend/supabase/migrations/20260930150000_phase45_open_booking.sql`,
verificado en vivo contra `reservaste-stg` (fuente viva de `book_slot()` y
`can_customer_book()` idéntica al archivo).

1. **Crear identidad de negocio (`Customer`) sólo desde una RPC `VOLATILE`
   de reserva real, nunca desde una función de lectura.** Hoy los únicos
   escritores de `customers` en `pg_proc` son `enroll_customer_by_email()`,
   `create_managed_customer()` y `book_slot()`. `can_customer_book()`,
   `can_customer_book_detail()`, `evaluate_customer_booking()`,
   `preview_recurring_booking()` y `quote_booking()` son `STABLE`
   (Postgres rechaza un `INSERT` ahí) — verificado además en vivo: 0 filas
   antes/después con el flag prendido. Convertir cualquiera de ellas en
   `VOLATILE` requiere gate.
2. **`organization_id` del alta sale de la ocurrencia ya lockeada**
   (`v_occurrence.organization_id`), nunca de un parámetro. La identidad es
   `auth.uid()`; **nunca** se busca ni se vincula un `Customer` por email
   (regla 1 de la Fase 41).
3. **Una baja del staff no se revierte por autoservicio.** Si existe
   *cualquier* fila `(organization_id, profile_id)` —activa o inactiva— no
   hay alta ni `OK_OPEN_BOOKING`: `NOT_A_CUSTOMER` como siempre.
4. **Toda RPC que escribe y después puede devolver un status de rechazo
   tiene que rechazar con excepción, no con `RETURN`.** Un `RETURN` no
   deshace lo escrito antes en la misma transacción. Patrón:
   `raise exception '<ABORT>' using detail = <status>` dentro de un bloque
   `BEGIN … EXCEPTION WHEN OTHERS` que devuelve el status sólo si
   `sqlerrm = '<ABORT>'` y re-lanza (`raise;`) todo lo demás
   (`MAKEUP_CREDIT_RACE_LOST` incluido). Verificado: con el flag apagado o
   un `Customer` preexistente, `book_slot()` devuelve exactamente lo mismo
   que la versión de la Fase 20 en 19 escenarios (OK, crédito de recupero,
   SLOT_FULL, ALREADY_BOOKED, PAYMENT_REQUIRED, ocurrencia
   cancelada/bloqueada/pasada/inexistente, servicio inactivo, cliente
   inactivo, sin cliente); `MAKEUP_CREDIT_RACE_LOST` forzado sale como
   `P0001` igual que antes. Toda salida con status tras un alta deja 0
   filas en `customers`.
5. **Concurrencia del alta:** dos transacciones del mismo `auth.uid()` en
   ocurrencias distintas → la segunda espera el `INSERT` de la primera y
   reusa su fila (`unique_violation` + re-select); si la primera hace
   rollback, la segunda crea la suya. Misma ocurrencia → serializa en el
   `FOR UPDATE`, la segunda ve `ALREADY_BOOKED`. Verificado en vivo.

6. **Tope de reservas `SELF_SERVICE`:** máximo 2 `Booking` `CONFIRMED` con
   `start_at >= now()` mientras `customers.source = 'SELF_SERVICE'`,
   contado *después* de lockear la fila del `Customer`.
   `create_recurring_booking()` rechaza `SELF_SERVICE` de entrada
   (`SELF_SERVICE_CANNOT_CREATE_RECURRING`). Sólo el staff con
   `MANAGE_CUSTOMERS` puede pasar `source` a `STAFF` (el cliente no puede:
   no existe policy de UPDATE propia, verificado 0 filas). Los caminos de
   staff (`admin_book_for_customer`, `admin_create_recurring_booking`,
   `book_slot_paying`) no aplican el tope: es una decisión del staff.
7. **Orden y modo de locks en `book_slot()`:** ocurrencia (`FOR UPDATE`)
   → advisory de organización → advisory de `auth.uid()` → fila del
   `Customer`. El lock sobre `customers` es **`FOR NO KEY UPDATE`, nunca
   `FOR UPDATE`**: `FOR UPDATE` choca con el `FOR KEY SHARE` que toma todo
   FK check hacia `customers`, y `admin_create_recurring_booking()` toma
   ese `KEY SHARE` (insert en `recurring_bookings`) *antes* de lockear
   ocurrencias: orden inverso, deadlock reproducido en vivo (segundo pase
   del gate, 2026-09-30). `NO KEY UPDATE` sigue chocando consigo mismo y
   con el `UPDATE` de `source`, así que serializa lo mismo. Regla general:
   un lock de fila nuevo sobre una tabla referenciada por FKs usa
   `FOR NO KEY UPDATE` salvo que haga falta bloquear inserts de hijos.
8. **Errores del alta:** `PLAN_LIMIT_REACHED*` / `SUBSCRIPTION_INACTIVE`
   del trigger de planes se devuelven como
   `ORGANIZATION_NOT_ACCEPTING_NEW_CUSTOMERS`, sin conteo (verificado por
   pg y por PostgREST). Rate limits de alta, serializados con
   `pg_advisory_xact_lock(int,int)`: 10/org/hora, 30/org/24h, 5 por
   `auth.uid()`/24h (global). Verificado exacto bajo concurrencia real y
   forzada.

**Cerrado en el segundo pase del gate (2026-09-30)** — los tres puntos
siguientes quedan como historial; ver reglas 6-8.

- **Un `Customer` `SELF_SERVICE` no tiene tope de reservas.** Una sola
  cuenta descartable (el alta de cuenta es sólo captcha, Fase 41) puede
  reservar todas las ocurrencias futuras de un servicio sin pago, y además
  llamar `create_recurring_booking()` sobre cada regla. Los rate limits
  del alta no acotan esto. Recomendado: tope de `Booking`s `CONFIRMED`
  futuras mientras `source = 'SELF_SERVICE'` (serializado con un lock por
  cliente), y `create_recurring_booking()` vedado a `SELF_SERVICE`; el
  staff "verifica" pasando `source` a `STAFF` (la policy
  `customers_update_staff` ya lo permite).
- **Las altas `SELF_SERVICE` consumen el `max_customers` del plan del
  tenant** (`enforce_plan_limit`, starter = 50 = el tope horario): en una
  hora se agota el cupo de clientes y el staff no puede dar de alta a
  nadie. Además, llegado el límite `book_slot()` propaga crudo
  `PLAN_LIMIT_REACHED: clientes (50/50)` a cualquier cuenta autenticada
  (filtra el conteo del tenant); lo mismo `SUBSCRIPTION_INACTIVE`.
- **Los rate limits no están serializados**: N llamadas concurrentes leen el
  mismo conteo (verificado: 3 concurrentes con 4 previas → 7 filas contra
  un tope de 5). Acotado por concurrencia; cerrar con
  `pg_advisory_xact_lock` por `auth.uid()` y por organización antes de
  contar.

## Fase 49 — disponibilidad pública por recurso (ADR-0048): regla de disclosure

Gate liviano de `security-engineer` (así lo pide la propia ADR-0048, por ser
una RPC `anon`) sobre
`backend/supabase/migrations/20260930190000_phase49_public_resource_availability.sql`.
`get_public_availability()` (pública, sin login) agrega `resource_id`
(siempre) y `resource_name` (sólo si `organizations.public_resource_names`,
opt-in nuevo, `default false`).

1. **No se filtra ninguna otra columna de `resources`.** El `select` final
   sólo lee `r.name`, envuelto en
   `case when v_org.public_resource_names then r.name else null end` —
   nunca `r.*`, `description`, `is_exclusive` ni `capacity`.
2. **El flag se lee del mismo `v_org` que ya filtra las filas**, resuelto
   una sola vez arriba por `p_organization_slug` — no hay un segundo
   lookup a `organizations` ni una organización compartida entre filas.
   Por construcción (el `resource_id` de una `SlotOccurrence` siempre sale
   de la `ScheduleRule` que la generó, y el trigger `schedule_rules_same_org`
   de la Fase 3 + la validación de la Fase 48 garantizan que ese recurso es
   de la misma organización), una organización con el flag en `false` nunca
   puede terminar mostrando el nombre de un recurso de otra organización
   con el flag en `true`.
3. **El `join public.resources r on r.id = so.resource_id` es seguro**:
   `slot_occurrences.resource_id` es `not null` con FK `on delete cascade`
   (Fase 3) — el `inner join` no cambia el conteo de filas que la función
   ya devolvía antes de este cambio, mismo patrón ya usado sin filtrar por
   `resources.is_active` en `agenda_occurrences()` (Fase 8, el equivalente
   de staff que la propia ADR cita como precedente).
4. **Grants idénticos** a la versión anterior (`anon, authenticated`,
   misma firma de 4 parámetros) — `drop` + `create`, nunca `create or
   replace`, porque agregar columnas al `returns table` cambia los OUT
   parameters (mismo patrón ya usado en Fases 5/16/20).

Verificado con tests nuevos contra `reservaste-stg`
(`test/phase49.public-resource-availability.test.ts`, 3 casos + 8 de
`phase4.public-calendar.test.ts` sin regresión = 11/11): flag apagado
(default) da `resource_name=null` siempre vía el cliente `anon` real; flag
prendido da el nombre real; aislamiento cross-tenant confirmado (una
organización con el flag apagado nunca filtra el nombre del recurso de
otra organización con el flag prendido, en la misma ventana de tiempo).

**Hallazgo BAJO, pre-existente, fuera de alcance de esta fase**:
`resources.organization_id` se puede cambiar vía `UPDATE` directo sin que
ningún trigger lo impida (la policy `resources_update_staff`, Fase 35, sólo
valida membresía en `USING`/`WITH CHECK`, nunca inmutabilidad de la
columna). Alguien miembro de dos organizaciones podría mover un recurso de
B a A; si B tenía el flag prendido, su calendario público seguiría
mostrando el nombre de un recurso que ya es de A. Exige ser miembro de
ambas organizaciones — no explotable por un tercero ni por `anon`. Mismo
hallazgo ya registrado en `docs/database.md` Fase 48 — pendiente de una
fase propia con ADR (trigger de inmutabilidad de `organization_id` en
`resources` y, por coherencia, en `services`).

## Fase 50 — disponibilidad dinámica para recursos exclusivos (ADR-0051): reglas

Gate de `security-engineer` en dos vueltas sobre
`backend/supabase/migrations/20261002100000_phase50a_...sql` /
`20261002100001_phase50b_...sql` / `20261002110000_phase50c_...sql`
(`hold_dynamic_slot()`/`book_dynamic_slot()`/`release_dynamic_hold()`/
`get_dynamic_availability()`/`create_resource_availability_window()`),
verificado en vivo contra `reservaste-stg`.

**Primera vuelta: NO LISTO**, dos hallazgos bloqueantes reales (ninguno
anticipado por el diseño original de ADR-0051):

1. **ALTO — cancelar una reserva dinámica nunca liberaba el horario.**
   `cancel_booking()` sólo pasa la `Booking` a `CANCELLED`; la
   `SlotOccurrence` dinámica quedaba `ACTIVE` para siempre (capacidad 1,
   sin reservas reales), contada como ocupada tanto por
   `get_dynamic_availability()` como por el `EXCLUDE`. Rompía el
   invariante no negociable de CLAUDE.md "cancelar libera el cupo
   inmediatamente" — con una sola cuenta se podía vaciar la agenda de un
   recurso dinámico de forma permanente (hold → confirmar → cancelar,
   repetido; el tope de 2 reservas futuras de ADR-0047 no frena nada
   porque cancelar libera el contador). **Regla**: toda `SlotOccurrence`
   sin `scheduleRuleId` (marca de "viene de disponibilidad dinámica,
   nunca de grilla") tiene que liberarse ella misma cuando su última
   `Booking` `CONFIRMED` se cancela — implementado como trigger `AFTER
   UPDATE OF status ON bookings`, no dentro de `cancel_booking()`, para
   cubrir todos los caminos de cancelación existentes sin tener que
   auditarlos ni mantenerlos sincronizados uno por uno.
2. **MEDIO — `book_dynamic_slot()` no validaba `held_by`.** Cualquier
   cuenta autenticada que consiguiera el `slot_occurrence_id` (filtrable
   por URL/logs, mismo patrón que `/reservar/confirmar?slot=`) podía
   confirmar el hold de OTRA persona a su propio nombre dentro de los 5
   minutos, o hacerlo fallar a propósito para que la reversión cancelara
   el hold de la víctima. Además funcionaba como oráculo de existencia
   (`HOLD_EXPIRED` vs. `HOLD_NOT_FOUND` revelaban si el id existía).
   **Regla**: un hold nunca es un token al portador — sólo `held_by =
   auth.uid()` puede confirmarlo, y **todo rechazo (no existe, no está
   `HELD`, venció, es de otra persona, o es una ocurrencia de grilla)
   devuelve el mismo `HOLD_NOT_FOUND`**, sin distinción observable desde
   afuera (mismo criterio no-oráculo que ya usaba `release_dynamic_hold()`).

Más un hallazgo MEDIO y dos BAJO, cerrados en la misma pasada:

3. **MEDIO — un STAFF podía escribir `HELD` directo vía PostgREST**,
   afectando el tope GLOBAL de 3 holds/perfil de clientes de OTRAS
   organizaciones (conociendo su `profile_id`, un STAFF de la
   Organización A podía convertir ocurrencias de su propio recurso en
   `HELD` con `held_by = <profile_id de la víctima>` y `held_until` muy
   en el futuro, dejándola sin poder holdear en NINGUNA organización de
   la plataforma). **Regla**: la policy que permite a un STAFF togglear
   `ACTIVE ↔ BLOCKED` directo sobre `slot_occurrences` de su propia
   organización tiene que excluir explícitamente `HELD` — ni leer-para-
   escribir una fila ya `HELD`, ni poder producir `HELD` por ese camino
   (eso es exclusivo de `hold_dynamic_slot()`, `security definer`).
4. **BAJO — inserts directos a `schedule_rules` esquivaban
   `RESOURCE_IS_DYNAMIC`** (el chequeo vive en `create_schedule_rules_batch()`,
   no en la Data API). Impacto confirmado nulo (ninguna ocurrencia real
   se genera para esas reglas), cerrado igual con un trigger `BEFORE
   INSERT OR UPDATE` que además sirve de defensa contra la carrera
   "¿prendo el flag o creo la regla primero?" (serializa con `FOR SHARE`
   contra el trigger del guard del toggle).
5. **BAJO — `resource_availability_windows` tenía `DELETE` habilitado**
   por PostgREST (policy `FOR ALL`), contra el patrón ya establecido de
   ADR-0036 (ninguna tabla de negocio tiene `DELETE` real — baja lógica
   vía `is_active`/`cancelled_at`). Separado en `INSERT`/`UPDATE`, sin
   `DELETE`; el `INSERT` ahora exige `created_by = auth.uid()` (antes se
   podía falsificar).

**Segunda vuelta: LISTO**, con 3 hallazgos BAJO residuales, ninguno
bloqueante, documentados como deuda técnica en `docs/database.md` "Fase
50": orden de lock del trigger de ALTO-1 (caso límite de deadlock sólo
posible entre dos pestañas del mismo cliente sobre una ocurrencia de
grilla, Postgres lo resuelve abortando una transacción limpiamente, sin
riesgo de integridad); `resource_availability_windows_update_staff` no
fija `created_by`/`cancelled_by` (sólo afecta trazabilidad dentro de la
propia organización); un STAFF todavía puede editar
`scheduleRuleId`/`startAt`/`endAt` directo sobre una fila `ACTIVE`/`BLOCKED`
de su propia organización (superficie pre-existente, no agravada por
esta fase, acotada al propio tenant).

**Verificado y confirmado correcto** (sin cambios): rate limiting de
holds con el mismo patrón `pg_advisory_xact_lock` + `count(*)` ya
verificado bajo concurrencia real en ADR-0047; el `EXCLUDE` ampliado
protege `ACTIVE`/`ACTIVE`, `ACTIVE`/`HELD` y `HELD`/`HELD` por igual;
`book_dynamic_slot()` delega el 100% de cobertura/pago/alta de
`Customer` en `book_slot()` sin duplicar lógica, y revierte la promoción
a `CANCELLED` ante cualquier salida no-`OK` (incluidos todos los status
de rate-limit/`ORGANIZATION_NOT_ACCEPTING_NEW_CUSTOMERS` de ADR-0047);
aislamiento multi-tenant correcto en las 5 RPCs nuevas, ningún
`organization_id` llega como parámetro libre.

## Fase 53 — `organization_id` inmutable en `resources`/`services` (ADR-0052): regla

Gate de `security-engineer` sobre
`backend/supabase/migrations/20261002140000_phase53_organization_id_immutable.sql`,
verificado en vivo contra `reservaste-stg`.

**Regla**: `resources.organization_id` y `services.organization_id` son
**inmutables tras el `INSERT`** — dos triggers `BEFORE UPDATE`
(`resources_guard_organization_id_immutable`/
`services_guard_organization_id_immutable`, vía la función genérica
`guard_organization_id_immutable()`) rechazan con
`ORGANIZATION_ID_IS_IMMUTABLE` cualquier intento de cambiarlos, **sin
excepción de rol** — incluido `service_role`: un trigger no es una
policy de RLS, así que el bypass que `service_role` normalmente tiene
no aplica acá. Mover un recurso/servicio entre tenants por soporte
manual, si alguna vez hiciera falta, requiere deshabilitar el trigger
como superusuario — nunca un `UPDATE` directo con ninguna key de la
aplicación.

**Por qué hacía falta**: las policies de escritura existentes
(`resources_update_staff`/`services_update_staff`, Fase 35) validan
`is_organization_member(organization_id)` en `USING` y `WITH CHECK`,
pero eso sólo exige ser miembro de la organización origen y de la
destino por separado — no impedía el cambio en sí. Alguien con
membresía en dos organizaciones a la vez podía mover un
`Resource`/`Service` de una a otra (nunca explotable por `anon` ni por
un tercero sin esa doble membresía), con efectos en cascada reales ya
documentados en ADR-0045 (`resources_propagate_is_exclusive()`) y en
los gates de Fase 48/49/50 que señalaron este hueco de forma
independiente cada vez.

**Confirmado, no asumido** (verificación propia del gate, releyendo el
código real en vez de confiar en lo que reportó `backend-engineer`):
ningún RPC, trigger, ni server action de frontend hace `UPDATE` sobre
estas dos columnas en una fila ya creada — el único `UPDATE public.services`
que existe en todo el repo (Fase 14, backfill ya corrido) toca sólo
`billing_type`/`billing_cycle`/`payment_required`; no existe ningún
`UPDATE public.resources`. `discontinue_schedule_rule()` y el archivado
de `Resource`/`Service` desde el frontend sólo tocan
`is_active`/`cancelled_*`. El fix es estrictamente más restrictivo, no
bloquea ningún camino legítimo.

**Cambio acompañante, mismo gate**: `check_schedule_rule_conflicts()`
(ADR-0044) unifica `NOT_AUTHORIZED`/`RESOURCE_NOT_FOUND` en un único
`RESOURCE_NOT_FOUND` — cerraba un oráculo menor de existencia de
recursos de otra organización (impacto mínimo con UUID v4, pero la rama
`NOT_AUTHORIZED` ya era inalcanzable desde los tres llamadores internos
que ya validan organización antes, así que sólo quedaba expuesta vía
RPC directa).

**Hallazgo nuevo del gate, deuda técnica no bloqueante**: el mismo
patrón de `organization_id` mutable probablemente existe en otras
tablas con policies `is_organization_member(organization_id)`
similares — `schedule_rules`, `service_plans`, `customers` son
candidatas, sin auditar todavía. La función genérica ya escrita hace
trivial extender el mismo trigger a esas tablas si se confirma.

## Pendiente de definir (Phase 1)

- Proveedor de auth concreto: **Supabase Auth** (ADR-0002, cerrado).
- Traducción de las políticas de ADR-0006 a SQL concreto por tabla (con
  `database-agent`).
- ~~Alcance exacto de permisos de `STAFF` vs. `OWNER`.~~ **Resuelto por
  ADR-0033 / Fase 32** (roles configurables dentro de `STAFF`) — ver la
  sección "Fase 32" más arriba.
- Si `GET /me/bookings` agrega across todas las organizaciones del
  profile o requiere parámetro de organización (con `backend-api-agent`,
  ver `api.md`).
- Shape público exacto a nivel de campo de `Organization`/`Service` (qué
  campos NUNCA van en la respuesta pública — facturación, config interna,
  referencias a `OrganizationMember`/`Profile`) — a definir con el schema
  de Phase 1.
