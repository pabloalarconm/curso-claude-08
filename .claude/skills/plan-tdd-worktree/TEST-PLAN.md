# Plan de prueba: avisos de Slack de `plan-tdd-worktree`

Objetivo: comprobar que la skill publica en `#planes-generales` un mensaje al crear el plan y otro al terminar la implementación, y que no se bloquea si Slack falla.

## Requisitos previos

| # | Requisito | Cómo comprobarlo |
|---|---|---|
| P1 | `node` y `npx` están en el `PATH` del proceso de Claude Code (Node está instalado con nvm) | En la terminal desde la que se lanza VS Code: `command -v npx` devuelve una ruta |
| P2 | Las variables `BOT_SLACK_TOKEN` y `TEAMID_SLACK` existen en ese mismo entorno | Las comprueba la persona usuaria; Claude no debe leer el token |
| P3 | El bot de Slack tiene los permisos `channels:read`, `chat:write` y `chat:write.public` (o está invitado a `#planes-generales`) | Configuración de la app en api.slack.com |
| P4 | El servidor MCP `slack` aparece como conectado | `/mcp` en Claude Code muestra `slack · connected` |

Si P1 falla, el síntoma es el error `Executable not found in $PATH: npx` al arrancar. Solución: cargar nvm en `~/.profile` (o `~/.bash_profile`) y abrir VS Code desde una terminal donde `node -v` funcione. Después reiniciar Claude Code.

## Casos de prueba

### T1 — El MCP expone las herramientas de Slack
- **Pasos:** pedir a Claude que busque las herramientas `mcp__slack__slack_list_channels` y `mcp__slack__slack_post_message`.
- **Resultado esperado:** las dos herramientas se encuentran.

### T2 — Se resuelve el canal `planes-generales`
- **Pasos:** llamar a `mcp__slack__slack_list_channels` y buscar `planes-generales` (paginando con `cursor` si hace falta).
- **Resultado esperado:** se obtiene un `channel_id` que empieza por `C`.

### T3 — Envío directo de un mensaje de prueba
- **Pasos:** llamar a `mcp__slack__slack_post_message` con el `channel_id` de T2 y el texto `:test_tube: Prueba de notificación de plan-tdd-worktree`.
- **Resultado esperado:** la respuesta tiene `ok: true` y el mensaje se ve en `#planes-generales`.

### T4 — Aviso de plan creado (flujo completo)
- **Pasos:** ejecutar `/plan-tdd-worktree añadir un test de caso feliz para DishService.findByRestaurantId` y parar en la confirmación del plan.
- **Resultado esperado:** en `#planes-generales` aparece `:memo: Plan creado: …` con la rama, el número de tareas, la ruta del plan y el resumen.

### T5 — Aviso de implementación terminada (flujo completo)
- **Pasos:** confirmar el plan de T4 y dejar que la skill termine.
- **Resultado esperado:** en `#planes-generales` aparece `:white_check_mark: Implementación terminada: …` con el número de tareas completadas y de tests en verde.

### T6 — Slack no disponible
- **Pasos:** ejecutar la skill con el MCP `slack` desconectado (por ejemplo, sin `npx` en el `PATH`).
- **Resultado esperado:** la skill avisa del fallo, muestra el texto que se iba a enviar y continúa con el siguiente paso sin bloquearse.

## Resultados

| Caso | Fecha | Resultado | Notas |
|---|---|---|---|
| T6 | 2026-10-07 | ✅ OK | El MCP falló con `npx` no encontrado; la skill mostró el mensaje del plan y siguió |
| T1-T5 | — | ⏳ Pendiente | Bloqueado por P1 (Node de nvm no está en el `PATH` de Claude Code) |
