# Iteración 3 — Plan (feedback del primer cliente)

Estado: **propuesta, pendiente de aprobación del usuario**
Fecha: 2026-09-21
Mantenido por: Orchestrator

Origen: feedback del primer cliente real (estudio de pilates) después de la
demo. Este documento traduce ese feedback a fases ejecutables con agentes
asignados. **Nada se implementa hasta que el usuario apruebe fase por fase.**

---

## 1. El feedback, tal como llegó

1. Cambiar/evaluar el registro: otras apps de reserva no exigen usuario. Él
   cree que conviene exigirlo, pero quiere evaluarlo.
2. El dueño habilita al cliente por **número de teléfono**, y eso abre un
   link de registro para ese negocio vía
   `https://api.whatsapp.com/send?phone={client_number}&text={url_de_activacion}`.
3. **Alta de clientes que no pueden auto-registrarse.**
4. **Alta de planes** (precio y veces por semana).
5. Caso pilates: individual 800, 1x/semana 1800, 2x 2500, 3x 3400.
6. Limitado a **3 personas en paralelo**.
7. Cada usuario paga una suscripción que **bloquea el espacio en ese horario
   específico**.
8. Si alguien con cupo fijo no asiste y **avisa liberando el cupo X tiempo
   antes**, puede **recuperar dentro del mismo mes**.
9. Un cupo liberado **se visualiza en la agenda pública**.
10. El que reserve **debe validarse como candidato o no**, considerando cupos
    a recuperar.
11. Usuario **sin cupo ⇒ siempre paga** (tarjeta o carga manual del admin).
12. Mejorar **toda la UI/UX**, para clientes y para dueños.

---

## 2. Qué ya está hecho y qué no

| Pedido | Estado hoy |
|---|---|
| 6 — 3 personas en paralelo | **Hecho.** `slot_occurrences.capacity` + `book_slot()` con lock real (ADR-0004). |
| 7 — la suscripción bloquea un horario específico | **Hecho como mecanismo.** `RecurringBooking` + reserva fija desde el mostrador (ADR-0018). Falta atarlo a un plan y a una frecuencia. |
| 9 — cupo liberado visible | **Casi.** `get_public_availability()` cuenta reservas reales, así que una cancelación ya reabre el cupo. Falta que la agenda pública lo *destaque* y que el candidato sepa que puede tomarlo. |
| 11 — carga manual del admin | **Hecho.** Módulo de Pagos (Fase F, ADR-0023). Falta la tarjeta. |
| 4, 5 — planes con precio y frecuencia | **No existe.** Hoy la unidad de cobro es el `Service` (ADR-0022): un solo precio, un solo ciclo, sin frecuencia. |
| 8, 10 — liberar y recuperar | **No existe.** Ningún concepto de crédito de recupero. |
| 2, 3 — alta por teléfono / cliente sin cuenta | **No existe.** `customers.profile_id` es `NOT NULL`; `enroll_customer_by_email` falla con `PROFILE_NOT_FOUND` si la persona no tiene cuenta. |
| 11 — tarjeta | **No existe.** Sin pasarela, sin `payment_intents`, sin webhook. |
| 12 — UI/UX | **Base mínima.** `components/ui` tiene 5 primitivas (button, card, input, label, separator). No hay dialog, sheet, toast, table, skeleton ni manejo visual de errores de formulario. |

---

## 3. El hallazgo central

Los puntos 4, 5, 7, 8, 10 y 11 parecen seis pedidos y son **uno**.

Hoy la regla de reserva es: *"¿existe un `Payment` PAID de este cliente para
este servicio cuyo período cubra la fecha del turno?"* (`payment_covers_slot`,
ADR-0022). Si la respuesta es sí, el cliente puede reservar **cuantos turnos
quiera** de ese servicio en ese período.

El cliente de pilates está describiendo algo distinto: el pago mensual **no
compra acceso ilimitado al período, compra una cantidad de cupos fijos por
semana**. Todo lo demás cae solo de ahí:

- "Alta de planes (precio y veces por semana)" = el pago se ancla a un plan
  que declara la frecuencia, no al servicio pelado.
- "La suscripción bloquea el espacio en ese horario específico" = los N cupos
  del plan **son** N `RecurringBooking`. La frecuencia no es un contador que
  haya que chequear en cada reserva: es el máximo de series activas.
- "Si soy un usuario sin cupo, siempre debo pagar" = una reserva que **no**
  proviene de una serie activa es una reserva *extra*, y una extra no la
  cubre el pago mensual.
