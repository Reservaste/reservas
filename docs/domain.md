# Modelo de dominio

Mantenido por: `domain-architect`. Aprobado por: Orchestrator.
Estado: **Phase 0 completa** (ver ADR-0003, ADR-0008 a ADR-0015 en
`decisions.md`). Quedan abiertos únicamente los puntos marcados
"Pendiente" al final del documento — ninguno bloquea Phase 1.

## Principio rector

La plataforma es genérica. **Nunca** modelar `Gym`, `Member`, `Trainer`,
`Class` como entidades centrales — son datos de configuración de una
`Organization` particular (un gimnasio), no conceptos del dominio.

## Entidades centrales

- **Organization** — el tenant. Cualquier rubro (gimnasio, consultorio,
  cancha, salón). Todo dato de negocio cuelga de un `organizationId`.
- **OrganizationMember** — vínculo entre un `Profile` y una `Organization`
  con un rol (`OWNER`, `STAFF`; ver `security.md`). No confundir con
  `Customer`. Desde ADR-0033 lleva además `roleId`, el `OrganizationRole`
  configurable que aplica dentro de `STAFF`; **null no es "sin permisos"**,
  es "el rol por defecto de la organización". Para un `OWNER` es siempre
  null (hay un `CHECK`): un `OWNER` corta antes de mirar cualquier permiso.
- **OrganizationRole** — (ADR-0033) el rol configurable de una
  `Organization`, con **nombre libre elegido por ella** ("Profesor",
  "Recepción", "Instructor" — el producto es genérico, así que el nombre no
  puede venir de un enum nuestro) y cinco permisos booleanos:
  `canViewPayments`, `canManagePayments`, `canManageBookings`,
  `canManageCustomers`, `canManageAttendance`. Es una capa **dentro** de
  `STAFF`, no un reemplazo del enum de rol: `OrganizationMember.role` sigue
  diciendo **quién manda** (`OWNER`) y el `OrganizationRole` dice **qué
  puede hacer el que no manda**. Invariantes: exactamente un rol
  `isDefault` por organización y siempre activo; `canManagePayments` implica
  `canViewPayments` (cobrar sin poder ver lo cobrado no es un rol, es un
  bug); un rol con miembros activos no se desactiva ni se borra; un rol
  nunca cruza de organización. Los permisos son columnas booleanas y no un
  `jsonb` a propósito — ver ADR-0033 §4.2 y el bypass de autorización de
  ADR-0026/ADR-0028.
- **TeamInvitation** — (ADR-0034) la **promesa** de una membresía: un link de
  un solo uso, con 24 horas de vida, que le permite a alguien **sin cuenta en
  la plataforma** sumarse al equipo de una `Organization` con un
  `OrganizationRole` ya elegido. No es un `OrganizationMember` en otro estado:
  no ocupa lugar en el padrón ni en el tope del plan, su vínculo es un `email`
  (al emitirla no hay `Profile` al que apuntar) y su ciclo de vida es el de un
  secreto — vence, se usa una sola vez y se revoca. **Nunca puede crear un
  `OWNER`**, el canje exige que el email de la sesión coincida, y jamás le
  cambia el rol a quien ya es miembro. Detalle más abajo.
- **Profile** — identidad de usuario autenticado (persona), independiente
  de su rol en cualquier organización.
- **Customer** — un `Profile` en tanto cliente de una `Organization`
  específica. Un mismo `Profile` puede ser `Customer` de varias
  organizaciones. `source` (ADR-0047) distingue el origen del alta:
  `STAFF` (mostrador — `enrollCustomerByEmail`, `createManagedCustomer`, alta
  directa del panel; default, cubre toda fila anterior a esta ADR) o
  `SELF_SERVICE` (auto-inscripto por `bookSlot()` cuando la `Organization`
  tiene `openBookingEnabled` prendido y la cuenta autenticada no tenía
  ninguna fila de `Customer`, ni siquiera inactiva, para esa organización).
  Una baja del staff (`isActive=false`) **nunca** se reactiva por esta vía.
- **Service** — lo que una `Organization` ofrece y se puede reservar (una
  clase, un turno de cancha, una sesión, una consulta). No depende de
  gimnasio: es genérico.
- **ServicePlan** — lo que un `Customer` compra respecto de un `Service`: un
  precio, un período de cobro y un **derecho de reserva de una forma
  determinada** (`DROP_IN` | `WEEKLY_QUOTA` | `UNLIMITED`). Genérico por
  diseño: una clase suelta, un turno fijo semanal o un abono libre. Un
  `Service` tiene N planes (ADR-0024) — es la lista de precios que la
  organización ofrece a sus clientes, y no hay que confundirla con
  `plans`, que son los planes del SaaS que paga la propia organización.
  `billing_type`/`billing_cycle` viven acá, no en `Service`: un mismo
  servicio tiene un plan suelto y varios mensuales. **El período de cobro no
  tiene por qué ser un mes** (ADR-0031): `billingPeriodMonths` (null = 1) lo
  generaliza a trimestre, semestre o año, y `billingCycle` gana
  `CALENDAR_PERIOD` (bloque fijo del año, anclado en `billingAnchorMonth`,
  igual para todos los clientes del plan) y `ROLLING_PERIOD` (N meses desde
  el día de la compra). Los dos valores mensuales de siempre son el caso
  `billingPeriodMonths = 1` y no se deprecan.
