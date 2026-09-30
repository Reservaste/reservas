# Propuesta ADR-0034 — Alta de equipo sin cuenta previa: `team_invitations` con link de activación

Estado: **Propuesta** (no implementada)
Propuesta por: `backend-engineer`
Fecha: 2026-09-23
Toca: autenticación, autorización, base de datos, contratos → **decisión del
Orchestrator** (CLAUDE.md)
Depende de: ADR-0026 (patrón de activación por link), ADR-0005 (la RPC de canje
no recibe un id), ADR-0028 (`revoke execute from public, anon`)
Relacionada: ADR-0033 (roles configurables) — ver §7, **no la bloquea**

---

## 1. Problema

Pedido textual: *"Para invitar nuevos perfiles necesariamente debe tener el
registro. Deberíamos agregar la misma funcionalidad de nuevo usuario sin
registro (quizás crearlos desde el backoffice y mandar un WhatsApp con una
contraseña temporal, que dure 24hs, o un link de activación vinculado a ese
email para que termine el registro)."*

**Confirmado en el código, no supuesto.** `invite_member_by_email()`
(`phase8:112`):

```sql
select id into v_profile_id from auth.users where lower(email) = lower(trim(p_email));
if v_profile_id is null then
  raise exception 'PROFILE_NOT_FOUND';
end if;
```

La persona tiene que existir en `auth.users` antes de poder ser invitada. El
server action lo traduce a un error de pantalla
(`frontend/app/actions/admin.ts:453-471`). O sea: el dueño no puede dar de alta
a un profesor; tiene que pedirle que se registre primero, esperar, y recién ahí
invitarlo — con el email exacto con el que se registró.

Es exactamente el problema que ADR-0026 resolvió para clientes, del otro lado
del mostrador. El comentario de `enroll_customer_by_email()` (`phase8:56-58`) lo
anticipaba: *"MVP limitation: the person must already have an account. Sending
an invitation to a stranger's email is a separate feature (...) deliberately not
invented here."* Esta propuesta es esa feature, para el caso equipo.

## 2. Mecanismo separado, no reutilizar `customer_activations`

**Recomendación: tabla y RPCs propias (`team_invitations`), con el mismo patrón
de ADR-0026.** La sospecha del Orchestrator es correcta, y los motivos son
estructurales, no de gusto:

1. **El destino del token no existe todavía.**
   `customer_activations.customer_id` es `not null references customers`: el
   cliente gestionado **ya existe**, se le agenda y se le cobra desde antes de
   activar; el token solo le pega una identidad encima. Una invitación de equipo
   no tiene fila a la que apuntar — la fila de `organization_members` **nace en
   el canje**. Reusar la tabla obligaría a hacer `customer_id` nullable y sumar
   `organization_id`, `email` y `role_id`, con un CHECK que mantenga separadas
   dos formas mutuamente excluyentes. Una tabla con dos significados es la
   receta para que una policy pensada para uno aplique mal al otro.
2. **Lo que el token otorga es distinto en naturaleza, y eso es una diferencia
   de seguridad.** El de ADR-0026 otorga *"sos este cliente"*: acceso a las
   reservas y pagos **propios**. El de equipo otorga acceso a **los datos de
   otras personas** en todo el tenant — padrón, teléfonos, cobranza. Mismo
   mecanismo, radio de explosión mucho mayor. Tablas separadas permiten
   endurecer TTL, rate limit y revocación del de mayor riesgo sin volver a
   testear el flujo de clientes, que ya está en producción.
3. **La unicidad "un link vivo por X" es sobre otro X.** Para clientes es
   `unique (customer_id) where redeemed_at is null and revoked_at is null`
   (`phase21:151`). Para equipo tiene que ser
   `(organization_id, lower(email))`, porque el destino es un email y no una
   fila.
4. **Lo que sí conviene compartir es el acuñado del token, no la tabla.** Un
   helper `new_activation_token()` que devuelva `(token text, hash bytea)` con
   la decisión de ADR-0026 adentro (256 bits de
   `extensions.gen_random_bytes`, base64url sin padding, `sha256` en reposo,
   schema-calificado para no depender del `search_path`). Es la parte que no
   debe divergir entre los dos flujos y la única que no tiene acoplamiento con
   el destino.

## 3. Link de activación, no contraseña temporal

El pedido menciona las dos opciones. **Recomendación: link**, y no por inercia
con ADR-0026 — los motivos son propios de este caso:

