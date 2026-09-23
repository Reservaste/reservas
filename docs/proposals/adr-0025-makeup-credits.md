# ADR-0025 — Crédito de recupero: liberar un cupo y recuperarlo dentro del mes

Fecha: 2026-09-22
Estado: **Propuesta** (no aplicada — requiere aprobación del Orchestrator)
Propuesta por: `domain-architect`
Origen: feedback del primer cliente real (estudio de pilates), puntos 8, 9, 10
y la mitad del 11 de `docs/iteration-3-plan.md`
Depende de: **ADR-0024** (qué es una reserva *extra*), ADR-0019 (una fecha no
confirmada es pendiente), ADR-0018 (una sola función de decisión), ADR-0010
(taxonomía de cancelación, `COMPLETED` derivado), ADR-0008 (disclosure
pública), ADR-0004 (concurrencia), ADR-0003/0009 (pg_cron)
Contesta: la pregunta que ADR-0019 dejó abierta ("quién se queda con un cupo
liberado")
Habilita: ADR-0027 (tarjeta) — el camino de cobro de un turno suelto nace acá

---

## 0. Cinco cosas del encuadre que están mal, y una que falta

El encargo pedía explícitamente que marque lo que esté mal. Hay cinco puntos,
todos verificados contra el schema real, y **dos de ellos son agujeros de
producto, no detalles de implementación**. Van primero porque tres de las
decisiones "ya cerradas" cambian de forma por culpa de ellos.

### 0.1 «Se consume bajo el mismo lock que el cupo, así dos reservas simultáneas no gastan el mismo crédito» — **es falso**

El `FOR UPDATE` de ADR-0004 lockea **la fila de la ocurrencia**. Serializa a
dos personas que pelean por el último lugar *de esa clase*. No serializa nada
para **un mismo cliente reservando dos ocurrencias distintas al mismo tiempo**,
que es exactamente el caso que gastaría dos veces el mismo crédito: son dos
filas de `slot_occurrences` distintas, dos locks distintos, cero conflicto.

El crédito necesita **su propio lock de fila** (o un `UPDATE ... WHERE id = X
AND status = 'AVAILABLE'` cuyo conteo de filas afectadas se verifica). Es
barato, pero hay que escribirlo, y sobre todo hay que escribir bien el test:

> El test "dos reservas simultáneas con un solo crédito ⇒ gana una" que la
> Fase I declara obligatorio, **si se escribe sobre la misma ocurrencia pasa
> por el motivo equivocado** — lo rechaza `ALREADY_BOOKED` o la capacidad, y
> el crédito ni se mira. Tiene que ser **sobre dos ocurrencias distintas del
> mismo servicio, con capacidad libre en ambas**. Como está enunciado hoy, el
> test verde no prueba nada.

### 0.2 `cancel_booking()` deja que el cliente elija el motivo de su propia cancelación — con créditos eso es una vulnerabilidad

Firma actual (`phase15`, y desde `phase5`):

```sql
cancel_booking(p_booking_id uuid, p_reason booking_cancellation_reason default 'CUSTOMER_REQUEST')
-- autorización: v_customer.profile_id = auth.uid() OR is_organization_member(...)
-- el motivo se guarda tal cual viene del caller
grant execute ... to authenticated;
```

El cliente llama esa RPC por `supabase.rpc()` desde el navegador. Hoy que un
cliente estampe `SLOT_CANCELLED` en su propia cancelación es un dato sucio en
un reporte. **Con ADR-0025 aprobado como está escrito, es un crédito gratis**:
"si cancela la organización se emite igual y sin exigir anticipación". Cancelo
diez minutos antes de la clase pasando `'SLOT_CANCELLED'` y me llevo el
crédito que la política de aviso existía para negarme.

**El motivo no puede ser input del actor: tiene que derivarse de quién es.** Si
`auth.uid()` es el `profile_id` del `Customer` dueño de la reserva, el motivo
es `CUSTOMER_REQUEST` y punto; sólo un miembro de la organización puede pasar
`SLOT_CANCELLED` / `RULE_DISCONTINUED`. Es un cambio de contrato de una RPC
existente → decide el Orchestrator (§5.1).

Nota lateral, del mismo peso conceptual: `bookings.cancelled_by` **no es** el
enum `CUSTOMER | ORGANIZATION` que dice `domain.md` — es una FK a `profiles`.
La propia migración de Phase 5 documenta esa divergencia y la justifica. O sea
que **el único discriminador de actor disponible es `cancellation_reason`**, y
por eso el punto anterior es estructural y no cosmético. De paso: "el mostrador
cancela en nombre del cliente" sale gratis — es `CUSTOMER_REQUEST` con
`cancelled_by` = el profile del staff. No hace falta nada nuevo.

### 0.3 «Se emite cuando una Booking que viene de una serie activa se cancela con anticipación» — así escrito es una máquina de imprimir créditos

`cancel_recurring_booking()` cascadea a las `Booking` futuras
`CONFIRMED` con `cancellation_reason = 'CUSTOMER_REQUEST'` cuando el que
cancela es el propio cliente. Cada una de esas fechas cumple literalmente la
regla de emisión: viene de una serie, está dentro de cuota, y se cancela con
semanas de anticipación.

El exploit es completo y no requiere nada raro:

1. Cliente con `WEEKLY_QUOTA 1`, serie de los lunes, 4 lunes confirmados.
2. Día 1 del mes cancela **la serie** → 4 créditos.
3. Vuelve a crear la serie sobre la misma regla: la anterior quedó
   `CANCELLED`, así que `customer_standing_series_on_rule()` no la ve y
   `customer_series_in_force_count()` da 0. Se acepta.
4. Se le regeneran los 4 lunes **y además tiene 4 créditos**.
5. Repetir.

Resultado: 1×semana pagado, clases ilimitadas. Es exactamente lo que ADR-0024
vino a impedir, entrando por la puerta de ADR-0025.

Se cierra con una regla que además es más simple que la del plan:

> **La emisión ocurre sólo en la cancelación de UNA fecha puntual
> (`cancel_booking`) y en las cancelaciones que origina la organización sobre
> la ocurrencia o la regla. Una cancelación en cascada de una serie no emite
> nunca, la pida quien la pida.**

Y destapa una impureza de la taxonomía de ADR-0010 que conviene arreglar
ahora: `cancel_recurring_booking()` ejecutado por el staff estampa
`RULE_DISCONTINUED` en las hijas, **pero la regla no se discontinuó** — se dio
de baja la suscripción de un cliente. Falta un valor `SERIES_CANCELLED`
(§5.2). No es imprescindible para cerrar el agujero (la regla de arriba se
apoya en el camino de llamada, no en el motivo), pero sin él el motivo miente
y el próximo que lea la tabla para decidir algo se va a equivocar igual que me
iba a equivocar yo.

### 0.4 «Motivos nuevos de `can_book_reason`: distinguir "te falta pagar esta clase suelta" de "estás fuera de tu frecuencia"» — **eso ya lo hizo ADR-0024**

`PAYMENT_REQUIRED` ("ningún pago cubre esa fecha") y `OUTSIDE_PLAN_QUOTA`
("el mes está pago, pero esto no viene de tus horarios fijos") ya son dos
valores distintos, con dos remedios distintos, y ya están implementados y con
etiqueta en castellano desde la Fase H. **ADR-0025 no necesita ningún motivo
de rechazo nuevo** y agregarlos sería empeorar (§2.4.4).

Lo que sí necesita es lo contrario y no está en el plan: **un veredicto
positivo que diga con qué se entra**. "Sí" no alcanza: el preview tiene que
poder decir *"lo tomás con tu crédito, que vence el 30/09"* y `book_slot()`
tiene que saber si consumir o no. Y ese qualifier **no puede ser un valor más
del enum**: todo el código compara `= 'OK'` / `<> 'OK'` en seis funciones, así
que un `'OK_WITH_MAKEUP_CREDIT'` sería un "sí" que la mitad del motor lee como
"no", en silencio. Tiene que romper en tiempo de compilación, no en
producción (§2.4.3).

### 0.5 Falta un interruptor: aprobado como está, **todas** las organizaciones empiezan a emitir créditos el día del deploy

El plan define `release_deadline_hours` y `makeup_credit_expiry` con defaults,
pero ninguna columna que diga si la organización **quiere** esto. Con defaults
de 12 h / fin de mes, cualquier tenant existente que cancele una reserva
empieza a acumular créditos que nadie le explicó a su dueño, y la regla de
cuota que acaba de comprarse con ADR-0024 se le afloja sola.

Va contra el patrón que este proyecto ya usó dos veces bien (ADR-0022 y el
`UNLIMITED` de ADR-0024): **nadie cambia de comportamiento el día del deploy**.

Propongo `makeup_credits_enabled boolean not null default false`, opt-in
explícito del dueño, con override por servicio. Los defaults de 12 h y
`END_OF_MONTH` siguen siendo los del usuario — aplican **una vez encendido**.

---

## 1. El problema

Textual del cliente:

> "Si un usuario con reserva (cupo fijo, exeptuando clase unitaria) no asiste a
> una sesión y avisa (liberando el cupo en la APP X tiempo antes), entonces
> puede recuperar dentro del mismo mes. Si se libera un cupo la idea es que se
> visualice en la agenda publica de la APP. El que reserve debe validar si es
> candidato o no, con cupos a recuperar. Si soy un usuario sin cupo, entonces
> siempre debo pagar (por TC o manualidad de ADMIN)."

Después de ADR-0024, el modelo ya expresa todo salvo una cosa. Un cliente con
`WEEKLY_QUOTA 2` tiene dos `RecurringBooking`; cada fecha de esas series está
cubierta por el pago del mes; **cualquier otra reserva es una extra y hoy
devuelve `OUTSIDE_PLAN_QUOTA`**, o sea "pagá el turno suelto".

Lo que falta es el objeto que dice *"una de esas extras te la debo, porque
liberaste un cupo a tiempo y lo pude revender"*. Sin él, el cliente que avisa
con 3 días de anticipación y el que no aparece sin avisar reciben exactamente
el mismo trato, que es justo el incentivo al revés del que el negocio quiere.

Y falta la otra mitad de la frase, que el plan ya identificó como hueco de la
Fase H: **"sin cupo siempre pagás" no tiene camino**. El pago de un turno
suelto exige `slot_occurrence_id`, no hay selector de ocurrencia en ninguna
pantalla, y los planes `DROP_IN` quedaron filtrados del selector de pagos.

### Lo que este ADR NO resuelve

- **La tarjeta.** ADR-0027 está postergado. Acá se diseña el hecho de dominio
  ("cobrar y reservar este turno es una sola cosa") de modo que la pasarela se
  enchufe sin rehacerlo — y se marca explícitamente la única decisión que
  ADR-0027 va a tener que tomar y que ADR-0025 no puede tomar por él (§5.8).
- **Prioridad sobre un cupo liberado.** Se contesta la pregunta de ADR-0019,
  pero contestándola con "no hay prioridad": orden de llegada. No hay waitlist
  ni cola, y sigue sin haberla.
- **Paquetes prepagos.** Un `makeup_credit` **no** es el `CREDITS` que ADR-0022
  eliminó (§2.1.5). No se compra, no se vende, no se recarga, y su cantidad no
  puede superar la cantidad de reservas que el cliente efectivamente perdió.

---

## 2. La decisión

### 2.1 `makeup_credits`: un derecho a una reserva extra, con vencimiento

#### 2.1.1 Shape

```
public.makeup_credits
  id                     uuid    pk
  organization_id        uuid    not null → organizations (on delete cascade)
  customer_id            uuid    not null → customers     (on delete cascade)
  service_id             uuid    not null → services      (on delete cascade)

  origin                 makeup_credit_origin not null
                         -- CUSTOMER_RELEASE | ORGANIZATION_CANCELLED | MANUAL
  source_booking_id      uuid    → bookings (on delete restrict)
                         -- null EXACTAMENTE cuando origin = 'MANUAL'
  issued_at              timestamptz not null default now()
  issued_by              uuid    → profiles          -- quién disparó la emisión

  expires_on             date    not null            -- CONGELADO al emitir
  expiry_basis           makeup_credit_expiry_basis not null  -- copia de auditoría
  expiry_basis_days      int                                   -- copia de auditoría

  status                 makeup_credit_status not null default 'AVAILABLE'
                         -- AVAILABLE | CONSUMED | REVOKED
  consumed_booking_id    uuid    → bookings (on delete restrict)
  consumed_at            timestamptz
  revoked_at             timestamptz
  revoked_by             uuid    → profiles
  note                   text                        -- obligatorio para MANUAL

  created_at, updated_at timestamptz not null default now()
```

#### 2.1.2 Por qué cada campo, y por qué no están otros

- **`service_id` es propio, no se deriva del `source_booking_id`.** Porque una
  emisión `MANUAL` no tiene reserva de origen, y porque el servicio es el
  *alcance* del crédito (§2.6), no un dato de su procedencia. Es el mismo
  criterio con el que ADR-0024 se quedó `payments.service_id`.
- **`expires_on` es `date`, no `timestamptz`.** El vencimiento es un hecho del
  calendario del negocio ("fin de mes"), y se compara contra la **fecha local
  del turno** (§2.5.1). Un `timestamptz` obligaría a hacer aritmética de
  timezone en cada lectura para responder una pregunta que ya es de días.
- **`expiry_basis` / `expiry_basis_days` son auditoría, y nada los vuelve a
  leer para calcular.** Sirven para contestar "¿por qué vence el 30?" seis
  meses después, cuando el dueño ya cambió la política tres veces. Si algún día
  alguien recalcula `expires_on` a partir de ellos, rompió la decisión 2 del
  Orchestrator; que quede escrito en el `comment on column`.
- **No hay `expired_at` ni estado `EXPIRED`** (§2.5.2).
- **No hay contador.** Un crédito es una fila. Dos créditos son dos filas. La
  clase de bugs de "decrementar / restituir / reconciliar" que ADR-0022 sacó
  al eliminar `CREDITS` y que ADR-0024 se cuidó de no reintroducir **no vuelve
  a entrar por acá**.
- **No hay `plan_id`.** Un crédito no pertenece al plan bajo el que se ganó
  (§2.6).

#### 2.1.3 CHECK / índices / triggers, y qué expresa cada uno

```sql
-- Coherencia de estado: el status nunca dice algo que las columnas no sostienen.
check ((status = 'CONSUMED') = (consumed_booking_id is not null and consumed_at is not null))
check ((status = 'REVOKED')  = (revoked_at is not null))
-- Un crédito manual no tiene reserva de origen, y uno automático la tiene siempre.
check ((origin = 'MANUAL') = (source_booking_id is null))
check (origin <> 'MANUAL' or note is not null)
-- La base de vencimiento y su N van juntos o no van.
check ((expiry_basis = 'DAYS_AFTER') = (expiry_basis_days is not null))
```

```sql
-- EL invariante anti-inflación. Una liberación ⇒ a lo sumo un crédito, sin
-- importar por cuántos caminos se llame a la emisión ni cuántas veces.
create unique index makeup_credits_one_per_source_idx
  on public.makeup_credits (source_booking_id)
  where source_booking_id is not null;

-- Una reserva consume a lo sumo un crédito. Parcial sobre CONSUMED a
-- propósito: un crédito devuelto conserva el rastro de quién lo usó y aun así
-- puede volver a usarse (§2.5.3).
create unique index makeup_credits_one_per_consumer_idx
  on public.makeup_credits (consumed_booking_id)
  where status = 'CONSUMED';

-- El camino caliente: se ejecuta adentro del lock de book_slot().
create index makeup_credits_usable_idx
  on public.makeup_credits (customer_id, service_id, expires_on)
  where status = 'AVAILABLE';

create index makeup_credits_organization_idx
  on public.makeup_credits (organization_id, status, expires_on);
```

Triggers (invariantes que cruzan tablas, así que no pueden ser CHECK — mismo
criterio y mismo precedente que `check_service_plan_same_org()`):

1. **Coherencia de tenant**: `customer.organization_id` = `service.organization_id`
   = `organization_id` de la fila, y si hay `source_booking_id` /
   `consumed_booking_id`, esas reservas también. Es **la** barrera
   cross-tenant de esta tabla.
2. **Inmutabilidad** de `customer_id`, `service_id`, `source_booking_id`,
   `origin`, `issued_at` y **`expires_on`** después del insert. Esto es la
   decisión 2 del Orchestrator ("el valor se congela al emitir") escrita donde
   se puede hacer cumplir. Precedente exacto:
   `check_service_plan_terms_immutable()`.
3. `set_updated_at`.

**RLS: `SELECT` de dos capas (ADR-0006) y ninguna policy de escritura.** El
cliente ve sus créditos; los miembros de la organización ven los de sus
clientes. `INSERT`/`UPDATE`/`DELETE` **no tienen policy**, como
`bookings` y `recurring_bookings`: todo escribe por RPC `security definer`. Sin
esto, un cliente se emite sus propios créditos con un `POST` a PostgREST, y
toda esta ADR es decorativa.

#### 2.1.4 `on delete restrict` hacia `bookings`, y lo que eso implica

`bookings.slot_occurrence_id` es `on delete cascade`. Con
`makeup_credits.source_booking_id` en `restrict`, **borrar una ocurrencia que
le costó un crédito a alguien falla ruidosamente**. Es deseado y es el mismo
criterio que el Orchestrator ya aplicó corrigiendo
`payments.slot_occurrence_id` a `restrict` en la Fase H: las ocurrencias se
cancelan, no se borran (ADR-0003 ya las congela en cuanto tienen una reserva).

#### 2.1.5 El nombre, y una advertencia

`makeup_credits` es genérico (ni "clase", ni "socio", ni "gimnasio") y es el
nombre que el plan ya usa, así que lo mantengo. Pero la palabra "crédito" va a
hacer que alguien —humano o agente— crea que volvió el paquete prepago que
ADR-0022 eliminó por decisión explícita del usuario. El `comment on table`
tiene que decirlo en una línea:

> No es un paquete prepago. No se compra ni se recarga; nace únicamente de una
> reserva perdida y su cantidad nunca puede superar la cantidad de reservas
> que el cliente efectivamente perdió.

### 2.2 La política: del dueño, resuelta como bloque, validada en la base

Cinco columnas, con el precedente exacto de `public_availability_display`
(ADR-0008): default en `Organization`, override nullable en `Service`, modo
efectivo = `coalesce(override, default)`.

```
organizations.makeup_credits_enabled          boolean not null default false
organizations.release_deadline_hours          int     not null default 12
                                              check (between 0 and 720)
organizations.makeup_credit_expiry            makeup_credit_expiry_basis not null
                                              default 'END_OF_MONTH'
organizations.makeup_credit_expiry_days       int     check (>= 1)

services.makeup_credits_enabled_override      boolean
services.release_deadline_hours_override      int
services.makeup_credit_expiry_override        makeup_credit_expiry_basis
services.makeup_credit_expiry_days_override   int
```

Valores de `makeup_credit_expiry_basis`: **`END_OF_MONTH`** |
**`END_OF_BILLING_PERIOD`** | **`DAYS_AFTER`**. El segundo no estaba en el
plan; ver §2.5.1 y §5.4.

Tres precisiones sin las cuales esto sale mal:

1. **La política se resuelve como un bloque, no columna por columna.** Si el
   servicio hace override del modo a `DAYS_AFTER` pero el `days` se toma con
   un `coalesce` independiente, queda un `DAYS_AFTER` sin N (o peor, con el N
   de otro modo de la organización). La resolución correcta es: *si el
   servicio define modo, se usan modo y días **del servicio**; si no, los dos
   de la organización.* Con su CHECK de coherencia en cada nivel.
2. **Se valida en la base, no en el formulario** — decisión ya tomada, y el
   motivo vale la pena repetirlo: `organizations` y `services` son escribibles
   por PostgREST directo. Zod en el form no es una defensa (mismo argumento de
   ADR-0020 para el color de marca y de ADR-0024 para el `weekly_quota`).
3. **El techo de 720 h (30 días) no es decorativo**: una anticipación exigida
   mayor que el horizonte de reservas volvería imposible ganar un crédito, y
   el dueño no tendría cómo darse cuenta salvo por el reclamo de un cliente.

### 2.3 Emisión: qué la dispara exactamente, y qué no

#### 2.3.1 Una sola función, tres llamadores

`issue_makeup_credit(p_booking_id, p_origin, p_enforce_deadline boolean)`,
idempotente vía `on conflict do nothing` sobre el índice único de
`source_booking_id`. Una sola implementación (ADR-0018), llamada desde:

| Llamador | Motivo que estampa | ¿Emite? | ¿Exige anticipación? |
|---|---|---|---|
| `cancel_booking()` — una fecha | `CUSTOMER_REQUEST` | **Sí** | **Sí** |
| `cancel_slot_occurrence()` | `SLOT_CANCELLED` | **Sí** | **No** |
| `discontinue_schedule_rule()` / `_group()` | `RULE_DISCONTINUED` | **Sí** | **No** |
| `cancel_recurring_booking()` — la serie entera | (ver §5.2) | **Nunca** | — |

`cancel_recurring_booking()` no emite **la pida quien la pida** (§0.3). Si la
organización le corta la serie a un cliente que pagó el mes, el remedio es
económico (`VOID` + recargar, o un crédito `MANUAL` deliberado), no ocho
créditos automáticos que nadie decidió emitir.

#### 2.3.2 Las cinco condiciones de emisión

Se emite si y sólo si, en el momento de cancelar, se cumplen **todas**:

1. **`makeup_credits_enabled` efectivo** para ese servicio.
2. La `Booking` estaba **`CONFIRMED`**. Una `NOT_GENERATED` nunca ocupó un
   cupo ni costó plata: no hay nada que devolver. (`CANCELLED` sale antes por
   el early-return que ya tiene `cancel_booking`.)
3. **El servicio exige pago** (`payment_required`). En un servicio gratis el
   crédito no vale nada: la cobertura ya da `OK` en el paso 1 de
   `evaluate_payment_coverage()` y el crédito no otorga prioridad, sólo precio
   (§2.4.1). Emitirlo sería llenar la tabla de filas inútiles y llenar el
   portal del cliente de un dato que no significa nada.
4. **La reserva estaba efectivamente cubierta**: se vuelve a llamar a
   `evaluate_payment_coverage(customer, service, occurrence, recurring_booking_id, false)`
   y tiene que dar `'OK'`. **Esta es la forma de expresar "vino de una serie
   dentro de cuota" sin reimplementar nada de ADR-0024**: si la reserva venía
   de una serie dentro de cuota, esa función ya devuelve `OK` por el camino de
   `customer_series_quota_position() <= weekly_quota`. Un llamado a una
   función `stable`, cero lógica duplicada. Y de yapa cierra el caso límite
   de "libero un cupo de un mes que no pagué" (§6.10).
5. **La reserva no consumió un crédito.** Si consumió, el remedio es
   devolverlo (§2.5.3), nunca emitir uno nuevo — o cancelar-y-reservar sería
   la fábrica infinita.

Y para `cancel_booking()` (el único con `p_enforce_deadline = true`), una
sexta:

6. `now() <= occurrence.start_at - (release_deadline_hours efectivas) * interval '1 hour'`.
   Se compara `timestamptz` contra `timestamptz`: no hay aritmética de
   timezone y por lo tanto no hay bug de DST (ADR-0014 se resuelve solo acá).

#### 2.3.3 Por qué la clase suelta no emite, y dónde esa regla **no** aplica

"Exceptuando clase unitaria" (textual). Un turno suelto no tiene nada que
recuperar: el cliente compró **esa** clase. Con la condición 4 sale gratis,
sin ninguna cláusula especial… salvo por un detalle: un pago `DROP_IN`
anclado a esa ocurrencia **también** da `OK` en la cobertura. Así que hace
falta decirlo explícito, y hay que separar dos situaciones que el plan mezcla:

- **El cliente libera su turno suelto** → **no emite**. Compró esa clase y
  decidió no ir. (Si el negocio le quiere devolver algo, `VOID` del pago o un
  crédito `MANUAL`. Es una decisión de plata, la toma una persona.)
- **La organización cancela la clase que ese cliente pagó suelta** → **emite
  igual**. No perdió nada por decisión propia, y sin el crédito queda con un
  `Payment PAID` anclado a una ocurrencia cancelada, que el índice único
  `payments_one_paid_per_occurrence_idx` le impide reutilizar en otro turno.
  El crédito es literalmente el vehículo que hace que su plata no se evapore.

O sea: la regla "no se emite para una reserva suelta" **vale para
`CUSTOMER_REQUEST` y no vale para las cancelaciones de la organización**. Como
está escrita en el plan (sin distinguir), dejaría al cliente que pagó 800 pesos
por una clase que el estudio canceló sin ningún remedio dentro del producto.

> Nota operativa que hay que decir en la pantalla: si además el mostrador le
> devuelve la plata (`VOID`), tiene que revocar el crédito, o el cliente
> cobró dos veces. Por eso existe `REVOKED` y por eso la pantalla de anular un
> pago tiene que avisar si ese pago tiene un crédito vivo asociado.

### 2.4 Consumo: dónde entra exactamente, y qué gana el que tiene crédito

#### 2.4.1 El crédito es el **último** recurso de cobertura, nunca el primero

La cadena de ADR-0024 no se toca. Se le agrega una única compuerta, al final,
que sólo se consulta cuando lo anterior ya falló:

```
1. servicio sin payment_required           → OK            (no toca el crédito)
2. fecha local del slot
3. pago DROP_IN de ESTA ocurrencia         → OK            (no toca el crédito)
4. pago por período en vigencia
   4a. hay pago:
       UNLIMITED                           → OK            (no toca el crédito)
       WEEKLY_QUOTA y la serie entra       → OK            (no toca el crédito)
       WEEKLY_QUOTA y no entra             → ↓ compuerta
       DROP_IN (dato corrupto)             → ↓ compuerta
   4b. no hay pago                         → ↓ compuerta
5. COMPUERTA DE CRÉDITO
   ¿hay crédito usable de (customer, service) para la fecha local del slot?
     sí → OK, con el id del crédito adjunto
     no → el veredicto que venía de arriba, intacto
          (OUTSIDE_PLAN_QUOTA / OVER_PLAN_QUOTA / PAYMENT_REQUIRED /
           SERVICE_HAS_NO_PLAN)
```

**Invariante: nunca se gasta un crédito si otra cobertura ya alcanzaba.** Es la
respuesta directa a "si el cliente tiene crédito **y** podría pagar suelto,
¿qué gana?": **gana no pagar, y gana no gastar el crédito al pedo.** Si ya
pagó ese turno suelto (paso 3), el crédito ni se mira y le queda para otra
clase. Si tiene el mes pago con `UNLIMITED`, ídem.

Dos decisiones dentro de la compuerta que hay que justificar:

- **El crédito también rescata `PAYMENT_REQUIRED`**, no sólo
  `OUTSIDE_PLAN_QUOTA`. Un crédito es una clase **ya pagada**; negárselo a
  alguien que dejó de pagar el mes siguiente sería cobrarle dos veces lo mismo.
  Es "pago ≠ permiso" aplicado en la dirección incómoda, que es la que
  importa. En la práctica casi nunca se nota, porque con `END_OF_MONTH` el
  crédito no le sobrevive al período pago; se nota con `DAYS_AFTER 30`.
- **El crédito también rescata `SERVICE_HAS_NO_PLAN`** por la misma razón: que
  el dueño haya desordenado su lista de precios no puede anular algo que el
  cliente ya tiene en la mano (es el invariante de ADR-0024 §6.d llevado al
  crédito).

#### 2.4.2 Qué crédito se elige cuando hay más de uno

`order by expires_on asc, issued_at asc, id asc`. **El que vence primero.**
Maximiza lo que el cliente conserva y es lo que cualquiera esperaría. El orden
tiene que ser **total** (por eso los tres campos): un orden no-total permitiría
que el preview y el confirm no coincidan en cuál crédito se va a usar, que es
la clase de divergencia que ADR-0018 vino a matar.

#### 2.4.3 El veredicto tiene que decir "sí, **con el crédito X**"

Requerimiento de dominio, con la implementación a cargo de `database-agent`
(mismo reparto que la resolución 7 de ADR-0024):

- El resolutor de cobertura devuelve **un par**: `(reason, makeup_credit_id)`.
  `reason = 'OK'` y `makeup_credit_id` no nulo ⇒ entra con crédito.
- **Prohibido expresarlo como un valor nuevo del enum `can_book_reason`.** Seis
  funciones comparan `= 'OK'` / `<> 'OK'` (`payment_covers_slot`, `book_slot`,
  `admin_book_for_customer`, `generate_recurring_booking`,
  `retry_not_generated_booking`, `evaluate_customer_booking`). Un `'OK_CON_CREDITO'`
  sería un "sí" que varias de ellas leen como "no", en silencio y sólo para el
  cliente que tiene crédito. Cambiar la **firma** rompe en el `supabase db
  reset`; agregar un valor de enum rompe en producción.
- `payment_covers_slot()` mantiene su contrato booleano y pasa a devolver
  `true` cuando hay crédito usable — es correcto, la pregunta que responde es
  "¿puede reservar?".
- **`generate_recurring_booking()` y `retry_not_generated_booking()` NO
  consumen créditos jamás.** Tres razones: corren sin nadie mirando (pg_cron),
  gastarían el crédito en una fecha que ADR-0019 considera *pendiente, no
  decidida* (y que se confirmaría sola si el cliente paga), y el recupero es un
  acto del cliente eligiendo otra clase, no algo que un job decide por él. Se
  les pasa "sin crédito" explícito.

#### 2.4.4 Ningún motivo de rechazo nuevo, y por qué eso **no** es colapsar dos situaciones

El criterio que ya costó dos correcciones (ADR-0018, Phase 12, ADR-0024) es:
*un motivo que colapsa dos situaciones distintas termina mintiéndole al
cliente*. Acá no se colapsa nada, porque **las dos situaciones no coexisten**:
si hay crédito no hay rechazo. El único rechazo posible sigue siendo "esto es
una extra y no tenés con qué pagarla", que es exactamente lo que
`OUTSIDE_PLAN_QUOTA` significa desde ADR-0024, con el remedio que ADR-0024 ya
le escribió ("clase suelta, crédito de recupero, o esperar").

El caso tentador es **"tenías un crédito y venció"**. Es información distinta,
sí, pero **no es un motivo distinto**: el remedio es el mismo (pagá el turno) y
un `MAKEUP_CREDIT_EXPIRED` daría a entender que el crédito era el único camino,
cuando pagar sigue estando. Va como **contexto del preview y del portal** ("no
tenés créditos disponibles; el último venció el 30/09"), no como código de
rechazo. La distinción es la que el proyecto ya viene usando: el `reason` dice
**por qué no se puede**; la pantalla dice **qué hacer** y con qué datos.

Lo que sí es vocabulario nuevo, y es de **lectura**, no de rechazo, es el
camino de cobertura que el preview/quote devuelve:
`FREE_SERVICE | DROP_IN_PAID | PLAN_UNLIMITED | PLAN_QUOTA | MAKEUP_CREDIT | NONE`.

#### 2.4.5 Atomicidad: la reserva y el consumo son **un solo hecho**

Adentro de `book_slot()` / `admin_book_for_customer()`, bajo el lock de la
ocurrencia (ADR-0004) y con **lock propio de la fila del crédito** (§0.1):

1. lock de la ocurrencia → capacidad → duplicado (explícito, antes de tocar
   nada);
2. resolver cobertura → obtener `makeup_credit_id` (o null);
3. insertar la `Booking`;
4. `update makeup_credits set status='CONSUMED', consumed_booking_id=<nueva>,
   consumed_at=now() where id = <credit> and status = 'AVAILABLE'`.

**Si el paso 4 afecta 0 filas, la operación entera se aborta.** No se devuelve
`OK`. No se devuelve un status "sin crédito". Se levanta excepción y rollea
todo, porque cualquier otra cosa deja una reserva extra confirmada sin haber
consumido nada — o sea, una reserva gratis. Es el único lugar de este diseño
donde una excepción es preferible a un status explícito, y conviene decirlo
porque contradice el patrón general de ADR-0004 ("resultado explícito, nunca
excepción genérica").

**Opt-out, no opt-in.** `p_use_makeup_credit boolean default true`: el cliente
(o el mostrador) puede decidir guardarse el crédito y pagar el turno. El
default es usarlo, y el preview **tiene que decir cuál va a usar y cuándo
vence** antes de confirmar. "Me gastaron el crédito sin avisar" es un reclamo
de mostrador evitable con una línea de texto.

### 2.5 Vencimiento y devolución

#### 2.5.1 Contra qué se compara el vencimiento: **la fecha del turno**, no `now()`

ADR-0013 dejó la lección como "se valida contra la fecha del slot, nunca
contra la fecha en que se reserva". Acá esa lección **no se copia, se aplica**:
"puede recuperar dentro del mismo mes" quiere decir que **la clase de recupero
ocurre** dentro del mes, no que se reserve dentro del mes. Reservar el 30 de
septiembre una clase del 15 de octubre no es recuperar en septiembre.

```
crédito usable para una ocurrencia  ⟺  status = 'AVAILABLE'
                                   ∧  slot_local_date(ocurrencia) <= expires_on
```

Como sólo se pueden reservar ocurrencias futuras (`end_at < now()` ya devuelve
`OCCURRENCE_NOT_AVAILABLE`), esta única comparación alcanza: no hace falta
además chequear contra `now()`.

Cálculo de `expires_on`, **anclado a la fecha local de la ocurrencia liberada**
(no a `issued_at`):

| `expiry_basis` | `expires_on` |
|---|---|
| `END_OF_MONTH` | último día del mes calendario de la fecha local de la ocurrencia liberada |
| `END_OF_BILLING_PERIOD` | `period_end` del `Payment` que cubría esa ocurrencia; si no hay (caso `MANUAL` o pago anulado), cae a `END_OF_MONTH` |
| `DAYS_AFTER` | fecha local de la ocurrencia liberada **+ N días** |

Anclar a la ocurrencia y no a `issued_at` importa: si aviso tres semanas antes,
`DAYS_AFTER 7` medido desde la emisión me quemaría la ventana **antes de que la
clase que perdí siquiera ocurra**. Y para `END_OF_MONTH`, libero el 30/09 una
clase del 02/10: el mes que corresponde es octubre, el de la clase que perdí.

#### 2.5.2 `EXPIRED` **no se persiste**: es derivado, como `COMPLETED`

ADR-0010 decidió que `SlotOccurrence.COMPLETED` no es un estado persistido
sino `start_at < now()` calculado en la lectura, "para evitar un cuarto estado
sin decisión de negocio detrás". Acá pasa literalmente lo mismo: nadie *decide*
que un crédito venza, vence solo por el paso del tiempo contra un valor que ya
está congelado en la fila.

Consecuencias, todas buenas:

- **No hay job de vencimiento.** Hay `pg_cron` (ADR-0003/0009) y no se usa para
  esto. Un job que marca `EXPIRED` introduce una ventana en la que el crédito
  ya venció y todavía figura disponible — y la lectura tendría que chequear
  `expires_on` igual **por las dudas**, o sea dos fuentes de verdad para el
  mismo hecho.
- `status` queda conteniendo únicamente cosas que **alguien hizo**:
  `AVAILABLE` (nada pasó), `CONSUMED` (se usó), `REVOKED` (se lo sacaron).
- La devolución se simplifica (§2.5.3).
- Un crédito vencido queda para siempre como `AVAILABLE` en la tabla, igual que
  una ocurrencia pasada queda `ACTIVE`. Es consistente con el precedente y el
  portal lo muestra como "vencido" derivándolo.

> Único costo real: el dueño no puede filtrar por `status = 'EXPIRED'` en
> PostgREST directo. Lo resuelve la RPC de lectura devolviendo `is_expired` ya
> calculado. Es el mismo costo que ya se paga con `COMPLETED`.

#### 2.5.3 Devolución: sin condición de vencimiento

El plan dice "vuelve a `AVAILABLE` si esa reserva se cancela **y el crédito no
venció**". Con el vencimiento derivado y `expires_on` congelado, **esa condición
sobra**: el crédito vuelve siempre, y si venció es inútil por sí solo. Un
condicional de menos en el camino más delicado del motor.

```
cancelar una Booking  →  update makeup_credits
                            set status = 'AVAILABLE', consumed_at = null
                          where consumed_booking_id = <booking> and status = 'CONSUMED'
```

`consumed_booking_id` **no se limpia**: conserva el rastro de en qué se había
usado, y el índice único parcial (`where status = 'CONSUMED'`) le permite
volver a consumirse igual.

Aplica a cualquier cancelación, la haga quien la haga. Y combinada con la
condición 5 de emisión (§2.3.2) da la respuesta a "¿se devuelve el original o
se emite uno nuevo?": **siempre el original, nunca uno nuevo**, incluso cuando
el que cancela es la organización (§6.6).

#### 2.5.4 El invariante que resume todo esto

> **El stock de créditos de un cliente nunca puede superar la cantidad de
> reservas que efectivamente perdió.**

Garantizado por tres cosas independientes, y por eso es robusto a que alguien
agregue un camino de cancelación nuevo el año que viene:
`unique (source_booking_id)`; la cancelación en cascada de una serie no emite;
y una reserva que consumió un crédito no tiene serie, así que no puede emitir.

### 2.6 Alcance del crédito

| Pregunta | Respuesta | Por qué |
|---|---|---|
| ¿Cruza organizaciones? | **Nunca.** Invariante. | Un `Customer` es por organización (ADR-0006). `organization_id` en la fila + trigger de coherencia + RLS de dos capas. Ni siquiera es una regla de negocio: es la frontera de tenant. |
| ¿Sirve para otro `Service`? | **No.** | Una hora de pilates no vale una hora de cancha. Un crédito multi-servicio sería una transferencia de valor entre listas de precios que el dueño nunca autorizó, y volvería a hacer incontestable "¿de qué servicio es esto?" — la regresión que ADR-0022 arregló matando el entitlement multi-servicio. |
| ¿Está atado al plan bajo el que se ganó? | **No.** | Los planes se desactivan (ADR-0024 §6.d: desactivar no invalida lo ya comprado). Atarlo al plan haría que ordenar la lista de precios anule créditos vivos. |
| ¿Sirve para otro día/semana dentro del mes? | **Sí — es toda la feature.** | Cualquier ocurrencia de ese servicio con fecha local ≤ `expires_on`: otro día, otra regla, otro `Resource`, otro profesional. |
| ¿Sirve para una fecha de una serie? | **No.** | Sólo habilita reservas *extra*. Las fechas de serie las resuelve `generate_recurring_booking()`, que no consume créditos (§2.4.3). |
| ¿Da prioridad sobre el cupo? | **No.** | §2.7. Orden de llegada; lo que cambia es el precio, no la prioridad. |

### 2.7 El cupo liberado en la agenda pública

#### 2.7.1 Esto contesta ADR-0019

> **El cupo liberado vuelve a la agenda pública y es por orden de llegada. Lo
> que cambia entre candidatos no es la prioridad, es el precio: con crédito no
> paga, sin crédito paga.**

Se escribe como invariante porque es una tentación permanente: la primera vez
que un cliente reclame "yo liberé mi cupo y me lo tomó otro", alguien va a
querer darle prioridad al que tiene crédito. Eso es una waitlist, está fuera de
alcance desde ADR-0012, y traería reintentos asíncronos que ADR-0010 descartó
explícitamente.

#### 2.7.2 Qué se muestra

Técnicamente el cupo liberado **ya se ve**: `get_public_availability()` cuenta
reservas `CONFIRMED` reales, así que una cancelación reabre el lugar sola. Lo
que falta es que se **note**. Una única columna derivada:

```
recently_released boolean
  = (remaining > 0)
  ∧ existe una Booking CANCELLED de esa ocurrencia
      con cancellation_reason = 'CUSTOMER_REQUEST'
      y cancelled_at >= now() - interval '72 hours'
```

#### 2.7.3 Qué NO se muestra, y las dos supresiones

- **Ningún número.** `recently_released` es booleano. ADR-0008 es tajante: el
  conteo exacto sólo existe en modo `EXACT`, y en los otros modos los campos
  numéricos **se omiten del payload**, no van en `null`. Un badge "1 lugar
  liberado" sería una fuga numérica disfrazada de adorno.
- **Ningún actor.** Ni nombre, ni inicial, ni avatar, ni "un alumno liberó".
- **Ningún timestamp.** "Liberado hace 5 minutos" es una huella temporal que
  permite correlacionar a una persona con una cancelación.
- **Ningún reordenamiento.** La lista pública no se ordena por "recién
  liberados": eso rankearía los turnos por actividad de cancelación, que es un
  canal lateral gratis para cualquiera que mire dos veces.
- **Supresión 1 — modo `BOOLEAN`:** no se expone. `BOOLEAN` es el modo que una
  organización elige precisamente para que no se pueda inferir la existencia de
  reservas individuales (el ejemplo del psicólogo en ADR-0008). Un flag que
  anuncia una *transición* dice más que el estado que ese modo oculta.
- **Supresión 2 — `capacity = 1`:** no se expone en ningún modo. Con capacidad
  1, "se liberó un lugar" es idéntico a "existía una reserva y la persona la
  canceló" — es el problema original de ADR-0007 en su forma más pura.

En `EXACT` y `LIMITED` con capacidad ≥ 2 sí se expone. Argumento: quien
*pollea* el endpoint ya ve la transición `FULL → AVAILABLE` con o sin badge, así
que el flag no le agrega información; lo que hace es **emparejar al usuario
casual con el que polea**, que es justo lo que el cliente pidió.

#### 2.7.4 El visitante anónimo no es candidato de nada

"El que reserve debe validar si es candidato o no, con cupos a recuperar"
requiere identidad. Sin sesión, la página pública muestra disponibilidad y **el
precio del turno suelto** (el plan `DROP_IN` activo — `service_plans` ya tiene
policy de `SELECT` público). Con sesión, muestra el *quote* (§2.8.1): "lo tomás
con tu crédito (vence el 30/09)" / "lo cubre tu plan" / "cuesta $800". Nunca al
revés: un anónimo no puede enterarse de que *existen* créditos de terceros.

### 2.8 "Sin cupo siempre pagás": el turno suelto como hecho de dominio

#### 2.8.1 Dos operaciones, no una pantalla de pagos con un selector más

**`quote_booking(customer, occurrence) → { can_book, reason, coverage_path,
price, currency, makeup_credit_id?, makeup_credit_expires_on? }`** — lectura.
Es el preview del cliente, el del mostrador y, cuando llegue, el que le dice al
checkout de la tarjeta cuánto cobrar. **Envuelve el mismo resolutor de
cobertura**, no una copia (ADR-0018). El precio sale del plan `DROP_IN` activo
del servicio — que es único por índice, así que "cuánto sale este turno" tiene
una única respuesta.

**`book_slot_paying(occurrence, customer, amount?)`** — escritura, atómica,
bajo el mismo lock. Y la propiedad que la define:

> **Cobrar y reservar son el mismo hecho.** O quedan el `Payment PAID` anclado a
> la ocurrencia **y** la `Booking CONFIRMED`, o no queda nada. Nunca un pago sin
> reserva porque el cupo se llenó en el medio (lo marcó `database-agent` y es
> correcto: hoy son dos escrituras separadas y el hueco existe).

Con dos reglas de orden:

1. **Si el turno ya está cubierto, no se cobra.** La operación resuelve
   cobertura primero; si da `OK` (por plan, por pago previo o por crédito),
   reserva y **no genera pago**, informando por qué camino entró. "Pagá este
   turno" nunca puede cobrarle a alguien que tenía cómo entrar gratis.
2. **El plan y el precio no son input del cliente.** El plan sale del `DROP_IN`
   activo; el `amount` puede ajustarlo el staff (descuento de mostrador) y
   **nunca** el cliente. Hoy el pago lo carga el ADMIN a mano — ADR-0027 está
   postergado — y este diseño funciona igual: `book_slot_paying` ejecutada por
   un miembro de la organización es exactamente "cobré y lo anoté".

#### 2.8.2 Que esto no se pinte en un rincón cuando llegue la tarjeta

Cuando exista `payment_intents` (ADR-0027), el flujo es: `quote_booking()` dice
el precio → se crea el intent contra **esa ocurrencia** → el webhook confirma →
se ejecuta **la misma** `book_slot_paying` con la referencia del intent en vez
de `MANUAL`. Nada del motor cambia.

Lo que ADR-0025 **no puede decidir** y ADR-0027 tiene que decidir (§5.8): qué
pasa si el cupo se llena entre el checkout y el webhook. Las dos salidas son
*retener el lugar antes de capturar* (implica un estado de `Booking` nuevo, que
toca el invariante de capacidad y por lo tanto es una decisión del
Orchestrator, no del agente de pagos) o *capturar y reembolsar si no hay
lugar*. Dejo el camino manual diseñado de modo que cualquiera de las dos se
pueda agregar arriba sin rehacerlo.

### 2.9 Portal del cliente

Hoy `me/servicios` lee `services.price` vía `my_services()`, que después de
ADR-0024 puede ser **directamente falso** (un servicio con cuatro planes tiene
cuatro precios y la columna tiene uno solo, deprecado). El portal tiene que
contestar tres preguntas y ninguna más:

1. **¿Qué compré?** El plan del pago vigente (nombre, `plan_kind`,
   `weekly_quota`), `covered_until`, y **cuántos de sus cupos fijos está
   usando** (`series en vigencia / weekly_quota`) — el dato que vuelve legible
   todo ADR-0024 para el cliente.
2. **¿Cuánto sale lo que no compré?** Los planes activos con su precio y la
   moneda de la organización. `services.price`, `billing_type` y
   `billing_cycle` **salen del payload**: no se muestran deprecados "por las
   dudas", se sacan.
3. **¿Qué tengo a favor?** `my_makeup_credits()`: por crédito, el servicio, de
   qué clase salió (fecha y hora de la ocurrencia liberada), `expires_on`,
   `is_expired` **calculado en SQL**, y el estado. Más un resumen "tenés 1
   crédito, vence el 30/09" que es lo que el cliente realmente mira.

Todo por RPC `security definer`, como `my_services()` / `my_bookings()` /
`my_payments()`: **el cliente sigue sin poder leer `slot_occurrences` ni
`services` directamente**. Dos precisiones sobre eso:

- La fecha/hora de la ocurrencia liberada la devuelve la RPC ya resuelta a la
  timezone de la organización. El cliente no consulta la ocurrencia; recibe un
  texto sobre **su propia** reserva pasada, que es dato suyo.
- `service_plans` **sí** tiene policy de `SELECT` público (es una lista de
  precios). El portal podría leerla directo; igual conviene que vaya por la
  misma RPC, para que "cuál es tu plan vigente" no se resuelva en TypeScript
  reimplementando `evaluate_payment_coverage`.

Cambiar el shape de `my_services()` es un contrato de API → §5.6.

---

## 3. El caso pilates, completo

`Service` "Pilates", capacidad 3. Planes: `DROP_IN` 800; `WEEKLY_QUOTA` 1/2/3 →
1800/2500/3400. Organización con `makeup_credits_enabled = true`,
`release_deadline_hours = 12`, `makeup_credit_expiry = END_OF_MONTH`.

| # | Hecho | Qué pasa |
|---|---|---|
| 1 | Ana paga 2500 (`WEEKLY_QUOTA` 2): martes 09:00 y jueves 09:00 fijos | Dos `RecurringBooking`. Cada fecha confirma porque la posición de serie (1 y 2) ≤ 2. |
| 2 | Lunes 20:00, Ana libera su martes 09:00 | `cancel_booking` → `CUSTOMER_REQUEST`. 13 h de anticipación ≥ 12 → cobertura re-evaluada da `OK` (serie en cuota) → **1 crédito, vence 30/09**. |
| 3 | El martes 09:00 aparece con lugar en la agenda pública | `remaining` pasa de 0 a 1 y `recently_released = true` (modo `LIMITED`, capacidad 3: sin número, sin nombre, sin hora de cancelación). |
| 4 | Bruno, sin plan, toma ese martes | `quote_booking` → `PAYMENT_REQUIRED`, `coverage_path = NONE`, precio 800. `book_slot_paying` cobra y reserva en una sola transacción. |
| 5 | Ana toma el sábado 10:00 (no es ninguno de sus horarios fijos) | Cobertura: no hay pago suelto; hay pago por período con plan de cuota; reserva sin serie ⇒ iba a `OUTSIDE_PLAN_QUOTA`; compuerta de crédito ⇒ **`OK` con crédito**. Se consume. No paga. |
| 6 | Ana cancela ese sábado | El crédito **vuelve** a `AVAILABLE` con su `expires_on` original. No se emite uno nuevo (la reserva no venía de una serie). |
| 7 | Ana libera su jueves 09:00 dos horas antes | Menos de 12 h → **no emite**. Pierde la clase. El cupo se libera igual y se ve en la agenda igual. |
| 8 | El estudio cancela el martes 09:00 de la semana siguiente | `cancel_slot_occurrence` → `SLOT_CANCELLED` → **créditos para todos los confirmados, sin exigir anticipación**, incluido Bruno, que había pagado suelto. |
| 9 | 1/10: Ana tiene 1 crédito vencido el 30/09 | Sigue en la tabla como `AVAILABLE`; el portal lo muestra "vencido" porque `expires_on < hoy`. No sirve para ninguna clase de octubre. Ningún job corrió. |

---

## 4. Migración que implica

Aditiva. **Sin backfill**: no existen créditos previos.

1. Tipos: `makeup_credit_origin`, `makeup_credit_status`,
   `makeup_credit_expiry_basis`.
2. Tabla `makeup_credits` + CHECKs + 4 índices + 3 triggers + RLS
   `SELECT`-only.
3. 4 columnas en `organizations`, 4 en `services`, con sus CHECK de coherencia
   por nivel.
4. `resolve_usable_makeup_credit()` + el resolutor de cobertura con veredicto
   compuesto; `payment_covers_slot()` se mantiene booleano.
5. `issue_makeup_credit()` y sus tres llamadores
   (`cancel_booking`, `cancel_slot_occurrence`, `discontinue_schedule_rule` /
   `_group`).
6. Devolución en toda cancelación.
7. `book_slot()` / `admin_book_for_customer()`: consumo atómico + `p_use_makeup_credit`.
8. `book_slot_paying()` + `quote_booking()`.
9. `get_public_availability()`: +1 columna ⇒ `drop function` + recrear, con el
   cuidado que ya documentó Phase 16 ("reescribir de memoria las reglas de
   disclosure de ADR-0008 es cómo cambian en silencio"). **Toca todos los call
   sites del frontend.**
10. `my_services()` reescrita, `my_makeup_credits()` nueva.
11. **Si el Orchestrator aprueba §5.1/§5.2:** motivo derivado del actor en
    `cancel_booking()` y valor `SERIES_CANCELLED` en
    `booking_cancellation_reason`.

**Nadie cambia de comportamiento el día del deploy**, porque
`makeup_credits_enabled` arranca en `false` (§0.5). El único cambio visible sin
encender nada es el shape de `my_services()` y la columna nueva de la
disponibilidad pública.

**Secuenciación vinculante**, mismo criterio que la Fase H: la migración,
`quote_booking` y la pantalla de "pagá este turno" **salen juntas**. Si sale la
migración sola, el cliente ve `OUTSIDE_PLAN_QUOTA` sin ninguna forma de
resolverlo desde la pantalla — que es el estado de hoy, pero ahora con una
tabla vacía al lado que promete otra cosa.

---

## 5. Lo que NO decido yo — necesita al Orchestrator

1. **Cambiar el contrato de `cancel_booking()`** para que el motivo se derive
   del actor (§0.2). Es contrato de API + seguridad. **Bloqueante**: sin esto
   la política de anticipación es voluntaria.
2. **Agregar `SERIES_CANCELLED` a `booking_cancellation_reason`** (§0.3). Es la
   taxonomía de ADR-0010. No bloquea el agujero (la emisión se decide por
   camino de llamada) pero sin él el motivo miente.
3. **`makeup_credits_enabled` con default `false`** (§0.5). Es un default de
   producto, y contradice la lectura literal de "defaults 12 h y fin de mes".
4. **Tercer valor `END_OF_BILLING_PERIOD`** (§2.5.1, §6.9). Si el Orchestrator
   prefiere dos valores, la respuesta a "¿el mes del crédito es el calendario o
   el del pago?" pasa a ser **siempre el calendario**, y hay que documentar que
   en un servicio `ROLLING_MONTH` un cliente que libera el día 20 recibe menos
   ventana que uno que libera el 2. Es defendible, pero es una decisión, no un
   detalle.
5. **`origin = 'MANUAL'`** (crédito de cortesía cargado por el OWNER) dentro o
   fuera de la Fase I. A favor: es el único remedio para media docena de casos
   límite (§6.6, §2.3.3) y evita que se inventen dos mecanismos distintos
   después. En contra: es la puerta por la que se emiten créditos sin origen, y
   hay que auditarla. Yo lo incluiría, gateado a OWNER y con `note` obligatoria.
6. **Shape de `my_services()`** (sacar `price`/`billing_*` deprecados) y **+1
   columna en `get_public_availability()`**. Contratos de API ya definidos.
7. **La firma exacta del veredicto compuesto** — detalle de `database-agent`,
   con el precedente de la resolución 7 de ADR-0024. Lo **no negociable** es
   que siga habiendo una sola función de decisión y que el qualifier **no sea
   un valor del enum** (§2.4.3).
8. **La pregunta que hereda ADR-0027**: cupo que se llena entre el checkout y
   el webhook — retener antes de capturar (estado de `Booking` nuevo, toca el
   invariante de capacidad) vs. capturar y reembolsar (§2.8.2).
9. **Ventana de `recently_released` (72 h) y su supresión en `BOOLEAN` /
   capacidad 1** (§2.7.3). Son decisiones de disclosure, o sea territorio de
   `auth-security-agent` + ADR-0008. Yo propongo, no cierro.

### Lo que no pude decidir por falta de datos

- **Cuántos créditos vivos tolera la capacidad de un negocio.** Si el estudio
  emite 20 créditos en un mes y tiene 3 lugares por clase, los créditos valen
  poco y el reclamo lo recibe el mostrador. No es un problema técnico y no lo
  puedo dimensionar sin datos de uso: lo mitiga que el panel admin muestre
  "créditos vivos por servicio y por mes". Lo dejo anotado, no resuelto.

---

## 6. Casos límite — contestados, no dejados abiertos

1. **Crédito emitido y después el cliente da de baja su plan.** El crédito
   sobrevive y sirve hasta `expires_on`, aunque no haya ningún pago vigente
   (§2.4.1). Ya pagó esa clase. Con `END_OF_MONTH` casi nunca se nota; con
   `DAYS_AFTER 30`, sí — y está bien que sí.
2. **Crédito de un servicio cuyo plan se desactivó.** Sirve igual. El crédito
   está atado al `Service`, no al plan (§2.6), y es el mismo invariante de
   ADR-0024: desactivar un plan no invalida lo ya comprado.
3. **Dos créditos disponibles y una sola reserva.** Se consume **uno**: el de
   `expires_on` más próximo, con desempate total `issued_at, id` (§2.4.2).
4. **Crédito emitido sobre una serie que después se cancela entera.** El
   crédito **queda**: se ganó por una fecha liberada a tiempo, y cancelar la
   serie después no deshace ese hecho. La cancelación de la serie, por su
   parte, **no emite nada** (§0.3).
5. **La organización descontinúa la regla (`RULE_DISCONTINUED`).** Emite **un
   crédito por cada `Booking` `CONFIRMED` futura perdida**, sin exigir
   anticipación. No es inflación, es conservación: esas fechas estaban
   confirmadas, y una fecha de serie sólo llega a `CONFIRMED` si la cobertura
   daba `OK` al generarla — o sea que **`CONFIRMED` ya implica "estaba
   pago"**, gratis. Las `NOT_GENERATED` no emiten: nunca ocuparon un cupo.
   *Operativo:* la pantalla tiene que decir "esto va a emitir N créditos" antes
   de confirmar, con el mismo espíritu del preview de ADR-0012.
6. **Un crédito usado para reservar un turno que después cancela la
   organización.** Se **devuelve** el original, con su `expires_on` intacto, y
   **no** se emite uno nuevo — si no, una cancelación de la organización
   duplicaría el crédito. Caso incómodo honesto: si la organización cancela el
   29 y el crédito vencía el 30, queda casi sin valor. **No se extiende
   automáticamente**: mover un `expires_on` congelado rompe la decisión 2. El
   remedio es un crédito `MANUAL` del mostrador (§5.5), auditado y con motivo.
7. **Reserva que consumió un crédito y se cancela a tiempo.** Devuelve el
   crédito y **no emite** otro (condición 5 de §2.3.2, y además no tiene
   serie). Cancelar-y-reservar no genera nada.
8. **El cliente cancela su propia serie completa.** Cero créditos (§0.3). Es
   una baja de suscripción, no una liberación de cupo.
9. **`END_OF_MONTH` con un servicio `ROLLING_MONTH`.** El "mes" del crédito es
   el **mes calendario de la clase liberada**, porque `END_OF_MONTH` es
   literalmente eso y es lo que el cliente describió. Para un servicio de mes
   corrido, el dueño debería elegir `END_OF_BILLING_PERIOD` (§5.4) y la UI
   tiene que sugerírselo cuando el plan del servicio es `ROLLING_MONTH` —
   **sugerir, no forzar**: no hay regla implícita que cambie el sentido de una
   política según el ciclo de cobro.
10. **Libera un cupo de un mes que todavía no pagó.** Para una fecha de serie
    es casi imposible: sin cobertura la fecha queda `NOT_GENERATED`, y las
    `NOT_GENERATED` no emiten. El resquicio real es una reserva confirmada cuyo
    pago se anuló después (ADR-0019 decidió a propósito que anular un pago no
    cancela clases confirmadas). La condición 4 de §2.3.2 —re-evaluar cobertura
    **en el momento de cancelar**— lo cierra: sin cobertura vigente, no hay
    crédito.
11. **Servicio gratuito (`payment_required = false`).** No emite nunca
    (condición 3). El crédito sólo compra precio, y ahí no hay precio que
    comprar; dar prioridad sí sería útil, pero prioridad es justamente lo que
    este ADR decidió no dar.
12. **El cliente tiene crédito y aun así quiere pagar.** Puede: `p_use_makeup_credit
    = false` (§2.4.5). Reserva como extra pagada y conserva el crédito.
13. **Cliente gestionado sin `profile_id` (ADR-0026, Fase J).** Nada se rompe:
    el crédito cuelga de `Customer`, no de `Profile`. Pero **no puede liberar su
    cupo por sí mismo** (no tiene sesión), así que el crédito lo gana cuando el
    mostrador cancela en su nombre con `CUSTOMER_REQUEST`. Y la policy de RLS
    del cliente sobre `makeup_credits` compara `profile_id = auth.uid()`: con
    `profile_id` nulo **no debe matchear nada** — está en la lista de policies
    que ADR-0026 mandó repasar, y ésta es una más.

---

## 7. Genericidad

Ninguna entidad, columna ni valor de enum de este diseño nombra un rubro. El
vocabulario es `Organization`, `Customer`, `Service`, `ServicePlan`,
`RecurringBooking`, `SlotOccurrence`, `Booking`, `Payment`, `MakeupCredit`.

**Consultorio.** Una paciente con control semanal (`WEEKLY_QUOTA` 1, martes
10:00). El consultorio pone `release_deadline_hours = 24` (una hora de
profesional se revende con más anticipación que una clase grupal) y
`makeup_credit_expiry = END_OF_MONTH`. Avisa el lunes que no puede → crédito →
toma el jueves 16:00, con el mismo profesional o con otro (el crédito es por
`Service`, no por `Resource`). Una **consulta suelta** que la paciente cancela
no genera nada: compró esa consulta. Si el **consultorio** cancela, sí genera,
aunque haya sido suelta (§2.3.3). Y `public_availability_display = BOOLEAN`
hace que el turno liberado **no** se destaque: nadie puede inferir que a la
psicóloga se le cayó el paciente de las 15:00 (§2.7.3).

**Cancha.** Un grupo con "martes 20:00 fijo" (`WEEKLY_QUOTA` 1). El club pone
`release_deadline_hours = 2` —una cancha se revende el mismo día— y
`makeup_credit_expiry = DAYS_AFTER 15`, porque su temporada no es mensual. El
grupo avisa a las 17:00 → crédito → juegan el domingo. Si el club cancela por
obras (`SLOT_CANCELLED`), crédito sin exigir anticipación. Un alquiler suelto
no genera nada.

**Dónde no cierra, dicho derecho:** para un negocio cuyo cliente **no vuelve**
(un salón de fiestas, un alquiler de una vez al año) el crédito de recupero no
tiene sentido — pero tampoco tiene sentido `WEEKLY_QUOTA`, o sea que el
mecanismo simplemente queda apagado (`makeup_credits_enabled = false`, que es
el default). Eso es configuración, no un límite del modelo.

---

## 8. Alternativas descartadas

| Alternativa | Por qué no |
|---|---|
| **Un contador `makeup_credits_remaining` en `Customer`** | Reintroduce exactamente la clase de bugs (decrementar, restituir, reconciliar) que ADR-0022 sacó al eliminar `CREDITS` y que ADR-0024 se cuidó de no reabrir. Además no puede tener vencimiento por crédito ni trazar de qué clase salió cada uno. |
| **Estado `EXPIRED` persistido + job de `pg_cron`** | Contradice el criterio de ADR-0010 para `COMPLETED`. Crea una ventana en la que el crédito venció y figura disponible, y la lectura tiene que chequear `expires_on` igual, o sea dos fuentes de verdad para el mismo hecho. |
| **Vencimiento evaluado contra `now()` (cuándo se reserva)** | "Recuperar dentro del mes" es que la clase **ocurra** en el mes. Reservar el 30/09 para el 15/10 no es recuperar en septiembre. |
| **Recalcular `expires_on` desde la política al leer** | Una política cambiada mañana le movería el vencimiento a un crédito ya entregado. Decisión 2 del Orchestrator, y es un reclamo de mostrador garantizado. |
| **Un valor `OK_WITH_MAKEUP_CREDIT` en `can_book_reason`** | Seis funciones comparan `= 'OK'`. Sería un "sí" que parte del motor lee como "no", en silencio y sólo para quien tiene crédito. |
| **Motivo de rechazo `MAKEUP_CREDIT_EXPIRED`** | El remedio es el mismo que `OUTSIDE_PLAN_QUOTA` (pagar el turno) y sugeriría que el crédito era el único camino. El dato va como contexto de pantalla, no como motivo. |
| **Prioridad del cupo liberado para quien tiene crédito** | Es una waitlist. Fuera de alcance desde ADR-0012, y traería la cola de reintentos asíncronos que ADR-0010 descartó explícitamente. Lo que cambia entre candidatos es el precio. |
| **Crédito multi-servicio o multi-organización** | Transferencia de valor entre listas de precios que el dueño no autorizó, y vuelve incontestable "¿de qué servicio es esto?" — la regresión que ADR-0022 arregló. Cross-organización, además, es frontera de tenant. |
| **Emitir en la cancelación de una serie completa** | El exploit de §0.3: cancelar la serie, recrearla y quedarse con los créditos. Convierte un plan de 1×semana en acceso ilimitado. |
| **Consumir créditos en `generate_recurring_booking()`** | Un job gastaría créditos sin que nadie mire, sobre fechas que ADR-0019 define como *pendientes, no decididas*, y que se confirmarían solas al registrarse el pago. |
| **Cobrar y reservar como dos operaciones (como hoy)** | Deja un `Payment PAID` sin `Booking` si el cupo se llena en el medio. Lo marcó `database-agent` y es correcto. |

---

## 9. Riesgos

1. **El camino de decisión se toca por segunda vez en dos fases.** ADR-0024 ya
   anotó que es el código más cargado del producto. La mitigación no es
   revisión: son los tests de concurrencia y de fecha que ya existen, más los
   cuatro que la Fase I declara obligatorios — **con la corrección de §0.1 al
   test de concurrencia**, que como está enunciado no prueba lo que dice.
2. **Riesgo de seguridad, no de diseño:** si §5.1 no se aprueba, la política de
   anticipación es voluntaria y el cliente se autoemite créditos. Es el riesgo
   número uno de esta ADR.
3. **El lock del crédito es un lock nuevo en el camino caliente.** Se toma
   *después* del de la ocurrencia, siempre en ese orden, en todos los
   llamadores. Orden inconsistente entre dos caminos = deadlock. Que quede
   escrito en el comentario de la función.
4. **`get_public_availability()` cambia de firma.** Phase 16 ya documentó que
   reescribir las reglas de disclosure de ADR-0008 de memoria es cómo cambian
   en silencio. Recrear la función **copiando** el cuerpo y agregando la
   columna, no reescribiéndolo.
5. **Riesgo de producto, no técnico:** los créditos crean demanda sobre los
   horarios buenos sin crear oferta. Si el estudio emite 20 y tiene 3 lugares,
   el reclamo aparece en el mostrador, no en los logs. Mitigación mínima:
   "créditos vivos por servicio y mes" en el panel admin.
6. **La palabra "crédito" va a hacer que alguien crea que volvió el paquete
   prepago** que el usuario eliminó por decisión explícita (§2.1.5).

---

## 10. Dónde vive cada validación (una sola fuente de verdad)

| Regla | Vive en | Por qué ahí |
|---|---|---|
| Emisión (las 6 condiciones), cálculo de `expires_on`, deadline | `issue_makeup_credit()` en SQL | Único llamado desde tres caminos de cancelación. Duplicarlo es ADR-0018 otra vez. |
| "Vino de una serie dentro de cuota" | **No se implementa**: se pregunta a `evaluate_payment_coverage()` | ADR-0024 ya la tiene. Reimplementarla es garantizar que diverjan. |
| Consumo, elección del crédito, atomicidad | `book_slot()` / `admin_book_for_customer()` bajo lock | Es correctness de concurrencia: en la base, no en la app (ADR-0004). |
| Devolución | La misma función de cancelación que ya toca la `Booking` | Un solo lugar donde se cancela ⇒ un solo lugar donde se devuelve. |
| Un crédito por liberación / uno por reserva | Índices únicos | Estructural, no convención. Sobrevive a que alguien agregue un camino nuevo. |
| Inmutabilidad de `expires_on` y del alcance | Trigger | La decisión 2 escrita donde se puede hacer cumplir. |
| Formato y coherencia de la política | CHECK + trigger en `organizations` / `services` | Columnas escribibles por PostgREST: Zod no es defensa (ADR-0020, ADR-0024). |
| "¿Venció?" | SQL, devuelto ya calculado por la RPC del portal | Si TS lo calcula también, un día no coinciden por timezone. TS **formatea**, no decide. |
| Etiquetas en castellano de origen/estado/camino de cobertura | `lib/` en TS, con test que falla ante palabras de rubro | Precedente de `lib/plan-labels` en la Fase H. Presentación, no regla. |
| Precio del turno suelto | Plan `DROP_IN` activo, leído en SQL | Un precio que viaja desde el cliente es un descuento del 100% esperando que alguien lo encuentre. |

---

## 11. Actualización de `docs/domain.md` que este ADR implica

1. **Entidad nueva** en "Entidades centrales":

   > **MakeupCredit** — el derecho a **una** reserva *extra* (ADR-0024) sobre un
   > `Service`, que nace de una reserva perdida: liberada por el `Customer` con
   > la anticipación que la `Organization` exige, o cancelada por la propia
   > organización (ahí sin exigir anticipación). Tiene vencimiento **congelado
   > al emitirse**. No es un paquete prepago: no se compra ni se recarga, y su
   > cantidad nunca supera la de reservas efectivamente perdidas. Genérico:
   > sirve igual en un consultorio, una cancha o un estudio.

2. **Invariantes nuevos**, en la sección de invariantes:
   - El stock de créditos de un cliente nunca supera la cantidad de reservas
     que efectivamente perdió.
   - Un crédito nunca cruza organizaciones ni servicios.
   - Un crédito se consulta **último**: nunca se gasta si otra cobertura
     alcanzaba.
   - Un crédito habilita una reserva **extra**; nunca una fecha de serie, y
     nunca da prioridad sobre el cupo.
   - El vencimiento se compara contra la **fecha local del turno**, no contra
     el momento de reservar.
   - `EXPIRED` **no se persiste**: es derivado de `expires_on`, igual que
     `COMPLETED` de `SlotOccurrence`.
   - Cancelar una reserva que consumió un crédito **lo devuelve**; nunca emite
     uno nuevo.
   - Cancelar una serie completa no emite créditos, la pida quien la pida.
   - Un cupo liberado vuelve a la agenda pública **por orden de llegada**: lo
     que cambia entre candidatos es el precio, no la prioridad. *(Cierra el
     punto que ADR-0019 dejó abierto — hay que editar ese párrafo, no sólo
     agregar éste.)*
   - Cobrar un turno suelto y reservarlo son **un solo hecho**: nunca queda un
     `Payment` sin su `Booking`.

3. **Corregir la ficha de `Booking`**: `cancelledBy` es una **FK a `Profile`**
   (quién exactamente), no el enum `CUSTOMER|ORGANIZATION` que el documento
   describe desde ADR-0010. El schema real lo hace así desde Phase 5 y lo
   documenta; `domain.md` quedó con la versión vieja. El actor se recupera de
   `cancellationReason`. **Esto importa más que un detalle de redacción**: es la
   razón por la que la emisión de créditos se apoya en el motivo y no en el
   actor, y por la que §0.2 es un agujero real.

4. **Matriz de auditoría**: agregar la fila de `MakeupCredit` —
   `created/updatedAt` sí, `createdBy` sí (`issued_by`), tripleta de
   cancelación **no**: tiene su propio concepto de baja (`REVOKED`), mismo
   criterio que `Payment` con `VOID`.

5. **Sección "Pago ≠ permiso"**: sumar el caso nuevo, que es el más claro de
   todos los que la sección lista — *el crédito de recupero es permiso sin pago
   presente, porque el pago ya ocurrió antes*.
