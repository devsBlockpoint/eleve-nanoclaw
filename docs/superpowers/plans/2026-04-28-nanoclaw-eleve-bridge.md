# nanoclaw × ÉLEVÉ Bridge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Customizar el fork de nanoclaw upstream (`devsBlockpoint/nanoclaw`) para integrarlo con ÉLEVÉ: agregar el `eleve-http` channel adapter (recibe webhooks de Supabase, despacha respuestas a `n8n-whatsapp-agent-response`), wirear el agente `monica` con `mcp-monica` vía MCP HTTP, y soportar carga del system prompt desde 3 fuentes (env/file/url).

**Architecture:** Mantener el patrón upstream de nanoclaw intacto (host Node + pnpm, container Bun, dos SQLite por sesión). Agregar piezas como skills propias del fork: nuevo channel adapter en `src/channels/`, un agent group `monica` en `groups/`, un loader del system prompt en `src/`, y extensión menor del schema `McpServerConfig` para soportar HTTP MCP servers. **No** modificar el motor del router/delivery — entrar como un adapter más.

**Tech Stack:** TypeScript, Node 20+ + pnpm 10+ (host), Bun (container, sin cambios), `@modelcontextprotocol/sdk` (ya presente upstream).

**Spec de referencia:** `docs/superpowers/specs/2026-04-28-eleve-nanoclaw-monica-design.md` secciones 4, 6, 7.

**Mapa de customizations:** `nanoclaw/docs/customizations.md` — leer secciones A-H para contexto técnico de cada tarea.

**Working directory:** `/home/morfi/eleve-nanoclaw/nanoclaw/` (repo `devsBlockpoint/nanoclaw`, fork de `qwibitai/nanoclaw`, branch `main`, user `devsBlockpoint <developers@blockpoint.cloud>`).

---

## Estructura de archivos

| Path | Acción | Responsabilidad |
|---|---|---|
| `nanoclaw/scripts/init-monica-agent.ts` | Crear | Bootstrap idempotente del agent group `monica` (DB seed + `groups/monica/` filesystem) |
| `nanoclaw/groups/monica/CLAUDE.md` | Crear | Placeholder; se sobreescribe en runtime con el system prompt loader |
| `nanoclaw/groups/monica/container.json` | Crear | Per-group config con `mcp-monica` HTTP MCP, `assistantName: 'Mónica'` |
| `nanoclaw/src/container-config.ts` | Modificar | Extender `McpServerConfig` a union: stdio \| http/sse |
| `nanoclaw/container/agent-runner/src/config.ts` | Modificar | Mismo, lado container |
| `nanoclaw/container/agent-runner/src/index.ts` | Modificar | Pasar HTTP MCP entries al SDK con la forma correcta |
| `nanoclaw/src/system-prompt-loader.ts` | Crear | Carga prompt env/file/url + cache + fallback; escribe `groups/monica/CLAUDE.local.md` |
| `nanoclaw/tests/system-prompt-loader.test.ts` | Crear | Unit tests del loader |
| `nanoclaw/src/channels/eleve-http.ts` | Crear | Channel adapter HTTP (inbound POST /messages + outbound POST n8n-...-response + auto-wiring) |
| `nanoclaw/tests/eleve-http.test.ts` | Crear | Unit tests del adapter (handler bearer, parsing, deliver) |
| `nanoclaw/src/channels/index.ts` | Modificar | Registrar `eleve-http` en barrel |
| `nanoclaw/.env.example` | Modificar/crear | Documentar env vars nuevas |
| `nanoclaw/docs/bridge-eleve.md` | Crear | Contrato HTTP completo (inbound/outbound/auth/retry) |

**No modificar:**
- `src/router.ts`, `src/delivery.ts`, `src/host-sweep.ts`, `src/session-manager.ts` — motor upstream queda intacto.
- `container/agent-runner/src/poll-loop.ts`, `formatter.ts`, etc. — lógica del runner intacta.

---

### Task 1: Extender `McpServerConfig` para soportar HTTP MCP

**Files:**
- Modify: `nanoclaw/src/container-config.ts` — `McpServerConfig` type
- Modify: `nanoclaw/container/agent-runner/src/config.ts` — `RunnerConfig.mcpServers` shape
- Modify: `nanoclaw/container/agent-runner/src/index.ts` — pasaje al SDK

**Reference:** `nanoclaw/docs/customizations.md` Sección C.

**Step 1: Localizar las definiciones actuales**

```bash
cd /home/morfi/eleve-nanoclaw/nanoclaw
grep -n "McpServerConfig\|mcpServers" src/container-config.ts container/agent-runner/src/config.ts container/agent-runner/src/index.ts
```

- [ ] **Step 2: Extender el host-side type a union (`src/container-config.ts`)**

Cambiar la definición existente de `McpServerConfig` (probablemente `{ command, args, env }`) a:

```typescript
export type McpServerConfig =
  | { type?: 'stdio'; command: string; args?: string[]; env?: Record<string, string> }
  | { type: 'sse' | 'http'; url: string; headers?: Record<string, string> };

export function isHttpMcpServer(
  cfg: McpServerConfig,
): cfg is { type: 'sse' | 'http'; url: string; headers?: Record<string, string> } {
  return 'url' in cfg && (cfg.type === 'sse' || cfg.type === 'http');
}
```

