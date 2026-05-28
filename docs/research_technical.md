# Technical Research: ChatMyFleet

> Loop 1 technical research. Distils the locked v0.4.0 build spec from `brief/chatmyfleet/mvp-architecture.md` and the eventual-state design from `brief/chatmyfleet/architecture-deep.md` into the Orbit `research-technical-tmpl` shape. Captures stack picks, build-vs-buy rationale, and the risk register that feeds the 9 spikes in `spec.md`.

Last updated: 2026-05-28
Source documents: `brief/chatmyfleet/architecture.md`, `brief/chatmyfleet/mvp-architecture.md`, `brief/chatmyfleet/architecture-deep.md`, `brief/chatmyfleet/integration-guide.md`, `CLAUDE.md`

---

## Technical Goal

Build a single-tenant, self-hostable chat gateway that sits between any agent platform and a chat channel (Slack at v0.4.0; LINE at v0.8; Telegram at v0.10), with HITL approval, append-only audit, and an operator-readable memory layer. Five-minute Docker Compose install, BYOK LLM keys, Apache 2.0 across the whole repo.

The v0.4.0 launch target (Wed 2026-08-19) ships:

- **Channel:** Slack only (`slack-bolt` `AsyncApp` mounted on FastAPI via `AsyncSlackRequestHandler`)
- **Connector:** OpenAI Assistants only (`AsyncOpenAI`, POLL pattern with `requires_action` handling for HITL)
- **HITL flow:** Block Kit approval buttons, blocking + callback builder API, auto-approve threshold counters with trust-delta logic, email fallback (aiosmtplib), 4h reminder → 24h escalation → 48h expire
- **Memory:** rules + entities + decisions tables on Postgres 16 + pgvector + HNSW; NL rule extraction via off-path LLM call with `/rule add` slash-command fallback; pre-context injection for non-MCP agents
- **Audit:** append-only enforced via Postgres role permission (no `UPDATE`/`DELETE` grant on `audit_events`); chat-thread linkback; cryptographic hash chain + Ed25519 signing **deferred to v0.9**
- **Web UI:** 5 admin pages (Plugged-in Agents · Audit Log · Policies · Members & Roles · Dashboard), React 18 + Vite + Tailwind + shadcn/ui, served as bundled static assets by FastAPI at launch
- **Self-host:** single multi-stage Dockerfile, distroless-python runtime, `linux/amd64` only, Docker Compose with 4 containers

The eventual-state target (v1.0.0 ~Aug 2027 or later) adds multi-tenant Cloud, KMS abstraction, LISTEN/NOTIFY hot-update, Ed25519 audit signing + hash chain, Helm chart, OpenTelemetry traces — none of which are launch blockers. Folder structure at v0.4.0 is forward-compatible with these additions.

Non-negotiable architectural calls (will not be reopened without explicit user decision):

1. **Gateway / worker split.** Slack's 3-second webhook timeout makes inline agent polling structurally impossible. `cmf-gateway` validates and enqueues to Redis Streams in <500ms; `cmf-worker` does the slow work (agent calls, message sends, audit writes, the 60s timeout-tick task).
2. **Postgres = source of truth, Redis = cache + queues.** One persistent store. Restart-safe. Trivially backed up.
3. **Append-only audit log via Postgres role permission.** Two lines per state transition. The app role has no `UPDATE`/`DELETE` on `audit_events`. Hash chain + signing land at v0.9; append-only is the v0.1 foundation.

---

## Stack Selection / Tech Decisions

Picks below are the locked v0.4.0 launch set. Items marked **deferred** ship in the named post-launch release. Anything not listed (channel SDKs, connector libs) only enters the dependency tree in the release that ships its integration.

### Application core

