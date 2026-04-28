# Context Map — ÉLEVÉ

Cómo se relacionan los bounded contexts. Notación inspirada en DDD context mapping (upstream/downstream, Anticorruption Layer, Conformist, Open Host Service, etc.).

## Diagrama

```
┌──────────────────────────┐
│  WhatsApp (Meta)         │   externo, fuera de control
└────────┬─────────────────┘
         │ HTTPS bidireccional
         ▼
┌──────────────────────────┐                  ┌──────────────────────────┐
│  ÉLEVÉ Backend           │ ◄──── HTTPS ──── │  Tool Gateway            │
│  (Supabase + n8n)        │   service_role   │  (mcp-monica)            │
│  ─────────────────────── │                  │  ─────────────────────── │
│  whatsapp-webhook        │                  │  proxy MCP → edge        │
│  n8n-whatsapp-agent-resp │                  │  functions               │
│  edge functions de tools │                  │                          │
│  Postgres (citas, etc.)  │                  │                          │
└────────┬─────────────────┘                  └──────────────▲───────────┘
         │ POST /messages (bearer)                            │ MCP HTTP
         │ POST n8n-whatsapp-agent-response (bearer)          │
         ▼                                                    │
┌─────────────────────────────────────────────────────────────┘
│  Conversational Agent (nanoclaw)
│  ─────────────────────────────────────────
│  eleve-http channel adapter (custom)
│  Claude Agent SDK + system prompt configurable
│  inbound.db / outbound.db por sesión
└──────────────────────────────────────────────
```

## Relaciones por par

### nanoclaw ↔ ÉLEVÉ Backend

- **Patrón:** Customer/Supplier — ÉLEVÉ es el customer (le pide trabajo a nanoclaw); nanoclaw es el supplier.
- **Inbound (ÉLEVÉ → nanoclaw):** `whatsapp-webhook` POSTea a `nanoclaw:/messages` con bearer. Contrato definido en el spec.
- **Outbound (nanoclaw → ÉLEVÉ):** nanoclaw POSTea a `n8n-whatsapp-agent-response` con bearer. Contrato preexistente, nanoclaw es **Conformist** (acepta el shape que ÉLEVÉ ya espera, sin negociar).
- **Anti-corruption layer:** está dentro del channel adapter `eleve-http` de nanoclaw. Traduce el payload de ÉLEVÉ al modelo interno de Session/Message.

### nanoclaw ↔ mcp-monica

- **Patrón:** Open Host Service — mcp-monica publica un protocolo estándar (MCP); cualquier cliente compatible (nanoclaw u otros agentes futuros) lo consume.
- **Transporte:** MCP over HTTP/SSE.
- **Descubrimiento:** nanoclaw conecta al boot, descubre tools dinámicamente (lista publicada por el server).
- **Versionado:** el server expone `version: "1"` en el manifest. Cambios breaking del schema requieren nueva major version.

### mcp-monica ↔ ÉLEVÉ Backend

- **Patrón:** Conformist — mcp-monica acepta el shape de respuesta que cada edge function ya devuelve. No negocia contratos nuevos.
- **Transporte:** HTTPS, bearer `service_role`.
- **Anti-corruption layer mínima:** mapeo de errores HTTP → MCP errors es la única traducción.

### ÉLEVÉ Backend ↔ WhatsApp (Meta)

- **Patrón:** Conformist puro — Meta dicta el contrato (Webhook payload, Send API). ÉLEVÉ se adapta.
- Out of scope para esta migración.

## Direcciones de cambio

- **El agente cambia más rápido que el backend de negocio.** El system prompt de Mónica, las tools nuevas, el comportamiento del LLM evolucionan continuamente. El modelo de citas/pacientes en Supabase cambia más despacio.
- **Por eso:** la frontera más cuidada es la entre el Tool Gateway y el ÉLEVÉ Backend. Mientras los contratos de las edge functions se mantengan estables, el agente puede iterar libre.
- **Riesgo:** acoplamiento accidental — el LLM "aprende" detalles del shape de respuesta de una edge function. Mitigación: documentar en cada `tools/<name>.md` el `output_schema` esperado y validarlo del lado mcp-monica antes de devolver al LLM.
