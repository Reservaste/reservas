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
