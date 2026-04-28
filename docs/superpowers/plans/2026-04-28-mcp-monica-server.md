# mcp-monica MCP Server Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implementar `mcp-monica` como MCP server HTTP que expone al LLM las edge functions de negocio de ÉLEVÉ. Lee `mcp/manifest.json` (ya existente) y por cada tool publica un handler MCP que proxea a la edge function en Supabase. Empaquetado en Docker.

**Architecture:** TypeScript + Node ≥ 20 + `@modelcontextprotocol/sdk`. Transport HTTP/SSE. Thin proxy stateless: cada tool call → POST a `${SUPABASE_URL}/functions/v1/${tool.edge_function}` con `Authorization: Bearer ${SUPABASE_SERVICE_ROLE_KEY}`. Errores HTTP mapeados a errores MCP. Sin lógica de negocio. Tests con Vitest. Endpoint `/health` para el healthcheck del compose.

**Tech Stack:** Node ≥ 20, TypeScript, `tsx` (ejecución TS sin compilar), `vitest` (test runner), `@modelcontextprotocol/sdk` (TS), `gray-matter` (ya usada en `_generator.ts`). Docker base image: `node:20-alpine`.

**Spec de referencia:** `docs/superpowers/specs/2026-04-28-eleve-nanoclaw-monica-design.md` sección 5.

**Working directory:** `/home/morfi/eleve-nanoclaw/mcp-monica/` (repo independiente `devsBlockpoint/mcp-monica`, branch `main`, user `devsBlockpoint <developers@blockpoint.cloud>`). Pre-existe el subdir `mcp/` con el registry de tools y el generador.

---

## Estructura de archivos

| Path | Responsabilidad |
|---|---|
| `package.json` | Node project, deps, scripts |
| `tsconfig.json` | Config TS estricto |
| `.gitignore` | Ignorar `node_modules`, `.env`, builds |
| `.dockerignore` | Ignorar `node_modules`, `.git`, tests |
| `README.md` | Cómo correr, agregar tools, dockerizar |
| `CLAUDE.md` | Índice DEV (no runtime) |
| `docs/tool-registry.md` | Cómo funciona el pipeline md→manifest→MCP |
| `docs/edge-functions-map.md` | Tabla tool ↔ edge_function ↔ tablas DB |
| `docs/auth.md` | Modelo de autenticación service_role + secretos |
| `docs/new-tool.md` | Walkthrough para agregar una tool nueva |
| `src/errors.ts` | Mapeo HTTP → MCP errors |
| `src/supabase-client.ts` | Cliente HTTP a edge functions |
| `src/tools-loader.ts` | Lee `mcp/manifest.json` y produce definiciones MCP |
| `src/server.ts` | Setup `@modelcontextprotocol/sdk` Server con handlers |
| `src/health.ts` | Endpoint `/health` (HTTP plano, fuera del MCP server) |
| `src/index.ts` | Entry: env loading, arranca MCP HTTP + health |
| `tests/errors.test.ts` | Tests del mapeo de errores |
| `tests/supabase-client.test.ts` | Tests con fetch mockeado |
| `tests/tools-loader.test.ts` | Tests con manifest fixture |
| `tests/server.test.ts` | Tests integración del server (list+call tool) |
| `Dockerfile` | Multi-stage build con node:20-alpine |

**Pre-existe (no crear, no modificar):**
- `mcp/README.md`, `mcp/_pipeline.md`, `mcp/_generator.ts`, `mcp/manifest.json`, `mcp/tools/*.md` — registry source of truth, ya documentado.

---

### Task 1: Inicializar proyecto Node + TypeScript

**Files:**
- Create: `mcp-monica/package.json`
- Create: `mcp-monica/tsconfig.json`
- Create: `mcp-monica/.gitignore`
- Create: `mcp-monica/.dockerignore`

- [ ] **Step 1: Crear `package.json`**

Crear `/home/morfi/eleve-nanoclaw/mcp-monica/package.json`:

```json
{
  "name": "mcp-monica",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "description": "MCP server that proxies the LLM to ÉLEVÉ Supabase Edge Functions",
  "scripts": {
    "start": "tsx src/index.ts",
    "dev": "tsx watch src/index.ts",
    "test": "vitest run",
    "test:watch": "vitest",
    "mcp:build": "tsx mcp/_generator.ts",
    "mcp:build:include-pending": "tsx mcp/_generator.ts -- --include-pending"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0",
    "gray-matter": "^4.0.3"
  },
  "devDependencies": {
    "@types/node": "^20.14.0",
    "tsx": "^4.19.0",
    "typescript": "^5.5.0",
    "vitest": "^2.1.0"
  },
  "engines": {
    "node": ">=20.0.0"
  }
}
```

- [ ] **Step 2: Crear `tsconfig.json`**

```json
{
  "compilerOptions": {
    "lib": ["ESNext"],
    "target": "ES2022",
    "module": "ESNext",
    "moduleDetection": "force",
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "noEmit": true,
    "strict": true,
    "skipLibCheck": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "types": ["node"]
  },
  "include": ["src/**/*", "tests/**/*", "mcp/**/*.ts"]
}
```

- [ ] **Step 3: Crear `.gitignore`**

```
node_modules/
.env
.env.local
*.log
dist/
build/
.DS_Store
tests/fixtures/*.tmp.json
```

- [ ] **Step 4: Crear `.dockerignore`**

```
node_modules
.git
.env
.env.local
*.log
tests/
*.test.ts
docs/
README.md
CLAUDE.md
.gitignore
.dockerignore
Dockerfile
```

- [ ] **Step 5: Instalar dependencias**

```bash
cd /home/morfi/eleve-nanoclaw/mcp-monica
npm install
```

Expected: `npm install` completa sin error. `node_modules/` y `package-lock.json` se crean.

- [ ] **Step 6: Verificar setup**

```bash
cd /home/morfi/eleve-nanoclaw/mcp-monica
node --version
test -f package.json && echo "package.json OK"
test -f tsconfig.json && echo "tsconfig.json OK"
test -d node_modules && echo "node_modules OK"
test -f package-lock.json && echo "lockfile OK"
```

Expected: 4 líneas "OK" + versión Node ≥ 20.

- [ ] **Step 7: Commit**

```bash
cd /home/morfi/eleve-nanoclaw/mcp-monica
git add package.json package-lock.json tsconfig.json .gitignore .dockerignore
git commit -m "chore: init Node + TypeScript project skeleton"
```

---

### Task 2: README.md y CLAUDE.md del proyecto

**Files:**
- Create: `mcp-monica/README.md`
- Create: `mcp-monica/CLAUDE.md`

- [ ] **Step 1: Escribir `README.md`**