- "Puede recuperar dentro del mismo mes" = el crédito de recupero es
  exactamente lo que habilita una reserva extra sin pagar de nuevo.
- "El que reserve debe validar si es candidato o no, con cupos a recuperar" =
  `evaluate_customer_booking()` suma un camino nuevo: extra + crédito
  disponible ⇒ OK; extra sin crédito ⇒ hay que pagar.

**Sin esa inversión, los pedidos se contradicen entre sí**: un plan de 1x
semana no significa nada si el mismo pago ya habilitaba reservar los 5 días, y
un crédito de recupero no tiene valor si reservar de más nunca costó nada.

Esa inversión es el ADR principal de esta iteración.

### La compatibilidad no se rompe: `UNLIMITED` es un tipo de plan

El comportamiento actual (pago mensual = acceso ilimitado al período) deja de
ser *la* regla del producto y pasa a ser **un tipo de plan con nombre**. La
migración le da a cada `Service` con cobro mensual un plan `UNLIMITED` con su
precio actual: nadie cambia de comportamiento el día del deploy. Es el mismo
patrón aditivo que funcionó en ADR-0022 (y por el que ahí no se rompieron 16
tests de una vez).

---

## 4. Decisiones estructurales que esta iteración necesita

Ninguna la toma un subagente por su cuenta (CLAUDE.md). Cada una entra como
ADR en `docs/decisions.md` antes de que se escriba el código de su fase.

### ADR-0024 — `ServicePlan`: qué compra un pago

- Tabla nueva **`service_plans`** — no `plans`, que ya son los planes del SaaS
  que paga el gimnasio. Nombre genérico a propósito: un consultorio vende
  "turno fijo semanal" (`WEEKLY_QUOTA` 1) y "consulta suelta" (`DROP_IN`), y
  una cancha lo mismo.
  **Corrección (2026-09-21):** acá había puesto "4 consultas al mes" como
  prueba de genericidad y estaba mal — eso es un contador sobre un período,
  no un conjunto de horarios, o sea el paquete prepago (`CREDITS`) que
  ADR-0022 eliminó a propósito. Queda **fuera de alcance**: meterlo obligaría
  a `evaluate_customer_booking()` a tener dos caminos de cuota distintos.
- Shape propuesto: `{ id, organization_id, service_id, name, price,
  billing_type, billing_cycle, plan_kind, weekly_quota, is_active, ... }`.
- **`plan_kind`**: `DROP_IN` (clase suelta, sin frecuencia) |
  `WEEKLY_QUOTA` (N cupos fijos por semana) | `UNLIMITED` (el comportamiento
  de hoy, para no romper a nadie).
- `weekly_quota` es obligatorio solo cuando `plan_kind = 'WEEKLY_QUOTA'`, vía
  CHECK — no una convención que la UI recuerde.
- **`payments.service_plan_id`**, con backfill desde el plan `UNLIMITED` que
  genera la migración. `service_id` se mantiene (es derivable del plan, pero
  sacarlo rompería el módulo de Pagos sin necesidad).
- El caso pilates queda: un `Service` "Pilates" con capacidad 3, y cuatro
  `service_plans` — DROP_IN 800, WEEKLY_QUOTA 1 → 1800, 2 → 2500, 3 → 3400.
  **Un solo servicio**: modelar "1x semana" y "2x semana" como dos `Service`
  partiría la capacidad de 3 en dos cupos separados, que es justo lo que no
  se quiere.

**Correcciones al shape, del diseño de `domain-architect`** (propuesta completa
en [`proposals/adr-0024-service-plan.md`](proposals/adr-0024-service-plan.md)),
todas verificadas contra el schema real:

- **`billing_type`/`billing_cycle` suben al plan.** Dejarlas en `Service` las
  vuelve una segunda fuente de verdad: un servicio con un plan suelto
  (`ONE_TIME`) y tres mensuales no se puede expresar con una sola columna.
  `billing_period_for()` pasa a recibir el plan; es barato, hoy solo la
  llaman la propia migración y los tests de phase7. `payment_required` se
  queda en `Service`.
- **La cuota no se cuenta contra series `ACTIVE`, se cuenta contra series en
  vigencia para la fecha local del turno.** ADR-0010 deja una serie `ACTIVE`
  aunque su `end_date` ya pasó, así que contar por estado sobrecuenta. Misma
  lección que ADR-0013: se evalúa contra la fecha del turno, nunca contra
  `now()`. Y hace falta un desempate determinístico (`created_at, id`) para
  cuando hay más series que cuota.
