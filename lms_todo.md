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
| 9 | `deseados_hero_banner_and_background` | `done` | ✅ | Reemplaza fondo roto (`fondoCursos.webp` no existe) por `fondo-cursos.webp` (el de /cursos) y añade hero/header banner con título "Mi lista" + imagen decorativa al estilo del sitio. |
| 10 | `deseados_filters_chips_search_price` | `done` | ✅ | Extrae lógica de filtros a hook `useDeseadosFilters`, rediseña sidebar con chips por categoría/profesor, añade input de búsqueda y rango de precio CLP, drawer en mobile. |
| **11** | `certificados_redesign` | `pending` | ✅ | Rediseño completo de /mis-cursos/certificados: hero/header con FuturaExtraBlack, miniatura del certificado en cada card, badges por tipo (oro/plata/lms-blue/gris), botón de descarga con color lms-blue, estados vacío/error con la paleta del sitio. PDF generation intacta. |
| **12** | `login_lms_logo_sizing_fix` | `pending` | ✅ | Corrige logo sobredimensionado en signin (logoPlacement: 'outside' + h-20 via elements Clerk) y sustituye las 4 referencias rotas a logo_hero.webp por logo_lms_1.webp en signup, resultado de evaluación y TransactionHandler. |
| **13** | `evaluacion_view_redesign` | `pending` | ✅ | Rediseño completo de la vista de tests: background-wrap+hero-bg en vez de evaluation-image (clase CSS inexistente), TestIntro limpio, cards por pregunta con respuestas clickeables, badge aprobado/reprobado en resultado, breadcrumb, tipografías y paleta del sitio. |

---

## Fase 2 — Refactor admin (pendiente de aprobación)

Continuación del refactor mayor. Las features 1-8 migraron schema + backend +
api-client, y los endpoints de admin. **Pero la app `apps/admin` quedó a
medio migrar y el flujo de subida de videos/recursos no funciona** en
algunos caminos. Este bloque cubre lo que falta para terminar el
refactor + endurecer S3 + cubrir las nuevas subidas.

### Estado real encontrado (auditoría del repo LMS)

- `apps/admin/app/cursos/[id]/page.tsx`, `FormEditCourse.tsx`,
  `Summary.tsx`, `formEditResources.tsx`, `utils.ts` siguen tipando el
  `Course` con `submodules: submodules[]` aunque ya no se usen para
  evaluaciones/recursos. Es residuo legacy.
- `apps/admin/app/cursos/components/FormEdit/Summary.tsx:48-55` itera
  `courseModule.modules.submodules` contando `sm.video_url` para
  "Recursos". Con el modelo nuevo ese conteo es 0/erróneo (los videos
  viven en `videos` por `courses`).