```markdown
# mcp-monica

MCP server que expone al LLM (Mónica, vía nanoclaw) las edge functions de negocio de ÉLEVÉ alojadas en Supabase.

## Qué es y qué no es

**Es** un thin proxy: lee `mcp/manifest.json`, registra cada tool en el MCP server, y al ser invocada hace `POST` a la edge function correspondiente en Supabase con auth `service_role`.

**No es** lógica de negocio. La verdad del dominio (citas, pacientes, pagos) vive en Supabase Edge Functions y Postgres. Si una tool pide un comportamiento nuevo, el cambio va en la edge function, no acá.

## Quick start

```bash
cp ../.env.example .env
# completar SUPABASE_URL y SUPABASE_SERVICE_ROLE_KEY
npm install
npm run mcp:build
npm start
# server escuchando en MCP_PORT (default 3000)
curl http://localhost:3000/health  # → {"status":"ok"}
```

## Estructura

- `mcp/` — registry source of truth (markdown frontmatter + generator). Pre-existe; no se modifica desde acá.
- `src/` — implementación del MCP server.
- `tests/` — tests unitarios e integración con Vitest.
- `docs/` — guías de desarrollo (tool-registry, auth, agregar tool nueva).

## Cómo agregar una tool

Ver [`docs/new-tool.md`](docs/new-tool.md). Resumen: crear `mcp/tools/<nombre>.md` con frontmatter, correr `npm run mcp:build`, commitear ambos archivos.

## Contratos

- **Inbound (MCP)**: protocolo MCP estándar sobre HTTP/SSE. Los clientes (nanoclaw) descubren tools dinámicamente al conectar.
- **Outbound (Supabase)**: HTTPS POST a `${SUPABASE_URL}/functions/v1/${edge_function}` con `Authorization: Bearer ${SUPABASE_SERVICE_ROLE_KEY}` y `Content-Type: application/json`.

## Mapeo de errores

| HTTP | MCP |
|---|---|
| 2xx | Tool result OK |
| 4xx | MCP error con `code` y `message` del payload (visible al LLM para que reaccione) |
| 5xx | MCP error genérico ("Internal error"), sin filtrar detalles. Stack logueado solo del lado del server. |
| Timeout (>10s) | MCP error timeout |

## Variables de entorno

Ver `.env.example` en el monorepo raíz. Las relevantes para mcp-monica:

- `SUPABASE_URL` — URL del proyecto Supabase de ÉLEVÉ.
- `SUPABASE_SERVICE_ROLE_KEY` — secreto, NO commitear. Permite invocar edge functions con auth=service_role.
- `MCP_PORT` — puerto interno (default 3000). En docker-compose se accede desde la red `eleve-net`.

## Repo padre

Este repo es subproyecto del monorepo [`devsBlockpoint/eleve-nanoclaw`](https://github.com/devsBlockpoint/eleve-nanoclaw) (a crear). Para levantar todo el stack ÉLEVÉ con nanoclaw + mcp-monica + docker-compose, ver el README del monorepo.
```

- [ ] **Step 2: Escribir `CLAUDE.md`**