- **La función de decisión necesita contexto de serie.** Riesgo concreto y
  verificado: `admin_preview_recurring_booking` llama a
  `evaluate_customer_booking()` ocurrencia por ocurrencia como si fueran
  reservas puntuales. Sin hilar ese contexto, el preview de una reserva fija
  diría "fuera de tu frecuencia" en **todas** las fechas, justo al cliente
  que tiene el plan que la permite.
- **Al `EXCLUDE` le falta la otra mitad.** Excluir del constraint a los pagos
  anclados a una ocurrencia los deja sin ninguna protección de doble cobro:
  hace falta además un índice único por `(customer, slot_occurrence)` sobre
  los pagos `PAID`.
- **Desactivar un plan no puede invalidar cobertura ya pagada.** Un `join
  service_plans ... and is_active` le cortaría la cobertura a todo el mundo el
  día que el dueño ordena su lista de precios. Queda como invariante escrita.
- Inconsistencia preexistente que este trabajo destapa:
  `ALREADY_HAS_STANDING_RESERVATION` tampoco mira `end_date`.
- **Restricción de secuenciación:** la migración y la pantalla de pagos con
  selector de plan tienen que salir en la **misma** fase. El formulario actual
  inserta en `payments` por PostgREST directo sin saber de planes.

### ADR-0025 — Crédito de recupero

- Tabla **`makeup_credits`**: `{ id, organization_id, customer_id, service_id,
  source_booking_id, issued_at, expires_at, status:
  AVAILABLE|CONSUMED|EXPIRED, consumed_booking_id?, ... }`.
- **Se emite** cuando una `Booking` que viene de una `RecurringBooking` activa
  se cancela con al menos X horas de anticipación, y el cancelador es el
  cliente (o el mostrador en su nombre). **No se emite** para una reserva
  suelta ("exceptuando clase unitaria", punto 8 textual) ni para una
  cancelación tardía.
- **La política la configura el dueño** (decisión del usuario, 2026-09-21), no
  es una constante del producto: `release_deadline_hours` (cuánta anticipación
  hace falta) y `makeup_credit_expiry` (`END_OF_MONTH` | `DAYS_AFTER` + N
  días). Defaults 12 h y `END_OF_MONTH`, que es lo que el cliente describió.
  Vive a nivel `Organization` con override opcional por `Service`: un estudio
  puede querer 24 h para pilates y 2 h para una cancha, y ya existe el
  precedente de `public_availability_display` (ADR-0008) configurable así.
  **Se valida en la base**, no en el formulario: la columna es escribible por
  PostgREST directo (misma razón que ADR-0020 para el color de marca).
- **El valor se congela al emitir el crédito.** `expires_at` se calcula y se
  guarda en la fila; si el dueño cambia la política mañana, no se le mueve el
  vencimiento a un crédito ya entregado. Una política retroactiva sobre algo
  que el cliente ya tiene en la mano es un reclamo en el mostrador.
- **Se consume** dentro de `book_slot()`, bajo el mismo lock que el cupo — si
  no, dos reservas simultáneas gastan el mismo crédito. Un crédito consumido
  vuelve a `AVAILABLE` si esa reserva se cancela y el crédito no venció.
- Motivos nuevos de `can_book_reason`: distinguir "te falta pagar esta clase
  suelta" de "estás fuera de tu frecuencia y no tenés crédito". Mostrar el
  mismo mensaje para los dos es lo que Phase 12 ya tuvo que corregir una vez.
- **Contesta la pregunta abierta de ADR-0019** (quién se queda con un cupo
  liberado): el cupo vuelve a la agenda pública y es por orden de llegada; lo
  que cambia entre candidatos no es la prioridad, es el precio — con crédito
  no paga, sin crédito paga.

### ADR-0026 — Cliente gestionado + activación por teléfono

- **`customers.profile_id` pasa a nullable**, más datos de contacto en la fila
  (`display_name`, `phone`). Un cliente gestionado existe, se le agenda y se
  le cobra, pero **no puede iniciar sesión ni ve nada** — no tiene identidad.
- Tabla **`customer_activations`**: `{ token, organization_id, customer_id,
  phone, expires_at, redeemed_at, redeemed_profile_id, created_by }`.
