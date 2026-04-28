# 0001 — Usar Claude Agent SDK (vía nanoclaw) en lugar de SDK plano de Anthropic

**Fecha:** 2026-04-28
**Estado:** Aceptada
**Decisores:** emorfin (Cyborgdev / ÉLEVÉ), Claude Code

## Contexto

ÉLEVÉ ya tiene un agente conversacional ("Mónica") implementado como una edge function de Supabase (`chat-assistant`) que carga un `manifest.json` de tools y orquesta llamadas al LLM. Funciona, pero acumula limitaciones:

1. **Acoplamiento al stack Supabase**: el agente vive como edge function Deno; agregar capacidades (memoria persistente, MCP, sub-agentes, herramientas filesystem) implica reinventar primitivas que el SDK ya resuelve.
2. **Dificultad para evolucionar el comportamiento del agente**: cambiar el system prompt o agregar tools requiere redeploy de edge function.
3. **Rigidez frente a multi-tenancy**: ÉLEVÉ podría querer en el futuro agentes con personalidades distintas (ventas vs soporte vs general). El esquema actual acopla todo en una sola función.

Necesitamos un agente con:
- Memoria persistente por conversación.
- Capacidad de invocar tools externas vía MCP.
- System prompt configurable sin redeploy.
- Aislamiento por agent group para futuras extensiones.
- Capacidad de invocar sub-agentes (Task tool).

## Opciones consideradas

### Opción A — SDK plano de Anthropic (`anthropic` SDK)

- Implementamos nosotros el agent loop, manejo de tool use, memoria, MCP client, etc.
- **Pros:** control total, dependencias mínimas.
- **Contras:** reinventamos lo que ya existe; semanas de trabajo de plumbing antes de funcionalidad de negocio; mantenimiento perpetuo.

### Opción B — Claude Agent SDK directo

- Usar `@anthropic-ai/claude-agent-sdk` y construir nuestro propio runtime.
- **Pros:** trae agent loop, MCP support, file tools, hooks, subagents listos.
- **Contras:** seguimos teniendo que armar la capa HTTP, channel adapters, persistencia, dockerización. Otro montón de plumbing.

### Opción C — Fork de un proyecto que ya use el Agent SDK

Investigando, encontramos [`qwibitai/nanoclaw`](https://github.com/qwibitai/nanoclaw): proyecto open source MIT que ya provee:
- Agent runtime containerizado con Claude Agent SDK.
- Modelo de "agent groups" con CLAUDE.md por grupo.
- Channel adapters extensibles (skills `/add-whatsapp`, `/add-telegram`, etc.).
- Sub-agentes vía Task tool.
- SQLite por sesión, sweep, cron, todo el plumbing.

Filosofía explícita: "small enough to understand, customization = code changes". Ese mindset encaja con que necesitamos forkear y modificar.

- **Pros:** reusamos meses de trabajo; nos enfocamos solo en la integración con ÉLEVÉ; codebase chico (~handful de files), entendible.
- **Contras:** dependencia de proyecto upstream; tracking de updates manual; nos obligamos a TS/Bun (perdemos opción de Python); customizaciones nuestras viven en fork (potencial drift).

## Decisión

**Opción C: forkear `qwibitai/nanoclaw` y customizar.**

Motivos:
1. Maximiza reuso de código probado.
2. La filosofía del upstream (skills + customization) coincide con nuestras necesidades.
3. La cantidad de customización necesaria (channel adapter custom para ÉLEVÉ + system prompt loader configurable) es acotada.
4. TS/Bun no es bloqueante; el ecosistema MCP en TS es maduro.

## Consecuencias

**Positivas:**
- Time-to-first-message dramáticamente más corto.
- Los avances upstream (mejoras al SDK, providers nuevos, etc.) los podemos rebasear si nos sirven.
- mcp-monica como container separado es trivial — nanoclaw acepta MCP servers HTTP directos.

**Negativas:**
- Dependemos de la salud del proyecto upstream. Mitigación: fork es nuestro; podemos pinear o divergir si hace falta.
- Cualquier customización nuestra en archivos del trunk genera fricción al rebase. Mitigación: documentar claramente en `nanoclaw/docs/customizations.md` qué se modificó vs upstream.
- Si en el futuro queremos cambiar a Python o a otro stack, hay costo de migración.

## Notas

- El system prompt runtime NO vive en `groups/<grupo>/CLAUDE.md` del fork como hace upstream. Lo cargamos por env (`AGENT_SYSTEM_PROMPT_SOURCE`) para evitar rebuilds en cada cambio de prompt y permitir editarlo desde Google Drive. Ver spec sección 6.
- Los channel adapters upstream (Telegram, Discord, etc.) se quedan pero no los usamos. nanoclaw expone solo el adapter custom `eleve-http`.
