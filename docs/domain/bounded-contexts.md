# Bounded Contexts — ÉLEVÉ

Identificación de los **bounded contexts** del sistema. Cada context tiene su propio modelo, vocabulario y responsabilidad. La frontera entre contexts es donde ocurren las traducciones de modelo (anti-corruption layers).

## 1. Conversational Agent (nanoclaw)

**Responsabilidad:** recibir mensajes del paciente, mantener memoria de conversación, generar respuestas con Claude, decidir cuándo invocar tools, devolver respuesta o señales de escalamiento.

**Modelo central:** `Session` (1:1 con `whatsapp_conversation`), `Message` (turno), `AgentGroup` (configuración de agente).

**Tecnología:** TypeScript + Bun + Claude Agent SDK. Persistencia: SQLite (`inbound.db`, `outbound.db` por sesión + `central.db`).

**No es responsable de:**
- Conocer el modelo de negocio de citas, pacientes, pagos. Eso lo descubre vía MCP tools.
- Hablar con WhatsApp. Eso lo hace ÉLEVÉ.

## 2. Tool Gateway (mcp-monica)

**Responsabilidad:** exponer al LLM, vía protocolo MCP, las operaciones de negocio que vive en Supabase Edge Functions. Validar inputs (input_schema), llamar la edge function correspondiente, mapear errores HTTP a errores MCP.

**Modelo central:** `Tool` (manifest entry), `EdgeFunctionEndpoint` (URL + auth), `MCPRequest`/`MCPResponse`.

**Tecnología:** TypeScript + Node 20 + `@modelcontextprotocol/sdk`. Sin estado. (Runtime Node, no Bun: ver decisión documentada en `docs/superpowers/plans/2026-04-28-mcp-monica-server.md`.)

**No es responsable de:**
- Lógica de negocio. Es thin proxy. La lógica vive en las edge functions.
- Decidir cuándo invocar una tool. Eso lo decide el LLM en nanoclaw.

## 3. ÉLEVÉ Backend (Supabase)

**Responsabilidad:** verdad del dominio del negocio. Tablas (`pacientes`, `citas`, `servicios`, `whatsapp_conversations`, etc.), reglas de negocio, edge functions de tools, pipelines de WhatsApp (inbound/outbound), cron jobs (auto-return-to-bot, send-reminders).

**Modelo central:** todo el dominio: `Paciente`, `Cita`, `Servicio`, `Esteticista`, `Promocion`, `WhatsAppConversation`, `EscalationQueue`, etc.

**Tecnología:** Supabase (Postgres + Edge Functions Deno + pg_cron) + n8n para orquestación de mensajería. Frontend Lovable (out of scope para esta migración).

**No cambia con esta migración**, salvo:
- `chat-assistant` deja de ser el orquestador (queda deprecated o se elimina cuando nanoclaw esté estable).
- `whatsapp-webhook` apunta a nanoclaw en lugar de a `chat-assistant`.

## 4. WhatsApp Channel (Meta)

**Responsabilidad:** entrega y recepción real de mensajes con el paciente. Externo, no bajo control nuestro más allá del registro de la WhatsApp Business API.

**Tecnología:** Meta WhatsApp Cloud API.

## Relación con el frontend Lovable

El frontend de ÉLEVÉ (CRM web, hecho con Lovable) consume las mismas tablas de Supabase pero **no participa en el flujo del agente**. Es un context aparte centrado en gestión humana del negocio. Para esta migración, el único punto de contacto es la posibilidad de que un operador humano "tome control" de una conversación (`agent_mode = human`).
