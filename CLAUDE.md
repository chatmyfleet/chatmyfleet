# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repo state: pre-build

Only `LICENSE`, `README.md` (stub), and `.gitignore` are committed. The application code does not exist yet — first private commits begin ~2026-06-01, the public `v0.1.0` tag lands ~2026-06-24, and the PH-launch `v0.4.0` tag lands ~2026-08-19. Do not invent build/lint/test commands; there is no `pyproject.toml`, `docker-compose.yml`, or `cmf/` package on disk. When code lands, the layout and toolchain below describe what to expect.

Three working directories on disk are **gitignored** and act as planning scaffolding only:

- `brief/chatmyfleet/` — source-of-truth product docs (locked v6 narrative as of 2026-05-23). These feed the eventual public `README.md`, `chatmyfleet.com`, and the in-repo `docs/` tree. The split between this folder and an internal sibling `/products/chatmyfleet-notes/` (outside this repo) is deliberate: public voice here, internal strategy there. Voice rules are listed in `brief/chatmyfleet/README.md` — do not introduce internal vocabulary ("Track A/B", "Mode 1", "kill criterion", "Week N", revenue projections) into public copy.
- `.agent/` — Orbit workflow runbooks, skills, and templates that govern how every change is made.
- `.claude/commands/` — slash-command surfaces for the Orbit loops (mirror of `.agent/workflows/`).

These three trees are the operating manual. Treat them as authoritative even though `git ls-files` won't show them.

## The Orbit workflow governs all work

Every non-trivial change runs through one of five **Orbit loops**, exposed as slash commands. Read `.agent/rules/orbit.md` first; it sets always-on standards. The loops:

| Loop | Command | When |
|---|---|---|
| 1 — Strategy | `/orbit1-strategy` | New project, scope work, write/update `docs/spec.md` |
| 2a/2b/2c — Design | `/orbit2a-UX-design`, `/orbit2b-UIUX-design`, `/orbit2c-UI-prototype` | UX research, design brief, HTML/React prototype |
| 3 — Plan | `/orbit3-plan` | Spec approved → break into epics in `docs/epics/`, write System Manifest + Acceptance Criteria, scaffold `tests/manual/epic-X.md` |
| 4 — Build | `/orbit4-build`, `/orbit4b-refine`, `/orbit4c-test` | Implement an approved epic on a `feature/<epic-name>` branch; refine post-build feedback; run the full test pass |
| 5 — Deploy | `/orbit5-deploy` | Release notes, infra, deploy |

Hard rules from `.agent/rules/orbit.md` that are easy to forget:

- **No jumping loops.** Loop 3 cannot start until Loop 2 is approved; Loop 4 cannot start until the epic exists in `docs/epics/` with status `Approved`.
- **All implementation work happens on `feature/<epic-name>` branches.** Never commit directly to `main`.
- **Collision lock.** Before claiming an epic, read `docs/agent_board.md`. If the epic or its files are locked, stop and pick another. Register your claim before starting; release after merge.
- **Status mirror.** The `Status:` line in `docs/epics/epic-{N}-{name}.md` must match the corresponding row in `docs/spec.md` at all times. Statuses are `Approved → In Progress → Review → Done`.
- **Micro-task test gate.** Inside Loop 4, each micro-task ships with its own passing test in `tests/backend/` or `tests/frontend/` before being marked complete. The full suite + `tests/manual/epic-X.md` runs once at epic close via `/orbit4c-test` — an epic with no full-suite run cannot become `Done`.
- **Completeness check.** Before requesting review, no artifact may contain "TODO", "TBD", or empty sections (the exception is when you are *asking* for clarification).
- **Stop & ask** on scope/budget/timeline changes, ambiguity, repeated test failures, or anything needing credentials.

## Always-on standards (from `.agent/skills/`)

- **Documentation** (`orbit_documentation_standard`): all docs live under `docs/`; doc updates ship in the **same PR** as the code they describe; deprecated docs move to `docs/archive/` (never delete — the "why" trail is the value). Prefer JSON for config, Markdown for prose, `.env` for secrets.
- **Testing** (`orbit_testing_standard`): hard line between `tests/` (deterministic, no paid external APIs — flakiness destroys trust in CI) and `scripts/` / `spikes/` (one-off, live data fine, not a CI signal). Coverage tier by project type: POC/Spike = none required; **MVP = critical happy path** (this is the v0.1–v0.4 tier); PROD = full coverage with sad paths.
- **Task management** (`task_management_shared`): one file per epic at `docs/epics/epic-{N}-{name}.md`, mirrored status in `docs/spec.md`, claimed via `docs/agent_board.md`.

