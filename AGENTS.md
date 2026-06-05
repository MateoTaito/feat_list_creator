# AGENTS.md — feat_list_creator

## Propósito del repositorio

Este repo es un **borrador / buffer** para redactar entradas del array
`features` de un `feature_list.json` consumido por un harness SDD. El
agente **no implementa código aquí**: solo produce una feature bien
formada por prompt. El usuario revisa y la mueve a mano al harness de
destino.

Harnesses de referencia cercanos (no se modifican desde aquí):

- `../nextjs-sdd-harness/` — NextJS 14+, TypeScript, Jest + RTL, Stryker.
- `../harness-sdd/` — variante Python / CLI.

## Reglas duras

- ❌ **Una sola feature por prompt.** Si el usuario pide varias, produce
  la primera, paras, y esperas confirmación antes de continuar.
- ❌ **No escribas a `../nextjs-sdd-harness/` ni a `../harness-sdd/`.**
  Todo lo que generas vive en `feature_list.json` de este repo.
- ❌ **No marques estados avanzados** (`spec_ready`, `in_progress`,
  `done`). El feature nuevo entra siempre como `"status": "pending"`.
- ❌ **No inventes detalles del proyecto objetivo.** Si el usuario
  menciona un proyecto que no conoces, **investígalo** (lee su `src/`,
  `docs/`, `README*`, o pide clarificación) antes de redactar la
  feature. Acepta specs ambiguas solo si el usuario lo dice
  explícitamente.
- ✅ **Salida:** un único objeto JSON que se pueda pegar tal cual dentro
  del array `features` del harness. Nada más.

## Esquema obligatorio (alineado con el harness)

Cada feature debe tener **exactamente** estos campos:

```json
{
  "id": <entero siguiente al último del array destino>,
  "name": "<snake_case corto, sin espacios, sin acentos>",
  "title": "<Título legible en español>",
  "description": "<1-3 frases: qué hace y por qué>",
  "acceptance": [
    "<criterio medible 1>",
    "<criterio medible 2>",
    "<criterio medible 3>"
  ],
  "status": "pending"
}
```

Campos opcionales (úsalos solo si aplican):

- `"sdd": true` — marca que la feature requiere spec Gherkin +
  conversación Spec Partner + TDD. **Regla del harness:** si la
  feature toca UI, integración entre módulos, o tiene más de un
  criterio de aceptación con lógica no trivial, marca `sdd: true`.
  Features puramente triviales (p. ej. renombrar, mover archivo)
  se quedan sin `sdd`.

### Estados válidos del harness

`pending` · `spec_ready` · `in_progress` · `done` · `blocked`

Tú solo produces `pending`. El resto los gestiona el harness.

## Convenciones de redacción

- `name` en `snake_case` inglés (`local_storage_setup`, `add_todo_form`).
- `title` y `description` en **español**, claros y concisos.
- `acceptance` con 3-5 bullets, cada uno **verificable** (un test o
  inspección manual concreta lo confirma). Evita "funciona
  correctamente", "se ve bien", "es rápido".
- Si la feature crea un archivo, el bullet debe decir la ruta exacta
  (p. ej. `src/lib/storage.ts`).
- Si la feature añade tests, el bullet nombra el archivo de test
  concreto.
- `id`: **no asumas un número**. Mira el `feature_list.json` del
  harness destino o pregunta al usuario si no puedes verificarlo.

## Flujo por prompt

1. Lee el pedido del usuario. Si nombra un proyecto que no conoces,
   explora su código/docs primero.
2. Decide `sdd: true/false` según la regla de arriba.
3. Redacta el objeto JSON siguiendo el esquema.
4. Devuélvelo en un bloque de código, sin preámbulo, sin postámbulo.
   Nada de "aquí tienes la feature", nada de resúmenes.

## Estado actual del repo

- `feature_list.json` existe pero está vacío. No tocar a menos que
  el usuario lo pida explícitamente (lo normal es entregar la
  feature suelta para pegarla a mano).
- No es un repo git, no tiene tests, no tiene build. No inventes
  comandos que no apliquen.
