# Product Specification: ChatMyFleet

> Source of truth for Loops 2–4. Operationalises the locked v6 narrative in `brief/chatmyfleet/` and the comprehensive `docs/user_brief.md` into goals, requirements, spikes, and a vertical-slice epic plan that maps to the v0.1 → v0.4 tag schedule.
>
> Tier per Orbit project types: **PROD by trajectory** — architecture-level PROD calls locked from v0.1.0, load-bearing surfaces at PROD by v0.4.0, polish surfaces ramp from MVP to PROD by v1.0.0. Project type, success criteria, and the not-doing list are defined in `docs/user_brief.md` and are not re-litigated here.

---

## 1. Goals and Background Context

### Goals

- Ship the open-source chat layer that lets AI builders hand day-to-day agent operation to their clients' ops teams, so the builder stops being the operational bottleneck.
- Hit the dated public launch: tag `v0.4.0` Wed 2026-08-19, with 4 meaningful pre-launch tags (`v0.1.0` Jun 24, `v0.2.0` Jul 12, `v0.3.0` Jul 26, `v0.4.0` Aug 19) demonstrating verifiable shipping cadence.
- Hold the **five-minute install contract** — clone + `docker compose up` on a fresh Linux box → OpenAI Assistant replying in Slack inside five minutes, first try.
- Run **a real customer in production** on launch day (founder's Thai SMB client, live refund/discount approvals through HITL) with zero silently-dropped operator or agent messages across the first two weeks.
- Prove **memory survives a connector swap** — a rule taught via natural language still applies after the underlying agent is reconfigured to a different connector.
- Make **append-only audit provably enforced** — Postgres role permission, not application convention. Hash chain + Ed25519 signing arrive at v0.9.
- Stand up the **5-year maintenance contract** as a signed `MAINTENANCE.md` shape the project can actually hold: monthly minor releases (first Friday), monthly transparency posts from M4, security patches within 30 days, breaking changes pre-announced ≥1 minor version, Apache 2.0 locked.

### Background Context

ChatMyFleet sits in a market gap none of the adjacent products occupy: **operator inbox for non-builders, BYO-everything, Apache 2.0**. The closest competitors all miss at least two of those three axes — Lindy/Dify/n8n assume the builder is the daily user; Slack Agentforce / Microsoft Copilot Studio are stack-locked; HumanLayer is SDK-only with no operator UI. Full landscape in `docs/research_business.md`.

The architecture is deliberately conservative: a gateway/worker split forced by Slack's 3-second webhook timeout, Postgres as single source of truth with Redis for cache + Streams, and append-only audit enforced at the database layer. Three non-negotiable architectural calls (gateway/worker split, Postgres-as-SoT, audit-via-role-permission) eliminate most categories of late-stage rework. Full picks in `docs/research_technical.md`.

The 12-week build window is solo-bandwidth-constrained (~14.5 engineer-weeks of work at 1.5× AI-assisted throughput ≈ 9.7 calendar weeks of effort, with ~2 weeks of buffer). Epic sizing in §7 respects this: 15 epics, each 3–8 micro-tasks implementable by one agent on one feature branch.

The locked architecture (component design, message flow, MCP roles, auth model, telemetry) is captured in `docs/architecture.md` — the implementation reference this spec's epics build toward.

### Project Expectation & Components

**Project Type: PROD by trajectory.** v0.4.0 ships the foundation; v1.0.0 (~Aug 2027 or later) is the production-stable, semver-locked milestone. The brief's locked tier table per surface is summarised below — full version in `docs/user_brief.md` §"Per-surface tier at v0.4.0".

| Surface | v0.4.0 tier | Reaches full PROD at |
|---|---|---|
| Architecture (gateway/worker split, Postgres SoT, append-only audit via role permission, 4-container shape) | **PROD** — locked, non-negotiable | `v0.1.0` |
| HITL approval flow + state machine | **PROD** (unit + integration + E2E happy + sad) | `v0.3.0` |
| Append-only audit log (enforced via Postgres role permission, exportable, chat-thread linkback) | **PROD** on enforcement + linkback; Ed25519 hash chain + signing at v0.9 | `v0.9.0` (signing) |
| Seven named failure modes | **PROD** (each tested with explicit recovery path) | `v0.4.0` |
| Slack adapter + OpenAI Assistants connector | **PROD** (Tier 1 round-trip happy + sad) | `v0.4.0` |
| Builder API (blocking + callback, idempotency keys) | **PROD** | `v0.4.0` |
| Trust-signal artifacts (LICENSE, TRADEMARK, MAINTENANCE, CHANGELOG, SECURITY, CONTRIBUTING, AFFILIATES, Sponsors page, chatmyfleet.com landing) | **PROD** — populated at `v0.1.0` | `v0.1.0` |
| Web UI (5 admin pages — Plugged-in Agents, Audit Log, Policies, Members & Roles, Dashboard) | **MVP** (happy path; sad paths added on adopter signal) | `v0.6.0` – `v0.8.0` |
| Memory layer (rules, entities, decisions) | **MVP** (happy path + one NL-extraction regression test) | `v0.6.0` – `v0.9.0` |
| Slash commands, daily brief, observability depth | **MVP** (polish iterates in monthly minors) | `v0.5.0` – `v1.0.0` |

**Components suitable for PROD-by-trajectory at v0.4.0:**
- Load-bearing surfaces ship with **sad-path tests** + **integration tests against real Postgres + Redis** from day one (Orbit Tier 1 — Zero Trust).
- Polish surfaces ship with **happy-path coverage** + at least one regression test guarding the feature's defining promise.
- Trust-signal artifacts ship at v0.1.0 when the repo flips public, not at v0.4.0.
- Breaking changes — even within the v0.x era — get a deprecation note in the prior minor release.

### Change Log

| Date | Version | Description | Author |
|---|---|---|---|
| 2026-05-28 | 0.1 | Initial Loop 1 spec drafted from locked v6 brief + comprehensive `user_brief.md`. 14 epics, 6 spikes deferred to Loop 4 pre-flight. | Natapone Charsombut (with Orbit Loop 1) |
| 2026-05-28 | 0.2 | Python 3.11+ (dropped from 3.12+). Spike 3 expanded to embedding-model selection (`intfloat/multilingual-e5-small` vs `google/embeddinggemma-300m`) + Thai recall + runtime model-swap design. New FR21 (operator-swappable embedding model). User-approved Epic 14 oversized override (kept as one epic, not split). All 14 epics flipped Draft → Approved. | Natapone Charsombut (with Orbit Loop 1) |
| 2026-05-28 | 0.3 | Locked architecture in new `docs/architecture.md`. Telemetry expansion: FR22 (token capture in audit rows), FR23 (5th Web UI page — Dashboard), NFR11 expanded 6 → 12 Prometheus metrics. New Epic 14 (Web UI Dashboard) inserted; previous Epic 14 (Failure Hardening + Launch Polish) renumbered to Epic 15. Spec total: 15 epics. Spike 6 retargeted to Epic 15. Tier 1 token capture absorbed into Epic 5 MT-02. | Natapone Charsombut (with Orbit Loop 1) |
| 2026-05-28 | 0.4 | Spike review. Confirmed Spikes 1–6 all needed. Added Spike 7 (append-only audit two-role DB setup → Epic 2), Spike 8 (distroless image + cold-boot → Epic 1), Spike 9 (Redis Streams reclaim/restart-safety → Epic 3) — each de-risks a foundation epic. Spec total: 9 spikes. FR17 corrected to 14 slash commands (10 discoverable + 4 setup); `/agent edit` added to Epic 5; full command + `@mention` surface documented in `docs/architecture.md` §6.9. | Natapone Charsombut (with Orbit Loop 1) |
| 2026-05-28 | 0.5 | Ran the 5 credential-free spikes (3, 6, 7, 8, 9) early — all PASS (results in `docs/spikes/`). **FR21 locked:** embedding default = `paraphrase-multilingual-MiniLM-L12-v2` (Apache 2.0, 384-dim) via FastEmbed/ONNX; `mpnet-base-v2` (Apache 2.0, 768-dim) swap; licenses verified clean for OSS distribution. Findings recorded: audit role enforcement (SQLSTATE 42501), Redis reclaim is at-least-once → idempotent worker side effects, idempotency claim pattern, distroless `libgomp` gotcha. Spikes 1/2/4/5 remain Pending (credential-gated). | Natapone Charsombut (with Orbit Loop 1) |

---

## 2. Requirements

### Functional

- **FR1**: A self-hoster can clone the public repo and run `docker compose up` on a fresh Linux box, and within five minutes have ChatMyFleet running with an OpenAI Assistant replying to `@agent hello` in their Slack workspace, on the first try, with no manual config diving.
- **FR2**: The gateway validates every incoming Slack webhook against Slack's signing secret and acknowledges within Slack's 3-second timeout window, enqueuing the work to Redis Streams for the worker to process asynchronously.
- **FR3**: An operator can `@mention` an agent in Slack (e.g., `@finance-ai`), and the message is routed to the correct agent platform via the OpenAI Assistants connector, with the agent's response rendered back to the Slack thread.
- **FR4**: Every state transition in `conversations`, `approval_requests`, and agent interactions writes exactly one row to `audit_events`. The application database role has no `UPDATE` or `DELETE` permission on `audit_events`; enforcement is at the Postgres layer, not in application code.
- **FR5**: When an agent reaches a tool call requiring approval (OpenAI Assistants `requires_action`), ChatMyFleet renders a Slack Block Kit message with `[Approve]` and `[Decline]` buttons; the operator's click resumes the agent's run with the decision.
- **FR6**: When the on-call approver does not respond inside the configured window, ChatMyFleet escalates: 4h → reminder in Slack thread; 24h → email to backup approver per policy; 48h → expire. The 60s asyncio timeout-tick task reconciles state from Postgres each tick.
- **FR7**: An approval click from the email fallback link verifies a signed token (HMAC + expiry), records the decision, writes the audit row, and links the audit entry back to the originating Slack thread URL.
- **FR8**: Auto-approve threshold logic counts consecutive successful approvals per (agent, action-class) and surfaces a trust-delta proposal to the operator (e.g., "auto-approve refunds under $200" after 14/14 manual approvals).
- **FR9**: Operators can teach the agent rules in natural language in chat (e.g., "auto-approve refunds under $200 for repeat customers"). ChatMyFleet's NL extraction parses the phrase into a structured policy mutation and stores it in the `memory_rules` table. Ambiguous phrasings trigger a confirmation prompt before storage.
- **FR10**: A `/rule add <rule text>` slash command provides a deterministic fallback path for rule storage when NL extraction confidence is below threshold.
- **FR11**: For agents that do not speak MCP, ChatMyFleet pre-injects the top-K relevant memory rules into the agent's system context before each run, using pgvector + HNSW semantic search over `memory_rules.embedding`.
- **FR12**: A rule taught via the memory layer continues to apply after the underlying agent platform is reconfigured to a different connector (memory layer is connector-independent).
- **FR13**: The Builder API exposes 5 endpoints: `POST /v1/agents/{id}/messages`, `POST /v1/agents/{id}/approval_request` (with `?wait=blocking|callback`), `GET /v1/agents/{id}/approval_request/{req_id}`, `POST /v1/agents/{id}/memory/search`, `POST /v1/agents/{id}/memory/write_rule`. The blocking approval endpoint long-polls until decision or timeout; the callback variant returns immediately and posts to a builder-supplied URL on resolution.
- **FR14**: All Builder API write endpoints accept an idempotency key; repeated requests with the same key return the same response without re-executing the action.
- **FR15**: The Web UI ships 5 admin pages served as a single React 18 + Vite SPA from FastAPI: (1) Plugged-in Agents (list, add, rename, start/stop, rotate keys); (2) Audit Log (search, filter, export, chat-thread linkback on every row); (3) Policies (editable documents with risky-override warnings); (4) Members & Roles (invite Staff, set permissions, deactivate); (5) Dashboard (KPI cards + per-agent activity table + failure-mode counts + time-range selector — see FR23).
- **FR16**: Every row in the Audit Log page links back to the Slack thread URL where the action originated; clicking the link opens the thread at the moment of the action.
- **FR17**: Fourteen slash commands work in Slack, split into two groups (full surface + `@mention` routing detail in `docs/architecture.md` §6.9). **Ten discoverable** (listed by `/help`): `/help`, `/fleet`, `/about`, `/brief`, `/audit`, `/policy`, `/rename` (Admin), `/start` (Admin), `/stop` (Admin), `/dashboard`. **Four setup/config** (not surfaced in `/help`, all Admin): `/agent add <type> <args>`, `/agent list`, `/agent edit <agent_id> <setting>`, `/rule add <rule text>`. Non-admins see admin commands grayed out with an "ask admin" tooltip; discoverability is preserved. Per-agent action subcommand trees (e.g. `/agent finance approve`) are deliberately NOT supported — approvals happen via Block Kit buttons.
- **FR18**: All BYOK secrets (OpenAI keys, connector credentials, SMTP credentials) are AESGCM-encrypted at rest in `connector_config` JSONB. API tokens for the Web UI / Builder API are hashed with Argon2id and stored only as hashes.
- **FR19**: The 7 named failure modes each have an explicit recovery path with a test in `tests/backend/` or `tests/manual/epic-X.md`: (1) agent platform timeout >2 min, (2) agent platform returns 500, (3) channel SDK rate-limited, (4) memory layer unreachable, (5) approval timeout 24h+, (6) webhook delivery fails, (7) state-machine inconsistency.
- **FR20**: Trust-signal artifacts are populated and accurate at v0.1.0 (Week 4) when the repo flips public: `README.md`, `LICENSE` (Apache 2.0), `TRADEMARK.md`, `SECURITY.md`, `CONTRIBUTING.md`, `MAINTENANCE.md`, `CHANGELOG.md`, `AFFILIATES.md`. Sponsors page live. `chatmyfleet.com` minimal landing live.
- **FR21**: The embedding model used for the memory layer is operator-swappable at install time via an `EMBEDDING_MODEL` env var. The default is **`sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`** (Apache 2.0, 384-dim) run locally via FastEmbed (ONNX, no torch) — chosen at Spike 3 for smallest footprint + 100% Thai recall@10 + sub-ms retrieval, requiring no paid API key. The documented higher-recall swap is **`sentence-transformers/paraphrase-multilingual-mpnet-base-v2`** (Apache 2.0, 768-dim). The `memory_*.embedding` column is `VECTOR(384)` for the default; the dim is fixed per deploy at the install-time Alembic migration (changing models post-install is a re-embed migration). Operators may also override to a paid provider (OpenAI / Voyage / Cohere) if its output dim matches the column. The FastEmbed version is pinned for reproducible embeddings.
- **FR22**: Every `agent.run.completed` row written to `audit_events` carries the agent platform's token usage (`prompt_tokens`, `completion_tokens`, `total_tokens`), the model name, the agent-call wall-clock duration, and the count of tool calls executed during the run — when the agent platform exposes these values. For OpenAI Assistants (v0.4 launch connector) all of these are available via the run completion event. Other connectors capture what their protocol surfaces; missing fields are stored as `null` rather than estimated.
- **FR23**: The Web UI ships a 5th admin page — **Dashboard** — accessible to the User Admin role only. Sourced from `audit_events` + `approval_requests` aggregations (not from Prometheus — Prometheus is for the operator of the CMF deployment; the Dashboard is for the User Admin inside their tenant). Surfaces, for a configurable time range (default last 7 days; selectable 30 days / custom): (a) total inbound message count + sparkline; (b) total approval count + decision breakdown (approved / declined / expired); (c) median operator decision time; (d) per-agent activity table — message count, approval count, total tokens per run; (e) failure-mode occurrence counts with linkback to filtered Audit Log; (f) active conversation count (live gauge).

### Non Functional

- **NFR1**: Gateway acknowledges Slack webhooks in <500ms p95, well inside Slack's 3s timeout, on a single-container deployment with realistic payload sizes.
- **NFR2**: pgvector + HNSW memory search returns top-10 results in <50ms p95 on `memory_rules` populated with 200 rules (target scale at MVP per `mvp-architecture.md`).
- **NFR3**: HITL round-trip — operator online → agent action paused → button rendered → click → agent resumed — completes end-to-end in <10s p95.
- **NFR4**: No operator or agent message is ever silently dropped. Every failure path produces either a visible operator error in chat or an audit event the operator can see in the Audit Log.
- **NFR5**: The application database role granted to `cmf-gateway` and `cmf-worker` has no `UPDATE` or `DELETE` permission on `audit_events`. An integration test asserts this by attempting both operations and verifying Postgres rejects them.
- **NFR6**: The repo is Apache 2.0 across the entire codebase. No open-core carve-out, no source-available shims, no "Pro" tier. Locked.
- **NFR7**: Single multi-stage Dockerfile, distroless-python runtime, `linux/amd64` only at v0.4.0. Image size optimised; no shell, no package manager in the runtime image.
- **NFR8**: Test coverage matches the project's PROD-by-trajectory tier: load-bearing surfaces (gateway, worker, audit, HITL, Builder API, connector) ship with unit + integration tests against real Postgres + Redis from day one; polish surfaces (Web UI, memory edge cases, slash command UX) ship with happy-path coverage; every epic's full suite runs once via `/orbit4c-test` before merge.
- **NFR9**: A worker restart preserves all in-flight state. Conversation state, pending approvals, timer state, and queue position all live in Postgres or Redis; no in-process-only state. A test simulates worker restart mid-approval and verifies the timeout-tick task picks up the unresolved request.
- **NFR10**: Structured logs (`structlog` JSON to stdout) include a `correlation_id` propagated end-to-end through gateway → worker → agent platform → audit row, so a single Slack message can be traced through the entire pipeline.
- **NFR11**: Twelve Prometheus counters/histograms/gauges ship at v0.4.0, exposed at `cmf-gateway:8080/metrics` in Prometheus scrape format. Infrastructure health (1–6): `cmf_gateway_request_seconds`, `cmf_worker_job_seconds`, `cmf_agent_call_seconds`, `cmf_approval_pending_count`, `cmf_audit_events_total`, `cmf_failure_mode_total{mode=...}`. Business KPIs (7–12, per the locked telemetry design in `docs/architecture.md` §10): `cmf_agent_run_total{agent_id,connector,outcome}`, `cmf_agent_tokens_total{agent_id,model,kind}`, `cmf_approval_decided_total{agent_id,decision,decided_via}`, `cmf_approval_decision_seconds{agent_id,decided_via}` (histogram), `cmf_messages_total{agent_id,direction}`, `cmf_memory_search_total{agent_id,hit}`. OpenTelemetry traces remain deferred post-launch.
- **NFR12**: Solo-maintainer bandwidth: every epic must be implementable as 3–8 micro-tasks by one agent on one feature branch. No epic touches >8 files or has >5 acceptance criteria; splits required if it does.

---

## 3. User Interface Design Goals

### Overall UX Vision

ChatMyFleet's UX premise is that chat is the operator's home, not a notification channel. About 95% of operations land in Slack — Staff never opens the Web UI. The Web UI is the admin escape hatch for the 5% of work chat genuinely can't do well (bulk operations, visual analytics, multi-step config, policy editing). Full UX direction in `brief/chatmyfleet/ux.md`; what's below is the Loop 1 summary that drives Loop 2 design briefs.

### Key Interaction Paradigms

- **Buttons before slash commands.** Rich approval messages with inline `[Approve] [Decline]` buttons lead; slash commands are a power-user backdoor.
- **`@mention`-driven natural-language interaction.** Operators talk to agents like teammates (`@finance-ai check refund 402`); ChatMyFleet routes by `@mention`, topic, or default.
- **Audit log links back to chat.** Every audit row carries the Slack thread URL; clicking opens the thread at the moment of the action. This is the differentiator most products miss.
- **Policies as editable documents, not settings panels.** Each policy reads like an operational handbook; admins edit lines directly; risky overrides surface warnings but stay allowed.
- **Email is a first-class fallback from launch.** When the Slack-side approver is offline, the approval escalates to email with the same buttons; the audit log links back to the original Slack thread regardless of where the approval click happened.
- **Permission discoverability.** Slash commands like `/rename` exist in both namespaces; if Staff tries to run one, ChatMyFleet returns "Ask your admin" with a tooltip explaining what the command does. Don't hide what Staff can't do.

### Core Screens and Views

- **Slack — primary surface (Staff + User Admin).** Approval message cards, agent reply threads, slash command output, `/help` discoverability, `@mention`-routed conversations, `/audit` inline last-10 view, daily brief DM.
- **Web UI — secondary surface (User Admin only). Five pages at v0.4.0:**
  - Plugged-in Agents (list with live status, add, rename, start/stop, rotate keys)
  - Audit Log (search, filter, export, chat-thread linkback)
  - Policies (editable documents: Approval, Proactive Messaging, House Style, Error Handling, BYOK Security)
  - Members & Roles (invite Staff, set permissions, deactivate)
  - Dashboard (KPI cards + per-agent activity table + failure-mode counts + time-range selector; aggregations from `audit_events` + `approval_requests`, with linkback to filtered Audit Log)
- **Email — fallback surface for HITL.** Plain HTML email with `[Approve] [Decline]` links signed with HMAC + expiry; click → Web UI confirms decision → audit row written with linkback.

Additional admin surfaces (Debug, Channels, Memory inspector) defer to post-launch on adopter demand.

### Accessibility: WCAG AA

Web UI targets WCAG AA at launch (keyboard navigation, color contrast, focus states, semantic HTML, ARIA labels on icon-only buttons). Slack-side UX inherits Slack's own accessibility. Higher accessibility tiers (WCAG AAA, screen-reader optimisation beyond defaults) iterate post-launch on adopter signal.

### Branding

No fixed brand system at v0.4.0. The product voice (warm, plain, occasionally funny — see the 6-axis teammate-feel rubric in `brief/chatmyfleet/ux.md`) is the only "brand" element. Web UI styling uses Tailwind defaults + shadcn/ui components; visual identity work happens post-launch with `chatmyfleet.com` polish.

### Target Device and Platforms: Web Responsive

Web UI is responsive (desktop primary, tablet/mobile usable for read-only audit lookups). Slack-side UX inherits Slack's own platform reach (web, macOS, Windows, Linux, iOS, Android). Email fallback is plain HTML, works on every mail client.

---

## 4. Technical Assumptions

### Repository Structure: Monorepo

Single `chatmyfleet/` repo. The `cmf/` Python package holds backend modules (`gateway.py`, `worker.py`, `db/`, `channels/`, `connectors/`, `memory/`, `approvals/`, `audit/`, `commands/`, `web/`, `observability/`); the React 18 SPA bundles into `cmf/web/static/`; migrations live at `migrations/`; tests split into `tests/backend/`, `tests/frontend/`, `tests/manual/`. One PR can touch backend + frontend + migrations + docs + tests, which the Orbit rule "doc updates ship in the same PR as code" requires.

### Service Architecture

**Modular monolith with two entry points, deployed as 4 containers.** `cmf-gateway` and `cmf-worker` are two ASGI / asyncio entry-point modules of the same `cmf/` package, sharing models, config, crypto, and audit. Deployed as separate containers (`cmf-gateway`, `cmf-worker`, `postgres`, `redis`). `cmf-scheduler` and `cmf-embeddings` stay folded into `cmf-worker` at v0.4.0; both splits documented in `architecture-deep.md` as Year-2 options when production load justifies. **The gateway/worker split is non-negotiable** — Slack's 3-second webhook timeout forbids inline agent polling.

### Testing Requirements

**Unit + Integration + Manual SIT/UAT per epic** (full pyramid for load-bearing surfaces; happy-path-only for polish surfaces).

- **`tests/backend/`** — pytest + pytest-asyncio. Unit tests for pure logic. Integration tests run against a real Postgres + Redis (via `docker-compose` for dev; CI uses GitHub Actions service containers). No mocked databases for load-bearing paths.
- **`tests/frontend/`** — Playwright against the Web UI, mocked backend APIs at the network layer. Happy path for all 5 pages at v0.4.0.
- **`tests/manual/epic-X.md`** — step-by-step SIT/UAT guides, executed by `/orbit4c-test` (AI) or by the maintainer. Runs once per epic before merge.
- **Micro-task test gate:** each Loop 4 micro-task ships with its own passing test before being marked complete. No epic closes without a green full-suite run + manual test pass.

### Additional Technical Assumptions and Requests

- **Language:** Python 3.11+ for backend, TypeScript for Web UI. No second backend language.
- **Env / deps:** `uv` for Python (lockfile-driven). `npm` for the bundled SPA (Vite-managed).
- **No CI/CD environment exists yet.** GitHub Actions + `uv` is the planned pick. Epic 1 stands up CI as part of the foundation epic so subsequent epics have a green baseline.
- **No paid external accounts pre-provisioned.** Slack workspace, OpenAI API key, SMTP provider — each gets confirmed before the relevant epic starts. Spikes that hit live services need credentials from the maintainer.
- **BYOK across the stack.** LLM provider (OpenAI / Anthropic / Gemini), connector credentials, SMTP — all bring-your-own. ChatMyFleet pays nothing.
- **Embedding model is operator-swappable via env var** (`EMBEDDING_MODEL`). Default locked by Spike 3: `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` (Apache 2.0, 384-dim) via FastEmbed (ONNX, no torch); documented swap `paraphrase-multilingual-mpnet-base-v2` (Apache 2.0, 768-dim). The chosen model's output dim must match the `memory_*.embedding` column dim, fixed per deploy at Epic 2's Alembic migration (model change = re-embed migration). FastEmbed version pinned; model baked into the distroless image (see FR21, `docs/architecture.md` §7).
- **AESGCM with static `.env` key for BYOK secret encryption at v0.4.0.** KMS abstraction (AWS / GCP / Vault) deferred post-launch. Per-tenant master keys deferred to v1.x (Cloud multi-tenant).
- **Append-only audit via Postgres role permission at v0.4.0.** Ed25519 hash chain + signing land at v0.9.0 when a compliance customer needs it. Append-only is the v0.1.0 foundation.
- **Single multi-stage Dockerfile, distroless-python runtime, `linux/amd64` only at v0.4.0.** Multi-arch + Helm chart defer post-launch.
- **No KMS abstraction, no hot-update via LISTEN/NOTIFY, no Helm chart, no Ed25519 signing, no separate scheduler/embeddings containers, no second channel, no second protocol-level connector, no affiliate-partner integrations** at v0.4.0. Full defer list in `brief/chatmyfleet/mvp-architecture.md` "Explicit defer list" table and `docs/user_brief.md` §"What we're NOT building at v0.4.0".
- **Branching:** `feature/<epic-name>` per Loop 4 epic. Direct commits to `main` only after the repo flips public at Week 4 (CHANGELOG-driven daily commits between tagged minors). No direct commits to `main` during private dev (Weeks 1-3) — feature branches throughout.
- **License:** Apache 2.0 across the entire repo. Locked. No relicensing under any circumstance.

---

## 5. Technical Spikes

9 spikes total, each scoped to run as the **first micro-task (MT-01)** of the epic that depends on it. **The 5 credential-free spikes (3, 6, 7, 8, 9) were run early during Loop 1 — all PASS** (results in `docs/spikes/spike-N-*.md`, consolidated in `docs/spikes_result.md`). The 4 credential-gated spikes (1, 2, 4, 5 — need Slack/OpenAI/SMTP keys) remain deferred to Loop 4 pre-flight. Spikes 1–6 are carried from `docs/user_brief.md` §"Open doubts"; Spikes 7–9 were identified during the Loop 1 architecture review (they validate non-negotiable architectural call #3 and the foundation epics' backbone — see `docs/architecture.md` §3, §6).

| # | Spike | Target epic | Credentials |
|---|---|---|---|
| 1 | Slack 3s ack on FastAPI | Epic 4 | Slack workspace |
| 2 | OpenAI Assistants POLL pattern | Epic 5 | OpenAI key |
| 3 | Embedding model + pgvector HNSW + swap | Epic 7 | None (OpenAI key optional) |
| 4 | NL rule extraction reliability | Epic 8 | OpenAI/Anthropic key |
| 5 | Email link signing + audit linkback | Epic 10 | SMTP + Slack |
| 6 | Builder API idempotency | Epic 15 | None |
| 7 | Append-only audit two-role DB setup | Epic 2 | None |
| 8 | Distroless image + cold-boot | Epic 1 | None |
| 9 | Redis Streams reclaim / restart-safety | Epic 3 | None |

### Spike 1

**Name**: Slack signature verify + Block Kit round-trip on FastAPI inside Slack's 3-second ack window
**Goal**: Confirm `slack-bolt` `AsyncApp` mounted via `AsyncSlackRequestHandler` on FastAPI acknowledges Slack webhooks in <500ms p95 under realistic payload sizes, while enqueuing the work to Redis Streams for the worker.
**Target Epic**: Epic 4 — Slack Adapter + First Round-Trip (runs as first micro-task)
**Credentials needed**: Slack workspace + bot token (maintainer-provided)
**Status**: Pending
**Results**: (To be filled after execution)

### Spike 2

**Name**: OpenAI Assistants `requires_action` POLL pattern under Redis Streams workers
**Goal**: Validate polling cadence vs latency trade-off; design straggler-run handler; confirm no double-execution under worker retry / restart. Measure cost (tokens + wall-clock) of polling at the chosen cadence.
**Target Epic**: Epic 5 — OpenAI Assistants Connector + Conversation State Machine (runs as first micro-task)
**Credentials needed**: OpenAI API key (maintainer-provided)
**Status**: Pending
**Results**: (To be filled after execution)

### Spike 3

**Name**: Embedding-model selection + pgvector HNSW recall/latency at 200-rule scale + runtime model-swap design
**Goal**:
1. Confirm `mvp-architecture.md`'s sub-50ms p95 retrieval claim against pgvector + HNSW at 200-rule scale (model-agnostic measurement).
2. Compare two open-source candidates head-to-head: `intfloat/multilingual-e5-small` (MIT, 384-dim, ~470MB) and `google/embeddinggemma-300m` (Apache 2.0 + Gemma terms, 768-dim default with Matryoshka to 128, ~1.2GB). Optional baseline: `openai:text-embedding-3-small` (1536-dim) if a key is available.
3. Measure **Thai-language recall** specifically — the test corpus must include Thai operator-rule phrasings alongside English. This is the differentiator for the user-zero Thai SMB deployment and the v0.8 LINE adapter's Asian-SMB target.
4. Design the **runtime model-swap mechanism**: how an operator picks a model at `docker compose up` time (env-var contract), how the `memory_*.embedding` column dim is reconciled with the chosen model (fixed-per-deploy / Matryoshka-truncate-to-fixed-dim / migration-on-model-change), and what the post-install upgrade path looks like.
**Target Epic**: Epic 7 — Memory Schemas + pgvector + HNSW Index (runs as first micro-task)
**Credentials needed**: None for the two open-source models (local CPU inference via FastEmbed). OpenAI API key optional if comparing the `text-embedding-3-small` baseline.
**Deliverables (recorded in `docs/spikes/spike-3-embedding-and-pgvector.md`)**:
- Per-model: native dim, container weight delta, CPU inference latency (p50/p95), Thai recall@10 and English recall@10 on the test corpus
- **Recommended default** (one model named) for v0.4.0 with rationale
- **Model-swap design**: env-var contract (e.g., `EMBEDDING_MODEL=fastembed:multilingual-e5-small`), column-dim strategy, and documented operator upgrade path
**Status**: Done
**Results**: **SUCCESS** (2026-05-28). FastEmbed lacks e5-small + embeddinggemma → pivoted (no-torch) to `paraphrase-multilingual-MiniLM-L12-v2` (384d, 0.22GB) vs `paraphrase-multilingual-mpnet-base-v2` (768d, 1.0GB). Both: Thai recall@10 = 100%; p95 retrieval @200 rows = 0.46ms / 1.88ms (~100× under the 50ms target). **Recommended default: MiniLM-384** (smallest, 100% Thai, honors 5-min install); mpnet-768 documented swap (100% all metrics). Column dim fixed-per-deploy. Full result + FR21/architecture lock follow-up in `docs/spikes/spike-3-embedding-and-pgvector.md`.

### Spike 4

**Name**: NL rule extraction reliability with ambiguity fallback
**Goal**: Build corpus of 30+ realistic operator phrasings ("lower auto-approve threshold to $300", "auto-approve refunds under $200 for repeat customers", "never auto-approve international shipping"). Measure structured-policy extraction accuracy. Design ambiguity-detection fallback that triggers a confirmation prompt when extraction confidence < threshold.
**Target Epic**: Epic 8 — NL Rule Extraction + /rule Slash Command + Pre-Context Injection (runs as first micro-task)
**Credentials needed**: OpenAI or Anthropic API key (maintainer-provided)
**Status**: Pending
**Results**: (To be filled after execution)

### Spike 5

**Name**: Email approval link signing + click → audit linkback
**Goal**: Design signed-link scheme (HMAC + expiry; resist replay). Verify the Slack-thread linkback survives the email round-trip — clicking an approval link from email writes an audit row that links back to the originating Slack thread URL.
**Target Epic**: Epic 10 — Builder API + Email Fallback (runs as first micro-task)
**Credentials needed**: SMTP provider (SendGrid / Resend / Postmark / self-hosted Postfix) + Slack workspace
**Status**: Pending
**Results**: (To be filled after execution)

### Spike 6

**Name**: Idempotency keys on the blocking Builder API endpoint
**Goal**: Design idempotency-key scheme that handles long-poll behaviour cleanly. Test under simulated network retry storm: same key submitted N times concurrently must execute the action exactly once and return the same response to all callers.
**Target Epic**: Epic 15 — Failure Hardening + Builder API Idempotency + Launch Polish (runs as first micro-task)
**Credentials needed**: None (local-only test scenario)
**Status**: Done
**Results**: **SUCCESS** (2026-05-28, `postgres:16` + asyncpg, 12 concurrent connections). `INSERT … ON CONFLICT (key) DO NOTHING RETURNING` claim pattern: exactly one winner runs the side effect, losers poll the row for the cached response; new key executes again. Same Postgres-row guard doubles as the worker side-effect guard for Spike 9's at-least-once reclaim. Full result in `docs/spikes/spike-6-idempotency.md`.

### Spike 7

**Name**: Append-only audit enforcement via Postgres role permission (two-role DB setup)
**Goal**: Validate non-negotiable architectural call #3 against running code. Prove a two-role setup works end-to-end: a privileged migration role runs Alembic DDL + creates the pgvector extension; a restricted app role gets `INSERT` (plus sequence `USAGE`) but **no `UPDATE`/`DELETE`** on `audit_events`. Confirm SQLAlchemy 2.0 async + asyncpg connect cleanly as the restricted role, and that the NFR5 enforcement test passes — the app role's `UPDATE` and `DELETE` on `audit_events` are rejected by Postgres, not by application code.
**Target Epic**: Epic 2 — Postgres Schema + Audit Append-Only Enforcement (runs as first micro-task)
**Credentials needed**: None (local Postgres via docker-compose)
**Status**: Done
**Results**: **SUCCESS** (2026-05-28, `pgvector/pgvector:pg16` + SQLAlchemy 2.0.50 async). App role INSERT/SELECT succeed; UPDATE and DELETE on `audit_events` rejected by Postgres with SQLSTATE 42501. Two-role pattern + GRANTs confirmed; key gotcha: `IDENTITY` PK needs `GRANT USAGE ON SEQUENCE` for the app role to INSERT. Full result + reusable GRANT block in `docs/spikes/spike-7-audit-role-enforcement.md`.

### Spike 8

**Name**: Distroless `linux/amd64` image with native deps + `docker compose up` cold-boot to green
**Goal**: De-risk the five-minute-install launch criterion. Build a multi-stage Dockerfile with a distroless runtime (no shell, no package manager) that correctly bundles native wheels (`asyncpg`, `cryptography`, `argon2-cffi`) — and, **conditional on Spike 3 picking a local embedding model**, the FastEmbed/`onnxruntime` runtime. Confirm `docker compose up` on a fresh box reaches a green `/health` with correct healthcheck ordering (postgres ready → migrate-on-boot → gateway + worker up) inside the five-minute budget.
**Target Epic**: Epic 1 — Project Skeleton + CI Foundation (runs as first micro-task)
**Credentials needed**: None (local-only)
**Status**: Done
**Results**: **SUCCESS** (2026-05-28, `gcr.io/distroless/python3-debian12`, arm64). Build 87s → 558MB; cold-boot to green `/health` in **7s** (≪ 5-min budget). asyncpg + AESGCM + argon2 + fastembed/onnxruntime all work in distroless. **Key gotcha:** onnxruntime needs `libgomp.so.1` (copy from builder — distroless lacks it); install-to-target-dir not venv; bake the model into the image; shell-less healthcheck via `python3`+urllib. amd64 to re-verify in Epic 1 CI. Full result + Dockerfile pattern in `docs/spikes/spike-8-distroless-cold-boot.md`.

### Spike 9

**Name**: Redis Streams consumer-group restart-safety / reclaim
**Goal**: Validate the NFR9 restart-safety guarantee against running code. Prove the consumer-group pattern (`XREADGROUP` + `XACK` + `XAUTOCLAIM` of stale pending entries) recovers cleanly after a simulated worker crash mid-processing: the unacked event is reclaimed by another worker, processed exactly once (no loss, no double-execution), across the 6-stream event bus. Confirm the pattern composes with Postgres-as-source-of-truth so in-flight state survives a full worker restart.
**Target Epic**: Epic 3 — Redis Streams Event Bus + Worker Loop (runs as first micro-task)
**Credentials needed**: None (local Redis via docker-compose)
**Status**: Done
**Results**: **SUCCESS** (2026-05-28, `redis:7` + redis-py 7.4.0). `XAUTOCLAIM` reclaimed all stale PEL entries after a simulated crash; zero loss, PEL fully drained. **Key finding:** reclaim is *at-least-once*, so `cmf-worker` side effects MUST be idempotent (state-machine guards + Spike 6 keys + `correlation_id` audit dedupe) — recorded as an Epic 3 design constraint. Full result in `docs/spikes/spike-9-redis-streams-reclaim.md`.

---

## 6. Epic List

15 epics, each a vertical slice deliverable as 3–8 micro-tasks on a `feature/<epic-name>` branch by one agent. Statuses follow the Orbit lifecycle: `Draft → Approved → In Progress → Review → Done`. All start at `Draft` until user approves this spec; on approval, they all advance to `Approved` so Loop 3 / Loop 4 can pick them up.

- **Epic 1: Project Skeleton + CI Foundation** [Status: Approved] — Stand up the repo, toolchain (`uv` + `ruff` + `mypy` + `pytest`), Dockerfile, docker-compose.yml, GitHub Actions CI green, and a `/health` canary endpoint, so all subsequent epics build on a known-green baseline.
- **Epic 2: Postgres Schema + Audit Append-Only Enforcement** [Status: Approved] — Alembic migrations for all 8 tables, SQLAlchemy 2.0 async models, AESGCM crypto for connector_config, and the database-role permission that proves `audit_events` is append-only.
- **Epic 3: Redis Streams Event Bus + Worker Loop** [Status: Approved] — Wire Redis 7, design the 6-stream event bus, stand up the `cmf-worker` asyncio main loop, and prove a test event flows gateway → stream → worker → audit row.
- **Epic 4: Slack Adapter + First @agent Round-Trip** [Status: Approved] — Mount `slack-bolt` `AsyncApp` on FastAPI via `AsyncSlackRequestHandler`, validate signature, render Block Kit, and prove `@cmf hello` echoes back inside Slack's 3-second window. **Spike 1 resolves here.**
- **Epic 5: OpenAI Assistants Connector + Conversation State Machine** [Status: Approved] — Wire `AsyncOpenAI` with the POLL pattern, implement the 5-state conversation state machine, write audit rows on every transition, and prove `@finance-ai hello` routes to a real OpenAI Assistant and replies in Slack with a full audit trail. **Spike 2 resolves here. This is the v0.1.0 capability.**
- **Epic 6: v0.1.0 Public Repo Trust Artifacts + Landing** [Status: Approved] — Populate `LICENSE` (Apache 2.0), `TRADEMARK.md`, `SECURITY.md`, `CONTRIBUTING.md`, `MAINTENANCE.md`, `CHANGELOG.md`, `AFFILIATES.md`, `README.md`, `.env.example`, demo GIF, Sponsors page link, and a minimal `chatmyfleet.com` landing. **Tag: v0.1.0 — repo flips public Wed Jun 24.**
- **Epic 7: Memory Schemas + pgvector + HNSW Search** [Status: Approved] — Build memory_rules / memory_entities / memory_decisions tables, inline embedding worker in `cmf-worker`, pgvector HNSW index, and the `memory.search` + `memory.write_rule` Builder API endpoints. **Spike 3 resolves here.**
- **Epic 8: NL Rule Extraction + /rule Command + Pre-Context Injection** [Status: Approved] — LLM-driven NL → structured policy extraction with ambiguity fallback, the `/rule add` slash command as deterministic backup, and pre-context injection for non-MCP agents. Prove an operator's "auto-approve refunds under $200" survives a simulated connector swap. **Spike 4 resolves here. Tag: v0.2.0.**
- **Epic 9: HITL Approval State Machine + Block Kit Buttons** [Status: Approved] — `approval_requests` state machine (pending/escalated/decided), Block Kit approval rendering, and auto-approve threshold counters with trust-delta logic. Prove `requires_action` → inline buttons → click → resume round-trip under 10s.
- **Epic 10: Builder API + Email Fallback + Timeout Tick** [Status: Approved] — All 5 Builder API endpoints (blocking + callback approval variants), aiosmtplib SMTP integration, signed approval-link verify, 60s asyncio timeout tick task (4h → 24h → 48h ladder), and audit linkback from email clicks. **Spike 5 resolves here. Tag: v0.3.0.**
- **Epic 11: Web UI Foundation + Plugged-in Agents Page** [Status: Approved] — Scaffold React 18 + Vite + Tailwind + shadcn/ui as a bundled SPA served by FastAPI, set up Argon2id-hashed API tokens for auth, and ship Page 1 (Plugged-in Agents: list, add, rename, start/stop, rotate keys).
- **Epic 12: Audit Log Page + Chat-Thread Linkback** [Status: Approved] — Page 2 (Audit Log: search, filter, export, chat-thread linkback on every row). Prove that clicking any audit entry opens the originating Slack thread — the differentiator.
- **Epic 13: Policies Page + Members & Roles Page** [Status: Approved] — Page 3 (Policies as editable documents with risky-override warning levels) and Page 4 (Members & Roles: invite Staff, set permissions, deactivate). Round out the User Admin Web UI surface.
- **Epic 14: Web UI Dashboard Page + Aggregation API** [Status: Approved] — 5th admin page (Dashboard) showing aggregated operator-visible KPIs from `audit_events` + `approval_requests`: message volume per agent, approval count + decision breakdown + median decision time, per-agent token usage, failure-mode counts, time-range selector. Wires the 6 business-KPI Prometheus counters (NFR11 items 7–12). Vertical slice: User Admin opens Dashboard → sees 7-day numbers for every agent.
- **Epic 15: Failure Hardening + Builder API Idempotency + Launch Polish** [Status: Approved] — All 7 named failure modes tested with explicit recovery; idempotency keys on the blocking Builder API endpoint; the remaining slash commands (`/about`, `/brief`, `/policy`, `/rename`, `/start`, `/stop`, `/dashboard`); demo GIF; README final pass; v0.4.0 tag prep. **Spike 6 resolves here. Tag: v0.4.0 — PH LAUNCH Wed Aug 19.**

---

## 7. Epic Details

> Per Orbit `spec_tmpl` rule: each epic lists 3–8 Estimated Micro-Tasks as action-phrase titles only. **No implementation steps, no code, no acceptance-criteria detail here** — those land in `docs/epics/epic-{N}-{name}.md` during Loop 3.

### Epic 1: Project Skeleton + CI Foundation

**Status**: Approved

Stand up the repository, toolchain, container layout, and CI pipeline so every subsequent epic builds on a known-green baseline. Ships a `/health` canary endpoint that proves the gateway boots and is reachable through Docker Compose. The vertical slice is "a self-hoster can `docker compose up` and curl `/health`." **Spike 8 resolves as MT-01.**

**Estimated Micro-Tasks** (action titles only — no implementation detail):
- MT-01: **Spike 8** — distroless `linux/amd64` image with native deps (+ conditional local embedding runtime) + `docker compose up` cold-boot to green health; record results in `docs/spikes/spike-8-distroless-cold-boot.md`
- MT-02: Initialise `uv`-managed Python 3.11+ project + `pyproject.toml`
- MT-03: Configure `ruff` + `mypy` + `pytest` baseline + pre-commit hooks
- MT-04: Author multi-stage Dockerfile (distroless runtime, `linux/amd64`)
- MT-05: Author `docker-compose.yml` for 4 containers (`cmf-gateway`, `cmf-worker`, `postgres`, `redis`) with healthcheck ordering
- MT-06: Wire FastAPI app skeleton with `/health` endpoint + structlog config
- MT-07: Set up GitHub Actions CI (uv install + lint + type-check + tests) and prove green

### Epic 2: Postgres Schema + Audit Append-Only Enforcement

**Status**: Approved

Build the 8-table Postgres schema with SQLAlchemy 2.0 async models, AESGCM encryption for `connector_config` secrets, and the role-permission enforcement that makes `audit_events` provably append-only. The vertical slice is "a developer runs `alembic upgrade head` and an integration test proves the app role cannot UPDATE or DELETE audit rows." **Spike 7 resolves as MT-01.**

**Estimated Micro-Tasks**:
- MT-01: **Spike 7** — two-role DB setup (privileged migration role vs restricted app role) + append-only `audit_events` enforcement via GRANTs, with SQLAlchemy 2.0 async + Alembic + pgvector extension creation; record results in `docs/spikes/spike-7-audit-role-enforcement.md`
- MT-02: Add Alembic + initial migration for `tenants` + `users` + `agents` (with `connector_config` JSONB)
- MT-03: Add migration for `conversations` + `approval_requests`
- MT-04: Add migration for `audit_events` + database role + GRANTs that omit UPDATE/DELETE on `audit_events` (incl. sequence `USAGE`)
- MT-05: Add migration for `memory_rules` + `memory_entities` + `memory_decisions` (no HNSW index yet — that's Epic 7)
- MT-06: Implement SQLAlchemy 2.0 typed models + async session factory + AESGCM crypto helpers for `connector_config`
- MT-07: Write integration test that asserts app role can INSERT into `audit_events` but UPDATE and DELETE are rejected by Postgres (NFR5)

### Epic 3: Redis Streams Event Bus + Worker Loop

**Status**: Approved

Wire Redis 7, design the 6-stream event bus (`gateway.inbound`, `worker.outbound`, `approval.pending`, `approval.resolved`, `embedding.requested`, `audit.write`), and stand up the `cmf-worker` asyncio main loop. The vertical slice is "the gateway publishes a test event, the worker consumes it, the worker writes a row to `audit_events`." **Spike 9 resolves as MT-01.**

**Estimated Micro-Tasks**:
- MT-01: **Spike 9** — Redis Streams consumer-group restart-safety: `XREADGROUP` + `XACK` + `XAUTOCLAIM` of stale pending entries after a simulated worker crash, proving no message loss and no double-processing (NFR9); record results in `docs/spikes/spike-9-redis-streams-reclaim.md`
- MT-02: Add `redis-py` async + Streams client wrapper with consumer-group setup
- MT-03: Define the 6 stream names + payload schemas (pydantic v2 models)
- MT-04: Implement `cmf-worker` asyncio main loop entrypoint with consumer-group claim + ack + stale-entry reclaim
- MT-05: Wire test publish from gateway → consume by worker → audit row written
- MT-06: Add `correlation_id` propagation through the stream payload and into the audit row

### Epic 4: Slack Adapter + First @agent Round-Trip

**Status**: Approved

Mount `slack-bolt` `AsyncApp` on FastAPI via `AsyncSlackRequestHandler`, validate Slack signing-secret signatures, render Block Kit messages, and prove the gateway acknowledges Slack webhooks inside the 3-second window. The vertical slice is "in a real Slack workspace, `@cmf hello` echoes back via the worker — with audit rows for the inbound and outbound." **Spike 1 resolves as MT-01.**

**Estimated Micro-Tasks**:
- MT-01: **Spike 1** — `slack-bolt` `AsyncApp` mounted on FastAPI; measure ack latency under realistic payload sizes; record results in `docs/spikes/spike-1-slack-3s-ack.md`
- MT-02: Wire Slack signing-secret verification + event subscription handler
- MT-03: Enqueue inbound Slack message to `gateway.inbound` stream; ack to Slack in <500ms
- MT-04: Worker consumes inbound stream, renders echo Block Kit response, posts back via `chat.postMessage`
- MT-05: Audit rows written for inbound + outbound events with `correlation_id` linkage
- MT-06: Write integration test against a recorded Slack signing fixture; manual SIT script in `tests/manual/epic-4.md`

### Epic 5: OpenAI Assistants Connector + Conversation State Machine

**Status**: Approved

Wire `openai-python` `AsyncOpenAI` with the POLL pattern for OpenAI Assistants runs, implement the 5-state conversation state machine (`idle → routing → in_run → awaiting_approval → completed`), and write audit rows on every state transition. The vertical slice is "in a real Slack workspace, `@finance-ai hello` routes to a real OpenAI Assistant, the assistant's reply renders back to Slack, and the full audit trail is queryable." **Spike 2 resolves as MT-01. This is the v0.1.0 capability.**

**Estimated Micro-Tasks**:
- MT-01: **Spike 2** — OpenAI Assistants POLL pattern; measure polling cadence vs latency; design straggler-run handler; record results in `docs/spikes/spike-2-openai-assistants-poll.md`
- MT-02: Implement `connectors/openai_assistants.py` (Connector Protocol implementation); capture token usage (`prompt_tokens`, `completion_tokens`, `total_tokens`), model name, duration, and `tool_call_count` on every `agent.run.completed` audit row per FR22
- MT-03: Implement 5-state conversation state machine + Postgres persistence in `conversations` table
- MT-04: Implement basic slash commands needed for v0.1: `/help`, `/agent add`, `/agent list`, `/agent edit`, `/fleet`, `/audit` (full command + `@mention`-routing surface in `docs/architecture.md` §6.9)
- MT-05: End-to-end test: send `@finance-ai hello` → state machine transitions → connector polls → reply renders → audit trail complete
- MT-06: Manual SIT script in `tests/manual/epic-5.md` covering the v0.1.0 acceptance for "first round-trip"

### Epic 6: v0.1.0 Public Repo Trust Artifacts + Landing

**Status**: Approved

Populate every trust-signal artifact required for the repo to flip public on Wed Jun 24, and publish a minimal `chatmyfleet.com` landing + Sponsors page. The vertical slice is "a first-time visitor lands on the public GitHub repo, reads the README, clones, runs `docker compose up`, and has ChatMyFleet talking to Slack — and the credibility signals (license, trademark, maintenance commitment, security policy, contributing guide, affiliate disclosure) all exist." **Tag: v0.1.0.**

**Estimated Micro-Tasks**:
- MT-01: Write `LICENSE` (Apache 2.0 verbatim) + `TRADEMARK.md` + `SECURITY.md` + `CONTRIBUTING.md`
- MT-02: Write `MAINTENANCE.md` (5-year commitment + cadence table) + `CHANGELOG.md` (v0.1.0 entry) + `AFFILIATES.md`
- MT-03: Write public `README.md` (from `brief/chatmyfleet/product.md`, OSS-builder voice) + `.env.example` with placeholders only
- MT-04: Author demo GIF showing the v0.1.0 round-trip (Slack `@finance-ai hello` → reply)
- MT-05: Stand up minimal `chatmyfleet.com` landing page + Sponsors page link
- MT-06: Flip repo public Wed Jun 24; tag `v0.1.0`; first dev.to / LinkedIn post

### Epic 7: Memory Schemas + pgvector + HNSW Search

**Status**: Approved

Add the pgvector HNSW index on `memory_rules.embedding`, stand up the inline embedding worker in `cmf-worker`, and expose `memory.search` + `memory.write_rule` via the Builder API. The vertical slice is "an agent calls `memory.write_rule('auto-approve refunds under $200 for repeat customers')`, then `memory.search('what refund threshold did the operator set?')` returns it in <50ms." **Spike 3 resolves as MT-01.**

**Estimated Micro-Tasks**:
- MT-01: **Spike 3** — open-source embedding-model bake-off (`multilingual-e5-small` vs `embeddinggemma-300m`) + pgvector HNSW recall/latency at 200-rule scale + runtime model-swap design; record results in `docs/spikes/spike-3-embedding-and-pgvector.md`
- MT-02: Add Alembic migration for HNSW index on `memory_rules.embedding` with tuned build params
- MT-03: Implement embedding worker (inline in `cmf-worker`) listening on `embedding.requested` stream
- MT-04: Implement `memory/search.py` (top-K semantic search with score threshold)
- MT-05: Implement Builder API endpoints `POST /v1/agents/{id}/memory/search` + `POST /v1/agents/{id}/memory/write_rule`
- MT-06: Integration test: write 200 rules, search, assert p95 latency < 50ms (per NFR2)

### Epic 8: NL Rule Extraction + /rule Command + Pre-Context Injection

**Status**: Approved

Build LLM-driven natural-language → structured-policy extraction with ambiguity fallback, the deterministic `/rule add` slash command, and pre-context injection for non-MCP agents. The vertical slice is "an operator says `@finance-ai auto-approve refunds under $200 for repeat customers` → ChatMyFleet stores it → a simulated connector swap to a fresh OpenAI Assistant still applies the rule via pre-context injection." **Spike 4 resolves as MT-01. Tag: v0.2.0.**

**Estimated Micro-Tasks**:
- MT-01: **Spike 4** — NL rule extraction reliability + ambiguity fallback; record results in `docs/spikes/spike-4-nl-rule-extraction.md`
- MT-02: Implement `memory/extract.py` — LLM call (off-path) parsing operator phrases into structured policy mutations
- MT-03: Implement ambiguity-detection fallback (confidence < threshold → confirmation prompt in Slack thread)
- MT-04: Implement `/rule add` slash command as deterministic fallback (commands/rule.py)
- MT-05: Implement pre-context injection for non-MCP agents (top-K memory rules pushed into Assistant system context per run)
- MT-06: Connector-swap regression test: rule taught via NL extraction survives reconfiguring agent to a fresh OpenAI Assistant
- MT-07: Tag `v0.2.0`; CHANGELOG entry; LinkedIn post; dev.to #1 ("Memory that survives agent swaps")

### Epic 9: HITL Approval State Machine + Block Kit Buttons

**Status**: Approved

Implement the `approval_requests` state machine (`pending → decided` or `pending → escalated → decided` or `pending → expired`), render Block Kit approval messages with `[Approve] [Decline]` buttons, and add the auto-approve threshold counters with trust-delta logic. The vertical slice is "an OpenAI Assistant raises `requires_action`, ChatMyFleet renders inline buttons in Slack, the operator clicks Approve, the agent resumes — all in <10s p95."

**Estimated Micro-Tasks**:
- MT-01: Implement `approvals/policy.py` — state machine + transitions + Postgres persistence
- MT-02: Implement Block Kit rendering for approval messages (with context: what action, which policy triggered HITL, what happens on Y/N)
- MT-03: Wire Slack interactivity handler — button click → resolve approval state → resume agent run
- MT-04: Implement auto-approve threshold counter with trust-delta logic (per `ux.md` Conversation 5 + Auto-approve Thresholds section)
- MT-05: Integration test: simulate `requires_action` → render button → click → resume → audit complete; assert <10s p95
- MT-06: Manual SIT script in `tests/manual/epic-9.md` for the approval round-trip

### Epic 10: Builder API + Email Fallback + Timeout Tick

**Status**: Approved

Ship all 5 Builder API endpoints (with blocking + callback variants for approvals), the aiosmtplib-based email fallback path with signed-link verification, and the 60s asyncio timeout-tick task that drives the 4h → 24h → 48h escalation ladder. The vertical slice is "an offline approver gets an approval email after 4h, clicks the signed link, the audit row records the decision and links back to the originating Slack thread." **Spike 5 resolves as MT-01. Tag: v0.3.0.**

**Estimated Micro-Tasks**:
- MT-01: **Spike 5** — email approval link signing + click → audit linkback; record results in `docs/spikes/spike-5-email-linkback.md`
- MT-02: Implement Builder API blocking variant: `POST /v1/agents/{id}/approval_request?wait=blocking` (long-poll until decision or 48h expire)
- MT-03: Implement Builder API callback variant + `GET /v1/agents/{id}/approval_request/{req_id}` for poll-style consumers
- MT-04: Implement `approvals/email.py` — aiosmtplib + HTML email template + HMAC-signed approval link + click handler with linkback
- MT-05: Implement `approvals/timer.py` — 60s asyncio tick reconciling from Postgres (`approval_requests.timeout_at`) for the 4h/24h/48h ladder
- MT-06: End-to-end test: pending approval → 4h reminder → 24h email → click email link → decision recorded → audit linkback to Slack thread
- MT-07: Tag `v0.3.0`; CHANGELOG entry; LinkedIn post; dev.to #2 ("HITL as the contract")

### Epic 11: Web UI Foundation + Plugged-in Agents Page

**Status**: Approved

Scaffold the React 18 + Vite + Tailwind + shadcn/ui SPA bundled into `cmf/web/static/` and served by FastAPI, set up Argon2id-hashed API tokens for User Admin auth, and ship Page 1 (Plugged-in Agents). The vertical slice is "User Admin logs into the Web UI, sees the list of plugged-in agents with live status, and can add a new OpenAI Assistant via the form."

**Estimated Micro-Tasks**:
- MT-01: Scaffold Vite + React 18 + TypeScript + Tailwind + shadcn/ui in `cmf/web/static-src/`; wire build → `cmf/web/static/` bundle
- MT-02: Wire FastAPI to serve the SPA + auth middleware (Argon2id-hashed API tokens, login form, session cookie)
- MT-03: Implement JSON API `cmf/web/api.py` endpoints for agents list / add / rename / start / stop / rotate-key
- MT-04: Build Plugged-in Agents page (list with live status + add form + per-row actions)
- MT-05: Playwright happy-path test for the add-agent flow
- MT-06: Manual SIT script in `tests/manual/epic-11.md`

### Epic 12: Audit Log Page + Chat-Thread Linkback

**Status**: Approved

Ship Page 2 (Audit Log) — searchable, filterable, exportable, with chat-thread linkback on every row. The vertical slice is "User Admin opens the Audit Log page, filters to today + `@finance-ai`, clicks any row, the Slack thread opens at the moment of the action." **This is the differentiator the brief calls out as one most products miss.**

**Estimated Micro-Tasks**:
- MT-01: Implement JSON API `cmf/web/api.py` endpoint for audit search/filter with pagination
- MT-02: Implement JSON API endpoint for audit export (CSV / JSON streamed)
- MT-03: Build Audit Log page (search bar + filter dropdowns + paginated table + export button)
- MT-04: Implement chat-thread linkback (every row carries the Slack thread URL; clicking opens it in a new tab)
- MT-05: Playwright happy-path test: filter → click row → linkback URL is correct + opens

### Epic 13: Policies Page + Members & Roles Page

**Status**: Approved

Ship Page 3 (Policies as editable documents) and Page 4 (Members & Roles). The vertical slice is "User Admin opens Policies, edits the Approval Policy document in plain English, saves; opens Members & Roles, invites a new Staff member, sets permissions, the invitee receives an email and can log in."

**Estimated Micro-Tasks**:
- MT-01: Implement JSON API endpoints for policy read/write + risky-override warning levels (per `ux.md` table)
- MT-02: Build Policies page (document list + editor with inline warnings for risky overrides; warnings flagged `RISK_OVERRIDE_ACTIVE`)
- MT-03: Implement JSON API endpoints for members list / invite / set-permissions / deactivate
- MT-04: Build Members & Roles page (invite form + members table with role selector + deactivate action)
- MT-05: Implement invite email (reuses `approvals/email.py` SMTP path)
- MT-06: Playwright happy-path test for both pages

### Epic 14: Web UI Dashboard Page + Aggregation API

**Status**: Approved

Ship the 5th admin page (Dashboard) and wire the 6 business-KPI Prometheus counters that feed the same data story to the operator of the CMF deployment via Grafana. The vertical slice is "User Admin opens Dashboard, selects 7-day range, sees inbound message volume + approval count + decision breakdown + median operator decision time + per-agent activity table + failure-mode counts — all live, all clickable into the Audit Log."

**Estimated Micro-Tasks**:
- MT-01: Add Postgres indexes (or materialized view) on `audit_events(tenant_id, created_at, event_type, agent_id)` to make time-range aggregation queries fast
- MT-02: Implement JSON API endpoints under `/admin/api/dashboard/*` for the aggregations (messages, approvals + decision breakdown, decision-time percentile, per-agent activity, failure-mode counts, active-conversation gauge)
- MT-03: Build Dashboard React page (KPI cards + per-agent table + failure-mode list + time-range selector); use `recharts` for the sparkline (small dep, no vendor lock-in)
- MT-04: Wire the 6 new business-KPI Prometheus counters (`cmf_agent_run_total`, `cmf_agent_tokens_total`, `cmf_approval_decided_total`, `cmf_approval_decision_seconds`, `cmf_messages_total`, `cmf_memory_search_total`) into the existing audit-write paths
- MT-05: Playwright happy-path test — seed audit data, load Dashboard, assert numbers reflect the seed
- MT-06: Manual SIT script `tests/manual/epic-14.md` covering all KPI cards + time-range switching + linkback to Audit Log

### Epic 15: Failure Hardening + Builder API Idempotency + Launch Polish

**Status**: Approved

Make every one of the 7 named failure modes produce a visible operator error or audit event (no silent drops), add idempotency keys to the blocking Builder API endpoint, ship the remaining slash commands, finalise the demo GIF and README, and tag `v0.4.0` for PH launch on Wed Aug 19. **Spike 6 resolves as MT-01. Tag: v0.4.0 — PH LAUNCH.**

**Estimated Micro-Tasks**:
- MT-01: **Spike 6** — idempotency keys on blocking Builder API endpoint under retry storm; record results in `docs/spikes/spike-6-idempotency.md`
- MT-02: Implement idempotency-key middleware on Builder API write endpoints; integration test under simulated retry storm
- MT-03: Implement explicit recovery + test for each of the 7 failure modes (agent timeout >2min, agent 500, Slack rate-limit, memory unreachable, approval timeout, webhook delivery fail, state-machine inconsistency)
- MT-04: Implement remaining slash commands: `/about`, `/brief`, `/policy`, `/rename`, `/start`, `/stop`, `/dashboard`
- MT-05: Refresh demo GIF (full v0.4.0 flow); README final pass; CHANGELOG v0.4.0 entry
- MT-06: Run full `/orbit4c-test` epic test suite on `main`; tag `v0.4.0` Tuesday Aug 18; PH launch Wed Aug 19
- MT-07: Reply-storm support during launch day (issues, discussions, sponsorships)

---

## 8. Checklist Results Report

To be populated by `/orbit4c-test` and the QA `verify-changes` skill at each epic's exit. Not applicable at Loop 1 — this section becomes meaningful after Loop 3 plans the System Manifest + Acceptance Criteria for each epic.

---

## 9. Next Steps

### UX Expert Prompt

> Run `/orbit2b-UIUX-design` against `docs/spec.md` to produce the design brief for the v0.4.0 Web UI's 5 admin pages: Plugged-in Agents, Audit Log (with chat-thread linkback), Policies (editable documents), Members & Roles, Dashboard (KPI summary with linkback to Audit Log). Source the Slack-side UX from `brief/chatmyfleet/ux.md` verbatim — that surface is fully specified and does not need a new design brief. Output: `docs/design_briefs/` per the orbit2b 6-phase template.

### Architect / Planner Prompt

> Run `/orbit3-plan` against `docs/spec.md`, `docs/architecture.md`, and the design brief from Loop 2b to decompose each of the 15 epics into `docs/epics/epic-{N}-{name}.md` with System Manifest + Acceptance Criteria + Status mirrored into `docs/spec.md`. Scaffold `tests/manual/epic-{N}.md` for each. Register the agent board at `docs/agent_board.md`. After Loop 3, Loop 4 picks up Epic 1 on a `feature/epic-1-skeleton` branch.
