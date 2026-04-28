# Monorepo Scaffolding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Crear la estructura de documentación y orquestación a nivel raíz del monorepo `eleve-nanoclaw`, dejando lista la base sobre la que después se construyen mcp-monica (plan 2), nanoclaw fork (plan 3) e integración Easypanel (plan 4).

**Architecture:** El root del monorepo (`/home/morfi/eleve-nanoclaw/`) es un repo git independiente (`devsBlockpoint/eleve-nanoclaw`) que aloja docs cross-proyecto, ADRs, glosario de dominio, `.env.example` y `docker-compose.yml`. Los subproyectos (`nanoclaw/` y `mcp-monica/`) viven en sus propios repos y están en el `.gitignore` del root. Idioma de docs: español para dominio (mantiene ubiquitous language del negocio); inglés cuando es natural (DDD terms, edge function, etc.).

**Tech Stack:** Markdown puro, `.gitignore`, `docker-compose.yml` (skeleton), `.env.example` (skeleton). Sin código ejecutable.

**Spec de referencia:** `docs/superpowers/specs/2026-04-28-eleve-nanoclaw-monica-design.md`.

**Working directory:** `/home/morfi/eleve-nanoclaw/` (root del monorepo, ya inicializado como git repo con remote a `git@github-eleve:devsBlockpoint/eleve-nanoclaw.git`).

---

## Estructura de archivos a crear

| Path | Responsabilidad |
|---|---|
| `README.md` | Entry point humano del monorepo: qué es, cómo arrancar, cómo navegar |
| `CLAUDE.md` | Entry point de Claude Code: índice global a todos los docs |
| `docs/domain/ubiquitous-language.md` | Glosario del dominio ÉLEVÉ derivado del registry de tools |
| `docs/domain/bounded-contexts.md` | Identificación de los bounded contexts del sistema |
| `docs/domain/context-map.md` | Cómo se relacionan los contexts (upstream/downstream, ACL, etc.) |
| `docs/architecture/overview.md` | Vista de alto nivel con diagrama del flujo |
| `docs/architecture/decisions/0001-claude-agent-sdk.md` | ADR justificando la elección de nanoclaw + Claude Agent SDK |
| `.env.example` | Plantilla de variables de entorno consolidada |
| `docker-compose.yml` | Skeleton para levantar nanoclaw + mcp-monica en local |

**Diferidos a planes posteriores:**
- `docs/domain/domain-events.md` — se completa cuando exista el bus de eventos (no hay aún).
- `docs/architecture/data-flow.md` — duplica spec mientras no haya nuevo flujo concreto.
- `docs/architecture/deployment.md` — se materializa en plan 4 (Easypanel).
- `nanoclaw/CLAUDE.md`, `nanoclaw/README.md`, `nanoclaw/docs/*` — plan 3.
- `mcp-monica/CLAUDE.md`, `mcp-monica/README.md`, `mcp-monica/docs/*` — plan 2.

---

### Task 1: Crear `README.md` raíz

**Files:**
- Create: `README.md`

- [ ] **Step 1: Escribir el README**

Crear `/home/morfi/eleve-nanoclaw/README.md` con este contenido exacto:

```markdown
# eleve-nanoclaw

Monorepo de orquestación para la migración de **Mónica**, el agente conversacional de [ÉLEVÉ](https://eleve.app), desde la edge function `chat-assistant` (Supabase) hacia una arquitectura desacoplada en dos contenedores:

- **`nanoclaw/`** — agent runtime basado en [Claude Agent SDK](https://docs.claude.com/en/api/agent-sdk/overview), fork de [`qwibitai/nanoclaw`](https://github.com/qwibitai/nanoclaw). Aloja al agente "Mónica" con su CLAUDE.md y MCP. Repo: `devsBlockpoint/nanoclaw`.
- **`mcp-monica/`** — MCP server propio que expone como tools del LLM las edge functions de negocio que ya existen en Supabase (citas, pacientes, pagos, etc.). Repo: `devsBlockpoint/mcp-monica`.

Este repo aloja únicamente:

- `docs/` — documentación cross-proyecto (dominio DDD, arquitectura, ADRs, specs de diseño, planes de implementación).
- `docker-compose.yml` — orquestación para dev local.
- `.env.example` — plantilla de configuración.

Los subproyectos están en `.gitignore` porque viven en sus propios repos.

## Estado

En desarrollo (v1). Ver [`docs/superpowers/specs/2026-04-28-eleve-nanoclaw-monica-design.md`](docs/superpowers/specs/2026-04-28-eleve-nanoclaw-monica-design.md) para el diseño aprobado.

## Cómo arrancar (dev local)

Cada subproyecto se clona por separado en su carpeta:

```bash
git clone git@github-eleve:devsBlockpoint/nanoclaw.git
git clone git@github-eleve:devsBlockpoint/mcp-monica.git
cp .env.example .env
# completar variables
docker compose up -d
```

## Cómo navegar la documentación

- **Si sos humano nuevo en el proyecto:** empezá por [`docs/architecture/overview.md`](docs/architecture/overview.md).
- **Si sos Claude Code:** abrí [`CLAUDE.md`](CLAUDE.md), es el índice global.
- **Si querés entender el dominio del negocio:** [`docs/domain/ubiquitous-language.md`](docs/domain/ubiquitous-language.md).
- **Si querés ver decisiones técnicas:** [`docs/architecture/decisions/`](docs/architecture/decisions/).

## Repos relacionados

| Repo | Propósito |
|---|---|
| `devsBlockpoint/eleve-nanoclaw` (este) | Docs y orquestación |
| `devsBlockpoint/nanoclaw` | Agent runtime (fork de qwibitai/nanoclaw) |
| `devsBlockpoint/mcp-monica` | MCP server hacia Supabase Edge Functions |

## Licencia

Propietario — Blockpoint / ÉLEVÉ.
```

