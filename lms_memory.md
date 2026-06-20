# LMS Uakari — memory

Resumen del proyecto destino en `~/CnD/proyectos/lms/`. Sirve para no
tener que re-leer el repo completo cada vez que se redacta una feature
desde `feat_list_creator`.

## Arquitectura

Monorepo Turborepo + pnpm-workspace. Tres apps + seis packages:

```
apps/
  web/      → Next.js 15, app router. Vista del estudiante.
  admin/    → Next.js 15, app router. CRUD de cursos / módulos / recursos / evaluaciones.
  services/ → Next.js 15 app router usado como API backend (rutas en src/app/v1/).

packages/
  database/         → Prisma schema (Postgres), cliente, seeder, recalculateApproval.ts.
  api-client/       → Funciones tipadas que pegan contra /v1/* (consumido por web y admin).
  common/           → Util compartido (checkRolWithAuth, progress, etc.).
  ui/               → Componentes UI compartidos.
  eslint-config/    → Config ESLint compartida.
  typescript-config/→ Config TS compartida.
```

**Source of truth de features:** `feature_list.harness.json` en la
raíz del LMS. NO el `feature_list.json` de `feat_list_creator` (ese
es solo buffer / borrador).

## Stack

- Next.js 15 (app router), TypeScript, Tailwind.
- Clerk (auth) — siempre `auth()` desde `@clerk/nextjs/server` en
  services.
- Postgres + Prisma (`@uakari/database`).
- Cloudflare Stream (videos de módulo y de presentación) + R2/S3
  (recursos de módulo, imágenes de curso/profesor).
- Shopify (legacy — la web paga con Transbank, no con Shopify; `shopify_id`
  y `shopify_handle` en `courses` son residuo de una integración anterior
  y nadie los lee después del write).
- Transbank (pagos).
- Joi para validación de payloads en services.
- SWR para fetch en web/admin.

## Modelo de datos (estado post-refactor 1-8 + 14-23)

- `courses`: `video_id` (presentación, 1-1), `image_id` (1-1),
  `professor_id`, `categorie_id`, `difficulty`, `shopify_id`/`handle`,
  `approval_threshold Float? @default(0.6)`.
- `course_modules`: join `courses ↔ modules` con `@@id([course_id, module_id])`.
- `modules`: 1-N a `modules_resources.resources`, 1-1 nullable a
  `evaluations` (`evaluations.module_id` es `@unique`), 1-N a
  `videos` (vía relación `module_videos`), 1-N a `submodules` (legacy).
- `videos`: `module_id Int?` (nullable — null = video de presentación
  del curso, NOT null = video del módulo).
- `student_courses.approved Boolean` se recalcula server-side en
  `packages/database/src/recalculateApproval.ts` cada vez que se
  persiste un `evaluation_result`.