| Component | Technology Choice | Rationale | Alternatives considered |
|---|---|---|---|
| Language | Python 3.11+ | Async ecosystem maturity; founder fluency; `slack-bolt` + `openai` + `langgraph-sdk` all first-class | Node/TS (slack-bolt-js exists but Python's async story is better for Postgres+pgvector); Go (loses LLM ecosystem) |
| Env / deps | `uv` | Fast (Rust-based); resolves lockfiles deterministically; single binary; pairs with GitHub Actions cleanly | `poetry` (slower, heavier), `pip-tools` (less ergonomic), `rye` (less mature) |
| Lint + format | `ruff` | One tool replaces black + isort + flake8 + pyupgrade + more; minimal config | `black + isort + flake8` (3 tools, slower) |
| Type check | `mypy` | Mature, widely understood; project is small enough that strict mode is tractable | `pyright` (faster but Node-based, extra toolchain) |
| Tests | `pytest` + `pytest-asyncio` | Standard; async fixture support; rich plugin ecosystem | `unittest` (verbose), `nose2` (declining) |
| Web framework | FastAPI + uvicorn | Async-native; pydantic v2 integration; ASGI app for `slack-bolt` `AsyncApp` mount; auto OpenAPI | Starlette (lower-level), Django (sync core, heavyweight), Litestar (smaller community) |
| Config | pydantic v2 + pydantic-settings | Typed env vars; validates on boot; same model layer as request schemas | `dynaconf`, `python-decouple` (less typed) |

### Storage + state

| Component | Technology Choice | Rationale | Alternatives considered |
|---|---|---|---|
| Relational DB | Postgres 16 (`pgvector/pgvector:pg16`) | Source of truth; pgvector for memory semantic search; mature, boring, restart-safe | MySQL (no first-class vector), SQLite (multi-process write contention), CockroachDB (overkill, no pgvector parity) |
| ORM | SQLAlchemy 2.0 async + asyncpg | 2.0 typed declarative style; asyncpg is the fastest Postgres driver | psycopg3 async (slower for high-throughput), SQLModel (thin wrapper, less control), Tortoise (smaller ecosystem) |
| Migrations | Alembic | SQLAlchemy-native; offline SQL generation; battle-tested | `yoyo`, `dbmate` (less Python-integrated) |
| Vector search | pgvector + HNSW index | Co-locates with relational data; no separate vector DB to operate; HNSW gives sub-50ms on ~200 rules at MVP scale (Spike 3 confirms latency; embedding-model choice is a separate Spike 3 deliverable) | Pinecone/Weaviate/Qdrant (extra service to operate), FAISS (in-process, not multi-worker safe) |
| Embedding provider | **Locked by Spike 3:** default `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` (Apache 2.0, 384-dim, 0.22GB) via FastEmbed (ONNX, no torch); documented swap `sentence-transformers/paraphrase-multilingual-mpnet-base-v2` (Apache 2.0, 768-dim, 1.0GB). Operator-swappable via `EMBEDDING_MODEL`; column dim fixed per deploy | Local-default keeps the OSS promise (zero paid keys); FastEmbed avoids `torch`; both models Apache 2.0 (safe to bake into the distributed image, verified); 100% Thai recall@10 + p95 retrieval ~0.5–1.9ms @200 rows measured. FastEmbed lacks e5-small/embeddinggemma (would force torch) — hence the pivot | OpenAI `text-embedding-3-small` (1536-dim, BYOK override); `embeddinggemma` (Apache 2.0 **+ Gemma Terms** restrictions — avoided); Jina v3 (**CC BY-NC**, blocked for commercial OSS); local `sentence-transformers`+torch (heavyweight container) |
| Cache + queues | Redis 7 + redis-py async + Streams | Same Redis for hot-path cache and the 6-stream event bus; restart-safe with AOF; mature | RabbitMQ (extra service), NATS (less battle-tested in Python async), Kafka (operational overhead) |
| Audit append-only | Postgres role permission (no `UPDATE`/`DELETE` grant on `audit_events`) | Enforcement at the DB layer, not the application — provably enforced. Ed25519 hash-chain signing deferred to v0.9 | App-level "don't update" (provably weaker), separate immutable log service (operational overhead) |

### Channels + connectors (v0.4.0 launch set)

| Component | Technology Choice | Rationale | Deferred |
|---|---|---|---|
| Slack adapter | `slack-bolt` `AsyncApp` mounted via `AsyncSlackRequestHandler` on FastAPI | Official Slack SDK; signature verify, Block Kit, interactivity, slash commands all built in; AsyncApp avoids running a second server | Raw Slack Events API + custom Block Kit serializer (more code, no benefit) |
| OpenAI Assistants connector | `openai-python` `AsyncOpenAI` | Official SDK; native async; `requires_action` event surface for HITL | HTTP-direct (more code, more brittle) |
| LINE adapter | `line-bot-sdk` | Deferred to **v0.8.0** — not in v0.4 dependency tree | — |
| Telegram adapter | `aiogram` | Deferred to **v0.10.0** — not in v0.4 dependency tree | — |
| MCP connector | `mcp` Python SDK | Deferred to **v0.6.0** — not in v0.4 dependency tree | — |
| LangGraph connector | `langgraph-sdk` | Deferred to **v0.11.0** — not in v0.4 dependency tree | — |
| Generic webhook connector | `httpx` (already in tree) | Deferred to **v0.5.0** — endpoint shape ships earlier as part of the Builder API | — |

### Web UI (admin SPA, 5 pages at v0.4.0)

| Component | Technology Choice | Rationale | Alternatives considered |
|---|---|---|---|
| Language | TypeScript | Type-safe Web UI; matches the typed Python backend discipline | Plain JS (loses contract safety with backend) |
| Framework | React 18 | Largest hire/contributor pool; shadcn/ui copy-paste pattern aligns with no-vendor-lock principle | Vue (smaller ecosystem in EN docs), Svelte (smaller hire pool), HTMX (would limit Web UI dynamism — but valid for v1+) |
| Build tool | Vite | Fast dev server; ESM-native; small config | webpack (slower), Next.js (overkill for a 5-page admin SPA, would force SSR) |
| Styling | Tailwind CSS + shadcn/ui | Copy-paste components (no vendor lock-in); rapid iteration; Tailwind is the de facto SPA styling | Mantine/MUI (component-vendor lock), CSS Modules (slower to build a 5-page admin) |
| Routing | React Router v6 | Standard; tiny; matches the SPA-served-by-FastAPI pattern | TanStack Router (newer, less stable), `wouter` (smaller, less docs) |
| Data fetching | TanStack Query | Cache + retry + invalidation; pairs cleanly with FastAPI JSON | SWR (smaller feature surface), raw fetch (would need to rebuild caching) |
| Forms | React Hook Form + Zod | Zod schemas shared between client validation and backend pydantic (via JSON-schema or manual mirror) | Formik (slower, less ergonomic) |
| State | Zustand (only if state-lifting becomes insufficient) | Tiny; no provider hell; opt-in, not default | Redux Toolkit (heavyweight for 5 pages), Jotai (smaller community) |
| Charting | `recharts` (Dashboard page sparkline/bars only) | Small, declarative, React-native; only dependency the Dashboard page adds; aligns with no-vendor-lock | Chart.js (imperative, canvas), visx (lower-level, more code), nivo (heavier) |
| Serving | Bundled static assets served by FastAPI | Single container at launch; nginx sibling available post-launch if cache/CDN behaviour needed | Separate nginx at launch (extra container, not justified for 5 pages) |

### Cross-cutting

| Component | Technology Choice | Rationale | Alternatives considered |
|---|---|---|---|
| HTTP client | `httpx` | Async-native; sync API too; same library for builder-API tests and outbound | `aiohttp` (separate sync/async APIs), `requests` (sync-only) |
| Retries | `tenacity` | Decorator API; exponential backoff; jitter; widely understood | Custom retry helpers (re-invented wheel) |
| CLI | `typer` | FastAPI-ergonomics for CLI; pydantic-friendly; one-file scripts | `click` (more boilerplate), `argparse` (low-level) |
| Email | `aiosmtplib` + BYO SMTP (SendGrid / Resend / Postmark / self-hosted Postfix) | BYO matches BYOK ethos; no markup, no surprise bills, no usage metering | Built-in SMTP relay (operational overhead), SaaS-only (locks self-hosters out) |
| Crypto — secrets | `cryptography` AESGCM with static `.env` key | KMS abstraction deferred to v1.x (multi-tenant Cloud); AESGCM with file key is sufficient for single-tenant self-host | AWS KMS / GCP KMS / Vault (extra deps + creds; deferred), Fernet (single-purpose, less general than AESGCM) |
| Crypto — API tokens | `argon2-cffi` Argon2id | Modern password-equivalent hashing; resists GPU attacks | bcrypt (older), scrypt (less reviewed) |
| Logging | `structlog` JSON to stdout | Structured logs; correlation_id support; trivial to ship to any log sink | stdlib `logging` (less ergonomic for structured output) |
| Metrics | `prometheus-client` | 12 counters/histograms/gauges at launch (6 infrastructure + 6 business KPI per the locked telemetry design in `docs/architecture.md` §10); standard Prometheus scrape; no agent needed | OpenTelemetry (deferred — adds collector complexity), StatsD (less standard now) |
| Container | Single multi-stage Dockerfile; distroless-python runtime; `linux/amd64` only at v0.4 | Smallest attack surface; reproducible; multi-arch deferred until adopter demand | Alpine (musl quirks with some Python wheels), full Debian (larger attack surface) |
| Deployment | Docker Compose (4 containers) | 5-minute install promise; trivially understood; no Kubernetes burden at launch | Helm (deferred to ~v0.12), bare-metal install (loses self-host ergonomics) |
| CI | GitHub Actions + `uv` | Matches existing repo on GitHub; `uv` is fast in CI; no additional vendor | CircleCI/Travis (extra account), self-hosted runners (operational overhead) |

### Repo + governance (technical decisions, not just process)

| Component | Choice | Rationale |
|---|---|---|
| Repository structure | **Monorepo** (single `chatmyfleet/` package + bundled SPA + docs + migrations + tests) | One PR can touch backend + frontend + docs + tests, which is enforced by the Orbit "doc updates ship in the same PR" rule |
| Service architecture | **Modular monolith** — `cmf-gateway` and `cmf-worker` are two entry-point modules of the same package, sharing models, config, crypto, audit | 4-container deployment without 4-codebase overhead; clean split to separate services later if production load justifies |
| License | Apache 2.0 across the whole repo | Locked. No open-core carve-out. No source-available rug-pull risk. No "Pro" tier |
| Branching | `feature/<epic-name>` per Loop 4 epic; merge to `main` after epic test gate | Per Orbit `task_management_shared` |

---

## Build vs Buy Analysis

- **Build:** the gateway adapter pattern itself, the operator-inbox UX, the HITL state machine, the memory layer's operator-editable semantics, the audit-thread linkback, and the connector library. **The product is precisely the integration patterns** — none of this is helped by buying an OSS competitor or wrapping a SaaS.
- **Build:** chat-thread linkback in the audit log. None of the comparable agent products implement this; it is the explicit differentiator.
- **Build:** NL rule extraction with structured-policy output. Existing rules engines (`durable-rules`, OPA) don't ingest natural language; the extraction is small but bespoke.
- **Build (thin):** approvals state machine. Custom for the 4h/24h/48h ladder + email fallback + audit linkback; the existing HumanLayer SDK is approval-only with no UI/audit/memory.
- **Buy (via affiliate-eligible SaaS):** none of the agent platforms ChatMyFleet wraps — Lindy, Dify, n8n, Flowise, OpenAI Assistants — get rebuilt or competed with. ChatMyFleet pays them affiliate-style attention by linking to them in integration docs.
- **Buy (open libraries):** every standard slice — Postgres, Redis, pgvector, `slack-bolt`, `openai-python`, `aiosmtplib`, `cryptography`, `argon2-cffi`, `structlog`, `prometheus-client`, FastAPI, React, shadcn/ui — bought, not built. The lock-in cost of each is low and replacement paths are documented in `architecture-deep.md` §15.
- **Buy (LLM provider, via BYOK):** ChatMyFleet never pays LLM bills. Builder/operator brings the key (OpenAI / Anthropic / Gemini). Removes a usage-metering subsystem from scope.
- **Defer (do not build until evidence demands):** KMS abstraction, multi-tenant Cloud, audit hash chain + Ed25519 signing (v0.9), LISTEN/NOTIFY hot-update, Helm chart, OpenTelemetry, separate `cmf-scheduler` and `cmf-embeddings` containers, multi-arch images, property-based tests, testcontainers in CI. Each has a named target version in `mvp-architecture.md`'s defer list.

---

## Technical Risks & Mitigation

The 9 spikes below are technical claims not yet validated against running code. Spikes 1–6 come from the brief's open doubts; Spikes 7–9 were added during the Loop 1 architecture review to de-risk the foundation epics (audit-role enforcement, distroless packaging, Redis Streams restart-safety). Each becomes `docs/spikes/spike-N-*.md` before any Loop 4 build starts; results record back into `docs/spec.md`. Per user direction, **all 9 spikes defer to Loop 4 pre-flight** — each runs as the first micro-task of the epic that depends on it.

| # | Risk | Spike to validate | Target epic (Loop 4) | Credentials needed |
|---|---|---|---|---|
| 1 | Slack signature verify + Block Kit round-trip on FastAPI inside Slack's 3s ack window | Spike 1 — `slack-bolt` `AsyncApp` mounted via `AsyncSlackRequestHandler`; measure ack latency under realistic payload sizes | Epic 4 (Slack Adapter + First Round-Trip) | Slack workspace + bot token |
| 2 | OpenAI Assistants `requires_action` POLL pattern under Redis Streams workers — polling cadence, straggler-run handling, worker contention | Spike 2 — measure polling cost vs latency; design straggler handler; confirm no double-execution under retry | Epic 5 (OpenAI Assistants Connector + State Machine) | OpenAI API key |
| 3 | Embedding-model choice for the memory layer + pgvector HNSW recall/latency at 200-rule scale + runtime model-swap mechanism | Spike 3 — open-source bake-off (`intfloat/multilingual-e5-small` vs `google/embeddinggemma-300m`) with Thai + English recall@10; pgvector p95 retrieval latency at 200 rules; design `EMBEDDING_MODEL` env-var contract and column-dim strategy (fixed / Matryoshka-truncate / migrate) | Epic 7 (Memory Schemas + pgvector + HNSW Search) | None for open-source candidates (local CPU via FastEmbed); OpenAI key optional for `text-embedding-3-small` baseline |
| 4 | NL rule extraction reliability — operator phrasings parsed into structured policy mutations with usable accuracy and a graceful ambiguity fallback | Spike 4 — corpus of 30+ realistic operator phrasings; measure extraction accuracy; design ambiguity-detection fallback | Epic 8 (NL Rule Extraction + /rule + Pre-Context Injection) | OpenAI/Anthropic key (for the extraction LLM call) |
| 5 | Email approval link signing + click → audit linkback to the originating Slack thread (cross-channel correlation) | Spike 5 — design signed-link scheme (HMAC + expiry); verify Slack-thread linkback survives the email round-trip | Epic 10 (Builder API + Email Fallback + Timeout Tick) | SMTP provider + Slack workspace |
| 6 | Idempotency keys on the blocking builder-API endpoint — long-poll plus network retry behaviour without double-execution | Spike 6 — design idempotency-key scheme; test under simulated network retry storm | Epic 15 (Failure Hardening + Idempotency + Launch Polish) | None (local-only) |
| 7 | Append-only audit enforcement via Postgres role permission — two-role setup (migration vs app role) composing with SQLAlchemy 2.0 async + Alembic + pgvector | Spike 7 — prove restricted app role has INSERT (+ sequence USAGE) but no UPDATE/DELETE on `audit_events`; NFR5 enforcement test passes at the DB layer | Epic 2 (Postgres Schema + Audit Append-Only Enforcement) | None (local Postgres) |
| 8 | Distroless `linux/amd64` image with native deps (+ conditional local embedding runtime) builds and cold-boots via `docker compose up` inside the 5-minute install budget | Spike 8 — multi-stage distroless build bundling `asyncpg`/`cryptography`/`argon2-cffi` (+ FastEmbed/onnxruntime if Spike 3 picks local); compose healthcheck ordering to green `/health` | Epic 1 (Project Skeleton + CI Foundation) | None (local-only) |
| 9 | Redis Streams consumer-group restart-safety — NFR9 in-flight state survives a worker crash without loss or double-processing | Spike 9 — `XREADGROUP` + `XACK` + `XAUTOCLAIM` reclaim of stale pending entries after simulated worker crash, across the 6-stream bus | Epic 3 (Redis Streams Event Bus + Worker Loop) | None (local Redis) |

### Cross-cutting risks not addressed by the 9 spikes

| Risk | Mitigation |
|---|---|
| Solo bandwidth slippage past Week 12 puts dated launch at risk | Epic sizing discipline (3–8 MTs per epic, vertical slice rule); ruthless defer-list enforcement; micro-task test gate prevents rework; ~2 weeks of schedule buffer |
| Append-only audit assumption violated by a future migration | Migrations reviewed against the rule that `audit_events` never receives an `UPDATE`/`DELETE` grant; CI check: integration test asserts the app role cannot delete an audit row |
| BYOK secret leak via misconfigured `.env` | `.env.example` ships with placeholders only; AESGCM-encrypted at rest in `connector_config`; secrets never logged (structlog redactor); `SECURITY.md` ships at v0.1.0 with a coordinated disclosure path |
| Slack rate-limit during a reply storm at PH launch | Queue-and-backoff in the worker; replay when rate clears; rate-limit failure mode is one of the 7 named failure modes tested at v0.4.0 |
| Worker restart loses in-flight approval timer state | Timer state lives in Postgres (`approval_requests.timeout_at`); 60s asyncio tick reconciles from Postgres on every iteration; tested as part of Epic 8 |
| OpenAI Assistants API change between Loop 1 and launch | Pinned SDK version; integration tests against recorded fixtures (`tests/fixtures/openai_assistants/`); manual SIT in `tests/manual/epic-X.md` runs against live API before each tag |
| Web UI bundle size grows past acceptable cold-load | shadcn/ui copy-paste pattern keeps only what's imported; Vite tree-shaking; admin SPA is 5 pages so the bound is naturally tight; `recharts` (the Dashboard's only added dep) is lazy-loaded on the Dashboard route |
| Single-architecture container (`linux/amd64`) blocks Apple Silicon self-hosters in dev | Documented in `mvp-architecture.md`; defer list says multi-arch ships post-launch on demand signal; `docker compose up` on Apple Silicon works via Rosetta 2 with acceptable performance for dev use |

