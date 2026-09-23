---
description: Checklist de cierre de fase — QA corre tests, Code Review revisa, Orchestrator aprueba.
---

Vas a evaluar si se puede cerrar la fase actual de `docs/roadmap.md`
(o la que se indique acá: $ARGUMENTS).

Seguí este checklist en orden, sin saltarte pasos:

1. **Releé el alcance de la fase** en `docs/roadmap.md` y confirmá qué se
   suponía que quedara funcional.

2. **Lanzá `qa-engineer`** para correr, en cada repo que corresponda
   (`backend/` y/o `frontend/` — son paquetes separados, sin script raíz):
   `typecheck`, `test`/`test:integration` en `backend/`; `lint`, `typecheck`,
   `test`, `build` en `frontend/`. Incluí los casos obligatorios de
   `docs/testing.md` que apliquen a esta fase. Pedile un reporte caso por
   caso, no un simple verde/rojo global.

3. **Lanzá `reviewer`** sobre los cambios de esta fase. Pedile
   hallazgos concretos (archivo, línea, escenario de falla), priorizados
   por severidad, incluyendo si `docs/domain.md`/`docs/database.md`/
   `docs/api.md` quedaron sincronizados con lo que cambió.

4. **Si hay hallazgos o tests rojos**: no cierres la fase. Reasigná cada
   fix al agente dueño de esa área (no lo arregles vos mismo salteando al
   especialista, salvo que sea trivial y de tu propia coordinación) y
   volvé a correr este checklist después del fix.

5. **Verificá también**, como parte del cierre:
   - ¿La documentación relevante (`docs/domain.md`, `docs/database.md`,
     `docs/security.md`, `docs/api.md`, `docs/architecture.md`) quedó
     actualizada con lo que efectivamente se construyó?
   - ¿Quedó algo implementado a medias o algún workaround temporal sin
     registrar como deuda conocida?
   - ¿Se introdujo, aunque sea accidentalmente, algún concepto específico
     de gimnasio (`Gym`/`Member`/`Trainer`/`Class`) en el dominio o en
     componentes base?

6. **Solo si todo lo anterior está en verde**, marcá la fase como cerrada,
   dejá constancia en `docs/decisions.md` si hubo alguna decisión relevante
   tomada durante la fase, y anunciá al usuario que se puede pasar a la
   siguiente fase de `docs/roadmap.md`.