- [ ] **Step 2: Verificar que el archivo existe y tiene contenido esperado**

Run: `cat /home/morfi/eleve-nanoclaw/README.md | head -20`
Expected: muestra el header `# eleve-nanoclaw` y el primer párrafo.

- [ ] **Step 3: Commit**

```bash
cd /home/morfi/eleve-nanoclaw
git add README.md
git commit -m "docs: add root README with monorepo overview"
```

---

### Task 2: Crear `CLAUDE.md` raíz (índice global)

**Files:**
- Create: `CLAUDE.md`

- [ ] **Step 1: Escribir el CLAUDE.md**

Crear `/home/morfi/eleve-nanoclaw/CLAUDE.md` con este contenido exacto:

```markdown
# CLAUDE.md — Índice global del monorepo `eleve-nanoclaw`

Este archivo es el **entry point** para Claude Code cuando trabajás en este monorepo. Apunta a los docs relevantes según lo que estés haciendo.

> ⚠️ **Importante:** este `CLAUDE.md` es de desarrollo, NO es el system prompt runtime del agente Mónica. El system prompt runtime se carga en nanoclaw vía env (`AGENT_SYSTEM_PROMPT_SOURCE`). Ver [`docs/architecture/overview.md`](docs/architecture/overview.md).

## Qué hay en este repo

```
eleve-nanoclaw/
├── README.md                # entry humano
├── CLAUDE.md                # este archivo — índice para Claude Code
├── docker-compose.yml       # orquestación dev local
├── .env.example             # plantilla de configuración
├── docs/
│   ├── domain/              # modelo del dominio (DDD)
│   ├── architecture/        # decisiones técnicas y diagramas
│   └── superpowers/
│       ├── specs/           # specs de diseño aprobadas
│       └── plans/           # planes de implementación
├── nanoclaw/                # subproyecto (repo separado, ignorado por git aquí)
└── mcp-monica/              # subproyecto (repo separado, ignorado por git aquí)
```

## Por dónde empezar según la tarea

| Estoy trabajando en... | Empezá por |
|---|---|
| Entender qué es el sistema | [`README.md`](README.md), [`docs/architecture/overview.md`](docs/architecture/overview.md) |
| Entender el dominio (negocio) | [`docs/domain/ubiquitous-language.md`](docs/domain/ubiquitous-language.md), [`docs/domain/bounded-contexts.md`](docs/domain/bounded-contexts.md) |
| Cómo se conectan los componentes | [`docs/domain/context-map.md`](docs/domain/context-map.md) |
| Decisiones técnicas | [`docs/architecture/decisions/`](docs/architecture/decisions/) |
| El diseño completo aprobado | [`docs/superpowers/specs/2026-04-28-eleve-nanoclaw-monica-design.md`](docs/superpowers/specs/2026-04-28-eleve-nanoclaw-monica-design.md) |
| Modificar nanoclaw | navegar a `nanoclaw/` y leer `nanoclaw/CLAUDE.md` |
| Agregar/modificar tools del MCP | navegar a `mcp-monica/` y leer `mcp-monica/CLAUDE.md` |
| Variables de entorno | [`.env.example`](.env.example) |
| Levantar todo en local | [`docker-compose.yml`](docker-compose.yml) |

## Convenciones

- **Idioma de docs de dominio:** español (mantiene la ubiquitous language del negocio).
- **Idioma de docs de arquitectura/ADRs:** español con términos técnicos en inglés cuando es natural.
- **Naming de tools:** snake_case en español (LLM-facing, ej. `agendar_cita`); kebab-case en inglés para edge functions (ej. `book-appointment`).
- **Commits:** conventional commits (`feat:`, `fix:`, `docs:`, `chore:`, `refactor:`).

## Repos relacionados (no clonados aquí)

- [`devsBlockpoint/nanoclaw`](https://github.com/devsBlockpoint/nanoclaw)
- [`devsBlockpoint/mcp-monica`](https://github.com/devsBlockpoint/mcp-monica) (a crear)
```