(Si la definición upstream usa interface en vez de union, refactorizar al union — el implementer adapta a la sintaxis real del archivo.)

- [ ] **Step 3: Mismo cambio en `container/agent-runner/src/config.ts`**

Mantener consistencia entre host y container. Mismo type, misma helper.

- [ ] **Step 4: En `container/agent-runner/src/index.ts`, pasar HTTP entries al SDK**

Buscar dónde se construye el array de MCP servers que se pasa a `query()` o equivalente del Claude Agent SDK. Para cada entry HTTP, pasar `{ type: 'sse' | 'http', url, headers }` directamente — el SDK ya lo soporta (ver `docs/SDK_DEEP_DIVE.md` en nanoclaw upstream).

- [ ] **Step 5: Verificar typecheck**

```bash
cd /home/morfi/eleve-nanoclaw/nanoclaw
pnpm exec tsc --noEmit 2>&1 | head -30
```

Expected: sin errores nuevos. (El typecheck puede tener errores preexistentes de upstream que no son tuyos — los ignorás si no se relacionan con tus archivos.)

```bash
cd /home/morfi/eleve-nanoclaw/nanoclaw/container/agent-runner
pnpm exec tsc -p tsconfig.json --noEmit 2>&1 | head -30
```

Expected: sin errores nuevos.

- [ ] **Step 6: Commit**

```bash
cd /home/morfi/eleve-nanoclaw/nanoclaw
git add src/container-config.ts container/agent-runner/src/config.ts container/agent-runner/src/index.ts
git commit -m "feat(mcp): support HTTP/SSE MCP servers in container config"
```

---

### Task 2: Crear `groups/monica/container.json` y CLAUDE.md placeholder

**Files:**
- Create: `nanoclaw/groups/monica/container.json`
- Create: `nanoclaw/groups/monica/CLAUDE.md`

**Reference:** `nanoclaw/docs/customizations.md` Sección E.

- [ ] **Step 1: Crear container.json**

```json
{
  "provider": "claude",
  "assistantName": "Mónica",
  "groupName": "monica",
  "agentGroupId": "REPLACED_AT_RUNTIME",
  "maxMessagesPerPrompt": 10,
  "mcpServers": {
    "mcp-monica": {
      "type": "sse",
      "url": "http://mcp-monica:3000/mcp/sse"
    }
  }
}
```

> Nota: `agentGroupId` se llena en runtime por `ensureRuntimeFields()` del host. Lo dejamos placeholder; no romper la forma esperada.

- [ ] **Step 2: Crear CLAUDE.md placeholder**

```markdown
# Mónica — placeholder

Este archivo se sobreescribe en runtime por `src/system-prompt-loader.ts` antes de cada wake del container, según la fuente configurada en `AGENT_SYSTEM_PROMPT_SOURCE` (env/file/url).

Si ves este texto en un container vivo, el loader no corrió o falló sin cache de fallback. Revisar logs del host: `logs/nanoclaw.log` busca `[system-prompt-loader]`.
```

- [ ] **Step 3: Verificar JSON válido**

```bash
cd /home/morfi/eleve-nanoclaw/nanoclaw
node -e "JSON.parse(require('fs').readFileSync('groups/monica/container.json','utf8')); console.log('JSON OK')"
```

Expected: `JSON OK`.

- [ ] **Step 4: Commit**

```bash
git add groups/monica/
git commit -m "feat(monica): add agent group skeleton with mcp-monica HTTP MCP wiring"
```

---

### Task 3: Bootstrap script `scripts/init-monica-agent.ts`

**Files:**
- Create: `nanoclaw/scripts/init-monica-agent.ts`

**Reference:** `nanoclaw/docs/customizations.md` Sección E + secciones D y G (assistantName overwrite, OneCLI secret mode).

**Contract:** script idempotente que (1) lee/crea fila en `agent_groups` con `name='Mónica'`, `folder='monica'` (campos exactos según schema upstream — leer `src/db/agent-groups.ts`); (2) si `groups/monica/` no existe, llama a `initGroupFilesystem` del upstream; (3) imprime el `agent_group_id` resultante para que se use en setup OneCLI.

- [ ] **Step 1: Inspeccionar el schema actual y `initGroupFilesystem`**

```bash
cd /home/morfi/eleve-nanoclaw/nanoclaw
grep -n "create.*[Aa]gent[Gg]roup\|initGroupFilesystem" src/db/agent-groups.ts src/group-init.ts | head -20
```

- [ ] **Step 2: Escribir el script**

```typescript
#!/usr/bin/env node
/**
 * Bootstrap idempotente del agent group `monica`.
 *
 * Run: pnpm exec tsx scripts/init-monica-agent.ts
 */
import { openDb } from '../src/db/connection.js';
import { createAgentGroup, getAgentGroupByFolder } from '../src/db/agent-groups.js';
import { initGroupFilesystem } from '../src/group-init.js';

async function main() {
  const db = openDb();

  let group = getAgentGroupByFolder(db, 'monica');
  if (!group) {
    group = createAgentGroup(db, {
      name: 'Mónica',
      folder: 'monica',
      // Campos adicionales según schema upstream: completar al ver db/agent-groups.ts
    });
    console.log(`Created agent group: id=${group.id} name=Mónica folder=monica`);
  } else {
    console.log(`Agent group already exists: id=${group.id}`);
  }

  await initGroupFilesystem(group);

  console.log('');
  console.log('Next steps:');
  console.log(`  1. Ensure OneCLI agent for this group has secret mode = "all":`);
  console.log(`     onecli agents set-secret-mode --id ${group.id} --mode all`);
  console.log(`  2. Wire eleve-http channel to this agent group via SQL or admin tool.`);
  console.log(`  3. Start nanoclaw and POST to /messages with bearer AGENT_INBOUND_TOKEN.`);
}

main().catch((err) => {
  console.error('Bootstrap failed:', err);
  process.exit(1);
});
```

