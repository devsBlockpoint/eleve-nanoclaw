# Deployment Runbook — Easypanel (Path B)

Pasos para deployar ÉLEVÉ × nanoclaw × mcp-monica en Easypanel después del Plan 4. Sigue **Path B**: ambos services corren como containers Docker dentro de Easypanel; nanoclaw usa el daemon Docker del host de Easypanel (vía socket mount) para spawnar agent containers per-sesión.

## Pre-requisitos

- ✅ Cuenta Easypanel con un proyecto creado.
- ✅ Las 3 imágenes pushadas a GHCR **(privadas)**:
  - `ghcr.io/devsblockpoint/mcp-monica:latest`
  - `ghcr.io/devsblockpoint/nanoclaw-host:latest`
  - `ghcr.io/devsblockpoint/nanoclaw-agent:latest` (la per-sesión, ~2GB)
- ✅ Acceso al panel de Supabase de ÉLEVÉ para configurar webhooks.
- ⬜ Anthropic API key (vía OneCLI o directa).
- ⬜ PAT de GitHub con scope `read:packages` (instrucciones abajo).

## 1. Generar PAT para pull de GHCR (privadas)

Como las 3 imágenes están privadas, Easypanel necesita autenticarse con un PAT classic.

1. Ir a: <https://github.com/settings/tokens/new>
2. **Note**: `easypanel-ghcr-pull`
3. **Expiration**: `No expiration` (o 1 año, lo que prefieras — rotás cuando vence).
4. **Scope (único)**: ✅ `read:packages`
5. Generar y **copiar el token** (formato `ghp_...`).

Guardalo en un password manager. NO lo commitees a ningún repo.

## 2. Configurar GHCR como Docker registry en Easypanel

Easypanel UI → **Project settings** o **Docker registry** (depende de la versión, busca en Settings):

- **Registry URL**: `ghcr.io`
- **Username**: `devsBlockpoint`
- **Password**: el PAT generado en el paso 1.

Guardar. A partir de acá Easypanel puede pullear cualquier imagen `ghcr.io/devsblockpoint/*`.

## 3. Service: mcp-monica

Easypanel UI → **Create service** → **App** → Source: **Image**.

| Campo | Valor |
|---|---|
| Image | `ghcr.io/devsblockpoint/mcp-monica:latest` |
| Port | `3000` (interno; **no** exponer públicamente) |
| Healthcheck | HTTP `/health` cada 30s |

**Env vars:**
```
SUPABASE_URL=https://YOUR_PROJECT.supabase.co
SUPABASE_SERVICE_ROLE_KEY=<secret real, NO compartir>
MCP_PORT=3000
```

**Network**: misma red interna que `nanoclaw-host` para que se vean por nombre de servicio.

Deploy. Verificar logs: `mcp-monica: listening on http://0.0.0.0:3000`.

## 4. Service: nanoclaw-host

Easypanel UI → **Create service** → **App** → Source: **Image**.

| Campo | Valor |
|---|---|
| Image | `ghcr.io/devsblockpoint/nanoclaw-host:latest` |
| Port | `3001` → exponer público con dominio Easypanel + TLS automático |
| Healthcheck | HTTP `/health` cada 30s |

**Volumes / Mounts (CRÍTICO):**

1. **Bind mount**: `/var/run/docker.sock` (host) → `/var/run/docker.sock` (container).
   - En Easypanel: sección "Mounts" → tipo "Bind" → host path `/var/run/docker.sock` → mount path `/var/run/docker.sock`.
   - Sin esto, nanoclaw no puede spawnar agent containers y arranca con error fatal `Failed to reach container runtime`.

2. **Volume persistente**: `nanoclaw-data` → `/app/data`.
   - SQLite DBs (central + por sesión), system-prompt cache, OneCLI state.
   - Tipo "Volume" en la UI; nombre `nanoclaw-data`.

**Env vars:**

```dotenv
# ÉLEVÉ inbound
AGENT_INBOUND_TOKEN=<bearer compartido con whatsapp-webhook de Supabase>
NANOCLAW_PORT=3001

# ÉLEVÉ outbound
ELEVE_OUTBOUND_URL=https://YOUR_PROJECT.supabase.co/functions/v1/n8n-whatsapp-agent-response
ELEVE_OUTBOUND_TOKEN=<bearer outbound, e.g. service_role>

# MCP server (red interna Easypanel — usar el nombre del service)
MCP_MONICA_URL=http://mcp-monica:3000/mcp/sse

# Imagen del agent runner (apuntar a GHCR)
CONTAINER_IMAGE=ghcr.io/devsblockpoint/nanoclaw-agent:latest

# System prompt — recomendado source=url para iterar sin redeploys
AGENT_SYSTEM_PROMPT_SOURCE=url
AGENT_SYSTEM_PROMPT_URL=https://docs.google.com/document/d/<DOC_ID>/export?format=txt
# AGENT_SYSTEM_PROMPT_URL_AUTH=  # solo si el doc de Drive es privado y usás bearer

# Anthropic — vía OneCLI o directa
ANTHROPIC_API_KEY=<secret>
# o:
# ANTHROPIC_AUTH_TOKEN=<onecli token>
# ANTHROPIC_BASE_URL=<onecli proxy URL>
```

Deploy. Verificar logs:
- `Central DB ready`
- `[boot] system prompt loaded source=url chars=<n>`
- `eleve-http adapter listening port=3001`
- `Channel adapter started channel=eleve-http`

## 5. Bootstrap del agent group `monica`

