# Propuesta ADR-0033 — Roles configurables por organización (una capa dentro de `STAFF`)

Estado: **Propuesta** (no implementada)
Propuesta por: `backend-engineer`
Fecha: 2026-09-23
Toca: autorización, multi-tenancy, base de datos, contratos → **decisión del
Orchestrator** (CLAUDE.md)
Depende de: ADR-0006 (RLS de dos capas), ADR-0028 (`GRANT` aditivo), ADR-0026
(gates de rol por RPC)
Relacionada: ADR-0034 (alta de equipo sin registro previo) — ver §9

---

## 1. Problema

Pedido textual: *"Nuevo nivel de perfil. Como se lo van a pasar a profesores no
deberían ver el tema pagos. Quiero que sea configurable por organización, roles
y nombre del rol."*

Hoy el rol es binario: `organization_member_role` es un enum de dos valores,
`OWNER` y `STAFF` (`phase1:102`). Todo lo que no está explícitamente cerrado a
`OWNER` lo puede hacer cualquier `STAFF`. La lista de lo que hoy **es**
OWNER-only, verificada en el código y no supuesta:

| Cerrado a OWNER hoy | Dónde |
|---|---|
| Editar la `Organization` (settings, branding, políticas de recupero) | policy `organizations_update_owner` (`phase1:278`) |
| Escribir `organization_members` | policy `organization_members_write_owner` (`phase1:289`) |
| `invite_member_by_email()`, `revoke_member()` | `phase8:128`, `phase8:177` |
| `unlink_customer_profile()`, `merge_customers()` | `phase21:583`, `phase21:660` |
| `grant_manual_makeup_credit()` | `phase22:1570` |
| Los 4 overrides de recupero en `services` | trigger `services_billing_override_owner` (`phase25:546`) |
| Subir/borrar el logo | policies de storage (`phase13:124`) |

**Todo lo demás es de cualquier miembro activo** (`is_organization_member()`):
pagos (`payments_select/insert/update_staff`, `phase7:517-534`), la cobranza
entera (`organization_payment_summary()`, `customer_payment_detail()`,
`customer_billing_horizon()`), reservas de mostrador, asistencia, alta y baja de
clientes, horarios, recursos.

Es decir: **hoy darle acceso al panel a un profesor es darle acceso a la
cobranza completa de la organización**. No hay forma de no hacerlo salvo no
darle acceso.

Dos observaciones que el diseño tiene que respetar y que no estaban escritas en
ningún lado:

- **`docs/security.md:470` ya tiene "Alcance exacto de permisos de `STAFF` vs.
  `OWNER`" como pregunta abierta desde la Fase 1.** Esta propuesta es la
  respuesta a esa pregunta, no un tema nuevo.
- **Dos gates de OWNER viven solo en TypeScript, no en la base.**
  `updateServiceSettings()` (`frontend/app/actions/services.ts:152`) y todo
  `service-plans.ts` (`:305`, `:324`, `:424`, `:481`) chequean
  `membership.role !== "OWNER"`, pero la RPC `create_service_plan()`
  (`phase22:508`) y las policies `service_plans_*_staff` (`phase17:1581`) solo
  exigen `is_organization_member()`. Un `STAFF` cambia el precio de un plan por
  `PATCH /rest/v1/service_plans` sin tocar el panel. Es un agujero preexistente
  y esta propuesta no lo causa, pero **no puede construirse encima de él**
  (§7, riesgo 2).

## 2. Qué NO es este problema (para no ensanchar el alcance)

- **No es una matriz de permisos general.** No se construye "permiso por
  pantalla" ni "permiso por endpoint". Se define un conjunto **cerrado y
  chico** (§4) y todo lo demás queda donde está.
- **No toca `OWNER`.** Ver §3.
- **No es un sistema de roles de plataforma.** `is_platform_admin()`
  (`phase10:92`) es otra cosa y no se toca.
- **No es auditoría.** Quién hizo qué es ADR-0032.

## 3. La intuición del Orchestrator es correcta, y por tres motivos concretos

El pedido dice "configurable por organización". La pregunta era si `OWNER`
sobrevive como enum fijo y lo nuevo es una capa **dentro** de `STAFF`.
**Sí, y no es una concesión a la compatibilidad: es la única forma segura.**

