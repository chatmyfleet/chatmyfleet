# Project Brief: ChatMyFleet

> The Orbit Loop 1 brief — what this project is for, who it's for, what "launched" means, and what we're deliberately not building. Written for anyone who clones the repo and opens `docs/` to figure out what they're looking at. Updated when intent changes; not when public copy is reworded.
>
> Public narrative (front-door pitch, OSS positioning, UX direction, integration patterns) ships to the repo `README.md`, `MAINTENANCE.md`, `chatmyfleet.com`, and the in-repo `docs/architecture/`, `docs/ux/`, `docs/integrations/`, `docs/oss-philosophy.md`, `docs/affiliates.md` artifacts as each capability lands.

## The core problem

You build AI agents for clients. The first delivery feels great. Then reality: every operational decision flows through you. Your client's marketers, salespeople, and fulfillment ops can't operate the agents without your help. You're the bottleneck. You can't scale past 3-5 clients.

The market doesn't ship an operator-facing chat layer that wraps any agent platform — that handles HITL approval, audit, and a memory layer the client's team can teach directly — under a license that's safe to deploy commercially per client.

ChatMyFleet is the open-source chat surface that closes that gap. You deploy it on the client's Slack, wire your agents through a connector, and hand day-to-day operation to the client's team. You stay in support mode.

## Who this is for

Three audiences. The product is the same for all of them — only the surface changes.

- **The builder** — solo AI builder or a 1-5 person AI agency that has already shipped an agent (Lindy, OpenAI Assistants, LangGraph, n8n, custom Python) and needs a chat surface so the client's team can actually use it. Reads the README, runs `docker compose up`, configures the Slack workspace, wires the connector. Likely operates 2-10 ChatMyFleet deployments in parallel — one per client. The five-minute install contract is for this person.
- **The User Admin** — the client's ops lead. COO, CTO, head of ops at the client org. Lives in chat for day-to-day approvals like Staff, opens the Web UI for the 5% of work chat genuinely can't do well: bulk operations, visual analytics, multi-step config, policy editing, member management.
- **The Staff operator** — front-line ops person, agency account manager, SMB owner running their own shop. Lives entirely in Slack. Approves/declines agent actions inline, talks to agents via `@mention`, runs `/audit`, `/fleet`, `/policy` for read-only context. Never opens the Web UI. About 95% of operations land here.

The version-zero adopter is the founder, deploying ChatMyFleet for an existing AI sales agent already running in production for a Thai SMB client. That deployment runs real refund and discount approvals through HITL on launch day. The product is being designed against a live production constraint, then generalised into OSS — not the other way around.

## What "launched" looks like (success criteria for v0.4.0)

The v0.4.0 tag ships Wed 2026-08-19. By that date, all of the following are true:

- **Five-minute install holds.** Someone who has never touched ChatMyFleet before clones the public repo, runs `docker compose up` on a fresh Linux box, and gets an OpenAI Assistant replying in Slack within five minutes. First try. No manual config diving.
- **A real customer runs in production on launch day.** The founder's existing Thai SMB client uses ChatMyFleet for live refund/discount approvals. Zero messages silently dropped across the first two weeks of operation — the hard guarantee: no operator or agent message is ever silently dropped, every failure path produces either a visible error to the operator or an audit event the operator can see.
- **HITL works in both states.** Approver online → end-to-end round-trip in under ten seconds. Approver offline → escalation through the 4h → 24h → 48h timeout ladder, with email fallback resolving cleanly and the audit log linking back to the originating Slack thread.
- **Memory survives a connector swap.** A rule taught via natural language ("auto-approve refunds under $200 for repeat customers") still applies after the underlying agent is reconfigured to a different connector. Regression test covers this at v0.4.0.
- **Append-only audit is provably enforced.** Every state transition in `conversations`, `approval_requests`, and agent interactions writes exactly one `audit_events` row. The application database role has no UPDATE/DELETE permission on `audit_events` — enforcement is in Postgres, not application code.
- **Four meaningful tags land on their dated weeks.** v0.1.0 by 2026-06-24 (public repo + first round-trip), v0.2.0 by 2026-07-12 (memory layer), v0.3.0 by 2026-07-26 (HITL approval), v0.4.0 by 2026-08-19 (Web UI + launch polish).
- **The trust-signal artifacts exist.** `README.md`, `LICENSE`, `TRADEMARK.md`, `SECURITY.md`, `CONTRIBUTING.md`, `MAINTENANCE.md`, `CHANGELOG.md`, `AFFILIATES.md` are all populated and accurate by Week 4 (when the repo flips public). Sponsors page live. `chatmyfleet.com` landing live.
- **The maintenance contract is real.** Five years of best-effort maintenance from launch (2026-08-19 → 2031-08-19). Monthly minor releases on the first Friday. Monthly transparency posts from M4 (Oct 2026) onward. Security patches within 30 days. Breaking changes pre-announced by at least one minor version. The commitment is signed in `MAINTENANCE.md` and the project ships in a shape that can hold it.