- [ ] **Step 2: Verificar**

Run: `cat /home/morfi/eleve-nanoclaw/CLAUDE.md | wc -l`
Expected: línea count > 30.

- [ ] **Step 3: Commit**

```bash
cd /home/morfi/eleve-nanoclaw
git add CLAUDE.md
git commit -m "docs: add CLAUDE.md as global index for Claude Code"
```

---

### Task 3: Crear `docs/domain/ubiquitous-language.md`

**Files:**
- Create: `docs/domain/ubiquitous-language.md`

**Origen del contenido:** derivado de `mcp-monica/mcp/tools/*.md` (tablas `related_db_tables` + descripciones) y `mcp-monica/mcp/_pipeline.md`.

- [ ] **Step 1: Escribir el glosario**

Crear `/home/morfi/eleve-nanoclaw/docs/domain/ubiquitous-language.md` con este contenido exacto:

```markdown
# Ubiquitous Language — ÉLEVÉ

Glosario de términos del dominio. Cuando el código, las tools del MCP, los docs y las conversaciones del equipo usan estas palabras, deben referirse exactamente a lo definido aquí.

Convenciones:
- **Term** (inglés/técnico) → **Término** (español/dominio): definición.
- Cuando el LLM las usa (en español preferido para ÉLEVÉ), aparece **(LLM)**.

## Personas

- **Paciente / Patient** **(LLM: paciente)**: persona que recibe servicios estéticos en ÉLEVÉ. Identificada principalmente por su número de WhatsApp (`+10 dígitos`) y `id` en tabla `pacientes`. Si un mensaje entra de un WhatsApp no registrado, una tool puede crear el paciente automáticamente.
- **Esteticista / Aesthetician** **(LLM: esteticista)**: empleada que ejecuta los servicios. Tabla `esteticistas`. Tiene estado `Activa | Inactiva`. La asignación a una cita se hace auto: la esteticista activa con menos citas ese día.
- **Operador humano / Human operator**: persona del equipo de ÉLEVÉ que toma el control de una conversación cuando el agente escala. Vive del lado del CRM web.

## Conversación con el agente

- **Mónica**: nombre del agente conversacional. Antes vivía en la edge function `chat-assistant`; ahora vive en nanoclaw + mcp-monica.
- **Conversación / WhatsApp Conversation**: una sesión persistente entre un paciente y Mónica. Tabla `whatsapp_conversations`. Su `id` (UUID) es la **clave de sesión** para nanoclaw.
- **Mensaje / Message**: cada turno de la conversación. Tiene `sender_type` ∈ `{user, agent, human}` y queda persistido del lado de ÉLEVÉ.
- **`agent_mode`**: estado de control de la conversación.
  - `bot`: Mónica responde automáticamente.
  - `human`: un operador humano tomó el control; Mónica no responde aunque entre un mensaje.
  - `escalated`: solicitud activa de escalamiento, pendiente de que un humano la tome.
- **Auto-return-to-bot**: cron job (cada 5 min, `pg_cron`) que devuelve conversaciones de `human`/`escalated` a `bot` después de un período de inactividad. Edge function `auto-return-to-bot`.

## Servicios y agendamiento

- **Servicio / Service** **(LLM: servicio)**: tratamiento ofrecido (ej. limpieza facial). Tabla `servicios`.
- **Tratamiento / Treatment** **(LLM: tratamiento)**: visión más detallada de un servicio (descripción, contraindicaciones, duración, precio). Sirve para `get_treatment_detail`.
- **Cita / Appointment** **(LLM: cita)**: reserva concreta de un servicio para un paciente con una esteticista en una fecha y hora. Tabla `citas`. Estados:
  - `Pendiente de Anticipo`: recién creada, falta el pago del anticipo. Estado por defecto al llamar `agendar_cita`.
  - `Confirmada`: anticipo pagado, cita firme.
  - `Realizada`: el servicio se ejecutó.
  - `Cancelada`: el paciente o el sistema canceló.
  - `Reprogramada`: se movió de fecha/hora.
- **Slot / Slot de horario**: combinación válida de fecha + hora para una cita. La tool `buscar_disponibilidad` devuelve slots con cantidad de esteticistas disponibles.

## Pagos

- **Anticipo / Deposit**: pago previo a la cita que la mueve de `Pendiente de Anticipo` a `Confirmada`.
- **Payment link / Link de pago** **(LLM: link de pago)**: URL única que se manda al paciente por WhatsApp para que pague el anticipo. Generado por `send_payment_link`.

## Promociones y leads

- **Promoción / Promotion** **(LLM: promoción)**: campaña vigente con descuento o paquete. Tabla `promociones`. La tool `get_current_promotions` filtra activas por fecha.
- **Lead**: contacto interesado capturado en una conversación que aún no es paciente confirmado. La tool `capture_lead_from_chat` lo registra para seguimiento del equipo comercial.

## Escalación y control

- **Escalación / Escalation**: cambio de `agent_mode` a `human` o `escalated`. Tabla `escalation_queue`. Razones típicas: queja, tema legal, complejidad médica, sentiment muy negativo, pedido explícito.
- **Prioridad de escalación**: `low | medium | high | urgent`. Default: `medium`.

## Infraestructura conversacional

- **Channel adapter**: pieza dentro de nanoclaw que adapta un canal externo (WhatsApp en este caso, vía ÉLEVÉ) al modelo interno de inbound/outbound. Para esta v1 hay un solo channel adapter custom: `eleve-http`.
- **Agent group / Grupo de agente**: agrupación lógica de configuración de un agente en nanoclaw (CLAUDE.md, MCP, container config). En v1 hay uno solo: `monica`.
- **Sesión nanoclaw**: par `inbound.db` + `outbound.db` ligado a un `whatsapp_conversation_id`. Persistente; sobrevive a restarts del container.

## Edge functions del dominio (lo que el MCP llama)

Documentadas en `mcp-monica/mcp/tools/*.md`. Resumen:

| Edge function | Tool MCP (LLM) | Cuándo |
|---|---|---|
| `book-appointment` | `agendar_cita` | Confirmar cita después de elegir slot |
| `check-availability` | `buscar_disponibilidad` | Ver slots libres en una fecha |
| `cancel-appointment` | `cancel_appointment` | Cancelar cita existente |
| `reschedule-appointment` | `reschedule_appointment` | Mover cita a otra fecha/hora |
| `get-appointments` | `get_appointments` | Listar citas de un paciente |
| `get-services` | `obtener_servicios` | Catálogo de servicios |
| `get-treatment-detail` | `get_treatment_detail` | Detalle de un tratamiento |
| `get-current-promotions` | `get_current_promotions` | Promociones activas |
| `search-patient` | `search_patient` | Buscar paciente por WhatsApp |
| `capture-lead-from-chat` | `capture_lead_from_chat` | Registrar lead nuevo |
| `send-payment-link` | `send_payment_link` | Enviar link de pago al paciente |
| `escalate-to-human` | `escalate_to_human` | Pasar conversación a humano |

## Lo que NO es ubiquitous language

- Detalles de implementación de Supabase, Postgres, Vercel, Bun, Docker.
- Nombres de variables de entorno (esos están en `.env.example`).
- Términos del Claude Agent SDK (`Task tool`, `MCP`, etc.) — esos son lenguaje técnico, no de dominio.
```

- [ ] **Step 2: Verificar**

Run: `wc -l /home/morfi/eleve-nanoclaw/docs/domain/ubiquitous-language.md`
Expected: > 80 líneas.

- [ ] **Step 3: Commit**

```bash
cd /home/morfi/eleve-nanoclaw
git add docs/domain/ubiquitous-language.md
git commit -m "docs(domain): add ubiquitous language glossary"
```

---

### Task 4: Crear `docs/domain/bounded-contexts.md`

**Files:**
- Create: `docs/domain/bounded-contexts.md`

- [ ] **Step 1: Escribir el documento**

Crear `/home/morfi/eleve-nanoclaw/docs/domain/bounded-contexts.md` con este contenido exacto:

```markdown
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

**Tecnología:** TypeScript + Bun + `@modelcontextprotocol/sdk`. Sin estado.

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
```

- [ ] **Step 2: Verificar**

Run: `grep -c "^## " /home/morfi/eleve-nanoclaw/docs/domain/bounded-contexts.md`
Expected: 5 (4 contexts + relación con frontend).

- [ ] **Step 3: Commit**

```bash
cd /home/morfi/eleve-nanoclaw
git add docs/domain/bounded-contexts.md
git commit -m "docs(domain): identify bounded contexts"
```

---

### Task 5: Crear `docs/domain/context-map.md`

**Files:**
- Create: `docs/domain/context-map.md`

- [ ] **Step 1: Escribir el documento**

Crear `/home/morfi/eleve-nanoclaw/docs/domain/context-map.md` con este contenido exacto:

````markdown
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
````

- [ ] **Step 2: Verificar**

Run: `grep -c "Patrón:" /home/morfi/eleve-nanoclaw/docs/domain/context-map.md`
Expected: 4.

- [ ] **Step 3: Commit**

```bash
cd /home/morfi/eleve-nanoclaw
git add docs/domain/context-map.md
git commit -m "docs(domain): add context map with DDD patterns"
```

---

### Task 6: Crear `docs/architecture/overview.md`

**Files:**
- Create: `docs/architecture/overview.md`

- [ ] **Step 1: Escribir el documento**

Crear `/home/morfi/eleve-nanoclaw/docs/architecture/overview.md` con este contenido exacto:

````markdown
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
mcp-monica container (Bun + MCP SDK)
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
| mcp-monica | TypeScript + Bun | Mismo runtime que nanoclaw; thin proxy no necesita Python |
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
````

- [ ] **Step 2: Verificar**

Run: `head -3 /home/morfi/eleve-nanoclaw/docs/architecture/overview.md`
Expected: muestra `# Architecture Overview`.

- [ ] **Step 3: Commit**

```bash
cd /home/morfi/eleve-nanoclaw
git add docs/architecture/overview.md
git commit -m "docs(arch): add high-level architecture overview"
```

---

### Task 7: Crear ADR `0001-claude-agent-sdk.md`

**Files:**
- Create: `docs/architecture/decisions/0001-claude-agent-sdk.md`

- [ ] **Step 1: Crear el ADR**

Crear `/home/morfi/eleve-nanoclaw/docs/architecture/decisions/0001-claude-agent-sdk.md` con este contenido exacto:

```markdown
# 0001 — Usar Claude Agent SDK (vía nanoclaw) en lugar de SDK plano de Anthropic

**Fecha:** 2026-04-28
**Estado:** Aceptada
**Decisores:** emorfin (Cyborgdev / ÉLEVÉ), Claude Code

## Contexto

ÉLEVÉ ya tiene un agente conversacional ("Mónica") implementado como una edge function de Supabase (`chat-assistant`) que carga un `manifest.json` de tools y orquesta llamadas al LLM. Funciona, pero acumula limitaciones:

1. **Acoplamiento al stack Supabase**: el agente vive como edge function Deno; agregar capacidades (memoria persistente, MCP, sub-agentes, herramientas filesystem) implica reinventar primitivas que el SDK ya resuelve.
2. **Dificultad para evolucionar el comportamiento del agente**: cambiar el system prompt o agregar tools requiere redeploy de edge function.
3. **Rigidez frente a multi-tenancy**: ÉLEVÉ podría querer en el futuro agentes con personalidades distintas (ventas vs soporte vs general). El esquema actual acopla todo en una sola función.

Necesitamos un agente con:
- Memoria persistente por conversación.
- Capacidad de invocar tools externas vía MCP.
- System prompt configurable sin redeploy.
- Aislamiento por agent group para futuras extensiones.
- Capacidad de invocar sub-agentes (Task tool).

## Opciones consideradas

### Opción A — SDK plano de Anthropic (`anthropic` SDK)

- Implementamos nosotros el agent loop, manejo de tool use, memoria, MCP client, etc.
- **Pros:** control total, dependencias mínimas.
- **Contras:** reinventamos lo que ya existe; semanas de trabajo de plumbing antes de funcionalidad de negocio; mantenimiento perpetuo.

### Opción B — Claude Agent SDK directo

- Usar `@anthropic-ai/claude-agent-sdk` y construir nuestro propio runtime.
- **Pros:** trae agent loop, MCP support, file tools, hooks, subagents listos.
- **Contras:** seguimos teniendo que armar la capa HTTP, channel adapters, persistencia, dockerización. Otro montón de plumbing.

### Opción C — Fork de un proyecto que ya use el Agent SDK

Investigando, encontramos [`qwibitai/nanoclaw`](https://github.com/qwibitai/nanoclaw): proyecto open source MIT que ya provee:
- Agent runtime containerizado con Claude Agent SDK.
- Modelo de "agent groups" con CLAUDE.md por grupo.
- Channel adapters extensibles (skills `/add-whatsapp`, `/add-telegram`, etc.).
- Sub-agentes vía Task tool.
- SQLite por sesión, sweep, cron, todo el plumbing.

Filosofía explícita: "small enough to understand, customization = code changes". Ese mindset encaja con que necesitamos forkear y modificar.

- **Pros:** reusamos meses de trabajo; nos enfocamos solo en la integración con ÉLEVÉ; codebase chico (~handful de files), entendible.
- **Contras:** dependencia de proyecto upstream; tracking de updates manual; nos obligamos a TS/Bun (perdemos opción de Python); customizaciones nuestras viven en fork (potencial drift).

## Decisión

**Opción C: forkear `qwibitai/nanoclaw` y customizar.**

Motivos:
1. Maximiza reuso de código probado.
2. La filosofía del upstream (skills + customization) coincide con nuestras necesidades.
3. La cantidad de customización necesaria (channel adapter custom para ÉLEVÉ + system prompt loader configurable) es acotada.
4. TS/Bun no es bloqueante; el ecosistema MCP en TS es maduro.

## Consecuencias

**Positivas:**
- Time-to-first-message dramáticamente más corto.
- Los avances upstream (mejoras al SDK, providers nuevos, etc.) los podemos rebasear si nos sirven.
- mcp-monica como container separado es trivial — nanoclaw acepta MCP servers HTTP directos.

**Negativas:**
- Dependemos de la salud del proyecto upstream. Mitigación: fork es nuestro; podemos pinear o divergir si hace falta.
- Cualquier customización nuestra en archivos del trunk genera fricción al rebase. Mitigación: documentar claramente en `nanoclaw/docs/customizations.md` qué se modificó vs upstream.
- Si en el futuro queremos cambiar a Python o a otro stack, hay costo de migración.

## Notas

- El system prompt runtime NO vive en `groups/<grupo>/CLAUDE.md` del fork como hace upstream. Lo cargamos por env (`AGENT_SYSTEM_PROMPT_SOURCE`) para evitar rebuilds en cada cambio de prompt y permitir editarlo desde Google Drive. Ver spec sección 6.
- Los channel adapters upstream (Telegram, Discord, etc.) se quedan pero no los usamos. nanoclaw expone solo el adapter custom `eleve-http`.
```

- [ ] **Step 2: Verificar**

Run: `grep "## Decisión" /home/morfi/eleve-nanoclaw/docs/architecture/decisions/0001-claude-agent-sdk.md`
Expected: una línea con `## Decisión`.

- [ ] **Step 3: Commit**

```bash
cd /home/morfi/eleve-nanoclaw
git add docs/architecture/decisions/0001-claude-agent-sdk.md
git commit -m "docs(adr): 0001 use Claude Agent SDK via nanoclaw fork"
```

---

### Task 8: Crear `.env.example`

**Files:**
- Create: `.env.example`

- [ ] **Step 1: Escribir el `.env.example`**

Crear `/home/morfi/eleve-nanoclaw/.env.example` con este contenido exacto:

```dotenv
# =============================================================================
# eleve-nanoclaw — variables de entorno
# =============================================================================
# Copiá este archivo a .env y completá con valores reales antes de levantar
# `docker compose up`. NO commitear .env (está en .gitignore).
# =============================================================================


# -----------------------------------------------------------------------------
# nanoclaw (agent runtime)
# -----------------------------------------------------------------------------

# Puerto en el que nanoclaw expone su HTTP API (POST /messages, GET /health)
NANOCLAW_PORT=3001

# Bearer token que ÉLEVÉ envía al llamar a POST /messages.
# Reusa el token que hoy ÉLEVÉ envía a n8n / chat-assistant para no tocar
# nada del lado Supabase.
AGENT_INBOUND_TOKEN=

# Credenciales de Anthropic. Usar UNA de las dos opciones.
# Opción 1: API key directa
ANTHROPIC_API_KEY=
# Opción 2: vía proxy (ej. OneCLI Agent Vault)
# ANTHROPIC_AUTH_TOKEN=
# ANTHROPIC_BASE_URL=

# Agent group único en v1
AGENT_GROUP=monica

# Sistema de carga del system prompt. Elegir UNA fuente:
#   env  → leer de AGENT_SYSTEM_PROMPT (string completo)
#   file → leer del archivo en AGENT_SYSTEM_PROMPT_PATH (mount de volumen)
#   url  → fetch desde AGENT_SYSTEM_PROMPT_URL (opcional auth + cache)
AGENT_SYSTEM_PROMPT_SOURCE=env

# Si AGENT_SYSTEM_PROMPT_SOURCE=env:
AGENT_SYSTEM_PROMPT=

# Si AGENT_SYSTEM_PROMPT_SOURCE=file:
# AGENT_SYSTEM_PROMPT_PATH=/data/system-prompt.md

# Si AGENT_SYSTEM_PROMPT_SOURCE=url:
# AGENT_SYSTEM_PROMPT_URL=
# AGENT_SYSTEM_PROMPT_URL_AUTH=        # opcional, ej. "Bearer xxx"

# Hot-reload del system prompt (segundos). 0 = off (default).
AGENT_SYSTEM_PROMPT_RELOAD_INTERVAL=0

# URL interna del MCP server mcp-monica (red de docker compose)
MCP_MONICA_URL=http://mcp-monica:3000

# Endpoint y bearer para enviar respuestas hacia ÉLEVÉ
ELEVE_OUTBOUND_URL=https://YOUR_PROJECT.supabase.co/functions/v1/n8n-whatsapp-agent-response
ELEVE_OUTBOUND_TOKEN=


# -----------------------------------------------------------------------------
# mcp-monica (MCP server)
# -----------------------------------------------------------------------------

# Puerto interno del MCP server (NO se expone al host en producción)
MCP_MONICA_PORT=3000

# URL del proyecto Supabase de ÉLEVÉ
SUPABASE_URL=https://YOUR_PROJECT.supabase.co

# Service role key — permite invocar edge functions con auth=service_role
# CUIDADO: secreto crítico. Solo en .env, nunca en repo.
SUPABASE_SERVICE_ROLE_KEY=
```

- [ ] **Step 2: Verificar**

Run: `grep -c "^[A-Z_]*=" /home/morfi/eleve-nanoclaw/.env.example`
Expected: número >= 10 (vars no comentadas).

- [ ] **Step 3: Commit**

```bash
cd /home/morfi/eleve-nanoclaw
git add .env.example
git commit -m "chore: add .env.example with all configurable vars"
```

---

### Task 9: Crear `docker-compose.yml` skeleton

**Files:**
- Create: `docker-compose.yml`

- [ ] **Step 1: Escribir el compose**

Crear `/home/morfi/eleve-nanoclaw/docker-compose.yml` con este contenido exacto:

```yaml
# =============================================================================
# eleve-nanoclaw — orquestación dev local
# =============================================================================
# Levanta nanoclaw + mcp-monica conectados por red interna.
# En producción (Easypanel), este mismo archivo sirve como referencia.
#
# Pre-requisitos:
#   - Repos clonados:
#       git clone git@github-eleve:devsBlockpoint/nanoclaw.git
#       git clone git@github-eleve:devsBlockpoint/mcp-monica.git
#   - .env configurado (cp .env.example .env y completar)
# =============================================================================

services:

  nanoclaw:
    build:
      context: ./nanoclaw
      dockerfile: Dockerfile
    container_name: eleve-nanoclaw
    restart: unless-stopped
    ports:
      - "${NANOCLAW_PORT:-3001}:3001"
    environment:
      AGENT_INBOUND_TOKEN: ${AGENT_INBOUND_TOKEN}
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY:-}
      ANTHROPIC_AUTH_TOKEN: ${ANTHROPIC_AUTH_TOKEN:-}
      ANTHROPIC_BASE_URL: ${ANTHROPIC_BASE_URL:-}
      AGENT_GROUP: ${AGENT_GROUP:-monica}
      AGENT_SYSTEM_PROMPT_SOURCE: ${AGENT_SYSTEM_PROMPT_SOURCE:-env}
      AGENT_SYSTEM_PROMPT: ${AGENT_SYSTEM_PROMPT:-}
      AGENT_SYSTEM_PROMPT_PATH: ${AGENT_SYSTEM_PROMPT_PATH:-}
      AGENT_SYSTEM_PROMPT_URL: ${AGENT_SYSTEM_PROMPT_URL:-}
      AGENT_SYSTEM_PROMPT_URL_AUTH: ${AGENT_SYSTEM_PROMPT_URL_AUTH:-}
      AGENT_SYSTEM_PROMPT_RELOAD_INTERVAL: ${AGENT_SYSTEM_PROMPT_RELOAD_INTERVAL:-0}
      MCP_MONICA_URL: ${MCP_MONICA_URL:-http://mcp-monica:3000}
      ELEVE_OUTBOUND_URL: ${ELEVE_OUTBOUND_URL}
      ELEVE_OUTBOUND_TOKEN: ${ELEVE_OUTBOUND_TOKEN}
      PORT: "3001"
    volumes:
      - nanoclaw-data:/data
    depends_on:
      mcp-monica:
        condition: service_healthy
    networks:
      - eleve-net
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:3001/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 30s

  mcp-monica:
    build:
      context: ./mcp-monica
      dockerfile: Dockerfile
    container_name: eleve-mcp-monica
    restart: unless-stopped
    expose:
      - "3000"
    environment:
      SUPABASE_URL: ${SUPABASE_URL}
      SUPABASE_SERVICE_ROLE_KEY: ${SUPABASE_SERVICE_ROLE_KEY}
      MCP_PORT: "3000"
    networks:
      - eleve-net
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 15s

networks:
  eleve-net:
    driver: bridge

volumes:
  nanoclaw-data:
    driver: local
```

- [ ] **Step 2: Validar el YAML (syntax)**

Run: `docker compose -f /home/morfi/eleve-nanoclaw/docker-compose.yml config --quiet 2>&1 || echo "YAML valid pero servicios no buildables aún (esperado)"`
Expected: o muestra el output del compose, o avisa que los Dockerfiles no existen aún (esto es esperado — los crearemos en planes 2 y 3). El YAML en sí debe parsear sin errores.

> Nota: si `docker compose config` se queja de que `./nanoclaw` o `./mcp-monica` no existen, está bien — esos serán llenados por planes posteriores. Para pasar el linter ahora, podés stubbear creando carpetas vacías con `Dockerfile` placeholder, pero NO es necesario para esta tarea. Lo importante es que el YAML parse.

- [ ] **Step 3: Commit**

```bash
cd /home/morfi/eleve-nanoclaw
git add docker-compose.yml
git commit -m "chore: add docker-compose skeleton for nanoclaw + mcp-monica"
```

---

### Task 10: Verificación final del scaffold

**Files:** ninguno nuevo, solo lectura.

- [ ] **Step 1: Listar el árbol final**

Run:
```bash
cd /home/morfi/eleve-nanoclaw
find . -type f \
  -not -path './nanoclaw/*' \
  -not -path './mcp-monica/*' \
  -not -path './.git/*' \
  -not -path './.claude/*' \
  | sort
```

Expected output (orden alfabético):
```
./.env.example
./.gitignore
./CLAUDE.md
./README.md
./docker-compose.yml
./docs/architecture/decisions/0001-claude-agent-sdk.md
./docs/architecture/overview.md
./docs/domain/bounded-contexts.md
./docs/domain/context-map.md
./docs/domain/ubiquitous-language.md
./docs/superpowers/plans/2026-04-28-monorepo-scaffolding.md
./docs/superpowers/specs/2026-04-28-eleve-nanoclaw-monica-design.md
```

- [ ] **Step 2: Verificar git log**

Run: `cd /home/morfi/eleve-nanoclaw && git log --oneline`
Expected: ~10 commits (1 inicial del spec + 9 de este plan).

- [ ] **Step 3: Verificar que el repo no incluyó accidentalmente subproyectos**

Run: `cd /home/morfi/eleve-nanoclaw && git ls-files | grep -E "^(nanoclaw|mcp-monica)/" | head`
Expected: vacío (nada).

- [ ] **Step 4: Verificar enlaces relativos en CLAUDE.md y README.md no apuntan a archivos inexistentes**

Run:
```bash
cd /home/morfi/eleve-nanoclaw
for link in $(grep -oE '\[.*\]\([^)]+\.md\)' CLAUDE.md README.md docs/**/*.md 2>/dev/null | grep -oE '\([^)]+\.md\)' | tr -d '()' | sort -u); do
  if [ ! -f "$link" ] && [ ! -f "${link#./}" ]; then
    echo "BROKEN: $link"
  fi