```markdown
# CLAUDE.md — mcp-monica (DEV index)

Entry point para Claude Code cuando trabajás dentro del subproyecto `mcp-monica`.

> Este `CLAUDE.md` es de desarrollo. mcp-monica NO carga ningún system prompt en runtime; es un servidor MCP, no un agente.

## Qué hay acá

```
mcp-monica/
├── README.md                # entry humano
├── CLAUDE.md                # este archivo
├── package.json             # Node + TS
├── Dockerfile
├── src/
│   ├── index.ts             # entry: env, arranca server
│   ├── server.ts            # MCP SDK wiring
│   ├── tools-loader.ts      # parse manifest, devuelve definiciones MCP
│   ├── supabase-client.ts   # POST a edge functions
│   ├── errors.ts            # HTTP → MCP errors
│   └── health.ts            # /health endpoint
├── tests/
│   ├── errors.test.ts
│   ├── supabase-client.test.ts
│   ├── tools-loader.test.ts
│   └── server.test.ts
├── mcp/                     # registry — preexistente, NO MODIFICAR estructura
│   ├── README.md            # formato del frontmatter
│   ├── _pipeline.md         # plumbing endpoints (no son tools)
│   ├── _generator.ts        # md → manifest.json
│   ├── manifest.json        # generado
│   └── tools/*.md           # tools (frontmatter source of truth)
└── docs/
    ├── tool-registry.md
    ├── edge-functions-map.md
    ├── auth.md
    └── new-tool.md
```

## Por dónde empezar según la tarea

| Tarea | Empezá por |
|---|---|
| Entender el server | [`docs/tool-registry.md`](docs/tool-registry.md), `src/server.ts`, `src/tools-loader.ts` |
| Agregar una tool nueva | [`docs/new-tool.md`](docs/new-tool.md) |
| Saber qué edge function llama cada tool | [`docs/edge-functions-map.md`](docs/edge-functions-map.md) |
| Cambiar manejo de errores | `src/errors.ts` + `tests/errors.test.ts` |
| Modificar transporte / wiring MCP | `src/server.ts`, `src/index.ts` |
| Tocar autenticación | [`docs/auth.md`](docs/auth.md), `src/supabase-client.ts` |

## Convenciones

- **Sin lógica de negocio acá**. La lógica vive en Supabase Edge Functions. Si una request requiere transformaciones de datos no triviales, eso va en la edge function, no en mcp-monica.
- **Naming de tools**: snake_case en español (LLM-facing, ej. `agendar_cita`). Edge function path: kebab-case en inglés (ej. `book-appointment`). Definido por el registry; este server lo respeta.
- **TDD**: cada módulo en `src/` tiene su test en `tests/`. Vitest (`npm test`).
- **Errores HTTP**: 2xx OK, 4xx visible al LLM (mapeado), 5xx genérico (no filtrar internals al LLM).
- **Conventional commits**: `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`.

## Repo padre

Este repo es subproyecto. El monorepo padre es `devsBlockpoint/eleve-nanoclaw` y aloja docs cross-proyecto, docker-compose, ADRs.
```

- [ ] **Step 3: Verificar y commitear**

```bash
cd /home/morfi/eleve-nanoclaw/mcp-monica
test -f README.md && test -f CLAUDE.md && echo "OK"
git add README.md CLAUDE.md
git commit -m "docs: add README and CLAUDE.md for mcp-monica subproject"
```

---

### Task 3: Docs del subproyecto

**Files:**
- Create: `mcp-monica/docs/tool-registry.md`
- Create: `mcp-monica/docs/edge-functions-map.md`
- Create: `mcp-monica/docs/auth.md`
- Create: `mcp-monica/docs/new-tool.md`

- [ ] **Step 1: Crear `docs/tool-registry.md`**

```markdown
# Tool Registry — pipeline md → manifest → MCP

El "registry" de mcp-monica vive en `mcp/` y es la **single source of truth** para qué tools expone el server.

## Pipeline

```
mcp/tools/<nombre>.md  (markdown con frontmatter)
       │
       │  npm run mcp:build
       ▼
mcp/_generator.ts  (parse, validate, filter)
       │
       ▼
mcp/manifest.json  (committed)
       │
       │  boot del server
       ▼
src/tools-loader.ts  (lee manifest)
       │
       ▼
src/server.ts  (registra cada tool en el MCP SDK)
```

## Frontmatter requerido

Documentado en [`mcp/README.md`](../mcp/README.md). Resumen:

- `tool_name` (snake_case ES, LLM-facing)
- `edge_function` (kebab-case EN, folder en `supabase/functions/`)
- `mcp_exposed` (bool)
- `description`
- `input_schema` (JSON Schema)
- `implementation_status` (`implemented` | `pending` | `blocked_on_payment_gateway` | `deprecated`)

## Filtros

`_generator.ts` excluye del manifest:
- Tools con `mcp_exposed: false`.
- Tools con `implementation_status` distinto de `implemented` (a menos que se invoque con `--include-pending`).

Por defecto, mcp-monica solo expone tools listas para producción. Las pending quedan documentadas pero no llamables.

## Validación al build

`_generator.ts` falla con código de salida ≠ 0 si:
- Falta un campo requerido del frontmatter.
- `tool_name` no es snake_case.
- Hay duplicados de `tool_name`.
- `implementation_status: implemented` pero `supabase/functions/<edge_function>/` no existe en el repo padre (asume layout monorepo Supabase preexistente; si no aplica en este monorepo, esa validación pasa solo cuando estés en el repo Supabase).

## Hot-reload (no, todavía)

`manifest.json` se bundlea en la imagen Docker. Cambiar tools = rebuild + redeploy. Si en el futuro necesitamos hot-reload (refetch del manifest cada N seg), se agrega como feature opcional sin breaking changes.
```

- [ ] **Step 2: Crear `docs/edge-functions-map.md`**

```markdown
# Edge Functions Map

Mapeo entre tools del MCP, edge functions de Supabase y tablas afectadas. **Source of truth: cada `mcp/tools/<nombre>.md`** (campo `related_db_tables` y `side_effects`). Este doc es resumen humano.

## Tools `implemented` (expuestas en runtime)

> Lista derivada de `mcp/manifest.json` actual. Para regenerar: `npm run mcp:build`.

| Tool (LLM) | Edge function | Tablas afectadas | Side effects |
|---|---|---|---|
| `agendar_cita` | `book-appointment` | `citas`, `pacientes`, `esteticistas` | INSERT pacientes (si nuevo), INSERT citas (Pendiente de Anticipo), auto-asigna esteticista |
| `buscar_disponibilidad` | `check-availability` | `citas`, `esteticistas` | Solo lectura |

## Tools `pending` (documentadas, no expuestas hasta migrar)

| Tool | Edge function | Estado |
|---|---|---|
| `cancel_appointment` | `cancel-appointment` | pending |
| `reschedule_appointment` | `reschedule-appointment` | pending |
| `get_appointments` | `get-appointments` | pending |
| `obtener_servicios` | `get-services` | pending |
| `get_treatment_detail` | `get-treatment-detail` | pending |
| `get_current_promotions` | `get-current-promotions` | pending |
| `search_patient` | `search-patient` | pending |
| `capture_lead_from_chat` | `capture-lead-from-chat` | pending |
| `send_payment_link` | `send-payment-link` | blocked_on_payment_gateway |
| `escalate_to_human` | `escalate-to-human` | pending |

## Cómo se llaman

mcp-monica hace siempre el mismo shape de request hacia Supabase:

```
POST ${SUPABASE_URL}/functions/v1/${edge_function}
Authorization: Bearer ${SUPABASE_SERVICE_ROLE_KEY}
Content-Type: application/json

<body = input validado por input_schema>
```

Si tu edge function usa `auth: service_role` (lo que asume mcp-monica), no necesita lógica de auth adicional — el JWT es válido y bypassa RLS.
```

- [ ] **Step 3: Crear `docs/auth.md`**

```markdown
# Auth — service_role y secretos

mcp-monica usa exclusivamente la **service_role key** de Supabase para invocar edge functions.

## Qué significa service_role

Es un JWT con privilegios elevados que bypassea Row Level Security (RLS). Las edge functions que validan `auth.role() === 'service_role'` aceptan estas requests.

**Implicaciones de seguridad:**
- mcp-monica con la service_role key puede leer/escribir cualquier tabla. Por eso es secreto crítico.
- mcp-monica NO está expuesto a Internet en producción. Solo es accesible desde la red interna de docker-compose por nanoclaw.
- Si un atacante consigue acceso a mcp-monica, tiene acceso al backend de ÉLEVÉ. Mitigaciones:
  - Container sin puertos expuestos al host (`expose:`, no `ports:`).
  - Firewall a nivel red (Easypanel maneja esto por defecto en stacks privados).
  - Rotación periódica de la key.

## Dónde vive el secreto

- **Dev local**: en `.env` del monorepo raíz, montado al container vía docker-compose. NUNCA commitear `.env`.
- **Easypanel**: en el panel de env vars del servicio mcp-monica. Encriptado at-rest.

## Qué NO usamos

- **anon key**: no la usamos. mcp-monica no expone funcionalidad pública.
- **JWT del usuario final**: el agente actúa como sistema, no como un usuario; la service_role es la representación correcta.

## Auditoría

Si necesitás trazar qué tool llamó qué edge function, los logs de Supabase Edge Functions registran cada invocación con timestamp y body. Del lado mcp-monica, cada tool call se loguea con `tool_name`, `edge_function`, status HTTP, y duración (sin loguear body por privacidad — los inputs pueden tener PII del paciente).
```

- [ ] **Step 4: Crear `docs/new-tool.md`**

```markdown
# Agregar una tool nueva — walkthrough

Pasos para exponer una nueva edge function al LLM.

## 1. ¿La edge function existe en Supabase?

- **Sí, ya implementada y testeada**: avanzá.
- **No**: implementala primero del lado Supabase. Este doc asume que ya hay un endpoint `POST ${SUPABASE_URL}/functions/v1/<nombre>` funcionando.

## 2. Crear `mcp/tools/<tool_name>.md`

Convención: `tool_name` en snake_case en español (ej. `obtener_servicios`).

```markdown
---
tool_name: obtener_servicios
edge_function: get-services
mcp_exposed: true
description: Lista los servicios estéticos disponibles en el catálogo.
input_schema:
  type: object
  properties:
    categoria:
      type: string
      description: "Filtrar por categoría (opcional)"
output_schema:
  type: object
  properties:
    servicios:
      type: array
      items:
        type: object
        properties:
          id: { type: string }
          nombre: { type: string }
          precio: { type: number }
side_effects: []
auth: service_role
implementation_status: implemented
related_db_tables: [servicios]
---

# obtener_servicios

Devuelve el catálogo de servicios. Si se pasa `categoria`, filtra.

## Cuándo invocar
- Usuario pregunta qué servicios hay
- Antes de `agendar_cita` si el usuario no especificó qué quiere
```

## 3. Regenerar el manifest

```bash
npm run mcp:build
```

Si el generator se queja de validación (campo faltante, edge_function no encontrada, etc.), corregí.

## 4. Test local

```bash
npm test                              # debería pasar; los tests están parametrizados sobre el manifest
npm start                             # arranca server
curl http://localhost:3000/health     # debe responder ok
```

Conectá un cliente MCP (ej. nanoclaw apuntando local) y verificá que la nueva tool aparece en `tools/list` y responde a `tools/call`.

## 5. Commit

```bash
git add mcp/tools/obtener_servicios.md mcp/manifest.json
git commit -m "feat(tools): expose obtener_servicios"
```

## 6. Build & deploy

Imagen Docker se reconstruye en CI/CD; redeploy de Easypanel toma el nuevo manifest sin que toques nada más.
```

- [ ] **Step 5: Verificar y commitear**

```bash
cd /home/morfi/eleve-nanoclaw/mcp-monica
ls docs/
test $(ls docs/*.md | wc -l) -eq 4 && echo "OK 4 docs"
git add docs/
git commit -m "docs(mcp-monica): add tool-registry, edge-functions-map, auth, new-tool guides"
```

---

### Task 4: `src/errors.ts` con TDD

**Files:**
- Create: `mcp-monica/tests/errors.test.ts`
- Create: `mcp-monica/src/errors.ts`

**Contract**: la función `mapHttpToMcpError(status, body)` toma un status HTTP y un body opcional (ya parseado a objeto JS o string), y devuelve un objeto `{ code: number, message: string, data?: unknown }` apto para retornar como error del MCP SDK. Reglas:

- Status 200-299 → no debería llamarse (lanzar `Error("not an error response")` si lo llamás con 2xx).
- Status 400-499 → `{ code: -32602 /* InvalidParams */, message: body.message ?? body.error ?? "Bad request", data: body }`.
- Status 500-599 → `{ code: -32603 /* InternalError */, message: "Internal error", data: undefined }` (NO filtrar body al LLM).
- "Timeout" (lo dispara el caller con status `0`) → `{ code: -32603, message: "Edge function timeout", data: undefined }`.

- [ ] **Step 1: Escribir el test (failing)**

`tests/errors.test.ts`:

```typescript
import { describe, test, expect } from "vitest";
import { mapHttpToMcpError } from "../src/errors.ts";