- `submodules` SIGUE en el schema (lo consume la web para YouTube
  embeds en `SubmoduleView.tsx`). El seeder los sigue creando.
  Endpoint `POST /v1/submodule/create` (autenticado, #8) y
  `SubmoduleCreateModal` (230 líneas, huérfano hasta #32) ya existen;
  #32 lo wirea en `formEditResources.tsx` con un botón "Agregar
  Submódulo" en la sección por módulo. NO eliminar en features
  nuevas — la web lo necesita.

## Auth

Patrón único en `services` (importar de `@uakari/common/util`):

```ts
import { auth } from "@clerk/nextjs/server";
import { checkRolWithAuth } from "@uakari/common/util";

if (!(await checkRolWithAuth(["admin", "root", "professor"], auth))) {
  const { userId } = await auth();
  return Response.json(
    { message: userId ? "Forbidden" : "Unauthorized" },
    { status: userId ? 403 : 401 }
  );
}
```

**Gaps conocidos (gaps sin cubrir al día de hoy):**

- `POST /v1/professor/course` → cubierto en #24 (pendiente).
- `PUT /v1/course/edit/[id]` → NO cubierto. Tampoco extrae
  `approval_threshold` del FormData que `FormEditCourse.tsx` sí manda.
  Sería #26 si se decide cubrir.
- 10 endpoints `/v1/s3/*` → cubierto en #19.

## UI admin (apps/admin)

```
apps/admin/app/
  cursos/
    page.tsx                       → tabla de cursos
    [id]/page.tsx                  → editar curso (FormEditCourse.tsx)
    nuevo/page.tsx + Form.tsx      → crear curso (monolítico, sin refactor)
    components/FormEdit/           → sub-componentes del editor:
      CourseBasicInfo, CourseSettings, ModuleContent, PresentationVideo,
      Summary, modales de module/resource/caption
  evaluaciones/                    → preguntas, asignaciones, resultados
  auth/                            → signin/signup con Clerk
  accesos/, admins/, categories/, perfil/, profesores/
```

El editor de curso (`FormEditCourse.tsx`) ya está refactorizado al
patrón seccionado (`bg-white/70 rounded-xl p-6 mb-6`). El form de
crear curso (`Form.tsx`) NO — es residuo pre-refactor.

## UI web (apps/web)

- `/cursos` — catálogo.
- `/mis-cursos` — cursos inscritos, certificados, progreso, tests.
- `/deseados` — wishlist con filtros.
- `SubmoduleView.tsx` — sigue mostrando `submodule.video_url` (YouTube
  embeds, legacy).

## Build / verificación

```bash
npx turbo run build                                # monorepo completo
npx turbo run build --filter=admin
npx turbo run build --filter=services
npx turbo run build --filter=web
npx turbo run generate --filter=@uakari/database   # regenerar Prisma Client
npm run migrate:dev                                 # aplicar migrations
npx turbo run test --filter=admin                   # si hay tests
docker compose up -d                                # postgres + r2 locales
```

## Workflow de features

`pending` → `spec_ready` (Spec Partner + Gherkin) → `in_progress`
(TDD craftsman: rojo → verde → refactor) → `done` (judge + mutation).
Modo `fast` para features triviales (≤ 3 acceptance). Modo `schema`
para features de schema + migración.

## Endpoints muertos / 410 (no pegar ahí)

- `POST /v1/videos/modules` (route.tsx) — 410 deprecated. Usar
  `POST /v1/videos/create` con `{cloudflare_id, title, module_id}`.
- `PUT /v1/videos/edit/[module_id]/[video_id]` — 410 deprecated. Usar
  `PUT /v1/videos/module/[module_id]/[video_id]`.

## Quirks / particularidades

- `description` en `courses` es `String` (sin max en el schema; Joi
  lo limita a 8192).
- `price` en `courses` es `Int` (CLP, sin decimales).
- `difficulty_enum`: `BASICO | INTERMEDIO | AVANZADO`.
- `categorie_id` (typo legacy: `categorie` no `category` en DB; en
  código ya se usa `categorie_id` consistentemente).
- `getCourseOverview` (`GET /v1/public/overview/course/[id]`) es
  público y devuelve `{tests, modules, resources, video_duration}`.
- El video de presentación del curso se guarda en `videos` con
  `module_id = NULL`, vinculado al course vía `courses.video_id`
  (1-1 unique, `onDelete: SetNull`).
- `evaluations.id_course` está en el schema pero la lógica actual
  usa `module_id` como source of truth (1 evaluation por módulo,
  no por curso).
- `submodules` está en el schema aunque la lógica de tests/recursos
  ya no lo usa (salvo la web para YouTube embeds). El seeder los
  sigue creando. NO eliminar.
- `cloudflare_id` y `title` del video de presentación se determinan
  en el cliente a partir del nombre del curso (no hay input de
  título propio en el form de crear/editar curso).
- `getHandle(shopify_id)` se llama solo si `shopify_id` no está vacío
  (soft deprecado en #29 — antes era required, ahora es opcional con
  nota "legacy" en el form). Si la API de Shopify falla, el handle
  queda `undefined` y el curso se crea igual.

## Estética visual (estado post-fase 4)

Dos paletas coexisten en `apps/admin/app/`; la fase 4 (#33-#39) cierra
esa divergencia.

### Paleta "nueva" (lms-blue) — la que se está adoptando

Adoptada por `apps/admin/app/cursos/` (CoursesTable + page.tsx) y
`apps/admin/app/components/Overview.tsx`. Es el target de #33-#39.

- Fondo de página: `style={{backgroundColor: '#F1F8FF'}}` o
  `bg-lms-blue-50` (mismo color, el segundo es alias del primero).
- Wrapper: `min-h-screen pt-{28|32}` + `max-w-7xl mx-auto` +
  `px-4/6 pb-8/10`.
- Header de página: `flex justify-between items-center mb-6` con
  `<h1 className="text-3xl font-semibold text-uakari-grey">` + botón
  primario `bg-lms-blue-500 hover:bg-lms-blue-600 text-white
  font-semibold px-6 py-2 rounded-lg shadow-md`.
- Tabla: `<thead>` con `bg-lms-blue-50 text-uakari-grey` y celdas
  `py-3 px-4 font-medium text-left text-sm`. `<tbody>` con filas
  `hover:bg-lms-blue-50/50 transition-colors` y separador
  `border-b border-gray-100`. Columna Acciones con
  `flex gap-2 justify-center items-center`.
- Botones de acción en filas: primario `bg-lms-blue-500` (Editar),
  peligro `bg-uakari-pink hover:bg-uakari-pink-600` (Eliminar).
- Input de búsqueda: `w-full px-4 py-2 bg-white border border-gray-200
  rounded-lg shadow-sm focus:outline-none focus:ring-2
  focus:ring-lms-blue-500`.
- Navbar (post-#38): `bg-white border-b border-lms-blue-100
  shadow-sm z-50 fixed w-full h-20` con logo y UserView propios.
  Páginas internas usan `pt-28` (antes era `pt-32` con la navbar
  oscura de 96px).
- Modales (post-#39): shell compartido
  `apps/admin/app/components/modal/AdminModal.tsx` con logo LMS,
  `<h3 className="text-lg font-semibold text-uakari-grey text-center">`,
  body con `AdminField` (input con `focus:ring-lms-blue-500
  focus:border-lms-blue-500`), footer con
  `AdminModalFooter` (Cancelar pink + Confirmar lms-blue).

### Paleta "vieja" (uakari-pale-white + naranja) — la que se va a migrar

Aún presente en `apps/admin/app/accesos/AccesosTable.tsx`,
`apps/admin/app/components/ProfessorsTable.tsx`,
`apps/admin/app/components/ProfessorsAction.tsx` (edit modal),
`apps/admin/app/components/CategoriesTable.tsx`,
`apps/admin/app/admins/components/AdminsTable.tsx`,
`apps/admin/app/admins/components/SelectModal.tsx`,
`apps/admin/app/admins/NewAdmin.tsx`,
`apps/admin/app/profesores/page.tsx` (create modal),
`apps/admin/app/components/Navbar/Navbar.tsx` (la navbar oscura
  `bg-[#393434] h-24`),
`apps/admin/app/page.tsx:9` (wrapper exterior landing).

Patrón viejo: `bg-uakari-pale-white min-h-screen` + `max-w-7xl
mx-auto` + tabla con `border-2 border-uakari-orange` + botones
`bg-uakari-orange` (primario) y `bg-uakari-pink` (peligro) +
tipografía centrada.

### Decisiones de diseño que fijan los acceptance (fase 4)

- **REGLA TRANSVERSAL — los tres registros son un espejo de /cursos**:
  las features #34 (accesos), #35 (profesores) y #36 (admins) deben
  implementarse como un calco visual de la vista actual de Cursos
  (`apps/admin/app/cursos/page.tsx` + `apps/admin/app/cursos/components/CoursesTable.tsx`).
  Esa página es la referencia canónica de la nueva estética y NO
  se toca en este bloque. Cualquier elemento de las tres páginas
  de registro (header, wrapper, fondo, tabla, input de búsqueda,
  botones, modales vía #39) que difiera del patrón establecido en
  /cursos debe justificarse explícitamente en el PR. El criterio
  de cierre del bloque en `lms_todo.md` incluye un diff visual
  side-by-side entre /cursos y cada uno de los tres registros. La
  descripción de cada una de las tres features y su último
  acceptance criterion repiten esta regla de forma explícita.

- **#33 (test container 3/4)**: `max-w-[75%]` con `mx-auto` en el
  `<div className="col-span-4">` interior. El wrapper exterior pierde
  el `mx-10`. Se mantiene `top-[100px]` y `bg-white p-5` para no
  romper el fondo decorativo de #13.
- **#34 (accesos)**: el botón "Acción" sin `onClick` se reemplaza por
  dos botones reales ("Editar" + "Ver"). El modal de creación se llama
  `InviteUserModal` y reusa `createAdmin` (un endpoint dedicado de
  invitación queda para otra fase; documentar en el código).
- **#35 (profesores)**: `ProfessorsAction.tsx` se mueve de
  `apps/admin/app/components/` a `apps/admin/app/profesores/`. La
  copia vieja se marca `@deprecated` con JSDoc, no se elimina (rompe
  el test de #30).
- **#36 (admins)**: `SelectModal.tsx` se renombra a
  `RolesCheckboxes.tsx` y deja de ser modal completo; el modal
  completo lo provee `AdminEditModal.tsx` (que envuelve el shell de
  #39 + el `RolesCheckboxes`).
- **#37 (landing)**: hero en `font-futuraExtraBlack text-4xl
  text-uakari-grey` con subtítulo en `font-jakarta`. Cards de
  Overview con `p-8` (antes `p-6`) e íconos `text-3xl` (antes
  `text-2xl`).
- **#38 (navbar)**: `bg-white border-b border-lms-blue-100 shadow-sm
  h-20` reemplaza `bg-[#393434] h-24`. El dropdown de Clerk de
  UserView recibe `variables: { colorPrimary: '#95B6E8', colorText:
  '#575756', borderRadius: '0.5rem' }`. `pt-32` → `pt-28` en todas
  las páginas de admin que se renderizan debajo de la navbar fija.
- **#39 (add modal)**: el shell compartido vive en
  `apps/admin/app/components/modal/AdminModal.tsx` y se importa vía
  barrel `apps/admin/app/components/modal/index.ts`. Footer default:
  Cancelar pink (`bg-uakari-pink hover:bg-uakari-pink-600`) +
  Confirmar lms-blue (`bg-lms-blue-500 hover:bg-lms-blue-600`). El
  logo LMS va arriba (`width={80} height={80}`) con
  `NEXT_PUBLIC_ADMIN_PREFIX` cuando aplique.