## Architecture (planned — v0.4.0 launch target)

Authoritative sources: `brief/chatmyfleet/mvp-architecture.md` (build spec), `brief/chatmyfleet/architecture-deep.md` (eventual state), `brief/chatmyfleet/architecture.md` (conceptual).

Three architectural calls are **non-negotiable** and will not be reopened without an explicit user decision:

1. **Gateway / worker split.** Slack's 3-second webhook timeout makes inline agent polling structurally impossible. `cmf-gateway` validates and enqueues to Redis Streams in <500ms; `cmf-worker` does the slow work (agent calls, message sends, audit writes, the 60s timeout-tick task).
2. **Postgres = source of truth, Redis = cache + queues.** One persistent store. Restart-safe. Trivially backed up.
3. **Append-only audit log via Postgres role permission.** The `audit_events` table grants no `UPDATE`/`DELETE` to the app role. Two lines per state transition. The hash chain + Ed25519 signing land at `v0.9` — append-only is the v0.1 foundation.

Four containers at MVP: `cmf-gateway`, `cmf-worker`, `postgres` (pgvector/pgvector:pg16), `redis`. No separate scheduler or embeddings container until production load justifies splitting — both stay folded into `cmf-worker`.

Deployment model: **one ChatMyFleet instance per client tenant**. Single-tenant Postgres per deployment. Multi-tenant Cloud is a post-v1.0 concern (see `architecture-deep.md`).

## Planned tech stack (v0.4.0)

When code lands, use these picks (full list in `brief/chatmyfleet/mvp-architecture.md`):

- Python 3.12+ · `uv` for env/deps · `ruff` lint+format · `mypy` types · `pytest` + `pytest-asyncio` tests
- FastAPI + uvicorn · pydantic v2 + pydantic-settings (config)
- Postgres 16 + pgvector + SQLAlchemy 2.0 async + asyncpg + Alembic (`migrations/`)
- Redis 7 (cache + Streams as the event bus)
- `slack-bolt` `AsyncApp` mounted on FastAPI via `AsyncSlackRequestHandler`
- `openai` `AsyncOpenAI` (Assistants connector at launch; webhook/MCP/LangGraph/LINE/Telegram/etc. follow on a monthly cadence v0.5+)
- `httpx`, `tenacity`, `typer`, `structlog` (JSON to stdout), `prometheus-client`
- `cryptography` AESGCM for BYOK secret encryption; `argon2-cffi` Argon2id for API-token hashing
- Single multi-stage `Dockerfile`, distroless runtime, `linux/amd64` only at v0.4
- CI: GitHub Actions + uv

Channel SDKs (`line-bot-sdk`, `aiogram`) and connector libs (`mcp`, `langgraph-sdk`) only enter the dependency tree in the release that ships their integration — do not add them speculatively.

## Planned package layout

```
cmf/
├── gateway.py            # FastAPI app entrypoint
├── worker.py             # asyncio main loop entrypoint
├── config.py             # pydantic-settings
├── db/                   # models, session, AESGCM crypto for connector_config secrets
├── channels/             # base Protocol + slack.py (line.py at v0.8, telegram.py at v0.10)
├── connectors/           # base Protocol + openai_assistants.py (webhook v0.5, mcp v0.6, ...)
├── memory/               # pgvector search, embedding worker (inline), NL rule extraction
├── approvals/            # policy evaluator, 60s timer task, email fallback
├── audit/                # append-only event emitter
├── commands/             # 10 slash command handlers
├── web/                  # admin SPA + JSON API (4 pages at v0.4)
└── observability/        # structlog config + 6 prometheus counters
```

The folder structure is forward-compatible — post-launch tags add modules (`channels/line.py`, `connectors/mcp_client.py`) without restructuring.

Tests live under `tests/` and split three ways: `tests/backend/` (mocked external services), `tests/frontend/` (Playwright, mocked APIs), `tests/manual/epic-X.md` (step-by-step SIT/UAT guides, executed by `/orbit4c-test` or by a human).

## What is intentionally out of scope

ChatMyFleet wraps agent platforms — it does not build agents, replace agent runtimes, or compete with Lindy/Dify/LangGraph/OpenAI Assistants. It does not build vertical bots, CRM features, or pair-programming surfaces. The "What ChatMyFleet does NOT do" table in `brief/chatmyfleet/product.md` is authoritative; align any feature suggestion against it.

Apache 2.0 across the whole repo. **No open-core carve-out, no "Pro" tier, no source-available.** Year 2+ may add managed hosting, but nothing in the OSS is paywalled. Sponsorship and affiliate revenue (disclosed in `brief/chatmyfleet/affiliates.md`) fund the work.
