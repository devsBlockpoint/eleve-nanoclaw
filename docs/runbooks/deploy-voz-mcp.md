# Runbook — Deploy del canal de voz: exponer `mcp-monica` + registrar MCP en ElevenLabs

> **Qué desbloquea:** que el agente de voz **Monica** (ElevenLabs) pueda usar las **tools**
> (`agendar_cita`, `buscar_disponibilidad`, `obtener_servicios`, `escalate_to_human`, …) llamando a
> `mcp-monica`. Sin esto, en una llamada Mónica **habla y sabe del catálogo (KB) pero no agenda ni
> cotiza en vivo**.
>
> **Idea central:** hoy `mcp-monica` corre **interno** (no expuesto). ElevenLabs es un SaaS externo →
> hay que **exponer `/mcp` público con HTTPS + token**. El token ya está soportado en el código
> (`src/auth.ts`, opt-in vía `MCP_AUTH_TOKEN`) y nanoclaw ya lo manda (header en
> `groups/monica/container.json`).
>
> **Fecha:** 2026-07-29 · **Estado:** listo para ejecutar en el deploy.

---

## Prerequisitos (ya hechos)

- ✅ `mcp-monica`: guard de token en `/mcp` (opt-in, `MCP_AUTH_TOKEN`) — 25/25 tests verde.
- ✅ `nanoclaw`: `groups/monica/container.json` manda `Authorization: Bearer ${MCP_AUTH_TOKEN}` al MCP.
- ✅ ElevenLabs: agente **Monica** (`agent_4101kynwpz59fesa9rjf5zqg29w4`) con prompt de voz + KB + voz Elena + Haiku.
- ✅ Número `+52 668 268 0353` importado y asignado a Monica (falta activarlo en Telnyx).

---

## 1. Generar el token del MCP

Generá un secreto fuerte (una sola vez) y guardalo en el password manager:

```bash
openssl rand -hex 32     # ej: 9f2c... (64 chars)
```

Este valor va **idéntico** en tres lugares: env de `mcp-monica`, env de `nanoclaw-host`, y el
`secret_token` del MCP server en ElevenLabs.

## 2. Easypanel — service `mcp-monica`

1. **Env var nueva:**
   ```
   MCP_AUTH_TOKEN=<el token del paso 1>
   ```
   (Ya tiene `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `MCP_PORT=3000`.)
2. **Exponer público:** agregar un **dominio Easypanel + TLS automático** al service (hoy está solo
   interno). Anotá la URL pública, ej. `https://mcp-monica.<tu-easypanel>.app`.
   - `/health` y `/mcp` quedan públicos; `/mcp` está protegido por el token, `/health` es inocuo.
3. Redeploy. Verificá:
   ```bash
   curl -s https://mcp-monica.<...>.app/health          # {"status":"ok","tools":12}
   curl -s -o /dev/null -w '%{http_code}\n' https://mcp-monica.<...>.app/mcp   # 401 (sin token = rechaza ✓)
   ```

## 3. Easypanel — service `nanoclaw-host`

1. **Env var nueva:** `MCP_AUTH_TOKEN=<el mismo token>` (nanoclaw la inyecta en el header del MCP).
2. **Corregir `MCP_MONICA_URL`** → debe apuntar a la ruta **`/mcp`** por la red interna (NO `/mcp/sse`,
   que da 404 con el server actual de StreamableHTTP):
   ```
   MCP_MONICA_URL=http://mcp-monica:3000/mcp
   ```
   nanoclaw sigue usando la **interna** (rápida); solo ElevenLabs usa la pública.
3. **Propagar el header a la config del grupo:** el cambio de `groups/monica/container.json` (header
   `Authorization`) debe estar en el `nanoclaw-host` deployado. Según cómo despleguen:
   - Si la imagen `nanoclaw-host:latest` **bakea** `groups/` → rebuild + push de la imagen.
   - Si el grupo vive en el **volumen persistente** → editar ahí el `container.json` y reiniciar.
4. Redeploy. Verificá que el texto (WhatsApp) sigue funcionando (smoke §8 del deploy general) — si el
   token quedó mal, verías 401 en los logs de `mcp-monica` desde nanoclaw.

## 4. Registrar el MCP server en ElevenLabs (lo ejecuto yo)

Cuando la URL pública de `mcp-monica` esté viva y el `/mcp` devuelva 401 sin token, **avisame y lo
registro por API** (creo el MCP server + lo adjunto al agente Monica), iterando si el schema pide
algún campo. Referencia de lo que se crea:

```
POST https://api.elevenlabs.io/v1/convai/mcp-servers
{
  "config": {
    "name": "mcp-monica",
    "url": "https://mcp-monica.<...>.app/mcp",
    "transport": "STREAMABLE_HTTP",
    "secret_token": { "secret_id": "<workspace secret con el token>" },
    "approval_policy": "require_approval_per_tool",
    "response_timeout_secs": 30
  }
}
```
- Guardo el token como **workspace secret** de ElevenLabs (no en texto plano).
- **`require_approval_per_tool`**: auto-aprobar lecturas (`buscar_disponibilidad`, `obtener_servicios`,
  `get_current_promotions`), y confirmar/verbalizar las que mutan (`agendar_cita`,
  `capture_lead_from_chat`, `escalate_to_human`).
- Luego adjunto el MCP al agente Monica (`prompt.mcp_server_ids`) sin pisar su modelo (Haiku), voz
  (Elena) ni la KB.

## 5. Defensa en profundidad (opcional, recomendado)

- **IP allowlist**: permitir en el ingress solo las IPs de egreso de ElevenLabs + la red interna.
- **HTTPS obligatorio** (ya con el dominio Easypanel).

## 6. Prueba

- **Sin el número aún:** desde el dashboard de ElevenLabs se puede hacer una **llamada de prueba web**
  al agente Monica para validar que las tools responden (agenda, disponibilidad) antes de tener el
  teléfono.
- **Con el número activo + SIP + desvío:** llamada real → Mónica agenda de verdad.

---

## Checklist de go-live (voz completa)

- [ ] Token generado y guardado.
- [ ] `mcp-monica`: `MCP_AUTH_TOKEN` seteado + dominio público + TLS; `/health` 200, `/mcp` 401.
- [ ] `nanoclaw-host`: `MCP_AUTH_TOKEN` seteado; `MCP_MONICA_URL=…/mcp`; header propagado; texto OK.
- [ ] MCP server registrado en ElevenLabs + adjunto a Monica (lo hago yo).
- [ ] Prueba web del agente: agenda/cotiza OK.
- [ ] (Telefonía) número Telnyx activo + SIP Connection + desvío Telcel → prueba de llamada real.

## Notas / gotchas

- El guard es **opt-in**: sin `MCP_AUTH_TOKEN` el `/mcp` queda **abierto** — por eso el token es
  obligatorio antes de exponerlo público.
- El header en `container.json` usa el placeholder `${MCP_AUTH_TOKEN}`; si la env no está, nanoclaw
  manda el literal sin expandir, pero como el guard del server estaría off, no rompe (dev). En prod,
  ambos deben tener el token.
- `MCP_MONICA_URL` debe terminar en **`/mcp`** (StreamableHTTP), no `/mcp/sse`.
