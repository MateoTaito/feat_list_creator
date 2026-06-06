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
 --[refactor mayor 1-8, ya done]--

#9 deseados hero (independiente)
   └─> #10 deseados filtros (depende de que #9 exista, pero no bloqueante estricta)
#11 certificados redesign (independiente)
#12 login logo fix (independiente)
#13 evaluacion view redesign (independiente; conceptualmente después de #5 rutas eliminadas)
```

Todas llevan `sdd: true`.

---

## Features

| # | Nombre | Estado | sdd | Descripción breve |
|---|--------|--------|-----|-------------------|
| 1 | `move_tests_and_resources_to_module` | `done` | ✅ | Schema Prisma + seeder: evaluations a nivel módulo (`module_id @unique`), reemplaza `submodules_resources` por `modules_resources`. Migración limpia, sin data real que preservar. |
| 2 | `module_endpoints_refactor` | `done` | ✅ | Reescribe endpoints `apps/services/src/app/v1/module/*`, `resource/search/[id]`, `professor/evaluations/create`, `student/courses/[course_id]` para operar sobre el nuevo modelo. Elimina archivos obsoletos (`submodule/addResource`, `students/progress`, `module/removeEvaluation`, `module/removeVideo`). |
| 3 | `api_client_refactor` | `done` | ✅ | Actualiza `@uakari/api-client`: nuevo tipo `CourseResponse`, `addResourceByID`/`deleteResourceByID` enviando `module_id` snake_case, elimina funciones muertas (`deleteEvaluationFromModule`, `deleteVideoByID`, wrapper a `/v1/students/progress`). |
| 4 | `admin_ui_module_resources_and_tests` | `done` | ✅ | Reemplaza `WIPBanner` de `FormEditResources` por vista funcional módulos↔recursos↔test. Form de creación de evaluation enviando `module_id` real. Input `approval_threshold` en `CourseSettings`. Limpia link muerto `/evaluaciones/asignacion` y componentes huérfanos. |
| 5 | `web_ui_module_view_and_visit_tracking` | `done` | ✅ | Vista del estudiante: contenidos a nivel módulo (`ModuleList.tsx`), `SubmoduleView` pierde botón de test y tab descargables. Side-effect de visita: dispara `updateSubmoduleProgress(...true)` al montar. Elimina rutas duplicadas `[submodule_id]/evaluacion/*`. |
| 6 | `course_approval_schema_and_backend` | `done` | ✅ | Schema delta: `courses.approval_threshold Float @default(0.6)` + `student_courses.approved Boolean @default(false)`. Recálculo de `approved` al persistir `evaluation_result` (función reutilizable invocada dentro del `$transaction`). |
| 7 | `web_ui_module_test_gate_and_dual_progress` | `done` | ✅ | Util compartido `packages/common/src/progress.ts` (elimina 3 cálculos duplicados). Gate del botón "Test del módulo": oculto / deshabilitado / habilitado. UI estudiante muestra dos métricas: barra de progreso (visitas) y badge "Curso aprobado". |
| 8 | `cleanup_preexisting_bugs` | `done` | ✅ | Fix del cálculo erróneo en `mis-cursos/page.tsx` (dividía entre módulos en vez de submódulos). Agrega `checkRolWithAuth` a endpoints `/v1/module/*`, `/v1/submodule/*`, `/v1/resource/*`, `/v1/professor/*` que estaban sin auth. |
| 9 | `deseados_hero_banner_and_background` | `pending` | ✅ | Reemplaza fondo roto (`fondoCursos.webp` no existe) por `fondo-cursos.webp` (el de /cursos) y añade hero/header banner con título "Mi lista" + imagen decorativa al estilo del sitio. |
| 10 | `deseados_filters_chips_search_price` | `pending` | ✅ | Extrae lógica de filtros a hook `useDeseadosFilters`, rediseña sidebar con chips por categoría/profesor, añade input de búsqueda y rango de precio CLP, drawer en mobile. |
| **11** | `certificados_redesign` | `pending` | ✅ | Rediseño completo de /mis-cursos/certificados: hero/header con FuturaExtraBlack, miniatura del certificado en cada card, badges por tipo (oro/plata/lms-blue/gris), botón de descarga con color lms-blue, estados vacío/error con la paleta del sitio. PDF generation intacta. |
| **12** | `login_lms_logo_sizing_fix` | `pending` | ✅ | Corrige logo sobredimensionado en signin (logoPlacement: 'outside' + h-20 via elements Clerk) y sustituye las 4 referencias rotas a logo_hero.webp por logo_lms_1.webp en signup, resultado de evaluación y TransactionHandler. |
| **13** | `evaluacion_view_redesign` | `pending` | ✅ | Rediseño completo de la vista de tests: background-wrap+hero-bg en vez de evaluation-image (clase CSS inexistente), TestIntro limpio, cards por pregunta con respuestas clickeables, badge aprobado/reprobado en resultado, breadcrumb, tipografías y paleta del sitio. |

---

## Decisiones de diseño que fijan los acceptance

### Features 1-8

- **Gate del test del módulo**: por módulo (cada módulo independiente).
- **"Submódulo visitado"** = se abrió la vista del submódulo (side-effect en mount).
- **Progreso del curso** = % de submódulos visitados. No depende del test.
- **Curso aprobado** = persistido en `student_courses.approved`. Recalculado server-side cuando se crea un `evaluation_result`.
- **Umbral de aprobación** = configurable por curso (`courses.approval_threshold`, default 0.6). Admin lo edita en CourseSettings.
- **Módulo sin test** → cuenta como aprobado, botón oculto en UI estudiante.
- **Endpoints obsoletos** → se eliminan del filesystem (no se dejan como 410).
- **No hay migración de datos real** (el LMS solo tiene dummies del seeder).

### Feature 9 — /deseados hero y fondo

- El fondo `fondoCursos.webp` no existe en `apps/web/public/images/` y nunca fue creado. Se reemplaza por `fondo-cursos.webp` (mismo que usa `/cursos`).
- Hero decorativo añadido arriba del contenido usando las tipografías y layout del sitio (`FuturaExtraBlack` para título, `josefinSans` para textos, `background-wrap` + `hero-bg`).
- El bug del 404 (`not-found.tsx` también referencia `fondoCursos.webp`) se queda fuera de esta feature.
- Responsive: hero se muestra en una línea en desktop, apilado en mobile.

### Feature 10 — /deseados filtros rediseño

- Lógica de filtrado extraída a `apps/web/src/app/deseados/hooks/useDeseadosFilters.ts`.
- UI renderizada por `apps/web/src/app/deseados/components/Filters/FiltroDeseados.tsx`.
- Chips redondeados (rounded-full) para categorías y profesores, con color activo uakari-orange/yellow-500.
- Búsqueda libre por nombre de curso en vivo (sin debounce).
- Filtro por rango de precio CLP (min / max, string vacío = sin límite).
- Los tres filtros se combinan con AND.
- Mobile: panel drawer colapsable reemplaza `<select>` nativo.
- Estado vacío nuevo cuando filteredCourses === [] pero el usuario sí tiene deseados.
- `/cursos` NO se toca en esta feature.

### Feature 11 — /mis-cursos/certificados redesign

- Hero/header en la parte superior con título "Mis Certificados" en FuturaExtraBlack + icono decorativo (FaCertificate o similar).
- Cada card tiene: arriba miniatura de `certificadoPlantilla.webp` (clickable → descarga PDF), abajo bloque con nombre del curso (FuturaHeavy), profesor (josefinSans), badge de tipo con color (oro=1.0, plata>=0.8, lms-blue>=0.6, gris<0.6), promedio, y aviso de evaluaciones incompletas.
- El botón "Descargar Certificado" cambia a `bg-lms-blue-600 hover:bg-lms-blue-700`.
- El componente `CertificateDownload.tsx` (html2canvas + jsPDF) **no se toca** salvo el color del botón — la plantilla con posiciones absolutas sobre el fondo `certificadoPlantilla.webp` se mantiene intacta.
- Estados vacío y error se rediseñan con la paleta y tipografía del sitio.
- Sin dependencias duras.

### Feature 12 — login logo sizing fix + logo_hero rotos

- **CustomSignin.tsx** (signin + modal Navbar): añadir `logoPlacement: 'outside'` y `elements.logoImage: 'h-20 w-auto object-contain'` en la appearance de Clerk. Logo queda fuera de la card, 80px max-height, centrado.
- **CustomSignup.tsx**: mismo fix + reemplazar `logoImageUrl: '/images/logo_hero.webp'` (no existe) por `logoImageUrl: '/images/logo_lms_1.webp'`.
- **evaluacion/resultado/** (2 archivos): reemplazar `src="/images/logo_hero.webp"` por `src="/images/logo_lms_1.webp"` (mantener dimensiones/clases existentes).
- **TransactionHandler.tsx** (pago finalizado): reemplazar `src="/images/logo_hero.png"` por `src="/images/logo_lms_1.webp"` (mantener dimensiones/clases).
- **Navbar.css** conserva su regla `.cl-logoBox img { width: 150px }` para el contexto del Navbar donde aplique.
- Sin dependencias duras.

### Feature 13 — evaluacion view redesign

- **evaluation-image** es una clase CSS referenciada en 4 archivos pero **no definida en ningún .css** del proyecto → fondo roto/blanco. Se reemplaza por `background-wrap` + `hero-bg` con `fondo_misCursos_1.webp` (mismo que /mis-cursos).
- **TestIntro.tsx**: nuevo componente extraído de `main.tsx` con pantalla limpia de inicio del test: hero/header con título en FuturaExtraBlack, card con info del test, botón "Empezar" en uakari-orange.
- **Lógica de intentos**: se mantiene en `main.tsx` con cards independientes para "ya rendiste" y "sin intentos", con badge de intentos restantes (1/2, 0/2).
- **Cards por pregunta**: cada pregunta en `bg-white rounded-lg shadow-md p-4 mb-4`, checkbox reemplazado por botón clickeable con estado visual, letra prefijo en chip gris.
- **Badge Aprobado/Reprobado**: badge redondeado en resultado: verde `bg-green-500/90` si score >= 0.6, rojo `bg-red-500/90` si score < 0.6.
- **Logo resultado**: se cambia de 300×300 a 120×120 (src `logo_lms_1.webp` por #12).
- **Breadcrumb**: 'Mis cursos > {course} > {module} > Resultado'.
- Sin dependencias duras sobre otras features (conceptualmente depende de #5 ya aplicada).

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

## Bugs preexistentes detectados en esta sesión (van todos en #12 y #13)

7. `apps/web/public/images/logo_hero.webp` **no existe**. Está referenciado en 4
   sitios del flujo de usuario: `CustomSignup.tsx:16`,
   `mis-cursos/*/evaluacion/resultado/page.tsx:137,147` (x2 archivos), y
   `finalizada/TransactionHandler.tsx:99`. En todos se ve la imagen rota o un
   espacio vacío. Se corrige en #12 (reemplazo por `logo_lms_1.webp`).

8. La clase CSS `evaluation-image` se referencia en 4 archivos de la vista de
   tests (`evaluacion/page.tsx`, `evaluacion/resultado/page.tsx` y sus
   equivalentes en `[submodule_id]/`) pero **no está definida en ningún .css**
   del proyecto. La vista de tests se renderiza sin fondo decorativo. Se corrige
   en #13 (reemplazo por `background-wrap` + `hero-bg` con
   `fondo_misCursos_1.webp`).

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
