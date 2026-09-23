# ADR-0026 — Cliente gestionado + activación por teléfono (WhatsApp)

Fecha: 2026-09-22
Estado: **Propuesta** (no aplicada — requiere aprobación del Orchestrator)
Propuesta por: `auth-security-agent`
Origen: feedback del primer cliente real, puntos 1, 2 y 3 de
`docs/iteration-3-plan.md` (§4, ADR-0026)
Depende de: ADR-0005 (la identidad sale de `auth.uid()`, nunca de un
parámetro), ADR-0006 (RLS de dos capas), ADR-0015 (transporte de estado a
través del login), ADR-0010 (taxonomía de cancelación), ADR-0024 (planes)
Habilita: Fase J; interactúa con ADR-0025 (crédito de recupero)
Implementa: Fase J del plan de Iteración 3

---

## 0. Lo que este ADR **no** reabre

Decidido por el usuario y por el Orchestrator, se toma como dado:

1. **Se exige cuenta para el autoservicio** (reservar, cancelar, ver el
   portal). Sin identidad no hay auditoría de quién canceló ni destinatario
   de un crédito de recupero.
2. **`customers.profile_id` pasa a nullable** + datos de contacto en la fila.
   Un cliente gestionado existe, se le agenda y se le cobra, y **no puede
   iniciar sesión ni ver nada**.
3. **El dueño manda el link desde su propio WhatsApp** vía deep link
   `api.whatsapp.com/send`. Sin Twilio, sin API de WhatsApp Business, sin
   costo por mensaje.

Este documento diseña el resto: shape, unicidad, el token, el canje, el
repaso de RLS, las superficies públicas, los gates de rol y los casos límite.

---

## 1. El problema

Hoy el alta de un cliente es imposible si la persona no tiene cuenta:

- `customers.profile_id` es `NOT NULL`
  (`20260914140249_phase1_auth_organizations_roles.sql:139`).
- `enroll_customer_by_email()` resuelve el email contra `auth.users` y tira
  `PROFILE_NOT_FOUND` si no existe
  (`20260914184943_phase8_admin_operations.sql:77-80`). El comentario de esa
  migración ya anticipaba esta ADR: *"MVP limitation: the person must already
  have an account. Sending an invitation to a stranger's email is a separate
  feature"*.

O sea: el dueño no puede dar de alta a nadie que no se haya registrado antes.
Para un estudio con 40 alumnos que ya están en su agenda de teléfono, eso es
un muro de entrada: tiene que convencer a 40 personas de registrarse antes de
poder usar el producto para lo que lo compró.

Lo que el cliente pidió, textual: *"dueño de la agenda habilita a través de
número de teléfono eso abre link de registro para ese gym"*, y *"Alta de
clientes que no pueden auto-registrarse"*.

### Por qué es un ADR de seguridad y no de UI

Porque el mecanismo introduce **un camino de identidad nuevo**: un secreto
portador que convierte a quien lo tenga en un cliente concreto, con su
historial de reservas y de pagos. Es el segundo camino por el que un
`auth.uid()` se ata a una fila de negocio (el primero es el registro propio +
`enroll_customer_by_email`), y el primero que puede activarse desde afuera del
panel. Todo lo demás de esta ADR —el shape, el merge, las RLS— existe para que
ese camino no abra nada más de lo que tiene que abrir.

---

## 2. La decisión

### 2.1 `customers` con `profile_id` nullable

```
customers
  id                        uuid pk
  organization_id           uuid not null → organizations
  profile_id                uuid NULL     → profiles      (era NOT NULL)
  display_name              text NULL     (nuevo)
  phone                     text NULL     (nuevo, E.164)
  claimed_at                timestamptz NULL  (nuevo)
  merged_into_customer_id   uuid NULL → customers (nuevo, ver §2.5)
  is_active, created_*, cancelled_*        (sin cambios)
```

**Dos estados de la misma entidad**, no dos entidades:

- **Gestionado** — `profile_id is null`. Existe, se le agenda, se le cobra, no
  tiene sesión y no ve nada.
- **Activado** — `profile_id is not null`, `claimed_at` cuenta cuándo y por
  qué camino.

#### Constraints

```sql
-- Un cliente sin identidad tiene que tener al menos un nombre visible: si no,
-- el mostrador tiene una fila anónima a la que le está cobrando.
constraint customers_identity_or_name check (
  profile_id is not null
  or (display_name is not null and length(trim(display_name)) > 0)
)

-- Formato normalizado, validado en la base. Sin esto "099 123 456",
-- "+59899123456" y "59899123456" son tres clientes distintos.
constraint customers_phone_e164 check (
  phone is null or phone ~ '^\+[1-9][0-9]{6,14}$'
)

-- claimed_at solo tiene sentido con identidad.
constraint customers_claimed_requires_profile check (
  claimed_at is null or profile_id is not null
)
```

**`phone` se guarda normalizado a E.164 con el `+`.** El deep link de WhatsApp
lo quiere sin `+` ni separadores; esa transformación es de la capa de UI, no
del dato. La normalización se hace en la RPC de alta (no en el formulario:
`customers` es escribible por PostgREST directo — mismo criterio que ADR-0020
para el color de marca y que ADR-0025 para la política de recupero).

#### El nombre efectivo

```
coalesce(nullif(trim(c.display_name), ''), p.full_name, 'Sin nombre')
```

**`display_name` gana sobre `profiles.full_name`**, y no se borra al activar.
Motivo: es el nombre con el que el negocio conoce a la persona, y si ganara el
profile, activar un cliente le cambiaría el nombre en la lista del mostrador
sin que nadie lo haya pedido. El cliente puede corregir su `full_name` en su
perfil y eso no le mueve nada al negocio, que es lo correcto: son dos nombres
distintos con dos dueños distintos.

#### Unicidad: qué impide dos filas para la misma persona

Tres capas, en orden de fuerza:

1. **`unique (organization_id, profile_id)`** — ya existe (phase1:147). Con
   `profile_id` nullable sigue valiendo y, como en Postgres los `NULL` son
   distintos entre sí por default, **permite N gestionados por organización**,
   que es exactamente lo que se quiere. No hace falta tocarla.
2. **`unique (organization_id, phone) where phone is not null`** — nueva. Es
   la que impide el duplicado del lado gestionado. El alta con un teléfono ya
   existente **no crea una fila**: devuelve la existente y la reactiva si
   estaba dada de baja — mismo comportamiento que `enroll_customer_by_email`
   (phase8:85-96), y por el mismo motivo: reenrolar a alguien no debe partirle
   el historial en dos.
3. **El canje rechaza el conflicto en vez de fusionarlo** (§2.4).

Deliberadamente **no** hay unicidad de teléfono entre organizaciones (§6.4).

#### `profile_id` pasa a `on delete set null`