- Flujo: el dueño da de alta nombre + teléfono → el sistema genera el token →
  la UI arma el link `api.whatsapp.com/send?phone=…&text=…` y **el dueño lo
  manda desde su propio WhatsApp** (sin Twilio, sin costo por mensaje, sin API
  de WhatsApp Business) → el cliente abre, crea cuenta o inicia sesión, y al
  redimir se **vincula** su `profile_id` a la fila de cliente que ya tenía
  historial.
- Riesgo real a cerrar con `auth-security-agent`, no con buenas intenciones:
  **el link es un secreto portador** — cualquiera que lo tenga se convierte en
  ese cliente. Token de alta entropía, un solo uso, con vencimiento, con rate
  limit, y el redeem tiene que ser imposible de apuntar a otra fila de
  cliente. También: `phone` es dato privado y no puede filtrarse por ninguna
  vista pública, y hay que repasar cada policy que hoy compara `profile_id`
  con `auth.uid()` para confirmar que un `NULL` no abre nada.
- Respuesta al punto 1 del feedback: **se exige cuenta para el autoservicio**
  (reservar, cancelar, ver el portal) porque sin identidad no hay auditoría de
  quién canceló ni a quién se le emite un crédito — pero deja de ser un muro
  de entrada, porque el dueño puede operar a un cliente sin cuenta desde el
  primer día y la activación por WhatsApp baja la fricción a un tap.

### ADR-0027 — Cobro con tarjeta

- `payments` suma `source: MANUAL | ONLINE` y aparece **`payment_intents`**
  (intento ≠ pago: un intento puede fallar, expirar o duplicarse; un `Payment`
  PAID recién nace del webhook confirmado).
- **La pasarela se decide con el usuario** (Uruguay: Mercado Pago, dLocal,
  Plexo). La arquitectura se construye contra una interfaz de proveedor para
  que la elección no toque el motor de reservas.
- Trampa a resolver explícitamente: hoy un `EXCLUDE` impide dos `Payment` PAID
  con períodos solapados del mismo `(customer, service)`. Una clase suelta
  comprada por alguien que **ya** tiene plan mensual cae dentro de ese período
  y sería rechazada. Se resuelve anclando el pago de una clase suelta a la
  ocurrencia (`slot_occurrence_id`) en vez de a un período, y dejando el
  `EXCLUDE` solo para los pagos por período.

---

## 5. Fases

Una por vez (regla de completitud del roadmap). Cada una cierra con
`/phase-review`: QA corre tests → Code Review revisa → Orchestrator aprueba y
asienta el ADR.

### Fase L0 — Sistema visual base — **COMPLETA** (2026-09-22)

10 primitivas nuevas en `components/ui/` (dialog, sheet, toast, table +
`DataList`, skeleton, badge, alert, form, select, empty-state) más tokens de
elevación, `.eyebrow` y `.focus-ring` en `globals.css`, aplicadas a 32
archivos: +382/−338, o sea el pase **sacó casi tanto código como agregó**.
Typecheck limpio y 27/27 tests, verificado por el Orchestrator (el
`ui-ux-agent` no tiene Bash y reportó que no podía correrlos, en vez de
inventar números).

**Sin dependencias nuevas.** `@base-ui/react` ya estaba en el proyecto y
cubre dialog, drawer y toast: no hizo falta Radix ni una librería de toasts.

Lo que el pase destapó, que es el argumento de por qué iba primero: 9 copias
divergentes de la misma constante de `<select>` (una sin anillo de foco), 18
errores de formulario inline **ninguno con `role`**, 8 estados vacíos con
tres paddings distintos, y 8 controles hechos a mano sin foco visible.

Documentado en `architecture.md` → "Sistema visual".

#### Backlog que dejó para la Fase L

1. **Tres alturas de control** (Button `h-8`, Input `h-8`, `Button size="lg"`
   `h-9`): hay que elegir una y escribirla.
2. Los dos `window.confirm()` de `service-row` / `resource-row` → no era
   reemplazo directo: el `confirm()` bloquea un `formAction` dentro del form
   y el diálogo se portalea fuera. `ConfirmDialog` ya está listo.
3. Densidad `touch` en el panel admin (hoy aplicada solo a las 4 pantallas de
   cliente/público): cambiar la densidad de escritorio pide revisión visual.
4. **`lucide-react` está instalado y no lo usa nadie**, con íconos dibujados a
   mano en `icons.tsx`. **Resolución del Orchestrator:** se adopta en la Fase
   L y `icons.tsx` pasa a re-exportar de ahí — tener un set de íconos pago y
   dibujar a mano es la deriva que L0 vino a cortar. Sacarlo sería la otra
   opción válida, pero la Fase L necesita íconos igual.