> El implementer ajusta los nombres de funciones/argumentos según los exports reales de `src/db/agent-groups.ts` y `src/group-init.ts`. Los nombres arriba son aproximados; lo crítico es: (a) idempotencia, (b) `name='Mónica'` exacto, (c) folder='monica'.

- [ ] **Step 3: Smoke run (no prod)**

```bash
cd /home/morfi/eleve-nanoclaw/nanoclaw
NANOCLAW_DATA_DIR=/tmp/nanoclaw-test-$$ pnpm exec tsx scripts/init-monica-agent.ts 2>&1 | head -20
# Re-run para verificar idempotencia
NANOCLAW_DATA_DIR=/tmp/nanoclaw-test-$$ pnpm exec tsx scripts/init-monica-agent.ts 2>&1 | head -20
```

> Si el upstream no tiene env var `NANOCLAW_DATA_DIR` o si el schema requiere migrations primero, ajustar — el smoke es para verificar que el script corre sin crashear, no para tener datos limpios.

Expected: dos invocaciones consecutivas, la primera crea, la segunda dice "already exists".

- [ ] **Step 4: Commit**

```bash
git add scripts/init-monica-agent.ts
git commit -m "feat(monica): bootstrap script for agent group seed"
```

---

### Task 4: System prompt loader `src/system-prompt-loader.ts` con TDD

**Files:**
- Create: `nanoclaw/tests/system-prompt-loader.test.ts`
- Create: `nanoclaw/src/system-prompt-loader.ts`

**Reference:** `nanoclaw/docs/customizations.md` Sección D + spec sección 6.

**Contract:** función `resolveSystemPrompt(opts: { source, env?, path?, url?, urlAuth?, cachePath })` retorna `Promise<string>`. Se invoca desde `src/index.ts` al boot y/o antes de cada container wake (decisión: una vez al boot es suficiente para v1; la spec menciona hot-reload como opcional). Escribe el resultado a `groups/monica/CLAUDE.local.md`.

- [ ] **Step 1: Tests (vitest, host)**

`tests/system-prompt-loader.test.ts`:

```typescript
import { describe, test, expect, beforeEach, afterEach, vi } from 'vitest';
import { mkdtemp, readFile, writeFile, rm } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import { resolveSystemPrompt } from '../src/system-prompt-loader.js';

let tmpDir: string;
beforeEach(async () => {
  tmpDir = await mkdtemp(join(tmpdir(), 'sp-loader-'));
});
afterEach(async () => {
  await rm(tmpDir, { recursive: true, force: true });
});

describe('resolveSystemPrompt', () => {
  test('source=env returns the env value', async () => {
    const out = await resolveSystemPrompt({
      source: 'env',
      env: 'You are Mónica.',
      cachePath: join(tmpDir, 'cache.md'),
    });
    expect(out).toBe('You are Mónica.');
  });

  test('source=file reads from disk', async () => {
    const path = join(tmpDir, 'prompt.md');
    await writeFile(path, '# Mónica\n');
    const out = await resolveSystemPrompt({
      source: 'file',
      path,
      cachePath: join(tmpDir, 'cache.md'),
    });
    expect(out).toContain('Mónica');
  });

  test('source=url fetches and writes to cache on success', async () => {
    const cachePath = join(tmpDir, 'cache.md');
    const fetchImpl = vi.fn(async () =>
      new Response('PROMPT FROM URL', { status: 200 }),
    ) as typeof fetch;
    const out = await resolveSystemPrompt({
      source: 'url',
      url: 'https://example.com/prompt',
      cachePath,
      fetchImpl,
    });
    expect(out).toBe('PROMPT FROM URL');
    expect(await readFile(cachePath, 'utf8')).toBe('PROMPT FROM URL');
  });

  test('source=url falls back to cache on fetch failure', async () => {
    const cachePath = join(tmpDir, 'cache.md');
    await writeFile(cachePath, 'CACHED PROMPT');
    const fetchImpl = vi.fn(async () => {
      throw new Error('network down');
    }) as typeof fetch;
    const out = await resolveSystemPrompt({
      source: 'url',
      url: 'https://example.com/prompt',
      cachePath,
      fetchImpl,
    });
    expect(out).toBe('CACHED PROMPT');
  });

  test('source=url throws if no cache and fetch fails', async () => {
    const cachePath = join(tmpDir, 'cache-none.md');
    const fetchImpl = vi.fn(async () => {
      throw new Error('network down');
    }) as typeof fetch;
    await expect(
      resolveSystemPrompt({ source: 'url', url: 'https://example.com', cachePath, fetchImpl }),
    ).rejects.toThrow();
  });
});
```

- [ ] **Step 2: Run tests (must fail)**

```bash
cd /home/morfi/eleve-nanoclaw/nanoclaw
pnpm exec vitest run tests/system-prompt-loader.test.ts 2>&1 | tail -30
```

