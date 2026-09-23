# ADR-0029 — Alcance configurable de `ServicePlan`: uno, varios o todos los servicios

Fecha: 2026-09-22
Estado: **Propuesta** (no aplicada — requiere aprobación del Orchestrator)
Propuesta por: `domain-architect`
Origen: pregunta del usuario en producción, con ADR-0024 ya desplegada
(backend + UI de la Fase H completos): *"que pasa si quiero crear planes
que habiliten a todos los servicios? osea tengo 2 servicios y 1 plan para
ambos, no puedo?"*, y la aclaración siguiente: *"[la cuota] debería poder
configurarlo, osea, para mí los planes pueden o no ser por servicio, ahora
están 100% sujetos."*
Depende de: ADR-0022 (`Service` como unidad de cobro), ADR-0024
(`ServicePlan`, `payment_covers_slot`, `evaluate_customer_booking`),
ADR-0010 (recurrencia, "en vigencia" vs `ACTIVE`)
Afecta directamente: ADR-0025 (crédito de recupero — **aceptada, aún sin
migración**: no hay `20260922...phase18...makeup_credits.sql` en el repo,
así que este ADR llega a tiempo de fijar la forma antes de que se
construya, no después)
Anotado en: ADR-0026, resolución 7 (`service_plans_select_public` se acota
en esta misma ADR, no antes)

---

## 0. Corrección al encuadre del pedido, antes de la decisión

El pedido de la tarea ofrece dos alternativas para expresar "este plan
cubre estos servicios" y pide elegir una, señalando que cada una tiene una
trampa. **La respuesta correcta no es elegir una — es que resuelven
problemas distintos y conviene tener las dos, y con la regla de
inmutabilidad de abajo (§3.5) dejan de pisarse.**

- Una fila explícita por servicio (`service_plan_services`) es lo correcto
  para *"este plan cubre exactamente Pilates y Musculación"* — una
  selección deliberada y cerrada. Nunca crece sola, que es exactamente lo
  que hace falta cuando el combo es una decisión de precio concreta, no
  "todo lo que haya".
- Un flag `applies_to_all_services` es lo correcto para *"este plan cubre
  todo lo que el negocio ofrezca, incluido lo que cree mañana"* — un pase
  VIP. Resolverlo dinámicamente (join contra `services` activos de la
  organización en el momento de evaluar) es la única forma de cumplir la
  promesa "incluido lo futuro": una fila materializada por servicio actual
  necesitaría un trigger que la mantenga al día, y ese trigger chocaría de
  frente con la regla de inmutabilidad que este mismo documento propone en
  §3.5 (el alcance no puede cambiar una vez que el plan tiene pagos) — un
  plan "todos los servicios" con pagos ya activos **tiene** que poder
  empezar a cubrir un servicio creado la semana que viene, o el flag no
  significa lo que dice. Ninguna de las dos rutas de la trampa original
  resuelve esto sola; la combinación sí, porque cada mecanismo es
  inmutable en lo que le corresponde (el *flag* nunca cambia de valor una
  vez pago; lo que el flag *resuelve* sí puede cambiar, por diseño).

Dicho de otro modo: la pregunta "¿fila explícita o flag dinámico?" tiene
trampa porque asume que es una sola decisión de modelado. Son dos modos de
uso distintos del mismo plan, y el esquema los deja convivir sin
ambigüedad porque nunca se usan a la vez (ver CHECK en §1).

---

## 1. La relación plan↔servicio pasa a N:M

`service_plans.service_id NOT NULL` se reemplaza por dos mecanismos que se
excluyen entre sí por plan:

```sql
alter table public.service_plans
  add column applies_to_all_services boolean not null default false;

create table public.service_plan_services (
  service_plan_id uuid not null references public.service_plans (id) on delete cascade,
  service_id      uuid not null references public.services (id) on delete cascade,
  primary key (service_plan_id, service_id)
);

-- mismo criterio que check_service_plan_same_org(): el trigger valida que
-- service_id pertenece a la misma organization_id que el plan.
```

