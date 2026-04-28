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