1. **`OWNER` es la raíz de confianza, y una raíz de confianza configurable no
   es una raíz.** Quien administra los roles no puede tener un rol
   administrable: si lo tuviera, existiría el ciclo "me edito el rol para
   poder editar roles". El producto ya dejó escrita esta idea en
   `revoke_member()` (`phase8:188`): *"An organization with no active OWNER is
   unadministrable: nobody could ever add one back, since adding owners is
   itself OWNER-gated."* Un `OWNER` no configurable es lo que garantiza que
   siempre exista alguien que pueda arreglar una configuración de permisos
   rota.
2. **`OWNER` nunca se evalúa contra un permiso — corta antes.** Eso permite
   que `has_org_permission()` devuelva `true` para `OWNER` sin consultar nada,
   y que un rol mal configurado no pueda dejar a la organización sin nadie que
   pueda entrar a arreglarlo.
3. **Compatibilidad exacta, cero riesgo de migración.** Las 8 referencias a
   `is_organization_owner()` y las 4 policies que lo usan no cambian una línea.
   Mismo patrón aditivo de ADR-0022/0024/0031: se agrega una capa, no se
   reescribe la existente.

**Corrección a la intuición, menor pero importante:** la capa nueva no es
"dentro de `STAFF`" en el sentido de heredar de `STAFF`. Es más simple:
`organization_member_role` sigue diciendo **quién manda** (`OWNER`) y el rol
configurable dice **qué puede hacer el que no manda**. Un `OWNER` simplemente
no consulta la segunda columna.

## 4. Cambio propuesto

### 4.1 `organization_roles`

```sql
create table public.organization_roles (
  id              uuid primary key default gen_random_uuid(),
  organization_id uuid not null references public.organizations (id) on delete cascade,
  name            text not null,              -- "Profesor", "Recepción", "Instructor"
  is_default      boolean not null default false,  -- el rol que recibe un STAFF sin rol asignado
  is_active       boolean not null default true,

  -- Los permisos, como columnas booleanas. Ver 4.2 para el porqué.
  can_view_payments      boolean not null default true,
  can_manage_payments    boolean not null default true,
  can_manage_bookings    boolean not null default true,
  can_manage_customers   boolean not null default true,
  can_manage_attendance  boolean not null default true,

  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  created_by uuid references public.profiles (id),

  -- Cobrar sin poder ver lo cobrado no es un rol, es un bug.
  constraint organization_roles_manage_implies_view
    check (not can_manage_payments or can_view_payments)
);

create unique index organization_roles_org_name_idx
  on public.organization_roles (organization_id, lower(trim(name)));

-- Exactamente un rol por defecto por organización, garantizado por la base.
create unique index organization_roles_one_default_idx
  on public.organization_roles (organization_id) where is_default;

alter table public.organization_members
  add column role_id uuid references public.organization_roles (id);
```

`role_id` es **nullable a propósito**, con la regla de resolución:

```
OWNER                      → todo, sin mirar role_id
STAFF con role_id          → ese rol
STAFF con role_id null     → el rol is_default de la organización
```

Por qué nullable y no `not null` con backfill: `organization_members` es
escribible por PostgREST directo (`organization_members_write_owner` es
`for all`), así que un `not null` rompería cualquier escritura que hoy no manda
la columna — y el proyecto ya aprendió en ADR-0028 que *lo que se confía a que
cada llamador recuerde hacer, en algún lado no se hace*. El fallback al rol por
defecto **no es fail-open**: el rol por defecto lo configura el `OWNER`, y el
día del deploy es el que preserva el comportamiento actual (§6).

### 4.2 Columnas booleanas, no `jsonb`

**Recomendación: columnas.** Los tres motivos, en orden de peso:

1. **`jsonb` reintroduce la lógica de tres valores que ya causó un bypass de
   autorización en este proyecto.** `(permissions->>'view_payments')::boolean`
   con una clave ausente o mal tipeada da `NULL`, y `invariants.md` documenta
   explícitamente (raíz de ADR-0028 y del hallazgo A de ADR-0026) que un
   `if NULL` **no entra**. Con `coalesce(..., false)` se falla cerrado pero en
   silencio: un typo en la clave apaga un permiso para todos y nadie se entera
   hasta que alguien reclama. Una columna booleana `not null` no tiene ese
   estado.