- **ServiceEntitlement** — **histórico, deprecado** (ADR-0022, ADR-0024). Era
  el derecho de uso de un `Customer` sobre un `Service`, otorgado a mano. La
  tabla se conserva por la procedencia de los datos viejos y **no se consulta
  al reservar**. Lo que hoy expresa el derecho de uso es el `ServicePlan` que
  el cliente pagó. La sección "Vigencia de ServiceEntitlement y
  `requiresActivePayment`" más abajo queda como registro histórico.
- **Resource** — lo que se ocupa al prestar el servicio: sala, profesional,
  cancha, equipo. Relación `Service`↔`Resource` es **N:M** (un `Service`
  puede requerir uno o más `Resource`, y un `Resource` puede prestarse a
  varios `Service`) — pero esa relación es "cualquiera de estos puede
  prestarlo", nunca "los necesita simultáneamente": todo camino de reserva
  (grilla o dinámico) ocupa exactamente un `Resource` por `Booking`.
  `ScheduleRule` referencia el/los `Resource` concretos que ocupa esa
  regla — no asumir 1:1 entre `Service` y `Resource` en el schema.
  `isExclusive` (ADR-0044, default `false`): un `Resource` exclusivo no
  puede tener dos `SlotOccurrence` `ACTIVE` que se solapen en el tiempo
  (constraint `EXCLUDE` a nivel de base, independiente de qué `Service`
  las originó). `dynamicAvailability` (ADR-0051, default `false`, sólo
  válido si `isExclusive=true`): opt-in adicional — ese `Resource` deja de
  usar una grilla de `ScheduleRule` pre-generada y calcula su
  disponibilidad al momento de reservar, contra una `ResourceAvailabilityWindow`
  propia (ver más abajo). Nunca puede tener `ScheduleRule` activas al
  mismo tiempo (se valida en varias capas).
- **ResourceAvailabilityWindow** (ADR-0051) — la "apertura general" de un
  `Resource` con `dynamicAvailability=true` para un día de la semana (ej.
  "lunes 9 a 17"), sin `Service` atado — a diferencia de `ScheduleRule`,
  una sola fila cubre un rango horario completo, no un punto de inicio
  fijo. `get_dynamic_availability()` calcula los horarios concretos
  dentro de esta ventana al momento de reservar.
- **ScheduleRule** — regla recurrente de horario para un `Service`/
  `Resource` (día de semana, hora, duración, capacidad, vigencia
  desde/hasta).
- **ScheduleException** — excepción puntual a una `ScheduleRule`
  (cancelación o modificación de una ocurrencia específica).
- **SlotOccurrence** — una ocurrencia concreta y reservable en el tiempo
  (fecha + hora + `Resource` + `Service` + capacidad efectiva). **Se
  persiste como fila materializada** (ADR-0003), con horizonte rodante de
  **90 días**, mantenido por pg_cron + fallback lazy (ADR-0009). Una vez
  que tiene al menos una `Booking` asociada queda congelada: no se
  regenera ni se borra por cambios posteriores en la regla que la
  originó. `startAt`/`endAt` son `timestamptz`, convertidos desde
  `weekday + localStartTime + Organization.timezone` en SQL al generar,
  nunca con un offset cacheado (ADR-0014 — así DST se resuelve solo).
  `scheduleRuleId` es nullable desde ADR-0051: `null` únicamente para una
  ocurrencia creada por `hold_dynamic_slot()` sobre un `Resource` dinámico
  (nunca proviene de una `ScheduleRule`); toda ocurrencia generada por el
  cron de grilla lo sigue teniendo poblado como siempre — es la forma en
  que el resto del sistema distingue "viene de grilla" de "viene de
  disponibilidad dinámica".
  Estados (ADR-0010, `HELD` agregado en ADR-0051): **`ACTIVE | BLOCKED |
  CANCELLED | HELD`** — `BLOCKED` es administrativo y reversible
  (mantenimiento, sin reservas activas), `CANCELLED` es terminal.
  `HELD` es una reserva temporal de 5 minutos (`heldUntil`/`heldBy`),
  exclusiva de `hold_dynamic_slot()`/`book_dynamic_slot()`: ocupa el
  `Resource` exclusivo igual que `ACTIVE` (mismo constraint `EXCLUDE`),
  pero nunca tiene una `Booking` real todavía. Se promueve a `ACTIVE` al
  confirmar, o vuelve a `CANCELLED` si vence sin confirmar o si la
  confirmación falla — nunca queda huérfana. `COMPLETED` **no se
  persiste**, es derivado de `startAt < now()` en la capa de lectura.