Expected: module not found.

- [ ] **Step 3: Implementar `src/system-prompt-loader.ts`**

```typescript
/**
 * System prompt loader — 3 fuentes (env / file / url) con cache + fallback.
 * Llamado desde src/index.ts al boot (y opcionalmente con interval de reload).
 */
import { readFile, writeFile } from 'node:fs/promises';

export type PromptSource = 'env' | 'file' | 'url';

export interface ResolveOptions {
  source: PromptSource;
  env?: string;
  path?: string;
  url?: string;
  urlAuth?: string;
  cachePath: string;
  timeoutMs?: number;
  fetchImpl?: typeof fetch;
}

const DEFAULT_TIMEOUT_MS = 5000;

export async function resolveSystemPrompt(opts: ResolveOptions): Promise<string> {
  switch (opts.source) {
    case 'env': {
      if (!opts.env) throw new Error('source=env requires opts.env');
      return opts.env;
    }
    case 'file': {
      if (!opts.path) throw new Error('source=file requires opts.path');
      return readFile(opts.path, 'utf8');
    }
    case 'url': {
      if (!opts.url) throw new Error('source=url requires opts.url');
      try {
        const content = await fetchPrompt(opts);
        await writeFile(opts.cachePath, content, 'utf8');
        return content;
      } catch (err) {
        try {
          const cached = await readFile(opts.cachePath, 'utf8');
          console.warn(
            `[system-prompt-loader] fetch failed (${(err as Error).message}); using cache`,
          );
          return cached;
        } catch {
          throw new Error(
            `[system-prompt-loader] fetch failed and no cache available: ${(err as Error).message}`,
          );
        }
      }
    }
  }
}

async function fetchPrompt(opts: ResolveOptions): Promise<string> {
  const fetchImpl = opts.fetchImpl ?? fetch;
  const timeoutMs = opts.timeoutMs ?? DEFAULT_TIMEOUT_MS;
  const controller = new AbortController();
  const t = setTimeout(() => controller.abort(), timeoutMs);
  try {
    const headers: Record<string, string> = {};
    if (opts.urlAuth) headers.Authorization = opts.urlAuth;
    const res = await fetchImpl(opts.url!, { headers, signal: controller.signal });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return await res.text();
  } finally {
    clearTimeout(t);
  }
}
```

- [ ] **Step 4: Wire en `src/index.ts`**

Localizar el bloque de inicialización del host. Antes de `initChannelAdapters()`, agregar:

```typescript
import { resolveSystemPrompt } from './system-prompt-loader.js';
import { writeFile } from 'node:fs/promises';
import { join } from 'node:path';

const promptSource = (process.env.AGENT_SYSTEM_PROMPT_SOURCE ?? 'env') as 'env' | 'file' | 'url';
if (process.env.AGENT_GROUP === 'monica' || !process.env.AGENT_GROUP) {
  const dataDir = process.env.NANOCLAW_DATA_DIR ?? './data';
  const cachePath = join(dataDir, 'system-prompt.cache.md');
  const groupClaudeMd = join('./groups/monica', 'CLAUDE.local.md');
  try {
    const prompt = await resolveSystemPrompt({
      source: promptSource,
      env: process.env.AGENT_SYSTEM_PROMPT,
      path: process.env.AGENT_SYSTEM_PROMPT_PATH,
      url: process.env.AGENT_SYSTEM_PROMPT_URL,
      urlAuth: process.env.AGENT_SYSTEM_PROMPT_URL_AUTH,
      cachePath,
    });
    await writeFile(groupClaudeMd, prompt, 'utf8');
    console.log(`[boot] system prompt loaded from source=${promptSource} (${prompt.length} chars)`);
  } catch (err) {
    console.error(`[boot] FATAL: system prompt resolution failed: ${(err as Error).message}`);
    process.exit(1);
  }
}
```

> El path exacto en `src/index.ts` el implementer lo localiza buscando `initChannelAdapters` o `migrateDb`. Insertar AFTER migrations, BEFORE channel adapters.

- [ ] **Step 5: Run tests (must pass)**

```bash
pnpm exec vitest run tests/system-prompt-loader.test.ts
```

Expected: 5 tests passing.

- [ ] **Step 6: Commit**

```bash
git add src/system-prompt-loader.ts tests/system-prompt-loader.test.ts src/index.ts
git commit -m "feat(prompt): 3-source loader (env/file/url) with cache fallback"
```

---

### Task 5: `eleve-http` channel adapter — inbound + auto-wiring

**Files:**
- Create: `nanoclaw/tests/eleve-http.test.ts`
- Create: `nanoclaw/src/channels/eleve-http.ts`

**Reference:** `nanoclaw/docs/customizations.md` Sección A (adapter contract) + Sección G (auto-wiring concern).

**Contract:** adapter HTTP que escucha en `NANOCLAW_PORT` (default 3001). `POST /messages` con bearer `AGENT_INBOUND_TOKEN`. Responde 202. Auto-wirea (en `processInbound`) el `messaging_group` para el `conversation_id` al agent group `monica` si no existe. Llama a `setup.onInbound(...)` con un `InboundEvent` formado correctamente. `GET /health` siempre devuelve `{status:"ok"}`.