2. **El `DEFAULT` de la columna *es* el mecanismo de migración.** Agregar un
   permiso más adelante es `add column can_x boolean not null default true`, y
   todos los roles existentes conservan su comportamiento sin backfill —
   exactamente el patrón aditivo de ADR-0024/0031. Con `jsonb` hay que decidir
   en cada lectura qué significa "la clave no está", y esa decisión termina
   repetida en cada policy.
3. **El conjunto es cerrado y chico a propósito** (§4.3). `jsonb` es la
   herramienta para conjuntos abiertos; este no lo es, y si algún día lo fuera,
   eso sería su propio ADR.

**Trade-off explícito**: cada permiso nuevo es una migración. Se acepta porque
cada permiso nuevo **ya** necesita una migración de todos modos — el punto
donde se aplica es una policy o una RPC, no una pantalla.

### 4.3 Los cinco permisos del primer corte, con precisión

| Permiso | Qué habilita exactamente | Default |
|---|---|---|
| `VIEW_PAYMENTS` | `payments` SELECT; `organization_payment_summary()`, `customer_payment_detail()`, `customer_billing_horizon()`; pantallas `/org/[slug]/payments/*` | `true` |
| `MANAGE_PAYMENTS` | `payments` INSERT/UPDATE (registrar, anular) + `set_payment_status()` | `true` |
| `MANAGE_BOOKINGS` | `admin_book_for_customer()`, `admin_create_recurring_booking()`, `cancel_booking()` sobre reservas ajenas, `cancel_slot_occurrence()` | `true` |
| `MANAGE_CUSTOMERS` | `create_managed_customer()`, `enroll_customer_by_email()`, `issue/revoke_customer_activation()`, escritura de `customers` | `true` |
| `MANAGE_ATTENDANCE` | `mark_attendance()` | `true` |

**Lo que NO es configurable en este corte** (explícito, para que nadie lo
agregue "de paso"):

- **Ver el calendario, la agenda y el padrón de clientes.** Todo miembro activo
  los ve. Sin eso no hay trabajo que hacer, y es la lectura que necesita cada
  pantalla del panel: hacerla opcional duplica el tamaño de la matriz sin
  resolver ningún caso pedido. Un rol "que no vea a los clientes" es un rol que
  no puede pasar lista.
- **Invitar/revocar equipo y administrar roles** → siguen OWNER-only (§3 y §5).
- **Configuración de la organización, branding, políticas de recupero** → ya son
  OWNER-only (`organizations_update_owner`, `services_billing_override_owner`).
- **Planes y precios** → deberían ser OWNER-only en la base; hoy lo son solo en
  TypeScript (§1, §7 riesgo 2, §8 pregunta 3).
- **Suscripción/plan SaaS de la organización** → `is_platform_admin()`, otro eje.
- **Créditos manuales de cortesía** (`grant_manual_makeup_credit()`), **merge** y
  **unlink** de clientes → ya OWNER-only.

Precisión que hay que decirle al usuario: **"no ver pagos" no es "no ver
precios".** `service_plans_select_public` es
`using (is_active or is_organization_member(organization_id))`
(`phase22:565`): la lista de precios de los planes activos es **pública**, la ve
cualquier visitante del calendario. `VIEW_PAYMENTS` oculta *quién pagó, cuánto,
cuándo y quién debe*, no la lista de precios.

### 4.4 El helper: uno solo, con clave de enum

```sql
create type public.org_permission as enum (
  'VIEW_PAYMENTS', 'MANAGE_PAYMENTS', 'MANAGE_BOOKINGS',
  'MANAGE_CUSTOMERS', 'MANAGE_ATTENDANCE'
);

create or replace function public.has_org_permission(
  p_organization_id uuid,
  p_permission public.org_permission
) returns boolean
language sql security definer stable set search_path = public
as $$
  select exists (
    select 1
    from public.organization_members om
    left join public.organization_roles r
      on r.id = coalesce(
           om.role_id,
           (select d.id from public.organization_roles d
             where d.organization_id = om.organization_id and d.is_default limit 1)
         )
    where om.organization_id = p_organization_id
      and om.profile_id = auth.uid()
      and om.is_active
      and (
        om.role = 'OWNER'                       -- corta antes de mirar el rol
        or (r.is_active and case p_permission
              when 'VIEW_PAYMENTS'     then r.can_view_payments
              when 'MANAGE_PAYMENTS'   then r.can_manage_payments
              when 'MANAGE_BOOKINGS'   then r.can_manage_bookings
              when 'MANAGE_CUSTOMERS'  then r.can_manage_customers
              when 'MANAGE_ATTENDANCE' then r.can_manage_attendance
              else false                        -- falla cerrado
            end)
      )
  );
$$;
```

