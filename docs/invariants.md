# Invariantes — lo que es verdad HOY

Mantenido por: Orchestrator. Última actualización: 2026-09-22 (tras ADR-0026).

**Este es el documento que hay que leer primero.** `decisions.md` tiene 26 ADR
y es **historia**: cómo llegamos acá, incluidas decisiones que otras
reemplazaron. Acá está solo el estado actual. Si los dos se contradicen, gana
`decisions.md` y este archivo está desactualizado — avisá.

**Cómo usarlo:** leé esto entero (son pocos minutos), y después **solo los 2 o
3 ADR que tu tarea toca**, que están linkeados en cada sección. No leas
`decisions.md` de punta a punta: es lo que hace lento cada arranque.

---

## Reglas que no se negocian

1. **El producto es genérico.** Nunca `Gym`, `Member`, `Trainer`, `Class` como
   modelo central — ni en nombres de columnas, ni en etiquetas de UI. Un
   consultorio, una cancha y un salón usan lo mismo. Si algo "solo aplica a
   gimnasios", el diseño está mal.
2. **Una sola función de decisión.** `evaluate_customer_booking()` decide si
   alguien puede reservar. El preview del mostrador, el del cliente y el
   confirm la comparten. Duplicarla es cómo se llega a "el preview decía que
   sí y el confirm dijo que no" (ADR-0018).
3. **`availableCapacity = maxCapacity - activeBookings`**, calculado, nunca
   persistido como fuente de verdad.
4. **Todo se valida contra la fecha local del turno, nunca contra `now()`.**
   Pagos, cuota, cobertura, créditos. Es el error más fácil del proyecto y ya
   costó varias correcciones (ADR-0013, ADR-0014, ADR-0024).
5. **Nada se borra.** Una `Booking` se cancela, un `Payment` se anula
   (`VOID`), un plan y un servicio se desactivan. El historial se conserva.
6. **Las validaciones viven en la base.** Toda tabla es escribible por
   PostgREST directo, así que un CHECK/trigger/índice es la defensa y el
   formulario es solo un buen mensaje de error (ADR-0020).
7. **Concurrencia real, no `if available > 0`.** `book_slot()` toma
   `FOR UPDATE` sobre la fila de la ocurrencia (ADR-0004). **Ojo:** ese lock
   serializa *esa ocurrencia*, no al cliente — dos ocurrencias distintas del
   mismo cliente no se serializan entre sí (ADR-0025).

---

## Modelo, en una pantalla

`Organization` (el tenant) → `OrganizationMember` (OWNER|STAFF) y `Customer`.
`Service` → N `ServicePlan` (la lista de precios) y N `ScheduleRule` (una por
día de semana, agrupadas por `group_id`). Una regla genera `SlotOccurrence`
materializadas con horizonte rodante de 90 días. Una `Booking` es de un
`Customer` sobre una `SlotOccurrence`. Una `RecurringBooking` es la
suscripción a **una** `ScheduleRule` y genera una `Booking` por ocurrencia.
`Payment` se ancla a `(customer, service_plan)`.

**Deprecado, no borrado:** `ServiceEntitlement` (ADR-0022/0024) y
`services.billing_type`/`billing_cycle`/`price` (ADR-0024). No se leen en
ningún camino de decisión ni de cobro. `services.payment_required` **sí** sigue
vigente: es el interruptor de "este servicio exige pago", no un precio.

---

## Qué compra un pago  → ADR-0024

- Un pago mensual **no** compra acceso ilimitado: compra lo que el
  `ServicePlan` diga. `plan_kind`: `DROP_IN` (turno suelto) | `WEEKLY_QUOTA`
  (N cupos fijos por semana) | `UNLIMITED` (el comportamiento viejo, con
  nombre).
- **"Veces por semana" ES "cantidad de series", no un contador.** Cada
  `ScheduleRule` es semanal y una `RecurringBooking` suscribe a una sola
  regla, así que *k* series **son** *k* reservas por semana. No hay contador
  que decrementar ni reconciliar, y la pregunta "¿semana calendario o
  corrida?" no existe porque nunca se calcula una ventana semanal.
- **La cuota se mide contra series en vigencia para la fecha local del slot**
  (`ACTIVE` + `start_date <= D` + `end_date is null or >= D`), nunca contra
  `status = 'ACTIVE'` a secas: una serie terminada en junio sigue `ACTIVE` en
  septiembre. Desempate `created_at, id`.
- Una reserva **extra** (la que no viene de una serie dentro de cuota) no la
  cubre el pago por período.
- **Desactivar un plan no invalida un pago ya hecho.** La cobertura se lee sin
  filtrar por `is_active`.
- `plan_kind` y `weekly_quota` son inmutables si el plan tiene pagos; el
  precio es libre.
- Un servicio con `payment_required = false` no puede tener planes de cuota.
- Doble cobro: `EXCLUDE` sobre períodos `PAID` solapados de
  `(customer, service)`, **más** índice único
  `(customer, slot_occurrence)` para los pagos sueltos — sacarlos del EXCLUDE
  sin ese índice los dejaría sin ninguna protección.

## Crédito de recupero  → ADR-0025 *(implementado y desplegado, salvo turno suelto)*