## What we're NOT building at v0.4.0

Each item below has a documented destination if it ever matters — they're "not now", not "never". The list exists so scope creep gets caught before it costs a week.

- **No agent platform.** ChatMyFleet wraps agents the builder already built. Lindy, Dify, LangGraph, OpenAI Assistants, n8n own the agent reasoning layer.
- **No vertical agents.** No sales bot, support bot, or marketing bot in-scope.
- **No general LLM chatbot.** ChatMyFleet never calls an LLM on the request/reply critical path. Small LLM calls happen off-path (memory semantic search, NL rule extraction, daily brief summary) with non-LLM fallbacks.
- **No second channel at launch.** Slack only. LINE lands at v0.8.0 (~2026-12). Telegram at v0.10.0 (~2027-02). Google Chat / WhatsApp / Teams / Discord are post-v1.0 and demand-gated.
- **No second protocol-layer connector at launch.** OpenAI Assistants only. The protocol-layer connectors (which carry no affiliate commission) ship per the version roadmap — generic webhook at v0.5.0 (~2026-09), MCP at v0.6.0 (~2026-10), LangGraph remote-graph at v0.11.0 (~2027-03). MCP at v0.6 already covers LangGraph users who speak MCP; the standalone LangGraph connector at v0.11 is for those who don't.
- **No affiliate-partner integrations at launch.** Affiliate-partner integrations are a separate workstream from the protocol connectors above. The first affiliate-partner cohort is **locked in this build order**: n8n → Dify → Lindy → Flowise. Each integration is a marketplace component or plugin riding on top of the relevant connector (n8n / Dify / Flowise via the generic webhook; Lindy via webhook plus a PartnerStack marketplace node). Subsequent cohorts (Make.com, Botpress, Relevance AI; then Voiceflow, MindStudio, CustomGPT.ai, AgentCenter.cloud) sit as unlocked planning queues — order shifts on builder demand signal.
- **No multi-tenant Cloud.** Single-tenant per deployment. Multi-tenant is post-v1.0; the `tenant_id` column exists on every table so the future migration is non-breaking.
- **No managed hosting.** Self-host on Docker Compose. Managed hosting is a Year 2+ consideration, only if demand funds the operational work, and it would run the same code as the self-host version.
- **No "Pro" tier.** Apache 2.0 across the whole repo. No open-core carve-out. No source-available rug-pull risk. No paywalled features. Sponsors don't unlock anything that's not in the OSS.
- **No cryptographic audit signing at v0.4.0.** Append-only is enforced via Postgres role permission, which is a real guarantee. Ed25519 signing + hash chain lands at v0.9.0 (~2027-01) when a compliance customer needs it.
- **No KMS abstraction at v0.4.0.** BYOK secrets are AESGCM-encrypted with a static key from `.env`. KMS providers (AWS KMS, GCP KMS, Vault) are post-v1.0.
- **No hot-update / LISTEN-NOTIFY at v0.4.0.** Configuration changes require a worker restart. Hot-update is a post-launch convenience, not a launch blocker.
- **No Helm chart at v0.4.0.** `docker-compose.yml` only. Helm ships in a later post-launch minor (target ~2027-04 — the exact version moves with the affiliate-partner build order).
- **No community management overhead.** Solo BDFL governance — no RFC process, no governance committee, no contributor mentorship program, no "good first issue" labels, no Discord. GitHub Issues and Discussions are the front door. PRs are reviewed at the maintainer's discretion. The community of ChatMyFleet is the user base, not the dev team.

## Project type

**PROD (Production) — by trajectory. v0.4.0 ships the foundation; v1.0.0 (~2027-08 anniversary, or later) is the production-stable milestone.**

The aim is production: robust, scalable, sustained for five-plus years under a real maintenance contract, with the API surface eventually locked at `v1.0.0` and breaking changes only at `v2.0+`. The brief's own definition of `v1.0.0` is explicit on this point — semver-locked, API contracts frozen, anniversary milestone, not the launch milestone. Labelling this MVP would undersell the architecture, the five-year commitment, and the way self-hosters need to be able to evaluate whether to depend on it.

`v0.4.0` is not the production-stable milestone. `v0.4.0` is the public-availability moment. The product gradually ramps from "PROD-class architecture + PROD-tier load-bearing surfaces + MVP-tier peripheral surfaces" at launch, to "full PROD across all surfaces" by `v1.0.0`. The Datasette / Livewire / SQLite pattern: long `v0` era while surfaces stabilise, `v1.0` as the maturity milestone.

