# paginas_familia — Memoria del proyecto destino

> Resumen de arquitectura, funcionamiento y particularidades. Léeme antes
> de redactar una feature para no tener que recorrer el proyecto cada vez.

## Identidad

- **Nombre interno:** `paginas_familia` (package.json) / "mantamar" (README placeholder, usado como **nombre de marca** tentativo en las features).
- **Tipo:** sitio web estático de una tienda de ropa adulta.
- **Ruta:** `../paginas_familia/` (hermano de `feat_list_creator/`).
- **README actual:** placeholder `# mantamar` (un solo línea).
- **Producto acordado (2026-06-15):** tienda de ropa centrada en público adulto, especializada en prendas de lana chilena (ponchos, chales y similares). Estética: clásica, elegante y rural; atrevida y moderna, simple pero no minimalista. Tono de marca: terrenal, oficio artesano.

## Dominio del producto (definido 2026-06-15)

- **Audiencia:** público adulto (no infantil).
- **Producto estrella:** ponchos de lana chilena. La marca presumiblemente ofrece también otras prendas de lana (chales, bufandas) — el `craftsman_lead` puede ampliar el catálogo de placeholder.
- **Tono de comunicación:** clásico, elegante, rural. Nada de "moda rápida" ni "fast fashion".
- **Identidad visual (codificada en la feature #1 como tokens Tailwind v4):**
  - Fondo principal: café muy claro (`#F5EDE0`).
  - Textos y detalles: negro casi puro (`#0F0F0F`).
  - Textos sobre detalles negros: blanco (`#FFFFFF`).
  - Acentos: variantes de café (`#E8D9C0`, `#C9A875`, `#8B6F47`).
  - Estilo: "atrevido, moderno, simple pero no minimalista". El `tdd_craftsman` debe usar tipografía con peso (serif display para titulares, sans para cuerpo) y bastante espacio en blanco para evitar el minimalismo plano.
- **Placeholders de imagen acordados:**
  - Productos (verticales): `public/product_placeholder_2.svg`.
  - Hero / banners (horizontales): `public/hero_placeholder.webp`.
  Ambos **no existen aún** en `public/`; los tests de las features 3, 4 y 5 fallarán hasta que se coloquen.

## Stack (versiones exactas)

| Capa | Versión | Notas |
|------|---------|-------|
| Next.js | `16.2.9` | App Router; **no** es el Next.js "de siempre" — leer `node_modules/next/dist/docs/` antes de tocar `src/`. El `AGENTS.md` raíz lo advierte. |
| React | `19.2.4` | |
| TypeScript | `^5` | |
| Tailwind | `v4` | Config **en CSS** (`src/app/globals.css` con `@import "tailwindcss"`), **no** existe `tailwind.config.js`. |
| Jest | `^29.7` | Usa `next/jest` (NO `ts-jest` directo). `jest.config.js` + `jest.setup.js` ya adaptados a Next 16. |
| ESLint | `^9` | `eslint.config.mjs`. |
| Stryker | `^8` | Para mutación (`tools/mutate.js` y `tools/stryker.conf.js`). |
| PostCSS | config en `postcss.config.mjs` (`.mjs`, no `.js`). |

Package manager: `pnpm` tiene lockfile (`pnpm-lock.yaml`); `npm` también
(`package-lock.json`). Ambos coexisten — preferir `pnpm` (lo detectó
`init.sh`).

## Harness SDD (estilo Uncle Bob)

Rama conceptual: `uncle-bob-harness`. El repo no es un proyecto normal;
es un **proceso** sobre un proyecto trivial para enseñar el pipeline.

### Pipeline

```
pending
  → spec_partner    → project-spec.md
  → gherkin_author  → features/<name>.feature   (status: spec_ready)
  → ⏸ HUMANO APRUEBA los escenarios
  → in_progress
  → tdd_craftsman   → src/ + tests/  (Rojo → Verde → Refactor, un test a la vez)
  → judge           → progress/judge_<name>.md
  → mutation_tester → progress/mutation_<name>.md
  → done
```

### Subagentes

- `craftsman_lead` — orquestador. Definido **dentro de** `AGENTS.harness.md`
  (no hay `.agents/agents/craftsman_lead.md`). **Yo (feat_list_creator)
  no actúo como este rol**; yo solo produzco features `pending`. El
  `craftsman_lead` se lanza dentro del proyecto destino, no desde aquí.
- `spec_partner`, `gherkin_author`, `tdd_craftsman`, `judge`,
  `mutation_tester` — viven en `../paginas_familia/.agents/agents/`.

### Archivos del harness que ya existen

- `AGENTS.harness.md` — reglas del orquestador.
- `AGENTS_2.harness.md` — mapa de navegación.
- `CHECKPOINTS.harness.md` — C1–C7 de "estado correcto".
- `init.sh` — verificación de entorno (auto-detecta `feature_list.json`).
- `tools/mutate.js` + `tools/stryker.conf.js`.
- `progress/current.md` (plantilla vacía) y `progress/history.md`
  (1 entrada: "Sesión 0 — Importación del harness").
- `docs/{architecture,conventions,gherkin,mutation-testing,tdd,verification,workflow}.md`.

### Estado del catálogo de features (2026-06-15)

- `feature_list.json` — **CREADO** con 7 features, todas `pending`,
  todas `sdd: true` (toda la primera capa de UI). El archivo está
  formateado como `{"features": [...]}` (verificado contra `init.sh`).
- Ids 1–7 reservados:
  1. `theme_and_globals`
  2. `header_navigation`
  3. `hero_section`
  4. `brand_story`
  5. `product_grid`
  6. `product_detail_page`
  7. `footer`
- Próximo `id` disponible: **8**.

### Lo que **NO** existe todavía

- `project-spec.md` — la spec conversada. La escribe `spec_partner`
  cuando se abra la primera feature. El `craftsman_lead` debería
  empezar por aquí.
- `features/*.feature` — directorio existe con solo `.gitkeep`. Se
  poblará al destilar cada feature (`@s1`, `@s2`, …).
- `tests/` — directorio **no existe**. Lo crea `tdd_craftsman` cuando
  llegue el momento.
- `public/hero_placeholder.webp` y `public/product_placeholder_2.svg`
  — los tiene que colocar el usuario; son los placeholders acordados
  para todas las imágenes de la web.

## Arquitectura (de `docs/architecture.md`)

Tres capas, solo tres. **No inventar más hasta tener razón documentada.**

```
src/lib/        — lógica de negocio pura (modelos, utilidades)
src/components/ — componentes React presentacionales
src/app/        — páginas y rutas Next (App Router)
```

Reglas duras:

1. Dependencias: solo Next, React, Tailwind. Cualquier dep nueva → estado `blocked`.
2. Errores: funciones que puedan fallar **lanzan** errores descriptivos, no devuelven `undefined`.
3. Inmutabilidad: modelos `readonly`, modificar = crear instancia nueva.
4. Persistencia: `localStorage` para simplicidad. **Nunca** mezclar IO con
   lógica de dominio dentro de `src/lib/`. Cargar al inicio, mutar en
   memoria, guardar al final.
5. `console.log` solo para errores reales con UI; nada de debug.
6. Sin sistema de configuración global; claves de localStorage explícitas
   o constante por defecto.

> **Caveat:** la doc de arquitectura usa un ejemplo de "todo list"
> (`AddTodo` → `TodoList` → `storage.ts`). **Es un ejemplo heredado
> del harness, no del producto real** (`paginas_familia` → tienda de
> ropa de lana chilena). No proponer features de todo list.

## Estado actual de `src/`

- `src/app/layout.tsx` — layout raíz por defecto de `create-next-app`
  (fuente Geist, body con `min-h-full flex flex-col`, `lang="en"`,
  metadata "Create Next App" / "Generated by create next app"). La
  feature #1 lo reescribirá a `lang="es"` con metadata de Mantamar.
- `src/app/page.tsx` — landing de `create-next-app` con logos de
  Next/Vercel. La feature #1 lo reescribirá con un placeholder
  `<main>` y las features 3, 4, 5 y 6 lo irán completando.
- `src/app/globals.css` — config de Tailwind v4. La feature #1
  añadirá el bloque `@theme {}` con los tokens de marca.
- **No** existen `src/lib/` ni `src/components/` todavía. La feature
  #5 introducirá `src/lib/products.ts` y la #2 introducirá
  `src/components/Header.tsx`.

## Convenciones de nombres (de `docs/conventions.md`)

_(No leído en detalle en esta pasada; consultar antes de la primera
feature para no inventar reglas.)_

## Cómo verificar

- Lint: `pnpm lint` o `npm run lint` → `eslint`.
- Tests: `pnpm test` (Jest) o `pnpm test:harness` (config explícita).
- Mutación: `pnpm mutate` → corre `node tools/mutate.js` con archivos
  específicos como argumento.
- Build: `pnpm build`.

## Riesgos / cosas a vigilar

1. **Next.js 16 es "nuevo"** para el modelo. Antes de cualquier feature
   UI, leer `node_modules/next/dist/docs/` (lo exige `AGENTS.md` raíz).
2. **Tailwind v4** rompe la intuición de `tailwind.config.js`; vive en
   `src/app/globals.css`. Si una feature necesita tokens/extenderse,
   tocar el CSS, no crear config JS.
3. **Doble package manager** (`pnpm-lock.yaml` + `package-lock.json`).
   No instalar con `npm` y luego con `pnpm` en la misma sesión.
4. **Brand name "Mantamar"** está pendiente de confirmación. Si el
   usuario lo cambia, hay que actualizar las `acceptance` de las 7
   features y posiblemente volver a redactar las que ya estén
   `spec_ready` o `in_progress`.
5. **Placeholders de imagen no existen** — los tests de las features
   3, 4 y 5 fallarán hasta que `public/hero_placeholder.webp` y
   `public/product_placeholder_2.svg` estén colocados.
6. **Ejemplo de la doc de arquitectura (todo list)** no es el
   producto. No atar las features a ese dominio.
7. **Feature #1, #2, #3, #4 y #5 modifican `src/app/page.tsx` y/o
   `src/app/layout.tsx`** — el `craftsman_lead` debe asegurarse de
   que se integren sin sobrescribirse unas a otras. Lo más seguro es
   implementar en orden (1 → 7) y que cada feature solo añada su
   pieza.