Una vez `nanoclaw-host` arrancado, una sola vez:

```bash
# Desde Easypanel UI → Service → Console (o ssh al host):
docker exec -it <nanoclaw-host-container-name> sh -c "node dist/scripts/init-monica-agent.js"
# Si el script no está compilado, alternativa:
docker exec -it <nanoclaw-host-container-name> sh -c "npx tsx scripts/init-monica-agent.ts"
```

Output esperado:
```
Created agent group: ag-<id> (monica)
Group filesystem initialized.

Next steps:
  1. Ensure OneCLI agent for this group has secret mode = "all":
     onecli agents set-secret-mode --id ag-<id> --mode all
  2. Wire eleve-http channel to this agent group via SQL or admin tool.
  3. Start nanoclaw and POST to /messages with bearer AGENT_INBOUND_TOKEN.
```

Copiar el `ag-<id>` para el siguiente paso.

## 6. OneCLI: secret mode

Si OneCLI está corriendo (debería arrancar al boot del nanoclaw-host con error si no está):

```bash
docker exec -it <nanoclaw-host-container-name> onecli agents set-secret-mode --id ag-<id> --mode all
```

Sin esto el container del agent puede arrancar pero recibirá `401 Unauthorized` al llamar a Anthropic.

> **Si no estás usando OneCLI** y la key Anthropic está en env `ANTHROPIC_API_KEY` directa, este paso no aplica — pero verificá que el container heredó la env (a veces el host pasa env al container; otras hay que setearla en `container.json` del group).

## 7. Configurar webhook en ÉLEVÉ Supabase

Modificar `supabase/functions/whatsapp-webhook/` (o equivalente upstream) para que en lugar de invocar `chat-assistant`, haga:

```typescript
await fetch(`${NANOCLAW_PUBLIC_URL}/messages`, {
  method: 'POST',
  headers: {
    Authorization: `Bearer ${AGENT_INBOUND_TOKEN}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    conversation_id,        // del registro whatsapp_conversations
    message: text,          // el texto del mensaje del paciente
    sender: { phone, name },
    metadata: { ... },      // opcional
  }),
});
```

`NANOCLAW_PUBLIC_URL` es el dominio que Easypanel asignó (visible en la UI del service `nanoclaw-host`).

`AGENT_INBOUND_TOKEN` debe ser el mismo que setearon en Easypanel — guardalo en los secrets de Supabase.

## 8. Smoke real

Mandar un WhatsApp al número productivo de ÉLEVÉ (o test number con cuenta sandbox de Meta):

1. **Logs `whatsapp-webhook`**: ver "POST /messages → 202 Accepted" desde nanoclaw.
2. **Logs `nanoclaw-host`**:
   - `eleve-http: created messaging_group conversationId=<id> mgId=mg-eleve-...`
   - `eleve-http: auto-wired conversation to monica`
   - `Session created`
   - `Spawning container ... containerName=nanoclaw-v2-monica-...`
3. **Easypanel** mostrará un container ad-hoc con prefijo `nanoclaw-v2-monica-` apareciendo y desapareciendo.
4. **Logs `mcp-monica`** (si el agente decide invocar tools): POST a `/functions/v1/<edge-function>` con su input.
5. **Logs `nanoclaw-host`**: `eleve-http: deliver POST -> ELEVE_OUTBOUND_URL`.
6. **WhatsApp**: el paciente recibe la respuesta del agente.

End-to-end debería ser < 30s.

## Troubleshooting

| Síntoma | Causa probable | Fix |
|---|---|---|
| `nanoclaw-host` exit fatal `Failed to reach container runtime` | Falta el bind mount del docker.sock | Agregar mount en service settings, redeploy |
| `mcp-monica` healthcheck KO | Faltan SUPABASE_URL o SERVICE_ROLE_KEY | Setear env vars |
| Agent container exit code 125 | Imagen `nanoclaw-agent` no se pudo pullear | Verificar que GHCR registry credentials estén configuradas; `docker pull ghcr.io/devsblockpoint/nanoclaw-agent:latest` desde el host de Easypanel |
| Agent container 401 al llamar Anthropic | OneCLI selective secret mode | Correr `onecli agents set-secret-mode --id <id> --mode all` |
| `MESSAGE DROPPED — unknown sender (strict policy)` | El messaging_group quedó con policy=strict | Bug: `unknown_sender_policy=public` ya está hardcoded en el adapter; revisar que estés usando la imagen correcta del nanoclaw-host |
| `/health` devuelve 401 con HTML | Estás golpeando otro service en :3001 | Verificar el dominio; probar curl directo `curl -v https://<eleve-host>/health` |
| MCP connection refused desde nanoclaw | mcp-monica y nanoclaw no están en la misma red Easypanel | Agregar ambos al mismo network (Settings → Network) |
| `whatsapp-webhook` 504 timeout | nanoclaw está respondiendo lento o muerto | Verificar que `/messages` responda 202 inmediato (es fire-and-forget); si tarda más de 5s hay bug |

## Próximos pasos (cuando todo esté vivo)

- Activar las 10 tools `pending` en `mcp-monica/mcp/tools/` cuando sus edge functions estén listas en Supabase.
- Configurar dominio custom (no el subdominio Easypanel) si hace falta.
- Setup de monitoring + alertas (uptime de `/health`, throughput, error rate).
- Plan de rollback: tag `mcp-monica:0.0` o `nanoclaw-host:0.0` con la versión chat-assistant para volver atrás si hay regresión.
- Migrar/deprecar `chat-assistant` upstream de Supabase (después de N días estables).
