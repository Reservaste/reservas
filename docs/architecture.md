# Arquitectura

Mantenido por: Orchestrator, consolidando propuestas de
`domain-architect`, `database-agent`, `auth-security-agent`,
`booking-engine-agent` y `scheduling-agent`.

Estado: **Phase 0 — COMPLETE** (2026-09-14). Dos rondas de revisión
coordinada: la primera (Domain Architect, Database Agent, Scheduling
Agent, Auth/Security Agent) cerró el modelo base y ADR-0003 a ADR-0007;
la segunda (los mismos cuatro + Booking Engine, Payments/Entitlements y
Backend/API Agent) cerró stack técnico, recurrencia, timezones, pagos por
período y el flujo de login durante reserva — ADR-0002, ADR-0008 a
ADR-0015. No quedan decisiones estructurales P0 abiertas que puedan
cambiar el schema. Ver el resumen de cierre en `decisions.md` y el punteo
DECISIONES CERRADAS/ABIERTAS/RIESGOS de la conversación de cierre de
Phase 0. Phase 1 puede arrancar.

## Visión de producto

SaaS multi-tenant de agenda, reservas, cupos dinámicos, servicios
habilitados, pagos y reservas recurrentes. Genérico por diseño: el primer
cliente es un gimnasio, pero ningún concepto central puede depender de ese
rubro (ver regla de dominio en `CLAUDE.md` y `domain.md`).

## Multi-tenancy

Aislamiento por `organizationId` en toda la capa de datos y de
autorización, reforzado con RLS (ver `security.md`, `database.md`). Sin
excepciones: todo dato de negocio cuelga de una `Organization`.

## Capas

```
UI (admin / customer / public)
  ↓
Backend / API (server actions o rutas — capa fina de transporte)
  ↓
Dominio (booking engine, scheduling, entitlements/payments — lógica de negocio)
  ↓
Base de datos (Postgres/Supabase — constraints, RLS, concurrencia)
```

La lógica de negocio vive en la capa de dominio, no en los handlers de
API ni en los componentes de UI. Esto es lo que evita que
`booking-engine-agent`, `payments-entitlements-agent` y `backend-api-agent`
dupliquen reglas entre sí.

## Estrategia de slots (SlotOccurrence)

**Resuelta — ADR-0003.** `SlotOccurrence` se persiste como filas
materializadas con horizonte rodante (60-90 días) a partir de
`ScheduleRule` + `ScheduleException`. Ocurrencias con `Booking` asociada
quedan congeladas. Detalle en `domain.md` y `database.md`.

## Estrategia de concurrencia

**Resuelta — ADR-0004.** RPC atómica en PostgreSQL (`SELECT ... FOR
UPDATE` + verificación de capacidad + insert, en una sola transacción
server-side), respaldada por `UNIQUE(customerId, slotOccurrenceId)`.
Justificación completa (por qué no row locking suelto vía supabase-js, ni
constraint+retry, ni advisory locks) en `database.md`.

## Seguridad y multi-tenancy

**Resueltas — ADR-0005, ADR-0006, ADR-0007.** Contrato de
`canCustomerBook()` sin `customerId` de confianza (previene IDOR), RLS de
dos capas (tenant + ownership de fila) para tablas de customer, y límite
de disclosure en disponibilidad pública de capacidad baja. Detalle en
`security.md`.

## Estructura de carpetas y repos (ADR-0016)

Dos repos GitHub separados bajo el org `Reservaste`, más este workspace
local de coordinación (no publicado como repo de producto):

```
reservas/                    <- workspace de coordinación (este directorio)
  docs/                      <- documentación compartida (este conjunto de archivos)
  .claude/                   <- sistema de agentes
  CLAUDE.md
  backend/                   <- git remoto: Reservaste/backend
    supabase/
      migrations/
      config.toml
    src/                     <- paquete @reservaste/domain
      types.ts
      schemas.ts
      invariants.ts
      mappers.ts
      index.ts
    package.json
  frontend/                  <- git remoto: Reservaste/frontend
    app/                     <- Next.js App Router
    components/
    lib/
    package.json             <- depende de @reservaste/domain vía git
```