describe("mapHttpToMcpError", () => {
  test("4xx with message in body returns InvalidParams with body data", () => {
    const result = mapHttpToMcpError(400, { message: "Slot ya no disponible" });
    expect(result.code).toBe(-32602);
    expect(result.message).toBe("Slot ya no disponible");
    expect(result.data).toEqual({ message: "Slot ya no disponible" });
  });

  test("4xx with error field falls back to error", () => {
    const result = mapHttpToMcpError(400, { error: "Bad input" });
    expect(result.code).toBe(-32602);
    expect(result.message).toBe("Bad input");
  });

  test("4xx without message defaults to 'Bad request'", () => {
    const result = mapHttpToMcpError(404, {});
    expect(result.code).toBe(-32602);
    expect(result.message).toBe("Bad request");
  });

  test("5xx returns generic InternalError, hides body", () => {
    const result = mapHttpToMcpError(500, { stack: "secret stack" });
    expect(result.code).toBe(-32603);
    expect(result.message).toBe("Internal error");
    expect(result.data).toBeUndefined();
  });

  test("status 0 (timeout sentinel) returns timeout error", () => {
    const result = mapHttpToMcpError(0);
    expect(result.code).toBe(-32603);
    expect(result.message).toBe("Edge function timeout");
  });

  test("2xx throws (caller bug)", () => {
    expect(() => mapHttpToMcpError(200, { ok: true })).toThrow();
  });
});
```

- [ ] **Step 2: Run tests (must fail)**

```bash
cd /home/morfi/eleve-nanoclaw/mcp-monica
npx vitest run tests/errors.test.ts 2>&1 | head -40
```

Expected: errores tipo "Cannot find module '../src/errors.ts'" — los tests aún no encuentran la implementación.

- [ ] **Step 3: Implementar `src/errors.ts`**

```typescript
export interface McpErrorPayload {
  code: number;
  message: string;
  data?: unknown;
}

export function mapHttpToMcpError(status: number, body?: unknown): McpErrorPayload {
  if (status >= 200 && status < 300) {
    throw new Error(`mapHttpToMcpError called with success status ${status}`);
  }

  if (status === 0) {
    return { code: -32603, message: "Edge function timeout" };
  }

  if (status >= 400 && status < 500) {
    const b = (body && typeof body === "object" ? (body as Record<string, unknown>) : {}) as Record<string, unknown>;
    const message =
      (typeof b.message === "string" && b.message) ||
      (typeof b.error === "string" && b.error) ||
      "Bad request";
    return { code: -32602, message, data: body };
  }

  // 5xx and any other unexpected non-success status
  return { code: -32603, message: "Internal error" };
}
```

- [ ] **Step 4: Run tests (must pass)**

```bash
npx vitest run tests/errors.test.ts
```

Expected: 6 tests passing.

- [ ] **Step 5: Commit**

```bash
git add tests/errors.test.ts src/errors.ts
git commit -m "feat(errors): map HTTP status to MCP error payload"
```

---

### Task 5: `src/supabase-client.ts` con TDD

**Files:**
- Create: `mcp-monica/tests/supabase-client.test.ts`
- Create: `mcp-monica/src/supabase-client.ts`

**Contract**: la función `callEdgeFunction(config, name, input)` toma config `{ baseUrl, serviceRoleKey, timeoutMs? }`, un nombre de edge function (string), y un input arbitrario. Devuelve `{ ok: true, data: <body> }` o `{ ok: false, error: McpErrorPayload }`.

- POST a `${baseUrl}/functions/v1/${name}` con `Authorization: Bearer ${serviceRoleKey}`, `Content-Type: application/json`.
- Timeout default: 10000ms. Si excede, devuelve `{ ok: false, error: <timeout> }` (status 0 → mapHttpToMcpError).
- Body de respuesta: intentar parsear JSON; si falla, devolver `{}` o tratar como error según status.

Inyectamos `fetch` para tests (default: globalThis.fetch).

- [ ] **Step 1: Escribir el test (failing)**

`tests/supabase-client.test.ts`:

```typescript
import { describe, test, expect } from "vitest";
import { callEdgeFunction } from "../src/supabase-client.ts";

const baseConfig = {
  baseUrl: "https://example.supabase.co",
  serviceRoleKey: "test-key",
  timeoutMs: 1000,
};

function mockFetch(response: { status: number; body?: unknown; delay?: number }): typeof fetch {
  return (async (_input: RequestInfo | URL, _init?: RequestInit) => {
    if (response.delay) await new Promise((r) => setTimeout(r, response.delay));
    const bodyText = response.body === undefined ? "" : JSON.stringify(response.body);
    return new Response(bodyText, {
      status: response.status,
      headers: { "Content-Type": "application/json" },
    });
  }) as typeof fetch;
}

