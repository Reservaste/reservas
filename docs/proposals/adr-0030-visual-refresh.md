# ADR-0030 — Refresh visual: marca de plataforma (violeta/cian), landing tipo funnel y pase de pulido

Fecha: 2026-09-22
Estado: **Propuesta** (no aplicada — requiere aprobación del Orchestrator)
Propuesta por: `ui-ux-agent`
Origen: pedido del usuario (dueño del producto) — "mejorar el diseño y la UX
de todo el producto" tomando `https://turnito.app/uy/` como referencia de
nivel: landing tipo funnel, paleta vibrante violeta/cian, tipografía grande
de alto contraste, tono conversacional, mucho whitespace.
Depende de: ADR-0020 (identidad por organización — **es el ADR que esta
propuesta tiene que no romper**), Fase L0 / Fase L del sistema visual
(`docs/architecture.md`), ADR-0017 (creación de organización por invitación),
ADR-0021 (hosting y dominio real), ADR-0023 (el calendario público es la
página del negocio).
No toca: modelo de dominio, schema, RLS, contratos de API. Es una propuesta
de diseño; la implementación la ejecuta `frontend-engineer`.

---

## 0. La pregunta central, contestada primero

> Si Reservaste adopta violeta/cian como color de marca, ¿dónde vive cada
> color respecto del acento configurable de cada organización (ADR-0020)?

**La respuesta no es "uno u otro": el producto necesita las dos cosas, y el
problema real de hoy no es que falte el violeta — es que `--primary` está
cumpliendo dos papeles incompatibles a la vez.**

Hoy `--primary` significa simultáneamente "el acento de esta pantalla" y "el
color de Reservaste". Por eso `BrandMark` (el logo de Reservaste,
`components/brand.tsx`) se pinta con `bg-gradient-to-br from-primary to-primary-hover`,
y como el header del panel (`app/org/[slug]/layout.tsx:19-32`) y la página
pública del negocio (`app/[organizationSlug]/page.tsx:73`) están dentro de
`<BrandTheme>` → `[data-brand]`, **el logo de Reservaste se pinta con el color
del cliente**. Si el gimnasio elige naranja, nuestro logo es naranja. Eso no
es branding por organización, es una fuga de token: ADR-0020 dice
explícitamente "la consola de plataforma queda con la marca del producto — es
nuestra, no de ellos", y esa intención hoy no está implementada en el
componente de marca.

### Decisión propuesta: tres capas de token, con una regla que las separa

| Capa | Tokens | Quién la controla | Dónde aparece |
|---|---|---|---|
| **P — Marca Reservaste** | `--brand-violet`, `--brand-violet-strong`, `--brand-violet-deep`, `--brand-cyan`, `--brand-cyan-deep` | El producto. **Fija, nunca sobreescribible por un tenant.** | `Brand`/`BrandMark`, landing pública, consola `/admin`, crédito "con Reservaste" del footer |
| **S — Semántica** | `--success`, `--warning`, `--destructive` (+ `-subtle`, `-foreground`), `--border`, `--muted`, etc. | El producto. **No cambia en esta ADR.** | Todo el producto, idéntico en todos los tenants |
| **T — Acento de quien es dueño de la pantalla** | `--primary`, `--primary-hover`, `--primary-subtle`, `--ring`, `.brand-wash` | Default = violeta de capa P. Dentro de `[data-brand]`, el `accent_color` de la organización. | Página pública del negocio, panel de la organización, y —con el default— auth, `/dashboard`, `/me`, `/admin`, landing |

**La regla, en una línea, que es lo que hay que poder recitar de memoria
dentro de seis meses:**

> `--primary` es *"el acento de quien es dueño de esta pantalla"*.
> `--brand-*` es *"Reservaste"*.
> Un componente nunca dibuja la identidad de Reservaste con `--primary`, ni
> la identidad de un tenant con `--brand-*`.

Consecuencias concretas, todas verificables con un grep:

1. **El violeta pasa a ser el valor por defecto de `--primary`** (reemplaza
   al índigo actual `oklch(0.5 0.19 260)`). Toda superficie que hoy no está
   envuelta en `[data-brand]` —login, signup, onboarding, `/activar`,
   `/dashboard`, `/me` y sus sub-pantallas, `/admin`, la landing— se vuelve
   violeta sin tocar una sola línea de esas pantallas. Es lo que hace que
   "todo el producto" se sienta renovado con un cambio de cuatro variables.
2. **Una organización que ya configuró su acento no ve ningún cambio.**
   `[data-brand]` sigue pisando `--primary` exactamente igual que hoy. Esto
   es deliberado: son clientes que pagan y ya eligieron su color; un refresh
   de nuestra marca no puede repintarles la página que le muestran a sus
   propios clientes.
3. **Una organización que *no* configuró acento pasa de índigo a violeta.**
   Es el mismo hecho que el punto 1 (hereda el default) y es correcto: sin
   color propio, la superficie la viste el producto.