- Opt-in por organización (`makeup_credits_enabled`, default `false`).
- **Emiten** la cancelación de una fecha puntual con anticipación, y las
  cancelaciones originadas por la organización (esas sin exigir anticipación).
  **Una cascada de serie no emite nunca** — si no, cancelar y recrear la serie
  imprime créditos infinitos.
- Se consume como **última compuerta**, después de toda la cadena de cobertura:
  nunca se gasta un crédito si otra cobertura alcanzaba. Los jobs nunca
  consumen.
- `expires_on` se congela al emitir y se ancla a la fecha de la ocurrencia
  liberada, no a `issued_at`. Vencimiento derivado, no persistido.
- El veredicto positivo viaja como `(reason, makeup_credit_id)`, **nunca como
  un valor nuevo del enum**: seis funciones comparan `= 'OK'`.

## Identidad y multi-tenancy  → ADR-0005, ADR-0006, ADR-0026

- **La identidad se resuelve desde `auth.uid()`, jamás de un parámetro del
  caller.** Una RPC que recibe un `customerId` y confía en él es un IDOR.
- RLS de dos capas. Toda función `SECURITY DEFINER` necesita **control de
  autorización propio y `revoke execute from public, anon` explícito antes
  del `grant to authenticated`**. `GRANT` es **aditivo, no reemplaza el
  default**: en Postgres el EXECUTE queda en PUBLIC salvo que se revoque, así
  que ~38 `grant to authenticated` de las Fases 1-18 sumaban un permiso
  encima de PUBLIC en vez de restringir — toda RPC de negocio del proyecto
  era invocable sin sesión (ADR-0028, verificado en vivo contra producción:
  un request sin token ejecutó una escritura real y devolvió `204`). Un
  helper interno que solo llama otra `security definer` se revoca de todo
  rol externo; una función correcta por construcción que RLS necesita
  evaluar como `anon` (`is_organization_member` y similares) se deja con
  PUBLIC a propósito.
- **Cuidado con la lógica de tres valores.** `profile_id <> auth.uid() and
  not es_miembro` da `NULL` cuando `profile_id` es `NULL`, y un `if NULL`
  **no entra**, salteando la autorización. Escribir siempre
  `if <es el dueño> / elsif <es miembro> / else raise`.
- Un `Customer` puede no tener `profile_id` (cliente gestionado): existe, se
  le agenda y se le cobra, pero no puede iniciar sesión.
- Público sin login: servicios, fechas, horarios y disponibilidad. Privado
  siempre: nombres de clientes, emails, teléfonos, pagos y reservas
  individuales.

## Cancelaciones  → ADR-0010, ADR-0025

- `cancelled_by` es **FK a `profiles`** (quién ejecutó), no un enum. El actor
  categórico lo carga `cancellation_reason`: `CUSTOMER_REQUEST |
  SLOT_CANCELLED | RULE_DISCONTINUED | SERIES_CANCELLED`.
- **El motivo se deriva del actor, no se acepta de un caller `authenticated`.**
  Aceptarlo era un crédito de recupero gratis.
- Cancelar una ocurrencia puntual no toca la serie. Discontinuar una regla
  cascadea a `Booking` **y** a `RecurringBooking`.

## Disponibilidad pública  → ADR-0008

`publicAvailabilityDisplay` por organización: `EXACT | LIMITED | BOOLEAN`. El
número exacto **solo** se expone en `EXACT`, y la regla se aplica en la base,
no se confía en que el frontend la recuerde.

## Timezones  → ADR-0014

`timestamptz` siempre, convertido desde `weekday + localStartTime +
Organization.timezone` en SQL al generar. Nunca un offset cacheado. 01:30 UTC
del 22 sigue siendo el 21 en Montevideo, y equivocarse pone una clase nocturna
en el día equivocado.

---

## Stack y despliegue  → ADR-0002, ADR-0016, ADR-0017, ADR-0021

Next.js App Router + TypeScript strict + Supabase (Postgres, Auth, RLS, RPC) +
Zod + Tailwind. **Sin ORM.** Dos repos: `Reservaste/backend` (schema + paquete
de dominio) y `Reservaste/frontend`, que lo consume desde `#main` — **si
cambiás tipos del paquete, el frontend no los ve hasta que el backend esté
pusheado**. Producción: droplet propio en DigitalOcean
(`161-35-63-60.sslip.io`) + Supabase free, con `pg_dump` nocturno.

## Sistema visual  → `architecture.md` §Sistema visual

`components/ui/` es el único lugar que decide cómo se ve un control. Tres
reglas duras: un solo anillo de foco, 44px mínimo en móvil, y los errores de
formulario van por `FormError` (lleva `role="alert"`), nunca un `<p>` rojo.
Casi ninguna lista de este producto es una tabla: `DataList`.

---

## Cómo se trabaja acá

1. Leé este archivo + los ADR que tu tarea toca. No `decisions.md` entero.
2. Cambios estructurales (dominio, schema, auth, multi-tenancy, contratos de
   API, estrategia de slots/pagos/recurrencia) **los aprueba el Orchestrator**
   y quedan como ADR. Un subagente que necesita uno escribe una **propuesta**,
   no el cambio.
3. Antes de cerrar una fase: tests → code review → aprobación.
4. Números reales en los reportes. Si no pudiste correr algo, decilo — no lo
   estimes.
