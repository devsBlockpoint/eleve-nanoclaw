# ÉLEVÉ × nanoclaw × mcp-monica — Diseño

**Fecha:** 2026-04-28
**Estado:** propuesta — pendiente de aprobación
**Autor:** brainstorming Claude Code + emorfin

## 1. Objetivo

Migrar el agente conversacional "Mónica" de ÉLEVÉ desde la edge function monolítica `chat-assistant` (Supabase) hacia una arquitectura desacoplada en dos contenedores:

1. **nanoclaw** — agent runtime basado en Claude Agent SDK, fork de [`qwibitai/nanoclaw`](https://github.com/qwibitai/nanoclaw), que aloja al agente "Mónica" con su CLAUDE.md y su MCP.
2. **mcp-monica** — MCP server propio que expone como tools del LLM las edge functions de negocio que ya existen en Supabase (citas, pacientes, pagos, promociones, etc.).

Ambos contenedores se levantan vía Docker. La configuración (system prompt, secrets, endpoints) se inyecta 100% por env vars — sin recompilar la imagen para cambiar comportamiento.

## 2. Contexto

ÉLEVÉ es una clínica estética con app web (Lovable + Supabase) que atiende pacientes por WhatsApp. El stack actual:

- **Backend**: Supabase (Postgres + Edge Functions + pg_cron)
- **Inbound WhatsApp**: edge function `whatsapp-webhook` (Meta WhatsApp Cloud API)
- **Outbound WhatsApp**: edge function `n8n-whatsapp-agent-response` (vía n8n hacia Meta)
- **Agente actual**: edge function `chat-assistant` que carga `manifest.json` y orquesta llamadas a tools
- **Tools registry**: `mcp-monica/mcp/` — 12 tools documentadas en markdown, generador `_generator.ts` que produce `manifest.json` y valida que cada `edge_function` exista en `supabase/functions/`

Lo que cambia:
- `chat-assistant` ya no orquesta. Su rol pasa a nanoclaw.
- `manifest.json` ya no se consume desde Supabase. Lo consume el nuevo MCP server `mcp-monica`, que nanoclaw conecta como tool provider.
- `whatsapp-webhook` deja de llamar a `chat-assistant`. Empieza a llamar a `nanoclaw` por HTTP.
- `n8n-whatsapp-agent-response` queda igual; ahora lo invoca nanoclaw para enviar la respuesta.

Lo que NO cambia:
- Las edge functions de negocio (`book-appointment`, `check-availability`, `cancel-appointment`, etc.) — mcp-monica las llama tal cual están.
- El modelo de datos en Supabase.
- La pipeline WhatsApp del lado Meta.

## 3. Arquitectura

```
WhatsApp (Meta)
  │
  ▼  webhook entrante
ÉLEVÉ Supabase: whatsapp-webhook
  │
  ▼  HTTP POST + bearer
nanoclaw container (Bun + Claude Agent SDK)
  │  ├─ eleve-http channel adapter (NUEVO)
  │  ├─ inbound.db / outbound.db (SQLite por sesión)
  │  └─ groups/monica/ (agent group único)
  │
  ▼  MCP over HTTP
mcp-monica container (Node + MCP SDK)
  │  └─ proxy a Supabase Edge Functions
  │
  ▼  HTTPS + service_role
Supabase Edge Functions (book-appointment, check-availability, ...)

Y de regreso:
nanoclaw → POST → ÉLEVÉ: n8n-whatsapp-agent-response → WhatsApp (Meta) → usuario
```

**Principio rector — env-driven setup:** levantar el container con un set de variables debe bastar para tener el agente operativo. Sin tocar archivos dentro de la imagen.

## 4. Bridge ÉLEVÉ ↔ nanoclaw

### 4.1 Inbound: ÉLEVÉ → nanoclaw

**Endpoint nuevo en nanoclaw:** `POST /messages`

**Auth:** `Authorization: Bearer <AGENT_INBOUND_TOKEN>`. Reusamos el bearer que ÉLEVÉ ya envía hoy hacia n8n / chat-assistant; en nanoclaw lo recibimos con nombre neutral `AGENT_INBOUND_TOKEN`. Si el header de origen tiene un nombre diferente (ej. `X-N8N-API-KEY`), nanoclaw acepta también ese alias para que del lado ÉLEVÉ no haga falta cambio.

**Request body:**
```json
{
  "conversation_id": "uuid de whatsapp_conversations en ÉLEVÉ",
  "message": "texto del usuario",
  "sender": {
    "phone": "5215512345678",
    "name": "opcional"
  },
  "metadata": { "any": "extra context" }
}
```

**Behavior:**
1. nanoclaw valida bearer.
2. Resuelve `agent_group = "monica"` (único por ahora; reservado para multi-agent en el futuro).
3. Mapea `conversation_id` → `session_id` de nanoclaw (relación 1:1; persistente).
4. Escribe el mensaje en `inbound.db` de la sesión y despierta el container del agent group.
5. Responde `202 Accepted` inmediatamente (fire-and-forget; el procesamiento es async).

### 4.2 Outbound: nanoclaw → ÉLEVÉ

Cuando el agente termina de procesar, nanoclaw lee `outbound.db` y POSTea a la edge function existente `n8n-whatsapp-agent-response`:

```
POST {ELEVE_OUTBOUND_URL}
Authorization: Bearer {ELEVE_OUTBOUND_TOKEN}
Content-Type: application/json

{
  "conversation_id": "...",
  "message": "respuesta del agente",
  "action": "escalate | transfer | close | schedule_followup",  // opcional
  "metadata": { ... }
}
```

Las acciones (`action`) ya están soportadas por `n8n-whatsapp-agent-response` según `mcp/_pipeline.md`.

### 4.3 Identificación de sesión

`conversation_id` (de la tabla `whatsapp_conversations` en ÉLEVÉ) es la clave estable. Una conversación = una sesión de nanoclaw = un par `inbound.db`/`outbound.db`. Esto da memoria persistente automática por contacto.

### 4.4 Async / timeouts

- ÉLEVÉ no espera respuesta sincrónica de nanoclaw. El webhook responde rápido y el agente responde cuando esté listo (5–30s típico, más si llama varias tools).
- El POST de salida a `n8n-whatsapp-agent-response` tiene retry con backoff exponencial (3 intentos, base 2s). Si falla los 3, se persiste en una cola local y se reintenta en el sweep.

## 5. mcp-monica — diseño

### 5.1 Stack

- **Lenguaje:** TypeScript
- **Runtime:** Node 20+ (`tsx` para ejecución TS sin compilar)
- **Test runner:** Vitest
- **MCP SDK:** `@modelcontextprotocol/sdk`
- **Transport:** HTTP/SSE (no stdio) — mcp-monica corre como container independiente, nanoclaw lo conecta por red.

Justificación: el registry actual (`_generator.ts`, frontmatter parser) ya está en TS; mcp-monica es thin proxy a Supabase, no necesita Python; Node está disponible en cualquier host del equipo sin instalar runtimes adicionales. **Nota sobre Bun:** nanoclaw upstream usa Bun dentro de su container, pero su host corre Node + pnpm. Para mcp-monica (container independiente) no había razón fuerte para forzar Bun; pivotamos a Node por simplicidad operativa. La elección está documentada en `docs/superpowers/plans/2026-04-28-mcp-monica-server.md`.

### 5.2 Estructura

```
mcp-monica/
├── mcp/                          # ya existe — registry source of truth
│   ├── tools/*.md
│   ├── _generator.ts
│   ├── _pipeline.md
│   ├── README.md
│   └── manifest.json             # generado en build
└── src/
    ├── index.ts                  # entry: arranca MCP server
    ├── server.ts                 # @modelcontextprotocol/sdk wiring
    ├── tools-loader.ts           # lee manifest.json → registra tools en MCP
    ├── supabase-client.ts        # POST a Supabase Edge Functions
    ├── auth.ts                   # service_role injection
    └── errors.ts                 # mapping HTTP → MCP errors
```

### 5.3 Boot

1. Lee env (`SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `MCP_PORT`).
2. Carga `manifest.json` (bundled en la imagen).
3. Por cada tool del manifest, registra un handler en el MCP server:
   ```
   handler(input) →
     POST {SUPABASE_URL}/functions/v1/{tool.edge_function}
     Authorization: Bearer {SUPABASE_SERVICE_ROLE_KEY}
     body: input  // ya validado por input_schema del manifest
   ```
4. Levanta servidor HTTP en `MCP_PORT`.

### 5.4 Manifest = bundled, no fetched

El `manifest.json` se commitea al repo y se copia al container en build. Cambiar tools → rebuild + redeploy. Inmutabilidad de la imagen sobre flexibilidad de hot-reload (que se puede agregar después si hace falta).

### 5.5 Mapeo de errores

| Edge function HTTP | MCP behavior |
|---|---|
| 2xx | Tool result OK, payload pasa al LLM |
| 4xx | MCP error con `code` y `message` del payload (visible al LLM para que reaccione, ej. "slot ya no disponible" → reintente con otro slot) |
| 5xx | MCP error genérico ("Internal error") sin filtrar detalles. Loguear stack del lado mcp-monica. |
| Timeout (>10s) | MCP error timeout |

### 5.6 Tools `pending`

Por defecto `_generator.ts` excluye tools con `implementation_status: pending` del manifest. mcp-monica solo expone tools `implemented`. Las pending quedan documentadas pero no llamables. Para incluir pending en dev, build con `npm run mcp:build -- --include-pending`.

## 6. System prompt — 3 fuentes configurables

```
AGENT_SYSTEM_PROMPT_SOURCE=env|file|url   # default: env
```

**`env` (default):** `AGENT_SYSTEM_PROMPT="texto completo"`. Práctico para prompts cortos.

**`file`:** `AGENT_SYSTEM_PROMPT_PATH=/data/system-prompt.md`. Mount de volumen en Easypanel apuntando al archivo. Práctico para prompts grandes editables fuera de envs.

**`url`:** `AGENT_SYSTEM_PROMPT_URL=https://...` con autenticación opcional `AGENT_SYSTEM_PROMPT_URL_AUTH=Bearer <token>`.

### 6.1 Google Drive como fuente

Soportamos Drive como caso particular de `url`. Para arrancar:

- Doc compartido en Drive como "anyone with the link can view".
- URL de export plain text: `https://docs.google.com/document/d/<DOC_ID>/export?format=txt`
- nanoclaw hace `fetch()` al boot.

Caveat: el prompt queda públicamente accesible si filtran el link. Si el prompt contiene info sensible de negocio, migrar a Service Account (out of scope para v1; se documenta como upgrade path).

### 6.2 Cache + fallback

```
boot:
  resolve source
  if source == url:
    try fetch (timeout 5s)
    on success: persistir a /data/system-prompt.cache.md, usar contenido
    on failure: leer /data/system-prompt.cache.md, log warning, usar cache
    on no cache disponible: error fatal, container no arranca
  pass content al Agent SDK como system prompt
```

### 6.3 Hot-reload (opcional)

`AGENT_SYSTEM_PROMPT_RELOAD_INTERVAL=0` por default (off). Si > 0, nanoclaw poll-ea el URL cada N segundos; si cambió, actualiza el prompt para sesiones nuevas (las en curso terminan con la versión vieja, sin restart).

## 7. Deployment

### 7.1 Imágenes

Dos imágenes Docker, cada una con su `Dockerfile`:
- `nanoclaw:latest`
- `mcp-monica:latest`

### 7.2 docker-compose.yml (raíz del repo)

Pensado para dev local y como referencia para Easypanel:

```yaml
services:
  nanoclaw:
    build: ./nanoclaw
    ports: ["3001:3001"]
    env_file: .env.nanoclaw
    volumes: ["nanoclaw-data:/data"]
    depends_on: [mcp-monica]
    networks: [eleve-net]

  mcp-monica:
    build: ./mcp-monica
    env_file: .env.mcp-monica
    networks: [eleve-net]

networks:
  eleve-net:
volumes:
  nanoclaw-data:
```

mcp-monica NO expone puerto al host — solo accesible desde la red interna por nanoclaw.

### 7.3 Env vars (resumen)

**nanoclaw:**
- `AGENT_INBOUND_TOKEN` — bearer que ÉLEVÉ envía
- `ANTHROPIC_API_KEY` (o `ANTHROPIC_AUTH_TOKEN` + `ANTHROPIC_BASE_URL` si vía proxy)
- `AGENT_GROUP=monica`
- `AGENT_SYSTEM_PROMPT_SOURCE=env|file|url`
- `AGENT_SYSTEM_PROMPT` / `AGENT_SYSTEM_PROMPT_PATH` / `AGENT_SYSTEM_PROMPT_URL` (según source)
- `AGENT_SYSTEM_PROMPT_URL_AUTH` (opcional)
- `AGENT_SYSTEM_PROMPT_RELOAD_INTERVAL=0`
- `MCP_MONICA_URL=http://mcp-monica:3000`
- `ELEVE_OUTBOUND_URL=https://<project>.supabase.co/functions/v1/n8n-whatsapp-agent-response`
- `ELEVE_OUTBOUND_TOKEN=<service_role_o_equivalente>`
- `PORT=3001`

**mcp-monica:**
- `SUPABASE_URL=https://<project>.supabase.co`
- `SUPABASE_SERVICE_ROLE_KEY=...`
- `MCP_PORT=3000`

### 7.4 Easypanel

- Stack docker-compose nativo en Easypanel.
- Volumen persistente `/data` (DBs SQLite + cache de prompt).
- Env vars en el panel UI; `.env.example` en el repo como referencia.
- Healthcheck: `GET /health` en nanoclaw.

### 7.5 Persistencia

- `/data/inbound.db`, `/data/outbound.db`, `/data/central.db` → volumen persistente.
- `/data/system-prompt.cache.md` → mismo volumen.
- Reiniciar el container NO debe perder historial de conversaciones.

## 8. Estructura de documentación (raíz del monorepo)

```
eleve-nanoclaw/
├── CLAUDE.md                     # ÍNDICE GLOBAL — entry para Claude Code en este monorepo
├── README.md                     # qué es el monorepo, cómo arrancar
│
├── docs/
│   ├── domain/                   # modelo del dominio (DDD)
│   │   ├── ubiquitous-language.md
│   │   ├── bounded-contexts.md
│   │   ├── context-map.md
│   │   └── domain-events.md
│   ├── architecture/
│   │   ├── overview.md
│   │   ├── data-flow.md
│   │   ├── deployment.md
│   │   └── decisions/
│   │       └── 0001-claude-agent-sdk.md
│   └── superpowers/specs/
│       └── 2026-04-28-eleve-nanoclaw-monica-design.md  # este doc
│
├── docker-compose.yml
├── .env.example
│
├── nanoclaw/
│   ├── CLAUDE.md                 # índice DEV (no es el system prompt runtime)
│   ├── README.md
│   ├── docs/
│   │   ├── bridge-eleve.md       # contrato HTTP con ÉLEVÉ
│   │   ├── agent-groups.md
│   │   ├── runtime-config.md
│   │   └── customizations.md     # qué se modificó vs upstream qwibitai/nanoclaw
│   └── ...código...
│
└── mcp-monica/
    ├── CLAUDE.md                 # índice DEV
    ├── README.md
    ├── docs/
    │   ├── tool-registry.md
    │   ├── edge-functions-map.md
    │   ├── auth.md
    │   └── new-tool.md
    ├── mcp/                      # ya existente — registry
    │   ├── README.md
    │   ├── _pipeline.md
    │   ├── _generator.ts
    │   ├── manifest.json
    │   └── tools/*.md
    └── src/                      # MCP server (a implementar)
```

**Idioma:** español para docs de dominio (mantiene la ubiquitous language del negocio: cita, esteticista, paciente, agent_mode, "Pendiente de Anticipo"). Términos técnicos en inglés cuando es natural (bounded context, edge function, etc.).

## 9. Out of scope (v1)

Cosas conocidas pero diferidas:

- Múltiples agent groups (más allá de "monica").
- Service Account de Drive para system prompt privado.
- Hot-reload de manifest.json sin redeploy.
- Migración de prompts/skills existentes desde otros agentes.
- Observabilidad (logs, métricas, traces) — se cubre en spec separada.
- Tests end-to-end del bridge — plan separado.
- Tenant multi-cliente (clonado del setup para otro cliente además de ÉLEVÉ) — la arquitectura lo permite por env, pero el playbook del clonado es spec aparte.

## 10. Decisiones tomadas (resumen)

| Decisión | Elección | Por qué |
|---|---|---|
| Stack del agente | nanoclaw upstream (TS/Bun + Claude Agent SDK) | Ya está construido, hace exactamente lo que necesitamos |
| Stack del MCP | TS + Node 20 + `@modelcontextprotocol/sdk` | Node ya disponible en el host; sin overhead de instalar Bun para un thin proxy. Bun queda solo dentro del container nanoclaw upstream (donde aporta `bun:sqlite` nativo), no en mcp-monica. |
| Transport MCP | HTTP/SSE | Containers independientes, env-driven, sin acoplar lifecycle |
| Manifest | Bundled en imagen | Inmutabilidad de la imagen sobre hot-reload |
| Bridge inbound | `POST /messages` con bearer | Patrón fire-and-forget, alineado con webhook async |
| Bridge outbound | POST a `n8n-whatsapp-agent-response` | Reusa pipeline existente sin tocar Supabase |
| Sesión | `whatsapp_conversation_id` 1:1 con sesión nanoclaw | Memoria persistente automática por contacto |
| System prompt | 3 fuentes (env/file/url) | Flexibilidad sin acoplar a una solución |
| Drive como prompt | Link público + cache + fallback | Setup mínimo, upgrade path a Service Account documentado |
| Hosting | Easypanel (docker-compose nativo) | Confirmado por el usuario |
| Sin agent group `whatsapp` upstream | nanoclaw NO conecta a Meta directo | ÉLEVÉ ya tiene la pipeline; el agente no debe duplicar |
