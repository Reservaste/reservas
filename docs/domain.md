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
  `Customer`.
- **Profile** — identidad de usuario autenticado (persona), independiente
  de su rol en cualquier organización.
- **Customer** — un `Profile` en tanto cliente de una `Organization`
  específica. Un mismo `Profile` puede ser `Customer` de varias
  organizaciones.
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
  servicio tiene un plan suelto y varios mensuales.
- **ServiceEntitlement** — **histórico, deprecado** (ADR-0022, ADR-0024). Era
  el derecho de uso de un `Customer` sobre un `Service`, otorgado a mano. La
  tabla se conserva por la procedencia de los datos viejos y **no se consulta
  al reservar**. Lo que hoy expresa el derecho de uso es el `ServicePlan` que
  el cliente pagó. La sección "Vigencia de ServiceEntitlement y
  `requiresActivePayment`" más abajo queda como registro histórico.
- **Resource** — lo que se ocupa al prestar el servicio: sala, profesional,
  cancha, equipo. Relación `Service`↔`Resource` es **N:M** (un `Service`
  puede requerir uno o más `Resource`, y un `Resource` puede prestarse a
  varios `Service`). `ScheduleRule` referencia el/los `Resource`
  concretos que ocupa esa regla — no asumir 1:1 entre `Service` y
  `Resource` en el schema.
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
  Estados (ADR-0010): **`ACTIVE | BLOCKED | CANCELLED`** — `BLOCKED` es
  administrativo y reversible (mantenimiento, sin reservas activas),
  `CANCELLED` es terminal. `COMPLETED` **no se persiste**, es derivado de
  `startAt < now()` en la capa de lectura.
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
- `plan_kind` y `weekly_quota` son **inmutables** una vez que el plan tiene
  pagos; el precio es libre. Los términos tienen que poder resolverse hacia
  atrás desde un pago vigente; el precio sólo afecta cobros futuros y cada
  `Payment` guarda su propio monto.
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

## Pendiente de definir

- Shape exacto de campos restante de cada entidad (se termina de definir
  junto con la migración SQL en `database.md`, Phase 1).
- Marcar un `Service` puntual como no-listado públicamente (gap conocido,
  no bloqueante para el gimnasio pero relevante para rubros sensibles —
  ver `security.md`).
