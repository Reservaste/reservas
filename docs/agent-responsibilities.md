# Agentes y responsabilidades

Este documento es el índice de coordinación del sistema de agentes. Todo
agente debe leerlo antes de trabajar, junto con `architecture.md`,
`domain.md` y `decisions.md`.

## Cómo está organizado el sistema

- **Orchestrator / Tech Lead** = el hilo principal de Claude Code (ver
  `CLAUDE.md`). No es un subagente separado: mantiene la visión global,
  reparte trabajo, revisa entregables y es el único que aprueba cambios
  estructurales.
- **12 subagentes especializados** viven en `.claude/agents/*.md`. Cada uno
  tiene responsabilidades, límites y un set de `tools` acotado. Se invocan
  con la herramienta `Agent` (`subagent_type: <nombre>`) o vía `/orchestrate`.
- **3 slash commands** en `.claude/commands/` formalizan el workflow:
  `/orchestrate`, `/propose-decision`, `/phase-review`.

## Catálogo de agentes

| Agente | Cuándo se usa | Puede editar | No puede |
|---|---|---|---|
| `domain-architect` | Diseñar/cambiar entidades, invariantes de negocio | `docs/domain.md`, tipos de dominio | Elegir DB/framework, tocar auth |
| `database-agent` | Schema, migraciones, constraints, RLS, concurrencia | `docs/database.md`, migraciones SQL | Cambiar el modelo de dominio sin acuerdo con Domain Architect |
| `auth-security-agent` | Auth, roles, RLS, multi-tenancy, IDOR, público vs. privado | `docs/security.md`, capas de auth/policies | Definir el modelo de dominio o el schema base |
| `booking-engine-agent` | Crear/cancelar reservas, cupos, duplicados, concurrencia, recurrencia | Código del motor de reservas, tests | Cambiar estrategia de slots sin Scheduling Agent |
| `scheduling-agent` | ScheduleRule/ScheduleException/SlotOccurrence, disponibilidad | Código de generación de horarios | Decidir solo si SlotOccurrence se persiste (requiere Database + Orchestrator) |
| `payments-entitlements-agent` | ServiceEntitlement, Payment, `canCustomerBook()` | Código de entitlements/pagos | Asumir pago = permiso |
| `backend-api-agent` | Server actions / API routes / validación Zod | Capa de transporte/API | Duplicar lógica de dominio en el endpoint |
| `frontend-admin-agent` | Panel admin (Agenda, Reservas, Clientes, Servicios, Horarios, Pagos, Equipo, Configuración) | UI admin | Usar lenguaje específico de gimnasio en componentes base |
| `frontend-customer-agent` | Flujo público de reserva + portal cliente, mobile-first | UI pública/cliente | Confiar validaciones de negocio solo en frontend |
| `ui-ux-agent` | Sistema visual, consistencia entre público/cliente/admin | Design tokens, componentes base | Introducir estilos inconsistentes por sección |
| `qa-testing-agent` | Tests, edge cases, regresión, lint/typecheck | Tests, reportes de fallas | Aprobar su propio trabajo como cierre de fase |
| `code-review-agent` | Revisar cambios de otros agentes | Nada (solo lectura) — reporta hallazgos | Implementar features nuevas |

## Qué agente toma cada tipo de tarea

- "Definí/ajustá el modelo de X" → `domain-architect`
- "Diseñá la tabla/migración/RLS de X" → `database-agent`
- "Revisá si esto tiene un IDOR / fuga cross-tenant" → `auth-security-agent`
- "Implementá la reserva/cancelación/cupos de X" → `booking-engine-agent`
- "Generá las ocurrencias de este horario recurrente" → `scheduling-agent`
- "¿Puede este cliente reservar esto?" / pagos → `payments-entitlements-agent`
- "Exponé este endpoint / server action" → `backend-api-agent`
- "Pantalla de agenda/reservas/clientes del admin" → `frontend-admin-agent`
- "Flujo público de reserva / portal del cliente" → `frontend-customer-agent`
- "¿Cómo se ve/consistencia visual?" → `ui-ux-agent`
- "Escribí/corré tests de esto" → `qa-testing-agent`
- "Revisá este cambio antes de mergear" → `code-review-agent`
- Cualquier cosa que toque schema/auth/contratos/estructura de carpetas →
  el Orchestrator decide, con propuesta de `/propose-decision` si viene de
  un subagente.

## Workflow de coordinación

1. **Orchestrator inspecciona el repo** (estado actual, qué falta).
2. **Domain Architect propone dominio** (entidades, invariantes).
3. **Database Agent propone schema** (tablas, constraints, estrategia de
   concurrencia).
4. **Auth & Security Agent revisa multi-tenancy/RLS** sobre esa propuesta.
5. **Booking Engine + Scheduling Agents revisan** slots, capacidad y
   concurrencia sobre el schema propuesto.
6. **Orchestrator consolida** en `docs/architecture.md` y registra decisiones
   en `docs/decisions.md`.
7. **Recién ahí arranca la implementación**, fase por fase según
   `docs/roadmap.md`.

## Regla de completitud de fase

No se abren cinco funcionalidades a la vez. Cada fase de `docs/roadmap.md`
debe quedar funcional, testeada, tipada y documentada antes de pasar a la
siguiente. Cierre de fase = `qa-testing-agent` corre tests →
`code-review-agent` revisa → Orchestrator aprueba y lo asienta en
`docs/decisions.md`. Usar `/phase-review` para este checklist.

## Propuestas de cambios estructurales

Si un subagente detecta que necesita un cambio central (schema, dominio,
auth, contratos, estructura de carpetas, estrategia de slots/pagos/
recurrencia), no lo aplica directamente: genera una propuesta con
`/propose-decision` que incluya problema, cambio, impacto, migración y
compatibilidad. El Orchestrator decide y lo registra en `docs/decisions.md`.
