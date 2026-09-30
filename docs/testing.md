# Testing

Mantenido por: `qa-testing-agent`. Revisión de hallazgos: `code-review-agent`.
Aprobación de cierre de fase: Orchestrator.

## Suites obligatorias antes de cerrar cualquier fase

`lint`, `typecheck`, `unit tests`, `integration tests` (comandos concretos
se agregan acá una vez elegido el stack en Phase 0).

## Casos de prueba obligatorios (P0)

### Booking completo
Capacidad 10, 10 reservas activas → el intento 11 se rechaza.

### Cancelación libera cupo
Capacidad 10, 10 reservas activas, uno cancela → quedan 9 reservas activas
y 1 lugar disponible.

### Concurrencia
9/10 reservas activas, dos usuarios reservan el último cupo
simultáneamente → solo uno obtiene la reserva, el otro recibe rechazo
claro (no error genérico).

### Servicio no habilitado
Usuario autenticado sin `ServiceEntitlement` activo intenta reservar →
rechazo.

### Pago vencido
`ServiceEntitlement` que requiere pago vigente y el pago está vencido →
rechazo.

### Calendario público
Usuario sin login consulta servicios/disponibilidad → puede verlos; no
puede ver clientes, bookings individuales ni pagos; no puede reservar.

### Cancelación recurrente individual
`RecurringBooking` semanal (ej. lunes fijo), se cancela una fecha puntual
→ esa fecha libera su cupo; las siguientes semanas de la serie siguen
activas sin cambios.

### Cross-tenant
Usuario/miembro de la Organization A intenta acceder a datos o acciones
de la Organization B → bloqueado (403 / no encontrado), sin importar si el
ID se pasa manualmente.

### Etiqueta de pago acotada al período vigente (Fase 25)
Cliente con el mes en curso pago y una reserva fija: la ventana rodante es
de 90 días (ADR-0009) y un pago mensual cubre uno → `upcoming_unpaid` debe
ser **0** y las fechas de los meses siguientes deben contarse en
`upcoming_beyond_period`, no como deuda. Caso espejo obligatorio: el mismo
cliente **sin** pagar → `upcoming_unpaid > 0`. Sin el segundo caso, acotar
el horizonte apaga el aviso real.

### Cobertura evaluada en la fecha local del turno, en la frontera del mes
Clase de las 21:00 del último día del mes pago (00:00 UTC del día 1
siguiente, ADR-0014) → `can_customer_book` da `OK`. Es el caso que
distingue "el motor está mal" de "la etiqueta está mal".

### Pago duplicado, también sin estar PAID (Fase 25)
Dos pagos del mismo cliente, servicio y período: el segundo se rechaza aunque
ambos sean `PENDING`/`OVERDUE` (`PAYMENT_DUPLICATE_PERIOD`). Dos pagos de
turno suelto sobre la misma ocurrencia, idem
(`PAYMENT_DUPLICATE_OCCURRENCE`). Contracaso obligatorio: con el primero
anulado (`VOID`), volver a cargar el período **sí** tiene que entrar —
corregir un error de carga no puede quedar bloqueado.

### Meses distintos nunca chocan
Pagos del mismo cliente y servicio en meses consecutivos, en cualquier
combinación de `PAID`/`PENDING` → todos entran. Es el test que separa "el
`EXCLUDE` está mal calculado" de "el formulario mandó el mes equivocado".

### Plan que cubre varios servicios (ADR-0029)
Un pago de un plan `applies_to_all_services` (con `service_id` nulo por
diseño) se registra, genera una fila de `payment_service_coverage` por
servicio cubierto y **aparece** en `customer_payment_detail()`. Un pago
invisible es indistinguible de uno que no se registró.

### El crédito de recupero es opt-in y hay que poder encenderlo
Organización recién creada → `makeup_credits_enabled = false` y liberar un
cupo **no** emite crédito. Con el interruptor encendido, el mismo flujo sí
emite. Todo test de créditos que encienda el interruptor en su setup oculta
este caso: hace falta el que no lo enciende.

### Un crédito sirve en cualquier turno del mismo servicio
Crédito emitido al liberar el lunes → se usa en el miércoles del mismo
servicio, y `can_customer_book_detail()` lo informa **antes** de gastarlo.
Contracaso: si otra cobertura ya alcanzaba, el veredicto es `OK` con
`makeup_credit_id` nulo (nunca se gasta un crédito si no hacía falta).

## Cómo se reporta

`qa-testing-agent` corre las suites y reporta fallas al Orchestrator con
el caso puntual que falló (no solo "tests failed"). Ningún cierre de fase
se aprueba con fallas pendientes sin decisión explícita del Orchestrator.
