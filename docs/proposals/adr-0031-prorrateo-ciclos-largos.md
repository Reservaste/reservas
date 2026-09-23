# Propuesta ADR-0031 — Prorrateo del primer período en ciclos de facturación más largos que el mes

Estado: **Propuesta** (no implementada)
Propuesta por: `backend-engineer`
Fecha: 2026-09-23
Toca: estrategia de pagos → **decisión del Orchestrator** (CLAUDE.md)
Depende de: ADR-0022 (unidad de cobro), ADR-0024 (`ServicePlan`), ADR-0029 (alcance del plan)

---

## 1. Problema

`billing_period_for(service_plan_id, from)` conoce hoy exactamente tres formas:

```
MONTHLY + CALENDAR_MONTH → [1 del mes de `from`, último día de ese mes]
MONTHLY + ROLLING_MONTH  → [from, from + 1 mes - 1 día]
ONE_TIME                 → [from, from]
```

No existe ningún ciclo trimestral ni anual. Un negocio que vende "Pilates,
plan anual" hoy sólo puede hacer dos cosas, y las dos son malas:

- cargar doce pagos mensuales a mano, perdiendo el hecho de que se compró
  un año (el descuento, la renovación, el reporte);
- cargar un único pago con `period_start`/`period_end` tipeados a mano
  cubriendo 365 días. Funciona —la cobertura es un rango y el motor ya lo
  evalúa contra la fecha del turno— pero `billing_period_for()` no lo
  propone, así que el mostrador tipea dos fechas cada vez y nada verifica
  que sean coherentes con el plan.

Y el pedido concreto del usuario es el caso que ninguna de las dos
resuelve: **un cliente que entra a mitad de ciclo**. Si el ciclo anual del
negocio va de enero a diciembre y alguien se suma en septiembre, cobrarle
el año completo es incorrecto y cobrarle desde septiembre a septiembre
desalinea su renovación de la de todos los demás para siempre.

## 2. Qué NO es este problema (para no ensanchar el alcance)

- **No es un contador de usos.** ADR-0024 resolución 3 dejó fuera
  `MONTHLY_QUOTA` / paquetes por cantidad, y esto no lo reabre: un ciclo
  trimestral con cuota semanal sigue midiéndose en series en vigencia
  (ADR-0024), exactamente igual que uno mensual. Lo único que cambia es
  **qué período compra un pago** y **cuánto sale ese primer pago**.
- **No toca el motor de reservas.** `evaluate_payment_coverage()` ya
  pregunta "¿la fecha local del turno cae dentro del período pago?"
  (ADR-0013). Un período de tres meses o de un año responde esa pregunta
  sin una línea nueva. Esta propuesta vive entera del lado de *escribir*
  un pago, no de leerlo.

## 3. Cambio propuesto

### 3.1 Dos columnas nuevas en `service_plans`

```sql
alter table public.service_plans
  add column billing_period_months int,           -- null = 1 (el comportamiento de hoy)
  add column billing_anchor_month  int;           -- 1..12, sólo para CALENDAR_* ; null = mes de la compra
```

`billing_cycle` gana dos valores: `CALENDAR_PERIOD` y `ROLLING_PERIOD`,
generalizaciones de los dos actuales. **`CALENDAR_MONTH`/`ROLLING_MONTH`
no se tocan ni se deprecan**: son el caso `billing_period_months = 1` y
son el 100% de los datos existentes. Agregar valores en vez de reescribir
los viejos es el mismo patrón aditivo que no rompió nada en ADR-0022 y
ADR-0024.

- `CALENDAR_PERIOD` + `billing_period_months = 3` + `billing_anchor_month = 1`
  → trimestres ene-mar, abr-jun, jul-sep, oct-dic, iguales para todos los
  clientes del plan.
- `ROLLING_PERIOD` + `billing_period_months = 12` → un año desde el día de
  la compra, distinto para cada cliente. **Un ciclo rodante no prorratea
  nunca**: empieza cuando el cliente entra, así que no hay fracción que
  descontar. Esto es importante porque elimina la mitad de los casos.

`billing_period_months` es **inmutable si el plan tiene pagos no `VOID`**,
por el mismo criterio de ADR-0024 resolución 5 que congela `plan_kind` y
`weekly_quota`: es un término que hay que poder resolver hacia atrás desde
un pago vigente. El precio sigue libre.

### 3.2 `billing_period_for()` se extiende, no se duplica

Se extiende. Una función nueva "sólo para el primer pago de un ciclo
largo" obliga a cada llamador a saber si el pago que está creando es el
primero — y "el primero" no es un dato que la base tenga a mano sin
interpretar el historial. Dos funciones que calculan períodos es
exactamente la segunda fuente de verdad que ADR-0024 evitó al subir
`billing_cycle` del `Service` al plan.

```sql
billing_period_for(plan, from) returns (period_start, period_end)
-- CALENDAR_PERIOD: el bloque de `billing_period_months` meses, anclado en
--   billing_anchor_month, que contiene a `from`. `from` = 2026-09-15,
--   trimestral anclado en enero → [2026-07-01, 2026-09-30].
-- ROLLING_PERIOD: [from, from + N meses - 1 día]. Postgres ya recorta
--   31-ene + 1 mes a 28-feb solo (mismo comentario que hoy).
```