describe("callEdgeFunction", () => {
  test("returns ok:true with parsed body on 200", async () => {
    const fetchImpl = mockFetch({ status: 200, body: { success: true, cita: { id: "abc" } } });
    const result = await callEdgeFunction(
      { ...baseConfig, fetchImpl },
      "book-appointment",
      { nombre: "Ana" },
    );
    expect(result.ok).toBe(true);
    if (result.ok) {
      expect(result.data).toEqual({ success: true, cita: { id: "abc" } });
    }
  });

  test("posts to correct URL with auth header and JSON body", async () => {
    let capturedUrl = "";
    let capturedInit: RequestInit | undefined;
    const fetchImpl: typeof fetch = (async (input: RequestInfo | URL, init?: RequestInit) => {
      capturedUrl = typeof input === "string" ? input : input.toString();
      capturedInit = init;
      return new Response(JSON.stringify({ ok: 1 }), { status: 200 });
    }) as typeof fetch;

    await callEdgeFunction(
      { ...baseConfig, fetchImpl },
      "check-availability",
      { fecha: "2026-05-01" },
    );

    expect(capturedUrl).toBe("https://example.supabase.co/functions/v1/check-availability");
    expect(capturedInit?.method).toBe("POST");
    const headers = new Headers(capturedInit?.headers);
    expect(headers.get("Authorization")).toBe("Bearer test-key");
    expect(headers.get("Content-Type")).toBe("application/json");
    expect(capturedInit?.body).toBe(JSON.stringify({ fecha: "2026-05-01" }));
  });

  test("4xx returns ok:false with mapped MCP error and body data", async () => {
    const fetchImpl = mockFetch({ status: 400, body: { message: "Slot ya no disponible" } });
    const result = await callEdgeFunction(
      { ...baseConfig, fetchImpl },
      "book-appointment",
      {},
    );
    expect(result.ok).toBe(false);
    if (!result.ok) {
      expect(result.error.code).toBe(-32602);
      expect(result.error.message).toBe("Slot ya no disponible");
    }
  });

  test("5xx returns ok:false with generic InternalError", async () => {
    const fetchImpl = mockFetch({ status: 500, body: { stack: "secret" } });
    const result = await callEdgeFunction(
      { ...baseConfig, fetchImpl },
      "book-appointment",
      {},
    );
    expect(result.ok).toBe(false);
    if (!result.ok) {
      expect(result.error.code).toBe(-32603);
      expect(result.error.message).toBe("Internal error");
      expect(result.error.data).toBeUndefined();
    }
  });

  test("timeout returns ok:false with timeout error", async () => {
    const fetchImpl = mockFetch({ status: 200, body: { ok: 1 }, delay: 2000 });
    const result = await callEdgeFunction(
      { ...baseConfig, timeoutMs: 100, fetchImpl },
      "slow-function",
      {},
    );
    expect(result.ok).toBe(false);
    if (!result.ok) {
      expect(result.error.message).toBe("Edge function timeout");
    }
  });
});
```

- [ ] **Step 2: Run tests (must fail)**

```bash
npx vitest run tests/supabase-client.test.ts 2>&1 | head -20
```

Expected: module not found.

- [ ] **Step 3: Implementar `src/supabase-client.ts`**

```typescript
import { mapHttpToMcpError, type McpErrorPayload } from "./errors.ts";

export interface SupabaseClientConfig {
  baseUrl: string;
  serviceRoleKey: string;
  timeoutMs?: number;
  fetchImpl?: typeof fetch;
}

export type EdgeFunctionResult =
  | { ok: true; data: unknown }
  | { ok: false; error: McpErrorPayload };

const DEFAULT_TIMEOUT_MS = 10_000;

export async function callEdgeFunction(
  config: SupabaseClientConfig,
  name: string,
  input: unknown,
): Promise<EdgeFunctionResult> {
  const url = `${config.baseUrl.replace(/\/$/, "")}/functions/v1/${name}`;
  const fetchImpl = config.fetchImpl ?? fetch;
  const timeoutMs = config.timeoutMs ?? DEFAULT_TIMEOUT_MS;

  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeoutMs);

  let response: Response;
  try {
    response = await fetchImpl(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${config.serviceRoleKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(input),
      signal: controller.signal,
    });
  } catch (err) {
    clearTimeout(timeoutId);
    if (err instanceof Error && err.name === "AbortError") {
      return { ok: false, error: mapHttpToMcpError(0) };
    }
    return { ok: false, error: { code: -32603, message: "Network error" } };
  } finally {
    clearTimeout(timeoutId);
  }

  let body: unknown = undefined;
  const text = await response.text();
  if (text) {
    try {
      body = JSON.parse(text);
    } catch {
      body = text;
    }
  }

  if (response.ok) {
    return { ok: true, data: body };
  }

  return { ok: false, error: mapHttpToMcpError(response.status, body) };
}
```

- [ ] **Step 4: Run tests (must pass)**

```bash
npx vitest run tests/supabase-client.test.ts
```

Expected: 5 tests passing.

- [ ] **Step 5: Commit**

```bash
git add tests/supabase-client.test.ts src/supabase-client.ts
git commit -m "feat(client): HTTP client to Supabase Edge Functions with timeout"
```

---

### Task 6: `src/tools-loader.ts` con TDD

**Files:**
- Create: `mcp-monica/tests/tools-loader.test.ts`
- Create: `mcp-monica/tests/fixtures/manifest.fixture.json`
- Create: `mcp-monica/src/tools-loader.ts`

**Contract**: la función `loadTools(manifestPath)` lee un manifest JSON del filesystem y devuelve `Promise<Array<{ name: string; description: string; inputSchema: object; edgeFunction: string }>>`.

- Si el archivo no existe → throw con mensaje claro.
- Si `tools` no es array → throw.
- Si una entry no tiene `tool_name` o `edge_function` → throw mencionando el index.
- Devuelve solo las tools del manifest (no aplica filtro adicional; el filtro de `pending` ya lo hizo el generator).

- [ ] **Step 1: Crear fixture**

`tests/fixtures/manifest.fixture.json`:

```json
{
  "version": "1",
  "generated_at": "2026-04-28T00:00:00.000Z",
  "tools": [
    {
      "tool_name": "agendar_cita",
      "edge_function": "book-appointment",
      "description": "Crea una cita.",
      "input_schema": { "type": "object", "required": ["fecha"], "properties": { "fecha": { "type": "string" } } },
      "output_schema": null
    },
    {
      "tool_name": "buscar_disponibilidad",
      "edge_function": "check-availability",
      "description": "Devuelve slots libres.",
      "input_schema": { "type": "object", "properties": { "fecha": { "type": "string" } } },
      "output_schema": null
    }
  ]
}
```

- [ ] **Step 2: Escribir test**

`tests/tools-loader.test.ts`:

```typescript
import { describe, test, expect } from "vitest";
import { loadTools } from "../src/tools-loader.ts";
import { join, dirname } from "node:path";
import { fileURLToPath } from "node:url";
import { writeFile } from "node:fs/promises";

const __dirname = dirname(fileURLToPath(import.meta.url));
const FIXTURE = join(__dirname, "fixtures/manifest.fixture.json");

describe("loadTools", () => {
  test("returns parsed tools with normalized field names", async () => {
    const tools = await loadTools(FIXTURE);
    expect(tools).toHaveLength(2);
    expect(tools[0].name).toBe("agendar_cita");
    expect(tools[0].edgeFunction).toBe("book-appointment");
    expect(tools[0].description).toBe("Crea una cita.");
    expect(tools[0].inputSchema).toEqual({
      type: "object",
      required: ["fecha"],
      properties: { fecha: { type: "string" } },
    });
  });

  test("throws on missing file", async () => {
    await expect(loadTools("/tmp/does-not-exist.json")).rejects.toThrow();
  });

  test("throws on missing required field", async () => {
    const badPath = join(__dirname, "fixtures/bad-manifest.tmp.json");
    await writeFile(badPath, JSON.stringify({ tools: [{ description: "no name" }] }));
    await expect(loadTools(badPath)).rejects.toThrow(/tool_name|edge_function/);
  });
});
```

- [ ] **Step 3: Run tests (must fail)**

```bash
npx vitest run tests/tools-loader.test.ts 2>&1 | head -20
```

Expected: module not found.

- [ ] **Step 4: Implementar `src/tools-loader.ts`**

```typescript
import { readFile } from "node:fs/promises";

export interface ToolDefinition {
  name: string;
  description: string;
  inputSchema: object;
  edgeFunction: string;
}

