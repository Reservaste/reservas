# ADR-0024 — `ServicePlan`: qué compra un pago

Fecha: 2026-09-21
Estado: **Propuesta** (no aplicada — requiere aprobación del Orchestrator)
Propuesta por: `domain-architect`
Origen: feedback del primer cliente real (estudio de pilates), puntos 4, 5,
7 y 11 de `docs/iteration-3-plan.md`
Depende de: ADR-0022 (el `Service` como unidad de cobro), ADR-0013 (qué es
un "pago válido"), ADR-0018 (reserva fija), ADR-0019 (fecha pendiente),
ADR-0010 (recurrencia)
Habilita: ADR-0025 (crédito de recupero), ADR-0027 (tarjeta)

---

## 1. El problema

Hoy la regla es una sola línea, en `payment_covers_slot()`
(`20260921140000_phase15_booking_without_entitlements.sql`):

```
existe Payment PAID de (customer, service) cuyo período contiene
la fecha local del slot  ⇒  puede reservar
```

Eso significa que **un pago mensual compra acceso ilimitado al servicio
durante el período**. El cliente puede reservar los 5 días de la semana con
el mismo pago.

El estudio de pilates está describiendo otra cosa: paga **1800 por una vez
por semana, 2500 por dos, 3400 por tres, 800 por una clase suelta**. La
frecuencia es lo que se compra. Y agrega dos consecuencias que sólo tienen
sentido si la frecuencia es real: *"cada usuario paga una suscripción que
bloquea el espacio en ese horario específico"* y *"un usuario sin cupo
siempre paga"*.

Con la regla actual esos tres pedidos se contradicen: un plan de 1x/semana
no significa nada si el mismo pago ya habilitaba los 5 días, y "sin cupo
pagás" no significa nada si reservar de más nunca costó nada.

Falta además el lugar donde vive un precio con frecuencia. `services.price`
es **un** precio y `services.billing_type` es **un** ciclo; un servicio con
cuatro precios simultáneos —uno de ellos suelto y tres mensuales— no se
puede expresar con esas columnas.

### Lo que este ADR **no** resuelve

Liberar y recuperar un cupo (puntos 8, 9, 10) es ADR-0025. Este ADR sólo
tiene que dejar bien definido **qué es una reserva extra**, porque es lo que
el crédito de recupero va a habilitar.

---

## 2. La decisión

### 2.1 Tabla nueva: `service_plans`

`public.plans` ya existe y son los planes del SaaS que paga la
organización (ADR de Phase 10). Esta es la lista de precios que la
organización le ofrece **a sus clientes**, y no lleva ninguna palabra de
rubro.

```sql
create type public.service_plan_kind as enum ('DROP_IN', 'WEEKLY_QUOTA', 'UNLIMITED');

create table public.service_plans (
  id               uuid primary key default gen_random_uuid(),
  organization_id  uuid not null references public.organizations (id) on delete cascade,
  service_id       uuid not null references public.services (id) on delete cascade,
  name             text not null,
  description      text,
  price            numeric(12, 2) not null,
  plan_kind        public.service_plan_kind not null,
  -- Sólo WEEKLY_QUOTA lo usa. La coherencia la garantiza el CHECK, no la UI.
  weekly_quota     int,
  -- Qué período compra un pago de este plan. Sube del Service al plan
  -- porque un mismo Service tiene ahora un plan ONE_TIME y tres MONTHLY.
  billing_type     public.billing_type not null,
  billing_cycle    public.billing_cycle,
  is_active        boolean not null default true,
  sort_order       int not null default 0,
  created_at       timestamptz not null default now(),
  updated_at       timestamptz not null default now(),
  created_by       uuid references public.profiles (id),
  cancelled_at     timestamptz,
  cancelled_by     uuid references public.profiles (id)
);
```

Cada invariante y el CHECK que la expresa:

| Invariante | Expresión |
|---|---|
| `weekly_quota` existe **si y sólo si** el plan es `WEEKLY_QUOTA`, y vale al menos 1 | `check ((plan_kind = 'WEEKLY_QUOTA' and weekly_quota is not null and weekly_quota >= 1) or (plan_kind <> 'WEEKLY_QUOTA' and weekly_quota is null))` |
| Un precio negativo no es un precio | `check (price >= 0)` |
| Sólo las tres combinaciones kind↔cobro que el producto soporta | `check ((plan_kind = 'DROP_IN' and billing_type = 'ONE_TIME' and billing_cycle is null) or (plan_kind in ('WEEKLY_QUOTA','UNLIMITED') and billing_type = 'MONTHLY' and billing_cycle is not null))` |
| `cancelled_at`/`cancelled_by` van juntos y sólo en un plan inactivo | `check ((cancelled_at is null and cancelled_by is null) or (cancelled_at is not null and not is_active))` |
| Dos planes activos con el mismo nombre en un servicio son un error de carga, no una opción | `unique index (service_id, lower(name)) where is_active` |
| **Un solo plan `DROP_IN` activo por servicio** — es el precio que el flujo de "pagá esta clase" cobra automáticamente; dos vuelven esa elección ambigua | `unique index (service_id) where is_active and plan_kind = 'DROP_IN'` |
| **Un solo plan `UNLIMITED` activo por servicio** — es el plan al que el puente de migración ancla un pago sin plan explícito (§2.6) | `unique index (service_id) where is_active and plan_kind = 'UNLIMITED'` |
| El plan y su servicio pertenecen a la misma organización | trigger `check_service_plan_same_org()`, mismo patrón que `check_payment_same_org()` — **no puede ser un CHECK**, cruza tablas, y la tabla es escribible por PostgREST |

**Sin tope superior para `weekly_quota` a propósito.** Un servicio puede
tener dos `ScheduleRule` el mismo día (pilates lunes 09:00 y lunes 18:00),
así que un `<= 7` sería incorrecto.

**`price` es editable siempre; `plan_kind` y `weekly_quota` no lo son una
vez que el plan tiene algún `Payment` no-`VOID`** (trigger). Son los dos
campos que el camino de reserva lee: cambiarlos le cambia retroactivamente
a todo el mundo qué compró el mes pasado. Mismo criterio que ADR-0025 le
aplica al vencimiento de un crédito ("el valor se congela al emitirlo").
La salida cuando el dueño se equivocó: desactivar el plan y crear otro.