1. **Una contraseña temporal nos obliga a crear la cuenta nosotros, con una
   contraseña que conocemos.** Hay que llamar al Admin API de Supabase Auth con
   la `service_role` key desde el servidor — una llave que hoy **no está en
   ningún camino de request de esta aplicación** (así lo dice el comentario de
   `phase21:169-172`) y meterla es infraestructura nueva con poderes de dios,
   para un resultado peor.
2. **"Que dure 24hs" no hace efímera a la contraseña.** Acota la ventana para el
   primer login; después de ese login la contraseña **sigue siendo la
   contraseña**, viajada en texto plano por WhatsApp y guardada para siempre en
   el historial del chat, salvo que construyamos rotación forzada al primer
   ingreso — un flujo que hoy no existe. Un link, en cambio, es de un solo uso
   por construcción: se consume y deja de servir.
3. **No hay contraseña que mandar si la persona entra con Google.** El producto
   soporta email/password **y** Google OAuth (ADR-0002). El camino de contraseña
   temporal no puede atender a la mitad de los usuarios, así que habría que
   construir igual el camino de link — dos flujos de auth para las mismas
   personas.
4. **El canal es WhatsApp; el vínculo es el email.** Son cosas distintas y esta
   propuesta las separa a propósito (§4.3): el link se manda por donde sea
   (WhatsApp, copiar y pegar), pero el canje exige que la sesión tenga **ese**
   email. Una contraseña temporal no tiene esa propiedad: quien la tenga entra,
   punto.
5. **Coherencia.** Tener dos respuestas opuestas a la misma pregunta en el mismo
   producto es como las inconsistencias se vuelven vulnerabilidades.

**Contraargumento honesto, porque existe:** "te mando la clave" es lo que un
usuario no técnico entiende sin explicación. Mitigación: es el mismo mensaje de
WhatsApp que el negocio ya manda a sus clientes desde la Fase J y que ya
funciona en producción — no es un patrón nuevo para ese usuario, es el que ya
usa todos los días.

## 4. Cambio propuesto

### 4.1 `team_invitations`

```sql
create table public.team_invitations (
  id              uuid primary key default gen_random_uuid(),
  organization_id uuid not null references public.organizations (id) on delete cascade,
  email           text not null,              -- normalizado a lower(trim(...))
  phone           text,                       -- solo canal de envío (E.164), opcional
  -- El rol con el que la persona entra, elegido al invitar (Sec 4.4).
  role            public.organization_member_role not null default 'STAFF',
  role_id         uuid references public.organization_roles (id),   -- solo si ADR-0033
  token_hash      bytea not null unique,      -- sha256 del token, nunca el token
  expires_at      timestamptz not null,
  created_at      timestamptz not null default now(),
  created_by      uuid not null references public.profiles (id),
  redeemed_at     timestamptz,
  redeemed_profile_id uuid references public.profiles (id),
  revoked_at      timestamptz,
  revoked_by      uuid references public.profiles (id),

  -- Un link portador NUNCA puede fabricar un OWNER. Sec 5, riesgo 3.
  constraint team_invitations_never_owner check (role = 'STAFF'),
  constraint team_invitations_email_lower  check (email = lower(trim(email))),
  constraint team_invitations_phone_e164   check (phone is null or phone ~ '^\+[1-9][0-9]{6,14}$')
);

-- Un solo link vivo por (organización, email). Reenviar revoca el anterior
-- adentro de la RPC, igual que ADR-0026: el link viejo es justo el que pudo
-- haber ido al lugar equivocado.
create unique index team_invitations_one_live_idx
  on public.team_invitations (organization_id, email)
  where redeemed_at is null and revoked_at is null;

create index team_invitations_org_created_idx
  on public.team_invitations (organization_id, created_at);

alter table public.team_invitations enable row level security;
-- Sin una sola policy y sin grants a anon/authenticated, igual que
-- customer_activations (phase21:163-172): la única puerta son las RPCs
-- SECURITY DEFINER de abajo.
```

### 4.2 Las cuatro RPCs

| RPC | Rol | Qué hace |
|---|---|---|
| `invite_team_member(org, email, role_id, phone)` → `(invitation_id, token, expires_at)` | **OWNER** | acuña el token, revoca el vivo anterior para ese email, devuelve el token **una sola vez** |
| `revoke_team_invitation(invitation_id)` | **OWNER** | idempotente, no falla si ya estaba revocada o canjeada |
| `organization_team_invitations(org)` | miembro | read model de pendientes para la pantalla; **nunca** devuelve `token_hash` |
| `claim_team_invitation(token)` → `jsonb` | authenticated | el canje |