4. **`BrandMark`/`Brand` dejan de usar `--primary` y pasan a los tokens
   fijos de capa P.** Esto arregla la fuga descrita arriba. Es el único
   cambio de esta ADR que un tenant con acento configurado va a ver: su
   header deja de tener el logo de Reservaste en su color y pasa a tenerlo
   en violeta/cian, al lado de su propio logo (`OrganizationLogo`). Es
   exactamente la distinción que ADR-0020 quería y que la implementación
   perdió.
5. **El cian nunca entra a `[data-brand]`.** Es decorativo y exclusivo de
   superficies de plataforma. Un cian fijo conviviendo con el acento
   arbitrario de un tenant es una colisión de color que no podemos
   garantizar, y no aporta nada que el acento del negocio no aporte mejor.

**Alternativa descartada:** que el violeta/cian sea sólo de la landing y el
resto del producto quede índigo. Descartada porque produce exactamente el
problema que el usuario pidió resolver — "aplicado a todo el producto, no
solo la landing" — y porque deja la marca de la plataforma diciendo dos
colores distintos según por qué puerta entraste.

---

## 1. Paleta

### 1.1 Capa P — marca Reservaste (tokens nuevos, fijos)

Se agregan a `:root` en `frontend/app/globals.css`. **No** se agregan a
`[data-brand]` (a propósito: son inmunes al tenant).

```css
/* Marca Reservaste (ADR-0030, capa P). Fija: ningún [data-brand] la pisa.
   Se escriben en hex y no en oklch porque son valores de marca literales,
   citables en un manual y comparables contra un asset exportado; el resto
   del sistema sigue en oklch, que es lo correcto para tokens derivados. */
--brand-violet:        #7c3aed;  /* acción de marca; blanco encima = 5.70:1 AA */
--brand-violet-strong: #6d28d9;  /* hover/pressed y texto violeta sobre blanco = 7.11:1 AAA */
--brand-violet-deep:   #4c1d95;  /* fondo de banda oscura; blanco encima = 10.96:1 */
--brand-cyan:          #22d3ee;  /* SOLO decorativo: 1.7:1 sobre blanco, nunca lleva texto */
--brand-cyan-deep:     #0e7490;  /* la versión del cian que sí puede ser texto: 5.36:1 AA */
--brand-ink:           #155e75;  /* stop frío de la banda oscura; blanco encima = 7.28:1 */

--brand-gradient: linear-gradient(135deg, var(--brand-violet), var(--brand-cyan));
```

**Regla dura del gradiente:** el degradado violeta→cian **nunca lleva texto
encima**. Su punto medio queda alrededor de 3.3:1 contra blanco, que no pasa
AA ni para texto grande con margen. El gradiente es para: el glifo de
`BrandMark` (decorativo, `aria-hidden`), filetes/separadores de 2-4px, y
washes de muy baja opacidad. Para "palabra destacada" en un titular se usa
`--brand-violet-strong` sólido (7.11:1), nunca `bg-clip-text` sobre el
gradiente.

### 1.2 Capa T — el default de `--primary` se mueve a violeta

En `:root`, reemplazar:

```css
--primary:        oklch(0.541 0.246 292.7);  /* #7c3aed */
--primary-hover:  oklch(0.491 0.238 292.6);  /* #6d28d9 */
--primary-subtle: oklch(0.945 0.048 292);    /* wash del mismo tono */
--ring:           oklch(0.541 0.246 292.7);
--chart-1:        oklch(0.541 0.246 292.7);
--sidebar-primary: oklch(0.541 0.246 292.7);
--sidebar-ring:    oklch(0.541 0.246 292.7);
```

`--primary-foreground` queda en `oklch(0.99 0 0)` (blanco): 5.70:1 sobre el
violeta, AA para texto normal y AAA para texto grande. No hace falta el
fallback de negro puro que ADR-0020 documentó para acentos de tenant en la
banda de luminancia 0.18–0.24 — el violeta está en 0.134, bien fuera de esa
banda.

`[data-brand]` **no se toca**: sigue derivando `--primary-subtle` con
`color-mix` contra `--background`, así que el wash del tenant se recalcula
solo.

### 1.3 Neutros: rotación de matiz, sin tocar luminancia ni croma

Los grises hoy llevan un tinte frío-azul deliberado (hue 253/258, ver el
comentario de `--background`). Con un primario violeta, ese azul lee como un
segundo color desafinado. Corrección mínima y de bajísimo riesgo:

> **Rotar el matiz de todos los neutros de `253 → 285` y `258 → 288`,
> dejando lightness y chroma exactamente como están.**

Afecta: `--background`, `--foreground`, `--card-foreground`,
`--popover-foreground`, `--surface-sunken`, `--secondary(-foreground)`,
`--muted(-foreground)`, `--accent(-foreground)`, `--border`, `--input`,
`--chart-5`, los `--sidebar-*` neutros y el color base de las tres sombras.
Ninguno cambia de contraste (el croma es ≤0.025), sólo de temperatura. Es un
find-and-replace verificable, no un rediseño.

### 1.4 Semánticos: no cambian. Punto.