5. `shadcn` está en `dependencies` (no en dev) y `globals.css` importa
   `shadcn/tailwind.css`: revisar al mirar el bundle.
6. `.dark` sigue siendo código muerto (ya anotado en ADR-0020): implementarlo
   o borrarlo.
7. Los "formularios que se abren inline" (enroll, invite ×2,
   occurrence-actions) son cuatro variantes del mismo gesto: piden primitiva
   propia o convertirse en Sheet.
8. `PageHeader` acepta `title` pero varias pantallas escriben su propio
   `<h1>`; y `ChevronRight` aparece como afordancia de fila sin regla.
9. **El sheet no se vio renderizado** (nadie puede correr la app todavía): el
   swipe-to-dismiss depende de una transición CSS sin verificar. Trigger,
   backdrop, Escape y cerrar funcionan igual.

#### Notas para la Fase H (anotadas, no construidas)

- La pantalla de planes va como **cuarta pestaña en `service-tabs.tsx`**, lo
  que de paso saca el cobro de la pestaña "Horarios", donde hoy está y
  confunde.
- `billing-form.tsx` ya tiene el patrón exacto: un `Select` que revela
  condicionalmente otro campo. `plan_kind === 'WEEKLY_QUOTA'` → revelar
  `weekly_quota` es el mismo gesto. **El CHECK de coherencia vive en la base
  (ADR-0024); la UI solo revela el campo.**
- El selector de plan del pago va en `RegisterPaymentForm`
  (`customers/[customerId]/customer-forms.tsx`), que ya es client component y
  **tiene dos call sites con props idénticas** — `customers/[customerId]` y
  `payments/[customerId]` —, así que sumarle `plans` toca las dos pantallas.
- La lista de planes es `DataList`, no `Table`: el dueño la abre desde el
  teléfono.

### Fase L0 — alcance original

**Por qué antes que todo lo demás:** las fases H, I y J traen pantallas nuevas
(planes, créditos, alta por teléfono). Si se construyen sobre las 5 primitivas
actuales, hay que rehacerlas cuando llegue el pase de UI/UX. Es la única parte
del punto 12 que conviene adelantar; el resto del pulido va al final, cuando
haya pantallas terminadas que pulir.

- Tokens (espaciado, tipografía, radios, elevación, estados) consistentes
  entre admin, portal y público, respetando el acento por organización de
  ADR-0020.
- Primitivas que faltan y que las fases siguientes van a necesitar: dialog,
  sheet/drawer (mobile), toast, table, skeleton, badge, errores de formulario,
  empty states.
- Agentes: **`ui-ux-agent`** (lead) → revisión de `code-review-agent`.

### Fase H — Planes con precio y frecuencia

Implementa ADR-0024. Puntos 4, 5 y la mitad del 7.

**Backend: COMPLETO** (2026-09-22) — `20260922120000_phase17_service_plans.sql`,
1547 líneas, los 11 pasos en orden. Verificado por el Orchestrator: 131/131
integración y 35/35 unitarios **contra base reseteada desde cero**, typecheck
limpio, `FOR UPDATE` intacto, y el diff de tests son call sites sin una sola
aserción tocada (los 27 que fallaron al principio fallaban todos porque el
harness no creaba planes — se arregló el harness, no la expectativa).

Backfill verificado con datos reales, no sobre base vacía: 4 servicios de
formas distintas + 2 pagos → 3 planes `UNLIMITED` creados con el ciclo
preservado, el servicio gratis sin plan, los pagos anclados, `NOT NULL`
aplicado sin fallar.

Tres correcciones del Orchestrator sobre lo entregado, todas verificadas
después:

1. `payments.slot_occurrence_id` → **`on delete restrict`** (el DDL de la
   propuesta decía `cascade` y chocaba con "un pago nunca se borra": habría
   borrado el registro económico en silencio).
2. **`create_recurring_booking()`** (autoservicio del cliente) nunca tuvo el
   chequeo de serie duplicada — solo el camino admin. Es load-bearing: todo
   el argumento de "k series = k reservas por semana" se apoya en que no haya
   dos series sobre la misma regla.
3. **`admin_preview_recurring_booking`** evaluaba siempre como serie nueva, así
   que a un cliente que ya tenía cupo fijo ahí le decía `OVER_PLAN_QUOTA` en
   vez de la verdad. Un preview que miente sobre el motivo es justo lo que
   ADR-0018 vino a resolver.

