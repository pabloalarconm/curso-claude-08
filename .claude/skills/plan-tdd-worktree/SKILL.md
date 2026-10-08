---
name: plan-tdd-worktree
description: Lee una issue de GitHub con gh, crea un git worktree aislado, redacta un plan por tareas pequeñas en docs/plans, lo publica como comentario en la issue, avisa por Slack en #planes-generales, lo implementa con TDD estricto y vuelve a avisar por Slack al terminar. Úsala cuando se pida planificar e implementar una issue del repositorio.
argument-hint: <número de issue>
---

# plan-tdd-worktree

Issue a planificar: #$ARGUMENTS

**Regla de entrada:** `$ARGUMENTS` debe ser **un número de issue** (por ejemplo `12` o `#12`; quita el `#` si lo trae). Si está vacío o no es un número, pide al usuario el número de la issue y no continúes hasta tenerlo.

**Regla de oro:** todos los cambios (plan, tests y código) se hacen **dentro del worktree**, nunca en el directorio principal del repositorio.

## 0. Leer la issue

1. Comprueba que `gh` está autenticado (`gh auth status`). Si no lo está, detente y pide al usuario que ejecute `gh auth login`.
2. Lee la issue del repositorio actual:

   ```bash
   gh issue view <numero> --json number,title,body,labels,state,url,comments
   ```

   - Si la issue no existe, detente y avisa al usuario.
   - Si está cerrada (`state: CLOSED`), pregunta al usuario si quiere planificarla igualmente.
3. La tarea a resolver es lo que describen el título, el cuerpo y los comentarios de la issue. Si es demasiado ambigua para planificarla, pregunta al usuario antes de continuar.
4. Guarda `number`, `title` y `url`: se usan en la rama, el plan, el comentario y los avisos de Slack.

## 1. Crear el worktree

1. Deduce el `<tipo>` de la tarea (`feat`, `fix`, `refactor`, `docs`, `chore`, `test`…, igual que en Conventional Commits) a partir de las etiquetas de la issue (`feature` o `enhancement` → `feat`, `bug` → `fix`, `documentation` → `docs`…) o, si no hay, de su contenido. La `<descripcion>` empieza por el número de la issue y sigue con un resumen corto en kebab-case, sin tildes ni espacios (por ejemplo `12-anadir-filtro-por-canal`).
2. La rama será `<tipo>/<descripcion>`.
3. Rama base: `development` si existe (`git rev-parse --verify --quiet development` u `origin/development`); si no, `main`; si tampoco existe, `master`.
4. Ruta del worktree: `../<nombre-del-repo>-worktrees/<tipo>-<descripcion>` (hermana del repositorio, para no ensuciar el árbol principal).
5. Créalo:

   ```bash
   git worktree add -b <tipo>/<descripcion> <ruta-worktree> <base>
   ```

   - Si la rama o la ruta ya existen, avisa al usuario en vez de sobrescribirlas.
   - Los cambios sin commitear del directorio principal no se tocan.
6. Instala dependencias dentro del worktree (es un monorepo con workspaces): `npm install --prefix <ruta-worktree>` o `cd <ruta-worktree> && npm install`.
7. A partir de aquí, **todas** las lecturas, ediciones y comandos usan rutas absolutas dentro de `<ruta-worktree>`.

## 2. Crear el plan

1. Lee `CLAUDE.md` y la documentación relevante de `docs/` y explora el código afectado para que las tareas sean realistas. Comprueba qué estilo arquitectónico usa el dominio (hexagonal en `employee`, por capas en el resto).
2. Guarda el plan en `<ruta-worktree>/docs/plans/<tipo>-<descripcion>.md`, en español, con esta estructura:

   ```markdown
   # <Título de la issue>

   | | |
   |---|---|
   | **Issue** | [#<numero>](<url>) |
   | **Rama** | `<tipo>/<descripcion>` |
   | **Worktree** | `<ruta-worktree>` |
   | **Fecha** | AAAA-MM-DD |

   ## Objetivo

   <Qué se quiere conseguir y por qué, en pocas líneas.>

   ## Criterios de aceptación

   - [ ] …

   ## Tareas

   - [ ] 1. <Tarea> — test: `<fichero.test.ts>` — ficheros: `<rutas>`
   - [ ] 2. …

   ## Riesgos y preguntas abiertas

   - …
   ```

   Reglas para las tareas:
   - Cada una se implementa en **5-10 minutos como máximo**; si es más grande, divídela.
   - Cada una describe un único cambio verificable e indica el test que lo cubre y los ficheros que toca.
   - Ordénalas para que el proyecto compile y los tests pasen tras cada una.