**Quién invita: `OWNER`**, igual que `invite_member_by_email()` hoy. Es una
asimetría deliberada con ADR-0026 (donde un `STAFF` emite activaciones de
cliente) y tiene motivo: allá el link otorga "ser vos mismo"; acá otorga acceso
a los datos de todos. ADR-0026 resolución 2 gateó la emisión a `STAFF`
justamente porque *"gatearlo a OWNER forzaría a compartir su cuenta"* — ese
argumento no aplica al alta de personal, que es ocasional y del dueño.

### 4.3 El canje

Aplicación literal de ADR-0005, igual que `claim_customer_activation()`: **un
solo parámetro, el token.** No nombra ninguna organización ni ninguna fila de
miembro. El destino sale entero del token; la identidad, entera de
`auth.uid()`. No hay superficie de IDOR porque no hay input que apunte a una
fila.

Orden de chequeos dentro de `claim_team_invitation(p_token)`:

1. `auth.uid() is null` → `AUTH_REQUIRED`.
2. Buscar por `token_hash = digest(token,'sha256')` **con `for update`** →
   `INVALID_TOKEN`.
3. `revoked_at` → `INVITATION_REVOKED`.
4. `redeemed_at`: si `redeemed_profile_id = auth.uid()` devolver OK
   (idempotencia de doble click); si no, `ALREADY_REDEEMED`.
5. `expires_at <= now()` → `INVITATION_EXPIRED`.
6. **El email de la sesión tiene que coincidir con el de la invitación** →
   `INVITE_WRONG_EMAIL`. Precedente exacto: `create_organization_with_owner()`
   ya hace esta comparación (`phase10:319-323`).
7. Organización activa y `organization_can_operate()` → si no,
   `ORGANIZATION_UNAVAILABLE`.
8. Rol vigente (si viene de ADR-0033 y el rol fue desactivado entre emisión y
   canje) → `INVITATION_ROLE_UNAVAILABLE`. **Falla cerrado**: no se cae al rol
   por defecto ni se adivina.
9. **Si `auth.uid()` ya es miembro activo de esa organización: devolver OK sin
   tocar nada.** Nunca cambiar el rol de un miembro existente desde un canje
   (§5, riesgo 3).
10. `insert` en `organization_members` (o reactivar la fila inactiva existente),
    con `created_by = created_by de la invitación` — quien decidió el alta es el
    que invitó, no el que hizo click.
11. `update team_invitations set redeemed_at = now(), redeemed_profile_id =
    auth.uid() where id = ... and redeemed_at is null`.

**El vínculo es el email; el canal es cualquiera.** Que el link exija la casilla
correcta convierte un secreto de un factor en uno de dos y mitiga el caso real
más probable (el número de WhatsApp mal tipeado). El costo es que si la persona
se registra con otro email (típico con Google), el canje falla con un error
claro y el `OWNER` reemite al email correcto. Es fricción visible y arreglable,
no un acceso silencioso al tenant equivocado.

### 4.4 El rol lo elige quien invita, al invitar

Va en la fila de invitación, no se decide en el canje. Tres motivos:

1. Asignar rol es OWNER-only (ADR-0033 §4.6 / policy actual
   `organization_members_write_owner`), y el `OWNER` está presente al invitar,
   no cuando la persona hace click.
2. Si se decidiera después, habría una ventana donde el miembro ya entró con
   *algún* rol: o ninguno (no puede trabajar) o el default (que puede ser más de
   lo que el dueño quería). Ninguna de las dos es aceptable.
3. La invitación describe completa su consecuencia, así la pantalla de equipo
   puede decir "Invitado como **Profesor** — pendiente desde el 12/03" en vez de
   "alguien va a entrar y después vemos".

**Sin ADR-0033**: la columna `role` es el enum y no hay nada que elegir
(siempre `STAFF`). **Con ADR-0033**: se agrega `role_id`. Por eso conviene el
orden de §7, pero no es un bloqueo.

### 4.5 El camino del link en el frontend

Calcado de ADR-0026, con una precaución concreta: **cookie y ruta propias.**
`/activar/[token]/route.ts` setea la cookie `activation_token` con
`path: "/activar"`. Si el flujo de equipo reusara esa ruta o ese nombre, una
persona que es cliente **y** a quien acaban de invitar al equipo (caso
perfectamente normal: la recepcionista que también entrena ahí) pisaría un token
con el otro. Propuesta: ruta `/equipo/[token]` → cookie
`team_invitation_token`, `path: "/equipo"`, mismos headers (`no-store`,
`no-referrer`, `X-Robots-Tag: noindex`, `httpOnly`, `Secure`, `maxAge` corto) y
misma regla: el token vive en una URL **una sola vez**, la del link, y de ahí en
más solo en la cookie.