`backend/` **no es un servicio HTTP separado** — sigue siendo Supabase +
el paquete de dominio compartido; Next.js (`frontend/`) sigue siendo la
capa de API vía server actions/route handlers, consistente con ADR-0002.
Estructura de rutas dentro de `frontend/app/` (`(public)`, `(customer)`,
`(admin)` alineado a los tres niveles de `api.md`) se define al escribir
las primeras páginas de Phase 1.

## Stack técnico

**Resuelto — ADR-0002.** Next.js (App Router, TypeScript strict) +
PostgreSQL vía Supabase + Supabase Auth + supabase-js + RPC de PostgreSQL
+ Zod + TailwindCSS/shadcn/ui + Vercel. Sin ORM adicional (confirmado por
`database-agent`). Detalle en `decisions.md`.

## Recurrencia, timezones y pagos (segunda ronda de Phase 0)

**Resueltas — ADR-0009 a ADR-0015.** Modelo completo de
`RecurringBooking`/estados de `SlotOccurrence`/taxonomía de cancelación
(ADR-0010), generación idempotente de bookings recurrentes vía RPC
hermana de `book_slot` (ADR-0011), política de cupo+recurrencia con
preview y confirmación explícita (ADR-0012), `requiresActivePayment` en
`ServiceEntitlement` + modelo de `Payment` por período (ADR-0013),
estrategia de timezones con conversión en SQL vía `AT TIME ZONE`
(ADR-0014), y preservación de booking intent a través del login
(ADR-0015). Detalle completo de cada una en `decisions.md`.

## Navegación (UI)

Tres primitivas compartidas, para que "volver" signifique lo mismo en
toda la plataforma. Si hace falta una cuarta forma de volver atrás,
primero revisar si alguna de estas alcanza.

- **`BackLink`** (`components/back-link.tsx`) — siempre **nombra su
  destino** ("Clientes", no "Volver"). Un link de retorno que dice a
  dónde va es la diferencia entre confiar en él y tener que probarlo.
- **`Breadcrumbs`** (`components/breadcrumbs.tsx`) — para pantallas
  anidadas. En móvil colapsa a un solo `BackLink` al padre inmediato
  (una ruta completa ocupa una línea y se trunca igual); desde `sm`
  muestra el camino entero, que es lo que dice **dónde estás**, no sólo
  cómo salir.
- **`TabNav`** (`components/tab-nav.tsx`) — la barra de secciones del
  panel y del portal. Scrollea horizontalmente con la scrollbar oculta,
  así que lleva dos cosas sin las cuales en un teléfono simplemente
  desaparecen secciones: un degradado en el borde que todavía tiene
  contenido detrás, y `scrollIntoView` de la sección activa, para que
  siempre se vea en cuál estás.

Reglas de rutas de salida: ninguna pantalla es un callejón sin salida.
El logo del panel de una organización vuelve a `/dashboard` (la lista de
organizaciones), el del portal también, y la página pública de un negocio
lleva a `/me` si hay sesión o a `/login` si no.

## Sistema visual (Fase L0, Iteración 3)

`components/ui/` es el único lugar donde se define cómo se ve un control.
Antes de escribir un estilo a mano, revisar si ya hay primitiva: el pase de
L0 encontró 9 copias divergentes de la misma constante de `<select>`, 18
mensajes de error inline sin `role`, y 8 estados vacíos con tres paddings
distintos. Esa es exactamente la deriva que esto viene a cortar.

**Tres reglas duras**, que valen para cualquier pantalla nueva:

1. **Un solo anillo de foco.** Button/Input/Select lo traen adentro; todo lo
   hecho a mano que reciba foco usa `.focus-ring` (definida en
   `app/globals.css`). Usa `outline` y no `box-shadow`, así no borra la
   sombra propia del elemento, y es `:focus-visible`, así que tocar una fila
   en el teléfono no deja el anillo pegado.
2. **44px de alto mínimo en móvil** para cualquier cosa que se toque
   (`size="touch"` en Button, `touch` en Input/Select), con densidad de
   escritorio desde `sm`. La pantalla de pasar lista mantiene sus 56px
   (ADR-0023).
3. **Los errores de formulario son `FormError`, no un `<p>` rojo.** Lleva
   `role="alert"`: sin eso el error existe visualmente y no existe para un
   lector de pantalla.

**Primitivas y cuándo usar cada una** (cada archivo tiene el detalle y, más
útil, cuándo *no* usarlo):

- `dialog` — confirmaciones y formularios cortos de admin. `sheet` — el
  default en pantallas de cliente y público (Drawer: swipe, safe-area,
  teclado virtual).
- `table` vs. **`DataList`** — casi ninguna lista de este producto es una
  tabla. `DataList` es el patrón de tarjetas en teléfono / fila dividida
  desde `sm`; `Table` es para datos comparables entre filas en pantallas
  sólo-admin.
- `alert` (el mensaje de la pantalla, no descartable) vs. `toast` (lo
  efímero, ya montado en `app/layout.tsx`). `badge` es uno solo:
  `StatusBadge` delega ahí.
- `form` (`Field`, `FieldHint`, `FormError`, `FormSuccess`), `select`
  (nativo a propósito: vive dentro de `<form action={serverAction}>` y en un
  teléfono el picker del sistema le gana a cualquier listbox), `skeleton`,
  `empty-state`.

**Ninguna primitiva pide callbacks.** El control de confirmación se pasa
como elemento (típicamente un `<form action={serverAction}>`), que sí es
serializable, así que un Server Component puede renderizar un `ConfirmDialog`
entero — la misma regla de límite que ADR-0023 fijó para el calendario.
Trampa documentada en `dialog.tsx`: un diálogo no controlado **no se cierra
solo** cuando su acción tiene éxito, y envolver el submit en `DialogClose` es
una carrera contra el desmontaje del form.

Base: `@base-ui/react`, que ya estaba en el proyecto. Dialog, sheet y toast
salieron **sin agregar ninguna dependencia**.

## Sistema visual — Fase L (pulido, Iteración 3)

Cierra el backlog que dejó L0. Cada punto es una decisión tomada, no solo un
arreglo puntual — la próxima pantalla tiene que poder resolver la misma duda
leyendo esto en vez de adivinar de nuevo.

1. **Una sola altura para controles no-`touch`: `h-8` (32px).** `Button`,
   `Input` y `Select` ya coincidían ahí. La tercera altura que L0 había
   marcado (`Button size="lg"`, `h-9`, sin ningún call site) se **eliminó**
   en vez de "arreglarse": nadie la usaba, y una talla que nadie pide es
   exactamente la deriva que este documento existe para cortar. Si una
   pantalla de escritorio necesita un botón más grande que el default, ya
   existe `size="touch"`, que en desktop (`sm:`) cae en `h-9` — no hace
   falta una tercera talla fija para eso.
2. **Los dos `window.confirm()` de `service-row.tsx` / `resource-row.tsx`
   pasaron a `ConfirmDialog`.** El truco no era el reemplazo directo: el
   `confirm()` bloqueaba un botón con `formAction` dentro del `<form>` de
   edición de la fila. `ConfirmDialog` portalea su contenido (incluido el
   `<form action={archiveService}>` que confirma), así que ese segundo
   `<form>` nunca queda anidado dentro del `<form>` de la fila en el DOM
   real aunque lo esté en el JSX — es justo el patrón que el docstring de
   `dialog.tsx` ya documentaba.