done
echo "---done---"
```

Expected: solo la línea `---done---` (sin `BROKEN:` antes). Los links a docs futuros (`nanoclaw/CLAUDE.md`, `mcp-monica/CLAUDE.md`) son ABSOLUTOS al repo respectivo y no se chequean aquí; solo chequeamos enlaces internos del root repo.

> Si aparece `BROKEN:` para un link interno, abrir el archivo y corregir.

- [ ] **Step 5: Push (opcional, solo si el repo ya existe en GitHub)**

El repo `devsBlockpoint/eleve-nanoclaw` se crea en GitHub al final de plan 4. **NO pushear todavía**. El commit local queda; el push se hace después de crear el repo en `gh`.

```bash
# NO ejecutar todavía:
# cd /home/morfi/eleve-nanoclaw
# gh repo create devsBlockpoint/eleve-nanoclaw --private --source=. --push
```

- [ ] **Step 6: Marcar tarea de scaffolding como completada en TaskList del workflow**

(Solo aplica si se está usando TaskList de Claude Code. No hay acción de archivo.)

---

## Self-review

**Spec coverage check:**

| Spec sección | Cubierto en el plan |
|---|---|
| 1. Objetivo | README + overview |
| 2. Contexto | bounded-contexts + ubiquitous-language |
| 3. Arquitectura | overview + ADR 0001 |
| 4. Bridge ÉLEVÉ ↔ nanoclaw | context-map (responsabilidad), spec referenciado |
| 5. mcp-monica | bounded-contexts + context-map (responsabilidad). Implementación → plan 2 |
| 6. System prompt 3 fuentes | .env.example + overview |
| 7. Deployment | docker-compose skeleton. Detalle Easypanel → plan 4 |
| 8. Estructura de docs | TODA la estructura de `docs/` creada en este plan |
| 9. Out of scope | implícito; no se documentan domain-events ni data-flow.md ni deployment.md |
| 10. Decisiones tomadas | ADR 0001; el resto vive en el spec |

**Placeholder scan:** ningún TBD/TODO en los archivos generados.

**Type consistency:** los nombres de env vars son consistentes entre `.env.example`, `docker-compose.yml`, `overview.md`, y el spec. Verificado.

**Gaps conocidos (intencionales):**
- `nanoclaw/CLAUDE.md`, `nanoclaw/README.md`, `nanoclaw/docs/*` → plan 3.
- `mcp-monica/CLAUDE.md`, `mcp-monica/README.md`, `mcp-monica/docs/*` → plan 2.
- `docs/domain/domain-events.md` → diferido (no hay event bus aún).
- `docs/architecture/data-flow.md` → diferido (duplica spec mientras no haya nuevo flujo).
- `docs/architecture/deployment.md` → plan 4.

---

## Próximos planes

Cuando este plan esté ejecutado y commiteado:

1. **Plan 2**: `mcp-monica` MCP server — implementación TS/Bun, Dockerfile, tests del proxy a Supabase.
2. **Plan 3**: nanoclaw fork — clonar `qwibitai/nanoclaw`, desactivar adapters upstream no usados, agregar `eleve-http` channel adapter custom, system prompt loader 3-fuentes, outbound a `n8n-whatsapp-agent-response`.
3. **Plan 4**: integración + Easypanel deployment — runbook, healthchecks, smoke tests end-to-end.