## 5. Riesgos de seguridad, nombrados

1. **El link es un secreto portador, con radio de explosión mayor que el de
   ADR-0026.** Quien lo tenga (y tenga la casilla, §4.3) queda con acceso al
   tenant. Mitigaciones: 256 bits de CSPRNG, `sha256` en reposo, un solo uso,
   vencimiento, revocable, vinculado al email, y **nunca `OWNER`** (CHECK de
   §4.1).
2. **TTL: 24 h, no 72 h.** ADR-0026 resolución 8 fijó 72 h para clientes; acá
   recomiendo 24 h y el motivo no es "más seguro por las dudas": un alta de
   personal se coordina en tiempo real con alguien con quien estás hablando, a
   diferencia de un cliente que puede ver el mensaje al otro día. Menos ventana
   para el mismo caso de uso, con mayor consecuencia si falla. Coincide además
   con lo que pidió el usuario.
3. **El canje no puede apuntar a otro miembro, ni cambiarle el rol a uno
   existente.** Lo primero está garantizado estructuralmente: la RPC recibe solo
   el token. Lo segundo es una regla explícita (paso 9 de §4.3) y merece
   subrayarse porque **`invite_member_by_email()` hoy sí pisa el rol**
   (`phase8:141-145`: `update ... set role = p_role`). Ahí es correcto —es
   OWNER-gated y sincrónico—, pero en un canje sería un camino de escalada: un
   `STAFF` que consigue una invitación a un rol mayor, o peor, un link que
   *degrada* a alguien.
4. **Rate limit de emisión, adentro de la RPC**, mismo criterio que ADR-0026
   resolución 4 (sin infraestructura nueva): contar invitaciones de esa
   organización en la última hora y el último día. Números propuestos: **10/h y
   30/día** — un equipo son 2-20 personas, no 200 clientes; los de clientes
   (30/h, 200/día) están dimensionados para otra cosa. Como en ADR-0026, son
   propuestas razonadas, no medidas.
5. **Fuerza bruta del canje: riesgo residual aceptado**, exactamente como
   ADR-0026 resolución 4 lo dejó por escrito. 256 bits lo vuelven
   computacionalmente irrelevante, y la infraestructura general de rate limiting
   sigue siendo deuda de ADR-0008: no se construye dos veces la misma pieza por
   partes.
6. **Enumeración de emails: esto la *reduce*.** Hoy `invite_member_by_email()`
   levanta `PROFILE_NOT_FOUND`, lo que convierte al panel en un oráculo de "¿tal
   email tiene cuenta en la plataforma?". El flujo nuevo no consulta
   `auth.users` en ningún momento, así que no responde esa pregunta.
7. **Límite de plan: se consume en el canje, no en la invitación.**
   `enforce_plan_limit()` es un `before insert on organization_members`
   (`phase10:241`) con `max_team_members`. Con este flujo el `INSERT` ocurre al
   activar, así que un `OWNER` con un lugar libre puede emitir cinco
   invitaciones y cuatro personas se comen `PLAN_LIMIT_REACHED` al hacer click
   — un error incomprensible para quien lo recibe. **Recomendación:** chequear
   el cupo *también* al emitir, contando miembros activos + invitaciones vivas.
   El trigger sigue siendo la regla; el chequeo de emisión es el mensaje de
   error. (El mismo reparto que el proyecto ya usa: la base manda, el action
   explica.)
8. **Un pendiente es invisible hoy.** `organization_team()` hace
   `join public.profiles p on p.id = om.profile_id` (`phase8:431`), y además una
   invitación pendiente **ni siquiera tiene fila de miembro**. Sin un read model
   propio, el dueño no tiene forma de ver a quién invitó, ni de reenviar, ni de
   revocar. Es el eco exacto del hallazgo B de ADR-0026, y por eso
   `organization_team_invitations()` entra en el alcance, no en "después".

## 6. Impacto

