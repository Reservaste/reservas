---
description: Generar una propuesta de cambio estructural (ADR) para que el Orchestrator la evalúe, en vez de aplicar el cambio directamente.
---

Un subagente (o vos mismo como Orchestrator) necesita un cambio
estructural: $ARGUMENTS

Esto aplica cuando el cambio toca: modelo de dominio, base de datos,
autenticación, multi-tenancy, contratos de API, autorización, estructura
de carpetas, estrategia de slots, estrategia de pagos, o estrategia de
recurrencia. Estos cambios **nunca se aplican directamente** — se
proponen.

Generá una entrada nueva en `docs/decisions.md`, siguiendo el formato ya
usado ahí (`ADR-000X`), con:

- **Problema**: qué limitación o necesidad concreta motiva el cambio (no
  una preferencia estética — un problema real que bloquea algo).
- **Cambio**: qué se propone exactamente.
- **Impacto**: qué otras áreas/agentes se ven afectados (dominio, schema,
  seguridad, API, UI, tests existentes).
- **Migración / compatibilidad**: si hay datos o código existente, cómo se
  migra sin romper lo ya construido.
- **Estado**: `Propuesta`.

No marques la entrada como `Aceptada` vos mismo si sos un subagente — eso
lo hace el Orchestrator después de evaluar el impacto cruzado con los
demás agentes relevantes (típicamente `backend-engineer` y/o
`security-engineer` según el tipo de cambio).

Si sos el Orchestrator evaluando una propuesta ya escrita: verificá que no
contradiga una decisión previa sin reemplazarla explícitamente
(`Reemplazada por ADR-000Y`), consultá a los agentes cuyo dominio se ve
afectado si hace falta, y luego marcá el estado como `Aceptada` o
`Rechazada` con la razón.