`--success`, `--warning`, `--destructive` y sus `-subtle`/`-foreground`
quedan **idénticos**. Son funcionales, no de marca: dicen "hay lugar", "ojo",
"esto cancela", y significan lo mismo en todos los tenants (ADR-0020). Si
alguien propone "armonizar el verde con el violeta nuevo", la respuesta es
no.

### 1.5 `.dark` sigue siendo código muerto

Mantengo la decisión explícita de la Fase L (punto 8) y de ADR-0020: no se
revive el tema oscuro en esta ADR. Los tokens de `.dark` se actualizan
**solamente** con la misma rotación de matiz de §1.3 y el mismo violeta, para
que no queden inconsistentes el día que alguien los encienda — pero no se
agrega ningún mecanismo para activarlos, ni se verifica contraste ahí.

**No tengo una razón de peso para revivirlo, y por eso no lo hago.** La única
zona gris es la banda oscura de la landing (§3.8), que **no es dark mode**:
es una sección con colores literales de capa P en su propio subárbol, sin
mecanismo de activación global, sin persistencia y sin controles de
formulario adentro. La distinción está escrita como regla en §1.6.

### 1.6 `.brand-band` — la banda oscura, acotada por escrito

```css
/* Banda oscura de marca (ADR-0030 §1.6). NO es dark mode: no hay toggle,
   no persiste, no cascadea fuera de su propio subárbol y sólo puede
   contener texto, links y botones -- nunca un input, select o dialog, que
   es donde el tema oscuro se vuelve un sistema de tokens propio en vez de
   una sección con un fondo.
   Uso permitido: app/page.tsx (hero opcional y CTA final). Nada más. */
.brand-band {
  background-image: linear-gradient(155deg, #2e1065 0%, var(--brand-violet-deep) 55%, var(--brand-ink) 100%);
  color: #ffffff;
  /* Los botones de adentro se invierten: superficie blanca, texto violeta.
     Esto es un override local de dos tokens, no un tema. */
  --primary: #ffffff;
  --primary-foreground: var(--brand-violet-deep);  /* 10.96:1 */
  --primary-hover: #f5f3ff;
  --ring: var(--brand-cyan);                        /* el foco tiene que verse sobre violeta */
}
.brand-band :where(p, li) { color: rgb(255 255 255 / 0.82); }  /* ≥ 8:1 en todo el gradiente */
```

Contraste verificado contra los tres stops: blanco sobre `#2e1065` = 15.2:1,
sobre `#4c1d95` = 10.96:1, sobre `#155e75` = 7.28:1. El párrafo al 82% de
opacidad se mantiene por encima de 5.9:1 en el stop peor.

---

## 2. Tipografía

