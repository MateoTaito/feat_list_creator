# paginas_familia — TODO de features

> Registro de las features producidas desde `feat_list_creator` y añadidas
> al `feature_list.json` del proyecto destino. Una entrada por feature.

## Estado

- **Proyecto:** `paginas_familia`
- **Ruta:** `../paginas_familia/`
- **Harness:** `nextjs-sdd-harness` (Next.js 16 + Uncle Bob flow)
- **`feature_list.json` destino:** `../paginas_familia/feature_list.json` (creado 2026-06-15)
- **Último `id` usado:** 7
- **Próximo `id` sugerido:** 8
- **Total features:** 7 (todas `pending`, todas `sdd: true`)

## Features producidas

| id | name | sdd | status | fecha |
|----|------|-----|--------|-------|
| 1  | `theme_and_globals`    | true | pending | 2026-06-15 |
| 2  | `header_navigation`    | true | pending | 2026-06-15 |
| 3  | `hero_section`         | true | pending | 2026-06-15 |
| 4  | `brand_story`          | true | pending | 2026-06-15 |
| 5  | `product_grid`         | true | pending | 2026-06-15 |
| 6  | `product_detail_page`  | true | pending | 2026-06-15 |
| 7  | `footer`               | true | pending | 2026-06-15 |

### Decisiones del batch (2026-06-15)

- **Nombre de marca usado:** "Mantamar" (heredado del placeholder del
  README `paginas_familia/README.md`). El usuario debe confirmar o
  cambiarlo. Si cambia, hay que buscar y reemplazar en todas las
  `acceptance` de las 7 features y, eventualmente, en el código que
  el `tdd_craftsman` produzca.
- **Paleta codificada en la feature #1** (tokens Tailwind v4 definidos
  en `src/app/globals.css` dentro de un bloque `@theme {}`):
  - `--color-cream: #F5EDE0` — café muy claro, fondo principal.
  - `--color-cream-dark: #E8D9C0` — variante de crema.
  - `--color-coffee: #C9A875` — acento cálido.
  - `--color-coffee-dark: #8B6F47` — acento oscuro.
  - `--color-ink: #0F0F0F` — negro casi puro, textos y detalles.
  - `--color-paper: #FFFFFF` — blanco, textos sobre `ink`.
- **Placeholders de imagen** aún no existen en `public/`:
  - `public/hero_placeholder.webp` (features #3).
  - `public/product_placeholder_2.svg` (features #4 y #5).
  Los tests de las features 3, 4 y 5 fallarán hasta que el usuario los
  coloque. Esto está documentado en sus `acceptance`.
- **CTA WhatsApp** en la feature #6: assumption cultural LATAM (los
  negocios pequeños de ropa en Chile suelen usar WhatsApp como canal
  principal). Si el usuario prefiere otro canal, hay que ajustar esa
  `acceptance`.
- **Estética codificada en descripciones/acceptance** (no se tocó
  `docs/conventions.md` del proyecto destino por decisión del usuario):
  "atrevida, moderna, simple pero no minimalista; clásica, elegante,
  rural; lana chilena y ponchos". El `craftsman_lead` lo extraerá de
  ahí al coordinar.

## Borradores en buffer (`feature_list.json` raíz)

_(vacío — este buffer solo se usa puntualmente; las features van directas
al proyecto destino.)_

## Ideas / features futuras (sin redactar)

> Chistera de ideas. Cuando se pida una, se redacta una a una y se borra
> de aquí al pasarla al proyecto destino.

- Página de contacto real (con formulario o mailto, en vez del ancla al footer).
- Sección de "testimonios" o "looks de clientes" (requeriría fotos reales).
- Filtro o categoría dentro del catálogo (ponchos, chales, bufandas, etc.).
- Página "Sobre el oficio" con más detalle del tejido y los materiales.
- Modo claro/oscuro (la paleta ya lo soporta, pero hay que decidir la UX).
- Internacionalización (es/en).

## Convenciones recordadas

- `name` en `snake_case` inglés, `title` y `description` en español.
- `status` siempre `pending` al crear; el harness lo mueve a
  `spec_ready → in_progress → done`.
- Marcar `"sdd": true` si la feature toca UI, integra módulos o tiene
  más de un criterio con lógica no trivial.
- Antes de añadir al `feature_list.json` del proyecto destino, verificar
  el último `id` existente.
