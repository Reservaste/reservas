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

## Cómo se reporta

`qa-testing-agent` corre las suites y reporta fallas al Orchestrator con
el caso puntual que falló (no solo "tests failed"). Ningún cierre de fase
se aprueba con fallas pendientes sin decisión explícita del Orchestrator.
