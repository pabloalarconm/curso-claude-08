---
name: implement-issue-plan
description: Implementa el plan publicado como comentario en una issue de GitHub (leída con gh), trabajando en git worktrees, con TDD estricto y un commit por tarea. Si el plan afecta a varios paquetes del monorepo, lanza un agente por paquete, cada uno en su propio worktree, e integra sus ramas al final. Úsala cuando se pida implementar el plan de una issue.
argument-hint: <número de issue>
---

# implement-issue-plan

Issue a implementar: #$ARGUMENTS

**Regla de entrada:** `$ARGUMENTS` debe ser **un número de issue** (por ejemplo `12` o `#12`; quita el `#` si lo trae). Si está vacío o no es un número, pide al usuario el número de la issue y no continúes hasta tenerlo.

**Regla de oro:** todo el trabajo (tests, código, commits) se hace **dentro de worktrees**, nunca en el directorio principal del repositorio.

## 1. Leer la issue y su plan

1. Comprueba que `gh` está autenticado (`gh auth status`). Si no lo está, detente y pide al usuario que ejecute `gh auth login`.
2. Lee la issue con sus comentarios:

   ```bash
   gh issue view <numero> --json number,title,body,labels,state,url,comments
   ```

   - Si la issue no existe, detente y avisa al usuario.
   - Si está cerrada, pregunta al usuario si quiere implementarla igualmente.
3. Localiza el plan: es el comentario **más reciente** que contiene una sección `## Tareas` con una lista de tareas (`- [ ] 1. …`), normalmente publicado por la skill `plan-tdd-worktree`.
   - Si no hay ningún plan, detente y sugiere ejecutar antes `/plan-tdd-worktree <numero>`.
   - Si hay varios, usa el más reciente e indícalo al usuario.
4. Extrae del plan:
   - La **rama** (fila `**Rama**`). Si no aparece, constrúyela como `<tipo>/<numero>-<descripcion>` (tipo de Conventional Commits deducido de las etiquetas: `feature`/`enhancement` → `feat`, `bug` → `fix`, `documentation` → `docs`…; descripción en kebab-case sin tildes).
   - Las **tareas**, en orden, con su test y sus ficheros.
   - Los **criterios de aceptación**.
5. Guarda la URL del comentario del plan para referenciarla al final.

## 2. Decidir cuántos agentes hacen falta

1. Asigna cada tarea a un **paquete** del monorepo según los ficheros que toca: `api`, `web-shared`, `web-admin`, `web-empleados` o `web-clientes` (`packages/<paquete>/…`). Las tareas que solo tocan `docs/` o `scripts/` van con el paquete más relacionado o, si no hay ninguno, con `api`.
2. Contrasta el resultado con las etiquetas `proyecto:*` de la issue. Si alguna tarea toca varios paquetes, divídela por paquete manteniendo el orden.
3. Muestra al usuario el reparto (paquete → tareas) y espera su confirmación.
4. Según el reparto:
   - **Un solo paquete** → modo individual (sección 3): lo implementas tú en un único worktree.
   - **Varios paquetes** → modo multiagente (sección 4): un agente por paquete, cada uno en su propio worktree.

## 3. Modo individual

1. Prepara el worktree (ver sección 6) para la rama del plan.
2. Implementa todas las tareas siguiendo el ciclo de la sección 5.
3. Continúa con el cierre (sección 7).

## 4. Modo multiagente

### 4.1. Ramas y worktrees

- **Rama de integración:** la rama del plan, `<tipo>/<numero>-<descripcion>`, en su propio worktree (sección 6). Aquí no se implementa nada: solo se integran las ramas de los agentes.
- **Rama de cada agente:** `<tipo>/<numero>-<descripcion>-<paquete>`, creada desde la rama de integración, en el worktree `../<nombre-del-repo>-worktrees/<tipo>-<numero>-<descripcion>-<paquete>`.

### 4.2. Orden de ejecución por oleadas

Los frontends dependen de la API y de `web-shared`, así que los agentes se lanzan en oleadas:

1. **Oleada 1:** `api` y `web-shared` (los que tengan tareas), en paralelo.
2. Integra sus ramas en la rama de integración (sección 4.4).
3. **Oleada 2:** los frontends con tareas (`web-admin`, `web-empleados`, `web-clientes`), en paralelo. Sus ramas se crean **después** de integrar la oleada 1, para que partan de la API y de `web-shared` ya terminados.
4. Integra sus ramas en la rama de integración.

Si una oleada no tiene tareas, sáltala.

### 4.3. Lanzar a cada agente

1. Antes de lanzar una oleada, crea el worktree de cada agente de esa oleada (sección 6) e instala dependencias en cada uno.
2. Lanza **un agente `general-purpose` por paquete** con la herramienta Agent, todos los de la oleada **en el mismo mensaje** para que trabajen en paralelo. No uses `isolation: "worktree"`: el worktree ya lo has creado tú.
3. Cada agente recibe un prompt autocontenido con:
   - El número, título y URL de la issue, y el objetivo del plan.
   - La **ruta absoluta de su worktree** y su rama, con la orden de usar solo rutas absolutas dentro de ese worktree y de no tocar ningún otro worktree ni el directorio principal.
   - El paquete que le corresponde y que **solo puede modificar ficheros de ese paquete**. Si necesita cambiar otro paquete, debe detenerse y explicarlo en su informe.
   - Sus tareas, copiadas literalmente del plan y con su numeración original.
   - El contrato relevante del resto del plan (endpoints, DTOs, modelos) para que los frontends sepan qué consumir.
   - Las reglas completas de la sección 5 (TDD estricto y un commit por tarea).
   - La orden de **no hacer push, merge ni eliminar el worktree**.
   - Lo que debe devolver: tareas completadas, hash y mensaje de cada commit, resultado final de los tests o del build, y problemas o tareas pendientes.