> **El auto-wiring es el riesgo más alto del plan**. Importar de `src/db/messaging-groups.ts` y `src/db/agent-groups.ts` desde un channel adapter es un boundary violation pero necesario por la arquitectura upstream. Si en el futuro upstream agrega un hook tipo `onUnwiredInbound`, migrar a eso.

- [ ] **Step 1: Inspeccionar funciones DB upstream**

```bash
cd /home/morfi/eleve-nanoclaw/nanoclaw
grep -n "export function" src/db/messaging-groups.ts src/db/agent-groups.ts | head -30
```

- [ ] **Step 2: Tests del adapter**

`tests/eleve-http.test.ts`:

```typescript
import { describe, test, expect, vi } from 'vitest';
import { createEleveHttpAdapter } from '../src/channels/eleve-http.js';

describe('eleve-http adapter', () => {
  test('rejects requests without bearer', async () => {
    const setup = makeFakeSetup();
    const adapter = createEleveHttpAdapter({
      token: 'secret',
      outboundUrl: 'https://eleve.test/out',
      outboundToken: 'out-token',
    });
    await adapter.setup(setup);
    const res = await fetch(`http://localhost:${adapter.port}/messages`, {
      method: 'POST',
      body: JSON.stringify({ conversation_id: 'c1', message: 'hi', sender: { phone: '5215...' } }),
    });
    expect(res.status).toBe(401);
    await adapter.shutdown();
  });

  test('accepts valid bearer and dispatches onInbound', async () => {
    const setup = makeFakeSetup();
    const adapter = createEleveHttpAdapter({
      token: 'secret',
      outboundUrl: 'https://eleve.test/out',
      outboundToken: 'out-token',
    });
    await adapter.setup(setup);
    const res = await fetch(`http://localhost:${adapter.port}/messages`, {
      method: 'POST',
      headers: { Authorization: 'Bearer secret', 'Content-Type': 'application/json' },
      body: JSON.stringify({
        conversation_id: 'conv-1',
        message: 'hola',
        sender: { phone: '5215512345678', name: 'Ana' },
      }),
    });
    expect(res.status).toBe(202);
    expect(setup.onInboundCalls).toHaveLength(1);
    expect(setup.onInboundCalls[0].event.platformId).toContain('conv-1');
    await adapter.shutdown();
  });

  test('GET /health returns ok regardless of auth', async () => {
    const setup = makeFakeSetup();
    const adapter = createEleveHttpAdapter({
      token: 'secret',
      outboundUrl: 'x',
      outboundToken: 'y',
    });
    await adapter.setup(setup);
    const res = await fetch(`http://localhost:${adapter.port}/health`);
    expect(res.status).toBe(200);
    expect(await res.json()).toEqual({ status: 'ok' });
    await adapter.shutdown();
  });

  test('deliver POSTs to outbound URL with bearer and JSON body', async () => {
    const setup = makeFakeSetup();
    const fetchImpl = vi.fn(async (_url: any, _init: any) => new Response('{}', { status: 200 }));
    const adapter = createEleveHttpAdapter({
      token: 'secret',
      outboundUrl: 'https://eleve.test/n8n-out',
      outboundToken: 'out-token',
      fetchImpl: fetchImpl as typeof fetch,
    });
    await adapter.setup(setup);
    await adapter.deliver({
      channelType: 'eleve-http',
      platformId: 'conv-42',
      threadId: null,
      message: 'tu cita está confirmada',
    });
    expect(fetchImpl).toHaveBeenCalled();
    const [url, init] = fetchImpl.mock.calls[0];
    expect(url).toBe('https://eleve.test/n8n-out');
    expect(init?.method).toBe('POST');
    const headers = new Headers(init?.headers);
    expect(headers.get('Authorization')).toBe('Bearer out-token');
    const body = JSON.parse(String(init?.body));
    expect(body).toMatchObject({ conversation_id: 'conv-42', message: 'tu cita está confirmada' });
    await adapter.shutdown();
  });
});

function makeFakeSetup() {
  const onInboundCalls: any[] = [];
  return {
    onInboundCalls,
    onInbound: (platformId: string, threadId: string | null, message: any) => {
      onInboundCalls.push({ event: { platformId, threadId, message } });
    },
    onInboundEvent: () => {},
    onMetadata: () => {},
    onAction: () => {},
  } as any;
}
```

- [ ] **Step 3: Run tests (must fail)**

```bash
pnpm exec vitest run tests/eleve-http.test.ts 2>&1 | tail -20
```

Expected: module not found.

- [ ] **Step 4: Implementar `src/channels/eleve-http.ts`**

Implementar la función `createEleveHttpAdapter(config)` que retorna `{ port, setup, deliver, shutdown }`. La estructura general:

```typescript
import { createServer, type IncomingMessage, type ServerResponse } from 'node:http';
import { registerChannelAdapter } from './channel-registry.js';
import type { ChannelAdapter, ChannelSetup, InboundEvent } from './adapter.js';

export interface EleveHttpConfig {
  token: string;
  outboundUrl: string;
  outboundToken: string;
  port?: number;
  fetchImpl?: typeof fetch;
}

export interface EleveHttpAdapterHandle {
  port: number;
  setup: (s: ChannelSetup) => Promise<void>;
  deliver: (msg: { channelType: string; platformId: string; threadId: string | null; message: string; metadata?: unknown }) => Promise<void>;
  shutdown: () => Promise<void>;
}