Mismas propiedades que `is_organization_member()`: `security definer` para
romper la recursión de RLS que documenta `phase1:204-213`, `stable` para que
Postgres la cachee dentro del statement, y **con `PUBLIC` a propósito** — es una
de las funciones "correctas por construcción que RLS necesita evaluar como
`anon`" que ADR-0028 dejó explícitamente exceptuadas del `revoke from public`.

La clave es un **enum, no `text`**: un typo en una policy (`'VIEW_PAYMENT'`)
falla en la migración con `invalid input value for enum`, no en producción con
un permiso que silenciosamente deniega. El `else false` es el segundo cinturón.

### 4.5 Dónde se aplica: RLS **y** RPC, nunca solo UI

Esta es la parte que decide si la feature sirve o es decorado, y tiene una
trampa concreta:

- **Las policies** se extienden: `payments_select_self_or_staff` pasa a
  `(rama del customer) or (is_organization_member(organization_id) and
  has_org_permission(organization_id, 'VIEW_PAYMENTS'))`. La rama del customer
  **no se toca** — ADR-0006 de dos capas sigue igual.
- **Las RPCs `security definer` NO se benefician de eso.** Corren como dueñas de
  la tabla y **bypassean RLS por completo**. `organization_payment_summary()`,
  `customer_payment_detail()` y `customer_billing_horizon()` chequean hoy
  `is_organization_member()` adentro; si solo se endurece la policy, esas tres
  siguen devolviendo la cobranza entera a un rol al que la pantalla le esconde
  la pestaña. **Cada una tiene que cambiar su propio chequeo.** Es el error que
  haría de esto una falsa sensación de seguridad, y por eso va como lista
  explícita en §6 y como test por RPC.
- **La UI solo esconde.** El proyecto ya tiene la frase escrita en
  `enforce_plan_limit()` (`phase10:170`): *"Hiding the button is not
  enforcement: the RPCs and PostgREST are callable directly."*

Para columnas sensibles dentro de una tabla que sigue siendo escribible por
`STAFF`, el patrón ya existe y es un **trigger por columna**, no una policy por
tabla: `check_service_billing_override_owner()` (`phase25:546`) es el precedente
exacto, con su comentario explicando por qué el chequeo va por columna.

### 4.6 Quién administra los roles: `OWNER`, sin excepción

```sql
create policy organization_roles_select_members on public.organization_roles
  for select using (public.is_organization_member(organization_id));

create policy organization_roles_write_owner on public.organization_roles
  for all using (public.is_organization_owner(organization_id));
```

Lectura para todo miembro porque la pantalla de equipo muestra el nombre del rol
de cada uno. Escritura solo `OWNER`, mismo criterio que `organization_members`,
`organizations` y `revoke_member()`.

Dos guardas, en la base y no en el action:

- **Un rol con miembros activos no se puede desactivar** → `ROLE_IN_USE`;
  primero se reasigna. Si no, esos miembros caen al rol por defecto de golpe y
  en silencio.
- **Siempre hay exactamente un rol por defecto** (índice único parcial de §4.1).

**Por qué "administrar roles" no es un permiso configurable**: si lo fuera,
existiría un rol capaz de ampliarse a sí mismo, y entonces los otros cuatro
permisos no significan nada. Es la misma razón por la que el `OWNER` no es
configurable (§3).

## 5. Impacto