Bug preexistente cerrado de paso: `ALREADY_HAS_STANDING_RESERVATION` ignoraba
`end_date`, así que una serie terminada en junio bloqueaba crear una nueva en
septiembre.

Los 25 casos de test que la fase tiene que sumar quedaron listados por
`database-agent` para `qa-testing-agent`. Los que no pueden faltar: preview de
serie prospectiva con cuota libre da `OK` en **todas** las fechas, plan
desactivado con pago vigente **sigue confirmando**, y serie con `end_date`
pasado **no** consume cuota.

**UI: COMPLETA** (2026-09-22) — pestaña Planes (segunda, pegada a Horarios:
el dinero junto a la configuración), selector de plan en el registro de pago
con las dos call sites, los tres motivos nuevos en castellano, y
`upcoming_over_quota` visible. Typecheck limpio y **36/36** tests (27 previos
+ 9 nuevos de `lib/plan-labels`), verificado por el Orchestrator. El bloqueo
de publicación del paquete se resolvió pusheando el backend a `main`
(`60dd824`).

Dos decisiones de UX del agente que vale la pena conservar:

- **Los términos del plan se deshabilitan siempre en edición, no solo cuando
  hay pagos** — `updateServicePlanSchema` nunca los manda, así que decir "está
  deshabilitado porque ya tiene pagos" en un plan sin pagos sería mentira. El
  texto escala según haya pagos o no, y las dos versiones terminan en la misma
  salida: desactivar y crear otro.
- **"Desactivar no corta cobertura" se dice tres veces a propósito** (diálogo
  de confirmación, pie de la lista, bloque de términos inmutables): es el
  invariante que más fácil se malinterpreta.
- Vocabulario genérico sostenido contra el enunciado de la tarea: `DROP_IN` es
  **"Turno suelto"**, no "clase suelta", con un test que falla si alguien mete
  "clase/socio/entrenador/alumno" en esos labels.

#### Lo que la UI destapó, y qué decidí

1. **Un CHECK de Phase 14 quedó gobernando una columna deprecada.**
   `services_payment_requires_priced_type` (`not payment_required or
   billing_type <> 'FREE'`) expresaba un invariante correcto **bajo la
   semántica de Phase 14**, cuando el precio vivía en el `Service`. Con
   ADR-0024 bloquea algo legítimo: `billing_type` arranca en `'FREE'`, así que
   un formulario que ya no pide modalidad de cobro nunca podría encender
   `payment_required`. **Migración 18 lo borra**; el invariante no se pierde,
   ya vive como `SERVICE_HAS_NO_PLAN` en el camino de decisión, donde se
   evalúa al reservar en vez de como CHECK sobre una columna que nadie debe
   volver a leer. El puente que la UI metió para esquivarlo se saca después.
2. **`DROP_IN` no tiene camino de cobro todavía** (el pago suelto exige
   `slot_occurrence_id` y no hay selector de ocurrencia). Los planes `DROP_IN`
   quedan filtrados del selector de pago. **Va a la Fase I**, que es donde vive
   "sin cupo siempre pagás".
3. **Planes gateados a OWNER** en la server action, siguiendo el precedente de
   `updateServiceBilling`. Es política de producto, no frontera de seguridad:
   la RLS admite a cualquier miembro. Lo mira `auth-security-agent` en la
   Fase M.
4. **El portal del cliente muestra precio deprecado** (`me/servicios` lee
   `services.price`). Después de esta fase esa información puede ser
   directamente falsa. Necesita que `my_services()` exponga los planes: **va a
   la Fase I**, que ya toca el portal.
5. **`organizations.currency` no tiene UI**: toda organización queda en `UYU`
   para siempre. Se agrega a Configuración.
6. N+1 menor en `listPaymentPlanOptions` (un `billing_period_for()` por plan).
   Aceptable con decenas de planes; anotado.
7. `billing-form.tsx` renombrado a `service-settings-form.tsx` por el
   Orchestrator — el agente no tiene Bash para `git mv`.

**Deuda de verificación de toda la fase:** ningún agente pudo ver una pantalla
renderizada (no hay shell para `next dev` ni credenciales de una cuenta real).
Los estados están implementados, no verificados visualmente. Es la misma deuda
que ADR-0023 ya había anotado y que la Fase M tiene que cerrar.

- Migración 17 (aditiva): `service_plans`, `payments.service_plan_id`,
  backfill `UNLIMITED`, CHECKs de coherencia, re-anclaje del `EXCLUDE`.
