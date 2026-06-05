# LMS Uakari — Roadmap de features SDD

Vista resumen del `feature_list.harness.json` que vive en
`~/CnD/proyectos/lms/feature_list.harness.json`. Este archivo es un espejo
de seguimiento para uso humano; la fuente de verdad es el JSON del harness.

**Proyecto:** `uakari` (LMS, monorepo Turborepo Next.js 15)
**Refactor en curso:** mover tests y recursos de submódulo → módulo, redefinir
progreso como visitas y aprobación como métrica separada.

**Estados válidos:** `pending` · `spec_ready` · `in_progress` · `done` · `blocked`

---

## Mapa de dependencias

```
#1 schema/seeder
   └─> #2 endpoints services
         └─> #3 api-client
               ├─> #4 admin UI
               └─> #5 web UI + tracking
                     └─> #6 schema delta + recálculo backend
                           └─> #7 gate + dos métricas UI
#8 cleanup (independiente, después del resto)
```

`#4` y `#5` son paralelizables entre sí. Todas llevan `sdd: true`.

---

## Features

| # | Nombre | Estado | sdd | Descripción breve |
|---|--------|--------|-----|-------------------|
| 1 | `move_tests_and_resources_to_module` | `pending` | ✅ | Schema Prisma + seeder: evaluations a nivel módulo (`module_id @unique`), reemplaza `submodules_resources` por `modules_resources`. Migración limpia, sin data real que preservar. |
| 2 | `module_endpoints_refactor` | `pending` | ✅ | Reescribe endpoints `apps/services/src/app/v1/module/*`, `resource/search/[id]`, `professor/evaluations/create`, `student/courses/[course_id]` para operar sobre el nuevo modelo. Elimina archivos obsoletos (`submodule/addResource`, `students/progress`, `module/removeEvaluation`, `module/removeVideo`). |
| 3 | `api_client_refactor` | `pending` | ✅ | Actualiza `@uakari/api-client`: nuevo tipo `CourseResponse`, `addResourceByID`/`deleteResourceByID` enviando `module_id` snake_case, elimina funciones muertas (`deleteEvaluationFromModule`, `deleteVideoByID`, wrapper a `/v1/students/progress`). |
| 4 | `admin_ui_module_resources_and_tests` | `pending` | ✅ | Reemplaza `WIPBanner` de `FormEditResources` por vista funcional módulos↔recursos↔test. Form de creación de evaluation enviando `module_id` real. Input `approval_threshold` en `CourseSettings`. Limpia link muerto `/evaluaciones/asignacion` y componentes huérfanos. |
| 5 | `web_ui_module_view_and_visit_tracking` | `pending` | ✅ | Vista del estudiante: contenidos a nivel módulo (`ModuleList.tsx`), `SubmoduleView` pierde botón de test y tab descargables. Side-effect de visita: dispara `updateSubmoduleProgress(...true)` al montar. Elimina rutas duplicadas `[submodule_id]/evaluacion/*`. |
| 6 | `course_approval_schema_and_backend` | `pending` | ✅ | Schema delta: `courses.approval_threshold Float @default(0.6)` + `student_courses.approved Boolean @default(false)`. Recálculo de `approved` al persistir `evaluation_result` (función reutilizable invocada dentro del `$transaction`). |
| 7 | `web_ui_module_test_gate_and_dual_progress` | `pending` | ✅ | Util compartido `packages/common/src/progress.ts` (elimina 3 cálculos duplicados). Gate del botón "Test del módulo": oculto / deshabilitado / habilitado. UI estudiante muestra dos métricas: barra de progreso (visitas) y badge "Curso aprobado". |
| 8 | `cleanup_preexisting_bugs` | `pending` | ✅ | Fix del cálculo erróneo en `mis-cursos/page.tsx` (dividía entre módulos en vez de submódulos). Agrega `checkRolWithAuth` a endpoints `/v1/module/*`, `/v1/submodule/*`, `/v1/resource/*`, `/v1/professor/*` que estaban sin auth. |

---

## Decisiones de diseño que fijan los acceptance

- **Gate del test del módulo**: por módulo (cada módulo independiente).
- **"Submódulo visitado"** = se abrió la vista del submódulo (side-effect en mount).
- **Progreso del curso** = % de submódulos visitados. No depende del test.
- **Curso aprobado** = persistido en `student_courses.approved`. Recalculado server-side cuando se crea un `evaluation_result`.
- **Umbral de aprobación** = configurable por curso (`courses.approval_threshold`, default 0.6). Admin lo edita en CourseSettings.
- **Módulo sin test** → cuenta como aprobado, botón oculto en UI estudiante.
- **Endpoints obsoletos** → se eliminan del filesystem (no se dejan como 410).
- **No hay migración de datos real** (el LMS solo tiene dummies del seeder).

---

## Bugs preexistentes detectados durante el spec (van todos en #8)

1. `apps/web/src/app/mis-cursos/page.tsx:128-140` calcula progreso dividiendo
   submódulos completados entre número de **módulos** (debería ser submódulos).
2. Endpoints de `/v1/module/*`, `/v1/submodule/*`, `/v1/resource/*`,
   `/v1/professor/evaluations/*` **no validan auth**: cualquiera con red puede
   crear cursos, módulos, evaluaciones, etc.
3. `apps/admin/app/cursos/components/evaluationsTable.tsx:31` redirige a
   `/evaluaciones/asignacion`, ruta que no existe (404). Se limpia en #4.
4. `addResourceByID` en api-client envía `moduleID` (camelCase) pero el handler
   lee `submoduleID` → siempre devuelve 500. Se corrige en #2 + #3.
5. Endpoint `/v1/students/progress` es duplicado funcional de
   `/v1/students/submodule-progress`. Se elimina en #2.
6. El `CourseResponse` devuelve `answers.is_correct` a estudiantes (filtraje de
   respuestas correctas). Anotado para revisar en #2 acceptance pero el fix
   completo escapa al refactor.

---

## Flujo del harness por feature

Toda feature con `sdd: true` recorre el pipeline definido en
`~/CnD/proyectos/lms/AGENTS.harness.md`:

```
pending
  → [spec_partner]   conversación → project-spec.harness.md
  → [gherkin_author] features.harness/<name>.feature   (status: spec_ready)
  → ⏸ HUMANO APRUEBA los escenarios
  → in_progress
  → [tdd_craftsman]  Rojo → Verde → Refactor (un test a la vez)
  → [judge]          review (el juego entero)
  → [mutation_tester] mata mutantes; valida que los tests muerden
  → done
```

El `craftsman_lead` del repo del LMS es quien coordina; este archivo solo
muestra el estado.
