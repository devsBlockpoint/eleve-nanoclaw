# Easypanel Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Empaquetar y publicar las 3 imágenes Docker (`nanoclaw-host`, `nanoclaw-agent`, `mcp-monica`) en GHCR público; pushear los 3 repos a GitHub bajo `devsBlockpoint`; producir runbook completo de Easypanel para que el usuario complete el deploy en la UI.

**Architecture:** mcp-monica como service Easypanel separado (stateless, http-only). nanoclaw-host como service Easypanel con Docker socket montado para que pueda spawnar containers `nanoclaw-agent` per-sesión usando el daemon de Easypanel. Volume persistente `/app/data` para SQLite. Imágenes públicas en `ghcr.io/devsblockpoint/<name>:<tag>`.

**Tech Stack:** Docker, GHCR (público), Easypanel UI, GitHub (gh CLI).

**Spec de referencia:** `docs/superpowers/specs/2026-04-28-eleve-nanoclaw-monica-design.md` sección 7.

**Working directory:** distintos según task — root del monorepo `/home/morfi/eleve-nanoclaw/` para algunos, `nanoclaw/` y `mcp-monica/` para otros.

---

### Task 1: Dockerfile del host nanoclaw

**Files:**
- Create: `nanoclaw/Dockerfile`
- Create: `nanoclaw/.dockerignore` (si no existe)

**Reference:** Path B del análisis previo. El host habla con el Docker socket montado del host de Easypanel; necesita `docker-cli` + Node + pnpm.

- [ ] **Step 1: Crear `.dockerignore`**

```
node_modules
.git
.env
.env.local
*.log
data/
logs/
docs/
*.md
container/
groups/main/
groups/global/
.github/
.husky/
```

> Ojo: NO ignorar `groups/monica/` ni `scripts/init-monica-agent.ts` ni `src/`.

- [ ] **Step 2: Crear `Dockerfile`**

```dockerfile
# syntax=docker/dockerfile:1.7
# ============================================================================
# nanoclaw HOST — Easypanel-targeted image
# ============================================================================
# Runs the host process inside a container; uses the host's Docker daemon
# (via socket mount) to spawn per-session nanoclaw-agent containers.

FROM node:20-alpine AS deps
WORKDIR /app
RUN corepack enable && corepack prepare pnpm@10.13.1 --activate
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY patches ./patches
RUN pnpm install --frozen-lockfile --prod=false

FROM node:20-alpine AS build
WORKDIR /app
RUN corepack enable && corepack prepare pnpm@10.13.1 --activate
COPY --from=deps /app/node_modules ./node_modules
COPY --from=deps /app/package.json ./
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc tsconfig.json ./
COPY patches ./patches
COPY src ./src
COPY scripts ./scripts
COPY groups/monica ./groups/monica
RUN pnpm exec tsc

FROM node:20-alpine AS runtime
WORKDIR /app

# docker-cli to talk to the mounted host daemon, plus runtime deps
RUN apk add --no-cache docker-cli wget tini

RUN corepack enable && corepack prepare pnpm@10.13.1 --activate
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY scripts ./scripts
COPY groups/monica ./groups/monica
COPY src ./src

VOLUME ["/app/data"]
EXPOSE 3001

ENV NODE_ENV=production
ENV NANOCLAW_PORT=3001

HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
  CMD wget --spider -q http://localhost:3001/health || exit 1

ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/index.js"]
```

> Notas:
> - Si el upstream usa `tsx` en runtime y no compila a `dist/`, ajustá la última stage a `CMD ["pnpm", "exec", "tsx", "src/index.ts"]` y eliminá el stage `build`. Al implementer: chequear `nanoclaw/package.json` scripts; si `start = "node dist/index.js"`, mantené el build; si no, simplificá.
> - Pinear `pnpm@10.13.1` o la versión que use el repo (mirar `packageManager` en `package.json`).

- [ ] **Step 3: Build local**