Hoy es `on delete cascade` (phase1:139), y los cuatro FK a `customers` también
lo son (`bookings` phase5:25, `payments` phase7:27, `recurring_bookings`
phase6:20, `service_entitlements` phase2:127). O sea: **hoy, borrar un
`auth.users` borra en cascada el profile, el customer y con él las reservas y
los pagos** — contra el invariante de `domain.md` ("una `Booking` cancelada
nunca se borra") y contra cualquier trazabilidad económica.

Con `profile_id` nullable la corrección es gratuita y elegante: una persona
que borra su cuenta **se convierte exactamente en lo que esta ADR acaba de
definir**, un cliente gestionado, con su historial intacto y sin identidad.
Ver Hallazgo C (§4.4).

### 2.2 `customer_activations` — el token

```
customer_activations
  id                  uuid pk
  organization_id     uuid not null → organizations   (redundante a propósito: RLS)
  customer_id         uuid not null → customers on delete cascade
  token_hash          bytea not null unique           -- sha256 del token
  phone               text not null                   -- a qué número se emitió
  expires_at          timestamptz not null
  created_at          timestamptz not null default now()
  created_by          uuid not null → profiles
  redeemed_at         timestamptz
  redeemed_profile_id uuid → profiles
  revoked_at          timestamptz
  revoked_by          uuid → profiles

  -- Un solo link vivo por cliente, garantizado en la base.
  unique (customer_id) where redeemed_at is null and revoked_at is null
```

#### Entropía: 256 bits de CSPRNG, no `random()`

```sql
v_token := encode(extensions.gen_random_bytes(32), 'base64');  -- → base64url en la app
```

**Mínimo no negociable: 128 bits.** Propongo 256 porque no cuesta nada (el
link no se tipea, se toca).

**Advertencia concreta para `database-agent`:** el precedente del proyecto,
`create_organization_invite()` (phase10:429-435), usa `random()` —
explícitamente, y el comentario dice por qué: *"Built from random() instead of
pgcrypto's gen_random_bytes, which lives in the extensions schema and isn't on
this function's search_path"*. `random()` **no es un PRNG criptográfico**: es
seedable y su estado es reconstruible a partir de salidas observadas. Para un
código de venta que emite el platform admin, tiene un `email` opcional que lo
ata y cuyo peor caso es "alguien crea una organización con un plan que no
pagó", el riesgo es comercial y defendible. **Para un token que vale
"convertite en este cliente", es inaceptable.** La solución al problema del
`search_path` no es bajar de primitiva: es
`set search_path = public, extensions` en la función, o llamar
`extensions.gen_random_bytes(32)` calificado. (Recomiendo además migrar
`organization_invites` a la primitiva correcta, pero es otra fase y no
bloquea.)

#### Se guarda **hasheado**, no en texto plano

`token_hash = digest(token, 'sha256')`, `bytea`, con índice único → la búsqueda
sigue siendo O(1), sin el problema de "no puedo buscar un hash salado".

**SHA-256 simple, no bcrypt/argon2.** No es una contraseña: es un secreto de
256 bits, no adivinable, así que un KDF lento no compra nada y cuesta latencia
en el camino crítico. Lo que se compra con el hash es otra cosa: que un dump
de la tabla, un log de query lento, o un `SELECT` de alguien con acceso de
lectura **no contengan credenciales usables**.

**Por qué acá sí y en `organization_invites` no.** El precedente guarda `code`
en texto plano (phase10:117) y ahí es defendible: el código se lee por
teléfono, tiene que ser corto y legible, y lo único que ve la tabla es el
platform admin. Acá hay dos diferencias que cambian la respuesta: (a) el peor
caso no es un plan regalado sino **suplantación de una persona con historial
de pagos**, y (b) existe un actor con acceso legítimo y cercano —cualquier
`STAFF` de la organización— que **no debe poder robar el link de otro**. Con
hash, ni siquiera el dueño de la base puede reconstruir un link ya emitido.

**Consecuencia operativa que hay que aceptar:** el token en claro existe **una
sola vez**, en el valor de retorno de la RPC que lo emite. Si el dueño cierra
la pantalla sin mandar el WhatsApp, no se recupera: hay que **reenviar**, que
rota el token e invalida el anterior. Es fricción real y es la fricción
correcta.

#### Un solo uso, vencimiento, invalidación al reenviar

- **Un solo uso:** `redeemed_at`, verificado bajo `for update` dentro de la
  misma transacción del canje. Es la mecánica ya probada de
  `create_organization_with_owner()` (phase10:302-344) — eso **sí** se hereda
  del precedente.
- **Vencimiento: 72 horas.** Es un mensaje de WhatsApp: se abre en minutos casi
  siempre, y 72 h cubre "lo vi el lunes". Más largo solo agranda la ventana en
  que un número equivocado puede canjearlo. **Constante del producto, no
  configurable por el dueño** — contraste deliberado con ADR-0025, donde la
  política sí la configura el dueño porque es política de negocio; ésta es una
  perilla que degrada seguridad sin que quien la mueve entienda el costo.
- **Invalidación al reenviar:** emitir un token nuevo pone `revoked_at = now()`
  en el anterior (no se borra: auditoría). El índice único parcial
  `unique (customer_id) where redeemed_at is null and revoked_at is null` lo
  garantiza **en la base**, no por convención. Sin eso, "reenviar" dejaría
  vivo el link viejo — que es justamente el que quizás fue a un número
  equivocado, o sea el caso que el reenvío viene a arreglar.
- **Revocación explícita:** el dueño puede matar un link desde la pantalla, sin
  emitir otro.
- **Revocación en cascada al dar de baja al cliente:** trigger, §6.5.

#### Rate limiting

Dos límites distintos con dos motivos distintos:

- **Emisión** (el que más me preocupa): límite por organización, propongo
  **30/hora y 200/día**, contado sobre `customer_activations.created_at`. Sin
  esto, un `OWNER` comprometido o un bug de la UI generan miles de tokens y de
  filas de cliente. Se apoya además en el límite que ya existe:
  `customers_plan_limit` (phase10:239) cortaría el alta masiva en `max_customers`.
- **Canje**: por IP, propongo 10 intentos fallidos / 15 min. Con 256 bits, la
  fuerza bruta es imposible; este límite es contra abuso y ruido, no contra
  adivinación. Es el menos crítico de los dos.

> **Dependencia real, no un detalle:** verifiqué el repo y **no existe ninguna
> infraestructura de rate limiting**. ADR-0008 ya lo pidió para
> `get_public_availability` y sigue pendiente. Si el Orchestrator no quiere
> construirlo en la Fase J, el límite de **emisión** se puede implementar
> barato dentro de la propia RPC (un `count(*)` sobre la ventana antes de
> insertar) y el de **canje** queda como riesgo residual aceptado por escrito
> (R9, §8).

#### Si el dueño se equivoca de número

**Ésta es la falla irreducible del diseño**, y ningún tamaño de token la
arregla, porque lo que está mal es el canal de entrega. Mitigaciones, en orden
de utilidad real:

1. **La UI muestra el número completo y exige confirmarlo** en un paso propio
   antes de abrir WhatsApp. El error se corrige donde se origina, que es el
   único lugar donde alguien sabe cuál era el número correcto.
2. **Estado visible + revocación de un clic**: "enviado / activado / vencido /
   revocado". Un dueño que se da cuenta a los dos minutos tiene que poder matar
   el link sin llamar a soporte.
3. **La pantalla de activación dice a quién va a vincular**: *"Vas a entrar
   como **Juan Pérez** en **Estudio Pilates X**"*, con confirmación explícita
   (mismo patrón que ADR-0015 post-login: nunca acción automática). Un tercero
   que no es Juan Pérez ve que algo está mal y no confirma. **Esto no es
   seguridad** —un atacante confirma igual— **es reducción del daño
   accidental**, que es el caso realista.
4. **`unlink_customer_profile()`, OWNER-only**: deshace un canje incorrecto
   (`profile_id = null`, conserva todo el historial, registra quién y cuándo).
   Sin esto un canje erróneo es irreversible y la única salida es SQL a mano.
5. **Acotar el daño por construcción**: el token no otorga rol, no otorga
   acceso a datos de terceros, no mueve dinero. El daño máximo es que alguien
   vea el historial de reservas y pagos de esa persona **en ese negocio** y
   pueda cancelarle turnos. Serio, acotado, y reversible desde el mostrador.

#### Confirmación por últimos dígitos del teléfono: **recomiendo no hacerla**

Se me pidió evaluarla y decir el costo en fricción. La respuesta es peor que
"cuesta un campo": **no defiende el caso que nos preocupa**.

- Escenario que motiva la medida: el dueño quiso mandar al número A y tipeó B.
  El sistema guardó `phone = B` (lo que tipeó). El link llegó a B. Si la
  pantalla pide "los últimos 4 dígitos de tu teléfono" y compara contra el
  `phone` guardado, está comparando contra **B**, que es el número de la
  persona que lo recibió por error. **Los sabe perfectamente.** El chequeo pasa.
  Para defender de verdad haría falta comparar contra el número **real** de A,
  que es justamente el dato que el sistema no tiene.
- Escenario que **sí** defiende: el link reenviado. El cliente legítimo lo
  manda a un grupo, o queda en un screenshot, y alguien que no conoce el número
  lo abre. Ése es un caso real (los links se reenvían) pero ya está bastante
  acotado por uso único + 72 h + revocación + la confirmación con nombre.
- Costo: un campo más en una pantalla que idealmente es "tocá y listo", en un
  teléfono, para alguien que —más seguido de lo que uno cree— no se sabe su
  propio número de memoria. En una función cuyo único KPI es que la activación
  no se caiga a la mitad, eso no es gratis.

**Veredicto:** no implementarla en la Fase J. Si el Orchestrator la quiere
igual, que sea **flag por organización, default OFF**, con 4 intentos antes de
revocar el token automáticamente.

### 2.3 Alta y emisión: quién puede

| Operación | Gate | Justificación |
|---|---|---|
| `create_managed_customer()` | **STAFF** (miembro activo) | Enrolar clientes es trabajo diario de mostrador: `enroll_customer_by_email` usa `is_organization_member` (phase8:73). Un cliente gestionado no agrega poder sobre el tenant, agrega una fila de cliente. |
| `issue_customer_activation()` (emitir / reenviar) | **STAFF** | El link no otorga rol ni acceso a datos de la organización: otorga "ser este cliente". Quien ya puede crear la fila, cobrarle, reservarle y cancelarle no gana nada nuevo. Gatearlo a OWNER tendría el efecto perverso de que el mostrador no pueda completar el alta y el dueño termine **compartiendo su cuenta** — peor postura que la que se quiso proteger. |
| `revoke_customer_activation()` | **STAFF** | Es la corrección de un error de mostrador; tiene que poder hacerla el que lo cometió, en el momento. Revocar nunca abre nada. |
| `unlink_customer_profile()` | **OWNER** | Corta el acceso de una persona identificada a su propia información. Se parece más a `revoke_member` (OWNER-only, phase8:177) que a enrolar un cliente. |
| `merge_customers()` | **OWNER** | Mueve pagos y reservas entre filas. Es dinero. |

El criterio que el proyecto ya usa y que acá se respeta: **enrolar clientes es
STAFF (phase8:73), tocar quién tiene poder o quién tiene acceso es OWNER
(phase8:128, phase8:177)**.

Los gates viven **en la RPC** (`security definer`), no en la server action —
la RLS de `customers` admite a cualquier miembro, y una regla que solo existe
en Next.js no es una frontera. Misma nota que el punto 3 de lo que destapó la
Fase H sobre los planes gateados a OWNER en la server action.

### 2.4 El canje

```sql
claim_customer_activation(p_token text) returns jsonb
```

**Un solo parámetro, y no nombra ninguna fila de cliente.** Ésta es la
aplicación directa de ADR-0005: la identidad sale de `auth.uid()` y la fila de
destino sale **del token**, nunca de un `customerId` del caller. No hay IDOR
posible porque no hay ningún input que apunte a una fila.

`security definer`, y explícitamente:

```sql
revoke execute on function public.claim_customer_activation(text) from public, anon;
grant  execute on function public.claim_customer_activation(text) to authenticated;
```

El `revoke ... from public` no es decorativo: en Postgres el `EXECUTE` de una
función nueva se otorga a `PUBLIC` por default, y PostgREST expone el schema
`public` (ver Hallazgo D, §4.4). **Nunca a `anon`**: el producto de esta
función es atar `auth.uid()` a una fila; sin sesión no hay nada que atar.

Pasos, todo en una transacción:

1. `auth.uid() is null` → `AUTH_REQUIRED`.
2. `select ... from customer_activations where token_hash = digest(p_token,'sha256') for update`.
   No hay fila → `INVALID_TOKEN`.
3. Estados del token, **distinguidos a propósito**: `revoked_at is not null` →
   `ACTIVATION_REVOKED`; `redeemed_at is not null` → `ALREADY_REDEEMED`;
   `expires_at <= now()` → `ACTIVATION_EXPIRED`. Distinguirlos no filtra nada
   —para llegar acá hay que **tener el secreto completo**, que ya es la
   credencial— y es lo que le permite a la pantalla decir "pedile al negocio
   que te lo reenvíe" en vez de un error mudo.
4. La fila de cliente existe, `is_active`, y la organización está activa. Si no
   → `CUSTOMER_UNAVAILABLE`.
5. **Conflicto de doble fila:**
   ```sql
   exists (select 1 from customers
            where organization_id = v_act.organization_id
              and profile_id = auth.uid()
              and id <> v_act.customer_id)
   ```
   → `ALREADY_CUSTOMER_OF_ORGANIZATION`, **sin marcar el token como canjeado ni
   revocarlo**: el dueño resuelve con `merge_customers()` y el mismo link sigue
   sirviendo. Consumir el token acá obligaría a reemitir por un problema que no
   es del cliente.
6. **El vínculo, atómico:**
   ```sql
   update customers
      set profile_id = auth.uid(), claimed_at = now()
    where id = v_act.customer_id
      and profile_id is null;          -- ← la guarda
   ```
   El `and profile_id is null` es lo que hace que dos canjes concurrentes
   tengan exactamente un ganador, incluso si el `for update` del paso 2 no
   alcanzara. Si `not found` → `ALREADY_CLAIMED`.
7. `update customer_activations set redeemed_at = now(), redeemed_profile_id = auth.uid() where id = v_act.id and redeemed_at is null`.
8. Devuelve `{ status: 'OK', organization_slug, customer_id }` para aterrizar
   en el portal de ese negocio.

#### Qué pasa si canjea dos veces

El `for update` + `redeemed_at is null` + `profile_id is null` son tres guardas
independientes. Regla de respuesta:

- Mismo profile que ya canjeó ese token → **`OK` idempotente** con el slug. Un
  doble tap en un celular no puede mostrar un error.
- Otro profile → `ALREADY_REDEEMED`, y no se toca nada.

#### Qué pasa si el que abre el link ya tiene sesión con otra cuenta

Es el caso **más frecuente**, no un borde: el cliente abre el link en un
teléfono donde quedó la sesión de su pareja, o el dueño lo abre en el suyo para
"ver cómo queda".

- La pantalla de activación **siempre** muestra con qué cuenta va a vincular y
  exige clic explícito: *"Continuar como mariana@… / Usar otra cuenta"*. Nunca
  canje automático (ADR-0015, mismo principio de confused deputy).
- "Usar otra cuenta" = logout + volver a la misma URL de activación.
- **Si el que abre es miembro (STAFF/OWNER) de esa organización**: no se
  bloquea —un profesor puede legítimamente ser cliente de su propio negocio—
  pero la pantalla **advierte** ("sos parte del equipo de este negocio") y pide
  la misma confirmación. `unlink_customer_profile()` lo deshace.

#### El transporte del token a través del login

Éste es el punto donde ADR-0026 **se aparta** de ADR-0015 y hay que decirlo
fuerte: en ADR-0015 el intent viaja en query params sin firmar **porque son
datos ya públicos**. Acá el query param **es un secreto**. Consecuencias
concretas, todas implementables:

- **En el primer GET del link**, el route handler toma el token de la URL, lo
  guarda en una **cookie `httpOnly`, `Secure`, `SameSite=Lax`, de vida corta
  (15 min)** y redirige a la misma ruta **sin el token**. Así no queda en el
  historial del navegador, ni en los logs del reverse proxy, ni se comparte al
  mandar "mirá esta página".
- **`Referrer-Policy: no-referrer`** en la página de activación: sin eso, el
  token sale hacia cualquier host de un recurso externo vía `Referer`.
- **`Cache-Control: no-store`** y `noindex`.
- **`returnTo` sigue validándose contra el allowlist server-side de ADR-0015**.
  Acá deja de ser solo una defensa contra open redirect: un open redirect con
  un token en la sesión es **exfiltración directa del token** a un host ajeno.
- El token **nunca** se loggea, ni en un `console.error` de la server action.

### 2.5 El merge — el caso feo, definido

Escenario: el mostrador dio de alta a "Juan, 099…" como gestionado hace tres
meses; Juan tiene 40 reservas y 3 pagos. En el medio Juan se registró solo por
la página pública y ya es customer de esa misma organización con su propia
fila. Ahora abre el link de activación. Hay dos filas que son la misma persona.

#### El canje **no fusiona**. Nunca.

Tres motivos, todos concretos:

1. El merge **mueve dinero** (repunta `payments`) y reservas entre filas.
   Dárselo como efecto secundario a un canje disparado por alguien que abrió un
   link es darle a un actor no verificado la capacidad de reescribir historial
   económico.
2. **Puede fallar por constraint, a mitad.** El `EXCLUDE` de ADR-0024 impide
   dos `Payment` `PAID` con períodos solapados del mismo `(customer, service)`:
   si cada fila tiene un pago de septiembre, el repunte explota. Lo mismo con
   `unique (customer_id, slot_occurrence_id)` de `bookings` si las dos filas
   están anotadas en el mismo turno. Una operación que puede fallar por
   constraint no puede vivir en el camino crítico del canje.
3. Un merge automático es **irreversible de hecho**: deshacerlo requiere saber
   qué fila era cuál, y esa información ya se perdió.

#### `merge_customers(p_target_customer_id, p_source_customer_id)` — OWNER-only

Operación de mostrador, explícita, con su propia pantalla:

- Valida que ambas filas son de **la misma organización** y que el caller es
  `OWNER` de ella.
- **Target = la fila que tiene `profile_id`.** Es la que el portal del cliente
  ya muestra, y `unique (organization_id, profile_id)` garantiza que es única.
  Source = la gestionada, con el historial del negocio.
- Repunta al target: `bookings`, `payments`, `recurring_bookings`,
  `service_entitlements` (histórico), y `makeup_credits` cuando exista
  (ADR-0025).
- **No repunta lo que chocaría.** Las filas que violarían el `EXCLUDE` de pagos
  o el `unique(customer, slot)` de bookings **se quedan en la fila origen**. No
  se borran, no se cancelan, no se "resuelven" con una heurística: se quedan
  donde están y la pantalla las lista. Un merge que adivina cuál de dos pagos
  solapados vale es un merge que se equivoca en silencio con la plata de un
  cliente.
- La fila origen queda `is_active = false`,
  `cancellation_reason = 'ORGANIZATION_REMOVED'`, y
  **`merged_into_customer_id = target`** — para que las lecturas puedan seguir
  el puntero y el mostrador entienda por qué esa fila está inactiva y medio
  vacía, en vez de encontrarse un fantasma.
- El teléfono se mueve al target (y por eso la fila origen libera el
  `unique(organization_id, phone)`).

**Marcado como decidible por el Orchestrator (§9.3):** si se prefiere diferir
`merge_customers()` a una fase posterior, la Fase J igual funciona —el canje
rechaza el conflicto con un mensaje accionable— pero el mostrador queda sin
remedio y alguien va a terminar arreglándolo con SQL a mano, que es peor. Mi
recomendación es que entre, aunque sea sin pantalla propia.

---

## 3. Genericidad

Vocabulario: **cliente gestionado**, `display_name`, `phone`,
`customer_activations`, `claim_customer_activation`. Ninguna palabra de rubro.

- Un consultorio da de alta pacientes por teléfono con **la misma RPC**. Una
  cancha, a quien alquila. Una academia, a quien se inscribe.
- Prohibido en la UI: "alta rápida de socios", "alumno", "miembro". El texto
  del mensaje de WhatsApp lleva el **nombre de la organización**, nunca una
  palabra de rubro.
- **WhatsApp no entra al modelo de dominio.** `customers.phone` es un teléfono;
  el deep link es una conveniencia de la capa de UI. El día que haga falta SMS,
  Telegram o un QR impreso, **no cambia el schema**: cambia cómo se entrega el
  mismo token. Eso también es lo que convierte "sin Twilio" en una decisión de
  UI y no de arquitectura.

---

## 4. Repaso de RLS con `profile_id` nullable — policy por policy

Verificado **contra el schema real**, no en abstracto. El razonamiento base:
en SQL `NULL = x` y `NULL <> x` dan `NULL`, y RLS trata `NULL` como fila **no
visible** (necesita `true`). Pero eso solo vale mientras la expresión esté en
una policy; en plpgsql un `NULL` en un `if` se comporta distinto, y ahí está el
problema real (Hallazgo A).

### 4.1 Policies de RLS (23, todas las del repo)

| # | Policy | Archivo:línea | Expresión relevante | Veredicto con `profile_id` NULL |
|---|---|---|---|---|
| 1 | `profiles_select_own` | phase1:252 | `id = auth.uid()` | **Sin impacto** — no toca `customers` |
| 2 | `profiles_update_own` | phase1:256 | `id = auth.uid()` | **Sin impacto** |
| 3 | `organizations_select_members` | phase1:265 | `is_organization_member(id)` | **Sin impacto** — `organization_members.profile_id` sigue `NOT NULL` |
| 4 | `organizations_update_owner` | phase1:278 | `is_organization_owner(id)` | **Sin impacto** |
| 5 | `organization_members_select_same_org` | phase1:285 | `is_organization_member(organization_id)` | **Sin impacto** |
| 6 | `organization_members_write_owner` | phase1:289 | `is_organization_owner(organization_id)` | **Sin impacto** |
| 7 | `customers_select_self_or_staff` | phase1:299 | `profile_id = auth.uid() or is_organization_member(...)` | **Seguro, sin cambio.** `NULL = auth.uid()` → `NULL`. `NULL or true` = `true` (staff ve, correcto); `NULL or false` = **`NULL`** → no visible. Ningún customer ve la fila gestionada de otro. Con `auth.uid()` NULL (anon) da `NULL`/`false`, y además `anon` no tiene grant sobre la tabla |
| 8 | `customers_write_staff` | phase1:306 | `for all using (is_organization_member(organization_id))` | **Seguro en cuanto al NULL**, pero **necesita una guarda nueva** — ver Hallazgo E |
| 9 | `resources_select_members` | phase2:211 | `is_organization_member` | **Sin impacto** |
| 10 | `resources_write_staff` | phase2:215 | `is_organization_member` | **Sin impacto** |
| 11 | `services_select_members` | phase2:219 | `is_organization_member` | **Sin impacto** |
| 12 | `services_write_staff` | phase2:223 | `is_organization_member` | **Sin impacto** |
| 13 | `service_resources_select_members` | phase2:227 | vía `services` | **Sin impacto** |
| 14 | `service_resources_write_staff` | phase2:237 | vía `services` | **Sin impacto** |
| 15 | `service_entitlements_select_self_or_staff` | phase2:250 | `exists(... c.profile_id = auth.uid())` `or` staff | **Seguro.** `NULL` nunca matchea dentro del `EXISTS` → `exists` = `false` → queda solo la rama de staff (tabla deprecada por ADR-0024, la policy sigue viva) |
| 16 | `service_entitlements_write_staff` | phase2:261 | `is_organization_member` | **Sin impacto** |
| 17 | `schedule_rules_select_members` / `_write_staff` | phase3:494 / 498 | `is_organization_member` | **Sin impacto** |
| 18 | `schedule_exceptions_select_members` / `_write_staff` | phase3:502 / 506 | `is_organization_member` | **Sin impacto** |
| 19 | `slot_occurrences_select_members` / `_toggle_active_blocked` | phase3:510 / 519 | `is_organization_member` | **Sin impacto** |
| 20 | `bookings_select_self_or_staff` | phase5:433 | `exists(... c.profile_id = auth.uid())` `or` staff | **Seguro.** Además **no hay policy de INSERT/UPDATE/DELETE en `bookings`** → denegadas por default, toda escritura por RPC. Eso es load-bearing acá |
| 21 | `recurring_bookings_select_self_or_staff` | phase6:462 | idem | **Seguro**, mismo análisis |
| 22 | `payments_select_self_or_staff` | phase7:517 | idem | **Seguro**, mismo análisis |
| 23 | `payments_insert_staff` / `payments_update_staff` | phase7:527 / 531 | `is_organization_member` | **Sin impacto** — cobrarle a un gestionado funciona igual. Sin policy de DELETE, correcto (`VOID`, nunca borrar) |
| 24 | `plans_select_all` | phase10:52 | `using (true)` | **Sin impacto** — catálogo del SaaS, sin datos de persona |
| 25 | `platform_admins_select_admin` | phase10:105 | `is_platform_admin()` | **Sin impacto** |
| 26 | `organization_invites_select_admin` | phase10:135 | `is_platform_admin()` | **Sin impacto** (precedente relevante, §2.2) |
| 27 | storage: `organization logos are public` + 3 de OWNER | phase13:114-134 | bucket / `is_organization_owner` | **Sin impacto** |
| 28 | `service_plans_select_public` | phase17:1577 | `using (true)` | **Sin impacto del NULL**, pero **hay que acotarla** — §5.2 |
| 29 | `service_plans_insert_staff` / `_update_staff` | phase17:1581 / 1585 | `is_organization_member` | **Sin impacto** |

**Conclusión del repaso: el `NULL` no abre ninguna policy.** Las cinco policies
de dos capas de ADR-0006 usan el mismo patrón (`= auth.uid()` en un `USING`
directo o dentro de un `EXISTS`), y en los dos casos un `NULL` produce `NULL`,
que RLS trata como no visible. Lo confirmo policy por policy arriba.

### 4.2 Funciones `SECURITY DEFINER`

| Función | Archivo:línea | Veredicto |
|---|---|---|
| `is_organization_member(uuid)` | phase1:215 | **Sin impacto.** Lee `organization_members.profile_id`, que sigue `NOT NULL`. `auth.uid()` NULL → `exists` false |
| `is_organization_owner(uuid)` | phase1:230 | **Sin impacto**, mismo análisis |
| `is_platform_admin()` | phase10:92 | **Sin impacto** |
| `organization_can_operate(uuid)` | phase10:147 | **Sin impacto** |
| `can_customer_book(uuid)` | phase11:107 (versión vigente) | **Seguro.** Resuelve `where organization_id = … and profile_id = auth.uid() and is_active`; una fila con `NULL` nunca matchea → `NOT_A_CUSTOMER`. Es exactamente el comportamiento buscado: un gestionado no reserva |
| `book_slot(uuid)` | phase17:838-841 | **Seguro**, misma resolución. Un gestionado no puede autoreservar |
| `create_recurring_booking(uuid)` | phase17:1346 (y phase6:322) | **Seguro**, misma resolución |
| `cancel_recurring_booking(uuid)` | phase6:369-377 | **Seguro** — y es el ejemplo a copiar: `if <self> then … elsif <member> then … else raise`. Con `profile_id` NULL el primer `if` da `NULL`, cae al `elsif`, y si no es miembro llega al `else raise`. **Correcto por construcción** |
| **`cancel_booking(uuid, reason)`** | **phase15:301** | 🔴 **ROTO. Ver Hallazgo A** |
| `admin_book_for_customer(...)` | phase8:216, reescrita en phase15/17 | **Seguro.** Recibe `p_customer_id` pero valida `is_organization_member(v_occurrence.organization_id)` **y** que el customer pertenezca a esa organización antes de tocar nada (phase8:237-244). Funciona igual con un gestionado, que es el objetivo |
| `mark_attendance(uuid, status)` | phase16:265-284 | **Seguro.** Gate `is_organization_member` puro, no toca `profile_id` |
| `my_bookings(boolean)` | phase9:44 | **Seguro** — `join customers c on … and c.profile_id = auth.uid()`: una fila NULL no joinea, devuelve 0 filas. Un gestionado no ve nada (lo decidido) |
| `my_entitlements()` | phase9:92 | **Seguro**, mismo join |
| `my_payments()` | phase15:600 (vigente) | **Seguro**, mismo join |
| `my_services()` | phase15:564 | **Seguro** — `where c.profile_id = auth.uid() and c.is_active` |
| `public_slot_detail(uuid)` | phase9:134 | **Seguro** — se apoya en `get_public_availability`, no toca `customers` |
| `get_public_availability(...)` | phase5:352 (vigente) | **Seguro** — toca `bookings` solo como `count(*)` (phase5:401-404); nunca `customers`, nunca una fila |
| `agenda_occurrences(...)` | phase8:305 | **Sin impacto** |
| `occurrence_attendance_summary(uuid)` | phase16:306 | **Sin impacto** (agregados) |
| `organization_usage` / `platform_organizations` | phase10:356 / 502 | **Sin impacto** (cuentan `customers`; un gestionado cuenta, ver §6.1) |
| `enroll_customer_by_email(...)` | phase8:60 | **Sigue funcionando**, pero necesita un ajuste: si ya existe una fila **gestionada** con ese teléfono/persona, hoy crearía una segunda fila con `profile_id`. Ver §7 paso 8 |
| `retry_not_generated_booking(uuid)` | phase12:24 | ⚠️ **Sin autorización alguna** — Hallazgo D, preexistente |
| **`organization_customers(uuid)`** | **phase8:407** | 🟠 **INNER JOIN a `profiles`** — Hallazgo B |
| **`occurrence_bookings(uuid)`** | **phase16:257** | 🟠 **INNER JOIN a `profiles`** — Hallazgo B |
| **`organization_payment_summary(...)`** | **phase16:408** | 🟠 **INNER JOIN a `profiles`** — Hallazgo B |
| **`customer_payment_detail(...)`** | **phase16:~422** | 🟠 A verificar en implementación, mismo patrón — Hallazgo B |
| **`schedule_rule_standing_reservations(uuid)`** | **phase17:1554** (y phase15:765) | 🟠 **INNER JOIN a `profiles`** — Hallazgo B |
| `organization_team(uuid)` | phase8:434 | **Sin impacto** — joinea por `organization_members.profile_id`, `NOT NULL` |

### 4.3 Vistas

| Vista | Archivo:línea | Veredicto |
|---|---|---|
| `organizations_public` | phase4:22, redefinida phase13:65 | `id, slug, name, timezone, brand_color, logo_path` de `organizations where is_active`. **No joinea `customers`. No filtra teléfono** |
| `services_public` | phase4:29 | `id, organization_id, name, description` de `services where is_active`. **No filtra** |

### 4.4 Hallazgos — cinco, tres bloqueantes

#### 🔴 Hallazgo A (CRÍTICO) — `cancel_booking()` deja de autorizar con un `profile_id` NULL

`backend/supabase/migrations/20260921140000_phase15_booking_without_entitlements.sql:301`

```sql
if v_customer.profile_id <> auth.uid() and not public.is_organization_member(v_booking.organization_id) then
  raise exception 'NOT_AUTHORIZED';
end if;
```

Con `v_customer.profile_id IS NULL`:

- `NULL <> auth.uid()` → `NULL`
- `NULL and true` → `NULL`  (el atacante **no** es miembro, así que
  `not is_organization_member(...)` = `true`)
- `if NULL then` → **no entra al `raise`**

**Escenario de explotación:** un usuario autenticado cualquiera —de otra
organización, o ninguna— llama
`POST /rest/v1/rpc/cancel_booking {"p_booking_id": "<uuid>"}` contra la reserva
de un cliente **gestionado**. La función es `SECURITY DEFINER` (bypasea RLS) y
está `grant execute … to authenticated` (phase5:257). La reserva queda
`CANCELLED`, el cupo se libera, `cancelled_by` queda apuntando al atacante, y
—cuando entre ADR-0025— **además se emite un crédito de recupero** a nombre de
una víctima que no pidió nada. El único obstáculo es conocer el `booking_id`,
que es un UUIDv4: no adivinable, pero es un identificador que circula por
payloads, logs y URLs, y "no adivinable" no es una autorización.

**Hoy no es explotable**, porque `profile_id` es `NOT NULL`. **Lo vuelve
explotable exactamente esta ADR.** Es el motivo por el que la migración 19 no
puede ser solamente `drop not null`.

**Corrección:** reescribir con la forma de tres ramas de
`cancel_recurring_booking` (phase6:369-377), que es la que se comporta bien con
`NULL` **y** además resuelve el motivo de cancelación correctamente:

```sql
if v_customer.profile_id = auth.uid() then
  -- el propio cliente
elsif public.is_organization_member(v_booking.organization_id) then
  -- el mostrador, en su nombre o por decisión del negocio
else
  raise exception 'NOT_AUTHORIZED';
end if;
```

**Lección transversal, que va al checklist de `security.md`:** cualquier
comparación con `auth.uid()` **dentro de plpgsql** (no en una policy) tiene que
escribirse de forma que un `NULL` caiga del lado seguro. En una policy, `NULL`
deniega; en un `if`, `NULL` **salta el `raise`**. Es la misma expresión con el
comportamiento opuesto.

#### 🟠 Hallazgo B (ALTO) — cinco INNER JOIN a `profiles` hacen desaparecer al cliente gestionado

Todos con la forma `join public.profiles p on p.id = c.profile_id`:

- `organization_customers()` — phase8:407 → **la pantalla Clientes**
- `occurrence_bookings()` — phase16:257 → **quién está anotado en el turno**
  (la pantalla de pasar lista de ADR-0023)
- `organization_payment_summary()` — phase16:408 → **el resumen de cobranza**
- `customer_payment_detail()` — phase16 (verificar) → el detalle por cliente
- `schedule_rule_standing_reservations()` — phase17:1554 (y phase15:765) →
  **quién tiene cupo fijo**

**Consecuencia:** un cliente gestionado, con reservas y pagos, **es invisible
en todo el panel**. No es una fuga —es lo contrario— pero es un fallo de
integridad operativa grave: el profesor pasa lista y falta gente que está en la
sala; el dueño mira la cobranza del mes y no ve lo que le deben.

**Corrección:** `left join` + `coalesce(nullif(trim(c.display_name), ''), p.full_name, 'Sin nombre')`,
y el mismo `coalesce` en los `order by`. Las cinco en la **misma migración**:
si no, la Fase J entrega un alta que no se ve en ninguna pantalla.

#### 🟡 Hallazgo C (MEDIO) — `customers.profile_id` es `on delete cascade`

phase1:139, más los cuatro FK a `customers` que también cascadean (phase5:25,
phase7:27, phase6:20, phase2:127). Borrar un `auth.users` hoy borra profile →
customer → **reservas y pagos**. Contradice el invariante de `domain.md` de que
una `Booking` cancelada nunca se borra, y deja al negocio sin registro
económico de alguien que le pagó.

**Corrección, de dos líneas y con la ocasión servida:** `on delete set null`.
Quien borra su cuenta se convierte en un cliente gestionado — el estado que
esta ADR acaba de definir — con historial intacto y sin identidad.

#### 🟡 Hallazgo E (MEDIO) — un STAFF puede reescribir `profile_id` por PostgREST directo

`customers_write_staff` (phase1:306) es `for all using (is_organization_member(organization_id))`
y **sin `with check` explícito** (Postgres usa el `using` también como `with
check`, así que el INSERT está cubierto). Pero la condición **solo mira
`organization_id`**: nada impide

```
PATCH /rest/v1/customers?id=eq.<fila_de_otro_cliente>
{ "profile_id": "<uuid del STAFF o de un cómplice>" }
```

Con `profile_id NOT NULL` esto ya era posible y ya era malo (pisar la identidad
de un cliente existente). Con clientes gestionados se vuelve el camino de
ataque **obvio**, porque ahora hay filas con `profile_id` vacío esperando a que
alguien las reclame, y reclamarlas directamente saltea el token entero.

**Corrección:** trigger `BEFORE UPDATE` sobre `customers` que rechace cualquier
cambio de `profile_id` que no venga de las dos RPC autorizadas
(`claim_customer_activation`, `unlink_customer_profile`, más
`merge_customers`), señalizado con una GUC de transacción:

```sql
if new.profile_id is distinct from old.profile_id
   and coalesce(current_setting('app.allow_profile_link', true), 'off') <> 'on' then
  raise exception 'PROFILE_LINK_NOT_ALLOWED';
end if;
```

y `set local app.allow_profile_link = 'on'` dentro de esas tres funciones. Se
pone como trigger y no como convención por el mismo motivo que `set_updated_at`
(phase1:16): hay varios caminos de escritura y un trigger es el único punto que
los cubre a todos.

#### ⚪ Hallazgo D (MENOR, preexistente, fuera de alcance) — `retry_not_generated_booking()` sin gate

phase12:24. No tiene ninguna verificación de autorización y su migración **no
hace `revoke execute … from public`**. En Postgres el `EXECUTE` de una función
nueva se otorga a `PUBLIC` por default, y PostgREST expone el schema `public`,
así que es invocable. El impacto es acotado (confirma una reserva que el
cliente igual quería) pero consume cupo ajeno.

No lo arreglo acá. Lo dejo anotado y **fijo la regla operativa** que sí aplica
a esta ADR: **cada RPC nueva lleva su `revoke execute … from public, anon`
explícito antes del `grant` puntual.**

---

## 5. El teléfono es dato privado

### 5.1 Verificación superficie por superficie

| Superficie | Accesible a | ¿Toca `customers`? | Veredicto |
|---|---|---|---|
| `organizations_public` (phase13:65) | `anon` | No | **No filtra** |
| `services_public` (phase4:29) | `anon` | No | **No filtra** |
| `get_public_availability()` (phase5:352) | `anon` | Solo `count(*)` sobre `bookings` (phase5:401-404) | **No filtra** — nunca una fila, solo el agregado que ADR-0008 ya modula |
| `public_slot_detail()` (phase9:134) | `anon` | No (delega en la anterior) | **No filtra** |
| `plans` (phase10:52) | `anon` | No | **No filtra** |
| `service_plans` (phase17:1577/1590) | `anon` | No | **No filtra teléfono**, pero expone de más — §5.2 |
| storage `organization-logos` (phase13:114) | `anon` | No | **No filtra** |
| `my_*()` (phase9, phase15) | `authenticated` | Sí, por `profile_id = auth.uid()` | **No filtra** el teléfono de terceros: cada uno ve su propia fila |
| `customer_activations` | — | Sí | **Sin `grant select` a nadie** — §7 paso 5 |

### 5.2 Reglas que quedan escritas (más útiles que la lista)

1. **`customers.phone` y `customers.display_name` no se exponen por ninguna
   vista ni RPC accesible a `anon`.** Punto.
2. **Ninguna superficie `anon` puede joinear `customers`, `bookings`,
   `payments` ni `customer_activations`.** Si una pantalla pública necesita un
   dato de ahí, es un **agregado calculado** (como `remaining`), nunca una
   fila. Esta regla es más fuerte que enumerar columnas, porque sobrevive a la
   próxima vista que alguien escriba.
3. **`customer_activations` no tiene `grant select` para nadie.** El panel lee
   el estado de una invitación por una vista/RPC `customer_activation_status`
   que devuelve `created_at / expires_at / redeemed_at / revoked_at / phone` y
   **nunca `token_hash`**. Motivo: RLS filtra filas, no columnas; confiar en un
   `grant select (col, col, …)` es confiar en que nadie lo amplíe después.

### 5.3 Donde el teléfono **sí** sale, y hay que decirlo

El deep link `api.whatsapp.com/send?phone=<número>&text=<link>` se abre desde
el navegador del dueño **hacia Meta**. Eso es inherente al mecanismo elegido y
es aceptable (el dueño ya tiene ese número en su agenda de teléfono).

Lo que hay que decir explícitamente es lo otro: **el `text=` contiene el
token**, así que el token de activación pasa por WhatsApp/Meta y queda en dos
historiales de chat. No hay forma de evitarlo con este diseño — *es* el diseño.
Consecuencias: refuerza el vencimiento corto y el uso único, y **descarta de
plano meter algo más valioso que "vinculá esta cuenta" en ese link** (ver §8.1).

### 5.4 Veredicto sobre `service_plans_select_public using (true)`

Me lo dejaron anotado como pendiente. **Hay que acotarlo.** Tres razones
concretas, ninguna teórica:

1. **Expone planes inactivos.** La UI de la Fase H se apoya en desactivar
   planes para ordenar la lista de precios sin cortar cobertura (invariante de
   ADR-0024). La lista pública muestra igual el precio que el dueño acaba de
   retirar.
2. **Expone planes de servicios inactivos y de organizaciones inactivas.**
   `services_public` filtra `where is_active` **a propósito**, así que hoy un
   `anon` puede leer `service_id`, nombre y precio de un servicio que la
   organización **sacó del público**. Es una inconsistencia de disclosure entre
   dos superficies que hablan del mismo objeto: el producto oculta el servicio
   y publica su lista de precios.
3. **Expone columnas internas**: `created_by` y `cancelled_by` (UUID de
   `profiles` del equipo), `created_at/updated_at`, `sort_order`, `cancelled_at`.
   Nada de eso es lista de precios. El `created_by` no es explotable solo
   (`profiles` es own-row), pero es exactamente lo que phase1:260-264 dice que
   no se hace: *"Anonymous public browsing goes through a dedicated public
   view/RPC that only surfaces the public shape, never this table's RLS"*.
   `service_plans` es **la primera tabla del proyecto que rompe ese patrón** —
   y el comentario que lo justifica (phase17:1566-1570) argumenta bien que
   "nada privado joinea desde acá", que es cierto, pero no cubre ni el
   `is_active` ni las columnas de auditoría.

**Propuesta:** `service_plans_select_public` pasa a
`is_organization_member(organization_id)`, se revoca el
`grant select … to anon`, y se crea la vista

```sql
create view public.service_plans_public as
select sp.id, sp.organization_id, sp.service_id, sp.name, sp.description,
       sp.price, sp.plan_kind, sp.weekly_quota, sp.billing_type,
       sp.billing_cycle, sp.sort_order
from public.service_plans sp
join public.services s on s.id = sp.service_id and s.is_active
join public.organizations o on o.id = sp.organization_id and o.is_active
where sp.is_active;
```

Mismo patrón que `services_public`, cuesta una vista, y la Fase H recién salió
así que el costo de corregirlo ahora es mínimo.

**No es bloqueante para ADR-0026.** Lo reporto porque se me pidió y porque es
exactamente la clase de error que esta ADR **no puede** cometer con `phone`.

---

## 6. Interacción con ADR-0024 y con la Fase I

### 6.1 Lo que un cliente gestionado **sí** puede tener

- **Pagos**: `payments_insert_staff` solo pide membership (phase7:527). El
  mostrador le carga el pago igual. ✔ sin cambios.
- **Planes** (ADR-0024): un `Payment` se ancla a un `service_plan_id`, que no
  sabe ni le importa si el customer tiene identidad. ✔ sin cambios.
- **Reservas fijas** (`RecurringBooking`): las crea el mostrador vía las RPC
  admin de ADR-0018, que resuelven el customer por `p_customer_id` + membership
  y no por `auth.uid()`. ✔ sin cambios.
- **Cuota semanal**: se mide contra series en vigencia para la fecha local del
  slot. Nada de eso mira `profile_id`. ✔ sin cambios.
- Cuenta contra **`max_customers` del plan del SaaS** (`customers_plan_limit`,
  phase10:239). Ver §7.1.

### 6.2 Lo que **no** puede, por construcción

`book_slot()`, `create_recurring_booking()`, `cancel_booking()` como cliente, y
`my_bookings/my_payments/my_services/my_entitlements`: todas resuelven por
`profile_id = auth.uid()` y una fila `NULL` nunca matchea. **Eso es
exactamente lo decidido**, y es bueno que salga solo del modelo en vez de
necesitar un chequeo explícito: no hay nada que alguien pueda olvidarse de
poner.

### 6.3 ¿Puede tener créditos de recupero si no puede entrar a la app?

**Sí, y el mecanismo ya está previsto.** ADR-0025 dice que el crédito se emite
cuando se cancela con anticipación una `Booking` de una serie, *"y el cancelador
es el cliente (**o el mostrador en su nombre**)"*. El gestionado avisa por
WhatsApp o por teléfono y el mostrador cancela. No hace falta nada nuevo.

**Qué significa para la auditoría de `cancelled_by`** — y acá hay que corregir
una lectura que parece obvia y no lo es:

`bookings.cancelled_by` **no es** el enum `CUSTOMER | ORGANIZATION` que
describe ADR-0010. En el schema real es un **FK a `profiles`** (phase5:32), y
la propia migración documenta la divergencia (phase5:5-11): todas las
migraciones construidas usan `cancelled_by` como "qué persona concreta" y
dejan la categoría de actor en `cancellation_reason` (`CUSTOMER_REQUEST` vs.
`SLOT_CANCELLED` / `RULE_DISCONTINUED`).

Con eso, para un cliente gestionado:

- `cancelled_by` = el profile del **staff que lo ejecutó**. Nunca va a ser él.
- `cancellation_reason = 'CUSTOMER_REQUEST'` = **a pedido del cliente**.

La fila completa se lee correctamente como *"a pedido del cliente, ejecutado
por Fulano del mostrador"*, que es **la verdad**. No hace falta ningún campo
nuevo. Lo que **sí** hace falta:

1. **La UI del mostrador tiene que preguntar el motivo, no asumirlo.**
   `cancel_booking` tiene `p_reason default 'CUSTOMER_REQUEST'` (phase15:283):
   un default silencioso en manos del staff es cómo se emiten créditos de
   recupero que nadie pidió, y —con ADR-0025— eso es dinero.
2. **Escribir el costo, porque es el argumento del punto 1 del feedback.** Para
   un cliente gestionado, la auditoría de "quién canceló" degrada de *quién lo
   decidió* a *quién lo ejecutó y qué dijo que era el motivo*. No hay prueba de
   que el cliente avisó a tiempo más allá de la palabra del mostrador. **Ése es
   el costo real de no exigir cuenta**, y es precisamente por qué la activación
   existe: para que ese costo sea temporal y opcional, no la regla.

### 6.4 Cross-tenant

ADR-0006 dice que no puede haber un "tenant de sesión" fijo para un `CUSTOMER`
porque un profile puede ser cliente de varias organizaciones. Esta ADR no lo
toca: el canje ata un profile a **una** fila de **una** organización, y un
mismo profile puede activarse en N organizaciones sin ninguna interacción entre
ellas.

---

## 7. Casos límite, contestados

### 7.1 Gestionado que nunca activa y acumula historial

**Es el estado normal, no un error.** No expira, no se degrada, no se le pide
nada, no se le manda un recordatorio automático. El único límite real: cuenta
contra `max_customers` del plan del SaaS (trigger `customers_plan_limit`,
phase10:239). Un dueño que carga 200 teléfonos "por las dudas" en el plan
`starter` (50 clientes) se queda sin poder dar de alta clientes reales, y el
error que ve es `PLAN_LIMIT_REACHED: clientes (50/50)`.

**No cambiar el trigger** (un gestionado es un cliente y consume soporte igual):
avisarlo en la UI de alta y mostrar el contador, que ya existe en
`organization_usage()`.

### 7.2 Activación de alguien que ya es cliente **de otra** organización con cuenta

Completamente normal y soportado sin nada extra. `customers` es por
organización y `unique (organization_id, profile_id)` no lo impide: el canje
ata ese profile a una segunda fila, en otra organización. Ninguna de las dos se
entera de la otra. Es exactamente el escenario para el que ADR-0006 prohibió el
"tenant de sesión".

### 7.3 Teléfono repetido **dentro** de una organización

Prohibido por `unique (organization_id, phone) where phone is not null`. El
alta con un teléfono existente **no crea fila**: devuelve la existente, y la
reactiva si estaba dada de baja — mismo comportamiento que
`enroll_customer_by_email` (phase8:85-96), por el mismo motivo (no partir el
historial en dos).

De paso cierra un abuso tonto: un STAFF no puede cargar el teléfono de un
cliente que ya existe para que el link salga por su propio flujo.

**Sub-caso:** el teléfono cae sobre una fila que **ya tiene `profile_id`** (la
persona se registró sola y el mostrador le carga el teléfono igual). El alta
devuelve esa fila y **no emite token**: `CUSTOMER_ALREADY_ACTIVE`. No hay nada
que activar.

### 7.4 Teléfono repetido **entre** organizaciones

Permitido y esperado. ¿Es la misma persona? Casi seguro que sí.
**¿Importa? No, y no debe importar.**

Deducir identidad entre tenants a partir del teléfono sería una **fuga
cross-tenant por construcción**: le diría al estudio A que su clienta también
va al estudio B. No hay ningún índice, ninguna vista y ninguna RPC que cruce
`customers` por teléfono entre organizaciones, y esto queda escrito como
**invariante**, no como omisión — para que nadie lo "arregle" después creyendo
que es un bug de deduplicación.

(La misma persona puede tener, además, dos teléfonos distintos en dos negocios.
Tampoco importa.)

### 7.5 Dar de baja a un gestionado con activación pendiente

La baja (`is_active = false` + `cancelled_at/by/reason`) **revoca el token en
el mismo acto**: `revoked_at = now()`. Si no, el link sigue vivo y alguien se
vincula a una fila dada de baja — un vínculo que además no aparece en ninguna
pantalla.

Se implementa como **trigger `AFTER UPDATE` sobre `customers`**, no como "hay
que acordarse de hacerlo en la RPC": hay varios caminos de escritura
(`customers_write_staff` permite `PATCH` directo por PostgREST), mismo criterio
que `set_updated_at` (phase1:16).

### 7.6 ¿Qué pasa con el link si el dueño da de baja al cliente?

Muere, por 7.5. El canje devuelve **`ACTIVATION_REVOKED`** (no
`INVALID_TOKEN`), que es lo que la pantalla necesita para decir *"pedile al
negocio un link nuevo"*.

Y si después lo vuelve a dar de alta, **hay que reenviar**: el token viejo no
resucita. Es la decisión correcta — un token que revive es un token que alguien
creía muerto.

### 7.7 El cliente activa y después borra su cuenta de la plataforma

Con `on delete set null` (Hallazgo C) vuelve a ser un gestionado, con su
historial intacto. Sin ese cambio, se lleva puestas sus reservas y sus pagos en
cascada.

### 7.8 Doble canje, carrera, sesión ajena

Resueltos en §2.4.

---

## 8. Qué **no** deberíamos construir

Se me pidió discutir el diseño antes que justificarlo. Ocho cosas que rechazo:

1. **Contraseña temporal por WhatsApp.** Sería una credencial permanente en un
   canal que no controlamos y que queda en dos historiales de chat. Un token de
   un solo uso con vencimiento corto es estrictamente mejor, y encima no hay
   que forzar un cambio de contraseña después.
2. **Login por teléfono / OTP en esta fase.** Es otro proveedor de identidad
   (Supabase Auth lo soporta, pero cambia el modelo: el teléfono pasa a ser
   identificador de `Profile`), o sea ADR propio y decisión estructural. Además
   cuesta plata por SMS, que es justo lo que el mecanismo de WhatsApp vino a
   evitar (ADR-0021: el margen es USD 14/mes por cliente).
3. **"Activar sin cuenta"** — un magic link que deje operar el portal sin crear
   un `Profile`. Rompe el punto 1 ya decidido, y técnicamente: sin `profile` no
   hay `auth.uid()`, y sin `auth.uid()` las cinco policies de dos capas de
   ADR-0006 se quedan sin anclaje. Sería reinventar la sesión, peor.
4. **Auto-merge en el canje.** Justificado en §2.5.
5. **Derivar el token del teléfono** (ej. `hmac(phone, secreto_org)`). Parece
   cómodo —"reenviar el mismo link"— y es una catástrofe: el espacio de
   teléfonos es enumerable, así que quien tenga el secreto puede generar el
   link de **cualquier** cliente. Y un secreto por organización que vive en la
   base es un secreto que se filtra con un backup.
6. **Un endpoint público que diga si un teléfono ya es cliente**, ni siquiera
   "para mejorar el alta". Es un oráculo de pertenencia sobre dato personal, y
   convierte la lista de clientes de un negocio en algo consultable con una
   agenda de teléfono.
7. **Mandar el token por email además de por WhatsApp "por las dudas".**
   Duplica el canal de fuga sin subir la tasa de activación.
8. **El token en el path de una ruta indexable** (`/activar/<token>` cacheado o
   compartible). Va en query param, se convierte en cookie en el primer GET, y
   la página lleva `no-store` + `noindex` + `Referrer-Policy: no-referrer`
   (§2.4).

---

## 9. Consecuencias

### 9.1 Positivas

- El dueño puede cargar su agenda entera el primer día. El producto deja de
  exigir que 40 personas se registren antes de servir para algo.
- `Customer` gana un estado que el dominio **ya necesitaba** para otra cosa:
  una persona que borra su cuenta deja de destruir el historial económico del
  negocio (Hallazgo C).
- Se corrige un agujero de autorización real (Hallazgo A) que hoy está latente.
- La activación es **opt-in y gradual**: nadie queda bloqueado si nunca activa.

### 9.2 Negativas / costos aceptados

- **`drop not null` es de ida.** A partir del primer cliente gestionado, volver
  atrás requiere inventar identidades. Hay que decirlo: no hay rollback.
- **Auditoría degradada** para gestionados (§6.3): "quién canceló" pasa a ser
  "quién lo ejecutó".
- **Un secreto portador nuevo** en el sistema, con toda la superficie que eso
  implica (§10).
- **El token pasa por Meta.** Aceptado, inherente al mecanismo elegido.
- El error de tipeo del número **no está completamente mitigado** (R3) y no lo
  puede estar: el canal de entrega lo elige una persona.
- Tres funciones y una policy preexistentes hay que tocarlas en la misma
  migración (Hallazgos A, B, E), lo que engorda la migración 19.

---

## 10. Riesgos de seguridad y su mitigación concreta

| # | Riesgo | Vector concreto | Mitigación |
|---|---|---|---|
| **R1** | Token portador robado | screenshot, reenvío del chat, backup del teléfono, alguien que agarra el celular | 256 bits CSPRNG · uso único · 72 h · revocable · **hash** en la base · confirmación explícita con nombre del negocio y del cliente antes de vincular |
| **R2** | Token en logs / URLs / `Referer` | query param que queda en historial, logs de proxy, `Referer` hacia un tercero | cookie `httpOnly`+`Secure` en el primer GET y redirect sin token · `Referrer-Policy: no-referrer` · `Cache-Control: no-store` · `noindex` · `returnTo` por allowlist (ADR-0015) · prohibido loggear el token |
| **R3** | Número mal tipeado → el link llega a un tercero | error de mostrador | confirmación del número completo en paso propio antes de abrir WhatsApp · estado visible · revocación de un clic · `unlink_customer_profile()` OWNER-only. **No mitigado del todo** — falla inherente del canal |
| **R4** | Canje apuntado a otra fila de cliente (IDOR) | `{"p_customer_id": "<ajeno>"}` | **La RPC no recibe ningún `customerId`**: la fila sale del token, la identidad de `auth.uid()` (ADR-0005) |
| **R5** | Doble canje / carrera | dos taps, dos pestañas, retry de red | `for update` + `redeemed_at is null` + `where profile_id is null` en el `update` — tres guardas independientes; idempotente para el mismo profile |
| **R6** | STAFF se ata a sí mismo la fila de un cliente | `PATCH /customers?id=eq.…` con `profile_id` (la policy solo mira `organization_id`) — **Hallazgo E** | trigger que prohíbe cambiar `profile_id` fuera de las tres RPC autorizadas (GUC `app.allow_profile_link`) |
| **R7** | Cancelación de la reserva de un gestionado por cualquier autenticado | **Hallazgo A**: `NULL <> auth.uid()` salta el `raise` | reescritura de `cancel_booking()` con la forma de tres ramas de `cancel_recurring_booking` |
| **R8** | Fuga del teléfono | una vista pública nueva, un join, una RPC que "solo agrega un campo" | ninguna superficie `anon` toca `customers` (verificado, §5.1) · reglas escritas en `security.md` · `customer_activations` **sin `grant select`** |
| **R9** | Enumeración / fuerza bruta del canje | POST repetido a `claim_customer_activation` | 256 bits hacen la fuerza bruta inviable (el límite es contra ruido, no contra adivinación) · rate limit por IP — **pendiente de infraestructura que no existe** |
| **R10** | Emisión masiva de tokens | OWNER comprometido, bug de la UI, loop | límite por organización dentro de la propia RPC (30/h, 200/día) + `max_customers` del plan (phase10:239) |
| **R11** | Suplantación vía merge | canje que fusiona historial automáticamente | el canje **no fusiona**: devuelve `ALREADY_CUSTOMER_OF_ORGANIZATION`. El merge es OWNER-only, explícito, y no repunta lo que chocaría |
| **R12** | Link vivo sobre un cliente dado de baja | baja + link previo sin revocar | trigger de revocación en cascada al desactivar (§7.5) |
| **R13** | Token legible por un STAFF de la organización | `select * from customer_activations` | `token_hash` + **sin `grant select`** sobre la tabla; el panel lee una vista de estado sin el hash |
| **R14** | RPC nueva callable por `anon` sin querer | `EXECUTE` a `PUBLIC` por default en Postgres (Hallazgo D) | `revoke execute … from public, anon` explícito en **cada** función nueva, antes del `grant` |

---

## 11. Alternativas descartadas

| Alternativa | Por qué no |
|---|---|
| **A. Tabla aparte `managed_customer_contacts`**, dejando `customers.profile_id` NOT NULL | Parte el `Customer` en dos y obliga a toda RPC a unirlos; el `unique (organization_id, phone)` quedaría en una tabla distinta de la de negocio. Y no resuelve nada: la fila de negocio sigue necesitando existir sin profile |
| **B. Profile "fantasma"**: crear un `auth.users` con email sintético que no puede loguearse | Contamina la tabla de identidad de la plataforma con cuentas que no son personas, y el día que alguien registre ese email de verdad hay un choque irresoluble. `handle_new_user` (phase1:47) crearía un profile real, o sea resuelve el síntoma metiendo basura en la tabla más sensible |
| **C. Código corto canjeable estilo `organization_invites`** (10 chars legibles, texto plano) | Ese código está diseñado para **leerse por teléfono**, y por eso es corto, legible y en claro. El nuestro viaja en un link que nadie tipea: no hay ninguna razón para bajarle la entropía ni para guardarlo en claro, y el peor caso es otro (suplantar a una persona vs. un plan regalado). **Sí se hereda** la mecánica de canje (`for update` + `redeemed_at`, phase10:302-344), ya probada. **No se hereda** `random()` como fuente de entropía (phase10:429) |
| **D. Exigir cuenta siempre** (statu quo) | Descartada por el usuario: punto 1 del feedback ya resuelto |
| **E. No exigir cuenta nunca** (reservar con nombre + teléfono, como las apps que el cliente probó) | Sin identidad no hay auditoría de cancelación, no hay destinatario para un crédito de recupero (ADR-0025) y el portal del cliente deja de existir. Es el camino que el usuario evaluó y descartó |
| **F. Confirmación obligatoria por últimos 4 dígitos** | **No defiende el caso que motiva la medida** (el tipeo): compara contra el número tipeado, que es el que la persona equivocada tiene. Solo defiende el link reenviado, ya bastante acotado. Análisis completo en §2.2 |
| **G. Token derivado del teléfono** | §8.5 |
| **H. Auto-merge en el canje** | §2.5 |

---

## 12. Migración que implica (migración 19, aditiva)

Ningún paso cambia el comportamiento de una fila existente. Mismo patrón
aditivo de ADR-0022/ADR-0024, que es el que no rompió tests de a 16.

1. `alter table customers alter column profile_id drop not null`.
2. `add column display_name text, phone text, claimed_at timestamptz, merged_into_customer_id uuid references customers(id)`.
3. CHECKs: `customers_identity_or_name`, `customers_phone_e164`,
   `customers_claimed_requires_profile`.
4. `unique (organization_id, phone) where phone is not null`.
5. FK `profile_id` → **`on delete set null`** (drop + add) — Hallazgo C.
6. `create table customer_activations` + índice único parcial de "un solo link
   vivo" + RLS habilitada **sin `grant select` a nadie**; vista/RPC
   `customer_activation_status` para el panel (sin `token_hash`).
7. Trigger `AFTER UPDATE` en `customers`: desactivar un cliente revoca sus
   activaciones pendientes (§7.5).
8. Trigger `BEFORE UPDATE` en `customers`: prohibir cambio de `profile_id`
   fuera de las RPC autorizadas (Hallazgo E).
9. RPCs, cada una con `revoke execute … from public, anon` **antes** del
   `grant` puntual: `create_managed_customer`, `issue_customer_activation`,
   `revoke_customer_activation`, `claim_customer_activation`,
   `unlink_customer_profile`, `merge_customers`.
10. Ajuste de `enroll_customer_by_email`: si ya existe una fila **gestionada**
    con ese teléfono, no crear una segunda — devolverla para que el mostrador
    decida (o vincular, si el profile resuelto coincide y la fila gestionada no
    tiene ninguno). Sin esto, el camino viejo de alta reintroduce el duplicado
    que el `unique` de teléfono viene a prevenir.
11. **Hallazgo A**: reescritura de `cancel_booking()`. **Bloqueante.**
12. **Hallazgo B**: `left join` + `coalesce` en `organization_customers`,
    `occurrence_bookings`, `organization_payment_summary`,
    `customer_payment_detail` y `schedule_rule_standing_reservations`.
    **Bloqueante.**
13. *(Opcional, recomendado)* acotar `service_plans` a vista pública (§5.4).

**Backfill: ninguno.** Todas las filas actuales tienen `profile_id`, así que el
CHECK `customers_identity_or_name` se cumple trivialmente al 100 %.
`display_name` y `phone` quedan `NULL` y el nombre sigue saliendo de
`profiles.full_name` vía el `coalesce`. **Riesgo de migración: nulo.**

**Rollback:** `drop not null` no se revierte si ya hay gestionados. Sin
mitigación posible, y es honesto decirlo.

### Tests que la Fase J no puede cerrar sin tener

1. Un `profile_id` NULL **no** vuelve visible la fila de un cliente a otro
   cliente de la misma organización (las cinco policies de dos capas).
2. **Hallazgo A**: un autenticado ajeno **no** puede cancelar la reserva de un
   cliente gestionado. (Este test falla hoy contra el código actual si se
   aplica solo el `drop not null` — es el test que prueba que el fix entró.)
3. Dos canjes concurrentes del mismo token: exactamente uno gana.
4. Canje con sesión de un profile que ya es customer de esa organización →
   `ALREADY_CUSTOMER_OF_ORGANIZATION` **y el token sigue vivo**.
5. Token vencido, revocado y ya canjeado → tres errores distintos, ninguno
   vincula.
6. Reenviar invalida el anterior (el token viejo devuelve `ACTIVATION_REVOKED`).
7. Dar de baja al cliente revoca su activación pendiente.
8. **Hallazgo E**: un STAFF **no** puede cambiar `profile_id` por PostgREST
   directo.
9. Un cliente gestionado aparece en Clientes, en la lista del turno y en el
   resumen de cobranza (Hallazgo B).
10. `phone` no aparece en ninguna respuesta de `anon`: `organizations_public`,
    `services_public`, `get_public_availability`, `public_slot_detail`,
    `service_plans`.
11. Un gestionado no puede reservar (`NOT_A_CUSTOMER`) ni ve nada en `my_*`.

---

## 13. Qué necesita del Orchestrator

Decisiones que no tomo yo (CLAUDE.md: modelo de dominio, schema, autenticación
y autorización):

1. **Aprobar el shape**: `profile_id` nullable + `display_name` + `phone` +
   `claimed_at` + `merged_into_customer_id`, con sus constraints y el
   `unique (organization_id, phone)`.
2. **Aprobar los gates de rol** de §2.3: STAFF para alta / emisión / revocación;
   OWNER para unlink y merge.
3. **Decidir si `merge_customers()` entra en la Fase J** o se difiere.
   *Recomiendo que entre*, aunque sea sin pantalla propia: sin ella el remedio
   al caso 7.3/§2.5 es SQL a mano en producción.
4. **Decidir sobre el rate limiting** (R9/R10). Es la única mitigación que
   necesita infraestructura **que hoy no existe** en el repo (ADR-0008 ya la
   pidió para disponibilidad pública y sigue pendiente). Si no entra, el de
   emisión se puede resolver dentro de la RPC y el de canje queda como riesgo
   residual **aceptado por escrito**.
5. **Decidir sobre la confirmación por últimos dígitos.** *Recomiendo que no*
   (§2.2), y si va, que sea flag por organización con default OFF.
6. **Decidir qué hallazgos entran en la migración 19.** Mi posición: **A y B son
   bloqueantes** de la Fase J (A es un agujero que esta ADR abre; B hace que la
   feature no se vea); **C cuesta dos líneas y la ocasión está servida**; **E es
   bloqueante** porque es el bypass directo del token.
7. **Decidir sobre `service_plans_select_public using (true)`** (§5.4).
   *Recomiendo acotarlo*; no bloquea ADR-0026.
8. **Confirmar el TTL del token: 72 h.**

### Lo que no pude decidir

- **El texto exacto del mensaje de WhatsApp** y si lleva el nombre del cliente.
  Es decisión de producto/UX con una arista de privacidad: el nombre en el
  `text=` del deep link hace que un link mal enviado le revele a un desconocido
  el nombre de otra persona. *Inclinación:* el mensaje lleva el nombre del
  **negocio** y no el del cliente; el nombre del cliente aparece recién en la
  pantalla de activación, detrás del token.
- **Si la organización debería poder desactivar el mecanismo entero** (un rubro
  sensible —salud mental, adicciones— puede no querer que exista un link que
  diga "sos cliente de X" circulando por WhatsApp). Es genuinamente un caso de
  genericidad del producto, no un detalle. No lo diseño acá; lo levanto.
- **Rate limits numéricos finales** (30/h, 200/día, 10/15 min son propuestas
  razonadas, no medidas).
- **`customer_payment_detail()`**: no leí su cuerpo completo; sigue el mismo
  patrón que sus hermanas de phase16 y presumo el mismo INNER JOIN. Verificar
  al implementar.

---

## 14. Actualización de `domain.md` que implica

- **`Customer` cambia de definición.** Deja de ser *"un `Profile` en tanto
  cliente de una `Organization`"* y pasa a ser **"la relación de una persona
  con una `Organization`, que puede o no tener una identidad de plataforma
  detrás"**, con dos estados: **gestionado** (`profile_id is null`) y
  **activado**.
- **Entidad nueva: `CustomerActivation`** — el permiso de un solo uso, con
  vencimiento, para que una persona reclame una fila de `Customer` existente.
  No es una invitación a la plataforma (eso es `organization_invites`): es un
  vínculo a una fila que **ya tiene historial**.
- **Tabla de auditoría por entidad**: `Customer` suma `claimed_at` y la nota
  sobre `merged_into_customer_id`.
- **Invariantes nuevas:**
  - Un `Customer` sin `profile_id` tiene que tener `display_name`.
  - Un teléfono identifica a lo sumo un `Customer` **por organización**.
  - El mismo teléfono en dos organizaciones **no** se deduplica, no se cruza y
    no se infiere: hacerlo sería una fuga cross-tenant.
  - Un cliente gestionado **no puede autenticarse**: todo camino de
    autoservicio resuelve por `profile_id = auth.uid()` y una fila `NULL` nunca
    matchea.
  - Nombre efectivo = `coalesce(display_name, profiles.full_name, 'Sin nombre')`,
    con `display_name` ganando.
  - Un canje **nunca** fusiona filas.
- **En "Pago ≠ permiso"**: un cliente gestionado puede tener pagos, planes y
  reservas fijas. Lo que no puede es **decidir** — reservar, cancelar o liberar
  un cupo por sí mismo.
- En "Pendiente de definir": sacar nada; agregar la pregunta abierta de si una
  organización puede desactivar el mecanismo (§13).

## 15. Actualización de `security.md` que implica

- **En "Roles"**: `CUSTOMER` se desdobla en **gestionado** (sin identidad; el
  mostrador opera en su nombre; no ve nada) y **activado**. `OWNER`/`STAFF` sin
  cambios.
- **Sección nueva: "Activación de cliente por token portador"** — el diseño
  completo de §2.2/§2.4: entropía CSPRNG, hash en reposo, un solo uso, TTL,
  invalidación al reenviar, revocación en cascada, el canje que no acepta
  `customerId`, y el transporte por cookie en vez de query param.
- **En "Público vs. privado"**: `customers.phone` y `customers.display_name` a
  la lista de datos que un anónimo **nunca** ve, más la regla general:
  **ninguna superficie `anon` joinea `customers`, `bookings`, `payments` ni
  `customer_activations`** — solo agregados.
- **En el checklist de revisión, dos líneas nuevas:**
  - *"¿Esta comparación con `auth.uid()` se comporta bien si la columna es
    `NULL`? En una policy, `NULL` deniega; en un `if` de plpgsql, `NULL`
    **salta** el `raise`."* (la lección del Hallazgo A, vale para todo el repo).
  - *"¿Esta función nueva tiene su `revoke execute … from public` antes del
    `grant`?"* (Hallazgo D).
- **En "Multi-tenancy"**: nota de que la deduplicación por teléfono entre
  organizaciones está **prohibida a propósito**.
- **Nota de auditoría**: para un cliente gestionado, `bookings.cancelled_by` es
  siempre el staff que ejecutó, y la categoría de actor la lleva
  `cancellation_reason`. Es el costo explícito de operar sin identidad, y el
  motivo por el que el mostrador **debe** elegir el motivo en vez de aceptar el
  default de `cancel_booking`.
- **Corregir en `security.md`** la descripción de `cancelledBy` como enum
  `CUSTOMER | ORGANIZATION`: el schema real usa un FK a `profiles` desde
  siempre, y la divergencia está documentada en phase5:5-11 pero no en los docs.