### Per-surface tier at v0.4.0 (and where each surface lands at PROD)

| Surface | v0.4.0 tier | Reaches full PROD at |
|---|---|---|
| Architecture (gateway/worker split, Postgres SoT, append-only audit via role permission, 4-container shape) | **PROD** — locked, non-negotiable from `v0.1.0` | `v0.1.0` |
| HITL approval flow + state machine | **PROD** (unit + integration + E2E happy + sad) | `v0.3.0` |
| Append-only audit log (enforced via Postgres role permission, exportable, chat-thread linkback) | **PROD** on enforcement + linkback; Ed25519 hash chain + signing arrive at `v0.9.0` | `v0.9.0` (signing) |
| Seven named failure modes (agent platform timeout, agent 500, channel rate-limit, memory unreachable, approval timeout, webhook delivery failure, state-machine inconsistency) | **PROD** (each tested with explicit recovery path) | `v0.4.0` |
| Slack adapter + OpenAI Assistants connector | **PROD** (Tier 1 round-trip happy + sad) | `v0.4.0` |
| Builder API (blocking + callback, idempotency keys) | **PROD** | `v0.4.0` |
| Trust-signal artifacts (`LICENSE`, `TRADEMARK.md`, `MAINTENANCE.md`, `CHANGELOG.md`, `SECURITY.md`, `CONTRIBUTING.md`, `AFFILIATES.md`, Sponsors page, `chatmyfleet.com` landing) | **PROD** — populated at `v0.1.0` when the repo flips public | `v0.1.0` |
| Web UI (5 admin pages) | **MVP** (happy path; sad paths added on adopter signal) | `v0.6.0` – `v0.8.0` |
| Memory layer (rules, entities, decisions) | **MVP** (happy path + one NL-extraction regression test) | `v0.6.0` – `v0.9.0` |
| Slash commands, daily brief, observability depth | **MVP** (polish iterates in monthly minors) | `v0.5.0` – `v1.0.0` |
| Multi-tenant Cloud, KMS abstraction, hot-update LISTEN/NOTIFY, Helm chart | Not in `v0.x` scope; separate spec | `v1.x+` (post-`v1.0.0`) |

### What "PROD-class discipline" demands from v0.4.0

- The three non-negotiable architecture calls (gateway/worker split, Postgres-as-SoT, append-only audit via role permission) don't get reopened.
- Load-bearing surfaces ship with sad-path tests and integration tests against a real Postgres + Redis from day one. The Orbit testing standard's **Tier 1 — Zero Trust** applies to these surfaces.
- Code quality and documentation are at the level expected of public OSS that one maintainer must own for five-plus years. The reader is a future self-hoster cloning the repo to decide whether to depend on it.
- Every trust-signal artifact is populated at `v0.1.0` when the repo flips public on Week 4, not at `v0.4.0`. By the time external readers find the repo, the credibility signals (license, trademark, maintenance commitment, security policy, contributing guide, affiliate disclosure) already exist.
- Breaking changes — even within the `v0.x` era — get a deprecation note in the prior minor release. The semver-lock at `v1.0.0` is forward; the discipline starts now.
- Monthly minor releases (first Friday) and monthly transparency posts (from M4, October 2026) keep the project shipping at a verifiable cadence after launch.

### What stays MVP-tier at v0.4.0

- **Web UI polish.** The five admin pages get happy-path coverage. Sad paths (network failure during edit, concurrent edits, malformed input deep in a form) get added in monthly minors as adopters report friction.
- **Memory layer breadth.** Happy path plus one regression test that an NL-extracted rule survives a connector swap. Edge cases in the NL extraction (ambiguous phrasings, multi-rule statements, contradictions with prior rules) iterate post-launch when real operator corpora arrive.
- **Slash command UX**, daily brief content quality, observability depth (twelve counters at launch — six infrastructure + six business-KPI behind the Web UI Dashboard page; OpenTelemetry traces and additional dashboards added on real-deployment signal), and admin Web UI accessibility beyond keyboard navigation.

These surfaces are MVP-tier *at v0.4.0*. By `v1.0.0` they're at PROD-tier. The ramp is the product roadmap.

### Why not the other Orbit types

- **POC** would skip the production customer and not ship to real users — invalid given a Day-1 production deployment is part of launch.
- **MVP across the board** would undersell the architecture and the five-year maintenance contract. Self-hosters reading the repo need to see PROD-shape commitments, not "minimum to learn."
- **SOW** doesn't fit — this is OSS released to an unknown audience, not done-for-you client work with a specific delivery contract.

PROD-by-trajectory captures all three constraints: real production customer Day 1, real five-year maintenance contract, real architecture commitments that don't get reopened — without claiming features (multi-tenant, signing, KMS) that don't ship at `v0.4.0`.

