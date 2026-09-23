# CHARTER

## Producto
Plataforma SaaS multi-tenant de agenda, reservas con cupos dinámicos y pagos. **Genérica:** el
primer cliente es un gimnasio, pero eso es configuración de una `Organization`, no dominio.

## Stack y estructura
Next.js 16.3.5 (App Router, React 19.2.8, server actions) + TypeScript estricto + Zod 4.6.5 +
Supabase (Postgres, Auth, RLS; `@supabase/supabase-js` 2.116.0) + Vitest 5. Dos paquetes separados,
cada uno con su propio repo git y `package.json` (no hay `package.json` raíz):
- `backend/supabase/migrations/`: schema, RLS y RPCs (backend-engineer).
- `backend/src/`: paquete `@reservaste/domain` — tipos, invariantes y schemas Zod (backend-engineer).
  El frontend lo consume como dependencia de GitHub, no lo duplica.
- `frontend/app/`, `frontend/components/`: UI pública, portal y admin (frontend-engineer).
- `frontend/app/actions/`: server actions (backend-engineer).

Comandos (parado en el paquete correspondiente):
- `backend/`: `npm run typecheck`, `npm test`, `npm run test:integration`. **Sin script `lint`.**
- `frontend/`: `npm run typecheck`, `npm run lint`, `npm test`, `npm run build`, `npm run dev`.

## Entidades
`Organization`, `OrganizationMember`, `Profile`, `Customer`, `Service`, `ServiceEntitlement`,
`Resource`, `ScheduleRule`, `ScheduleException`, `SlotOccurrence`, `Booking`, `RecurringBooking`,
`Payment`. Detalle en `docs/domain.md`.

## Roles
OWNER / STAFF (vía `OrganizationMember`) · CUSTOMER (vía `Customer`) · anónimo.

## Niveles de acceso
- **PUBLIC** (sin auth): servicios y disponibilidad agregada.
- **CUSTOMER** (auth): bookings, servicios y pagos propios.
- **ADMIN** (membership OWNER/STAFF): gestión de servicios, horarios, clientes y pagos.

## Invariantes no negociables
- `availableCapacity = maxCapacity - activeBookings`: se calcula, nunca se persiste a mano.
- `SlotOccurrence` se persiste como filas materializadas, generadas con horizonte rodante (ADR-0003,
  Aceptada) — nunca calculada dinámicamente.
- La reserva se crea solo vía RPC atómica (ADR-0004). Una booking activa por cliente y slot.
- La capacidad nunca baja de las reservas activas.
- Cancelar = `CANCELLED` + libera cupo. Nunca se borra.
- Pago ≠ permiso. `canCustomerBook()` es la única puerta (ADR-0005, ADR-0013).
- RLS por `organizationId` en toda tabla de negocio.
- Horas en UTC. Ocurrencias en la zona de la `Organization`.

## Flujo de trabajo
Orchestrator → engineer(s) → security-engineer (si hay auth/RLS/roles/pagos) → qa-engineer → reviewer
→ release-manager.

## Git
`backend/` y `frontend/` son dos repos separados (sin repo/`package.json` raíz), cada uno con su
propio `development`/`main`. Commits a `development` (`git -C backend ...` / `git -C frontend ...`);
`main` solo por PR a pedido. Mensajes en imperativo.

Cuando cambia `@reservaste/domain` (paquete en `backend/src/`, consumido por `frontend/` vía
GitHub contra `backend#main`): 1) commit y push de `backend` primero (y su PR a `main` si aplica),
2) recién ahí actualizar la referencia en `frontend/package.json`, 3) verificar `npm run build` en
`frontend/` con la versión nueva, 4) solo si pasa, commit y push de `frontend`. Nunca al revés.
