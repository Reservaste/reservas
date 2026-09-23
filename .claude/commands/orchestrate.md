---
description: Repartir una tarea/feature entre los subagentes especializados del proyecto, respetando dependencias y contratos centrales.
---

Actuá como Orchestrator/Tech Lead de este proyecto (ver rol completo en
`CLAUDE.md`) para la siguiente tarea: $ARGUMENTS

Seguí este proceso, sin saltarte pasos:

1. **Leé el estado compartido**: `docs/architecture.md`, `docs/domain.md`,
   `docs/decisions.md`, `docs/roadmap.md`, y `docs/agent-responsibilities.md`.
   No asumas nada que no esté ahí o que no puedas verificar en el repo.

2. **Determiná el alcance real**: ¿qué fase de `docs/roadmap.md` toca esto?
   ¿toca solo una especialidad o varias? ¿requiere una decisión
   estructural nueva (schema, dominio, auth, contratos de API, estructura
   de carpetas, estrategia de slots/pagos/recurrencia)? Si sí, resolvé esa
   decisión primero (proponela vos como Orchestrator o pedile una
   propuesta al agente relevante) y registrala en `docs/decisions.md`
   antes de repartir implementación.

3. **Armá el plan de subagentes**, respetando dependencias:
   - Modelo, schema, migraciones, RLS, motor de reservas/horarios, pagos/permisos y server actions →
     un solo `backend-engineer` de punta a punta (no lo partas por capa, ver CHARTER).
   - Seguridad transversal (auth, roles, RLS, IDOR, multi-tenant, pagos) → `security-engineer`,
     revisa sobre lo que ya propuso `backend-engineer`.
   - UI → `ux-ui-designer` (si falta un componente base) y luego `frontend-engineer` (calendario
     público, portal del cliente y panel admin).
   - Cierre → `qa-engineer` corre tests, `reviewer` revisa, `release-manager` commitea (recordá que
     `backend/` y `frontend/` son repos separados: ver orden de coordinación en `orchestrator.md`
     cuando cambia `@reservaste/domain`).

4. **Lanzá los subagentes** con la herramienta `Agent`, uno por tarea
   acotada, dándole a cada uno el contexto puntual que necesita (no le
   pegues todo el pedido del usuario sin filtrar). Los que no dependen
   entre sí, lanzalos en paralelo; los que sí dependen, en secuencia.

5. **Revisá cada entrega** antes de pasar al siguiente paso: ¿respeta
   `docs/domain.md`? ¿usa terminología genérica (nunca Gym/Member/
   Trainer/Class)? ¿mantiene multi-tenancy? ¿duplica lógica que ya existía
   en otro agente?

6. **Cerrá con QA + Review**: no des la tarea por terminada sin que
   `qa-engineer` haya corrido los casos relevantes de
   `docs/testing.md` y `reviewer` haya revisado el diff.

7. **Actualizá la documentación compartida** (`docs/architecture.md`,
   `docs/decisions.md`, y el doc específico del área tocada) para que la
   próxima tarea no tenga que redescubrir esto.

8. **Reportá al usuario**: qué se hizo, qué agentes intervinieron, qué
   quedó pendiente o abierto como decisión, y qué sigue según
   `docs/roadmap.md`.