```bash
cd /home/morfi/eleve-nanoclaw/nanoclaw
docker build -t nanoclaw-host:dev .
```

Expected: build OK. Tamaño esperado < 500MB.

- [ ] **Step 4: Smoke run del container (sin Docker socket por ahora — solo verificar boot)**

```bash
docker run --rm -d --name nanoclaw-host-smoke \
  -e AGENT_INBOUND_TOKEN=smoke \
  -e ELEVE_OUTBOUND_URL=http://example.test/out \
  -e ELEVE_OUTBOUND_TOKEN=smoke-out \
  -e AGENT_SYSTEM_PROMPT_SOURCE=env \
  -e AGENT_SYSTEM_PROMPT="Smoke test prompt" \
  -p 3013:3001 \
  -v nanoclaw-data-smoke:/app/data \
  nanoclaw-host:dev
sleep 4
echo "=== /health ==="
curl -s http://localhost:3013/health
echo ""
docker logs nanoclaw-host-smoke 2>&1 | head -20
docker stop nanoclaw-host-smoke
docker volume rm nanoclaw-data-smoke
```

Expected: `/health` → 200, logs muestran "eleve-http adapter listening".

> Si el host se queja por falta de `/var/run/docker.sock`, está bien — no podrá spawnar containers pero el HTTP layer funciona.

- [ ] **Step 5: Commit**

```bash
git add Dockerfile .dockerignore
git commit -m "chore(docker): host image for Easypanel deployment"
```

---

### Task 2: Build nanoclaw-agent (per-session container image)

**Files:** ningún cambio de código; solo build.

- [ ] **Step 1: Inspeccionar `container/build.sh`**

```bash
cd /home/morfi/eleve-nanoclaw/nanoclaw
cat container/build.sh | head -40
```

> Lectura para entender: qué tags pone, qué args acepta. La imagen output esperada es `nanoclaw-agent:latest` o similar.

- [ ] **Step 2: Build de la imagen**

```bash
cd /home/morfi/eleve-nanoclaw/nanoclaw
bash container/build.sh
docker images | grep nanoclaw-agent
```

> Si el script requiere setup adicional (env vars, registries), seguí lo que diga su output.

Expected: imagen `nanoclaw-agent:latest` (o tag que el script use) creada.

- [ ] **Step 3: Smoke local del agent — opcional**

Si el script provee un test mode, correrlo. Si no, saltar.

- [ ] **Step 4: NO commitear nada** — esta task solo construye la imagen.

---

### Task 3: Push de las 3 imágenes a GHCR

**Files:** ningún cambio de código.

**Pre-requisito:** `gh auth` autenticado con scope `write:packages`. Verificar:

```bash
gh auth status
gh auth refresh -s write:packages,read:packages
```

- [ ] **Step 1: Login a GHCR via gh CLI**

```bash
echo $(gh auth token) | docker login ghcr.io -u devsBlockpoint --password-stdin
```

Expected: "Login Succeeded".

- [ ] **Step 2: Tag y push de mcp-monica**

```bash
docker tag mcp-monica:dev ghcr.io/devsblockpoint/mcp-monica:0.1
docker tag mcp-monica:dev ghcr.io/devsblockpoint/mcp-monica:latest
docker push ghcr.io/devsblockpoint/mcp-monica:0.1
docker push ghcr.io/devsblockpoint/mcp-monica:latest
```

- [ ] **Step 3: Tag y push de nanoclaw-host**

```bash
docker tag nanoclaw-host:dev ghcr.io/devsblockpoint/nanoclaw-host:0.1
docker tag nanoclaw-host:dev ghcr.io/devsblockpoint/nanoclaw-host:latest
docker push ghcr.io/devsblockpoint/nanoclaw-host:0.1
docker push ghcr.io/devsblockpoint/nanoclaw-host:latest
```

- [ ] **Step 4: Tag y push de nanoclaw-agent**

