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
- Shopify (cursos comprables, `shopify_id` y `shopify_handle`).
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
  embeds en `SubmoduleView.tsx`). El seeder los sigue creando. NO
  eliminar en features nuevas.

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
- `getHandle(shopify_id)` se llama solo si `shopify_id` está presente;
  si la API de Shopify falla, el handle queda `undefined` y el curso
  se crea igual.