export function createEleveHttpAdapter(config: EleveHttpConfig): EleveHttpAdapterHandle {
  // Implementación:
  //  - createServer con 2 rutas: GET /health → {status:'ok'}; POST /messages → procesar
  //  - POST /messages: validar bearer, parsear JSON body, ensureMessagingGroupWired (auto-wiring),
  //    construir InboundEvent { channelType: 'eleve-http', platformId: conversation_id, threadId: null, message: {...} },
  //    llamar setup.onInbound(...), responder 202.
  //  - deliver(msg): POST a outboundUrl con bearer + body { conversation_id: msg.platformId, message: msg.message, metadata: msg.metadata }
  //  - shutdown: cerrar server.
  // El implementer completa la lógica HTTP siguiendo el contrato de los tests.
  // ...
}

// Auto-wiring helper — llama a DB layer del host
async function ensureMessagingGroupWired(conversationId: string): Promise<void> {
  // Importar de '../db/messaging-groups.js' y '../db/agent-groups.js'
  // 1. Buscar messaging_group por (channel_type='eleve-http', platform_id=conversationId).
  // 2. Si no existe, crearlo.
  // 3. Buscar wiring a agent group folder='monica'.
  // 4. Si no existe, crearlo via createMessagingGroupAgent.
  // El implementer mira customizations.md sección A.4 para los nombres exactos de funciones.
}

// Self-registration al import (igual que cli.ts)
registerChannelAdapter('eleve-http', {
  factory: () => {
    const token = process.env.AGENT_INBOUND_TOKEN;
    if (!token) {
      console.warn('[eleve-http] AGENT_INBOUND_TOKEN not set; adapter disabled');
      return null;
    }
    return createEleveHttpAdapter({
      token,
      outboundUrl: process.env.ELEVE_OUTBOUND_URL ?? '',
      outboundToken: process.env.ELEVE_OUTBOUND_TOKEN ?? '',
      port: Number(process.env.NANOCLAW_PORT ?? '3001'),
    });
  },
});
```

> El implementer completa los TODOs. Lo crítico:
> - `port` debe asignarse efectivamente al iniciar el server (escuchar en `0.0.0.0:port`).
> - Validar bearer ANTES de parsear body.
> - Responder 202 inmediatamente, NO bloquear esperando que el agente procese.
> - `platformId` = `conversation_id` (ÉLEVÉ envía esto, lo guardamos así para que el outbound sea simétrico).
> - Auto-wiring: si en los tests podés mockear las DB calls (con dependency injection), perfecto; si no, los tests del wiring quedan para Task 6 con tests integration. Los unit tests de Task 5 pueden no probar wiring.

- [ ] **Step 5: Run tests (must pass)**

```bash
pnpm exec vitest run tests/eleve-http.test.ts
```

Expected: 4 tests passing.

- [ ] **Step 6: Commit**

```bash
git add src/channels/eleve-http.ts tests/eleve-http.test.ts
git commit -m "feat(channels): eleve-http adapter with bearer auth and outbound delivery"
```

---

### Task 6: Registrar `eleve-http` en barrel y verificar startup

**Files:**
- Modify: `nanoclaw/src/channels/index.ts`

- [ ] **Step 1: Agregar import**

Buscar el archivo (es un barrel de auto-registro):

```bash
cat src/channels/index.ts
```

Agregar al final:

```typescript
import './eleve-http.js';
```

- [ ] **Step 2: Smoke run del host**

```bash
cd /home/morfi/eleve-nanoclaw/nanoclaw
AGENT_INBOUND_TOKEN=test-secret \
ELEVE_OUTBOUND_URL=https://example.test/out \
ELEVE_OUTBOUND_TOKEN=test-out \
NANOCLAW_PORT=3011 \
AGENT_SYSTEM_PROMPT_SOURCE=env \
AGENT_SYSTEM_PROMPT="You are Mónica, a test prompt." \
NANOCLAW_DATA_DIR=/tmp/nanoclaw-smoke-$$ \
timeout 5 pnpm run dev > /tmp/nanoclaw-smoke.log 2>&1 &
SERVER_PID=$!
sleep 3
echo "=== /health ==="
curl -s http://localhost:3011/health || echo "(no response)"
echo ""
echo "=== POST without auth ==="
curl -s -o /dev/null -w "%{http_code}" -X POST http://localhost:3011/messages -H "Content-Type: application/json" -d '{}'
echo ""
echo "=== logs head ==="
head -20 /tmp/nanoclaw-smoke.log
kill $SERVER_PID 2>/dev/null
wait 2>/dev/null
echo "---done---"
```

Expected: `/health` → 200; POST sin auth → 401; logs muestran adapter registrado y server escuchando.

> Si el smoke falla porque `pnpm run dev` requiere setup adicional (DB migration, OneCLI, etc.), el implementer ajusta — el objetivo es que el adapter HTTP responda. Si hay errores upstream que no son tuyos (DB no migrada, etc.), correr migrations primero.

- [ ] **Step 3: Commit**

```bash
git add src/channels/index.ts
git commit -m "feat(channels): register eleve-http adapter in barrel"
```

---

### Task 7: `.env.example` y `docs/bridge-eleve.md`

**Files:**
- Create/update: `nanoclaw/.env.example`
- Create: `nanoclaw/docs/bridge-eleve.md`

**Reference:** `nanoclaw/docs/customizations.md` Sección F.

- [ ] **Step 1: `.env.example`**

Si existe, hacer append de las vars nuevas con sección "ÉLEVÉ integration"; si no existe, crear:

```dotenv
# =============================================================================
# nanoclaw — env vars (incluye customizations ÉLEVÉ)
# =============================================================================