```bash
# Confirm exact source tag — could be nanoclaw-agent:latest or similar.
docker tag nanoclaw-agent:latest ghcr.io/devsblockpoint/nanoclaw-agent:0.1
docker tag nanoclaw-agent:latest ghcr.io/devsblockpoint/nanoclaw-agent:latest
docker push ghcr.io/devsblockpoint/nanoclaw-agent:0.1
docker push ghcr.io/devsblockpoint/nanoclaw-agent:latest
```

- [ ] **Step 5: Hacer públicas las 3 packages**

Por defecto los packages nuevos en GHCR son privados. Necesitamos hacerlos públicos via la UI de GitHub o `gh api`:

```bash
for pkg in mcp-monica nanoclaw-host nanoclaw-agent; do
  gh api -X PATCH /orgs/devsBlockpoint/packages/container/$pkg \
    -f visibility=public 2>&1 | head -5
done
```

> Si `gh api` no soporta `-f visibility=public` en este endpoint, hacerlo manual via web UI: Settings → Packages → cada package → "Change visibility" → Public.

- [ ] **Step 6: Verificar pull público**

```bash
docker logout ghcr.io
docker pull ghcr.io/devsblockpoint/mcp-monica:0.1
```

Expected: pull OK sin auth.

---

### Task 4: Configurar nanoclaw para usar la imagen del agent desde GHCR

**Files:**
- Modify: `nanoclaw/src/container-runner.ts` (donde se especifica la imagen del agent)

- [ ] **Step 1: Localizar referencias a `nanoclaw-agent:latest`**

```bash
cd /home/morfi/eleve-nanoclaw/nanoclaw
grep -rn "nanoclaw-agent" src/ container/build.sh 2>/dev/null | head -10
```

- [ ] **Step 2: Hacer la imagen configurable por env**

Agregar env var `NANOCLAW_AGENT_IMAGE` (default: `nanoclaw-agent:latest` para dev local; en prod se setea a `ghcr.io/devsblockpoint/nanoclaw-agent:latest`).

Ubicación: probablemente en `src/container-runner.ts`, donde se construye el `docker run` args. El implementer lee el archivo, encuentra la string hardcoded, la reemplaza por `process.env.NANOCLAW_AGENT_IMAGE ?? 'nanoclaw-agent:latest'`.

- [ ] **Step 3: Agregar al `.env.example`**

```
# Imagen Docker del agent-runner (per-session container).
# Local dev: nanoclaw-agent:latest (build con container/build.sh)
# Prod (Easypanel/Path B): ghcr.io/devsblockpoint/nanoclaw-agent:latest
NANOCLAW_AGENT_IMAGE=nanoclaw-agent:latest
```

- [ ] **Step 4: Smoke + commit**

```bash
pnpm test 2>&1 | tail -5
pnpm exec tsc --noEmit 2>&1 | tail -5
git add src/container-runner.ts .env.example
git commit -m "feat: configurable agent image via NANOCLAW_AGENT_IMAGE env"
```

---

### Task 5: Crear repos en GitHub y push

**Files:** ningún cambio de código.

**Pre-requisitos:** `gh auth` ya hecho. Saber que:
- `devsBlockpoint/nanoclaw` ya existe (lo clonamos al inicio).
- `devsBlockpoint/mcp-monica` no existe.
- `devsBlockpoint/eleve-nanoclaw` (root) no existe.

- [ ] **Step 1: Crear `devsBlockpoint/mcp-monica`**

```bash
cd /home/morfi/eleve-nanoclaw/mcp-monica
gh repo create devsBlockpoint/mcp-monica --public --description "MCP server proxying ÉLEVÉ Supabase Edge Functions"
git push -u origin main
```

- [ ] **Step 2: Crear `devsBlockpoint/eleve-nanoclaw` (monorepo root)**

```bash
cd /home/morfi/eleve-nanoclaw
gh repo create devsBlockpoint/eleve-nanoclaw --public --description "ÉLEVÉ × nanoclaw integration: docs, ADRs, orchestration"
git push -u origin main
```

