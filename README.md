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