| Área | Qué cambia |
|---|---|
| Schema | 1 tabla, 1 enum, 1 columna en `organization_members`, 2 índices únicos, 1 CHECK, 2 policies. Todo aditivo. |
| Policies existentes | `payments_select/insert/update_staff` ganan un `and has_org_permission(...)`. Las demás no se tocan. |
| RPCs de pagos | `organization_payment_summary()`, `customer_payment_detail()`, `customer_billing_horizon()` (lectura) y `set_payment_status()` (escritura) cambian su chequeo interno. **Ninguna cambia de firma.** |
| RPCs operativas | `admin_book_for_customer()`, `admin_create_recurring_booking()`, `cancel_booking()`, `cancel_slot_occurrence()`, `mark_attendance()`, `create_managed_customer()`, `enroll_customer_by_email()`, `issue/revoke_customer_activation()` cambian su chequeo interno. Sin cambio de firma. |
| `organization_team()` | Gana `role_id` y `role_name`. **Cambio de contrato** → avisar a frontend-engineer. |
| `invite_member_by_email()` | Necesita `p_role_id`. Postgres no permite agregar un parámetro con `create or replace`: hay que `drop function` + crear, y el `p_role` de 3 argumentos deja de resolver. **Cambio de contrato duro** → `frontend/app/actions/admin.ts:453`. |
| `requireOrganizationMembership()` | Debería devolver también `permissions` (el único punto por donde pasan todas las pantallas del panel, `organizations.ts:89`). **Cambio de contrato** → frontend-engineer. |
| `@reservaste/domain` | Tipos `OrganizationRole`, `OrgPermission`, `OrganizationPermissions` + schemas Zod + mapper. El frontend los importa, no los duplica. |
| Frontend (UI) | Pantalla de roles (OWNER), selector de rol en equipo, ocultar la pestaña de pagos. **No bloquea la migración.** |
| Seguridad | Revisión de `security-engineer` obligatoria: es autorización y roles. |
| Tests | Uno por permiso × (policy directa por PostgREST, RPC `security definer`), un OWNER que nunca es filtrado, un miembro sin `role_id` que se comporta como el rol por defecto, un rol de otra organización que no aplica. |

## 6. Migración / compatibilidad

**Nadie pierde acceso el día del deploy.** El backfill, en la misma migración:

```sql
insert into public.organization_roles (organization_id, name, is_default, created_by)
select o.id, 'Equipo', true, o.created_by from public.organizations o;

update public.organization_members om
   set role_id = r.id
  from public.organization_roles r
 where r.organization_id = om.organization_id and r.is_default;
```

Los cinco booleanos tienen `default true`, así que el rol "Equipo" puede todo lo
que un `STAFF` puede hoy: comportamiento **idéntico**, bit a bit. Restringir es
una acción deliberada del `OWNER`, posterior al deploy. Mismo patrón aditivo que
`makeup_credits_enabled` default `false` (ADR-0025) y `billing_period_months`
nulo (ADR-0031), en la dirección que acá corresponde.

Orden dentro de la migración, y **este orden importa**: primero la tabla y el
backfill, después el helper, y recién al final el endurecimiento de policies y
RPCs. Al revés hay una ventana en la que `has_org_permission()` devuelve `false`
para todos porque todavía no existe el rol por defecto.

## 7. Riesgos

1. **Las RPCs `security definer` que bypassean RLS.** §4.5. Si se olvida una de
   pagos, la pestaña queda escondida y el dato sigue saliendo por
   `POST /rest/v1/rpc/organization_payment_summary`. Es el modo de fallar más
   probable de esta feature. Mitigación: la lista de §5 es exhaustiva y cada
   entrada tiene su test; no alcanza con un test "del permiso".
2. **El agujero preexistente de precios de plan** (§1). Si se despliega esto sin
   cerrarlo, queda un producto donde el `OWNER` cree que configuró quién toca la
   plata y un `STAFF` cambia precios por PostgREST igual. No lo causa esta
   propuesta, pero sí lo vuelve visible y contradictorio. Recomendación: cerrarlo
   en la misma migración (§8, pregunta 3).