**Se queda Inter. No cambia la fuente.** Ya fue una decisión tomada con
criterio (`app/layout.tsx:6-8`: "es la tipografía que lee toda esta
categoría"), y el "punch" de la referencia no viene de la familia sino del
tamaño, el peso y el tracking. Cambiar de familia es riesgo de layout en 60
pantallas a cambio de un delta chico.

**Lo que sí cambia — sólo escala y pesos, y sólo para la landing:**

```css
/* Escala display (ADR-0030 §2). SOLO app/page.tsx y componentes de
   components/marketing/. La escala de producto (text-2xl para títulos de
   pantalla, text-sm de base) no se toca: son densidades distintas para
   trabajos distintos, y mezclarlas es como se arruina un sistema. */
.display-1 { font-size: clamp(2.25rem, 6.2vw, 3.75rem); line-height: 1.04; font-weight: 800; letter-spacing: -0.035em; text-wrap: balance; }
.display-2 { font-size: clamp(1.75rem, 4vw, 2.5rem);    line-height: 1.12; font-weight: 700; letter-spacing: -0.03em;  text-wrap: balance; }
.lead      { font-size: clamp(1.0625rem, 1.6vw, 1.25rem); line-height: 1.6; text-wrap: pretty; }
```

Verificado a 320px: `clamp(2.25rem, 6.2vw, …)` se planta en 36px, que con
`text-wrap: balance` entra sin desbordar un titular de 6-8 palabras.

**Opcional, sin costo y sin dependencia nueva:** agregar
`font-optical-sizing: auto` en `html`. Inter v4 (la que sirve `next/font`)
trae el eje `opsz`, así que los titulares grandes reciben el corte display
—contrapuntos más finos, tracking natural más cerrado— automáticamente.
Lo dejo como pregunta (§6.7) por si preferís cero cambios tipográficos fuera
de la landing.

**No cambia:** la regla de la Fase L de que `h1` es `font-bold` y `h2-h4`
`font-semibold`, ni `.eyebrow`, ni `.tnum`.

---

## 3. Landing pública (`app/page.tsx`)

Hoy son dos secciones (hero + tres features) y un footer de una línea. La
propuesta es un funnel de nueve bloques. Contenedor `max-w-6xl` (hoy
`max-w-5xl`), padding de sección `py-16 sm:py-24`, `px-5 sm:px-8`.

**Escala de spacing de marketing** (nueva, y declarada como tal):
`py-16/sm:py-24` entre secciones, `gap-10` entre el encabezado de sección y
su contenido, `gap-6` entre tarjetas de una grilla, `gap-3` dentro de una
tarjeta. **Estos pasos no entran a las pantallas de producto**, que siguen
con la escala de `globals.css` (1/1.5/2/3/5). Toda pieza nueva vive en
`components/marketing/` — esa carpeta es el límite físico que impide que la
escala de marketing se filtre al panel.

### 3.1 Header (sticky)

`Brand` + anclas `Funciones · Precios · Preguntas` (ocultas bajo `sm`) +
`Ingresar` (ghost) + CTA primario. Fondo `bg-card/80` con
`backdrop-blur` y borde inferior que aparece recién al scrollear (una clase
condicional; sin JS de scroll si se resuelve con `sticky` + un borde
permanente muy tenue — preferir esto último).

Las anclas necesitan `scroll-margin-top: 5rem` en cada `<section id>`, o el
header tapa el título de destino. Y `scroll-behavior: smooth` va envuelto en
`@media (prefers-reduced-motion: no-preference)`.

### 3.2 Hero

- **Eyebrow pill** — no un carrusel. Texto fijo: *"Para consultorios,
  estudios, canchas y cualquier negocio con horarios"*. La referencia rota
  rubros automáticamente; **no lo copiamos**: contenido que se mueve solo y
  no se puede pausar es WCAG 2.2.2, y acá no compra nada que una lista no
  compre.
- **`h1.display-1`**: *"La agenda de tu negocio, funcionando sola"* con
  *"funcionando sola"* en `--brand-violet-strong`. Una sola palabra o frase
  destacada, sólido, nunca gradiente.
- **`p.lead`**: *"Publicás tus horarios, tus clientes reservan solos y vos
  ves todo desde un panel. Sin WhatsApp a las once de la noche ni cuaderno."*
- **Dos CTA**, ambos `size="touch"`: primario (ver §6.2 — el destino real
  está abierto) + `Ver una agenda de ejemplo` (outline) que va a un slug
  público de demo.
- **Línea de confianza** debajo, `text-xs text-muted-foreground`:
  *"Te acompañamos en el alta. Tu página queda publicada el mismo día."*
- **Prueba visual a la derecha (desde `lg`)**: una tarjeta estática que
  imita el calendario público real —tres franjas horarias con su badge de
  disponibilidad, una `FULL`— construida con los mismos tokens, **no una
  captura de pantalla ni un `<img>`**. Es la mejor pieza de la landing porque
  muestra el producto en vez de describirlo, y no se desactualiza como una
  captura. Debe ser `aria-hidden` con un `<p class="sr-only">` describiendo
  qué muestra.

### 3.3 Franja de rubros — genericidad, verificable

Tres tarjetas, **nunca un gimnasio como caso único** (regla de `CLAUDE.md`).
Cada una: ícono, rubro, una línea de escenario concreto:

| Rubro | Escenario |
|---|---|
| **Consultorio** | "Turnos de 30 minutos, uno por vez, con la ficha del paciente a mano." |
| **Estudio de pilates** | "Clases de 8 lugares, el que viene todos los martes se anota una vez." |
| **Cancha de fútbol** | "Dos canchas, turnos de una hora, y el que reserva ya pagó." |

Un cuarto rubro (peluquería, estética) puede entrar si la grilla lo pide,
pero los tres de arriba son los que cubren los tres modos reales del producto
(capacidad 1 / capacidad N / recurso múltiple).

### 3.4 Funciones — seis tarjetas, 3×2 desde `md`

Cada una: ícono en cuadrado `rounded-lg bg-primary-subtle text-primary`
(patrón que ya existe hoy en `page.tsx:81`, pero con un ícono real de
`icons.tsx`, no el `BrandMark` con el gradiente cancelado — eso fue un
apaño y se nota), `h3` y dos líneas.

1. **Cupos que son de verdad** — cada horario sabe cuánta gente entra; si
   alguien libera, el lugar vuelve al instante.
2. **Turnos fijos** — el que viene todas las semanas se anota una vez;
   cancelar una fecha no rompe la serie.
3. **Planes y cuotas** — "dos veces por semana" es una regla del plan, no
   algo que tengas que contar a mano.
4. **Pagos al día** — quién está al día y hasta cuándo, con becas y
   cortesías incluidas.
5. **Crédito de recupero** — el que avisa a tiempo se recupera la fecha
   dentro del mes, sin que lo arregles vos.
6. **Tu marca, no la nuestra** — tu logo y tu color en la página que ven tus
   clientes. *(Esta es la tarjeta que le da valor comercial a ADR-0020 y hoy
   no está contada en ningún lado.)*

### 3.5 Cómo funciona — tres pasos

`1. Cargás lo que ofrecés y en qué horarios. → 2. Compartís tu link.
→ 3. Mirás la agenda llenarse.` Numerados con un círculo
`bg-primary text-primary-foreground`, conectados por una línea de 1px en
`md+` (decorativa, `aria-hidden`).

### 3.6 Señales de confianza — sin inventar nada

**No hay testimonios reales para mostrar** (un cliente en producción). La
referencia usa reseñas; nosotros **no vamos a fabricarlas**, y quiero que eso
quede escrito acá y no se resuelva de memoria en el momento de implementar.
En su lugar, tres señales factuales verificables:

- *"Tus clientes no instalan nada. Entran a un link y reservan."*
- *"Backup diario de tu información."* (ADR-0021 — es cierto y es
  diferencial frente a una planilla.)
- *"Hablás directo con quien lo construye, no con un ticket."*

Si el primer cliente autoriza a ser nombrado, un único testimonial real
reemplaza a la tercera (§6.4).

### 3.7 Precios — datos reales, leídos de la base

Los planes **ya existen y son públicos**: tabla `plans`, policy
`plans_select_all using (true)`, columna `is_public`, orden por `sort_order`.
La landing los consulta en el server component; **no se hardcodean**.

| `code` | `name` | USD/mes | Servicios | Recursos | Clientes | Equipo |
|---|---|---|---|---|---|---|
| `starter` | Starter | 20 | 3 | 2 | 50 | 2 |
| `pro` | Pro | 40 | 10 | 5 | 200 | 5 |
| `full` | Full | 80 | sin límite | sin límite | sin límite | sin límite |

`null` en un límite se renderiza **"Sin límite"**, nunca "null" ni un guión.

Tres `PriceCard`. **Pro destacada** (borde `--primary`, cinta "El más
elegido", `shadow-raised`, `sm:-translate-y-2`): es la del medio y la que
tiene la mejor relación límites/precio. La tarjeta destacada necesita
`aria-label` que incluya la distinción, no sólo color.

**Estados de esta sección (única parte dinámica de la landing):**
- *loading* — `Suspense` con tres `Skeleton` de la altura exacta de la
  tarjeta, para que no haya salto de layout.
- *error* (la query falla) — un panel en lugar de la grilla: *"No pudimos
  cargar los planes ahora mismo. Escribinos y te los pasamos."* + CTA de
  contacto. **Nunca precios hardcodeados como fallback**: un precio viejo
  mostrado como actual es peor que no mostrar precio.
- *empty* (`plans` sin filas `is_public`) — mismo panel que *error*. Una
  lista de precios vacía y una que falló son el mismo hecho para el visitante.
- *success* — las tres tarjetas.

### 3.8 Preguntas frecuentes — `<details>` nativo, sin dependencia

No existe primitiva de accordion en `components/ui/` y no hace falta
agregarla: `<details>/<summary>` da teclado, lectores de pantalla y estado
abierto/cerrado sin una línea de JS y sin un `aria-expanded` que mantener a
mano. Se estiliza el marcador con `[&::-webkit-details-marker]:hidden` + un
`ChevronRight` que rota con `group-open:rotate-90`.

`<summary>` necesita: `.focus-ring`, `min-height: 44px`, `cursor-pointer` y
`list-style: none`. Seis preguntas sugeridas:

1. ¿Mis clientes tienen que crear cuenta? (sí para reservar, no para ver la
   agenda — ADR-0023/ADR-0026)
2. ¿Puedo poner mi logo y mi color? (sí — ADR-0020)
3. ¿Cobra la plataforma los pagos de mis clientes? (no: registrás quién está
   al día; el cobro es tuyo)
4. ¿Qué pasa si alguien cancela? (el lugar se libera al instante; según cómo
   configures, le queda crédito para recuperar)
5. ¿Sirve si atiendo de a uno? (sí: capacidad 1)
6. ¿Y si tengo más de un local / más de una cancha? (`Resource`)

### 3.9 CTA final + footer

`.brand-band` (§1.6) de ancho completo, `rounded-3xl` dentro del contenedor:
`display-2` blanco, una línea de apoyo al 82%, y un botón blanco con texto
`--brand-violet-deep`. Footer: `Brand`, una línea legal, link de contacto,
y el año. Nada más.

### 3.10 Estados de la landing completa

| Estado | Qué pasa |
|---|---|
| *loading* | Sólo la sección de precios (§3.7). El resto es estático y se renderiza en el server: no lleva skeleton. |
| *error* | Acotado a precios. Un fallo de la landing entera es un 500 y lo cubre `error.tsx`, que **no existe todavía** en la raíz — hay que crearlo, con la marca puesta y un link a `/`. |
| *empty* | Sólo precios. |
| *success* | La página. |
| *sesión iniciada* | Se mantiene el `redirect("/dashboard")` de hoy (`page.tsx:28-30`). |

---

## 4. Resto del producto — dónde rinde el pulido

Priorizado por tráfico real, no por cantidad de pantallas. Cada una hereda el
violeta automáticamente (§1.2) o conserva el acento del tenant (§0.2); lo que
sigue es lo que **además** hay que tocar a mano.

### 4.1 Página pública del negocio + calendario (máxima prioridad)

`app/[organizationSlug]/page.tsx`, `components/calendar/public-calendar.tsx`.
Es la pantalla que más gente ve y la única que ve gente que no es cliente
nuestro. **Mantiene el acento del tenant** (`[data-brand]`), sin excepción.

- `BrandMark` del header deja de teñirse (§0.4). Además, bajar su presencia:
  en la página de un negocio, la marca de Reservaste es un crédito, no un
  co-protagonista. Propuesta: sacar `Brand` del header y dejar sólo el
  crédito "con Reservaste" que ya está en el footer (`page.tsx:118`), con el
  link de retorno resuelto por el `OrganizationLogo`.
- **Jerarquía del header**: hoy el nombre del negocio es `text-lg` en una
  fila secundaria debajo de nuestra marca. Debería ser lo primero y lo más
  grande de la pantalla.
- **Tarjeta de horario**: la relación entre hora, nombre del servicio y badge
  de disponibilidad es lo único que importa. Hora en `.tnum` y peso alto,
  servicio en `text-sm`, badge alineado a la derecha. Un slot `FULL` debe
  leerse distinto por más de un canal (opacidad + badge de texto, no sólo
  color).
- **Estados**: *loading* (skeleton de 3 días con 2 slots cada uno — hoy la
  página es un server component sin `loading.tsx`: **falta**), *error*
  (organización inexistente ya va a `notFound()`; falta el caso "la
  disponibilidad falló pero el negocio existe"), *empty* (ya resuelto con
  `EmptyState`, `page.tsx:103`), *success*.

### 4.2 Confirmación de reserva (`reservar/confirmar`)

Es el momento de éxito del producto entero y merece el único tratamiento
celebratorio de toda la aplicación: check en `--success`, resumen del turno
en una tarjeta con `shadow-raised`, y dos salidas claras ("Ver mis reservas"
/ "Reservar otro horario"). Mantiene `[data-brand]`. Necesita los cuatro
estados explícitos: *loading* del submit (botón con pending, ya existe el
patrón), *error* (cupo tomado mientras confirmabas — copy específico, no
genérico), *empty* (no aplica), *success*.

### 4.3 Portal del cliente (`/me`, `/me/pagos`, `/me/servicios`, `/me/creditos`)

Superficie **sin** `[data-brand]` (un cliente puede serlo de varias
organizaciones): hereda el violeta. Lo que hay que corregir:

- **Vocabulario de rubro filtrado en copy de producto.** Hallazgo concreto:
  `app/me/page.tsx:51` y `:135,:142`, y `app/me/creditos/page.tsx:13,:40,:47`
  dicen **"la clase"** literal. En un consultorio eso está mal. Esos textos
  deben decir "la fecha" / "el turno" de forma neutra, o —mejor— salir de un
  término configurable por `Organization` (§6.6, no lo decido yo porque
  implica una columna nueva). Mismo problema en
  `org/[slug]/services/[serviceId]/asistencia/page.tsx:44,49,50` ("Clases que
  ya ocurrieron", "Todavía no hay clases dictadas").
- **Cada fila de reserva** debería decir de qué organización es sin que haya
  que deducirlo: hoy el portal es multi-organización y la marca del negocio
  no aparece por fila. Un `OrganizationLogo size="sm"` en la fila lo resuelve.
- Estados: los cuatro ya están, salvo *loading* (no hay `loading.tsx`).

### 4.4 Autenticación (`/login`, `/signup`, `/activar/continuar`, `/onboarding`)

Primera impresión y superficie sin marca de tenant. Hoy son tarjetas
centradas correctas pero planas. Pulido barato y de alto retorno: un
`.brand-wash` detrás de la tarjeta, la tarjeta a `shadow-overlay`, y el
`Brand` arriba con el gradiente nuevo. **No** usar `.brand-band` acá (hay
inputs — lo prohíbe §1.6).

### 4.5 Panel de la organización — Agenda y detalle de ocurrencia

`org/[slug]/agenda`, `agenda/[occurrenceId]`, `.../asistencia`. Es lo que el
dueño abre desde el teléfono. Ya está en densidad `touch` (Fase L, punto 6).
Lo pendiente es la **deuda declarada** de esa misma fase: Clientes, Pagos,
Servicios, Recursos y Equipo siguen en densidad de escritorio. Esta ADR
propone saldarla en su propia fase (§5, Fase 6), no antes: sigue necesitando
verificación visual con `next dev`, que es exactamente lo que faltó.

### 4.6 Consola de plataforma (`/admin`)

Es nuestra (ADR-0020). Hereda el violeta y **además** debería distinguirse a
simple vista de un panel de organización, para que nadie confunda en qué
consola está operando: propuesta mínima, un `.eyebrow` "Consola de
plataforma" con un punto `--brand-cyan` en el header. Sin inventar un segundo
sistema visual.

### 4.7 Inconsistencias ya existentes que este pase debería arreglar de paso

1. **Dos `EmptyState`.** `components/empty-state.tsx` es un re-export de
   `components/ui/empty-state.tsx` y hay pantallas importando de las dos
   rutas (`[organizationSlug]/page.tsx:12` usa la vieja). Migrar los call
   sites y borrar el shim.
2. **`BrandMark` usado como ícono genérico** en `page.tsx:81`, con
   `bg-none shadow-none` cancelando la mitad de sus propios estilos. Si hace
   falta un contenedor de ícono, es una primitiva (`IconTile`), no el logo
   desactivado.
3. **Falta `loading.tsx` y `error.tsx`** en la raíz y en las rutas públicas
   de más tráfico. El sistema visual exige los cuatro estados en cada
   pantalla y hoy *loading* y *error* dependen del default de Next.

---

## 5. Plan de fases

Cada fase es un commit (o PR) propio en el repo `frontend/`, verificable por
separado. **El orden está elegido para que el riesgo suba recién cuando ya
hay evidencia visual de que la paleta funciona**, y por eso la landing va
antes que el cambio de `--primary`, no después.

| Fase | Qué | Riesgo | Depende de |
|---|---|---|---|
| **1 — Tokens, aditiva** | Agregar capa P (§1.1), `.brand-band` (§1.6), `.display-*`/`.lead` (§2). **No tocar `--primary` ni los neutros.** | **Nulo.** Ningún píxel existente cambia: son variables que todavía nadie usa. | — |
| **2 — Landing** | `app/page.tsx` completa (§3) + `components/marketing/*` + query a `plans` + `error.tsx` raíz. | Bajo. Ningún cliente logueado la ve (hay `redirect` con sesión). | 1, y §6.2 resuelta (destino del CTA) |
| **3 — De-fuga de la marca** | `Brand`/`BrandMark` pasan a capa P (§0.4). `IconTile` nuevo para §4.7.2. | Bajo, pero **visible para tenants con acento**: su header cambia de color de logo. Avisar al cliente actual. | 1 |
| **4 — Flip de `--primary` + neutros** | §1.2 y §1.3. | Medio: toca toda superficie sin `[data-brand]`. Mecánico, pero **requiere pasada visual con `next dev`** por login, signup, `/dashboard`, `/me` ×4, `/admin`. | 2 (la landing ya probó que el violeta funciona en pantalla) |
| **5 — Superficies de cliente** | §4.1, §4.2, §4.3 + los `loading.tsx` faltantes. **Verificar con al menos 3 acentos de tenant distintos** (uno claro tipo amarillo, uno saturado, uno oscuro) para confirmar que nada asumió violeta. | Medio. Es la superficie que ven clientes de clientes. | 3, 4 |
| **6 — Deuda de densidad del panel** | Saldar Fase L punto 6: Clientes, Pagos, Servicios, Recursos, Equipo a `touch`. | Medio-alto por volumen (decenas de archivos). **No arrancar sin `next dev` disponible** — es el error que la Fase L evitó a propósito. | 4 |
| **7 — `/admin`** | §4.6. | Nulo (una consola, un usuario). | 4 |

Fases 2 y 3 pueden ir en paralelo si hay dos pases disponibles. **La 4 no
arranca hasta que la 2 esté en producción y mirada.** La 6 es explícitamente
posponible sin bloquear nada.

---

## 6. Lo que no puedo decidir / necesita al Orchestrator

1. **El violeta exacto.** Propongo `#7c3aed` (violet-600 de la escala de
   Tailwind) por dos razones concretas: pasa AA con blanco (5.70:1) con
   margen, y su vecino `#6d28d9` da un hover genuinamente más oscuro y un
   texto violeta sobre blanco AAA. Si preferís uno más frío (más cerca del
   índigo de hoy) o más magenta, decime el hex y recalculo los derivados —
   pero necesito que el elegido pase 4.5:1 con blanco o hay que cambiar
   `--primary-foreground`, y eso arrastra a `[data-brand]`.
2. **Destino del CTA principal — bloqueante para la landing.** La creación de
   organización está **gated por código de invitación** (ADR-0017;
   `onboarding-form.tsx:19-32`: "Te lo damos al contratar el plan"). Es
   decir: **hoy no existe alta self-service, y el CTA actual "Empezar gratis"
   (`page.tsx:66`) promete algo que el producto no hace.** Una landing con
   funnel y precios necesita una salida real. Opciones: (a) ruta `/contacto`
   con formulario que genere un lead, (b) link a WhatsApp/mail (cero
   desarrollo, coherente con la activación por WhatsApp de ADR-0026), (c)
   abrir el signup self-service con `TRIALING`, que es una decisión de
   producto y no mía. **Elegí una antes de la Fase 2**; mi recomendación es
   (b) para no bloquear, con (a) después.
3. **Moneda y presentación del precio.** `plans.monthly_price_usd` está en
   USD y la audiencia es uruguaya. ¿Se muestra "USD 20/mes"? ¿Se aclara
   IVA? ¿Hay precio en pesos? No lo invento.
4. **Testimonios.** ¿El cliente actual autoriza a ser nombrado/mostrado?
   Mientras no haya un sí explícito, §3.6 va con señales factuales y **no se
   fabrica ninguna reseña**.
5. **Dominio.** La UI dice `reservaste.app` (`onboarding-form.tsx:41`) pero
   ADR-0021 despliega en `161-35-63-60.sslip.io`. Una landing con precios
   sobre una URL `sslip.io` daña justamente la confianza que la landing
   existe para construir. ¿Hay dominio comprado? Es previo a la Fase 2.
6. **Vocabulario por organización ("clase" vs "turno").** Hay copy de rubro
   filtrado en pantallas de cliente (§4.3). La corrección mínima es
   neutralizar los textos. La correcta es un término configurable en
   `Organization` (singular/plural), que es **una columna nueva y por lo
   tanto tu decisión, no mía** — si te interesa, pediría una propuesta corta
   a `domain-architect`.
7. **Alcance de la tipografía.** ¿Aceptás `font-optical-sizing: auto` global
   (§2), que afecta sutilmente todo el producto, o preferís que la escala
   display quede confinada a la landing y nada más cambie?
8. **`.brand-band` en la landing.** Necesito tu visto bueno explícito de que
   una banda oscura acotada (§1.6) no cuenta como revivir `.dark`. Si
   preferís cero superficies oscuras, el CTA final se resuelve con un fondo
   `--primary-subtle` — pierde impacto pero la propuesta no se cae.

---

## 7. Alternativas descartadas

1. **Cian como color de acción (botones).** Descartada: `#22d3ee` da 1.7:1
   con blanco y ~2.9:1 con negro. Un botón cian accesible obliga a un cian
   tan oscuro que deja de ser vibrante. El cian rinde como decoración y como
   segundo stop de gradiente, que es además cómo lo usa la referencia.
2. **Dejar que la organización configure toda la paleta.** Ya descartada por
   ADR-0020 ("no un editor de temas") y la ratifico: un tenant que puede
   tocar los semánticos produce un "Cancelar" verde.
3. **Cambiar la familia tipográfica** (§2). Riesgo de layout en 60 pantallas
   a cambio de un delta que el tamaño y el peso ya entregan.
4. **Armonizar `--success`/`--warning`/`--destructive` con el violeta.**
   Descartada: son funcionales, no de marca (§1.4).
5. **Carrusel de rubros auto-rotativo**, como la referencia (§3.3).
   Descartada por WCAG 2.2.2 y porque una grilla de tres dice lo mismo sin
   JS.
6. **Landing después del flip de `--primary`** (el orden "intuitivo").
   Descartada en §5: la landing es la única superficie donde se puede
   evaluar la paleta nueva sin exponer a un cliente logueado. Probar primero
   donde no duele.
7. **Violeta/cian sólo en la landing, producto índigo.** Descartada en §0:
   es el pedido explícito del usuario al revés, y parte la marca en dos.

---

## 8. Resumen ejecutivo

**El problema de fondo no es el color, es que `--primary` significa dos cosas
a la vez.** Hoy el logo de Reservaste se pinta con `--primary`, y como el
panel y la página pública de cada organización viven dentro de
`[data-brand]`, **nuestro logo se tiñe con el color del cliente** — lo
contrario de lo que ADR-0020 decidió. Todo lo demás de esta propuesta se
apoya en separar eso.

**Tres capas de token, una regla:** `--brand-*` es Reservaste (fija, ningún
tenant la pisa: logo, landing, `/admin`); `--primary` es *el acento de quien
es dueño de la pantalla*; los semánticos (success/warning/destructive) no se
tocan. El violeta `#7c3aed` **pasa a ser el valor por defecto de
`--primary`**, así que toda superficie sin `[data-brand]` —auth,
`/dashboard`, `/me`, `/admin`, la landing— se renueva cambiando cuatro
variables, y **ninguna organización con acento configurado ve cambiar su
página**. Cian `#22d3ee` es decorativo, sólo en superficies de plataforma,
nunca con texto encima (el gradiente violeta→cian no pasa AA). Neutros:
rotación de matiz de 253/258 a 285/288, sin tocar luminancia. `.dark` sigue
muerto; la banda oscura de la landing está acotada por escrito y no es dark
mode.

**Tipografía:** se queda Inter. Cambia sólo la escala, y sólo en la landing
(`.display-1/2`, `.lead` con `clamp`), confinada a `components/marketing/`
para que la densidad de marketing no se filtre al panel.

**Landing:** nueve bloques de funnel (hero con prueba visual construida con
tokens reales, no una captura → rubros → 6 funciones → cómo funciona →
confianza → precios → FAQ → CTA → footer). Tres rubros distintos, ninguno
gimnasio como caso único. **Los precios se leen de la tabla `plans`, que ya
es pública** (Starter 20 / Pro 40 / Full 80 USD), con `null` = "Sin límite" y
los cuatro estados definidos — sin fallback hardcodeado, porque un precio
viejo es peor que ninguno. FAQ con `<details>` nativo: teclado y lectores de
pantalla gratis, cero dependencias. **Sin testimonios inventados.**

**Siete fases**, ordenadas para que el riesgo suba recién con evidencia:
tokens aditivos (riesgo nulo) → landing (nadie logueado la ve) → de-fuga del
logo → flip de `--primary` → superficies de cliente → deuda de densidad del
panel → `/admin`.

**Lo bloqueante para vos, antes de implementar:** (1) el CTA principal
promete un alta self-service **que no existe** — la creación de organización
está gated por invitación (ADR-0017), así que hace falta decidir a dónde
lleva el botón; (2) el dominio real (`reservaste.app` vs. la IP de
ADR-0021); (3) moneda/IVA en precios; (4) confirmar el hex del violeta y el
visto bueno de la banda oscura.