3. **`lucide-react` reemplaza los íconos dibujados a mano.** `icons.tsx`
   re-exporta desde ahí (`ChevronLeft`, `ChevronRight`, `CloseIcon`,
   `CheckIcon`, `AlertCircleIcon`, `AlertTriangleIcon`, `InfoIcon`), y
   `dialog.tsx`, `toast.tsx` y `form.tsx` (el `X` de cerrar, el check de
   `FormSuccess`, el círculo de `FormError`) ya lo usan en vez de su propio
   `<svg>`. Todo ícono nuevo entra por `icons.tsx`, nunca importando
   `lucide-react` directo en una pantalla — es lo que permite auditar de un
   vistazo qué set de íconos usa el producto. `brand.tsx` (el logo) queda
   afuera a propósito: es un asset de marca, no un ícono de UI.
4. **Regla de `ChevronRight` como afordancia de fila:** aparece **solo**
   cuando toda la fila es un link que navega a una sub-página propia
   (`Clientes` → ficha del cliente, `Servicios` → horarios del servicio).
   No aparece en una fila que dispara una acción en el lugar (editar,
   archivar) ni en una que no tiene a dónde ir (un `Recurso` no tiene
   sub-página, y por eso `resource-row.tsx` nunca tuvo chevron — no era una
   omisión, era la regla ya aplicada sin que estuviera escrita).
5. **`PageHeader` es para título + descripción + acciones en una fila.**
   Se adoptó donde encajaba (`Mis reservas`, `Mis pagos`, `Mis créditos`,
   `Mis servicios`, `Tus organizaciones`, la ficha de pagos de un cliente).
   No se fuerza en dos formas legítimamente distintas: la tarjeta de auth
   centrada de `login`/`signup`/`onboarding`/`activar/continuar` (título +
   subtítulo, sin fila de acciones, ancho de tarjeta angosto) y un
   encabezado compuesto con avatar (`customers/[customerId]`, que apila
   avatar + nombre + badge). Forzar `PageHeader` ahí habría sido peor que
   dejarlo a mano: la API no modela ninguna de las dos formas.
6. **Densidad `touch` en el panel admin — decisión parcial, explícita.**
   No se pasó todo el panel a `touch`: es un cambio mecánico de decenas de
   archivos que no se puede verificar sin `next dev` (ver punto 8), y
   convertirlo a ciegas es más riesgo del que este pase puede pagar. Lo que
   sí se decidió y se aplicó: la **Agenda** (`agenda/[occurrenceId]` y su
   `OccurrenceActions`, la pantalla de mayor prioridad y la que el dueño
   abre desde el teléfono según el feedback que originó esta iteración) y
   los formularios de alta que se convirtieron a `Sheet` (punto 7) son
   `touch` de punta a punta. El resto del panel (Clientes, Pagos, Servicios,
   Recursos, Equipo como *listas*) se queda en densidad de escritorio por
   ahora. **Esto es deuda, no una decisión de producto**: la extensión
   completa queda para un pase que pueda levantar `next dev` y confirmar
   visualmente cada pantalla, en vez de tocar 40 archivos sin poder verlos.

   Un barrido chico y de bajo riesgo sí se hizo en todas las pantallas
   públicas/cliente que ya estaban en `touch` pero tenían un botón suelto en
   `sm`/`xs` sin razón (el header de `/`, el de la agenda pública de una
   organización, "Salir" en `/me` y `/dashboard`, "usar otra cuenta" en
   `/activar/continuar`, "Liberar cupo" en `/me`): son controles sueltos,
   no listas densas, así que no había nada que verificar visualmente para
   corregirlos. De paso salió `size="lg"` de `app/page.tsx` (los dos
   `buttonVariants({ size: "lg" })` de la landing, que un primer grep por
   `size="lg"` con comillas no encontró porque acá se llama con
   `size: "lg"` vía objeto) — sin ese barrido, borrar `lg` del punto 1
   hubiera roto el typecheck de la landing.