---

## Implementation Strategy

The implementation strategy is the v0.1 → v0.4 tag plan from `brief/chatmyfleet/mvp-architecture.md`. Each tag is a meaningful capability landing, not a calendar checkpoint. Tag dates are dated to the maintainer's 12-week build calendar.

1. **Phase 1 — Foundation (Week 1-3, no public tag).** Skeleton: 4 containers (`cmf-gateway`, `cmf-worker`, `postgres`, `redis`), CI green, Postgres migrations for 8 tables (`tenants`, `users`, `agents`, `conversations`, `approval_requests`, `audit_events`, `memory_rules`, `memory_entities`, `memory_decisions`), gateway + worker boot, Slack adapter (signature verify + Block Kit send), OpenAI Assistants connector (POLL pattern), conversation state machine (5 states), audit log table, basic slash commands. **Spikes 1, 2, 7, 8, 9 resolved here** (8 → Epic 1, 7 → Epic 2, 9 → Epic 3, 1 → Epic 4, 2 → Epic 5).
2. **Phase 2 — v0.1.0 PUBLIC REPO (Week 4, Wed Jun 24).** Trust-signal artifacts populated (LICENSE, TRADEMARK, SECURITY, CONTRIBUTING, MAINTENANCE, CHANGELOG, AFFILIATES). `chatmyfleet.com` minimal landing live. Sponsors page live. **Repo flips public.** First moment a self-hoster can clone, run `docker compose up`, and have their OpenAI Assistant talking to Slack with an audit trail.
3. **Phase 3 — v0.2.0 Memory layer (Week 5-6, Sat Jul 12).** pgvector + HNSW index. Embedding worker (inline in `cmf-worker`). Memory rules + entities + decisions schemas. NL rule extraction (LLM parse → structured policy). `/rule add` slash command. Pre-context injection for non-MCP agents. **Spikes 3 + 4 resolved here.**
4. **Phase 4 — v0.3.0 HITL approval flow (Week 7-8, Sat Jul 26).** Approval state machine (`pending` / `escalated` / `decided`). Block Kit approval rendering. Auto-approve threshold counter logic with trust-delta. Builder API endpoints (blocking + callback). Email fallback (SMTP setup + signed approval-link verify + audit linkback). Timeout tick task (60s asyncio loop). **Spike 5 resolved here.**
5. **Phase 5 — v0.4.0 LAUNCH (Week 9-12, Wed Aug 19).** Web UI pages 1-2 (Plugged-in Agents + Audit Log with chat linkback). Web UI pages 3-4 (Policies + Members & Roles). Web UI page 5 (Dashboard — KPI aggregations + business-KPI Prometheus counters). Failure hardening — all 7 named failure modes tested with explicit recovery. Idempotency keys on builder API. Demo GIF + README final. **Spike 6 resolved here.** PH launch Wed Aug 19. Reply storm.
6. **Phase 6 — Post-launch monthly cadence (v0.5 → v0.12+, Sep 2026 → Apr 2027+).** One major capability per monthly minor: v0.5 generic webhook, v0.6 MCP, v0.7 n8n, v0.8 LINE + Dify, v0.9 Lindy + audit Ed25519 signing + hash chain, v0.10 Flowise + Telegram, v0.11 LangGraph, v0.12 Helm + Wave 2 affiliate. v1.0.0 ships when the API surface is locked and the product is fully production-stable (expected ~M12 anniversary or later).