`service_plans.service_id` **se retira** como columna NOT NULL de
catálogo (no hay más "el servicio del plan", hay "los servicios que el
plan cubre"). Ver §4 para qué pasa con el nombre `service_id` que hoy
usan `payments` y varias funciones de lectura — no es lo mismo que esta
columna de catálogo y se trata aparte.

Coherencia, en un solo `CHECK` (ninguna de las tres queda implícita):

```sql
constraint service_plans_scope_exclusive check (
  not (applies_to_all_services and exists (
    select 1 from service_plan_services where service_plan_id = id
  ))
  -- (expresado como trigger, no CHECK puro, porque cruza tablas)
)
```

y un plan siempre cubre **al menos un servicio**: si
`applies_to_all_services = false`, tiene que existir al menos una fila en
`service_plan_services` (trigger, no constraint declarativo — se valida
al confirmar la transacción, no en cada INSERT parcial).

**Helper de lectura única, para que nada vuelva a bifurcar la lógica**
(mismo espíritu que `is_organization_member`: una sola definición de "qué
cubre este plan hoy"):

```sql
create or replace function public.service_plan_covered_service_ids(p_service_plan_id uuid)
returns setof uuid
language sql stable security definer set search_path = public
as $$
  select sp.id -- reemplazado abajo por la unión real
$$;
```

En la práctica: `select service_id from service_plan_services where
service_plan_id = $1`, **unión** con `select id from services where
organization_id = (select organization_id from service_plans where id =
$1) and is_active` cuando `applies_to_all_services`. Todo lector — la
cadena de cobertura, el selector de precios público, la pantalla de
administración de planes — pasa por esta función en vez de repetir el
`UNION`. Es SQL, no PL/pgSQL: firma de referencia, `database-agent`
resuelve la implementación exacta.

### 1.1 `DROP_IN` se queda fuera de esta ADR, a propósito

Restrinjo el alcance multi-servicio a `WEEKLY_QUOTA` y `UNLIMITED`.
`DROP_IN` sigue exigiendo `applies_to_all_services = false` y exactamente
un servicio en `service_plan_services` (mismo `CHECK` de hoy,
`service_plans_one_active_drop_in_idx` se mantiene igual, reubicado sobre
la fila del join en vez de sobre la columna vieja).

**Por qué:** un turno suelto se paga anclado a una `SlotOccurrence`
concreta (`payments.slot_occurrence_id`), que ya determina sin ambigüedad
de qué servicio se trata. Un `DROP_IN` "para cualquiera de estos dos
servicios, mismo precio" no es "una clase suelta más flexible" — es un
contador de usos sin horario fijo ("comprás 1 uso, lo gastás en el
servicio que quieras"), y eso es exactamente el paquete prepago
(`CREDITS`/`MONTHLY_QUOTA`) que ADR-0022 y ADR-0024 sacaron de alcance por
decisión explícita del usuario. Reabrirlo por esta puerta lateral sería
meter el mismo segundo camino de cuota que esas dos ADR ya rechazaron. Si
en el futuro hace falta un "bono de N usos sobre cualquiera de estos
servicios", es su propio diseño — no una variante de `DROP_IN`.

---

## 2. Migración de los datos existentes — aditiva, sin tocar comportamiento

Un solo paso, sin puente de compatibilidad de dos fases (a diferencia de
ADR-0022/0024, acá no hace falta: la columna vieja no la lee ningún
llamador externo por PostgREST, la escribe únicamente el formulario de
planes vía server action, que `backend-api-agent` actualiza en el mismo
PR):

1. Crear `service_plan_services` y `applies_to_all_services` (default
   `false`, aditivo puro).
2. Backfill: `insert into service_plan_services select id, service_id from
   service_plans` — una fila por plan existente, exactamente el servicio
   que ya tenía. Ningún plan cambia de alcance.
3. Retirar `service_plans.service_id` de catálogo (columna eliminada, no
   deprecada — a diferencia de `services.billing_type/price`, acá no hay
   dato histórico que preservar: la fila de `service_plan_services`
   backfillada **es** el dato histórico, sin pérdida de información).
4. Todo lector que hacía `service_plans sp ... where sp.service_id = X`
   pasa a `join service_plan_services sps on sps.service_plan_id = sp.id
   and sps.service_id = X` (para "¿este plan cubre X?") o a
   `service_plan_covered_service_ids(sp.id)` (para "¿qué cubre este
   plan?", ej. UI). Es mecánico porque el backfill garantiza que todo plan
   pre-existente tiene exactamente una fila — el resultado de la query no
   cambia para ningún plan de hoy, sólo la forma de escribirla.

Nadie cambia de comportamiento el día del deploy: mismo criterio que
ADR-0022/0024/0025, pero acá el paso 3 puede ir en la misma migración
porque no hay ningún camino de escritura externo que dependa de la
columna vieja (a diferencia de `payments.service_plan_id`, que si tenía un
formulario insertando por PostgREST directo).

---

## 3. Cuota compartida vs. independiente — configurable, con semántica exacta

```sql
create type public.plan_quota_scope as enum ('PER_SERVICE', 'SHARED_ACROSS_SERVICES');

alter table public.service_plans
  add column quota_scope public.plan_quota_scope;

constraint service_plans_quota_scope_matches_kind check (
  (plan_kind = 'WEEKLY_QUOTA' and quota_scope is not null)
  or (plan_kind <> 'WEEKLY_QUOTA' and quota_scope is null)
)
```

`UNLIMITED` no tiene cuota que repartir (cubre todo lo que declare, sin
límite, en cualquiera de los servicios que abarque) — `quota_scope` no
aplica y el `CHECK` lo prohíbe, igual que `weekly_quota` hoy.

### 3.1 Semántica exacta

**`PER_SERVICE`** (default conceptual — es el comportamiento de hoy,
generalizado): `weekly_quota` se aplica **independientemente a cada
servicio cubierto**. Un plan `WEEKLY_QUOTA=2` que cubre Pilates y
Musculación con `PER_SERVICE` permite hasta 2 series en vigencia en
Pilates **y** hasta 2 series en vigencia en Musculación — el máximo total
posible es 2×(cantidad de servicios cubiertos), no 2. Es literalmente "cada
servicio se cuenta como si tuviera su propio plan", que es exactamente el
comportamiento que ya existe hoy para un plan de un solo servicio: esta
ADR no le cambia nada a ningún plan actual, sólo permite declarar el mismo
número para más de un servicio a la vez en el mismo plan/pago, en vez de
obligar a comprar un plan por servicio.

**`SHARED_ACROSS_SERVICES`**: `weekly_quota` es un **pool único**
repartido entre **todos** los servicios que el plan cubre, sumando series
de cualquier combinación de ellos. Un plan `WEEKLY_QUOTA=2` compartido
entre Pilates y Musculación permite 2 series en Pilates, o 2 en
Musculación, o 1+1 — nunca más de 2 en total, sin importar cómo se
repartan.

### 3.2 La pregunta incómoda, contestada sin dejarla abierta

> *"¿un cliente puede tener 1 serie fija en cada uno (total 2), o eso no
> tiene sentido porque son horarios de servicios distintos con schedule
> rules distintas?"*

**Sí, tiene sentido y es exactamente lo que `SHARED_ACROSS_SERVICES`
significa.** El error de la premisa de la pregunta es asumir que "serie"
está atada al servicio de alguna forma que impediría sumarlas — no lo
está. El hallazgo central de ADR-0024 fue justamente que "veces por
semana" **es** "cantidad de `RecurringBooking`", sin ningún otro
significado: cada serie apunta a una `ScheduleRule`, cada regla pertenece
a un servicio, pero **contar series nunca necesitó que todas apuntaran al
mismo servicio** — la función de hoy (`customer_series_in_force_count`)
sólo filtra por un único `service_id` porque hasta ahora un plan sólo
podía tener uno. Generalizar el filtro de "= un servicio" a "IN (el
conjunto que el plan cubre)" no rompe ninguna invariante de ADR-0024, la
extiende literalmente: sigue sin haber contador persistido, sigue siendo
"cuántas series en vigencia para la fecha", sólo que la fecha de una serie
de Musculación cuenta contra el mismo pool que la fecha de una serie de
Pilates cuando el plan las declaró compartidas.

### 3.3 Reescritura de `customer_series_in_force_count()`

Firma nueva (reemplaza `p_service_id uuid` por un conjunto):

```sql
create or replace function public.customer_series_in_force_count(
  p_customer_id uuid,
  p_service_ids uuid[],   -- antes: p_service_id uuid
  p_local_date date
) returns int language sql stable security definer set search_path = public as $$
  select count(*)::int
  from public.recurring_bookings rb
  join public.schedule_rules sr on sr.id = rb.schedule_rule_id
  where rb.customer_id = p_customer_id
    and sr.service_id = any(p_service_ids)
    and rb.status = 'ACTIVE'
    and rb.start_date <= p_local_date
    and (rb.end_date is null or rb.end_date >= p_local_date);
$$;
```

Un único parámetro que sirve para los dos casos: `PER_SERVICE` invoca la
función una vez por servicio cubierto con un arreglo de un elemento
(`array[service_id]`, idéntico resultado a la firma vieja); `SHARED` la
invoca **una sola vez** con el arreglo completo de servicios cubiertos.
Es una función, dos formas de llamarla — mismo criterio que
`evaluate_payment_coverage` ya usa para "sin serie / serie existente /
serie prospectiva" (ADR-0024 §nota 7): no se bifurca la lógica, se
parametriza. `customer_series_quota_position()` se generaliza igual (el
desempate `created_at asc, id asc` pasa a ser sobre el conjunto completo
de series del pool, no por servicio, cuando `SHARED_ACROSS_SERVICES`).

**Nota importante sobre el desempate compartido:** si un cliente con pool
compartido 2x tiene 3 series (2 en Pilates + 1 en Musculación, en ese
orden de contratación), el desempate `created_at, id` sobre el pool
completo puede dejar afuera la serie de Musculación aunque Pilates
"tenga sitio de sobra" en un sentido ingenuo — es correcto: el pool es
compartido, no hay sitio de sobra por servicio, sólo por fecha de
contratación. La pantalla de mostrador (`schedule_rule_standing_reservations`)
tiene que decirlo así ("2/2 del pase compartido, esta serie quedó afuera
por orden de contratación"), no como si fuera un problema de ese servicio
en particular.

### 3.4 Un solo `weekly_quota` por plan, nunca uno por servicio dentro del plan

Decisión explícita, para cerrar una variante que la tarea no pidió pero
que el modelo podría insinuar: **no** se admite `weekly_quota` distinto
por servicio dentro de un mismo plan (ej. "2x Pilates + 1x Musculación en
un solo plan"). Si un negocio quiere eso, son dos planes — cada uno con su
propio `weekly_quota` y su propio pago, exactamente como se resuelve hoy.
Meter una cuota heterogénea por servicio dentro de un plan reabriría en
otra forma el mismo problema que `MONTHLY_QUOTA` (ADR-0024 §3): un
segundo esquema de cuota que nadie pidió, por un caso que ya tiene
solución con lo que existe.

---

## 4. `Payment`, `payment_covers_slot()` y el `EXCLUDE` anti-doble-cobro

### 4.1 El pago se ancla a `(customer, plan)`; el servicio se vuelve derivado, y para más de un servicio no cabe en una columna

`payments.service_plan_id` (ADR-0024) ya es el ancla real — eso no
cambia. Lo que cambia es que `payments.service_id`, hoy una columna
escalar "derivada y verificada por trigger" de un plan que sólo podía
tener un servicio, **deja de poder representar el derivado completo**
cuando el plan cubre varios. Se resuelve así:

- **`payments.service_id` se mantiene, pasa a nullable.** Se sigue
  poblando por trigger exactamente igual que hoy **cuando el plan cubre
  un solo servicio** (el caso mayoritario, sin cambios). Cuando el plan
  cubre más de uno, queda `NULL` — no hay "el" servicio del pago.
- **Tabla nueva `payment_service_coverage`**, la fuente de verdad completa
  de "qué servicios cubre este pago, en qué rango de fechas", con una fila
  **por cada servicio** que el plan cubría en el momento del pago
  (una fila incluso para el caso de un solo servicio — ver por qué en
  §4.2):

```sql
create table public.payment_service_coverage (
  id uuid primary key default gen_random_uuid(),
  payment_id uuid not null references public.payments (id) on delete cascade,
  customer_id uuid not null references public.customers (id) on delete cascade,
  service_id uuid not null references public.services (id) on delete cascade,
  status public.payment_status not null,     -- copia de payments.status, mantenida por trigger
  period_start date not null,
  period_end date not null
);

create index payment_service_coverage_payment_idx on payment_service_coverage (payment_id);

create extension if not exists btree_gist; -- ya está habilitada (EXCLUDE de payments la usa hoy)

alter table public.payment_service_coverage
  add constraint payment_service_coverage_no_overlap
  exclude using gist (
    customer_id with =,
    service_id with =,
    daterange(period_start, period_end, '[]') with &&
  ) where (status = 'PAID');
```

Poblada por trigger `AFTER INSERT` en `payments` (para pagos anclados a
período, es decir `slot_occurrence_id is null` — los `DROP_IN` de turno
suelto no la usan, siguen con el índice único
`payments_one_paid_per_occurrence_idx` de ADR-0024 sin cambios, consistente
con §1.1): una fila por cada id que devuelva
`service_plan_covered_service_ids(new.service_plan_id)`, con
`period_start/period_end` copiados del pago. Un trigger `AFTER UPDATE OF
status` propaga `VOID`/reactivación a las filas hijas, para que el
`EXCLUDE` libere el rango cuando el pago se anula.

### 4.2 Por qué reemplaza —no convive con— el `EXCLUDE` viejo sobre `payments`

Consideré mantener el `EXCLUDE` actual de `payments` (por
`customer_id, service_id, rango`) para los pagos de plan único, y sumar
esta tabla nueva sólo para los multi-servicio. **Lo descarto**: dejaría
dos mecanismos de exclusión distintos operando sobre el mismo hecho de
negocio ("¿este cliente ya tiene cobertura pagada de este servicio en
esta fecha?"), sin forma de que Postgres los cruce. El caso que se
rompería: un cliente con un plan de un solo servicio (Pilates) `PAID` para
septiembre, más un plan multi-servicio (Pilates+Musculación) también
`PAID` para septiembre — dos pagos vigentes que ambos cubren Pilates,
exactamente lo que la resolución 1 de ADR-0024 ("un solo pago vigente por
`(customer, service)`, VOID + recargar para cambiar de plan") existe para
impedir. El `EXCLUDE` de `payments` sólo ve su propia tabla y el de
`payment_service_coverage` sólo la suya — ninguno de los dos, por
separado, detecta el choque cruzado.

Por eso **todo pago anclado a período** (sin importar si su plan cubre uno
o varios servicios) genera sus filas en `payment_service_coverage`, y el
`EXCLUDE` viejo de `payments` (`payments_no_overlapping_paid`) **se
retira**: la protección anti-doble-cobro por período pasa a vivir
enteramente en `payment_service_coverage`, un solo lugar, sin excepción
por tipo de plan.

### 4.3 `evaluate_payment_coverage()` — el paso 4 cambia de fuente, la forma no

Hoy busca el pago vigente con `p.service_id = p_service_id`. Pasa a buscar
vía la tabla puente:

```sql
select p.* into v_payment
from public.payments p
join public.payment_service_coverage psc on psc.payment_id = p.id
where psc.customer_id = p_customer_id
  and psc.service_id = p_service_id
  and p.status = 'PAID'
  and psc.period_start <= v_local_date
  and psc.period_end >= v_local_date
order by psc.period_start desc, p.created_at desc
limit 1;
```

Todo lo demás del paso 4 en adelante (leer `v_plan` sin filtrar
`is_active`, `UNLIMITED` → `OK`, `DROP_IN` → defensivo, `WEEKLY_QUOTA` →
posición contra cuota) sigue igual, salvo que el cálculo de posición
(§3.3) recibe el conjunto de servicios cubiertos por `v_plan`
(`PER_SERVICE` → `array[p_service_id]`; `SHARED_ACROSS_SERVICES` →
`service_plan_covered_service_ids(v_plan.id)` completo) en vez de siempre
`array[p_service_id]`. **Ninguna función nueva de decisión** — sigue
siendo `evaluate_payment_coverage()` y `evaluate_customer_booking()`, con
un paso interno que ahora sabe leer un conjunto.

### 4.4 ¿Puede convivir un `UNLIMITED` de un solo servicio con un `UNLIMITED` multi-servicio que también lo cubre?

**Sí, es un caso legítimo del catálogo, no un conflicto — y dejo de exigir
"un solo `UNLIMITED` activo por servicio".**

El índice `service_plans_one_active_unlimited_idx` de ADR-0024 no
protegía una regla de negocio: protegía el **puente de compatibilidad de
la migración** (`fill_payment_service_plan()`, que hace `... where
plan_kind = 'UNLIMITED' limit 1` cuando un INSERT en `payments` no trae
`service_plan_id` explícito). Verificado contra el estado real: la Fase H
ya cerró ese puente en la práctica — la nota de cierre de la fase dice
"selector de plan en el registro de pago con las dos call sites", es
decir, el formulario de pagos ya manda `service_plan_id` siempre. El
`limit 1` no determinista de la migración queda como código muerto de
transición, no como una regla que algo actual dependa de que sea
determinística.

Sin esa dependencia, "Pilates ilimitado $2000" y "Pilates+Musculación
ilimitado $3000" activos a la vez son dos filas normales de una lista de
precios — el cliente elige una, su pago dice cuál explícitamente, y la
cadena de cobertura (§4.3) resuelve por `service_plan_id`, nunca
adivinando "el" `UNLIMITED` de un servicio. Se retira el índice; no lo
reemplazo por uno nuevo porque no hay invariante de negocio que expresar
acá (a diferencia de `DROP_IN`, ver §1.1, donde el índice sigue vigente
porque ahí sí hay un flujo de cobro automático — `quote_booking()`/
`book_slot_paying()` de ADR-0025 — que necesita "el" precio de turno
suelto sin preguntarle nada al cliente).

**Riesgo que dejo señalado, no resuelto acá:** con el puente muerto pero
la función `fill_payment_service_plan()` todavía instalada, si algún
llamador nuevo llegara a insertar sin `service_plan_id` explícito
(bypass del formulario, ej. un script), el `limit 1` sin orden
determinístico entre dos `UNLIMITED` activos ahora sí importaría, porque
ya no hay un único candidato. Recomiendo a `database-agent` decidir si
esta función se retira del todo en la migración de esta ADR (ya no tiene
llamador vivo, según la nota de cierre de Fase H) o si se endurece con un
`raise exception` cuando hay más de un `UNLIMITED` activo — no lo decido
yo porque es una llamada sobre código de transición ajeno a esta ADR, no
sobre el modelo de dominio.

---

## 5. Impacto en ADR-0025 (crédito de recupero) — todavía sin migración, a tiempo de fijarlo bien

Verificado: no existe `makeup_credits` en `backend/supabase/migrations/`
(sólo `phase18_drop_service_billing_check.sql`, que es un ajuste no
relacionado). ADR-0025 está **Aceptada** pero **sin construir** — este
ADR llega antes, no después, de que la tabla se cree.

### 5.1 La pregunta, contestada sin simetría falsa

> *"¿el crédito sirve para reservar en cualquiera de ellos [los servicios
> del plan], o solo en el servicio de la reserva que se liberó? [...] no
> es simétrico: liberaste un cupo de Pilates, ¿el crédito te deja
> reservar Musculación?"*

**Depende del `quota_scope` del plan que originó la serie liberada — y
tiene que depender de eso, o el crédito representaría más o menos
derecho del que el cliente realmente tenía:**

- **`PER_SERVICE`** (incluye todo plan de un solo servicio, que es
  `PER_SERVICE` trivial): el crédito queda **anclado a ese servicio**.
  Liberaste Pilates, el crédito sirve para recuperar en Pilates. Es
  literal: bajo `PER_SERVICE` el cupo que tenías en Pilates nunca fue
  fungible con el de Musculación (ver §3.1) — devolvértelo como fungible
  al liberarlo sería **regalarte más de lo que comprabas**.
- **`SHARED_ACROSS_SERVICES`**: el crédito queda **utilizable en
  cualquiera de los servicios que el plan cubre**. Bajo cuota compartida,
  el cupo que liberaste nunca fue "de Pilates" en primer lugar — era una
  unidad del pool que en ese momento estaba asignada a una serie de
  Pilates, pero el pool en sí es indiferente al servicio (§3.1/3.2).
  Restringir el crédito a Pilates sería **darte menos de lo que tenías**:
  el pool del que salió esa unidad podía repartirse con Musculación desde
  el principio.

### 5.2 Shape de `makeup_credits`

```sql
create table public.makeup_credits (
  ...  -- shape de ADR-0025, sin cambios en lo demás
  service_id uuid references public.services (id),        -- nullable ahora
  service_plan_id uuid not null references public.service_plans (id),  -- nuevo
  ...
);

constraint makeup_credits_scope_matches_plan check (
  -- resuelto por trigger contra service_plans.quota_scope del plan
  -- referenciado, no por CHECK puro: cruza tablas.
)
```

- `service_id` **no nulo** cuando el plan de origen es `PER_SERVICE` (el
  servicio exacto de la reserva liberada).
- `service_id` **nulo** cuando el plan de origen es `SHARED_ACROSS_SERVICES`
  — "nulo" acá significa "cualquiera de los que el plan cubre", resuelto
  en el momento de **consumo** vía `service_plan_covered_service_ids()`
  contra `service_plan_id` (lectura en vivo, no una foto congelada — ver
  §6 sobre por qué el alcance es inmutable una vez que el plan tiene
  pagos, así que leer en vivo y leer congelado dan el mismo resultado
  para cualquier plan que ya haya emitido créditos).
- Al **consumir** (dentro de `book_slot()`/`generate_recurring_booking()`,
  aún sin construir): un crédito con `service_id` no nulo sólo cubre una
  reserva de ese servicio exacto; un crédito con `service_id is null`
  cubre una reserva de cualquier servicio en
  `service_plan_covered_service_ids(service_plan_id)`.

**No es una tabla nueva de decisión**: sigue siendo una comparación extra
antes de devolver el veredicto compuesto `(reason, makeup_credit_id)` que
ADR-0025 ya define — el "buscar un crédito `AVAILABLE` que aplique" gana
una condición (`service_id = X or (service_id is null and X = any(...))`),
no una función nueva.

---

## 6. Inmutabilidad del alcance — necesaria, pero no uniforme

`plan_kind` y `weekly_quota` son inmutables con pagos (ADR-0024,
resolución 5) porque `evaluate_payment_coverage()` los lee **en vivo**
desde el plan, sin foto congelada por pago — si pudieran cambiar, un pago
de septiembre se releería con las reglas de octubre. El alcance de
servicios necesita la misma protección, **pero no puede aplicarse por
igual a los dos mecanismos de §1**, porque uno de ellos existe
específicamente para poder cambiar:

- **`applies_to_all_services` (el valor del flag) es inmutable con
  pagos.** Pasar de `true` a `false` (o viceversa) después de tener pagos
  cambiaría retroactivamente qué significó "todos los servicios" para un
  pago ya hecho — mismo argumento que `plan_kind`.
- **Lo que el flag *resuelve*, cuando está en `true`, no es inmutable —
  es dinámico por diseño.** Un plan `applies_to_all_services = true` con
  pagos activos **sí** empieza a cubrir un servicio creado después del
  primer pago, automáticamente, porque literalmente eso es lo que el
  cliente compró: "todo lo que ofrezcan", no "todo lo que ofrecían el día
  X". No es una excepción a la regla de inmutabilidad — la regla protege
  el *valor* que el pago fijó (`true`), no el conjunto que ese valor
  resuelve en cada instante, que por construcción es móvil.
- **`service_plan_services` (la selección explícita) es inmutable con
  pagos**, en el sentido literal: no se permite `INSERT`/`DELETE` sobre
  esas filas para un plan que ya tiene pagos no-`VOID` — trigger análogo a
  `check_service_plan_terms_immutable()` (ADR-0024 §6), extendido para
  cubrir esta tabla además de las columnas de `service_plans`. Agregar
  Musculación a un plan de "sólo Pilates" que ya tiene clientes pagando
  cambiaría retroactivamente qué cubrió cada pago histórico — el remedio
  es el de siempre: desactivar el plan y crear uno nuevo con el alcance
  correcto, que no corta cobertura a nadie (mismo criterio que corregir un
  `weekly_quota` mal cargado).
- **`quota_scope` es inmutable con pagos**, mismo argumento exacto que
  `weekly_quota` — de hecho viaja en el mismo trigger.

**Alternativa descartada:** tratar el alcance completo como inmutable sin
distinción (incluido lo que `applies_to_all_services` resuelve). La
descarto porque directamente contradice el pedido del usuario: un pase
"todos los servicios" que deja de cubrir servicios nuevos en cuanto tiene
el primer cliente pagando no es un pase "todos los servicios", es un pase
"los que había el día que alguien pagó por primera vez" — otro producto,
no el que se pidió.

### 6.1 Los casos límite de la tarea, contestados

- **Un servicio se da de baja (`is_active = false`) y era parte de un plan
  multi-servicio:** el plan sigue vendiéndose para los que quedan, sin
  cambio ninguno de este ADR. No hace falta tocar `service_plan_services`
  ni el plan: `evaluate_customer_booking()` ya corta con `SERVICE_INACTIVE`
  para cualquier intento de reservar el servicio dado de baja, exactamente
  igual que si el plan fuera de un solo servicio y ese fuera el que se dio
  de baja. Es gratis, no hace falta diseñar nada nuevo.
- **`applies_to_all_services` + servicio nuevo mañana:** lo cubre
  automáticamente (§6, arriba). No hace falta agregarlo a mano — ese es
  el argumento de negocio para elegir este mecanismo en vez del explícito.
- **Cambiar un plan de "un servicio" a "varios" con pagos activos:** **no
  se permite**, mismo criterio que `plan_kind`/`weekly_quota` (§6). Se
  desactiva y se crea un plan nuevo con el alcance correcto.

---

## 7. RLS de `service_plans` y `service_plan_services` — lo que ADR-0026 dejó pendiente

`service_plans_select_public using (true)` hoy expone también los planes
**desactivados** (`is_active = false`) a cualquier visitante sin sesión —
no es un dato privado (nombre y precio de un plan no lo son), pero es
información falsa una vez retirado de venta, y el usuario ya la señaló
como algo a resolver en esta misma migración (ADR-0026, resolución 7).

**Decisión:** se acota a `is_active`:

```sql
drop policy service_plans_select_public on public.service_plans;

create policy service_plans_select_public
  on public.service_plans for select
  using (is_active or public.is_organization_member(organization_id));
```

`service_plan_services` es sólo una lista de referencias a `services`
(ya públicos por su propia RLS) — no hay nada privado que filtrar por sí
misma, pero para que un visitante nunca vea "este plan cubre X" de un
plan que ni siquiera puede ver, la policy espeja la de `service_plans`:

```sql
create policy service_plan_services_select_public
  on public.service_plan_services for select
  using (exists (
    select 1 from public.service_plans sp
    where sp.id = service_plan_id
      and (sp.is_active or public.is_organization_member(sp.organization_id))
  ));

create policy service_plan_services_write_staff
  on public.service_plan_services for all
  using (exists (
    select 1 from public.service_plans sp
    where sp.id = service_plan_id and public.is_organization_member(sp.organization_id)
  ));
```

Sin política de `DELETE` propia más allá de la de escritura general —
igual que hoy no hay `DELETE` de `service_plans` (un plan se desactiva,
nunca se borra), y a partir de §6 esta tabla tampoco se borra por fila una
vez que el plan tiene pagos (el trigger de inmutabilidad lo impide, no la
policy).

**Qué necesita `auth-security-agent`, no lo decido yo:** confirmar que
`is_organization_member` evaluado dentro de una policy de `select` sobre
`service_plan_services` no reintroduce el patrón de lógica de tres valores
que ADR-0026 tuvo que corregir en `plpgsql` (acá es SQL puro dentro de un
`EXISTS`, que debería comportarse bien con `NULL`, pero quiero que lo
confirme quien hizo ese repaso, no asumirlo).

---

## 8. UI/UX — en prosa, sin construir

**Admin, alta/edición de plan:** el formulario de planes (Fase H, pestaña
"Planes") gana un selector de alcance con tres opciones, visible sólo para
`WEEKLY_QUOTA`/`UNLIMITED` (para `DROP_IN` el selector de servicio único
de hoy no cambia — ver §1.1):

- **"Un servicio"** (default, preselecciona el servicio desde el que se
  abrió el formulario — comportamiento idéntico al de hoy).
- **"Varios servicios"** → revela un checklist de los servicios activos de
  la organización (mismo patrón visual que un multi-select ya usado en
  otras pantallas del admin).
- **"Todos los servicios (actuales y futuros)"** → oculta el checklist por
  completo; un texto fijo aclara "este plan cubrirá automáticamente
  cualquier servicio nuevo que agregues".

Cuando "Varios"/"Todos" está elegido y `plan_kind = WEEKLY_QUOTA`, se
revela el selector de `quota_scope` con las dos opciones en lenguaje llano
("Cada servicio tiene su propio cupo de N" / "Un cupo de N compartido
entre todos") y un ejemplo numérico inline usando el propio `weekly_quota`
que el admin ya cargó, para que la diferencia entre las dos opciones sea
concreta y no una explicación abstracta.

**Alcance deshabilitado en edición si el plan tiene pagos**, con el mismo
patrón ya usado para `plan_kind`/`weekly_quota` (Fase H: "los términos se
deshabilitan siempre en edición, no sólo cuando hay pagos" — extender
literal a esta selección).

**Agenda pública:** un plan multi-servicio **no se repite** como tarjeta
idéntica bajo cada servicio que cubre (eso lo mostraría como N ofertas
distintas cuando es una sola). Se agrega una sección separada en la página
pública de la organización, "Planes combinados" (o similar, nombre a
definir por `ui-ux-agent`), con una tarjeta por plan multi-servicio que
lista los servicios cubiertos ("Pilates + Musculación") — distinta de la
lista de planes que ya aparece bajo cada servicio individual, que sigue
mostrando sólo sus planes de alcance único para no duplicar.

**Portal del cliente:** `my_services()` (ADR-0025) pasa a mostrar, para un
plan de alcance múltiple, la lista de servicios cubiertos junto al plan
("Pase Full — Pilates, Musculación") en vez de asumir un solo nombre de
servicio. `my_makeup_credits()` muestra un crédito con `service_id null`
como "Crédito de recupero (Pilates o Musculación) — vence el 30/09", no
como si fuera de un servicio indefinido.

---

## 9. Genericidad

**Gimnasio (cierra):** pase full = `UNLIMITED`, `applies_to_all_services`
o selección explícita de {Musculación, Clases grupales}. Directo.

**No-gimnasio que cierra:** un centro de estética con "depilación +
manicura" en un solo abono, **si ambos servicios se ofrecen con turno fijo
semanal** (ej. todos los martes a las 15hs, cualquiera de los dos) — un
plan `WEEKLY_QUOTA` compartido 2x entre los dos servicios permite "una
semana depilación, la siguiente manicura, o una de cada" exactamente con
la semántica de §3.2.

**Lo que esta ADR NO resuelve, y hay que decirlo con la misma honestidad
que ADR-0024 usó para descartar `MONTHLY_QUOTA`:** un combo de estética
sin horario fijo ("2 sesiones al mes, cualquiera de las dos, sin turno
recurrente") **sigue fuera de alcance**, exactamente igual que antes de
este ADR. `WEEKLY_QUOTA`/`SHARED_ACROSS_SERVICES` generaliza "cantidad de
series semanales" a más de un servicio — no convierte el modelo en un
contador de usos sueltos. Si el negocio real necesita eso, es el mismo
paquete prepago que ADR-0022/0024 excluyeron a propósito, y esta ADR no
reabre esa puerta por la vía lateral del alcance multi-servicio.

---

## 10. Alternativas descartadas

1. **Elegir un solo mecanismo (fila explícita XOR flag dinámico)**, tal
   como insinuaba el encuadre original de la tarea. Descartada en §0: cada
   uno resuelve una intención de negocio distinta y no compiten entre sí
   una vez fijada la regla de inmutabilidad de §6.
2. **Entidad `Bundle`/`ServicePlanGroup` que agrupa varios `Service` como
   un concepto nuevo de dominio.** Descartada: reintroduce una forma con
   sabor a rubro ("combo"), duplica lo que `Service` ya modela, y obligaría
   a decidir cómo se reconcilian capacidades entre servicios agrupados —
   pregunta que no hace falta contestar porque cada servicio ya mantiene
   su propia `ScheduleRule`/capacidad de forma independiente y correcta.
3. **`weekly_quota` heterogéneo por servicio dentro de un mismo plan**
   (§3.4). Descartada por ser el mismo problema que `MONTHLY_QUOTA` en
   otra forma: un segundo esquema de cuota sin pedido real detrás.
4. **Mantener el `EXCLUDE` viejo de `payments` en paralelo al nuevo de
   `payment_service_coverage`** (§4.2). Descartada: dos mecanismos de
   exclusión sobre el mismo hecho de negocio que Postgres no puede cruzar
   entre sí, con un caso de doble cobro concreto que se cuela por el medio.
5. **`DROP_IN` multi-servicio** (§1.1). Descartada: es el paquete prepago
   que ADR-0022/0024 ya excluyeron, disfrazado de variante de turno suelto.
6. **Materializar `applies_to_all_services` como filas de
   `service_plan_services` mantenidas por trigger en cada alta de
   servicio.** Descartada en §0/§6: choca de frente con la inmutabilidad
   de la selección explícita, y necesitaría lógica especial ("esta tabla
   es inmutable, salvo cuando la llena este trigger en particular") que el
   flag dinámico evita por construcción, sin excepciones que recordar.

---

## 11. Lo que no pude decidir / necesita al Orchestrator

1. **El destino final de `fill_payment_service_plan()` y el índice
   `service_plans_one_active_unlimited_idx`** (§4.4): tengo evidencia de
   que el puente de compatibilidad ya está muerto en la práctica (Fase H
   manda `service_plan_id` explícito), pero retirar del todo una función
   de transición de una fase anterior es una decisión sobre código
   ajeno a este ADR — la dejo señalada para que el Orchestrator la asigne
   junto con la migración de esta fase, no la resuelvo yo.
2. **Nombre de la sección pública "Planes combinados"** (§8): es
   `ui-ux-agent` quien tiene que nombrarla en el vocabulario genérico del
   producto, sin que se cuele lenguaje de rubro — dejo la necesidad
   funcional escrita, no el copy.
3. **Feasibility exacta del `EXCLUDE` sobre `payment_service_coverage`**
   (§4.1): el patrón (`btree_gist` + `daterange` + igualdad de dos
   columnas) es el mismo que ya funciona hoy sobre `payments`, así que el
   riesgo técnico es bajo, pero la validación final de sintaxis, el orden
   de los triggers (`AFTER INSERT`/`AFTER UPDATE OF status` en `payments`
   escribiendo en una tabla hija) y si hace falta un `FOR EACH ROW` con
   `WHEN` para evitar disparos innecesarios en pagos `DROP_IN`, es de
   `database-agent`.
4. **Alcance del reporte de pagos por servicio** (§4.1, "columna
   `payments.service_id` nullable"): las pantallas de administración que
   hoy filtran pagos por `service_id` van a dejar de mostrar los pagos de
   planes multi-servicio hasta que se actualicen para leer también
   `payment_service_coverage`. No es un bug de seguridad ni de
   corrección de cobertura (la cadena de reserva sí ve todo, vía §4.3) —
   es una laguna de reporte que señalo explícitamente y que no resuelvo
   acá porque toca pantallas ya construidas por `frontend-admin-agent`,
   fuera del alcance de un ADR de dominio.
5. **Migración de tests existentes:** los 131 tests de integración y 35
   unitarios de la Fase H asumen `service_plans.service_id` en varios
   call sites de test (no sólo en producto) — no los conté uno por uno,
   así que no puedo dar un número de cuántos rompen contra el nuevo
   esquema. Es tarea de `qa-testing-agent` al implementar, no una
   estimación que yo pueda dar de forma responsable sin correrlos.

---

## 12. Actualización de `docs/domain.md` que implica (si se acepta)

- **`ServicePlan`**: reemplazar "un `Service` tiene N planes" por "un
  `ServicePlan` cubre uno, varios o todos los servicios activos de la
  organización (`service_plan_services` N:M, o `applies_to_all_services`
  resuelto dinámicamente contra los servicios activos) — configurable por
  plan, no una decisión fija del producto (ADR-0029)". Agregar
  `quotaScope: PER_SERVICE | SHARED_ACROSS_SERVICES`, obligatorio sólo
  para `WEEKLY_QUOTA`, con la semántica exacta de §3.1.
- **`Payment`**: `serviceId` pasa de "derivada y verificada por trigger,
  siempre presente" a "derivada, nullable — presente sólo cuando el plan
  cubre exactamente un servicio; la cobertura completa multi-servicio de
  un pago vive en `PaymentServiceCoverage`, no en `Payment`".
- **Entidad nueva, `PaymentServiceCoverage`** (o el nombre que el
  Orchestrator prefiera): fila por `(payment, servicio cubierto)`, con su
  propio rango de fechas y estado — la fuente real del invariante
  anti-doble-cobro por período, reemplazando al `EXCLUDE` que hoy vive
  directamente sobre `Payment`.
- **`ServiceEntitlement` (histórico):** vale la pena una nota de una
  línea señalando que el modelo N:M que esta ADR introduce para
  `ServicePlan` es, en la práctica, el mismo problema que la sección
  histórica de `ServiceEntitlement` ya había anticipado ("puede cubrir
  varios servicios a la vez... modelar `serviceId` nullable + relación
  N:M") antes de que ADR-0022/0024 lo reemplazaran — no es una idea nueva,
  es la misma resuelta en el modelo vigente en vez del deprecado.
- **Invariantes de cuota (ADR-0024)**: sumar la generalización de §3.3 (el
  conteo de series en vigencia toma un conjunto de servicios, no uno solo)
  y la inmutabilidad de alcance/`quota_scope` de §6, con la distinción
  explícita entre "el flag es inmutable" y "lo que el flag resuelve no lo
  es" — es la parte más fácil de simplificar mal si alguien la resume de
  memoria más adelante.