7. **Los cuatro "formularios que se abren inline" se unificaron a `Sheet`.**
   `EnrollForm`, `InviteForm` (equipo) y `ManagedCustomerForm` (un tercero
   que ADR-0026 sumó después de L0, con el mismo gesto exacto al lado de
   `EnrollForm` en la misma pantalla) tenían un botón que revelaba un
   `<form>` en el lugar y un link de texto para cerrarlo, cada uno con su
   propio wording. `OccurrenceActions` resultó **no ser un candidato real**:
   su único caller (`agenda/[occurrenceId]/page.tsx`) siempre pasaba
   `alwaysOpen`, así que la rama colapsada era código muerto — se borró en
   vez de convertirla. `admin/invite-form.tsx` (la consola de plataforma)
   tampoco calificaba: es una sección siempre visible de su propia pantalla,
   no un gesto de revelar/ocultar. El patrón que quedó, para la próxima vez
   que alguien necesite "un botón que abre un formulario corto": `Sheet` +
   `<form id="…">` en el `SheetBody` + `<Button form="…">` en el
   `SheetFooter` — el atributo `form` de HTML asocia el submit sin anidar
   un segundo `<form>` ni depender del portaleo para evitarlo.
8. **`.dark` sigue siendo código muerto — decisión explícita de no
   implementarlo en esta fase**, no un olvido. ADR-0020 ya lo había
   marcado; implementar un tema oscuro real (tokens, contraste con el
   acento por organización de ADR-0020, y una forma de activarlo) es una
   feature de superficie propia, no algo que quepa como ítem de un pase de
   pulido general. Queda para cuando alguien lo pida.
9. **El `Sheet` no se pudo ver renderizado.** Ningún agente de esta fase
   tuvo acceso a un shell con `next dev` — herramienta no disponible, no
   omisión. El trigger, el backdrop, `Escape` y el cierre por botón son los
   mismos primitivos de Base UI que ya probó `Dialog` (que sí se usa en
   producción desde antes de esta fase), así que el riesgo real está
   acotado al swipe-to-dismiss y a la transición CSS de `translate`
   documentada en `sheet.tsx` — **eso sigue sin verificar**. La Fase M
   (cierre) tiene que confirmarlo con sesión real antes de darlo por bueno.
10. **Hallazgo de esta pasada, no del backlog original:** 10 pantallas
    (`customers`, `me/pagos`, `me/creditos`, `org/[slug]` home, `admin`,
    `services/[serviceId]/asistencia`, `payments/[customerId]`,
    `customer-forms.tsx` ×2, `team`) reconstruían a mano, carácter por
    carácter, la misma clase de `DataList` (`divide-y overflow-hidden
    rounded-xl border bg-card shadow-card`) en vez de usar la primitiva que
    L0 ya había escrito — con la diferencia real de que la versión a mano
    **no** colapsa a tarjetas separadas en el teléfono, que es la mitad del
    punto de `DataList`. Las diez se migraron a `DataList`/`DataListRow`.
    No se tocó `services/[serviceId]/plans/**` (ADR-0029, en curso en
    paralelo) ni los filtros por servicio de `payments/page.tsx`.
11. **Los motivos de rechazo de reserva (`lib/booking-reasons.ts`) ahora
    distinguen tono.** `bookingReasonTone()` clasifica cada
    `can_book_reason` en `neutral` (el cupo cambió mientras mirabas — nadie
    tiene la culpa: `SLOT_FULL`, `OCCURRENCE_NOT_AVAILABLE`,
    `ALREADY_BOOKED`, `DUPLICATE`), `customer` (hay algo que la persona
    puede hacer ahora — pagar, iniciar sesión, sumarse a un plan) y `owner`
    (el negocio no configuró algo — `SERVICE_HAS_NO_PLAN`,
    `ORGANIZATION_INACTIVE`, `SERVICE_INACTIVE` — y ni pagar ni reintentar
    lo resuelve). Antes las tres se mostraban igual, en un `<p>` amarillo a
    mano en `reservar/confirmar/page.tsx`; ahora es `Alert` con
    `info`/`warning`/`danger` e ícono acorde. `DESK_BOOKING_REASONS` (la
    vista del mostrador en `standing-reservations.tsx`) se dejó con su
    binario muted/destructive: es una previsualización compacta de muchas
    fechas a la vez y la categorización de tres vías ahí sería ruido, no
    señal.