# --- ÉLEVÉ integration ---

# Bearer token que ÉLEVÉ envía al llamar POST /messages.
AGENT_INBOUND_TOKEN=

# Puerto HTTP del eleve-http channel adapter (default 3001).
NANOCLAW_PORT=3001

# Endpoint de salida (Supabase Edge Function n8n-whatsapp-agent-response).
ELEVE_OUTBOUND_URL=https://YOUR_PROJECT.supabase.co/functions/v1/n8n-whatsapp-agent-response

# Bearer para el outbound (típicamente service_role o token similar).
ELEVE_OUTBOUND_TOKEN=

# --- System prompt loader ---

# Fuente: env | file | url
AGENT_SYSTEM_PROMPT_SOURCE=env

# Si source=env: contenido completo del prompt.
AGENT_SYSTEM_PROMPT=

# Si source=file: path al archivo (montado en el container o en disco).
# AGENT_SYSTEM_PROMPT_PATH=/data/system-prompt.md

# Si source=url: URL del prompt (ej. Google Drive público).
# AGENT_SYSTEM_PROMPT_URL=
# AGENT_SYSTEM_PROMPT_URL_AUTH=

# --- Existing nanoclaw upstream vars (no tocar a menos que sepas qué hacés) ---
# Ver nanoclaw upstream README/docs.
```

- [ ] **Step 2: `docs/bridge-eleve.md`**

```markdown
# ÉLEVÉ ↔ nanoclaw HTTP bridge contract

Documento del contrato entre ÉLEVÉ Supabase y nanoclaw, implementado por el `eleve-http` channel adapter.

## Inbound: ÉLEVÉ → nanoclaw

`POST {NANOCLAW_PUBLIC_URL}/messages`

```
Authorization: Bearer {AGENT_INBOUND_TOKEN}
Content-Type: application/json

{
  "conversation_id": "uuid de whatsapp_conversations",
  "message": "texto del usuario",
  "sender": { "phone": "5215512345678", "name": "Ana" },
  "metadata": { ... }
}
```

Response: `202 Accepted` (cuerpo vacío). Procesamiento asíncrono.

Auth: bearer estático compartido. Sin firma HMAC en v1.

Errores:
- `401 Unauthorized` — bearer ausente o inválido.
- `400 Bad Request` — body malformado / falta `conversation_id` o `message`.
- `405 Method Not Allowed` — para no-POST.

## Outbound: nanoclaw → ÉLEVÉ

`POST {ELEVE_OUTBOUND_URL}` (= `n8n-whatsapp-agent-response`).

```
Authorization: Bearer {ELEVE_OUTBOUND_TOKEN}
Content-Type: application/json

{
  "conversation_id": "...",
  "message": "respuesta del agente",
  "action": "escalate | transfer | close | schedule_followup",
  "metadata": { ... }
}
```

`action` es opcional; si está, ÉLEVÉ aplica las semánticas documentadas en `mcp-monica/mcp/_pipeline.md`.

Retry: el host de nanoclaw reintenta automáticamente si `deliver()` lanza error. Backoff lo maneja `src/delivery.ts`.

## Auto-wiring

La primera vez que un `conversation_id` aparece, el adapter crea automáticamente el `messaging_group` y lo wirea al agent group `monica`. No hay paso manual de "registrar conversación".

## Health

`GET /health` → `200 {"status":"ok"}`. No requiere auth. Lo usa el healthcheck del docker-compose.

## Sesión

Cada `conversation_id` distinto = una sesión nanoclaw distinta = pareja `inbound.db`/`outbound.db` propios. Memoria persistente automática por contacto.
```

- [ ] **Step 3: Commit**

```bash
git add .env.example docs/bridge-eleve.md
git commit -m "docs: env example and bridge-eleve contract"
```

---

### Task 8: Verificación end-to-end local

**Files:** ninguno; smoke test integral.

**Pre-condiciones:**
- `mcp-monica:dev` imagen Docker construida (Plan 2 Task 9 ya lo hizo).
- Host nanoclaw funcional con migrations corridas y `AGENT_INBOUND_TOKEN` configurado.

- [ ] **Step 1: Levantar mcp-monica en background**

```bash
docker run --rm -d \
  --name mcp-monica-e2e \
  -e SUPABASE_URL=https://example.supabase.co \
  -e SUPABASE_SERVICE_ROLE_KEY=fake-key \
  -p 3000:3000 \
  mcp-monica:dev