- **Booking** — una reserva de un `Customer` sobre un `SlotOccurrence`.
  Independiente del medio de pago.
- **RecurringBooking** — una serie de `Booking` generadas a partir de un
  patrón recurrente, con soporte de excepciones individuales (cancelar una
  fecha sin afectar el resto de la serie). Campos (ADR-0010):
  `id, organizationId, customerId, scheduleRuleId` (FK a la regla que
  sigue — **nunca duplicar** día/hora acá, es una segunda fuente de
  verdad que puede divergir), `status: ACTIVE|CANCELLED`, `startDate`,
  `endDate?`, `createdAt/updatedAt/createdBy`,
  `cancelledAt?/cancelledBy?/cancellationReason?`. Cada `Booking`
  generada por la serie referencia tanto `recurringBookingId` como
  `slotOccurrenceId`. `RecurringBooking.status` permanece `ACTIVE` aunque
  **todas** sus `Booking` actuales estén canceladas individualmente —
  expresa la suscripción del cliente, no un agregado de sus hijos; el job
  de horizonte rodante sigue generando `Booking` nuevas mientras esté
  `ACTIVE` y dentro de `[startDate, endDate]`.
- **MakeupCredit** — ADR-0025. El derecho a recuperar, dentro de una
  ventana, un turno que un `Customer` liberó a tiempo (o que la
  `Organization` canceló). Opt-in por `Organization`
  (`makeupCreditsEnabled`, default `false`). **Se emite** al cancelar una
  fecha puntual de una serie dentro de cuota con la anticipación
  configurada (`releaseDeadlineHours`), o cuando cancela la organización
  (sin exigir anticipación ahí — el cliente no perdió nada por decisión
  propia). **Nunca se emite desde la cascada de una serie completa**: es
  el invariante que impide fabricar créditos infinitos cancelando y
  recreando la misma serie. `expiresOn` se calcula y se congela al
  emitir, ancla a la fecha de la ocurrencia liberada — nunca se recalcula
  si la política cambia después. Se consume como **última compuerta**
  dentro de `evaluate_customer_booking()`, después de toda la cadena de
  cobertura de ADR-0024: nunca gasta un crédito si otra cobertura ya
  alcanzaba. Bajo lock propio, distinto del `FOR UPDATE` de `book_slot()`
  (ADR-0004) — ese lock serializa una `SlotOccurrence`, no dos ocurrencias
  distintas que el mismo `Customer` reserve a la vez, que es justo donde
  un crédito se gastaría dos veces sin su propio candado.
- **PlanChangeRequest** — Fase 28 (feedback de producción). El pedido de un
  `Customer` para pasarse a otro `ServicePlan`. **No es el cambio**: el
  cambio real sigue siendo VOID + recargar del mostrador (ADR-0024
  resolución 1), porque es una operación de dinero y todavía no hay cobro
  online (ADR-0027, bloqueada). Por eso **ninguna función de decisión de
  reserva la lee**: un pedido pendiente no habilita ni bloquea una sola
  reserva — es un hecho accionable para el negocio, no una cobertura.
  Guarda el plan pedido y el plan **vigente resuelto en la base** al
  momento de pedir (nunca enviado por el caller), que es lo que la
  convierte en "upgrade" o "downgrade" a ojos del mostrador. Un solo
  pedido pendiente por `(Customer, ServicePlan)`. Se cierra con
  `APPLIED`/`DISMISSED`, y registrar el pago del plan pedido la cierra
  sola. Genérica por construcción: es "quiero otro plan", no nada de
  gimnasio.
- **Payment** — el movimiento/estado económico real de un `Customer`
  (pagó, pendiente, vencido). Su **ancla de registro es `servicePlanId`**
  (ADR-0024): es lo que hace contestable no sólo "¿de qué servicio es este
  pago?" sino "¿qué compró?". `serviceId` se mantiene como columna
  **derivada y verificada por trigger** (no segunda fuente de verdad) porque
  es el grano correcto de "quién pagó este mes" y lo usan el `EXCLUDE`, sus
  índices y varias funciones de lectura. `slotOccurrenceId` es no nulo
  exactamente para los pagos de un plan `DROP_IN`: una clase suelta se paga
  por clase, no por período. Estados
  `PAID|PENDING|OVERDUE|VOID`; `VOID` (nunca `DELETE`) para un pago cargado
  por error, mismo criterio que una `Booking` cancelada. Gaps entre períodos
  son estado válido y esperado (el cliente no pagó ese tramo); solapamientos
  de `PAID` sobre el mismo `(customerId, serviceId)` se previenen por
  constraint, y los pagos sueltos por un índice único sobre
  `(customerId, slotOccurrenceId)` — sin ese índice quedarían sin ninguna
  protección de doble cobro al sacarlos del `EXCLUDE`.