## Open doubts (carried into spec.md as Spikes)

These are technical claims that haven't been validated against running code. Each becomes a `docs/spikes/spike-N-*.md` before any Loop 4 build starts. Doubts 1–6 are from the original brief; doubts 7–9 were identified during the Loop 1 architecture review and de-risk the foundation epics:

1. **Slack signature verify + Block Kit round-trip on FastAPI inside the 3s ack window** — `slack-bolt` `AsyncApp` mounted via `AsyncSlackRequestHandler`.
2. **OpenAI Assistants `requires_action` POLL pattern under Redis Streams workers** — polling cadence, straggler-run handling, worker contention.
3. **pgvector + HNSW recall and latency on `memory_rules`** at the ~200-rule scale the architecture claims sub-50ms for (plus open-source embedding-model selection + runtime model-swap).
4. **NL rule extraction reliability** — operator phrasings ("lower auto-approve threshold to $300") parsed into structured policy mutations with usable accuracy and a graceful ambiguity fallback.
5. **Email approval link signing + click → audit linkback** to the originating Slack thread (cross-channel correlation).
6. **Idempotency keys on the blocking builder-API endpoint** — long-poll plus network retry behaviour without double-execution.
7. **Append-only audit enforcement via Postgres role permission** — two-role setup (privileged migration role vs restricted app role) composing with SQLAlchemy 2.0 async + Alembic; the app role must be unable to UPDATE/DELETE `audit_events`.
8. **Distroless `linux/amd64` image + `docker compose up` cold-boot** — native wheels (asyncpg, cryptography, argon2-cffi) and the conditional local embedding runtime building in a shell-less distroless image, reaching a green health check inside the five-minute install budget.
9. **Redis Streams consumer-group restart-safety** — `XAUTOCLAIM` reclaim of stale pending entries after a worker crash, with no message loss and no double-processing across the 6-stream bus.

## Constraints we're building under

- **Solo maintainer.** One developer ships every Epic in Loop 4. No parallel agents. Epic sizing has to respect that.
- **No CI/CD environment exists yet.** GitHub Actions + uv is the planned pick; no workflows defined. Epic 1 stands up CI as part of the foundation Epic so the rest of the build has a green baseline to run against.
- **No paid external accounts pre-provisioned.** Slack workspace, OpenAI API key, SMTP provider, hosting target — each gets confirmed before the relevant Epic starts. Spikes that hit live services need credentials from the maintainer.
- **Internal working scaffolding is gitignored.** The `docs/` tree is the public-facing surface once the repo flips public on Week 4. Anything that should not appear on a public reader's first browse of the repo stays out of `docs/`.
- **Twelve-week build window is fixed.** Work starts 2026-06-01. PH launch is 2026-08-19. ~14.5 engineer-weeks of work at 1.5× AI-assisted throughput ≈ 9.7 calendar weeks. ~2 weeks of buffer for slippage and demo polish. Past Week 12, the PH date is at risk.
- **Distribution stack is Apache 2.0 + GitHub + Docker.** No sales, no marketing team, no trial credit card. The repo and the five-minute install are the entire go-to-market path. Anything that breaks either of those breaks the launch.
- **Funding is patronage + affiliates, not VC.** GitHub Sponsors (four tiers, $5 to $500/mo) plus agent-platform affiliate revenue from the first affiliate-partner cohort — **n8n → Dify → Lindy → Flowise**, locked in that build order. Partner selection and ordering use a four-criteria bundle: audience match (the platform's users currently lack a chat surface), affiliate economics (commission rate × commission window), integration cost (webhook / MCP / marketplace API beat proprietary SDK), and distribution leverage (cross-promotion potential). No single criterion dominates. Subsequent cohorts (Make.com, Botpress, Relevance AI; then Voiceflow, MindStudio, CustomGPT.ai, AgentCenter.cloud) sit as unlocked planning queues — order shifts on builder demand. Monthly transparency posts (M4 onward) publish Sponsors totals and per-partner affiliate revenue; per-integration disclosure on the public `docs/affiliates.md` page when it propagates with v0.1.0.
- **Governance is solo BDFL.** Architecture, releases, merges, direction — one person decides. No RFC process, no voting, no quarterly community calls. The shape matches Datasette, Livewire, AlpineJS, SQLite. Coherent direction, slower pace than community-driven projects, sustainable from one developer's bandwidth.
- **Five-year maintenance commitment starts on launch day.** Monthly minor releases (first Friday). Security patches within 30 days. Bug acknowledgments within 5 business days. Breaking changes pre-announced by ≥1 minor version. Apache 2.0 is locked — no relicensing, no paywalling of existing capabilities, no surprise pivots to closed-source. If the project ever has to end, the failure mode is a clean public archive, not a relicense.