**Sin `cancellation_reason`.** La matriz de auditoría de ADR-0010 la pide
donde hay varios motivos posibles; acá hay uno solo ("el dueño lo sacó de
la lista") y un enum de un valor es ruido.

**`sort_order`** existe porque las cuatro filas del pilates son una
escalera de precios y el orden es editorial: "clase individual 800" primero
no es ni alfabético ni por precio.

### 2.2 Taxonomía de `plan_kind`: se confirma, con una corrección a la justificación

`DROP_IN | WEEKLY_QUOTA | UNLIMITED` es correcta **para los tres casos que
el cliente pidió**, y `UNLIMITED` cumple bien su función: el
comportamiento de hoy pasa a ser un plan con nombre y nadie cambia de
comportamiento el día del deploy (mismo patrón aditivo de ADR-0022).

**Pero la taxonomía no cubre uno de los dos ejemplos de genericidad de
`iteration-3-plan.md`, y eso hay que corregirlo en el plan, no taparlo:**
**"un consultorio: 4 consultas al mes" NO entra en este modelo.** Es una
cantidad por mes sin horario fijo, es decir un **contador** sobre un
período — no un conjunto de horarios reservados. No es un parámetro
distinto del mismo mecanismo: es el mecanismo de paquete prepago que
ADR-0022 eliminó explícitamente (`entitlement_type = 'CREDITS'`). Meterlo
acá obligaría a que `evaluate_customer_booking()` tenga **dos** caminos de
cuota —uno por series y otro por conteo— que es exactamente donde nacen
los bugs de cupo.

Recomendación: `MONTHLY_QUOTA` queda **fuera de alcance**, con su propio
ADR el día que un cliente lo pida, y la genericidad de este modelo se
enuncia por lo que realmente cubre (§7).

Lo que **sí** queda abierto sin costo: un pase de día (una sola compra,
uso libre ese día) es `UNLIMITED` + `billing_type = 'ONE_TIME'`, y un plan
trimestral es `WEEKLY_QUOTA` + un ciclo nuevo. Los dos se habilitan
relajando un CHECK, **sin agregar un valor a `plan_kind`** — que es
justamente la razón por la que `plan_kind` y `billing_type` se mantienen
como dos columnas y no se colapsan en una (hoy están correlacionadas; las
preguntas que contestan no son la misma: una es "qué derecho de reserva
compra", la otra es "qué período cubre el pago").

### 2.3 Relación con `Service`: **1 servicio → N planes** (`service_id NOT NULL`)

Un plan pertenece a exactamente un `Service`. No hay "pase full" en este
ADR.

Por qué:

- Todo el camino de decisión y de plata está cableado por servicio:
  `payment_covers_slot(customer, service, slot)`, el `EXCLUDE` de
  `(customer_id, service_id, rango)`, `organization_payment_summary()`,
  `customer_payment_detail()`, `my_services()`. Un plan multi-servicio
  vuelve **inconstestable** "¿de qué servicio es este pago?", que es
  exactamente la regresión que ADR-0022 arregló al matar
  `ServiceEntitlement` (que era justamente la indirección multi-servicio).
- La cuota es por servicio: 2 series de Pilates. Un pase full con cuota 3
  sobre Pilates + Funcional abre una pregunta que nadie contestó: ¿son 3
  compartidas o 3 de cada uno? Adivinar es peor que no tenerlo.

**Qué se rompe el día que haga falta el pase full** (explícito, para que no
sea una sorpresa):

1. `payments.service_id NOT NULL` deja de ser cierto.
2. El `EXCLUDE` sobre `(customer_id, service_id, rango)` **no se puede
   expresar**: o el pase genera N filas de pago (doble contabilidad) o el
   `EXCLUDE` se reancla a `(customer_id, service_plan_id, rango)`, que es
   **más débil** — dos planes distintos del mismo servicio con períodos
   solapados dejan de detectarse como doble cobro.
3. `payment_covers_slot()` cambia de contrato: necesita resolver "¿algún
   plan pago de este cliente cubre este servicio?" vía una tabla N:M.
4. Hay que decidir si la cuota es un pozo compartido o por servicio.
5. Las pantallas de pago y `my_services()` agrupan por servicio; un pase
   full aparece N veces o necesita su propia agrupación.

**Cobertura barata que se toma ahora para abaratar ese día:**
`payments.service_plan_id` es el **ancla de registro** y
`payments.service_id` queda como columna **derivada y verificada** desde el
plan (§2.6). Así, el día del pase full, `service_id` se vuelve nullable y
sólo hay que rehacer el `EXCLUDE` y `payment_covers_slot()`; ninguna
lectura lo está usando como fuente de verdad.

### 2.4 La frecuencia: la hipótesis es correcta, y hay tres puntos donde falta precisión

> "Los N cupos del plan **son** N `RecurringBooking` contra N
> `ScheduleRule`, así que `weekly_quota` es el máximo de series activas y
> no un contador que haya que chequear en cada reserva."

Se valida, y el argumento fuerte es más lindo de lo que parece:
**cada `ScheduleRule` es semanal por construcción** (`weekday` +
`local_start_time`) y cada `RecurringBooking` es la suscripción a **una**
regla, con `ALREADY_HAS_STANDING_RESERVATION` impidiendo dos series sobre
la misma regla. Entonces *k* series en vigencia producen exactamente *k*
reservas por semana. "Veces por semana" **es** "cantidad de series", no una
aproximación. Consecuencias:

- No se persiste ningún contador. El invariante
  `availableCapacity = maxCapacity - activeBookings` no se toca.
- No hay que decrementar, restituir al cancelar, ni reconciliar un
  contador — toda la clase de bugs que ADR-0022 eliminó al borrar
  `CREDITS`.
- **La pregunta "semana calendario o semana corrida" se disuelve**: nunca
  se calcula una ventana semanal, así que no hay nada que anclar (§6.e).

Las tres precisiones que faltan y sin las cuales la implementación sale
mal:

**(a) `status = 'ACTIVE'` no alcanza: sobrecuenta.** ADR-0010 dejó
`RecurringBooking.status` como expresión de intención, no agregado de sus
hijas — una serie con `end_date` en junio sigue `ACTIVE` en septiembre.
La cuota se compara contra las series **en vigencia para la fecha que se
está evaluando**:

```
series en vigencia para D, del cliente C, en el servicio S =
  recurring_bookings rb
  join schedule_rules sr on sr.id = rb.schedule_rule_id
  where rb.customer_id = C
    and sr.service_id  = S
    and rb.status = 'ACTIVE'
    and rb.start_date <= D
    and (rb.end_date is null or rb.end_date >= D)
```

**(b) `D` es la fecha local del slot, nunca `now()`.** Es la misma lección
de ADR-0013 aplicada a la cuota: una serie que arranca el mes que viene no
consume cuota hoy, pero sí la consume para un turno del mes que viene.
Evaluar la cuota contra "hoy" es el bug fácil, igual que evaluar el pago
contra "hoy".

**(c) Cuando hay más series en vigencia que cuota, hace falta un desempate
determinístico, no una decisión humana.** Orden `created_at asc, id asc`:
entran las primeras `weekly_quota` series que el cliente contrató. Es
estable, explicable en el mostrador ("las dos primeras que contrataste") y
no obliga a nadie a elegir cuál se cae al bajar de plan (§6.b).

**Nota: `RecurringBooking` no gana ninguna columna.** El servicio se
alcanza por `schedule_rule_id → schedule_rules.service_id`; copiarlo sería
la segunda fuente de verdad que ADR-0010 prohibió para día/hora.

**Dónde se aplica la cuota — dos puertas, con pesos distintos:**

- **Puerta blanda, al crear la serie** (`create_recurring_booking`,
  `admin_create_recurring_booking`): si el cliente **tiene** un pago en
  vigencia y su plan es `WEEKLY_QUOTA`, crear la serie N+1 se rechaza en
  la RPC con `OVER_PLAN_QUOTA`. Si **no tiene** ningún pago en vigencia, la
  creación **no se rechaza**: ese es el flujo de mostrador de ADR-0018/0019
  (se arma la reserva fija y se cobra después), y romperlo sería una
  regresión. La cuota que se usa es la del pago en vigencia en
  `greatest(series.start_date, current_date)`.
- **Puerta dura, por fecha, al generar** (`generate_recurring_booking`,
  `retry_not_generated_booking`): es la única que puede ser exacta, porque
  recién ahí "qué plan pagó para ese mes" tiene respuesta definida. Una
  fecha de una serie fuera de cuota queda `NOT_GENERATED` con
  `not_generated_reason = 'OVER_PLAN_QUOTA'`.

No puede ser un constraint de base: es un agregado que cruza
`recurring_bookings → schedule_rules → services` contra un `weekly_quota`
al que se llega por `payments → service_plans`. **Implicancia para
`database-agent`**: hoy todas las escrituras de `recurring_bookings` pasan
por RPC (la RLS es SELECT-only), así que el chequeo en la RPC alcanza; si
en algún momento se abre un `INSERT` por PostgREST, hace falta además un
trigger `BEFORE INSERT`.

### 2.5 Qué es una reserva "extra", y la nueva regla de decisión

**Definiciones.** Para un `Customer` C, un `Service` S con
`payment_required = true`, y un slot cuya fecha local es D:

- Una reserva es **de serie** si tiene `recurring_booking_id` no nulo, esa
  serie está en vigencia para D (§2.4a) y su posición en el orden de §2.4c
  es `<= weekly_quota`.
- Una reserva es **extra** en cualquier otro caso: toda reserva puntual
  (`recurring_booking_id is null`) bajo un plan `WEEKLY_QUOTA`, y toda
  fecha de una serie fuera de cuota.
- Bajo `UNLIMITED` **no existe el concepto de extra** — es el
  comportamiento de hoy, intacto.
- Bajo `DROP_IN` toda reserva es extra por definición y se paga una por
  una.

**`payment_covers_slot()` — nueva semántica, en prosa precisa.** Gana un
cuarto parámetro opcional, así que ningún llamador existente deja de
compilar y `default null` significa literalmente "esta reserva no viene de
una serie":

```sql
payment_covers_slot(
  p_customer_id uuid,
  p_service_id uuid,
  p_slot_occurrence_id uuid,
  p_recurring_booking_id uuid default null
) returns boolean
```

1. Si `service.payment_required = false` → **true**. Sin cambios: es lo que
   hace que un servicio gratis sea gratis y que la cuota no se pueda
   fingir sobre uno (§6.i).
2. `D := slot_local_date(slot)`. Si es null → **false**.
3. **Pago de clase suelta primero**: si existe un `Payment` `PAID` con
   `slot_occurrence_id` = este slot y `customer_id` = C → **true**. Se
   consulta antes que todo lo demás porque es el más específico: una clase
   pagada está pagada, independientemente del estado de cualquier plan.
4. Buscar el **pago por período en vigencia para D**: `PAID`,
   `period_start <= D <= period_end`, de `(C, S)`, con
   `slot_occurrence_id is null`. Si no hay ninguno → **false**
   (`PAYMENT_REQUIRED`: falta pagar el mes, el diagnóstico de hoy).
5. Leer `plan_kind` del `service_plan` de ese pago — **sin filtrar por
   `is_active`** (§6.d):
   - `UNLIMITED` → **true**.
   - `WEEKLY_QUOTA` → **true** si y sólo si `p_recurring_booking_id` no es
     nulo **y** esa serie está en vigencia para D **y** su posición en el
     orden de §2.4c es `<= weekly_quota`. En cualquier otro caso →
     **false**.
   - `DROP_IN` como pago por período → **false** (defensivo: el CHECK de
     §2.1 lo hace imposible, un pago así es dato corrupto y no debe
     habilitar nada).

**`evaluate_customer_booking()` — nueva semántica.** Sigue siendo **una
sola función** para el preview del mostrador, el del cliente y el confirm
(ADR-0018: duplicarla es cómo se llega a "el preview decía que sí y el
confirm dijo que no"). Gana el mismo contexto de serie, con tres formas:

- **sin serie** (reserva puntual): la que usan `book_slot()` y
  `admin_book_for_customer()`.
- **serie existente** (`p_recurring_booking_id`): la que usa
  `generate_recurring_booking()` / `retry_not_generated_booking()`.
- **serie prospectiva**: la que necesita `preview_recurring_booking()`
  antes de que la serie exista — se evalúa como si la serie fuera la
  (k+1)-ésima en vigencia.

> **Regresión concreta a evitar, y no es teórica:** hoy
> `preview_recurring_booking()` llama a `can_customer_book(occurrence)`,
> que es de forma "reserva puntual". Si se le agrega la cuota sin hilar el
> contexto de serie, **el preview de una reserva fija va a devolver
> "fuera de tu frecuencia" en todas las fechas** de un cliente que tiene
> justamente el plan que la permite.

Orden de los pasos (el actual, con el paso de pago abriéndose en tres):
ocurrencia activa y no terminada → organización activa → servicio activo →
es cliente → **cobertura** → cupo → duplicado.

**Motivos nuevos de `can_book_reason`, y por qué `PAYMENT_REQUIRED` no
alcanza:**

| Valor | Significa | Qué se hace al respecto |
|---|---|---|
| `PAYMENT_REQUIRED` (ya existe) | No hay ningún pago que cubra esa fecha | "Cobrale el mes" |
| **`OUTSIDE_PLAN_QUOTA`** (nuevo) | El mes **está pago**, pero esta reserva no viene de uno de sus horarios fijos | Pagar la clase suelta, o usar un crédito de recupero (ADR-0025), o esperar |
| **`OVER_PLAN_QUOTA`** (nuevo) | Esta **serie** excede la frecuencia del plan | Subir de plan, o dar de baja una serie |
| **`SERVICE_HAS_NO_PLAN`** (nuevo) | El servicio exige pago y no tiene ningún plan activo | Es un error de configuración del dueño, no del cliente |

`PAYMENT_REQUIRED` no alcanza porque **le diría "tenés que pagar" a alguien
que acaba de pagar**. Es exactamente el error que ADR-0018 ya tuvo que
corregir una vez con `not_generated_reason` (`NOT_GENERATED` mezclaba "está
llena" con "el mes no está pago", y el portal mostraba "sin lugar" para los
dos) y que Phase 12 volvió a corregir. `OUTSIDE_PLAN_QUOTA` y
`OVER_PLAN_QUOTA` tampoco son el mismo mensaje: cambian de actor (el
cliente intentando de más vs. el mostrador habiendo vendido de más), de
pantalla y de remedio.

`not_generated_reason` suma **`OVER_PLAN_QUOTA`** (es el enum que va en
`bookings`). `OUTSIDE_PLAN_QUOTA` **no** hace falta ahí: una fecha
`NOT_GENERATED` siempre viene de una serie.

**Trampa para `database-agent`:** `ALTER TYPE ... ADD VALUE` y usar el
valor nuevo en la misma migración funciona dentro del cuerpo de una función
(así se agregó `PAYMENT_REQUIRED` en Phase 7) pero **falla si el valor se
usa en un CHECK o en un DML de la misma transacción**.

### 2.6 Anclaje del pago

```sql
alter table public.payments
  add column service_plan_id uuid references public.service_plans (id) on delete restrict,
  add column slot_occurrence_id uuid references public.slot_occurrences (id) on delete cascade;
```

- **`service_plan_id` es el ancla de registro**, y queda `NOT NULL`
  después del backfill (§4). `on delete restrict`: los planes se
  desactivan, no se borran, y un plan con pagos no puede desaparecer.
- **`payments.service_id` se mantiene**, como columna **derivada y
  verificada**, no como segunda fuente de verdad. Motivo: `EXCLUDE`,
  `payments_customer_service_idx`, `organization_payment_summary()`,
  `customer_payment_detail()`, `my_payments()` y `my_services()` están
  todos cableados por servicio, y sigue siendo el grano correcto de "quién
  pagó este mes" (todo el punto de ADR-0022). Un trigger valida
  `payments.service_id = service_plans.service_id`, así que no puede
  divergir. Dropearla rompería el módulo de Pagos sin ganar nada.
- **`slot_occurrence_id`** es no nulo exactamente para los pagos de plan
  `DROP_IN` (§2.7). Sus `period_start`/`period_end` se setean los dos a la
  fecha local del slot, para que un pago suelto siga apareciendo en el mes
  correcto en `organization_payment_summary()` (que filtra por solapamiento
  de período) sin tocar esa función.
- **`billing_period_for()` se muda de `service_id` a `service_plan_id`.**
  Es donde vive la diferencia entre mes calendario y mes corrido, y esa
  configuración ahora es del plan. Barato: hoy sólo la usan los tests
  (`test/phase7.payments.test.ts`) — el formulario de pago manda
  `periodStart`/`periodEnd` desde el form.
- `services.billing_type`, `services.billing_cycle` y `services.price`
  quedan **deprecadas** (se conservan, dejan de leerse en el camino de
  decisión y de cobro). **Esto falta en `iteration-3-plan.md` y es la
  omisión más importante del shape que proponía**: sin mover el ciclo de
  facturación al plan, quedan dos fuentes de verdad para "qué período
  compra un pago", una por servicio y otra por plan, y van a divergir el
  primer día que un servicio tenga un plan calendario y otro corrido.
  `services.payment_required` **no** se toca: es el interruptor de "¿este
  servicio exige pago para reservar?", no un precio.

**Invariantes que cruzan tablas y por eso son trigger, no CHECK**
(`check_payment_plan_consistency()`): `plan.organization_id =
payment.organization_id`; `plan.service_id = payment.service_id`;
`(plan.plan_kind = 'DROP_IN') = (slot_occurrence_id is not null)`; y si hay
`slot_occurrence_id`, que esa ocurrencia sea del mismo `organization_id` y
del mismo `service_id` — sin eso, un insert armado a mano paga la clase de
un servicio con un pago rotulado a otro.

### 2.7 El conflicto con el `EXCLUDE`: la hipótesis se valida, y le falta una mitad

El diagnóstico del plan es correcto. Hoy:

```sql
exclude using gist (customer_id with =, service_id with =,
                    daterange(period_start, period_end, '[]') with &&)
  where (status = 'PAID')
```

Alguien con plan mensual que compra una clase suelta cae dentro de su
propio período y el insert se rechaza. La solución propuesta —anclar el
pago suelto a la `SlotOccurrence` y dejar el `EXCLUDE` sólo para los pagos
por período— es la correcta, y además es **más fuerte** que la alternativa
de un período de un día `[D, D]`: dice *qué clase* se pagó, que es
exactamente lo que el flujo "sin cupo pagás" necesita saber, y un período
de un día no distingue dos clases del mismo día.

```sql
-- Los pagos por período siguen protegidos igual que antes.
alter table public.payments drop constraint payments_no_overlapping_paid;
alter table public.payments add constraint payments_no_overlapping_paid
  exclude using gist (
    customer_id with =, service_id with =,
    daterange(period_start, period_end, '[]') with &&
  ) where (status = 'PAID' and slot_occurrence_id is null);
```

**La mitad que falta:** con ese `WHERE`, los pagos sueltos se quedan **sin
ninguna** protección de doble cobro — justo el tipo de pago nuevo. Pagar
dos veces la misma clase es un doble cobro con la misma gravedad, sólo con
otra forma:

```sql
create unique index payments_one_paid_per_occurrence_idx
  on public.payments (customer_id, slot_occurrence_id)
  where status = 'PAID' and slot_occurrence_id is not null;
```

Alternativa evaluada y descartada: reanclar el `EXCLUDE` a
`(customer_id, service_plan_id, rango)`. Deja pasar dos planes mensuales
distintos del mismo servicio para el mismo mes, que es un doble cobro real
y hoy se detecta.

**Consecuencia que el plan no menciona y hay que decidir (§5):** incluso
con esto arreglado, **subir de plan a mitad de período sigue bloqueado por
el `EXCLUDE`**. Cobrar los 900 de diferencia entre 2x y 3x para el mes en
curso es un segundo pago por período solapado. Recomendación: **VOID +
volver a cargar el mes al precio nuevo**, con la explicación en
`payments.notes`. Motivo: dos pagos `PAID` por período del mismo servicio
para el mismo mes vuelven ambiguo el paso 5 de `payment_covers_slot()`
(¿qué plan aplica, el de mayor cuota, el último cargado, los dos?), y
resolver esa ambigüedad en el camino de decisión es peor que hacer que el
mostrador anule y recargue. Si el dueño necesita el registro en dos
cuotas, eso es un modelo de `payment_lines`/`payment_installments` y su
propio ADR (emparienta naturalmente con `payment_intents` de ADR-0027).

---

## 3. El caso pilates, completo

Un solo `Service` "Pilates" con `payment_required = true` y
`schedule_rules.capacity = 3` (la capacidad ya funciona desde Phase 3 y es
**ortogonal** al plan: un plan nunca otorga cupo, otorga el derecho a
tener series o a reservar). Cuatro `service_plans`:

| name | plan_kind | weekly_quota | billing_type | price |
|---|---|---|---|---|
| Clase individual | `DROP_IN` | — | `ONE_TIME` | 800 |
| 1 vez por semana | `WEEKLY_QUOTA` | 1 | `MONTHLY` | 1800 |
| 2 veces por semana | `WEEKLY_QUOTA` | 2 | `MONTHLY` | 2500 |
| 3 veces por semana | `WEEKLY_QUOTA` | 3 | `MONTHLY` | 3400 |

Un cliente de 2x paga 2500 por octubre y el mostrador le arma dos reservas
fijas (lunes 09:00, miércoles 09:00). Las dos confirman todas sus fechas de
octubre. Si quiere el viernes también: sin plan 3x, `OUTSIDE_PLAN_QUOTA`, y
paga 800 (o usa un crédito de recupero cuando exista ADR-0025). Noviembre
sin pagar: cada fecha queda `PAYMENT_REQUIRED` sola (ADR-0018), y al
cobrarse se confirman solas (ADR-0019). Nada de eso es nuevo.

Confirmo además la nota del plan: modelar "1x" y "2x" como dos `Service`
partiría la capacidad de 3 en dos cupos separados. Es la razón correcta.

---

## 4. Migración

Aditiva y en el mismo orden que ADR-0022, que es el que no rompió 16 tests
de una vez.

1. Enum `service_plan_kind`; tabla `service_plans` con sus CHECKs, índices
   parciales y trigger de organización.
2. `payments.service_plan_id` (nullable), `payments.slot_occurrence_id`
   (nullable).
3. **Backfill**: para cada `Service` que hoy tenga al menos un `Payment` o
   `payment_required = true`, crear **un** plan `UNLIMITED` con el `name`
   del servicio, su `price`, su `billing_type`/`billing_cycle` actuales (o
   `MONTHLY`/`CALENDAR_MONTH` si son nulos) y `is_active = true`. Luego
   `update payments set service_plan_id = <ese plan>` por `service_id`.
   **Nadie cambia de comportamiento el día del deploy**: todos los pagos
   existentes quedan `UNLIMITED` y el paso 5 de `payment_covers_slot()`
   devuelve `true` igual que hoy.
4. `payments.service_plan_id` a `NOT NULL`, **después** del puente del
   punto 5.
5. **Puente de compatibilidad**, mismo patrón que
   `fill_payment_service_from_entitlement()` de ADR-0022, y necesario
   porque el formulario actual inserta en `payments` por PostgREST directo
   (`frontend/app/actions/billing.ts`) sin saber de planes: trigger
   `BEFORE INSERT` que, si `service_plan_id` es nulo, lo completa con el
   plan `UNLIMITED` activo del servicio; si no hay ninguno, **levanta
   `PAYMENT_REQUIRES_PLAN`** en vez de dejar la fila sin plan. Fallar
   ruidoso es correcto acá: un pago sin plan que el camino de decisión
   tuviera que interpretar como "ilimitado por las dudas" es la peor de las
   dos opciones. El índice único de un solo `UNLIMITED` activo por servicio
   (§2.1) es lo que hace determinístico a este puente. Muere en la fase en
   que la UI de pagos manda el plan explícito.
6. `payment_covers_slot()` con el cuarto parámetro; `evaluate_customer_booking()`
   con el contexto de serie; `generate_recurring_booking()`,
   `retry_not_generated_booking()`, `preview_recurring_booking()`,
   `book_slot()`, `admin_book_for_customer()`,
   `admin_create_recurring_booking()` y `create_recurring_booking()`
   actualizados para pasar (o no) ese contexto.
7. `ALTER TYPE can_book_reason ADD VALUE` ×3;
   `ALTER TYPE not_generated_reason ADD VALUE 'OVER_PLAN_QUOTA'`.
8. Re-anclaje del `EXCLUDE` + índice único de pago suelto por ocurrencia.
9. `billing_period_for(service_plan_id, from)`.
10. RLS de `service_plans`: **`SELECT` público** (es una lista de precios;
    la página pública del negocio la muestra, igual que `plans` del SaaS es
    legible por cualquiera), escritura sólo para `is_organization_member`.
    Esto se lo confirma `auth-security-agent`: un precio es público, pero
    hay que verificar que ninguna vista pública arrastre por join datos
    privados del cliente que compró ese plan.
11. `comment on column` marcando `services.billing_type`,
    `services.billing_cycle` y `services.price` como deprecadas.

**Compatibilidad hacia atrás:** un pago sin `service_plan_id` no existe
después del paso 4, así que el camino de decisión **no** tiene rama de
legado. `services.payment_required = false` sigue cortocircuitando todo en
el paso 1. El único cambio de comportamiento observable para un cliente
existente es que el servicio ahora tiene un plan con nombre visible.

**Restricción de secuenciación para el Orchestrator:** la migración y la
pantalla de pagos con selector de plan **tienen que salir en la misma
fase**. Con el puente del paso 5 la UI vieja sigue funcionando mientras
exista un `UNLIMITED` activo, pero en cuanto el dueño desactive ese plan
para dejar sólo los de cuota, el mostrador no puede registrar un pago.

---

## 5. Lo que NO pude decidir y necesita al Orchestrator

1. **Cambio de plan a mitad de período**: VOID + recargar (mi
   recomendación, §2.7) vs. permitir filas de cuota relajando el `EXCLUDE`.
   Es política de registro de plata, no de dominio.
2. **¿Entra `MONTHLY_QUOTA` / paquete por cantidad en el alcance?**
   Recomiendo que no (§2.2), pero eso implica **corregir el ejemplo del
   consultorio en `iteration-3-plan.md`**, y reabre la eliminación de
   `CREDITS` de ADR-0022.
3. **Al bajar de plan, ¿se cancelan las series excedentes automáticamente?**
   Recomiendo que **no** (§6.b): se autolimitan solas y el dueño decide.
   Es una política operativa del mismo tipo que el punto abierto de
   ADR-0019.
4. **Inmutabilidad de `plan_kind`/`weekly_quota` con pagos asociados**
   (§2.1) vs. una columna `payments.weekly_quota_snapshot`. Elegí
   inmutabilidad (una columna menos, sin riesgo de desincronización, misma
   semántica retroactiva), pero le impone al dueño "desactivar y crear
   otro" para corregir un tipeo.
5. **¿Puede un servicio con `payment_required = false` tener planes de
   cuota?** Recomiendo prohibirlo por trigger (§6.i): la cuota se apoya en
   la cobertura de pago, así que sobre un servicio gratis es
   inaplicable y aparentaría estar vigente.
6. **Moneda.** `service_plans.price` no la tiene, igual que
   `services.price` hoy. Con un solo cliente en Uruguay no molesta; es un
   hueco conocido, no una decisión que quiera tomar sola.
7. **Contrato exacto de la firma con contexto de serie** (un `uuid` +
   un `boolean` de prospectiva, o un parámetro de modo). Es detalle de
   `database-agent`; lo que **no** es negociable es que siga habiendo una
   sola función de decisión (ADR-0018).

---

## 6. Casos límite — contestados, no dejados abiertos

**(a) Cliente que sube de 2x a 3x a mitad de mes.** Para el **período
siguiente**: se carga un pago del plan 3x con el período nuevo; no hay
solapamiento, no hay conflicto, y la tercera serie se puede crear en el
acto. Sus fechas ya materializadas dentro del período pago las confirma
**la reconciliación que ya existe** (trigger de ADR-0019 sobre
`payments`), sin maquinaria nueva — sólo hay que asegurarse de que
`reconcile_pending_recurring_bookings()` reintente también las fechas con
`OVER_PLAN_QUOTA` y no sólo las de `PAYMENT_REQUIRED` (hoy reintenta todas
las `NOT_GENERATED`, así que ya está bien). Para el **período en curso**:
requiere VOID + recarga (§5.1).

**(b) Cliente que baja de 3x a 1x teniendo 3 series en vigencia.**
**No se le cancela ninguna serie.** Las tres siguen `ACTIVE`; a partir del
período con el plan 1x, la primera por `created_at` sigue confirmando y las
otras dos producen `NOT_GENERATED / OVER_PLAN_QUOTA`. Es la misma "serie
autolimitada" de ADR-0018, aplicada a la cuota en vez de al pago, y si
vuelve a subir, ADR-0019 las reconstituye solas. Tres consecuencias que
hay que decir en voz alta:

- **No hay fuga de cupo**: una `Booking NOT_GENERATED` no consume
  capacidad, así que esos lugares quedan libres para otros clientes.
- **Las fechas ya `CONFIRMED` del período ya pago no se cancelan.**
  Simétrico con ADR-0019: una escritura que **restringe** nunca cancela
  retroactivamente. Y no hay ventana de 90 días de regalo, porque una
  serie sólo confirma fechas que un pago cubre: fuera del período pago las
  fechas ya venían `NOT_GENERATED`.
- **La UI de admin tiene que mostrarlo**: "esta persona tiene 3 series y su
  plan cubre 1". `schedule_rule_standing_reservations()` ya separa
  `upcoming_unpaid`; necesita el equivalente para over-quota, si no el
  dueño no se entera de que vendió menos de lo que agendó.

**(c) Plan que vence impago con series activas.** Sin cambios respecto de
hoy: cada fecha futura queda `NOT_GENERATED / PAYMENT_REQUIRED`
(ADR-0018), y al registrarse el pago se confirman solas (ADR-0019). Este
ADR no agrega ni un estado de "suspendida".

**(d) Plan desactivado con clientes que todavía lo pagaron.**
`is_active = false` significa **"no se ofrece más"**, nunca "los pagos
viejos dejan de valer". El paso 5 de `payment_covers_slot()` lee
`plan_kind`/`weekly_quota` **sin filtrar por `is_active`**. Es una trampa
real: un `join service_plans sp on ... and sp.is_active` le sacaría la
cobertura a todos los clientes en el momento en que el dueño ordena su
lista de precios. Queda como invariante escrita, no como cuidado del
implementador.

**(e) ¿`weekly_quota` se mide por semana calendario o por semana corrida?**
**Por ninguna de las dos: no se mide sobre una semana.** Es la cardinalidad
de las series en vigencia (§2.4), y como cada `ScheduleRule` es semanal,
*k* series son *k* reservas por semana sin que nadie calcule una ventana.
Dos corolarios:

- Una semana con feriado (excepción que cancela esa fecha) da **menos** de
  N, y está bien: eso es precisamente lo que el crédito de recupero de
  ADR-0025 viene a compensar. Un mes con 5 lunes da 5, y también está
  bien: es lo que significa "por semana".
- Si alguna vez se muestra "usaste 2 de 2 esta semana", ese texto **tiene
  que derivarse de las series**, no de contar reservas en una ventana
  semanal: en la semana del feriado diría "1 de 2 usadas" e invitaría a la
  conclusión equivocada. El día que entre `MONTHLY_QUOTA`, *ese* plan sí
  necesita la decisión calendario/corrido, y el precedente ya existe
  (`billing_cycle`).

**(f) Cuota 3 sobre un servicio con sólo 2 horarios por semana.** Se puede
vender y no se puede cumplir: hacen falta 3 `ScheduleRule` distintas
porque `ALREADY_HAS_STANDING_RESERVATION` impide dos series sobre la misma
regla. No lo bloqueo con un constraint (el plan es un precio, la agenda
puede crecer mañana), pero la UI de alta de plan debería avisar.

**(g) Cliente con series en dos servicios distintos.** La cuota es por
servicio, no interactúan. Correcto por construcción — y es exactamente la
pregunta que el pase full reabriría (§2.3).

**(h) Mudar el horario fijo a mitad de semana.** Cancelar la serie del
lunes y crear una del miércoles respeta la cuota (una serie `CANCELLED` no
está en vigencia). Pero si el lunes ya estaba `CONFIRMED`, esa semana el
cliente hace **dos** clases con un plan de 1x. Es una fuga acotada a la
semana del cambio, es la misma laxitud que el modelo actual ya tiene, y el
dueño puede cancelar la fecha si le importa. Lo digo en vez de esconderlo.

**(i) Servicio gratis con plan de cuota.** El paso 1 de
`payment_covers_slot()` devuelve `true` antes de mirar el plan, así que la
cuota **no se aplica** sobre un servicio con `payment_required = false`, y
un plan `WEEKLY_QUOTA` ahí aparentaría estar vigente sin estarlo. Ver §5.5:
recomiendo prohibirlo por trigger, en los dos sentidos (crear un plan de
cuota sobre un servicio gratis, y apagar `payment_required` en un servicio
que tiene planes de cuota).

**(j) Inconsistencia preexistente que este trabajo vuelve visible.**
`admin_create_recurring_booking()` chequea
`ALREADY_HAS_STANDING_RESERVATION` con `status = 'ACTIVE'` y sin mirar
`end_date`, así que una serie terminada en junio todavía bloquea crear una
nueva sobre la misma regla. No es regresión de este ADR, pero conviene
alinearlo con la definición de "en vigencia" de §2.4a en la misma fase.

---

## 7. Genericidad

Ninguna entidad nueva nombra un rubro: `service_plans` es la lista de
precios de un `Service`, y `weekly_quota` es una cantidad de horarios
fijos por semana.

**Cancha de fútbol 5** — `Service` "Cancha 1", `capacity = 1`, una
`ScheduleRule` por hora. `WEEKLY_QUOTA` 1 = "tu turno fijo de los martes
20:00, X por mes". `DROP_IN` = alquiler suelto. `UNLIMITED` simplemente no
se crea. ✔

**Consultorio** — `WEEKLY_QUOTA` 1 = el turno fijo semanal (el caso
canónico de psicoterapia). `DROP_IN` = consulta suelta. `UNLIMITED` = abono
mensual con sesiones libres. ✔

**Consultorio, "4 consultas al mes"** — **no entra**, y es la señal
importante: es un contador sobre un período, no un conjunto de horarios, y
necesita el mecanismo de paquete prepago que ADR-0022 eliminó a propósito
(§2.2, §5.2). El modelo cierra para **cadencia semanal + clase suelta +
ilimitado**, no para cantidades por mes. Lo digo derecho porque el plan de
la iteración lo usa como prueba de genericidad y no lo es.

**Lo que sí es genérico de fondo:** la capacidad ("3 personas en paralelo")
vive en `slot_occurrences.capacity` y la frecuencia ("2 veces por semana")
vive en el plan. Son dos perillas independientes, y esa separación es la
que permite que "limitado a 3 personas" y "1x/2x/3x por semana" no se
pisen — y la razón por la que modelar cada frecuencia como un `Service`
aparte estaría mal.

---

## 8. Alternativas descartadas

| Alternativa | Por qué no |
|---|---|
| **Cuota como contador de reservas por semana** (`bookings` contadas en una ventana) | Obliga a definir semana calendario vs. corrida, a decidir qué pasa en una semana con feriado, y a proteger el conteo bajo concurrencia (dos reservas simultáneas gastando el mismo cupo de cuota). Además contradice lo que el cliente describió: él vende **un horario**, no un crédito. |
| **Cuota persistida en una columna de `customers` o de `recurring_bookings`** | Segunda fuente de verdad de algo derivable, y el mismo error que ADR-0010 prohibió al no copiar día/hora en `RecurringBooking`. Además la cuota cambia con el pago del mes, así que una columna por cliente sería inmediatamente falsa. |
| **Una frecuencia = un `Service`** ("Pilates 1x", "Pilates 2x") | Parte la capacidad de 3 en cupos separados por servicio, que es justo lo que el cliente no quiere. Y multiplica `ScheduleRule`, `SlotOccurrence` y agenda pública por cada precio. |
| **`plan_kind` y `billing_type` colapsados en una sola columna** | Hoy están correlacionados, pero contestan preguntas distintas (derecho de reserva vs. período del pago). Colapsarlas vuelve imposible el pase de día (`UNLIMITED` + `ONE_TIME`) y el plan trimestral sin agregar un valor a `plan_kind`, que es el enum que el camino de decisión ramifica. |
| **Plan multi-servicio (`service_plan_services` N:M) desde ahora** | Reintroduce la indirección que ADR-0022 eliminó, vuelve inexpresable el `EXCLUDE` de doble cobro, y obliga a decidir "pozo compartido o por servicio" sin que nadie lo haya pedido (§2.3). |
| **Clase suelta como período de un día `[D, D]`** | Choca contra el `EXCLUDE` del plan mensual, y no distingue dos clases del mismo día — que es exactamente lo que el flujo "sin cupo pagás" tiene que saber. |
| **`EXCLUDE` reanclado a `(customer, service_plan, rango)`** | Deja pasar dos planes mensuales distintos del mismo servicio para el mismo mes: un doble cobro real que hoy se detecta. |
| **Cancelar automáticamente las series excedentes al bajar de plan** | Destruye intención que el cliente puede querer recuperar el mes siguiente, y hace que una escritura de plata cancele reservas — que ADR-0019 ya rechazó explícitamente ("anular un pago no cancela clases ya confirmadas"). |
| **Reusar `PAYMENT_REQUIRED` para "fuera de tu frecuencia"** | Le dice "tenés que pagar" a alguien que acaba de pagar. Es el mismo error que ADR-0018 corrigió con `not_generated_reason` y que Phase 12 volvió a corregir. |
| **Dropear `payments.service_id` porque es derivable del plan** | Rompe `EXCLUDE`, índices y cuatro funciones de lectura del módulo de Pagos a cambio de una columna. Derivada + verificada por trigger da la misma garantía sin el costo. |

---

## 9. Riesgos

1. **Preview de reserva fija mostrando "fuera de tu frecuencia" en todo.**
   El riesgo más concreto: `preview_recurring_booking()` evalúa hoy como
   reserva puntual (§2.5). Test obligatorio: preview de una serie
   prospectiva para un cliente con plan 2x y 1 serie en vigencia ⇒ todas
   las fechas `OK`.
2. **Un `join ... and sp.is_active` en el camino de decisión** le corta la
   cobertura a todos los clientes cuando el dueño ordena su lista de
   precios (§6.d). Test: desactivar el plan con pagos vigentes ⇒ las
   reservas siguen confirmándose.
3. **El desempate de cuota cambiando de resultado entre dos llamadas.**
   Si el `order by` no es total (`created_at` puede empatar), qué serie
   queda dentro de la cuota puede alternar entre el preview y el confirm.
   Por eso el orden es `created_at asc, id asc`, no sólo `created_at`.
4. **Backfill incompleto** = un pago sin plan. Mitigado con `NOT NULL`
   después del backfill y con el puente que falla ruidoso; el riesgo real
   es un `Service` con pagos y `price is null`, que el backfill tiene que
   tolerar (precio 0 en el plan `UNLIMITED` generado, y que el dueño lo
   corrija).
5. **La pantalla de pagos queda inutilizable** si el dueño desactiva el
   plan `UNLIMITED` antes de que exista el selector de plan (§4,
   secuenciación).
6. **Clase suelta pagada y después `SLOT_FULL`.** El orden operativo es
   cobrar la clase y después reservar (si no, la reserva se rechaza por
   `OUTSIDE_PLAN_QUOTA` antes de que exista el pago), así que puede quedar
   un pago suelto sin reserva. Como está anclado a la ocurrencia, ese pago
   es **legible** ("pagó esa clase") y sigue habilitando reservarla si se
   libera un lugar; y cancelar la reserva **no** anula el pago (eso lo
   decide una persona, mismo criterio que ADR-0019). Con tarjeta esto pasa
   a ser el problema de `payment_intents` de ADR-0027: **hay que no
   pintarse en un rincón ahora**, dejando el cobro y la reserva en una
   sola acción de servidor.
7. **`ALTER TYPE ADD VALUE` usado en un CHECK o DML de la misma
   migración** falla (§2.5).
8. **`weekly_quota` vendida por encima de los horarios disponibles**
   (§6.f): no es un bug de datos, es una promesa que el negocio no puede
   cumplir, y se detecta en el mostrador si la UI avisa.

---

## 10. Dónde vive cada validación (una sola fuente de verdad)

- **La regla de cuota y de cobertura: sólo en PostgreSQL**
  (`payment_covers_slot()`, `evaluate_customer_booking()`). Ni el frontend
  ni las server actions la reimplementan; renderizan el
  `can_book_reason` que reciben. Es la razón por la que este ADR no
  propone ninguna función TypeScript que decida si alguien puede reservar.
- **Invariantes de una sola fila: `CHECK`** (`weekly_quota` ↔ `plan_kind`,
  combinación kind↔cobro, `price >= 0`, coherencia de cancelación).
  `service_plans` es escribible por PostgREST, así que Zod en el
  formulario no es una defensa — mismo argumento que ADR-0020 para el
  color de marca.
- **Invariantes que cruzan tablas: trigger** (organización, coherencia
  plan↔pago↔ocurrencia, inmutabilidad de `plan_kind`/`weekly_quota`,
  servicio gratis sin planes de cuota). No pueden ser CHECK y no hay que
  fingir que sí.
- **Doble cobro y ambigüedad: índices únicos parciales y `EXCLUDE`** (§2.7,
  §2.1).
- **Tipos y Zod en `@reservaste/domain`** (`backend/src/types.ts`,
  `schemas.ts`, `mappers.ts`): `ServicePlanKind`, `ServicePlan`, el mapper
  snake_case→camelCase y el esquema de alta/edición. Es forma de input y
  tipado compartido, **no** una segunda implementación de la regla.

---

## 11. Actualización de `docs/domain.md` que este ADR implica

(No aplicada — la aplica el Orchestrator al aprobar.)

- Agregar **`ServicePlan`** a las entidades centrales: *"lo que un
  `Customer` compra respecto de un `Service`: un precio, un período de
  cobro y un derecho de reserva (`DROP_IN` | `WEEKLY_QUOTA` |
  `UNLIMITED`). Genérico por diseño: una clase suelta, un turno fijo
  semanal o un abono libre."*
- Reescribir la sección de `Payment`: ancla de registro `service_plan_id`,
  `service_id` derivado y verificado, `slot_occurrence_id` para los pagos
  sueltos.
- Marcar `ServiceEntitlement` y la sección "Vigencia de
  ServiceEntitlement / `requiresActivePayment`" como histórica (ya lo era
  de hecho desde ADR-0022; sigue descripta como vigente en `domain.md`).
- Invariantes nuevos: la cuota se mide en series en vigencia para la fecha
  del slot y nunca en una ventana semanal; desactivar un plan no invalida
  un pago; `plan_kind`/`weekly_quota` son inmutables con pagos asociados;
  una reserva extra no la cubre un pago por período de cuota.
- El párrafo de "Pago ≠ permiso" **se refuerza, no se debilita**: un pago
  ahora compra un derecho **de forma** específica, así que "pagó" y "puede
  reservar esto" se separan todavía más que antes.
