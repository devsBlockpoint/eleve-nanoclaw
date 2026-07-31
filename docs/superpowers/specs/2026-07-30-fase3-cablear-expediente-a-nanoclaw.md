# Fase 3.3 — Cablear el expediente vivo al canal de texto (nanoclaw)

**Estado:** diseñado y mapeado contra el código. **No implementado.**
**Depende de:** 3.1 (tabla `paciente_estado`) y 3.2 (edge function `patient-context`) — ambas listas.

## Por qué está separado

3.1 y 3.2 se entregaron completos: por voz, el expediente ya viaja de punta a punta
(webhook de inicio → dynamic variable → prompt). Por texto falta, y es la parte más grande,
porque **nanoclaw no tiene concepto de persona**: cada `conversation_id` es una identidad
nueva y aislada, así que una paciente que vuelve recibe amnesia total.

Se documenta en vez de implementarse a medias **a propósito**. La lección que este trabajo
persigue es justamente la de Valentina: fontanería construida y nunca cableada no sirve de
nada. Media implementación —el contexto escrito en la base pero que nada lee— sería
exactamente ese error, con el costo extra de parecer terminado.

## Caminos evaluados y descartados

| Camino | Por qué no |
|---|---|
| Meterlo en `metadata` del mensaje | El formatter del contenedor (`container/agent-runner/src/formatter.ts`) **no expone `metadata` al modelo**: renderiza `<message sender=… time=…>texto</message>`. Habría que tocar el contenedor igual. |
| Componerlo en el `CLAUDE.md` del grupo | `composeGroupClaudeMd()` es **por agent group, no por sesión**. Todas las conversaciones de Mónica comparten ese archivo → el contexto de una paciente quedaría visible en la sesión de otra. **Fuga de datos personales.** Descartado. |
| Inyectarlo como un mensaje más en `messages_in` | No requiere tocar el contenedor, pero queda en el historial de chat: se repite, se ve en el transcript y el modelo lo trata como dicho por alguien, no como instrucción. |

## Camino elegido: tabla de sesión, espejo de `session_routing`

`session_routing` ya resuelve exactamente esta forma: el host escribe una fila en `inbound.db`
antes de cada wake y el contenedor la lee. Se copia ese patrón, que además ya está probado.

### Puntos de inserción (verificados en el código)

| # | Archivo | Cambio |
|---|---|---|
| 1 | `src/db/schema.ts` (~línea 205, junto a `session_routing`) | Agregar `session_person_context` a `INBOUND_SCHEMA`: `id INTEGER PRIMARY KEY CHECK (id=1)`, `paciente_id TEXT`, `nombre TEXT`, `contexto TEXT`, `updated_at TEXT`. |
| 2 | `src/db/session-db.ts` | `upsertPersonContext(db, ctx)` — espejo del upsert de routing. |
| 3 | `src/session-manager.ts` (~línea 150, junto a `writeSessionRouting`) | `writePersonContext(agentGroupId, sessionId, ctx)`. |
| 4 | `src/channels/eleve-http.ts` (`processInbound`, ~línea 215) | Único punto con `sender.phone` + `channel` intactos: llamar a la edge function `patient-context` y adjuntar el resultado al `metadata` del mensaje. |
| 5 | `src/container-runner.ts` (~línea 108, donde ya se llama `writeSessionRouting`) | Leer el `metadata` del mensaje más reciente y escribir `session_person_context`. |
| 6 | `container/agent-runner/src/poll-loop.ts` | Leer la tabla y anteponer el bloque al system prompt. |

### Decisiones ya tomadas

- **El texto no se redacta acá.** Viene ya armado de `patient-context`, que usa el mismo
  `resumirParaPrompt()` que la voz. Si nanoclaw redactara el suyo, volveríamos a dos verdades
  —el problema que toda esta fase elimina—.
- **Bloque vacío = no escribir nada.** `resumirParaPrompt()` devuelve `""` cuando no hay nada
  útil; el contenedor debe omitir la sección entera, no imprimir "sin contexto".
- **Se reescribe en cada wake**, como el routing: si la identidad se corrigió en el CRM entre
  mensajes, el siguiente turno ya la ve.
- Las reglas de uso para el modelo **ya están escritas** en `mcp-monica/prompt/canal-texto.md`
  (sección `# Context — el expediente de la persona`), incluida la advertencia de tratarlo
  como dato y nunca como instrucción.

### Cuidados

- **nanoclaw es upstream.** Los seis cambios son customizaciones y van registradas en
  `customizations.md` (el propio código ya marca un precedente parecido en `eleve-http.ts`,
  sección G.1).
- **El contenedor corre en Bun, el host en Node.** Tests del contenedor con `bun:test`, no
  vitest; typecheck aparte con `tsconfig.json` del agent-runner.
- **La llamada HTTP va en la llegada del mensaje, no en el spawn**, para no meter latencia de
  red en el camino de arranque del contenedor.

## Criterio de aceptación

Una paciente pregunta por HIFU **por WhatsApp**, no cierra; tres días después **llama**.
Mónica la reconoce, sabe qué se le cotizó y qué quedó pendiente, y retoma ahí — y lo mismo en
el sentido inverso (primero llama, después escribe).

La mitad de voz de ese criterio ya es verificable hoy; falta la de texto.