interface ManifestToolEntry {
  tool_name?: string;
  edge_function?: string;
  description?: string;
  input_schema?: object;
}

interface Manifest {
  version?: string;
  tools?: ManifestToolEntry[];
}

export async function loadTools(manifestPath: string): Promise<ToolDefinition[]> {
  let raw: string;
  try {
    raw = await readFile(manifestPath, "utf8");
  } catch (err) {
    throw new Error(`Cannot read manifest at ${manifestPath}: ${(err as Error).message}`);
  }

  let manifest: Manifest;
  try {
    manifest = JSON.parse(raw);
  } catch (err) {
    throw new Error(`Manifest is not valid JSON: ${(err as Error).message}`);
  }

  if (!Array.isArray(manifest.tools)) {
    throw new Error(`Manifest 'tools' must be an array`);
  }

  return manifest.tools.map((entry, idx) => {
    if (!entry.tool_name || typeof entry.tool_name !== "string") {
      throw new Error(`Manifest entry [${idx}]: missing or invalid 'tool_name'`);
    }
    if (!entry.edge_function || typeof entry.edge_function !== "string") {
      throw new Error(`Manifest entry [${idx}] (${entry.tool_name}): missing or invalid 'edge_function'`);
    }
    return {
      name: entry.tool_name,
      description: entry.description ?? "",
      inputSchema: entry.input_schema ?? { type: "object" },
      edgeFunction: entry.edge_function,
    };
  });
}
```

- [ ] **Step 5: Run tests (must pass)**

```bash
npx vitest run tests/tools-loader.test.ts
```

Expected: 3 tests passing.

- [ ] **Step 6: Commit**

```bash
git add tests/tools-loader.test.ts tests/fixtures/manifest.fixture.json src/tools-loader.ts
git commit -m "feat(loader): parse manifest.json into MCP tool definitions"
```

---

### Task 7: `src/server.ts` con TDD

**Files:**
- Create: `mcp-monica/tests/server.test.ts`
- Create: `mcp-monica/src/server.ts`

**Contract**: la función `createMcpServer(deps)` toma `{ tools, callEdgeFn }` y devuelve un MCP `Server` configurado con handlers para `ListToolsRequestSchema` y `CallToolRequestSchema`. `callEdgeFn(name, input)` es un alias del cliente Supabase (inyectable para tests).

Lo testeamos invocando los handlers internos directamente, sin levantar HTTP — esa parte la cubre el index.

> **Nota sobre el SDK**: el `@modelcontextprotocol/sdk` exporta `Server` desde `@modelcontextprotocol/sdk/server/index.js` y los schemas desde `@modelcontextprotocol/sdk/types.js`. Si la API real difiere de lo escrito acá, ajustá imports y nombres preservando la semántica (handler para list y handler para call).

- [ ] **Step 1: Escribir test**

`tests/server.test.ts`:

```typescript
import { describe, test, expect, vi } from "vitest";
import { createMcpServer } from "../src/server.ts";
import type { ToolDefinition } from "../src/tools-loader.ts";

const sampleTools: ToolDefinition[] = [
  {
    name: "agendar_cita",
    description: "Crea cita",
    inputSchema: { type: "object", properties: { fecha: { type: "string" } } },
    edgeFunction: "book-appointment",
  },
];

describe("createMcpServer", () => {
  test("listTools returns tool definitions in MCP shape", async () => {
    const callEdgeFn = vi.fn(async () => ({ ok: true as const, data: {} }));
    const server = createMcpServer({ tools: sampleTools, callEdgeFn });
    const list = await server._handlers.listTools();
    expect(list.tools).toHaveLength(1);
    expect(list.tools[0].name).toBe("agendar_cita");
    expect(list.tools[0].description).toBe("Crea cita");
    expect(list.tools[0].inputSchema).toEqual({
      type: "object",
      properties: { fecha: { type: "string" } },
    });
  });

  test("callTool routes to callEdgeFn with correct edge function name and arguments", async () => {
    const callEdgeFn = vi.fn(async (_name: string, _input: unknown) => ({
      ok: true as const,
      data: { success: true, cita: { id: "x" } },
    }));
    const server = createMcpServer({ tools: sampleTools, callEdgeFn });
    const result = await server._handlers.callTool({
      params: { name: "agendar_cita", arguments: { fecha: "2026-05-01" } },
    });
    expect(callEdgeFn).toHaveBeenCalledWith("book-appointment", { fecha: "2026-05-01" });
    expect(result.content[0].type).toBe("text");
    expect(JSON.parse(result.content[0].text)).toEqual({ success: true, cita: { id: "x" } });
    expect(result.isError).toBeFalsy();
  });

  test("callTool returns isError:true when callEdgeFn returns ok:false", async () => {
    const callEdgeFn = vi.fn(async () => ({
      ok: false as const,
      error: { code: -32602, message: "Slot ya no disponible" },
    }));
    const server = createMcpServer({ tools: sampleTools, callEdgeFn });
    const result = await server._handlers.callTool({
      params: { name: "agendar_cita", arguments: {} },
    });
    expect(result.isError).toBe(true);
    expect(result.content[0].text).toContain("Slot ya no disponible");
  });

  test("callTool with unknown tool name returns isError:true", async () => {
    const callEdgeFn = vi.fn(async () => ({ ok: true as const, data: {} }));
    const server = createMcpServer({ tools: sampleTools, callEdgeFn });
    const result = await server._handlers.callTool({
      params: { name: "no_existe", arguments: {} },
    });
    expect(result.isError).toBe(true);
    expect(result.content[0].text).toContain("no_existe");
  });
});
```

> Nota: testeamos `server._handlers.listTools()` y `server._handlers.callTool(req)` — la implementación expone esos handlers como propiedad `_handlers` del objeto que devuelve `createMcpServer` para facilitar tests sin transport. La API pública del MCP server real se mantiene intacta.

- [ ] **Step 2: Run tests (must fail)**

```bash
npx vitest run tests/server.test.ts 2>&1 | head -20
```

Expected: module not found.

- [ ] **Step 3: Implementar `src/server.ts`**

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import type { ToolDefinition } from "./tools-loader.ts";
import type { EdgeFunctionResult } from "./supabase-client.ts";

export interface ServerDeps {
  tools: ToolDefinition[];
  callEdgeFn: (name: string, input: unknown) => Promise<EdgeFunctionResult>;
}

export interface McpServerHandle {
  server: Server;
  // Exposed for tests; do not use from production code paths
  _handlers: {
    listTools: () => Promise<{ tools: Array<{ name: string; description: string; inputSchema: object }> }>;
    callTool: (req: {
      params: { name: string; arguments?: Record<string, unknown> };
    }) => Promise<{
      content: Array<{ type: "text"; text: string }>;
      isError?: boolean;
    }>;
  };
}

export function createMcpServer(deps: ServerDeps): McpServerHandle {
  const server = new Server(
    { name: "mcp-monica", version: "0.1.0" },
    { capabilities: { tools: {} } },
  );

  const toolsByName = new Map(deps.tools.map((t) => [t.name, t]));

  const listTools: McpServerHandle["_handlers"]["listTools"] = async () => ({
    tools: deps.tools.map((t) => ({
      name: t.name,
      description: t.description,
      inputSchema: t.inputSchema,
    })),
  });

  const callTool: McpServerHandle["_handlers"]["callTool"] = async (req) => {
    const { name, arguments: args = {} } = req.params;
    const tool = toolsByName.get(name);
    if (!tool) {
      return {
        content: [{ type: "text", text: `Tool "${name}" not found` }],
        isError: true,
      };
    }

    const result = await deps.callEdgeFn(tool.edgeFunction, args);
    if (!result.ok) {
      return {
        content: [{ type: "text", text: result.error.message }],
        isError: true,
      };
    }
    return {
      content: [{ type: "text", text: JSON.stringify(result.data) }],
    };
  };

  server.setRequestHandler(ListToolsRequestSchema, listTools);
  server.setRequestHandler(CallToolRequestSchema, callTool);

  return { server, _handlers: { listTools, callTool } };
}
```