| Área | Qué cambia |
|---|---|
| Schema | 1 tabla, 2 índices, 3 CHECK, 0 policies (RLS sin policies, a propósito). Aditivo. |
| RPCs nuevas | 4 (§4.2) + el helper de acuñado `new_activation_token()`. Todas con `revoke execute from public, anon` antes del `grant to authenticated` (ADR-0028). |
| RPCs existentes | `invite_member_by_email()` **no cambia** (§8). `organization_team()` no cambia por esta propuesta (sí por ADR-0033). |
| Contratos | Server actions nuevas en `frontend/app/actions/admin.ts` (invitar con link, revocar, listar pendientes) + `frontend/app/actions/activation.ts` o un módulo hermano para el canje. **Avisar a frontend-engineer.** |
| Frontend (UI) | Pantalla de equipo: lista de pendientes, botón de WhatsApp/copiar link, revocar. Ruta `/equipo/[token]` + página de confirmación. |
| `@reservaste/domain` | Tipo `TeamInvitation` + schemas Zod + mapper. |
| Seguridad | Revisión de `security-engineer` **obligatoria**: es auth + roles + token portador. |
| Tests (integración) | Canje feliz; token vencido; revocado; reusado por otra cuenta; email que no coincide; `OWNER` no fabricable (el CHECK); canje por alguien que ya es miembro (no cambia el rol); dos canjes concurrentes con un solo ganador; rate limit; `PLAN_LIMIT_REACHED` al emitir y al canjear; una invitación de otra organización que no sirve acá. |

## 7. ¿Depende de ADR-0033?

**No la bloquea.** Son independientes en mecanismo: el token, el canje, el TTL,
el rate limit y el read model no cambian según cómo se modele el rol.

- **Si ADR-0033 no se aprueba**: `team_invitations.role` es el enum, siempre
  `'STAFF'`, y no hay nada que elegir al invitar. Implementable tal cual.
- **Si ambas se aprueban**: conviene **implementar ADR-0033 primero**, para que
  `team_invitations` nazca con `role_id` en vez de necesitar una segunda
  migración sobre una tabla que ya tiene tokens vivos en producción (y con ella,
  decidir qué rol le corresponde a las invitaciones emitidas antes de la
  columna).

## 8. Preguntas abiertas para el Orchestrator

1. **¿Se conserva `invite_member_by_email()` como camino rápido?**
   Recomiendo **sí**. Para alguien que ya tiene cuenta, el alta es instantánea y
   quitarlo sería una regresión en un flujo que funciona. El costo es tener dos
   puertas a la misma tabla; se acepta porque ambas son OWNER-gated y sincrónicas
   y comparten las mismas invariantes a nivel base. La pantalla puede ofrecer
   una sola cosa ("invitar por email") y el action elegir el camino según si el
   email ya tiene cuenta — pero eso reintroduce el oráculo de §5.6, así que
   **recomiendo que la elección sea explícita del dueño**, no automática.
2. **¿El canje exige que el email de la sesión coincida (§4.3), o alcanza con el
   token?** Recomiendo exigirlo. Es la diferencia entre un secreto de un factor
   y uno de dos, y el caso de falla (se registró con otro email) tiene un remedio
   obvio y visible.
3. **¿TTL 24 h (recomendado) o 72 h por coherencia con ADR-0026?**
4. **¿El `phone` se guarda en la invitación o solo se usa para armar el link y se
   descarta?** Recomiendo **guardarlo**: permite reenviar sin volver a tipearlo y
   deja registro de a dónde se mandó el link — que es exactamente el dato que hace
   falta cuando alguien dice "no me llegó". Contra: es un dato personal más en una
   tabla que no lo necesita para funcionar.
5. **¿Una invitación puede crear un `OWNER`?** Recomiendo **no**, con CHECK
   (§4.1). Promover a dueño sigue siendo un acto deliberado sobre un miembro que
   ya existe y ya se autenticó.

## 9. Alternativas descartadas

- **Contraseña temporal por WhatsApp.** §3.
- **Reutilizar `customer_activations` con `customer_id` nullable.** §2.
- **Extraer un mecanismo genérico de tokens (`activation_tokens` polimórfica,
  con `target_table`/`target_id`).** Rechazada por ahora: con dos usos, la
  abstracción cuesta más que la duplicación, y obligaría a que las dos policies
  y los dos rate limits vivan en la misma tabla — justo lo que §2.2 quiere poder
  ajustar por separado. Si aparece un tercer caso, se revisita; el helper de
  acuñado (§2.4) ya cubre la parte que **no debe** divergir.
- **Mandar un magic link de Supabase Auth y resolver la membresía después.**
  Rechazada: el magic link autentica pero no dice nada de la organización ni del
  rol, así que hace falta igual una fila que diga a qué tenant y con qué rol
  entra. Sería el mismo trabajo más una dependencia de entrega de emails que el
  producto hoy no usa para esto (el canal real es WhatsApp).
- **Crear la fila de `organization_members` inactiva al invitar y activarla en el
  canje.** Rechazada: `max_team_members` se contaría sobre gente que todavía no
  entró (o habría que excluir inactivos y entonces el límite no limita), y
  `organization_team()` mostraría miembros que no existen. Una invitación
  pendiente **no es** un miembro; modelarla como uno es mentirle al padrón.