- Cupo fijo limitado por la frecuencia del plan: crear la serie N+1 para un
  cliente con plan de N se rechaza **en la RPC**, no en la pantalla.
- UI admin: alta/edición de planes por servicio; al asignar un cupo fijo se ve
  cuántos le quedan disponibles al cliente según su plan.
- Agentes: **`domain-architect`** (shape) → **`database-agent`** (migración,
  constraints, índices) → **`payments-entitlements-agent`** (qué cubre un
  pago) → **`backend-api-agent`** (acciones) → **`frontend-admin-agent`** (UI)
  → **`qa-testing-agent`** → **`code-review-agent`**.

### Fase I — Liberar y recuperar

Implementa ADR-0025. Puntos 8, 9, 10 y la parte de "sin cupo pagás" del 11.

- Migración 18: `makeup_credits`, política de anticipación configurable,
  emisión en la cancelación, consumo atómico dentro de `book_slot()`,
  devolución al cancelar, vencimiento.
- `evaluate_customer_booking()` suma el camino de la reserva extra. Sigue
  siendo **una sola función** — el preview del mostrador y el del cliente no
  pueden divergir (es lo que ADR-0018 ya tuvo que arreglar).
- Cliente: "liberar mi cupo" con el aviso claro de hasta cuándo sirve el
  crédito, y "mis créditos" en el portal.
- Público: un cupo liberado se destaca en la agenda, y al elegirlo el sistema
  dice si lo toma con crédito o pagando.
- Agentes: **`domain-architect`** → **`booking-engine-agent`** (lead, es
  motor) → **`database-agent`** (concurrencia del consumo) →
  **`payments-entitlements-agent`** → **`frontend-customer-agent`** +
  **`frontend-admin-agent`** → **`qa-testing-agent`** →
  **`code-review-agent`**.
- Tests que esta fase **tiene** que tener, si no no cierra: dos reservas
  simultáneas con un solo crédito ⇒ exactamente una gana; cancelación tardía ⇒
  no emite; crédito vencido ⇒ no sirve; cancelar la reserva que consumió el
  crédito ⇒ lo devuelve.

### Fase J — Cliente gestionado + activación por WhatsApp

Implementa ADR-0026. Puntos 1, 2, 3.

- Migración 19: `profile_id` nullable + contacto, `customer_activations`,
  `create_managed_customer()`, `claim_customer_activation()`, repaso completo
  de RLS.
- UI admin: alta por nombre + teléfono, botón que abre WhatsApp con el link ya
  armado, estado visible de la invitación (enviada / activada / vencida),
  reenviar.
- Página de activación: token → signup/login → vinculación → aterriza en el
  portal de ese negocio.
- Agentes: **`auth-security-agent`** (lead — es un camino de identidad nuevo)
  → **`database-agent`** → **`backend-api-agent`** →
  **`frontend-admin-agent`** + **`frontend-customer-agent`** →
  **`qa-testing-agent`** → **`code-review-agent`**.

### Fase K — Cobro con tarjeta *(bloqueada por una decisión del usuario)*

Implementa ADR-0027. Punto 11 completo.

- No arranca hasta que el usuario elija pasarela y consiga credenciales. Lo
  que **sí** se puede hacer sin eso: `payments.source`, `payment_intents`, el
  anclaje del pago de clase suelta a la ocurrencia y la interfaz de proveedor.
- Agentes: **`payments-entitlements-agent`** (lead) → **`database-agent`** →
  **`backend-api-agent`** → **`auth-security-agent`** (firma del webhook,
  idempotencia) → **`frontend-customer-agent`** → **`qa-testing-agent`**.

### Fase L — Pulido de UI/UX

Resto del punto 12, con las pantallas nuevas ya existiendo.

- Dueño: onboarding, agenda con acciones rápidas, jerarquía de información,
  estados de carga y error, densidad en desktop.
- Cliente y visitante: flujo de reserva mobile-first, portal, cómo se comunica
  "no podés reservar y este es el motivo" (que hoy es lenguaje de sistema).
- Agentes: **`ui-ux-agent`** (lead) → **`frontend-admin-agent`** +
  **`frontend-customer-agent`** → **`code-review-agent`**.

### Fase M — Cierre

- Tests de las pantallas (la deuda que ADR-0023 dejó anotada: lo probado son
  las RPC, no las pantallas), revisión de seguridad completa, y verificar todo
  renderizado con sesión iniciada contra el proyecto real.