- [ ] **Step 4: Run tests (must pass)**

```bash
npx vitest run tests/server.test.ts
```

Expected: 4 tests passing.

> Si el SDK rechaza la firma o nombres de schemas, ajustá imports y firma del handler conservando la semántica. La estructura del handler (lista de tools, switch por name, JSON.stringify del resultado, isError en caso de fallo) se mantiene.

- [ ] **Step 5: Commit**

```bash
git add tests/server.test.ts src/server.ts
git commit -m "feat(server): wire MCP server with list/call tool handlers"
```

---

### Task 8: `src/health.ts` y `src/index.ts` (entry)

**Files:**
- Create: `mcp-monica/src/health.ts`
- Create: `mcp-monica/src/index.ts`

**Health endpoint** (siempre disponible en `GET /health`): el docker-compose lo usa para el healthcheck. Lo montamos como una ruta del mismo HTTP server que sirve MCP.

**index.ts**: lee env, carga manifest, instancia client, crea server, arranca HTTP server con rutas `/health` y `/mcp/sse`.

> **Nota sobre transport HTTP del MCP SDK**: `@modelcontextprotocol/sdk` provee `SSEServerTransport` para Server-Sent Events sobre HTTP. Toma el `(req, res)` de Node directamente. La implementación abajo lo usa. Si el SDK ofreciera un nuevo transport HTTP "streamable" más limpio, usalo y ajustá; lo que importa es que el server escucha en `MCP_PORT` y nanoclaw conecta con un cliente MCP HTTP.

- [ ] **Step 1: Crear `src/health.ts`**

```typescript
import { createServer } from "node:http";

export function startHealthServer(port: number): { stop: () => void } {
  const server = createServer((req, res) => {
    if (req.url === "/health") {
      res.writeHead(200, { "Content-Type": "application/json" });
      res.end(JSON.stringify({ status: "ok" }));
      return;
    }
    res.writeHead(404);
    res.end("Not Found");
  });
  server.listen(port);
  return {
    stop() {
      server.close();
    },
  };
}
```

- [ ] **Step 2: Crear `src/index.ts`**

```typescript
import { createServer, type IncomingMessage, type ServerResponse } from "node:http";
import { dirname, join } from "node:path";
import { fileURLToPath } from "node:url";
import { SSEServerTransport } from "@modelcontextprotocol/sdk/server/sse.js";
import { loadTools } from "./tools-loader.ts";
import { callEdgeFunction } from "./supabase-client.ts";
import { createMcpServer } from "./server.ts";
import { startHealthServer } from "./health.ts";

const __dirname = dirname(fileURLToPath(import.meta.url));

function requireEnv(name: string): string {
  const v = process.env[name];
  if (!v) {
    console.error(`Missing required env var: ${name}`);
    process.exit(1);
  }
  return v;
}

async function main() {
  const supabaseUrl = requireEnv("SUPABASE_URL");
  const serviceRoleKey = requireEnv("SUPABASE_SERVICE_ROLE_KEY");
  const port = Number(process.env.MCP_PORT ?? "3000");

  const manifestPath =
    process.env.MCP_MONICA_MANIFEST_PATH ?? join(__dirname, "..", "mcp", "manifest.json");

  const tools = await loadTools(manifestPath);
  console.log(`mcp-monica: loaded ${tools.length} tools from ${manifestPath}`);

  const callEdgeFn = (name: string, input: unknown) =>
    callEdgeFunction({ baseUrl: supabaseUrl, serviceRoleKey }, name, input);

  const { server: mcpServer } = createMcpServer({ tools, callEdgeFn });

  // Single HTTP server: /health for healthcheck, /mcp/sse for MCP transport.
  const httpServer = createServer(async (req: IncomingMessage, res: ServerResponse) => {
    const url = new URL(req.url ?? "/", `http://${req.headers.host ?? "localhost"}`);

    if (url.pathname === "/health") {
      res.writeHead(200, { "Content-Type": "application/json" });
      res.end(JSON.stringify({ status: "ok", tools: tools.length }));
      return;
    }

    if (url.pathname === "/mcp/sse") {
      const transport = new SSEServerTransport("/mcp/sse", res);
      await mcpServer.connect(transport);
      // SSEServerTransport keeps `res` open; do NOT call res.end() here.
      return;
    }

    res.writeHead(404);
    res.end("Not Found");
  });

  httpServer.listen(port, () => {
    console.log(`mcp-monica: listening on http://0.0.0.0:${port}`);
    console.log(`  health: GET  /health`);
    console.log(`  mcp:    GET  /mcp/sse`);
  });

  // startHealthServer kept as helper module for future split scenarios.
  void startHealthServer;

  // Graceful shutdown
  const shutdown = () => httpServer.close(() => process.exit(0));
  process.on("SIGTERM", shutdown);
  process.on("SIGINT", shutdown);
}

main().catch((err) => {
  console.error("Fatal:", err);
  process.exit(1);
});
```

> **Realismo**: la integración SSE con `node:http` y `SSEServerTransport` puede requerir un par de iteraciones para que el handshake funcione (el SDK MCP espera un `ServerResponse` de Node y headers SSE específicos). Si el implementer ve que el SDK no acepta `res` tal cual, ajustar — la idea es: server escucha en `port`, ruta `/mcp/sse` para el transport MCP, ruta `/health` para healthcheck.

- [ ] **Step 3: Smoke check — server arranca**

```bash
cd /home/morfi/eleve-nanoclaw/mcp-monica
SUPABASE_URL=https://example.supabase.co \
SUPABASE_SERVICE_ROLE_KEY=fake-test-key \
MCP_PORT=3000 \
MCP_MONICA_MANIFEST_PATH=$PWD/mcp/manifest.json \
timeout 3 npm start &
SERVER_PID=$!
sleep 1
curl -s http://localhost:3000/health
echo ""
kill $SERVER_PID 2>/dev/null
wait 2>/dev/null
```

Expected: respuesta `{"status":"ok","tools":<n>}` donde `<n>` es la cantidad de tools del manifest actual.

> Si ves errores de import del SDK MCP o de `SSEServerTransport`, abrí `node_modules/@modelcontextprotocol/sdk/` y buscá el path correcto del export. El plan documentó la API más probable; si el SDK la cambia, los nombres de los imports son lo único que se ajusta. Re-correr el smoke después.

- [ ] **Step 4: Commit**

```bash
git add src/health.ts src/index.ts
git commit -m "feat(server): index entry with /health and /mcp/sse routes"
```

---

### Task 9: Dockerfile

**Files:**
- Create: `mcp-monica/Dockerfile`

- [ ] **Step 1: Escribir Dockerfile**

```dockerfile
# syntax=docker/dockerfile:1.7
# ============================================================================
# mcp-monica — MCP server image
# ============================================================================

