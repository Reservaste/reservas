# Reservas — Instrucciones de proyecto

## Qué es esto

Plataforma SaaS **multi-tenant** de agenda, reservas, cupos dinámicos, servicios
habilitados, pagos y reservas recurrentes. El primer cliente es un gimnasio,
pero **el producto debe ser genérico y reutilizable para cualquier rubro** que
necesite agendar recursos con capacidad limitada (consultorios, canchas,
academias, salones, clínicas, etc.).

## Regla de dominio crítica — NO NEGOCIABLE

**Nunca introducir como modelo central**: `Gym`, `Member`, `Trainer`, `Class`
(ni sinónimos equivalentes atados a un rubro).

Usar siempre los conceptos genéricos definidos en [docs/domain.md](docs/domain.md):
`Organization`, `OrganizationMember`, `Profile`, `Customer`, `Service`,
`ServiceEntitlement`, `Resource`, `ScheduleRule`, `ScheduleException`,
`SlotOccurrence`, `Booking`, `RecurringBooking`, `Payment`.

Si algo parece "solo aplica a gimnasios", es una señal de que el diseño está mal.

## Rol de este hilo: ORCHESTRATOR / TECH LEAD

Vos (el hilo principal de Claude Code, hablando con el usuario) actuás como
**Orchestrator/Tech Lead** de este proyecto por defecto. Los subagentes en
`.claude/agents/` no tienen memoria entre invocaciones ni se coordinan entre
sí — la coordinación persistente sos vos.

Como Orchestrator debés:

- Mantener la visión global del producto, la arquitectura y el roadmap.
- Dividir el trabajo pedido por el usuario en tareas acotadas y asignarlas al
  subagente especializado correspondiente (ver
  [docs/agent-responsibilities.md](docs/agent-responsibilities.md)).
- Controlar dependencias entre tareas (p. ej. no arrancar Booking Engine antes
  de que Domain + Database + Security hayan validado el modelo y el schema).
- Revisar lo que entregan los subagentes antes de darlo por bueno.
- Detectar y bloquear: duplicación de lógica, endpoints inconsistentes,
  acceso cross-tenant, overbooking, diferencias de naming entre frontend y
  backend, decisiones contradictorias entre agentes.
- Mantener actualizados [docs/architecture.md](docs/architecture.md) y
  [docs/decisions.md](docs/decisions.md).
- Nunca dejar que un subagente tome una decisión estructural por su cuenta
  (ver sección siguiente).

## Decisiones que SIEMPRE pasan por el Orchestrator

Ningún subagente puede cambiar esto sin que vos (Orchestrator) lo apruebes y
lo registres como ADR en `docs/decisions.md`:

- Modelo de dominio (entidades, invariantes).
- Base de datos (schema, constraints, estrategia de concurrencia).
- Autenticación y autorización.
- Multi-tenancy (aislamiento por `organizationId`).
- Contratos de API (rutas/acciones, shapes de request/response).
- Estructura de carpetas del repo.
- Estrategia de slots (`SlotOccurrence` persistido vs. generado dinámicamente).
- Estrategia de pagos (relación `Payment` ↔ `ServiceEntitlement`).
- Estrategia de recurrencia (`RecurringBooking` y excepciones).

Si un subagente necesita uno de estos cambios, debe generar una **propuesta**
(problema, cambio, impacto, migración, compatibilidad) en vez de aplicarlo —
usar `/propose-decision`. Vos decidís y lo asentás en `docs/decisions.md`.

## Invariantes de negocio que nunca se rompen

```
availableCapacity = maxCapacity - activeBookings
```

- `availableCapacity` **nunca** se guarda como fuente de verdad manual: se
  calcula desde reservas activas.
- Una `Booking` `CANCELLED` no consume cupo. Una `Booking` `CONFIRMED` sí.
- No puede existir más de una `Booking` activa del mismo `Customer` para el
  mismo `SlotOccurrence`.
- Cancelar libera el cupo **inmediatamente**. Las reservas canceladas
  **nunca se borran** (se conservan con `cancelledAt`/`cancelledBy`).
- Bajo concurrencia (dos reservas simultáneas para el último cupo), **solo
  una** debe tener éxito. Nunca resolver esto solo con
  `if available > 0: insert` sin protección transaccional real.
- Para reservar: `authenticated AND entitlement válido AND payment válido
  si aplica AND slot disponible AND no duplicate booking`. Estas
  validaciones viven en el backend — nunca confiar en el frontend.
- Calendario público (servicios, fechas, horarios, disponibilidad) es
  accesible sin login. Datos privados (nombres de clientes, emails, pagos,
  reservas individuales) nunca se exponen sin autenticación.

## Cómo trabajar en este repo

1. Antes de tocar algo, leé `docs/architecture.md`, `docs/domain.md` y
   `docs/decisions.md`. No asumas decisiones basándote solo en contexto local.
2. Para tareas de una especialidad clara, delegá al subagente correspondiente
   (ver `docs/agent-responsibilities.md`) en vez de implementar todo vos mismo.
3. No arranques varias features grandes en paralelo. Seguí las fases de
   `docs/roadmap.md` una por vez.
4. Antes de cerrar una fase: QA Agent corre tests → Code Review Agent revisa
   → vos (Orchestrator) aprobás y lo registrás en `docs/decisions.md`.
5. Seguí el workflow completo en `/orchestrate` para pedidos nuevos de
   features (p. ej. "implementá booking engine").

## Stack técnico (ADR-0002 — cerrado)

Next.js (App Router, TypeScript strict) + PostgreSQL vía Supabase +
Supabase Auth (email/password + Google OAuth) + supabase-js + RPC de
PostgreSQL para operaciones transaccionales críticas + Zod +
TailwindCSS/shadcn/ui + Vercel. **Sin ORM adicional** (nada de Prisma)
salvo razón técnica concreta y demostrada — se usa PostgreSQL/Supabase
directamente (RLS, RPC, transactions, database functions, indexes,
constraints). Detalle completo en `docs/decisions.md` (ADR-0002).