sleep 2
curl -s http://localhost:3000/health
echo ""
```

- [ ] **Step 2: Bootstrap del agent monica + arrancar nanoclaw host**

```bash
cd /home/morfi/eleve-nanoclaw/nanoclaw
NANOCLAW_DATA_DIR=/tmp/nanoclaw-e2e-$$ pnpm exec tsx scripts/init-monica-agent.ts
# Arranca el host (background)
NANOCLAW_DATA_DIR=/tmp/nanoclaw-e2e-$$ \
AGENT_INBOUND_TOKEN=e2e-token \
ELEVE_OUTBOUND_URL=http://localhost:9999/echo \
ELEVE_OUTBOUND_TOKEN=out-e2e \
NANOCLAW_PORT=3011 \
AGENT_SYSTEM_PROMPT_SOURCE=env \
AGENT_SYSTEM_PROMPT="You are Mónica, the receptionist." \
MCP_MONICA_URL=http://host.docker.internal:3000/mcp/sse \
pnpm run dev > /tmp/nanoclaw-e2e.log 2>&1 &
NANOCLAW_PID=$!
sleep 5
```

- [ ] **Step 3: Mock outbound endpoint y smoke POST**

```bash
# Server tonto que captura el outbound
node -e "require('http').createServer((req,res)=>{let b='';req.on('data',c=>b+=c);req.on('end',()=>{console.log('OUTBOUND:',req.url,b);res.writeHead(200);res.end('{}');});}).listen(9999)" &
ECHO_PID=$!

# POST inbound
curl -s -X POST http://localhost:3011/messages \
  -H "Authorization: Bearer e2e-token" \
  -H "Content-Type: application/json" \
  -d '{"conversation_id":"e2e-conv-1","message":"Hola Mónica","sender":{"phone":"5215512345678"}}'
echo ""

# Esperar el agente procesando
sleep 30

# Verificar que el outbound llegó
echo "=== nanoclaw log tail ==="
tail -30 /tmp/nanoclaw-e2e.log

# Cleanup
kill $NANOCLAW_PID $ECHO_PID 2>/dev/null
docker stop mcp-monica-e2e
wait 2>/dev/null
```

Expected: el server tonto en :9999 imprime el outbound POST con `conversation_id` y un `message` generado por el agente.

> Si el agente no responde por temas de OneCLI/secret mode, ese es el riesgo conocido (customizations.md sección G). El smoke valida la cadena hasta donde podamos sin Anthropic key real. Si tenés Anthropic key, podés dejar el agente respondiendo de verdad.

- [ ] **Step 4: Documentar el resultado del smoke en docs/bridge-eleve.md (sección "Verified")**

Agregar al final de `docs/bridge-eleve.md`:

```markdown

## Verified locally

Smoke test ejecutado el 2026-04-28 con:
- mcp-monica:dev en :3000
- nanoclaw host en :3011 (puerto del adapter)
- ÉLEVÉ Supabase mockeado en :9999

Resultado: <DONE | PARTIAL — describir qué piezas confirmamos y cuáles no>.
```

- [ ] **Step 5: Commit**

```bash
git add docs/bridge-eleve.md
git commit -m "docs(bridge-eleve): record local smoke test result"
```

---

### Task 9: Verificación final

- [ ] **Step 1: Tests host + container typecheck**

```bash
cd /home/morfi/eleve-nanoclaw/nanoclaw
pnpm test 2>&1 | tail -20
pnpm exec tsc --noEmit 2>&1 | tail -20
pnpm exec tsc -p container/agent-runner/tsconfig.json --noEmit 2>&1 | tail -20
```

Expected: nuestros tests passing; typecheck sin errores nuevos (errores upstream preexistentes son tolerables si no se relacionan con nuestros archivos).

- [ ] **Step 2: Listar archivos nuevos/modificados**

```bash
git diff --stat $(git rev-list HEAD --merges --first-parent | head -1)..HEAD 2>/dev/null || git log --oneline | head -15
```

Expected: ~9-12 commits del Plan 3, con paths que matchean la estructura propuesta.

- [ ] **Step 3: NO pushear**

El repo `devsBlockpoint/nanoclaw` ya existe en GitHub (clonamos de ahí). El push final va al cierre de Plan 4 después del go-no-go del usuario.

---

## Self-review

**Spec coverage:**

| Spec sección | Cubierto en plan |
|---|---|
| 4.1 Inbound POST /messages | Task 5 |
| 4.2 Outbound a n8n-whatsapp-agent-response | Task 5 (deliver) |
| 4.3 Identificación de sesión | Task 5 (platformId = conversation_id) + auto-wiring |
| 4.4 Async fire-and-forget | Task 5 (responde 202 inmediato) |
| 5 mcp-monica wiring | Task 1 (HTTP MCP support) + Task 2 (container.json) |
| 6 System prompt 3 fuentes | Task 4 |
| 7.3 Env vars resumidas | Task 7 |

**Placeholder scan:** ningún TBD. Algunos pasos delegan en `<el implementer ajusta>` cuando dependen del schema upstream real (DB function names, exact line numbers); esos están explícitamente reconocidos y son irreductibles sin leer el código real.

**Riesgos asumidos:**
- Auto-wiring desde adapter (boundary violation conocido — sección G de customizations.md).
- OneCLI secret mode no automatizable; documentado como paso manual en bootstrap.
- `assistantName` exacto = `Mónica`.
- Sin tests de auto-wiring por mockeo de DB (DB tests son integration; quedan a una iteración futura).

---

## Próximo plan

**Plan 4** — integración Easypanel + smoke end-to-end con WhatsApp real:
- Crear repos en GitHub (`devsBlockpoint/eleve-nanoclaw`, `devsBlockpoint/mcp-monica`).
- Push de los tres repos.
- Easypanel stack con docker-compose final + healthchecks + secrets.
- Configurar webhook en ÉLEVÉ Supabase para apuntar a la URL pública de nanoclaw.
- Smoke con un WhatsApp real → ver respuesta de Mónica.