- [ ] **Step 3: Push de `nanoclaw` (fork)**

```bash
cd /home/morfi/eleve-nanoclaw/nanoclaw
git push origin main
```

> Si `git push` falla por divergence (upstream tags, etc.), usar `--force-with-lease`. Reportar.

- [ ] **Step 4: Verificar 3 repos en GitHub**

```bash
gh repo list devsBlockpoint --limit 10
```

Expected: ver `nanoclaw`, `mcp-monica`, `eleve-nanoclaw`.

---

### Task 6: Easypanel Runbook

**Files:**
- Create: `docs/architecture/deployment.md`

**Contract:** doc step-by-step que el usuario sigue en la UI de Easypanel para deployar todo. No es ejecutable; es manual.

- [ ] **Step 1: Escribir runbook**

Contenido sugerido (markdown):

```markdown
# Deployment Runbook — Easypanel

Pasos para deployar ÉLEVÉ × nanoclaw × mcp-monica en Easypanel después de Plan 4.

## Pre-requisitos

- Cuenta Easypanel con un proyecto creado.
- Acceso al panel de Supabase de ÉLEVÉ para configurar el webhook.
- Anthropic API key (o token equivalente vía OneCLI).
- Las 3 imágenes ya pushadas a GHCR público:
  - `ghcr.io/devsblockpoint/nanoclaw-host:latest`
  - `ghcr.io/devsblockpoint/nanoclaw-agent:latest`
  - `ghcr.io/devsblockpoint/mcp-monica:latest`

## 1. Service: mcp-monica

Easypanel UI:
- **Nuevo service** → tipo "App" → Source: "Image".
- **Image**: `ghcr.io/devsblockpoint/mcp-monica:latest`.
- **Port**: 3000 (no exponer al público — solo red interna).
- **Env vars**:
  ```
  SUPABASE_URL=https://YOUR_PROJECT.supabase.co
  SUPABASE_SERVICE_ROLE_KEY=<secret>
  MCP_PORT=3000
  ```
- **Healthcheck**: HTTP `/health` cada 30s.
- **Deploy**.

## 2. Service: nanoclaw-host

Easypanel UI:
- **Nuevo service** → tipo "App" → Source: "Image".
- **Image**: `ghcr.io/devsblockpoint/nanoclaw-host:latest`.
- **Port**: 3001 → exponer públicamente con dominio Easypanel + TLS.
- **Volumes / Mounts**:
  - Bind mount: `/var/run/docker.sock:/var/run/docker.sock` (CRÍTICO — el host habla con el daemon de Easypanel).
  - Volume persistente: `nanoclaw-data:/app/data`.
- **Env vars**:
  ```
  AGENT_INBOUND_TOKEN=<bearer compartido con Supabase webhook>
  ELEVE_OUTBOUND_URL=https://YOUR_PROJECT.supabase.co/functions/v1/n8n-whatsapp-agent-response
  ELEVE_OUTBOUND_TOKEN=<token outbound>
  NANOCLAW_PORT=3001
  AGENT_SYSTEM_PROMPT_SOURCE=url
  AGENT_SYSTEM_PROMPT_URL=<URL pública del prompt; ej. Drive doc /export?format=txt>
  AGENT_SYSTEM_PROMPT_URL_AUTH=
  MCP_MONICA_URL=http://mcp-monica:3000/mcp/sse  # network interna Easypanel
  NANOCLAW_AGENT_IMAGE=ghcr.io/devsblockpoint/nanoclaw-agent:latest
  ANTHROPIC_API_KEY=<secret>
  ```
- **Healthcheck**: HTTP `/health` cada 30s.
- **Deploy**.

## 3. Bootstrap monica agent group

Una vez `nanoclaw-host` esté arriba (ver logs):

```bash
# Desde Easypanel UI o ssh al host:
docker exec -it <nanoclaw-host-container> sh -c "cd /app && pnpm exec tsx scripts/init-monica-agent.ts"
```

Output: `Created agent group: ag-... (monica)`. Copiar el `<id>`.

```bash
# OneCLI: poner el agente en modo all (si OneCLI está corriendo):
onecli agents set-secret-mode --id <id> --mode all
```

## 4. Configurar webhook ÉLEVÉ Supabase

En `whatsapp-webhook` (edge function), modificar para que en lugar de invocar `chat-assistant` haga:

```
POST {NANOCLAW_PUBLIC_URL}/messages
Authorization: Bearer ${AGENT_INBOUND_TOKEN}
Content-Type: application/json
{ "conversation_id": "...", "message": "...", "sender": {...} }
```

`NANOCLAW_PUBLIC_URL` es el dominio que Easypanel asignó al service `nanoclaw-host`.

## 5. Smoke real

Mandar un WhatsApp al número de ÉLEVÉ. Verificar:
- Logs de `nanoclaw-host`: ver `eleve-http: created messaging_group ... auto-wired ... Session created ... Spawning container`.
- Logs del container per-session (Easypanel los muestra como containers ad-hoc): `nanoclaw-v2-monica-...`.
- Logs de `mcp-monica`: ver una llamada a alguna tool si el agente decide invocarlas.
- WhatsApp recibe respuesta del agente.

## Troubleshooting

- **OneCLI 401**: el agente no tiene secrets asignados. Revisar `onecli agents secrets --id <id>`.
- **Container exit 125**: la imagen `nanoclaw-agent` no se pudo descargar. Verificar `docker pull ghcr.io/devsblockpoint/nanoclaw-agent:latest` desde el host.
- **MCP connection refused**: la red interna de Easypanel no está conectando services. Verificar que ambos services estén en el mismo proyecto.
- **`/health` 401**: estás golpeando un servicio diferente (otra app en :3001). Verificar el dominio asignado.
```