### Vigencia de ServiceEntitlement y `requiresActivePayment` — HISTÓRICO

> Esta sección describe el modelo anterior a ADR-0022/ADR-0024. Se conserva
> porque explica de dónde vienen los datos viejos. **Nada de acá se consulta
> al reservar**: `requiresActivePayment` fue reemplazado por
> `services.payment_required`, y el derecho de uso lo expresa hoy el
> `ServicePlan` pagado. Los bonos (`CREDITS`) se eliminaron por decisión
> explícita del usuario.

`ServiceEntitlement` necesita un mecanismo explícito de vigencia — no
alcanza con "activo/inactivo". Debe soportar al menos dos formas (no
mutuamente excluyentes):

- **Por tiempo**: `validFrom`/`validUntil` (ej. membresía mensual).
- **Por uso**: contador de créditos/usos restantes (ej. "bono de 10
  clases"), que se descuenta al confirmar una `Booking` y se repone al
  cancelarla dentro de la política de la organización.

Un `ServiceEntitlement` no es necesariamente 1:1 con un `Service`: puede
cubrir varios servicios a la vez (ej. "pase full") o provenir de un
paquete de créditos compartido entre servicios. Modelar `serviceId` como
nullable + una relación N:M a servicios elegibles cuando aplique, en vez
de asumir 1:1 forzado.

**`requiresActivePayment: boolean`** (ADR-0013) vive en
**`ServiceEntitlement`, no en `Service`** — decisión del Orchestrator
resolviendo un desacuerdo entre `domain-architect` y
`payments-entitlements-agent` en el cierre de Phase 0. Motivo: el mismo
`Service` puede tener entitlements que requieren pago y otros de
cortesía/beca (ya listado como caso válido abajo) — un flag a nivel
`Service` no puede expresar esa variación por cliente. Semántica: si
`false`, `canCustomerBook()` no chequea `Payment` en absoluto (el
entitlement `ACTIVE` alcanza); si `true` y el entitlement es por tiempo,
exige un `Payment PAID` cuyo período cubra `SlotOccurrence.startAt` (ver
ADR-0013 para la definición exacta de "pago válido" — se valida contra la
fecha del turno, nunca contra la fecha en que se hace la reserva); si
`true` y el entitlement es por créditos, el chequeo de `Payment` se omite
igual (el crédito ya implica que el paquete se pagó al comprarse).

## Invariantes de negocio (no negociables)

```
availableCapacity = maxCapacity - activeBookings
```

- `availableCapacity` se **calcula**, nunca se persiste como fuente de
  verdad manual.

Invariantes de `ServicePlan` y cuota (ADR-0024):

- **"Veces por semana" es "cantidad de series", no un contador.** Cada
  `ScheduleRule` es semanal y una `RecurringBooking` es la suscripción a una
  regla, así que *k* series producen exactamente *k* reservas por semana. No
  se persiste ningún contador de usos: nada que decrementar, restituir ni
  reconciliar.
- **La cuota se mide contra las series en vigencia para la fecha local del
  slot**, nunca contra `status = 'ACTIVE'` a secas (una serie con `end_date`
  pasada sigue `ACTIVE` por ADR-0010) y nunca contra `now()`. Por eso no hay
  ninguna "ventana semanal" que anclar a semana calendario o corrida: no se
  calcula.
- Cuando hay más series en vigencia que cuota, el desempate es
  determinístico (`created_at`, `id`): entran las primeras que el cliente
  contrató.
- Una reserva **extra** (la que no viene de una serie dentro de cuota) **no
  la cubre** el pago por período de un plan de cuota: se paga suelta o se usa
  un crédito de recupero.
- **Desactivar un plan no invalida un pago ya hecho.** La cobertura se lee
  sin filtrar por `is_active`, o el dueño le cortaría el acceso a todos el día
  que ordena su lista de precios.
- `plan_kind`, `weekly_quota` y —desde ADR-0031— `billing_period_months` y
  `billing_anchor_month` son **inmutables** una vez que el plan tiene pagos;
  el precio es libre. Los términos tienen que poder resolverse hacia atrás
  desde un pago vigente; el precio sólo afecta cobros futuros y cada
  `Payment` guarda su propio monto.

Invariantes del período de cobro largo (ADR-0031):

- **El período no se recorta nunca: lo que se prorratea es el precio.** Quien
  entra el 15 de septiembre en un plan trimestral jul-sep compra el trimestre
  jul-sep, y su cobertura empieza el 1 de julio. Recortar el inicio
  desalinearía la renovación de todos los clientes del plan, que es el único
  motivo para elegir un ciclo calendario.
- **El prorrateo se cuenta en meses enteros**, con el mes de alta como
  completo (1 de 3, 2 de 3), redondeado a la unidad entera de moneda. Nunca
  en días: el mismo plan no puede costar distinto según cuántos días tenga
  el mes.
- **Un ciclo rodante no prorratea jamás**: arranca el día de la compra, así
  que no hay fracción que descontar.
- **El prorrateo es una cotización, no un cobro.** `quote_service_plan_period()`
  sugiere un monto; `Payment.amount` sigue libre y el mostrador puede
  ignorarlo. Nada en la base aplica un descuento por su cuenta.
- **El prorrateo aplica sólo al alta en un ciclo largo, nunca al cambio de
  plan a mitad de período**: eso sigue siendo anular y recargar el período
  completo (ADR-0024). No hay nota de crédito por lo no consumido de un plan
  anterior — no existe esa entidad.
- En un plan de ciclo largo, el crédito de recupero con vencimiento "fin del
  período de facturación" vence igual **a fin de mes**: un crédito vivo tres
  meses triplica el riesgo que ADR-0025 dejó anotado.
- Un `Service` con `payment_required = false` no puede tener planes de cuota:
  sin pago vigente no hay de dónde resolver la cuota.
- `Booking.status = CANCELLED` no consume cupo. `Booking.status = CONFIRMED`
  sí.
- No puede existir más de una `Booking` activa del mismo `Customer` para el
  mismo `SlotOccurrence` (constraint de unicidad, no solo validación de
  aplicación).
- Nunca permitir que la capacidad configurada de una ocurrencia quede por
  debajo de las reservas activas que ya tiene (una reducción de capacidad
  se valida contra `activeBookings` antes de aplicarse).
- Cancelar una `Booking`: cambia a `CANCELLED`, conserva historial, guarda
  `cancelledAt` y `cancelledBy`, libera el cupo inmediatamente. **Nunca se
  borra** una `Booking` cancelada.
- Una `RecurringBooking` puede tener excepciones: cancelar una fecha
  puntual no cancela el resto de la serie. Cambiar "desde esta fecha en
  adelante" es una operación distinta de cancelar una única ocurrencia.
- `Booking.status` incluye `NOT_GENERATED` — **terminal** (ADR-0010),
  para una fecha de una serie recurrente que nunca llegó a confirmarse
  por falta de cupo (ya sea anunciado en el preview de ADR-0012, o
  sobrevenido si el slot se llenó antes de generarse). No se reintenta
  automáticamente si se libera cupo después — evita una cola de
  reintentos asíncronos no pedida.
- `Booking` cancelada (ADR-0010) lleva dos campos ortogonales:
  **`cancelledBy`** (QUIÉN: **FK a `Profile`** — el perfil concreto que
  ejecutó la cancelación, no un enum categórico; corregido acá el
  2026-09-22, el schema real lo tiene así desde Phase 5 y esta sección
  quedó desactualizada hasta ahora) + **`cancellationReason`** (POR QUÉ:
  `CUSTOMER_REQUEST | SLOT_CANCELLED | RULE_DISCONTINUED |
  SERIES_CANCELLED` — este último desde ADR-0025/ADR-0028: una serie
  cancelada por el cliente ya no se confunde con una regla discontinuada
  por la organización). El actor **categórico** (cliente vs. organización)
  se lee de `cancellationReason`, no de `cancelledBy`: por eso separar los
  campos permite filtrar por categoría sin parsear el motivo, y agregar
  motivos nuevos sin tocar `cancelledBy`. Sin esta distinción no hay forma
  de notificar correctamente al cliente ni de reportar con precisión.
- Cancelar **una sola `SlotOccurrence`** puntual (staff, sin tocar la
  regla) cancela sus `Booking` asociadas con `cancellationReason =
  SLOT_CANCELLED`; `RecurringBooking` (si las hay) **no** se toca, sigue
  `ACTIVE` — la regla y la serie siguen vivas, solo esa fecha se perdió.
- Desactivar/cancelar un `ScheduleRule` completo con `Booking`/
  `RecurringBooking` activas **nunca** las deja huérfanas ni las cancela
  en silencio: dispara una cancelación explícita en cascada
  (`cancellationReason = RULE_DISCONTINUED`, `cancelledBy = ORGANIZATION`)
  que toca **tanto** las `Booking` futuras **como** las `RecurringBooking`
  que seguían esa regla (`RecurringBooking.status → CANCELLED` también —
  de lo contrario la serie queda `ACTIVE` apuntando a una regla
  inexistente). Esto es distinto de la regla de "nunca reducir capacidad
  por debajo de reservas activas" — discontinuar un horario completo debe
  poder hacerse aunque tenga reservas, siempre que la cascada sea
  explícita y notificada.

## Pago ≠ permiso

ADR-0024 **refuerza** esta separación, no la debilita: un pago ahora compra un
derecho **de una forma específica** (una clase, N horarios fijos, o libre), así
que "pagó" y "puede reservar *esto*" se separan todavía más que antes. Alguien
con el mes pago puede tener perfectamente prohibido reservar un turno concreto
porque está fuera de su frecuencia — y ese caso necesita su propio mensaje
(`OUTSIDE_PLAN_QUOTA`), nunca "tenés que pagar".

Casos históricos donde entitlement y pago no coincidían 1 a 1 (el mecanismo
cambió; la razón por la que hay que mantenerlos separados, no):

- Servicio regalado / beca / cortesía (entitlement activo, sin `Payment`).
- Admin habilita manualmente un entitlement.
- Pago externo registrado sin gatillar automáticamente el entitlement.
- Pago pendiente o vencido con entitlement todavía técnicamente activo
  hasta que se resuelva.
- Servicio pausado (entitlement existe pero no habilita reserva).

La función que decide si alguien puede reservar (`canCustomerBook()`) es **la
única fuente de verdad** para esta lógica — no se duplica en cada endpoint, y
el preview del mostrador, el del cliente y el confirm la comparten (ADR-0018:
duplicarla es cómo se llega a "el preview decía que sí y el confirm dijo que
no"). Evalúa, en este orden: ocurrencia activa y no terminada, organization
activa, service activo, customer activo, **cobertura** (pago suelto de esa
clase → pago por período en vigencia para la fecha del slot → y si el plan es
de cuota, que la reserva venga de una serie dentro de la frecuencia), cupo
disponible, ausencia de duplicado.

Recibe **contexto de serie** (ADR-0024) en tres formas: sin serie (reserva
puntual), serie existente, y **serie prospectiva** — esta última porque el
preview de una reserva fija corre antes de que la serie exista, y sin ella el
preview le diría "fuera de tu frecuencia" en todas las fechas justo al cliente
que tiene el plan que la permite.

**Reserva abierta (ADR-0047).** Cuando no hay `Customer` activo,
`canCustomerBook()` devuelve `NOT_A_CUSTOMER` salvo que la `Organization`
tenga `openBookingEnabled` prendido (opt-in, default apagado) **y** no exista
absolutamente ninguna fila de `Customer` para ese par (organización, perfil) —
ni siquiera inactiva; en ese único caso devuelve `OK_OPEN_BOOKING` en vez de
`NOT_A_CUSTOMER`, para que la UI pública ofrezca "Reservar" en lugar de "pedile
al negocio que te habilite". `canCustomerBook()` nunca crea nada — es
puramente de lectura, invocada también por `previewRecurringBooking()` y
`canCustomerBookDetail()`, que no pueden tener el side-effect de dar de alta un
`Customer` real solo por mostrar una vista previa. El alta real vive
exclusivamente dentro de `bookSlot()`: si corresponde, inserta el `Customer`
(`source='SELF_SERVICE'`) **en la misma transacción** que la reserva, detrás de
un rate limit (altas por perfil en 24h, altas por organización por hora); si la
reserva falla después (sin cupo, sin cobertura), el rollback deshace también el
alta — nunca queda un `Customer` huérfano sin `Booking`.

## Auditoría por entidad (ADR-0010)

Criterio: `cancelledAt/cancelledBy/cancellationReason` solo en entidades
con lifecycle real de baja donde importa quién/por qué (dinero, acceso, o
cascada de terceros); `createdBy` solo cuando hay un actor humano
identificable.

| Entidad | created/updatedAt | createdBy | cancelledAt/By/Reason |
|---|---|---|---|
| Organization | Sí | Sí | No |
| OrganizationMember | Sí | Sí | Sí |
| Profile | Sí | No (autocreado) | No |
| Customer | Sí | Sí | Sí |
| Service | Sí | Sí | Sí |
| ServiceEntitlement | Sí | Sí | Sí |
| Resource | Sí | Sí | Sí |
| ScheduleRule | Sí | Sí | Sí |
| ScheduleException | Sí | Sí | No (la excepción ya es el registro de la desviación) |
| SlotOccurrence | Sí | No (generada por job) | Sí |
| Booking | Sí | Sí (distinguir self-booking vs. STAFF en su nombre) | Sí |
| RecurringBooking | Sí | Sí | Sí |
| Payment | Sí | Sí | No — usa `status = VOID` en su lugar, no la tripleta genérica |

`updatedAt` se mantiene con trigger `BEFORE UPDATE` de Postgres (no
convención de aplicación) — hay múltiples caminos de escritura (server
actions, RPCs, SQL editor) y un trigger es el único punto que garantiza
el invariante sin depender de que cada uno lo recuerde.

## `AuditLog` — el hecho, no el estado (ADR-0032)

La tabla de arriba registra **estados**: una `Booking` cancelada sabe
quién la canceló, un `Payment` anulado sabe que está `VOID`. Lo que no
había era el **hecho**: quién anuló ese pago, quién anotó a alguien en una
clase que no pidió, desde cuándo y por obra de quién un plan pasó de 2500
a 3400 (el precio de un `ServicePlan` es libre por ADR-0024 resolución 5
y no dejaba ningún rastro).

`AuditLog` es una entidad **transversal y de solo lectura**: no participa
de ninguna decisión de reserva, de cobertura ni de cupo, y ninguna función
del motor la lee. Registra ocho acciones y nada más (`AuditAction` en
`@reservaste/domain`): alta y cambio de estado de `Payment` (incluida la
anulación), alta y cancelación de `Booking` **cuando el actor no es el
propio cliente**, alta / cambio de precio-nombre-orden / activación de
`ServicePlan`, y cambio de suscripción SaaS de la `Organization`.

| Campo | Sentido |
|---|---|
| `organizationId` | Nullable en el schema; hoy siempre presente (los ocho casos tienen organización). |
| `actorId` | `auth.uid()` del momento. `null` = lo hizo el sistema (job/cron), que es información: un uuid sintético de "sistema" sería una mentira con forma de dato. |
| `action` | Enum cerrado de 8 valores, nunca texto libre. |
| `targetTable` + `targetId` | A qué fila se refiere. **Sin FK a propósito**: el log tiene que sobrevivir al borrado de lo que audita. |
| `metadata` | El diff mínimo (`{"price":{"from":2500,"to":3400}}`). **Nunca la fila entera ni datos personales** — esos viven en sus tablas con su RLS y copiarlos acá los saca de ese control. |
| `createdAt` | Cuándo. No hay `updatedAt` ni `cancelledAt`: una fila de auditoría es inmutable por definición. |

Invariantes propios:

- **Una fila de auditoría nunca se edita ni se borra** (ni por
  `service_role`). Es la mitad del valor: un log que el auditado puede
  editar no es un log.
- **El hecho no depende de que el llamador lo recuerde**: lo escribe un
  trigger, porque la mitad de las escrituras del alcance no pasa por
  ninguna RPC (ver `database.md`, Fase 30).
- **"Lo hizo el propio cliente" no es un hecho de auditoría.** La
  comparación es `profileId is distinct from auth.uid()` — con un cliente
  gestionado (ADR-0026) `profileId` es `null` y un `<>` haría desaparecer
  del log justo la reserva que sí hizo el mostrador.
- **Lo que ya tiene su propio registro no se duplica**: los créditos de
  recupero son su propio log (`makeup_credits`, ADR-0025) y no entran acá.

## `OrganizationRole` — qué puede el que no manda (ADR-0033)

El pedido original fue textual: *"como se lo van a pasar a profesores no
deberían ver el tema pagos, configurable por organización, roles y nombre
del rol"*. Antes de ADR-0033 el rol era binario y **darle acceso al panel a
un profesor era darle acceso a la cobranza completa** — no había forma de
no hacerlo salvo no darle acceso.

Reglas de resolución, en este orden y sin excepciones:

```
OWNER                      -> todo, sin mirar roleId
STAFF con roleId           -> ese rol
STAFF con roleId null      -> el rol isDefault de la organización
miembro inactivo / ajeno   -> nada
```

- **`OWNER` no es configurable, y no es una concesión a la compatibilidad.**
  Es la raíz de confianza: si el rol de quien administra roles fuera
  administrable, existiría el ciclo "me edito el rol para poder editar
  roles". El producto ya había escrito esta idea en `revoke_member()`
  (*"an organization with no active OWNER is unadministrable"*); esto es la
  misma idea aplicada a los permisos.
- **Administrar roles tampoco es un permiso configurable.** Si lo fuera,
  existiría un rol capaz de ampliarse a sí mismo y los cinco permisos no
  significarían nada.
- **Los cinco permisos del primer corte son un conjunto cerrado**:
  `VIEW_PAYMENTS`, `MANAGE_PAYMENTS`, `MANAGE_BOOKINGS`,
  `MANAGE_CUSTOMERS`, `MANAGE_ATTENDANCE`. Lo que no está ahí **no** es
  configurable: ver el calendario, la agenda, el padrón de clientes y los
  horarios es de todo miembro activo (un rol que no ve a los clientes es un
  rol que no puede pasar lista), y configuración, branding, invitar equipo,
  planes/precios, créditos manuales y merge/unlink siguen siendo OWNER-only.
- **"No ver pagos" no es "no ver precios".** La lista de precios de los
  planes activos es pública (la ve cualquier visitante del calendario);
  `VIEW_PAYMENTS` oculta *quién pagó, cuánto, cuándo y quién debe*.
- **"No ver pagos" tampoco es "no ver que algo depende de un pago"**
  (ADR-0033 resolución 2): un rol sin `VIEW_PAYMENTS` sigue viendo
  `upcoming_unpaid` y `PAYMENT_REQUIRED` en las pantallas operativas. Es
  información de pago, aunque no sea un monto, y se acepta documentada: sin
  ella el rol no entendería por qué no puede anotar a alguien.

Un permiso **nunca** se decide leyendo `roleId` ni un booleano suelto en
TypeScript: se decide en SQL (`has_org_permission()`, las policies y cada
RPC `security definer`). Lo que el frontend hace con los permisos es
esconder y deshabilitar — *hiding the button is not enforcement*.

## `TeamInvitation` — la promesa de una membresía, no una membresía (ADR-0034)

El pedido original fue textual: *"Para invitar nuevos perfiles necesariamente
debe tener el registro. Deberíamos agregar la misma funcionalidad de nuevo
usuario sin registro (...) o un link de activación vinculado a ese email para
que termine el registro"*.

`TeamInvitation` es **una entidad nueva, no un `OrganizationMember` en otro
estado.** Esa distinción es la decisión de modelado, no un detalle de
implementación:

- Una invitación pendiente **no ocupa un lugar en el padrón**. Si se modelara
  como un miembro inactivo, `organization_team()` mostraría gente que no
  existe y el tope de `maxTeamMembers` del plan contaría a quien nunca entró
  (o habría que excluir inactivos, y entonces el límite no limita).
- Su ciclo de vida es el de un **secreto**, no el de una persona: nace con un
  token, vence a las 24 horas, se usa **una sola vez** y se puede revocar. Un
  `OrganizationMember` no tiene ninguna de esas propiedades.
- Su destino todavía **no tiene identidad**. Por eso el vínculo es un `email` y
  no un `profileId`: al emitirla no hay `Profile` al que apuntar.

```
TeamInvitation --(canje)--> OrganizationMember
       email                    profileId
    24 h, un uso              permanente
```

Reglas del dominio, todas verificadas en la base y no por convención:

- **Una invitación nunca crea un `OWNER`.** Promover a dueño sigue siendo un
  acto deliberado sobre un miembro que ya existe y ya se autenticó, nunca la
  consecuencia de que alguien tenga un link. Es un `CHECK`, no una regla que la
  RPC deba recordar.
- **El rol lo elige quien invita, al invitar** (`roleId`, nullable = el rol
  `isDefault` de la organización, igual que `OrganizationMember.roleId`).
  Asignar rol es `OWNER`-only y el `OWNER` está presente al invitar, no cuando
  la persona hace click; si se decidiera después habría una ventana donde el
  miembro ya entró con *algún* rol — o ninguno (no puede trabajar) o el default
  (que puede ser más de lo que el dueño quería). Y así la pantalla puede decir
  "invitado como **Profesor** — pendiente desde el 12/03" en vez de "alguien va
  a entrar y después vemos".
- **El canje exige que el email de la sesión coincida con el de la
  invitación**, sin excepción. El `phone` es **canal** (el mensaje de WhatsApp),
  el `email` es **vínculo**: son cosas distintas a propósito. Eso convierte un
  secreto de un factor en uno de dos y mitiga el caso de falla más probable, el
  número mal tipeado. Si la persona se registró con otro email (típico con
  Google), el canje falla con un error claro y el dueño reemite — fricción
  visible y arreglable, no un acceso silencioso al tenant equivocado.
- **El canje nunca le cambia el rol a quien ya es miembro.** Un link que
  *asciende* a alguien es escalada de privilegios; uno que lo *degrada* es un
  ataque de denegación con forma de invitación. `inviteMemberByEmail` sí puede
  cambiar el rol, y ahí es correcto: es sincrónico y lo ejecuta el `OWNER` en
  ese momento.
- **Un solo link vivo por (organización, email).** Reenviar **revoca** el
  anterior: un link viejo es justo el que pudo haber ido al lugar equivocado.
- **Estado derivado, no columna**: `PENDING | REDEEMED | REVOKED | EXPIRED` se
  calcula de `redeemedAt`/`revokedAt`/`expiresAt`. Una invitación vencida sigue
  siendo "nadie la usó" — es justo la que hay que reenviar, no una que haya que
  esconder.
- **Link, no contraseña temporal.** Una contraseña obligaría a crear la cuenta
  nosotros con una contraseña que conocemos (Admin API con la `service_role`
  key, hoy fuera de todo camino de request), seguiría siendo la contraseña
  después de la ventana de 24 h, quedaría para siempre en el historial del chat
  de WhatsApp, y no serviría para quien entra con Google. Un link es de un solo
  uso por construcción.

El mecanismo es **separado** del de activación de clientes (ADR-0026) aunque se
parezca: el de cliente otorga *"sos este cliente"* (acceso a lo propio), el de
equipo otorga acceso a **los datos de otras personas** en todo el tenant. Mismo
patrón, radio de explosión mucho mayor — y por eso TTL más corto (24 h contra
72 h), rate limit más chico y `OWNER` en vez de `STAFF` para emitir. Lo único
que comparten es el acuñado del token, que es la parte que **no debe**
divergir.


## Pendiente de definir

- Shape exacto de campos restante de cada entidad (se termina de definir
  junto con la migración SQL en `database.md`, Phase 1).
- Marcar un `Service` puntual como no-listado públicamente (gap conocido,
  no bloqueante para el gimnasio pero relevante para rubros sensibles —
  ver `security.md`).