El período **no se recorta**: el cliente que entra el 15 de septiembre en
un plan trimestral jul-sep compra el trimestre jul-sep, y su cobertura
empieza el 1 de julio. Recortar el `period_start` a la fecha de alta
parece más justo y es peor: desalinea el `EXCLUDE`, hace que dos pagos
consecutivos dejen un hueco de un día distinto por cliente, y rompe la
propiedad "todos los clientes del plan renuevan el mismo día" que es el
único motivo para elegir `CALENDAR_PERIOD`.

**Lo que se prorratea es el precio, no el período.** Ésa es la decisión de
diseño central de esta propuesta.

### 3.3 El cálculo: una función de sólo lectura, separada

```sql
quote_service_plan_period(
  p_service_plan_id uuid,
  p_from            date      -- fecha local de alta, en la tz de la Organization
) returns table (
  period_start      date,
  period_end        date,
  full_price        numeric,  -- service_plans.price, tal cual
  prorated_price    numeric,  -- lo que hay que cobrar
  prorated          boolean,  -- si difiere de full_price
  units_charged     int,      -- meses (o días) cobrados
  units_total       int       -- meses (o días) del período completo
)
```

El mostrador **cotiza** y después registra el pago con el monto que la
cotización devolvió. No se cobra automáticamente: `Payment.amount` ya es
un campo libre y cada pago guarda su propio monto (ADR-0024 resolución 5),
así que el prorrateo entra como un **valor sugerido y trazable**, no como
una regla que el motor aplica por su cuenta. Un descuento comercial de
mostrador sigue siendo posible sin pelearse con la función.

Por qué separada de `billing_period_for()`: esa función es `stable` y la
llaman cuatro caminos de lectura; meterle precio la convierte en la
función que todos llaman para todo. `quote_service_plan_period()` la
envuelve, no la reimplementa.

### 3.4 La unidad del prorrateo: meses enteros, no días

**Recomendación: prorratear por meses enteros restantes**, contando el mes
de alta como completo.

- Trimestre jul-sep, alta el 15 de septiembre → 1 mes de 3 → 1/3 del
  precio.
- Trimestre jul-sep, alta el 2 de agosto → 2 meses de 3 → 2/3.

Por qué meses y no días:

1. **Es lo que el mostrador puede explicar.** "Te cobro un mes del
   trimestre" se dice en una frase. "Te cobro 47/92 del trimestre" no.
2. **No depende del largo del mes.** Prorratear por días hace que el mismo
   plan cueste distinto según el trimestre, y que febrero valga menos que
   marzo. Eso genera reclamos y no expresa ninguna intención de negocio.
3. **Coincide con el grano del producto.** Todo lo demás acá es mensual:
   la cuota es semanal, el pago es mensual, el crédito vence a fin de mes.

Un modo `PRORATE_DAILY` puede agregarse después como valor de una columna
`proration_mode` (`NONE | MONTHLY | DAILY`, default `MONTHLY`) sin tocar
nada de lo anterior. **No se construye ahora**: ningún cliente lo pidió.

### 3.5 Redondeo

`round(full_price * units_charged / units_total, 0)` — **a la unidad
entera de la moneda**, hacia arriba en el empate (`round()` de Postgres
sobre `numeric` es half-up, que es lo que queremos).

- `organizations.currency` es ISO 4217 (ADR-0024 resolución 2) pero **no
  guardamos sus decimales**. UYU y ARS se cobran en enteros en el
  mostrador; USD y EUR tienen centavos. Redondear a entero es correcto
  para el cliente actual y conservador para el resto: un peso de más o de
  menos en un plan anual no es un problema, medio centavo sí lo sería si
  el monto alimentara una pasarela.
- **Pregunta abierta para el Orchestrator**: ¿se agrega
  `organizations.currency_minor_units` (0 o 2) ahora, o se difiere hasta
  ADR-0027 (cobro con tarjeta), que es donde los centavos empiezan a
  importar de verdad? Mi recomendación es **diferirlo**: hoy no hay ningún
  consumidor que necesite el centavo, y la columna sin consumidor se
  llena mal.
- El monto sugerido nunca puede ser mayor que `full_price` ni menor que 0.
  Un `CHECK` no alcanza (el monto es libre, a propósito); va como
  `least(greatest(...))` dentro de la función de cotización.

### 3.6 Interacción con "cambio de plan a mitad de período = VOID + recargar" (ADR-0024 resolución 1)

**Recomendación: el prorrateo aplica SÓLO al alta inicial en un ciclo
largo, y NO al cambio de plan.** Y creo que se puede fundamentar, no
queda como pregunta abierta:

ADR-0024 resolución 1 no es una regla de cobro, es una regla de
**resolución**: existe un solo pago vigente por `(customer, service)` para
que el motor de reservas nunca tenga que decidir cuál de dos planes manda.
El VOID + recargar existe para que no queden dos pagos vivos, no para
determinar cuánta plata se cobra. El monto del pago nuevo ya es libre hoy:
el mostrador que sube a alguien de 2x a 3x a mitad de mes cobra la
diferencia si quiere, y el sistema no se entera ni tiene por qué.

Entonces, cuando alguien cambia de plan a mitad de un trimestre:

1. Se anula (`VOID`) el pago vigente — sin cambios respecto de hoy.
2. Se registra el pago del plan nuevo por el **período completo** que
   corresponde (el mismo trimestre), y el monto sugerido es el
   **completo**, no prorrateado.
3. Si el negocio quiere reconocer lo ya pagado, lo descuenta a mano en
   `amount`, que es precisamente para lo que ese campo es libre.

Meter el prorrateo en el cambio de plan obligaría a la función de
cotización a leer el pago anulado para saber qué reconocer, y eso sí es
una regla de precedencia entre dos planes — justo lo que ADR-0024
resolución 1 y su alternativa descartada ("relajar el `EXCLUDE`") vinieron
a impedir, sólo que del lado del dinero en vez del lado del motor.

**Lo que sí queda como pregunta para el Orchestrator**, porque es una
decisión de producto y no técnica: ¿alcanza con que el mostrador ajuste el
monto a mano, o el usuario espera que la pantalla le sugiera el crédito
por lo no consumido del plan anterior? Si espera lo segundo, eso es una
**nota de crédito**, una entidad que hoy no existe, y merece su propio ADR
en vez de esconderse dentro de `quote_service_plan_period()`.

### 3.7 Dónde se ve

- **Pantalla de planes**: el plan gana "cada cuántos meses se cobra" y,
  para ciclos calendario, el mes de anclaje.
- **Registrar pago**: al elegir un plan de ciclo largo, el formulario
  muestra el período completo, el precio completo, y —si el alta cae a
  mitad de ciclo— el monto prorrateado sugerido con la cuenta a la vista
  ("1 de 3 meses del trimestre jul-sep"). El campo de monto sigue
  editable.

## 4. Impacto

| Área | Qué cambia |
|---|---|
| Schema | 2 columnas + 2 valores de enum en `billing_cycle`. Aditivo. |
| `billing_period_for()` | Dos ramas nuevas; las tres actuales intactas. |
| Motor de reservas | **Nada.** La cobertura ya es un rango evaluado contra la fecha del turno. |
| `EXCLUDE` / duplicados | Nada: sigue siendo solapamiento de rangos, con rangos más largos. |
| Crédito de recupero | `END_OF_BILLING_PERIOD` (ADR-0025 resolución 4) pasa a poder significar "fin del trimestre". Es coherente pero es **un crédito que vive tres meses**. Ver riesgos. |
| Frontend | Formulario de plan + formulario de pago. |
| Tests | Cotización a mitad de ciclo, borde de mes, ciclo rodante que nunca prorratea, inmutabilidad con pagos. |

## 5. Migración / compatibilidad

Backfill nulo: `billing_period_months` nulo se lee como 1, así que ningún
pago ni plan existente cambia de comportamiento el día del deploy. Mismo
patrón que el `UNLIMITED` de ADR-0024 y que `makeup_credits_enabled`
default `false` de ADR-0025.

## 6. Riesgos

1. **El crédito de recupero de ADR-0025 con `END_OF_BILLING_PERIOD`.** En
   un plan trimestral, un crédito emitido el primer día del trimestre vive
   tres meses. El "riesgo abierto no técnico" que ADR-0025 ya registró
   (cuántos créditos vivos tolera la capacidad real) se multiplica por
   tres. Sugerencia: si se acepta esta propuesta, el default de
   vencimiento para planes de ciclo largo debería ser `END_OF_MONTH`, no
   `END_OF_BILLING_PERIOD`. **Es una decisión del Orchestrator.**
2. **`billing_anchor_month` es configuración con la que se puede errar.**
   Un anclaje mal elegido desalinea a todos los clientes del plan a la vez
   y es inmutable una vez que hay pagos. Mitigación: la pantalla muestra
   los cuatro trimestres resultantes antes de guardar.
3. **Prorratear el precio y no el período** deja una asimetría explicable
   pero real: la cobertura del cliente arranca antes de que fuera cliente.
   Para reservas no importa (no hay turnos pasados que reservar), pero sí
   se ve en un reporte de "quién estaba cubierto en julio".

## 7. Alternativas descartadas

- **Prorratear recortando `period_start` a la fecha de alta.** §3.2.
- **Una función `first_billing_period_for()` separada.** §3.2.
- **Prorrateo por días.** §3.4 — se deja como modo futuro, no como default.
- **Guardar el prorrateo en `payments` como columna propia.** `amount` ya
  guarda el monto real cobrado y `service_plans.price` el de lista: la
  diferencia es derivable y una tercera columna sería una segunda fuente
  de verdad, el mismo motivo por el que ADR-0024 rechazó
  `payments.weekly_quota_snapshot`.
