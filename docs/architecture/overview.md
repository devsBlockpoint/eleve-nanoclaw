# Architecture Overview

Vista de alto nivel de la arquitectura de la migración Mónica. Para el detalle exhaustivo de decisiones, ver el [spec original](../superpowers/specs/2026-04-28-eleve-nanoclaw-monica-design.md).

## Vista de componentes

```
WhatsApp (Meta)
  │
  ▼  webhook entrante
ÉLEVÉ Supabase: whatsapp-webhook
  │
  ▼  HTTP POST + bearer (AGENT_INBOUND_TOKEN)
nanoclaw container (Bun + Claude Agent SDK)
  │  ├─ eleve-http channel adapter (custom)
  │  ├─ inbound.db / outbound.db (SQLite por sesión)
  │  └─ groups/monica/ (agent group único en v1)
  │
  ▼  MCP over HTTP
mcp-monica container (Node + MCP SDK)
  │  └─ proxy a Supabase Edge Functions
  │
  ▼  HTTPS + service_role
Supabase Edge Functions (book-appointment, check-availability, ...)

Y de regreso:
nanoclaw → POST → n8n-whatsapp-agent-response → WhatsApp (Meta) → usuario
```

## Principios rectores

1. **Env-driven setup**: levantar el container con un set de variables debe bastar para tener el agente operativo. Sin recompilar imagen para cambiar comportamiento.
2. **Separación de responsabilidades**: el agente (nanoclaw) no conoce el modelo de negocio; lo descubre vía MCP. La lógica de negocio vive en Supabase. mcp-monica es thin proxy.
3. **Sesiones persistentes por conversación**: cada `whatsapp_conversation_id` de ÉLEVÉ = una sesión de nanoclaw. Memoria automática entre mensajes.
4. **Async fire-and-forget**: ÉLEVÉ no espera respuesta sincrónica de nanoclaw. nanoclaw responde cuando puede.

## Stack por componente

| Componente | Lenguaje/Runtime | Justificación |
|---|---|---|
| nanoclaw | TypeScript + Bun | Stack nativo del fork upstream qwibitai/nanoclaw + Claude Agent SDK |
| mcp-monica | TypeScript + Node 20 | Sin requisitos especiales (thin proxy a HTTP); el host del equipo ya corre Node. Bun no aporta valor en este componente. Nota: nanoclaw upstream usa Bun **dentro** de su container, no en el host. |
| Supabase Edge Functions | TypeScript + Deno | Preexistente, no cambia |
| Frontend ÉLEVÉ | TypeScript + React (Lovable) | Preexistente, no cambia |

## System prompt: 3 fuentes configurables

```
AGENT_SYSTEM_PROMPT_SOURCE=env|file|url   # default: env
```

- `env`: `AGENT_SYSTEM_PROMPT="texto completo"`. Para prompts cortos.
- `file`: `AGENT_SYSTEM_PROMPT_PATH=/data/system-prompt.md`. Para mount de volumen.
- `url`: `AGENT_SYSTEM_PROMPT_URL=https://...` con auth opcional. Soporta Google Drive (link público + cache + fallback).

Detalle completo en el spec, sección 6.

## Deployment

- **Dev local:** `docker compose up -d` desde la raíz del monorepo.
- **Producción:** Easypanel (acepta docker-compose nativo). Cada container con su volumen persistente para `/data`.
- **Imágenes:** `nanoclaw:latest` y `mcp-monica:latest`, builds independientes.

## Qué NO hace nanoclaw

- No habla con WhatsApp directamente. ÉLEVÉ es el dueño del canal.
- No conoce el modelo de citas/pacientes/servicios. Lo descubre vía MCP.
- No persiste datos de negocio. Solo memoria conversacional (SQLite local).

## Qué NO hace mcp-monica

- No tiene lógica de negocio. Es thin proxy a Supabase Edge Functions.
- No habla con WhatsApp ni con la base de datos directamente.
- No genera contenido. Solo expone tools.
