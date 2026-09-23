# Roadmap de implementación

Mantenido por: Orchestrator. Una fase no arranca hasta que la anterior esté
funcional, testeada, tipada y documentada, y haya pasado el checklist de
`/phase-review` (QA → Code Review → aprobación del Orchestrator).

## PHASE 0 — Architecture — **COMPLETE** (2026-09-14)

Dos rondas de revisión coordinada con los 7 agentes técnicos. Modelo de
dominio, schema conceptual, RLS, concurrencia, slots, recurrencia,
timezones, pagos por período y flujo de login-durante-reserva resueltos
como ADR-0001 a ADR-0015 en `decisions.md`. Sin código de producto
todavía — Phase 1 es la primera fase de implementación.

## PHASE 1 — Auth + Organizations + Roles — **COMPLETE** (2026-09-14)

Implementado (2026-09-14): schema + RLS + RPC en
[`Reservaste/backend`](https://github.com/Reservaste/backend), signup/
login (email+password + Google OAuth) + onboarding + dashboard mínimo en
[`Reservaste/frontend`](https://github.com/Reservaste/frontend).

Validado con tests reales, no solo revisión de código — el suite de
integración de RLS cross-tenant (`backend/test/rls.cross-tenant.test.ts`)
encontró y forzó a corregir dos bugs reales antes de este punto:
1. Recursión infinita en la policy de `organization_members` (una policy
   se referenciaba a sí misma) — resuelto con funciones `SECURITY
   DEFINER` (`is_organization_member`, `is_organization_owner`).
2. `organizations` permitía `INSERT` directo saltando la RPC atómica,
   pudiendo crear una organización sin `OWNER` — policy de INSERT
   eliminada, solo la RPC puede crear organizaciones.

Pendiente para cerrar la fase formalmente (no bloquea seguir
desarrollando, pero sí un `/phase-review` completo):
- Conectar un proyecto Supabase real (o mantener el local) y configurar
  credenciales de Google OAuth en el dashboard de Supabase — hoy solo
  validado contra `supabase start` local.
- CI en cada repo (typecheck + tests en cada push/PR) — todavía se corre
  todo a mano.
- `code-review-agent` formal sobre el diff (hoy la revisión la hizo el
  Orchestrator directamente).

## PHASE 2 — Services + Resources + Entitlements — **COMPLETE** (2026-09-14)

Implementado: schema + RLS + guards de integridad cross-tenant para
`Service`, `Resource`, `ServiceEntitlement` (con vigencia TIME/CREDITS y
`requiresActivePayment`, ADR-0013), validado con 9 tests de integración
nuevos (18 en total). UI admin mínima (`/org/[slug]/services`,
`/org/[slug]/resources`) para poder probarlo de punta a punta.

**Alcance deliberadamente recortado**: no se construyó UI de
`ServiceEntitlement` todavía — otorgar un entitlement necesita un
`Customer` al cual otorgárselo, y la gestión de clientes es un tema de
Phase 8 ("Clientes"). El modelo de datos de `ServiceEntitlement` está
completo y probado (incluye la función pura `isEntitlementCurrentlyValid`
en `@reservaste/domain`), listo para que Phase 7/8 lo conecten sin
rediseñar nada.

Sin booking todavía (Phase 5).

## PHASE 3 — ScheduleRules + SlotOccurrences — **COMPLETE** (2026-09-14)

Implementado: generación de `SlotOccurrence` con horizonte rodante
(pg_cron diario + sincrónico al crear/editar una regla, ADR-0009),
conversión de timezone vía `AT TIME ZONE` recalculada por fecha
(ADR-0014), `ScheduleException` con reconciliación retroactiva sobre
ocurrencias ya materializadas, y las RPCs `cancel_slot_occurrence`/
`discontinue_schedule_rule` con la cascada de `domain.md`. 7 tests de
integración nuevos (25 en total) — incluyen verificar que la conversión
de timezone da la hora local correcta, no solo que insertó algo. UI
mínima para crear horarios y ver las ocurrencias generadas.

## PHASE 4 — Public calendar — **COMPLETE** (2026-09-14)

`organizations_public`/`services_public` (vistas que exponen solo
columnas seguras) + `get_public_availability()` (RPC, no vista — la
regla de ADR-0008 de nunca exponer el número exacto salvo en modo
`EXACT` se aplica en la base, no se confía en que el frontend recuerde
sacarlo). Página pública `/[organizationSlug]`. 9 tests nuevos (33 en
total) con un cliente **anónimo real** confirmando que no ve nada de
`customers`/`organization_members`/tablas base.

**Nota para Phase 5**: `active_bookings` está hardcodeado a `0` en
`get_public_availability()` — no existe `Booking` todavía. Booking
Engine tiene que reemplazar eso por un conteo real.

## PHASE 5 — Booking Engine + cancellation + capacity — **COMPLETE** (2026-09-14)

`can_customer_book()` (fuente única de verdad, resuelta desde
`auth.uid()`, nunca un `customerId` del caller — ADR-0005) +
`book_slot()` (RPC atómica, `FOR UPDATE` real — ADR-0004) +
`cancel_booking()`. Cierra los TODOs de Phase 3 (cascada a bookings al
cancelar una ocurrencia/regla) y Phase 4 (`get_public_availability` ya
cuenta reservas reales, no hardcodeado a 0). 10 tests nuevos (42 en
total) — el que importa: dos customers llamando `book_slot()` en el
mismo instante por un cupo de capacidad 1, exactamente uno gana,
verificado con un conteo de filas real después.

**Sin UI de cliente todavía**: reservar/cancelar desde la UI necesita el
flujo completo de identidad pública + booking intent (ADR-0015), que es
Phase 9. Construir una versión a medias ahora sería el mismo error de
scope creep que evité en Phase 2 con `ServiceEntitlement`. El motor está
completo y probado con 10 tests de integración reales contra las RPCs
directamente — eso es lo que esta fase pedía ("Booking Engine", no UI).

## PHASE 6 — Recurring Bookings — **COMPLETE** (2026-09-14)

`RecurringBooking` (suscripción a una `ScheduleRule`) genera una
`Booking` por ocurrencia, con `generate_recurring_booking()` como RPC
hermana de `book_slot()` — no un insert plano, porque un customer puede
estar en dos series que caen en la misma ocurrencia (lo que
`database-agent` predijo en Phase 0). Cancelar una fecha = `cancel_booking()`
sobre esa fila; cancelar la serie cascadea solo a futuras. Discontinuar
la regla ahora cascadea hasta `RecurringBooking`, cerrando el gap que
`domain-architect` marcó. 9 tests nuevos (51 en total), incluido el caso
Mathias/CrossFit exacto.

Dos bugs reales que atraparon los tests: `ON CONFLICT` contra un índice
parcial necesita repetir el predicado del índice (42P10), y el hook de
generación solo debe dispararse para ocurrencias realmente creadas.

## PHASE 7 — Payments — **COMPLETE** (2026-09-14)

`Payment` ligado a `ServiceEntitlement` (no a `Service`, ADR-0013), con
validación contra la fecha **del slot**, nunca contra "hoy" — hay un test
específico para eso porque es el error más fácil de cometer.
`PAYMENT_REQUIRED` es un motivo distinto de `NO_ENTITLEMENT`: esa
diferencia de cara al cliente es justamente el motivo de separar permiso
de pago.

Además implementa el consumo de créditos que `domain.md` especificaba
desde el principio pero ninguna fase había construido, y cierra un
agujero de Phase 6: `generate_recurring_booking()` solo miraba capacidad,
así que una serie seguía confirmando fechas para alguien con la membresía
vencida. Ahora una serie con bono de 2 clases se auto-limita.

`EXCLUDE` constraint contra períodos `PAID` solapados (doble cobro);
huecos permitidos a propósito. Sin política de DELETE en `payments` —
anular es la única forma de retractar uno. 10 tests nuevos (61 en total).

## PHASE 8 — Admin UI — **COMPLETE** (2026-09-14)

Agenda (vista día/semana, ocupación `8 / 12 · 4 disponibles`, anotados por
slot, anotar cliente, cancelar reserva, cambiar capacidad, cancelar
ocurrencia), Clientes (alta por email + detalle con entitlements y pagos),
Equipo (invitar/revocar, OWNER-only), Configuración.

El backend sumó: guard de capacidad (`domain.md` lo pedía desde Phase 0 y
ninguna fase lo había implementado — nada impedía bajar capacidad por
debajo de las reservas confirmadas), `enroll_customer_by_email` /
`invite_member_by_email` (staff no puede leer `profiles` ajenos, a
propósito), `revoke_member` con protección del último OWNER,
`admin_book_for_customer` (mostrador), y las RPCs de lectura de la Agenda.
6 tests nuevos (67 en total).

## PHASE 9 — Customer UI — **COMPLETE** (2026-09-15)

Flujo público completo de ADR-0015: `/[slug]/reservar` (servicio → día →
horario, mobile-first, sin login) → "Reservar" lleva el intent a login
como query param → vuelve a `/reservar/confirmar`, que **revalida y pide
confirmación explícita**, nunca reserva sola. `returnTo` se valida como
destino (no se "limpia"), con tests unitarios propios de los vectores
reales de open redirect.

Portal `/me`: reservas (cancelar), servicios (marca "activo pero
impago", que es justo el estado que justifica separar permiso de pago) y
pagos. Backend: `my_bookings`/`my_entitlements`/`my_payments`/
`public_slot_detail` — un cliente no puede leer `slot_occurrences` ni
`services`, así que el portal necesita sus propias RPCs. 8 tests de
integración (75) + 8 unitarios de `returnTo`.

Bug de ruteo que destapó esta fase: `/dashboard` mandaba a cualquiera sin
organización a "creá tu negocio" — la puerta equivocada para un cliente.

## PHASE 10 — Tests + security review + polish

Cobertura de edge cases (`docs/testing.md`), revisión de seguridad
completa, pulido de UI/UX.

## PHASE 11 — Reserva fija desde el mostrador — **COMPLETE** (2026-09-15)

Ver ADR-0018. El motor de recurrencia existía desde Phase 6 pero era
autoservicio del cliente y **ninguna pantalla lo llamaba**. Se agregó el
camino de administración: `admin_create_recurring_booking` /
`admin_preview_recurring_booking` (membresía, no propiedad) y
`schedule_rule_standing_reservations` para ver quién tiene cada cupo fijo
y cuántas de sus próximas fechas se están confirmando de verdad.

`can_customer_book()` se refactorizó sobre `evaluate_customer_booking()`
para que el preview del mostrador y el del cliente no puedan divergir.
`bookings.not_generated_reason` distingue "está llena" de "falta el
pago", que el portal mostraba igual.

UI: bloque de horario fijo por regla en
`/org/[slug]/services/[id]/schedule` (elegir cliente → ver próximas 8
fechas con su motivo → confirmar → quitar). 10 tests de integración
(100 en total).

## PHASE 12 — Fechas pendientes que se reconcilian solas — **COMPLETE** (2026-09-15)

Ver ADR-0019. Phase 11 dejó la serie autolimitada pero sin vuelta atrás:
registrar el pago no desbloqueaba las fechas ya marcadas
`NOT_GENERATED`. Ahora cualquier escritura que amplíe lo que el cliente
puede reservar (pago `PAID`, alta de permiso, reactivación, recarga de
créditos) reevalúa sus fechas futuras pendientes, en orden cronológico y
con el mismo lock que `book_slot`.

Queda abierto y documentado: quién se queda con un cupo liberado por una
cancelación (política de equidad, no bug). 9 tests de integración (109 en
total). `hookTimeout` del config de integración subido a 30s — los
`beforeAll` saturaban GoTrue con 12 archivos en paralelo y fallaban
archivos enteros sin una sola aserción rota.

## PHASE 13 — Identidad por organización — **COMPLETE** (2026-09-15)

Ver ADR-0020. Color de acento y logo por organización, en la página
pública de reservas y en el panel. Los colores semánticos no se tocan.

El contraste del texto sobre el acento se **deriva** por luminancia
(`brandTheme()` en `@reservaste/domain`), no se elige. El formato del
color lo enforza un CHECK y el path del logo está atado al id de la
propia organización — la columna es escribible por PostgREST directo, así
que la validación del frontend no cuenta. Bucket público, escritura solo
del OWNER, sin SVG.

9 tests de integración (constraints + políticas de storage) y 12
unitarios de paleta. 118 de integración, 35 unitarios.

## ITERACIÓN 2 — Fases A y B — **COMPLETE** (2026-09-21)

Ver ADR-0022. Se eliminó la habilitación manual de servicios; el Service
pasó a ser la unidad de cobro y el pago se ancla a (cliente, servicio).

**Fase A** (migración 14, aditiva): `billing_type`/`billing_cycle`/
`price`/`payment_required`/`color` en Service, `service_id` en Payment
con backfill, asistencia en Booking, `group_id` en ScheduleRule. Un
trigger puente mantuvo vivos a los llamadores viejos.

**Fase B** (migración 15): `payment_covers_slot()` como única fuente de
verdad, validada contra la fecha del slot. `billing_period_for()` es
donde viven CALENDAR_MONTH y ROLLING_MONTH — y a propósito es sobre
*escribir* un pago, no sobre leerlo, así el motor de reservas nunca
aprende de modalidades de cobro. Las ocurrencias terminadas dejan de
aceptar reservas (corte en `end_at`, no en `start_at`, para no romper al
que llega tarde). Reconciliación re-anclada a pagos.

Se quitaron los bonos (CREDITS) por decisión explícita. La tabla
`service_entitlements` queda deprecada, no borrada.

Dos bugs que atraparon los tests, ambos míos: dropear
`resolve_bookable_entitlement` dejó a `admin_book_for_customer` llamando
a una función inexistente, y renombrar el valor del enum dejó a
`schedule_rule_standing_reservations` compilando pero fallando al
invocarse. 120 tests de integración, 35 unitarios.

**Pendiente de esta iteración:** C (horarios multi-día), D (calendario
compartido), E (asistencia), F (módulo de pagos), G (pulido público).

## ITERACIÓN 2 — Fases C a G — **COMPLETE** (2026-09-21)

Ver ADR-0023.

**C** Horarios multi-día: una acción crea N reglas con `group_id`
compartido. El modelo sigue siendo una regla por día (ADR-0022).

**D** `ScheduleCalendar` compartido: día/semana/semana laboral/mes en el
panel, día/semana en el público. Tres pantallas, un componente.
`lib/calendar.ts` con 15 tests, los de timezone incluidos.

**E** Asistencia: pasar lista en pantalla completa con objetivos de 56px,
tab de Asistencia por servicio con historial, y resúmenes por ocurrencia.

**F** Módulo de Pagos: listado por cliente con filtros y búsqueda,
desglose por servicio, marcar pagado. Más configuración de cobro por
servicio (punto 14), que hasta ahora nadie escribía.

**G** Agenda pública rediseñada mobile-first: tira de días más lista, no
la grilla de admin achicada.

131 tests de integración, 35 unitarios de dominio, 27 unitarios de
frontend. Desplegado.

**Sin cubrir:** las pantallas nuevas no tienen tests propios (lo probado
son las RPC), y no las vi renderizadas con sesión iniciada.

## ITERACIÓN 3 — Feedback del primer cliente — **EN CURSO** (desde 2026-09-21)

Plan completo en [`iteration-3-plan.md`](iteration-3-plan.md). Origen: feedback
del primer cliente real (estudio de pilates) después de la demo.

**El hallazgo:** los pedidos de planes, frecuencia semanal, cupo fijo,
recupero y "sin cupo pagás" son **un solo cambio**, no cinco. Hoy un pago
mensual cubre cualquier cantidad de turnos del servicio en el período
(ADR-0022). El cliente quiere que compre **N cupos fijos por semana**, y que
lo que exceda se pague o se recupere. Eso invierte qué compra un pago, y es el
ADR principal de la iteración.

Decidido con el usuario (2026-09-21): se exige cuenta para el autoservicio más
cliente gestionado por el dueño sin cuenta; la política de recupero la
configura el dueño del local, no el producto; la pasarela de tarjeta se
posterga y solo se deja la arquitectura lista.

| Fase | Qué | Estado |
|---|---|---|
| L0 | Sistema visual base (tokens + primitivas que faltan) | **COMPLETA** (2026-09-22) |
| H | `ServicePlan`: precio + veces por semana (ADR-0024) | **COMPLETA** (2026-09-22) — desplegada |
| I | Liberar y recuperar: créditos de recupero (ADR-0025) | Diseño en curso |
| J | Cliente gestionado + activación por WhatsApp (ADR-0026) | Pendiente |
| K | Cobro con tarjeta (ADR-0027) — parcial, sin pasarela | Pendiente |
| L | Pulido de UI/UX de pantallas | Pendiente |
| M | Tests de pantallas, revisión de seguridad, cierre | Pendiente |

L0 va primero a propósito: las pantallas de H, I y J son nuevas, y
construirlas sobre las 5 primitivas actuales obligaría a rehacerlas en L.

## Regla de completitud

No arrancar varias fases en paralelo. Cada fase cerrada = funcional +
testeada + tipada + documentada + sin errores conocidos, aprobada vía
`/phase-review`.