- [ ] **Step 2: Commit en root**

```bash
cd /home/morfi/eleve-nanoclaw
git add docs/architecture/deployment.md
git commit -m "docs(deployment): Easypanel runbook"
```

---

### Task 7: Smoke real (manual; user-driven)

Esta task la ejecuta el usuario en Easypanel UI siguiendo el runbook. No automatizable.

**Criterios de éxito:**
- Mandar mensaje WhatsApp → recibir respuesta del agente Mónica en menos de 30s.
- Logs muestran toda la cadena: webhook ÉLEVÉ → nanoclaw inbound → spawn container → tool call (si aplica) → outbound → ÉLEVÉ → WhatsApp.
- Memory persistente: 2 mensajes seguidos en la misma conversación → la respuesta refleja contexto del primero.

**Si falla:** el runbook tiene troubleshooting; ajustar; iterar.

---

## Self-review

**Spec coverage:** spec sección 7 (deployment) cubierto. Sección 6 (system prompt loader) cubierto vía env var en runbook (URL source).

**Placeholder scan:** algunos `<...>` en el runbook (URLs, secrets) son por diseño — son inputs del usuario, no placeholders del plan.

**Riesgos restantes:**
- Path B (Docker socket mount) puede tener problemas no anticipados con la sandbox de Easypanel; troubleshooting puede requerir iteración.
- `container/build.sh` puede tener supuestos no portables (paths, env vars de upstream); el implementer adapta si hace falta.
- Cold start del primer pull del `nanoclaw-agent` (~1GB) puede ser lento; aceptable.

---

## Cierre del proyecto

Cuando Plan 4 termine con éxito, ÉLEVÉ × nanoclaw × mcp-monica está vivo en producción. Los próximos workstreams (out of scope):
- Migrar `chat-assistant` (deprecar / eliminar después de validación).
- Habilitar las 10 tools `pending` cuando sus edge functions estén listas.
- Agregar más agent groups si el negocio lo pide (ventas vs soporte separados).
- Hot-reload del system prompt sin redeploy.
- Migrar a `StreamableHTTPServerTransport` cuando el SDK lo recomiende firmemente.