---

## Technical Recommendation

**Feasible. Proceed.** The stack picks are conservative and well-understood. The three non-negotiable architectural calls (gateway/worker split, Postgres-as-SoT, append-only audit via Postgres role permission) eliminate the most common categories of late-stage rework. The 12-week build calendar fits the scope at 1.5× AI-assisted throughput with ~2 weeks of buffer.

The recommended path forward:

1. Approve `docs/spec.md` (Loop 1 exit) with the 9 spikes recorded as Pending and scheduled into Loop 4 pre-flight per dependent epic.
2. Hand off to **`/orbit2b-UIUX-design`** for the Web UI's 5 admin pages (Plugged-in Agents, Audit Log, Policies, Members & Roles, Dashboard) — the only surface that needs a design brief. The Slack-side UX is fully specified by `brief/chatmyfleet/ux.md` already.
3. After design brief, hand off to **`/orbit3-plan`** for epic decomposition into the System Manifest + Acceptance Criteria + `tests/manual/epic-X.md` scaffolding.
4. Then **`/orbit4-build`** per epic, each on a `feature/<epic-name>` branch, with the dependent spikes resolved before the epic's first micro-task is claimed.

Risks worth naming again at this point: (a) solo bandwidth and (b) MVP-tier surfaces shipping without sad-path coverage. Both are mitigated by the Orbit micro-task test gate + monthly minor cadence + transparent CHANGELOG. Neither is a blocker for the "Go" decision.