4. Cuando terminen todos los agentes de la oleada, revisa sus informes y comprueba en cada worktree que `git log <rama-de-integración>..HEAD --oneline` tiene un commit por tarea. Si un agente falló o dejó tareas sin hacer, informa al usuario y pregunta cómo seguir antes de lanzar la siguiente oleada.

### 4.4. Integrar las ramas de los agentes

En el worktree de integración, por cada rama de agente de la oleada:

```bash
git merge --no-ff <tipo>/<numero>-<descripcion>-<paquete> -m "merge(<paquete>): integrate tasks for #<numero>"
```

- Si hay conflictos, **no los resuelvas a ciegas**: muéstralos al usuario y pregúntale.
- Tras integrar la oleada, ejecuta `npm install` y `npm test` (y el build de los frontends integrados) en el worktree de integración. Si algo falla, detente e informa.

## 5. Ciclo de implementación: TDD estricto y un commit por tarea

Para **cada** tarea, en orden, dentro del worktree que corresponda:

1. **Red**: escribe primero el test (junto al código, `*.test.ts`, usando los dobles de `repositories/mocks/` o `contexts/employee/application/mocks/` cuando aplique). Ejecútalo y comprueba que **falla por el motivo correcto**. Si pasa sin código nuevo, el test no sirve: corrígelo.
2. **Green**: escribe el mínimo código de producción para que pase.
3. **Refactor**: limpia el código y los tests sin salir del verde.
4. Ejecuta la **suite completa** (`npm test` desde la raíz del worktree) y comprueba que pasa. No avances si no está en verde.
5. **Commit de la tarea**: añade solo los ficheros de esa tarea y haz un commit que siga Conventional Commits, como el comando `/commit`:
   - `<tipo>(<paquete>): <descripción en inglés, imperativo, minúsculas, sin punto>`.
   - Footer `Refs #<numero>` y, en el cuerpo, la referencia a la tarea (`Task <n> of the plan`).
   - Nunca añadas `node_modules/`, `.DS_Store`, `dist/`, `.angular/` ni `packages/api/resttek.db`.
   - Usa un HEREDOC para el mensaje y confirma con `git log --oneline -1`.
6. Pasa a la siguiente tarea solo después del commit.

Notas:
- No escribas código de producción sin un test en rojo que lo justifique.
- Test de un solo fichero: `cd <worktree>/packages/api && npx vitest run <ruta/al/test.ts>`.
- Los frontends no tienen tests: verifica cada tarea de frontend con `npm run build -w @resttek/<paquete>` antes de su commit y menciónalo en el cuerpo del commit (`Verified with build, no test runner in this package`).
- El código, los tests y los mensajes de commit se escriben en inglés; los mensajes al usuario y a la issue, en español.

## 6. Preparar un worktree

1. Rama base de la rama del plan: `development` si existe (`git rev-parse --verify --quiet development` u `origin/development`); si no, `main`; si tampoco, `master`.
2. Ruta: `../<nombre-del-repo>-worktrees/<rama-con-/-sustituido-por-->` (hermana del repositorio).
3. Comprueba antes qué existe (`git worktree list` y `git rev-parse --verify --quiet <rama>`):
   - **El worktree de esa rama ya existe** (por ejemplo, porque lo creó `plan-tdd-worktree`): reutilízalo. Si tiene cambios sin commitear que no son el fichero del plan, avisa al usuario antes de seguir.
   - **La rama existe pero no tiene worktree**: `git worktree add <ruta> <rama>`.
   - **No existe ninguna**: `git worktree add -b <rama> <ruta> <base>` (en modo multiagente, la base de las ramas de los agentes es la rama de integración).
   - Si la ruta existe pero corresponde a otra rama, avisa al usuario en vez de sobrescribirla.
4. Instala dependencias: `cd <ruta> && npm install`.
5. Si el plan está en `docs/plans/` dentro del worktree de la rama del plan sin commitear, haz un primer commit `docs(plans): add implementation plan for #<numero>` antes de empezar.

## 7. Cierre

1. En el worktree final (el único en modo individual, o el de integración en modo multiagente) ejecuta la suite completa y el build de cada frontend tocado.
2. Comprueba los criterios de aceptación del plan.
3. Publica un comentario de progreso en la issue:

   ```bash
   gh issue comment <numero> --body-file <fichero-temporal>
   ```

   Con este contenido, en español:
   - Enlace al comentario del plan.
   - Rama con el resultado final.
   - Lista de tareas con el hash corto de su commit (`- [x] 1. <tarea> — <hash>`).
   - Criterios de aceptación cumplidos o pendientes.
   - Resultado de los tests y los builds.

   Si `gh` falla, avisa al usuario, muéstrale el texto y continúa.
4. Resume al usuario lo hecho: rama final, worktrees creados y commits por tarea.
5. **No hagas push, no abras PR, no cierres la issue y no elimines worktrees** salvo que el usuario lo pida. Para limpiar después: `git worktree remove <ruta>` y `git branch -d <rama>` en cada rama de agente ya integrada.