- `apps/admin/app/evaluaciones/preguntas/Menu.tsx:244-269` renderiza un
  **WIPBanner** ("Esta sección está siendo adaptada al nuevo sistema
  de submódulos") cuando no hay evaluation para el módulo. La feature
  #4 se marcó `done` pero dejó este banner.
- `apps/admin/app/evaluaciones/preguntas/Menu.tsx` aún tiene todo el
  estado de `submoduleID` y el `<ListSelect name="chooseSubmodule">`,
  pero ese dato ya no se usa (las evaluations son a nivel módulo).
- `resourceCreateModal.tsx:99-104` llama
  `createVideoModule(videoId, title, moduleID, token)` que pega contra
  `/v1/videos/modules` y ese endpoint **devuelve 410 deprecated**
  (ver `apps/services/src/app/v1/videos/modules/route.tsx:3-10`).
  La subida de video desde el botón "video" del resourceCreateModal
  está rota.
- `LazyVideoPlayer.tsx` (apps/admin) **no se importa en ningún
  archivo del repo** → componente huérfano.
- `apps/admin/app/evaluaciones/components/evaluationsTable.tsx:173-189`
  usa `router.push("/evaluaciones/preguntas")` (válido) y guarda
  `id_evaluation`+`id_module` en sessionStorage. Este flujo sí está
  bien, pero la página destino muestra el WIPBanner.
- **Auth bypass masivo en S3**: `apps/services/src/app/v1/s3/*` (10
  endpoints: `delete`, `get`, `upload`, `signed_url`,
  `images/delete`, `images/get`, `images/get_object`,
  `images/signed_url`, `images/update`, `images/upload`) **NO llaman
  `checkRolWithAuth`**. Cualquiera con la URL puede pedir URLs
  presigned para subir/bajar/borrar del bucket R2. La feature #8
  endureció `/v1/module/*`, `/v1/submodule/*`, `/v1/resource/*`,
  `/v1/professor/*` pero se olvidó de `/v1/s3/*`.
- `app/cursos/[id]/page.tsx:127` usa
  `bg-[url(/admin/images/fondo_curso_1.webp)]` y el layout tiene
  ese fondo embebido en la prop `className` (no en CSS). No es bug,
  pero la imagen `/admin/images/fondo_curso_1.webp` debe existir
  (verificar antes de tocar).

### Features planificadas (#14 en adelante)

| # | Nombre | sdd | Descripción breve |
|---|--------|-----|-------------------|
| **14** | `admin_query_drop_submodules` | ✅ | Quita `submodules` del SELECT/typing en `apps/admin/app/cursos/[id]/page.tsx`, `FormEditCourse.tsx`, `Summary.tsx`, `formEditResources.tsx`, `utils.ts`. La query de `prisma.courses.findUnique` deja de traer `submodules`; los tipos del `Course` ya no exponen `submodules`; el bug del contador "Recursos" en `Summary.tsx` se corrige contando solo `modules_resources` + 1 video de presentación si existe. |
| **15** | `admin_remove_submodule_legacy_ui` | ✅ | Elimina de `apps/admin/app/evaluaciones/preguntas/Menu.tsx` todo el submódulo: state `submoduleID`, `useEffect` de sessionStorage `id_submodule`, el `ListSelect name="chooseSubmodule"`, la prop `submodulesForSelectedCourse` y el WIPBanner ("adaptada al nuevo sistema de submódulos"). Deja el menú limpio Curso→Módulo→Evaluation. La prop `submodules` en `Props` de Menu desaparece. |
| **16** | `admin_module_video_endpoints` | ✅ | Crea el modelo real "1 video Cloudflare por módulo": migration Prisma `videos.module_id Int?` + FK opcional, relación inversa `modules.video`. Endpoints nuevos `POST /v1/videos/create` (cuerpo: `{cloudflare_id, title, module_id}`, crea fila en `videos` con duración calculada por `getVideoDuration`), `GET /v1/videos/module/[module_id]` (devuelve el video del módulo o null), `PUT /v1/videos/module/[module_id]/[video_id]` (reemplaza), `DELETE /v1/videos/module/[module_id]` (limpia el campo y borra fila en `videos` si no la usa otro módulo). Todos con `checkRolWithAuth(['admin','root','professor'])`. **Elimina del filesystem** `apps/services/src/app/v1/videos/modules/route.tsx` y `apps/services/src/app/v1/videos/edit/[module_id]/[video_id]/route.ts` (los dos 410 deprecated). |
| **17** | `api_client_module_video` | ✅ | Añade en `packages/api-client/src/videos.ts` las funciones `createModuleVideo(moduleId, cloudflareId, title, token)`, `getModuleVideo(moduleId, token)`, `replaceModuleVideo(moduleId, videoId, cloudflareId, title, token)`, `deleteModuleVideo(moduleId, token)`. Pegan a los nuevos endpoints de feature #16. Tipos `ModuleVideo` exportados. Elimina del api-client `createVideoModule` (el que apuntaba al endpoint 410) y `updateVideoData` con firma de `module_id` muerto. `npx turbo run build` sin errores. |
| **18** | `admin_module_video_upload_ui` | ✅ | Reescribe `apps/admin/app/cursos/components/FormEdit/formEditResources.tsx`: en el acordeón del módulo añade un bloque "Video del módulo" con su propio `UploadVideo` (con `pushToDatabase=false` para que solo suba a Cloudflare), previsualización con `Stream` o `LazyVideoPlayer`, y botones "Reemplazar"/"Eliminar" que invocan las funciones de feature #17. `resourceCreateModal.tsx` deja de ofrecer "video" como opción dentro de "Agregar recurso" (eso ya es tarea del nuevo bloque). **Borra** `LazyVideoPlayer.tsx` (sigue huérfano) o reemplázalo por el nuevo wrapper de previsualización inline. |
| **19** | `s3_endpoints_auth_hardening` | ✅ | Agrega `checkRolWithAuth(['admin','root','professor'])` a los 10 endpoints `/v1/s3/*` que están sin auth: `s3/delete`, `s3/get`, `s3/signed_url`, `s3/upload`, `s3/images/delete`, `s3/images/get`, `s3/images/get_object`, `s3/images/signed_url`, `s3/images/update`, `s3/images/upload`. Tests: curl sin token → 401, con student → 403, con admin/professor → 2xx. `npx turbo run build --filter=services` sin errores. |
| **20** | `admin_upload_resource_single_call` | ✅ | `resourceCreateModal.tsx` actualmente hace `uploadResource(...)` que ya crea el `modules_resources`, y **luego** llama `addResourceByID(...)` que intenta crear el mismo `modules_resources` y recibe 200/500 espurios. Refactor del cliente: `uploadResource` sigue creando fila + `modules_resources` en una sola llamada (no tocar el backend); `addResourceByID`/`deleteResourceByID` se mantienen para futuras re-asociaciones pero el modal ya no los llama en el flujo de "agregar recurso". El bug es de cliente: borrar la segunda llamada. |
| **21** | `admin_ui_logo_cleanup` | ✅ | Sustituye las 5 referencias a `/images/logo_hero.webp` (no existe) en `apps/admin` por `/images/logo_lms_1.webp` con `NEXT_PUBLIC_ADMIN_PREFIX` cuando aplique: `moduleCreateModal.tsx:143`, `moduleRemoveModal.tsx:22`, `resourceCreateModal.tsx:227`, `resourceRemoveModal.tsx:22`, `showModal.tsx:74`, `confirmSaveModal.tsx:23`. También en `auth/sign-in/[[...sign-in]]/page.tsx:9` y `auth/sign-up/[[...sign-up]]/page.tsx:9`. (La feature #12 cubre la web; esta cubre admin.) Mantener dimensiones/clases existentes. |
| **22** | `admin_evaluation_form_remove_submodule_select` | ✅ | `apps/admin/app/evaluaciones/nuevo/Form.tsx` y la query de `evaluaciones/nuevo/page.tsx` aún permiten elegir módulo; el submodule no aparece aquí, pero la `Props.coursesList[].course_modules[].modules` debería quitar cualquier referencia residual. Auditar: no hay `submodule` en este flujo, pero verificar que `modules.evaluation` (nullable) está disponible y se muestra "ya tiene evaluation" en el select. |
| **23** | `admin_summary_recursos_count_fix` | ✅ | `apps/admin/app/cursos/components/FormEdit/Summary.tsx:48-55` cuenta `submodules.video_url` (modelo viejo). Reescribir `countAllResources`/`countVideosByModule` para que la card de "Resumen" muestre: Lecciones = `course_modules.length`, Evaluaciones = `modules.filter(m => m.evaluation !== null).length`, Recursos = `modules_resources.length` + 1 si `course.video_id !== null` (video presentación). La función vive en `utils.ts` con tests unitarios del cálculo. |

### Dependencias entre features nuevas

```
#14  #15  #20  #21  #22  #23   (independientes, todas UI/admin-only)
   \  |  /          |   /
    \ | /           |  /
   #16 ──> #17 ──> #18       (cadena videos)
              #19            (independiente, endurece S3)
```

### Criterio de cierre del bloque

- `npx turbo run build` (monorepo completo) sin errores.
- `npx turbo run typecheck` (si existe) sin errores.
- `grep -r "submodules:" apps/admin/app` → 0 ocurrencias (excepto imports de tipos que ya no se usen → también 0).
- `grep -r "logo_hero" apps/admin/app` → 0 ocurrencias.
- `grep -r "/v1/videos/modules" apps/admin` → 0 ocurrencias.
- `curl` manual a `/v1/s3/upload` sin token → 401; con student → 403; con admin → 201.
- Flujo manual en admin: crear curso → agregar módulo → subir PDF como recurso (queda en `modules_resources`) → subir video de módulo (queda en `videos` con `module_id`) → asignar evaluation al módulo (queda en `evaluations.module_id`) → eliminar todo y verificar que no quedan filas huérfanas.

### Riesgos y notas

- **Submodules no se eliminan de la web del estudiante** (siguen
  mostrando `submodule.video_url` YouTube en `SubmoduleView.tsx`).
  Eso queda fuera de este bloque (sería otro refactor mayor). Aquí
  solo limpiamos admin.
- **El `seeder` aún crea submodules** (`packages/database/src/seed.ts:108`).
  Si las features 14-23 eliminan `submodules` del typing de admin,
  el seeder sigue siendo válido para web. No tocar.
- **`feature #4 marked done pero dejó WIPBanner + submodules en typing
  de admin** — esto es un bug de aceptación. Las features 14, 15
  cierran ese gap.

---

## Fase 3 — Adaptar crear curso al modelo módulo-céntrico

Continuación del refactor mayor. Las features 14-23 cerraron el lado
admin (edición de curso, gestión de preguntas, video por módulo,
endurecimiento S3). **Pero la página de crear curso
(`apps/admin/app/cursos/nuevo/`) quedó sin adaptar**: el form sigue
con el patrón monolítico viejo, no expone `approval_threshold` que la
edición sí tiene en `CourseSettings.tsx`, y el endpoint
`POST /v1/professor/course` no tiene `checkRolWithAuth` (mismo bypass
que la #19 cerró para `/v1/s3/*`).

### Estado real encontrado (auditoría del repo LMS)

- `apps/admin/app/cursos/nuevo/Form.tsx` es un único
  `useState<FormState>` con 9 campos y un único `handleSubmit`. No
  expone `approval_threshold`.
- `apps/services/src/app/v1/professor/course/route.ts` (POST) **no
  llama `checkRolWithAuth`**. Es el endpoint más crítico de admin: en
  una sola transacción crea fila en `videos`, sube imagen al bucket
  R2 con `PutObjectCommand`, consume `getHandle` de Shopify y crea
  fila en `courses`.
- `apps/services/src/app/v1/professor/course/schema.ts` (Joi) no
  acepta `approval_threshold`. El form tampoco lo manda.
- `apps/services/src/app/v1/course/edit/[id]/route.ts` (PUT)
  **también no tiene `checkRolWithAuth` y no extrae
  `approval_threshold` del FormData** (aunque `FormEditCourse.tsx`
  sí lo appendea). Es un bug aparte, fuera de este bloque.
- El form no usa el patrón seccionado (`bg-white/70 rounded-xl`) que
  la edición ya adoptó. Es residuo pre-refactor.

### Features planificadas (#24 en adelante)

| # | Nombre | sdd | Descripción breve |
|---|--------|-----|-------------------|
| **24** | `admin_create_course_auth_hardening` | ✅ | Agrega `checkRolWithAuth(['admin','root','professor'])` a `apps/services/src/app/v1/professor/course/route.ts` (POST). Replica el patrón de `/v1/professor/evaluations/create` (auth + `userId` para 401/403). Mueve `req.formData()` para que sea posterior al check. `npx turbo run build --filter=services` sin errores. Batería de curl: sin token → 401, student → 403, admin/professor/root → 200. |
| **25** | `admin_create_course_approval_threshold_field` | ✅ | Suma `approval_threshold` al form de crear curso (`apps/admin/app/cursos/nuevo/Form.tsx`): input `type="number" min=0 max=1 step=0.01`, valor inicial `0.6` (alineado con `@default(0.6)` del schema Prisma). Actualiza `FormState`, `handleChange` (con validación NaN/rango), `handleSubmit` (append al FormData). Backend: `schema.ts` añade `Joi.number().min(0).max(1).optional()` con fallback a `undefined` si vacío; `route.ts` persiste `course.approval_threshold ?? 0.6` en el `prisma.courses.create`. Cierra la fricción de tener que ir a `/cursos/[id]` después de crear para setear el umbral. |
| **26** | `admin_edit_course_endpoint_hardening` | ✅ | Cierra dos bugs preexistentes de `PUT /v1/course/edit/[id]/route.ts`: (1) agrega `checkRolWithAuth(['admin','root','professor'])` al inicio del handler (mismo patrón que #24); (2) extrae `approval_threshold` del FormData (que `FormEditCourse.tsx:154-168` sí manda) y lo incluye en el `prisma.courses.update` con validación de rango [0,1]. Antes del fix, editar el umbral en CourseSettings y "Actualizar Información" no persistía el cambio silenciosamente. |
| **27** | `admin_create_course_form_redesign` | ✅ | Refactor del `apps/admin/app/cursos/nuevo/Form.tsx` monolítico al patrón seccionado que ya usa la edición (`bg-white/70 rounded-xl p-6 mb-6`). Crea 3 sub-componentes: `CourseBasicInfo.tsx` (name, shopify_id, price, profesor, dificultad, categoría, image, descripción), `PresentationVideo.tsx` (UploadVideo sin preview), `CourseSettings.tsx` (campo approval_threshold de #25). El `Form.tsx` queda como orquestador. Depende de #25. |
| **28** | `admin_create_course_validation_hardening` | ✅ | Endurece el `handleSubmit` con validación por campo: trim+non-empty en name/shopify_id/description, max length (255/8192), price > 0, image tipo `image/*` y <5MB, mensajes específicos con `swal` por cada fallo. Cambia `disabled={withVideo && !isVideoFinished}` por `disabled={isValidating || !isFormValid}`. Feedback visual: `border-red-500` en inputs inválidos + `<p className="text-red-500 text-sm">` con el mensaje. Depende de #27. |

### Dependencias entre features nuevas

```
#24  #26               (independientes, auth en create/edit)

#25 ──> #27 ──> #28    (cadena: threshold field → form redesign → validation)
```

#24 y #26 son foundational (seguridad). #25 puede ir en paralelo. #27
necesita #25 aplicada para que `CourseSettings.tsx` tenga el campo
approval_threshold que mostrar. #28 necesita #27 para que la
validación opere sobre la nueva estructura seccionada.

### Criterio de cierre del bloque

- `npx turbo run build --filter=admin --filter=services` sin errores.
- `npx turbo run build` (monorepo completo) sin errores.
- `curl` manual a `POST /v1/professor/course` con FormData válido:
  sin token → 401, con student → 403, con admin/professor/root → 200.
- `curl` manual a `PUT /v1/course/edit/[id]` con FormData válido:
  sin token → 401, con student → 403, con admin/professor/root → 200.
- `curl` con `approval_threshold=1.5` o `=-0.1` (POST o PUT) → 400
  con mensaje Joi.
- `curl` sin `approval_threshold` en el FormData (POST) → 200 con la
  fila persistida con `0.6`.
- Crear curso desde `/cursos/nuevo` con threshold `0.8` → fila en
  `courses` con `0.8` (verificar con
  `SELECT approval_threshold FROM courses WHERE id = X`).
- Editar curso desde `/cursos/[id]`, cambiar el umbral en
  CourseSettings de 0.6 a 0.8, "Actualizar Información" → la fila
  queda con 0.8.
- Validación client-side (#28): submit con name vacío → swal con
  error; submit con PDF como image → swal con error; submit con
  image de 6MB → swal con error; submit con price=0 → swal con
  error; submit válido → POST se hace.
- Inspección visual: `/cursos/nuevo` muestra 3 cards (BasicInfo,
  PresentationVideo, Settings) con styling
  `bg-white/70 rounded-xl p-6 mb-6`, mismo orden visual que la
  versión monolítica.

### Riesgos y notas

- **#27 (form redesign) es la más invasiva** de la fase: 3 archivos
  nuevos + reducción de Form.tsx. Conviene hacer #25 antes para que
  `CourseSettings.tsx` tenga contenido.
- **#28 podría ser `sdd: false`** si las validaciones fueran
  puramente triviales, pero como añade estado de errores, feedback
  visual y computed `isFormValid`, queda `sdd: true`.
- **#24, #25, #26, #27, #28 son todas `sdd: true`** porque tocan el
  endpoint crítico de admin y/o el form con múltiples criterios.

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