## Pase de corrección post-Fase L (2026-09-22)

Una auditoría de UX (`ux-ui-designer`, revisión estática, sin `next dev`) sobre
las pantallas ya desplegadas encontró bugs reales, no solo pulido: dos server
actions (`setSubscription`, `markAttendance`) ignoraban el `error` del
`.rpc()` y fallaban en silencio, `SubscriptionControls` suspendía/reactivaba
una organización sin confirmación ni feedback de error, y el roll-call de
asistencia no revertía ni avisaba si `markAttendance` fallaba. Corregido con
el mismo patrón `{ error: string | null }` que ya usa `services.ts`.

Un detalle vale la pena dejar escrito porque puede repetirse: al arreglar el
revert de `RollCall`, el primer intento cambió `useOptimistic` por
`useState(initialAttendees)` para poder revertir a mano con un `previous`
capturado. `security-engineer` y `qa-engineer`, en paralelo y sin verse,
encontraron el mismo problema: sin `key` en el padre, `useState` nunca vuelve
a leer la prop después del mount, así que la pantalla queda congelada frente
a cualquier `revalidatePath` ajeno (otra cancelación, otro dispositivo
marcando el mismo turno). La solución correcta no fue agregar un `key` ni un
efecto de resync manual: fue **no abandonar `useOptimistic`** — se re-basa en
la prop en cada render fuera de una transición, y al fallar cae solo al valor
real sin revert manual ni riesgo de que una respuesta fuera de orden pise una
marca posterior. El error se guarda aparte, en un `useState` propio que no
participa del reducer optimista. Regla general: si una pantalla ya usa
`useOptimistic` y hace falta agregar manejo de error, la respuesta casi nunca
es reemplazarlo por `useState` — es sumarle un estado de error al lado.

También se corrigió el copy de confirmación de suspender/reactivar una
organización: decía que cortaba el panel y las reservas existentes, y
`subscription_status` en realidad solo bloquea **crear** entidades nuevas
(triggers `BEFORE INSERT`) — el panel y las reservas ya confirmadas siguen
funcionando. Qué debería significar "suspender" de verdad queda como pregunta
de producto abierta, no resuelta acá.

Se agregaron `app/error.tsx`/`app/not-found.tsx` (no existían: cualquier
fallo no controlado caía en la página default de Next, sin marca) y
`loading.tsx` en las 9 rutas de mayor tráfico que no tenían ningún estado de
carga — de ~60 pantallas, antes de este pase solo `plans/page.tsx` manejaba
los 4 estados (loading/error/vacío/éxito) completos. Se terminó la migración
a `DataList` que el punto 10 de arriba dejó incompleta (`me/page.tsx`,
`me/servicios/page.tsx`, lista de organizaciones en `admin/page.tsx`), y se
sumó densidad `touch` en `activation-panel.tsx` y la consola de plataforma —
instancias nuevas de esa deuda, construidas después del barrido original.

Sigue pendiente, sin cambios por este pase: verificación visual con
`next dev`/sesión real (Fase M), y el resto de la densidad `touch` del panel
admin (Clientes, Pagos, Servicios, Recursos, Equipo como listas).

## Próximos pasos

Arrancar Phase 1 (`roadmap.md`): Auth + Organizations + Roles, con el
modelo de RLS de dos capas (ADR-0006) y el schema completo (entidades,
constraints, RPCs de ADR-0004/ADR-0011) como base desde la primera
migración. Definir estructura de carpetas de Next.js como primer paso de
implementación (ver sección de arriba).