3. **Fuga residual: la señal de deuda se ve sin `VIEW_PAYMENTS`.**
   `schedule_rule_standing_reservations()` devuelve `upcoming_unpaid`
   (`phase21:926`) y `evaluate_customer_booking()` devuelve `PAYMENT_REQUIRED`;
   ambos aparecen en pantallas operativas. Un rol sin pagos sigue viendo
   *"a esta persona no la puedo anotar porque no pagó"*. Es información de pago,
   aunque no sea un monto. Recomendación: **aceptarla y documentarla** — sin ella
   el rol no puede hacer su trabajo (no entendería por qué no puede anotar a
   alguien) — pero es una decisión del Orchestrator (§8, pregunta 2).
4. **Un `OWNER` que se configura un rol restrictivo a sí mismo no se bloquea**
   (el `OWNER` corta antes), lo cual es correcto, pero puede confundir: "le puse
   el rol Profesor al dueño y sigue viendo todo". Mitigación: la pantalla de
   equipo no ofrece rol para un `OWNER` y lo dice.
5. **Crecimiento de la matriz.** El pedido real es un permiso ("pagos"); se
   proponen cinco. Cada uno agregado después es barato en schema y caro en
   testing combinatorio. Mitigación: el enum cerrado y este documento como la
   lista de lo que quedó afuera **a propósito**.

## 8. Preguntas abiertas para el Orchestrator

1. **¿`VIEW_PAYMENTS` y `MANAGE_PAYMENTS` separados, o un solo permiso
   "pagos"?** Recomiendo separados con el CHECK de §4.1: "el profesor ve quién
   debe pero no cobra" es un rol de recepción realista, y unirlos después es
   imposible sin romper datos, mientras que separarlos después es una migración
   de una línea.
2. **La fuga de §7.3 (`upcoming_unpaid` / `PAYMENT_REQUIRED`): ¿se acepta o se
   oculta?** Recomiendo aceptarla, documentada en `docs/security.md`.
3. **¿Se cierra `create_service_plan()` y `service_plans` UPDATE a OWNER **en la
   base** en esta misma migración?** Recomiendo **sí**: son ~10 líneas, el gate
   ya existe en TypeScript (o sea, la decisión de producto ya está tomada) y
   hoy es un agujero real.
4. **¿`MANAGE_ATTENDANCE` es configurable o siempre permitido?** Recomiendo
   configurable: el caso inverso al pedido ("recepción cobra pero no pasa
   lista") es igual de real y no cuesta nada tenerlo.
5. **¿Tope de roles por organización?** Recomiendo **no** por ahora. `plans` ya
   tiene `max_team_members` y si hiciera falta, un `max_roles` entra por el
   mismo camino sin tocar nada de esto.

## 9. Relación con ADR-0034

**Son independientes en mecanismo, y ADR-0034 no necesita que esta se apruebe.**
Si ADR-0033 no se aprueba, la invitación de ADR-0034 lleva el enum `STAFF` y no
hay nada que elegir. Si ambas se aprueban, **conviene implementar ADR-0033
primero**, para que `team_invitations` nazca con `role_id` en vez de necesitar
una segunda migración sobre una tabla que ya tiene tokens vivos. Detalle en
ADR-0034 §7.

## 10. Alternativas descartadas

- **Agregar valores al enum `organization_member_role`** (`INSTRUCTOR`,
  `RECEPTION`, `PROFESOR`). Rechazada: el nombre lo elegimos nosotros y no el
  negocio, no es configurable por organización (que es literalmente el pedido),
  y cada rubro nuevo agregaría un valor — la señal exacta de diseño atado a un
  vertical que CLAUDE.md prohíbe.
- **Permisos en `organization_members`, por persona, sin tabla de roles.**
  Rechazada: el pedido dice "roles y nombre del rol", y N personas con N
  combinaciones hace imposible contestar "¿qué puede hacer un Profesor?".
  Una capa de override por persona puede sumarse después encima de esto; al
  revés no.
- **`permissions jsonb`.** §4.2.
- **Reemplazar `OWNER` por un rol "puede todo".** §3.
- **Aplicar los permisos solo en los server actions** (el patrón que hoy usa
  `service-plans.ts`). Rechazada: PostgREST es una puerta abierta a toda tabla
  de negocio (invariante 6 de `invariants.md`), y ADR-0028 ya demostró en vivo
  que lo que no está en la base no está.
- **Roles de Postgres / RLS por rol de base.** Rechazada: todas las requests
  llegan como `authenticated`; el rol de base no distingue personas.