- Agentes: **`qa-testing-agent`** → **`auth-security-agent`** →
  **`code-review-agent`** → Orchestrator.

---

## 6. Orden y dependencias

```
L0 (sistema visual)
 └─> H (planes + frecuencia)
      └─> I (liberar + recuperar)      I necesita saber qué es "tu frecuencia"
           ├─> J (cliente gestionado)  independiente del modelo de cobro
           └─> K (tarjeta)             necesita el precio del plan (H);
                │                      bloqueada por la elección de pasarela
                └─> L (pulido) ─> M (cierre)
```

J no depende de H ni de I, así que si la activación por WhatsApp es lo que más
urge para vender, puede adelantarse a H sin costo técnico. Es una decisión de
prioridad comercial, no de arquitectura.

---

## 7. Decisiones del usuario — resueltas 2026-09-21

1. **Se exige cuenta para el autoservicio**, más cliente gestionado por el
   dueño sin cuenta. → ADR-0026.
2. **La política de recupero la configura el dueño del local**, no la fija el
   producto. → ADR-0025, con defaults 12 h / fin de mes calendario y override
   por servicio.
3. **Tarjeta postergada**: se construye la arquitectura (`payment_intents`,
   interfaz de proveedor, pago de clase suelta anclado a la ocurrencia) sin
   integrar ninguna pasarela. La elección no bloquea vender. → ADR-0027,
   Fase K parcial.
4. **Arrancamos por L0 → H.**

### Resolución de los 7 puntos que `domain-architect` dejó abiertos en ADR-0024

Resueltos por el Orchestrator el 2026-09-21 (los dos primeros, con el usuario).
Se asientan como ADR-0024 en `decisions.md` cuando el usuario apruebe la
Fase H.

1. **Cambio de plan a mitad de período: VOID + recargar.** Queda **un solo
   pago vigente** por `(customer, service)`, así que nunca hay que inventar
   una regla de "cuál de los dos planes manda" dentro del motor de reservas —
   una regla así se equivoca en silencio. El `EXCLUDE` no se relaja.
2. **Moneda: `organizations.currency`** (ISO 4217, default `UYU`). Cierra
   también el hueco preexistente de `services.price`.
3. **`MONTHLY_QUOTA` queda fuera de alcance.** Es el paquete prepago que
   ADR-0022 eliminó por decisión explícita; reabrirlo por un caso que ningún
   cliente pidió metería un segundo camino de cuota en la única función de
   decisión. El ejemplo del consultorio se corrigió arriba.
4. **Bajar de plan NO cancela series.** Las excedentes se autolimitan
   (`NOT_GENERATED` / `OVER_PLAN_QUOTA`) y ADR-0019 las reconstituye si el
   cliente vuelve a subir. Cancelar series por un cambio de plan destruiría
   la agenda del cliente de forma irreversible a partir de un hecho
   administrativo. El desempate de qué serie queda afuera es
   determinístico (`created_at`, `id`) y la pantalla tiene que mostrar qué
   fechas quedaron pendientes y por qué.
5. **`plan_kind` y `weekly_quota` son inmutables** una vez que el plan tiene
   pagos; **`price` es libre**. Ese corte es el criterio: el precio solo
   afecta cobros futuros y cada `Payment` ya guarda su propio monto, mientras
   que los términos tienen que poder resolverse hacia atrás desde un pago
   vigente. Sin columna `weekly_quota_snapshot`: duplicaría la verdad justo
   para las lecturas que ya joinean contra el plan. Corregir un tipeo de
   cuota = desactivar y crear otro plan, que no corta cobertura (invariante
   de arriba).
6. **Un servicio con `payment_required = false` no puede tener planes de
   cuota**, por trigger. Sin pago vigente no hay de dónde resolver la cuota:
   aparentaría estar vigente y no se podría aplicar.
7. **Firma con contexto de serie: detalle de `database-agent`.** Lo no
   negociable es que siga habiendo **una sola** función de decisión
   (ADR-0018) — el preview del mostrador y el del cliente no pueden divergir.

### Resuelta al entrar en la Fase I (2026-09-22)

Si la **organización** cancela una clase, **se emite crédito igual y sin
exigir anticipación**. El cliente no perdió nada por decisión propia, y esa
cancelación ya viaja con su propio `cancellation_reason = SLOT_CANCELLED`
(ADR-0010), así que distinguirla no cuesta nada. No exigir anticipación acá
es el punto: la política de aviso existe para que el negocio pueda revender
el cupo, y cuando el que cancela es el negocio esa razón no aplica.