3. **Publica el plan como comentario en la issue**:

   ```bash
   gh issue comment <numero> --body-file <ruta-worktree>/docs/plans/<tipo>-<descripcion>.md
   ```

   - Guarda la URL del comentario que devuelve `gh` para mostrarla al usuario.
   - Si el comando falla, **no detengas el flujo**: avisa al usuario del error y continúa.

4. **Avisa por Slack** (ver sección 5) en el canal `planes-generales` con un mensaje como:

   > :memo: Plan creado: *<Título>* (issue #<numero>)
   > Rama: `<tipo>/<descripcion>` · Tareas: <N>
   > Fichero: `docs/plans/<tipo>-<descripcion>.md`
   > Resumen: <objetivo en 1-2 líneas>

5. Muestra el plan al usuario junto con el enlace al comentario de la issue y espera su confirmación antes de implementar.

## 3. Implementar con TDD estricto

Para **cada** tarea, en orden, dentro del worktree:

1. **Red**: escribe primero el test (colocado junto al código, `*.test.ts`, usando los dobles de `repositories/mocks/` o `contexts/employee/application/mocks/` cuando aplique). Ejecútalo y comprueba que **falla por el motivo correcto**. Si pasa sin código nuevo, el test no sirve: corrígelo.
2. **Green**: escribe el mínimo código de producción para que pase.
3. **Refactor**: limpia código y tests manteniéndolo todo en verde.
4. Ejecuta la **suite completa** (`npm test` desde la raíz del worktree) y comprueba que pasa.
5. Marca la tarea en el plan (`- [ ]` → `- [x]`) **inmediatamente**, antes de pasar a la siguiente.

Notas:
- No escribas código de producción sin un test en rojo que lo justifique.
- No avances si la suite no está en verde.
- Test de un solo fichero: `cd <ruta-worktree>/packages/api && npx vitest run <ruta/al/test.ts>`.
- Los frontends no tienen tests: si una tarea es solo de frontend, verifícala con `npm run build -w <paquete>` y descríbelo en el plan.
- El código y los tests se escriben en inglés; el plan y los mensajes, en español.

## 4. Cierre

1. Ejecuta la suite completa una última vez y, si se tocó algún frontend, su build.
2. Marca los criterios de aceptación cumplidos en el plan.
3. **Avisa por Slack** (ver sección 5) en el canal `planes-generales`:

   > :white_check_mark: Implementación terminada: *<Título>* (issue #<numero>)
   > Rama: `<tipo>/<descripcion>` · Tareas completadas: <N>/<N>
   > Tests: <nº pasados> en verde
   > Cambios: <resumen en 2-3 líneas>

4. Resume al usuario lo hecho e indica la ruta del worktree. No hagas commit, push ni elimines el worktree salvo que el usuario lo pida (para commits está el comando `/commit`; para limpiar, `git worktree remove <ruta-worktree>`).

## 5. Cómo avisar por Slack

Usa el servidor MCP `slack` (configurado en `.mcp.json`):

1. Obtén el ID del canal con `mcp__slack__slack_list_channels` buscando el nombre `planes-generales` (pagina con `cursor` si hace falta).
2. Envía el mensaje con `mcp__slack__slack_post_message` (`channel_id`, `text`).
3. Si las herramientas de Slack no están disponibles o el envío falla, **no detengas el flujo**: avisa al usuario del fallo, muéstrale el texto que se iba a enviar y continúa.