# Stage 1: install deps (production only)
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev --ignore-scripts

# Stage 2: install full deps (incl. tsx) for runtime TS execution
FROM node:20-alpine AS deps-full
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --ignore-scripts

# Stage 3: runtime
FROM node:20-alpine AS runtime
WORKDIR /app

# wget para el healthcheck (alpine ya lo trae como busybox applet, redundante pero explícito)
RUN apk add --no-cache wget

# Non-root user
RUN addgroup -S app && adduser -S app -G app

COPY --from=deps-full /app/node_modules ./node_modules
COPY package.json tsconfig.json ./
COPY src ./src
COPY mcp ./mcp

USER app

ENV MCP_PORT=3000
ENV NODE_ENV=production
EXPOSE 3000

# Healthcheck (también definido en compose; este es fallback)
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD wget --spider -q http://localhost:3000/health || exit 1

CMD ["npx", "tsx", "src/index.ts"]
```

> Nota: usamos `tsx` en runtime (sin compilar a JS) porque mcp-monica es chico y la latencia de boot es irrelevante. Si después querés imagen más liviana, agregás un step de `tsc --emit` y CMD `node dist/index.js`. Para v1, `tsx` mantiene el flujo idéntico a dev.

- [ ] **Step 2: Build local**

```bash
cd /home/morfi/eleve-nanoclaw/mcp-monica
docker build -t mcp-monica:dev .
```

Expected: build OK, imagen `mcp-monica:dev` creada. Tamaño esperado < 200 MB.

- [ ] **Step 3: Smoke run del container**

```bash
docker run --rm -d \
  --name mcp-monica-smoke \
  -e SUPABASE_URL=https://example.supabase.co \
  -e SUPABASE_SERVICE_ROLE_KEY=fake-key \
  -p 3000:3000 \
  mcp-monica:dev
sleep 2
curl -s http://localhost:3000/health
echo ""
docker stop mcp-monica-smoke
```

Expected: `{"status":"ok","tools":<n>}` y el container se detiene limpio.

- [ ] **Step 4: Commit**

```bash
git add Dockerfile
git commit -m "chore(docker): add multi-stage Node image"
```

---

### Task 10: Verificación final del subproyecto

**Files:** ninguno; solo checks.

- [ ] **Step 1: Tests completos**

```bash
cd /home/morfi/eleve-nanoclaw/mcp-monica
npm test
```

Expected: todos los tests passing (al menos `errors` 6, `supabase-client` 5, `tools-loader` 3, `server` 4 = 18+).

- [ ] **Step 2: Listar archivos del subproyecto**

```bash
cd /home/morfi/eleve-nanoclaw/mcp-monica
find . -type f \
  -not -path './node_modules/*' \
  -not -path './.git/*' \
  -not -path './tests/fixtures/bad-manifest.tmp.json' \
  | sort
```

Expected (orden alfabético):
```
./CLAUDE.md
./Dockerfile
./README.md
./.dockerignore
./.gitignore
./docs/auth.md
./docs/edge-functions-map.md
./docs/new-tool.md
./docs/tool-registry.md
./mcp/...        (preexistente, varios archivos)
./package.json
./src/errors.ts
./src/health.ts
./src/index.ts
./src/server.ts
./src/supabase-client.ts
./src/tools-loader.ts
./tests/errors.test.ts
./tests/fixtures/manifest.fixture.json
./tests/server.test.ts
./tests/supabase-client.test.ts
./tests/tools-loader.test.ts
./tsconfig.json
```

- [ ] **Step 3: Verificar git log**

```bash
git log --oneline
```

Expected: ~9-10 commits en este plan, todos con conventional commit messages.

- [ ] **Step 4: Limpiar fixture temporal del test**

El test de `loadTools` crea un archivo temporal `tests/fixtures/bad-manifest.tmp.json`. Verificar que no fue commiteado (debería estar en `.gitignore`?). Si no está y aparece en `git status`:

```bash
echo "tests/fixtures/bad-manifest.tmp.json" >> .gitignore
git add .gitignore
git commit -m "chore: ignore tmp test fixtures"
```

Si no aparece en git status, no hacer nada.

- [ ] **Step 5: NO pushear**

El repo `devsBlockpoint/mcp-monica` se crea en GitHub al final de plan 4. **NO ejecutar `gh repo create` ni `git push`** todavía.

- [ ] **Step 6: Reportar al controller**

Final summary. Reportar:
- ¿Pasaron todos los tests? (cantidad).
- ¿Builda el Docker image? (sí/no, tamaño).
- ¿Healthcheck responde correctamente con la cantidad esperada de tools? (sí/no).

---

## Self-review

**Spec coverage check** (spec sección 5):

| Spec sección | Cubierto en el plan |
|---|---|
| 5.1 Stack TS+Node+MCP SDK (cambio runtime: Node en lugar de Bun, ver task 0) | Task 1 (deps), Tasks 4-8 (impl) |
| 5.2 Estructura de archivos | Tasks 4-8 |
| 5.3 Boot (env, manifest, register, listen) | Task 8 (index.ts) |
| 5.4 Manifest bundled, no fetched | Dockerfile copia `mcp/`; Task 9 |
| 5.5 Mapeo de errores | Task 4 (errors.ts) + Task 5 (client) |
| 5.6 Tools `pending` excluidas | Cubierto por `_generator.ts` upstream; Task 6 (loader) lee solo lo que está en manifest |
| Healthcheck `/health` | Task 8 |
| Container sin puerto al host (en compose) | Cubierto por compose en plan 1 (`expose:`, no `ports:`) |
| Bearer service_role en outbound | Task 5 |
| Timeout HTTP (10s default) | Task 5 |

**Placeholder scan**: ninguno. Todos los archivos tienen contenido real.

**Type consistency**:
- `ToolDefinition` (de `tools-loader.ts`) usado por `server.ts` ✓.
- `EdgeFunctionResult` (de `supabase-client.ts`) usado por `server.ts` y consistente con tipo de retorno de `callEdgeFn` ✓.
- `McpErrorPayload` (de `errors.ts`) consumido por `supabase-client.ts` ✓.
- `MCP_PORT` env consistente entre `index.ts`, `Dockerfile`, `docker-compose.yml` (en plan 1).

**Riesgos conocidos** (intencionales):
- La API exacta del MCP SDK puede haber cambiado. El plan documenta la firma esperada; si el SDK difiere, ajustar imports y nombres conservando la semántica (lista de tools, call con name+args, isError en fallo).
- El SSE transport con `node:http` puede requerir adaptación. Plan menciona y deja flexibilidad.

---

## Próximos planes

Cuando este plan esté ejecutado:
- **Plan 3**: nanoclaw fork — clonar `qwibitai/nanoclaw`, agregar `eleve-http` channel adapter custom, implementar el system prompt loader 3-fuentes, conectar a este `mcp-monica` por HTTP.
- **Plan 4**: integración Easypanel + smoke test end-to-end con WhatsApp real.
