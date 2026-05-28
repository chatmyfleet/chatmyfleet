# ChatMyFleet — Architecture (Design Lock)

> **Internal implementation reference for v0.4.0 launch scope.** This is the authoritative answer to "how does ChatMyFleet work" for the maintainer, Loop 3/4 implementation agents, and future contributors evaluating the code. It locks the design decisions made during Orbit Loop 1.
>
> This is distinct from `brief/chatmyfleet/architecture.md` (the public OSS-builder-voice conceptual model that feeds `chatmyfleet.com` and the eventual public `/docs/architecture/`) and from `docs/spec.md` (the Orbit Loop 1 template output — FRs, NFRs, spikes, epics). When this doc and the brief differ in detail, **this doc governs implementation**; the brief governs public voice.

## 1. Purpose, audience, status

**Purpose.** Capture the locked architecture so the design survives outside the chat transcript and downstream loops have one reference. Every FR/NFR/Spike/Epic cited here maps to `docs/spec.md`.

**Audience.** The maintainer; Loop 3 planning agents decomposing epics; Loop 4 build agents implementing micro-tasks; future contributors deciding whether to depend on the code.

**Scope.** v0.4.0 public launch (Wed 2026-08-19). Post-launch additions (MCP, LINE, Telegram, Ed25519 signing, multi-tenant Cloud) are noted where they affect a v0.4 design choice, but the detail lives in `brief/chatmyfleet/architecture-deep.md`.

**Status.** Locked Loop 1 design. Changes require a new lock-revision row below and a matching `docs/spec.md` Change Log entry.

| Lock revision | Date | Change |
|---|---|---|
| L1 | 2026-05-28 | Initial design lock from Loop 1. Includes Python 3.11+, expanded embedding-model Spike 3, Tier 1+2+3 telemetry (token capture + 12 metrics + Dashboard page). |

## 2. Big picture — 4 containers, 3 layers

```
┌────────────────────────────────────────────────────────────┐
│  HUMAN-FACING SURFACES                                     │
│     Slack (95%)      Web UI (admin)     Email (fallback)   │
└────────┬────────────────┬──────────────────┬───────────────┘
         │ webhook        │ HTTP             │ signed link
         ▼                ▼                  ▼
┌────────────────────────────────────────────────────────────┐
│  CHATMYFLEET — 4 containers                                │
│                                                            │
│  ┌──────────────────────┐      ┌────────────────────────┐  │
│  │  cmf-gateway         │ ───▶ │  cmf-worker            │  │
│  │  (FastAPI, fast ack) │      │  (asyncio main loop)   │  │
│  └──────────┬───────────┘      └─────────┬──────────────┘  │
│             ▼                            ▼                 │
│  ┌──────────────────────┐      ┌────────────────────────┐  │
│  │  redis 7             │ ◀──▶ │  postgres 16 +pgvector │  │
│  │  cache + 6 Streams   │      │  SOURCE OF TRUTH       │  │
│  └──────────────────────┘      └────────────────────────┘  │
└──────────────────────────────────────────┬─────────────────┘
                                           │ outbound HTTP (BYOK)
                                           ▼
┌────────────────────────────────────────────────────────────┐
│  EXTERNAL SERVICES                                          │
│   Agent platform     LLM provider      SMTP provider       │
│   (OpenAI Asst.)     (embed + NL ext.) (email approvals)   │
└────────────────────────────────────────────────────────────┘
```

| Container | Job |
|---|---|
| `cmf-gateway` | FastAPI. Validates channel webhooks + Builder API requests, enqueues to Redis Streams, acks in <500ms. Serves the Web UI SPA. Never makes slow agent calls. |
| `cmf-worker` | asyncio main loop. Reads Streams, calls agent platforms via connectors, runs the conversation + approval state machines, sends channel messages, writes audit, runs the 60s timeout-tick task. |
| `postgres` | `pgvector/pgvector:pg16`. Single source of truth. 8 tables. Append-only audit via role permission. HNSW vector index for memory. |
| `redis` | Redis 7. Hot-path cache + 6-stream event bus between gateway and worker. |

These are two entry-point modules (`cmf/gateway.py`, `cmf/worker.py`) of **one Python package** (`cmf/`) — a modular monolith deployed as separate containers (see `docs/spec.md` §4 "Service Architecture"). Shared models, config, crypto, audit. Split into more containers only when production load justifies it.

## 3. The three non-negotiable architectural calls

These do not get reopened without an explicit user decision.

**1. Gateway / worker split.** Slack requires webhook acknowledgment within 3 seconds or it times out and retries. Agent calls take 3–25 seconds (reasoning is the slow part). Therefore inline agent polling in the webhook handler is structurally impossible — the gateway must ack fast and hand the slow work to the worker via Redis Streams. *Consequence of violating:* Slack retries fire, producing duplicate processing and message storms on the first slow agent call.

**2. Postgres = source of truth, Redis = cache + queues.** One persistent store. *Consequence of violating:* two persistent stores doubles the backup story and the failure-mode debugging surface — unacceptable for a self-hosted Apache 2.0 OSS one maintainer must own for five years.

**3. Append-only audit via Postgres role permission.** The application DB role has no `UPDATE`/`DELETE` grant on `audit_events`. Enforcement is at the database layer, not application convention. *Consequence of violating:* the audit log becomes a wish instead of a guarantee; compliance customers can't verify it with `\dp audit_events`. Ed25519 hash-chain signing (v0.9) builds on this foundation, it does not replace it.

## 4. End-to-end message flow

Example: operator Sarah types `@finance-ai please refund Marc $450` in Slack.

### Phase 1 — Slack → CMF (fast path, <500ms; NFR1)

1. Slack POSTs a webhook to `cmf-gateway`.
2. `slack-bolt` `AsyncApp` (mounted via `AsyncSlackRequestHandler`) verifies the signing-secret HMAC.
3. Gateway identifies `tenant_id`, `agent_id`, Slack `thread_id`, operator `user_id`.
4. Gateway writes an `audit_events` row (inbound) with a fresh `correlation_id`.
5. Gateway publishes to the `gateway.inbound` Redis Stream.
6. Gateway returns `200 OK` to Slack — ~200–400ms total, inside the 3s budget. **It did not call OpenAI.**

### Phase 2 — Worker async pickup (seconds)

7. `cmf-worker` pulls the event from `gateway.inbound` (consumer group, ~1–50ms).
8. Worker loads/creates the `conversations` row.
9. Worker runs memory search (pgvector + HNSW over `memory_rules.embedding`) for relevant rules.
10. Worker injects top-K rules into the agent's system context (pre-context injection for non-MCP agents).
11. Worker calls OpenAI Assistants (`AsyncOpenAI`), creating a run.
12. Worker POLLs the run status (cadence tuned by Spike 2).

### Phase 3 — HITL approval (operator-online round-trip <10s p95; NFR3)

13. The agent emits `requires_action` with a `refund(amount=450)` tool call.
14. Worker writes an `approval_requests` row (`state=pending`, `timeout_at=now+4h`).
15. Worker writes `audit_events` (approval requested).
16. Worker renders a Slack Block Kit card with `[Approve] [Decline]` (each button's `value` carries the `approval_id`).
17. **Paused.** The OpenAI run is suspended; no tokens burned; all state is in Postgres + Redis.
18. Sarah taps `[Approve]`. Slack POSTs an interactivity webhook → gateway verifies, acks <500ms.
19. Worker resolves the approval: `UPDATE approval_requests SET state='decided', decision='approve' WHERE state='pending'` (atomic; double-click finds 0 rows).
20. Worker writes `audit_events` (decided).
21. Worker calls OpenAI `submit_tool_outputs` to resume the run.
22. Agent runs the refund tool, emits the final reply.
23. Worker captures the `agent.run.completed` event → writes audit row **including token usage + model + duration + tool_call_count** (FR22).
24. Worker renders the reply to Slack, sets `conversations.state=completed`.

### When Sarah is offline

The 60s timeout-tick task (see §5) advances the request: T+4h reminder → T+24h email escalation → T+48h expire. Email approval clicks hit a different gateway route (`/approve/{token}`, HMAC-signed) but execute the same `UPDATE approval_requests` logic, and the audit row links back to the originating Slack thread (FR7, FR16). No message is ever silently dropped (NFR4).

## 5. The two timing regimes

ChatMyFleet has two independent timing systems. Confusing them is the most common misread of the architecture.

```
PATH 1 — near-realtime conversation flow (event-driven)
  Slack webhook → gateway (ack <500ms) → Redis Stream → worker
  → agent call (3-25s, agent side) → Slack reply
  Total operator-visible round-trip: 5-10s typical, 15-30s complex.
  HITL round-trip when operator online: <10s p95 (NFR3).
  ► Nothing polls. Each step fires when the previous finishes.

PATH 2 — 60s timer tick (polling, background reconciliation ONLY)
  every 60s:  SELECT * FROM approval_requests
              WHERE state='pending' AND timeout_at < now()
  ► Drives the 4h/24h/48h approval escalation ladder.
  ► Has ZERO relationship to message latency.
```

The 60s granularity is irrelevant because the deadlines it checks are measured in hours (a 60s window on a 4h deadline = 0.4% jitter). Why polling instead of Postgres LISTEN/NOTIFY: the latter is in the `brief/chatmyfleet/mvp-architecture.md` defer list (post-launch); a 15-line asyncio loop is restart-safe (state in Postgres, self-heals on worker restart), trivial to debug, and zero extra operational burden at MVP scale. The daily-brief 8am job uses the same polling pattern.

## 6. Layer-by-layer architecture

### 6.1 Channels — `cmf/channels/`
Speaks each messenger's protocol. `slack.py` (Block Kit) at v0.4; `line.py` at v0.8; `telegram.py` at v0.10. All implement the `ChannelAdapter` Protocol (`base.py`). The rest of the system is channel-agnostic. *Not at v0.4:* any channel but Slack.

### 6.2 Gateway — `cmf/gateway.py`
FastAPI app. The fast door: validate → enqueue → ack <500ms. Also serves the Web UI SPA (bundled static) and the Builder API (`cmf/web/api.py`). *Not at v0.4:* any slow call inline; the gateway never talks to an agent platform.

### 6.3 Redis Streams — event bus
6 streams: `gateway.inbound`, `worker.outbound`, `approval.pending`, `approval.resolved`, `embedding.requested`, `audit.write`. Consumer groups let worker replicas claim disjoint events. Restart-safe via AOF + the fact that source-of-truth state lands in Postgres before enqueue. *Not at v0.4:* Kafka/RabbitMQ/NATS — Redis Streams covers the scale.

### 6.4 Worker — `cmf/worker.py`
asyncio main loop. Owns the conversation state machine (`idle → routing → in_run → awaiting_approval → completed`), connector calls, memory search, the 60s timer tick, and all audit writes for worker-side transitions. No in-memory state that matters — reconciles from Postgres on restart. *Not at v0.4:* separate `cmf-scheduler` / `cmf-embeddings` containers (folded in here).

### 6.5 Connectors — `cmf/connectors/`
How CMF talks to the agent platform. `openai_assistants.py` (POLL pattern, `requires_action` handling) at v0.4. Then `webhook.py` (v0.5), `mcp_client.py` (v0.6), `n8n.py`/`dify.py`/`lindy.py`/`flowise.py`/`langgraph.py` monthly thereafter. All implement the `Connector` Protocol (`base.py`). **The agent's reasoning happens entirely in the agent platform — CMF never decides what the agent says.** *Not at v0.4:* every connector except OpenAI Assistants.

### 6.6 Postgres — `cmf/db/`
8 tables: `tenants`, `users`, `agents` (with AESGCM-encrypted `connector_config` JSONB), `conversations`, `approval_requests`, `audit_events` (append-only), `memory_rules`, `memory_entities`, `memory_decisions` (the three memory tables carry `embedding VECTOR(N)` + HNSW index). `crypto.py` holds AESGCM for secrets. *Not at v0.4:* hash chain on audit (v0.9), per-tenant master keys (v1.x), separate `approval_policies` table (v0.13+).

### 6.7 Memory — `cmf/memory/`
`search.py` (pgvector queries), `embed.py` (embedding calls — local or remote, see §7), `extract.py` (NL rule extraction via off-path LLM call). Operator teaches a rule in chat → NL extraction → structured policy + embedding stored → later retrieved by semantic search and pre-injected into the agent's context. **Memory lives in CMF, not the agent platform**, so a rule survives a connector swap (FR12). *Not at v0.4:* BM25 fallback (post-launch; until then, no embedding key = memory features cleanly disabled).

### 6.8 Web UI — `cmf/web/`
React 18 + Vite + Tailwind + shadcn/ui SPA, bundled static, served by FastAPI. **5 admin pages at v0.4** (User Admin only; Staff never opens it): Plugged-in Agents, Audit Log (chat-thread linkback on every row — the differentiator), Policies (editable documents), Members & Roles, and **Dashboard** (KPI summary — see §10). *Not at v0.4:* Debug / Channels / Memory-inspector pages.

### 6.9 Commands & @mention routing — `cmf/commands/` + router

Two operator affordances on the chat surface: `@mention` (the front door) and `/slash` (the power-user backdoor). Per the UX direction (`brief/chatmyfleet/ux.md`), **buttons and `@mention` lead; slash commands are a backdoor**.

**@mention routing.** The router resolves an `@mention` to an `agent_id` and the conversation thread:

| Pattern | Example | Behaviour |
|---|---|---|
| Operator → agent | `@finance-ai check refund 402` | Route the message to that agent's connector |
| Agent → agent (CC) | `@finance-ai` CCs `@sales-ai` for context | Multi-agent coordination in one thread (no agent platform knows about the others — CMF is the only layer where the team metaphor lives) |
| Unaddressed message | `what's the refund total?` | Default routing: to the thread's last-addressed agent, or a prompt asking which agent |

Agent names are arbitrary and operator-set (`@finance-ai`, `@sales-ai`, `@reports-ai`, …) via `/agent add` or `/rename`. The underlying platform is hidden in chat (shown only in `/fleet` and the Web UI).

**Slash command surface — 14 commands at v0.4.** Ten are discoverable (shown in `/help`); four are setup/config (used during install, not surfaced in `/help`).

*Discoverable day-to-day (10):*

| Command | Args | Does | Who |
|---|---|---|---|
| `/help` | `[command]` | List commands or describe one | Everyone |
| `/fleet` | — | List org's agents + live status | Everyone |
| `/about` | `@agent` | Who built it, version, platform | Everyone |
| `/brief` | `[today\|yesterday]` | Daily brief on demand | Everyone |
| `/audit` | `[@agent\|today\|7d]` | Last 10 actions inline + `[Open full audit →]` | Everyone |
| `/policy` | `[@agent]` | Show current rules in plain language | Everyone (Admin to edit) |
| `/rename` | `@agent newname` | Rename agent (confirms in dashboard) | Admin |
| `/start` | `@agent` | Activate a stopped agent | Admin |
| `/stop` | `@agent [reason]` | Stop an agent | Admin |
| `/dashboard` | — | Deep-link to the Web admin | Everyone |

*Setup / config (4, not in `/help`):*

| Command | Args | Does | Who |
|---|---|---|---|
| `/agent add` | `<type> <args>` | Register an agent (e.g. `/agent add openai-assistants <assistant_id> <api_key>`) | Admin |
| `/agent list` | — | List configured agents + IDs | Admin |
| `/agent edit` | `<agent_id> <setting>` | Edit agent config (e.g. `/agent edit <id> memory pre-context`) | Admin |
| `/rule add` | `<rule text>` | Deterministic memory-rule fallback when NL extraction confidence is low (FR10) | Admin/operator |

**Conventions** (from `ux.md`): all lowercase single word; verb-noun for actions with args; single noun for reads; output ephemeral by default; every follow-up response ships with action buttons; non-admins see admin commands grayed out with an "ask admin" tooltip (discoverability preserved for promotion day).

**Deliberately NOT supported:** per-agent action subcommand trees like `/agent finance approve` — approvals happen via Block Kit buttons, not slash commands (`ux.md` anti-patterns). The `/agent` namespace is for config (add/list/edit) only, never for per-agent verbs.

## 7. Where the embedding model sits

The embedding model powers memory semantic search. It has **two possible homes**, chosen by `EMBEDDING_MODEL` env var at boot. **Locked by Spike 3:**

```
🟢 LOCAL (v0.4 DEFAULT)                  🟠 REMOTE (operator override)
embedding-worker task inside            HTTPS call from cmf-worker to
cmf-worker, ONNX via FastEmbed          OpenAI / Voyage / Cohere via BYOK key
(no torch):                               openai:text-embedding-3-small (1536-d)
  DEFAULT: paraphrase-multilingual-       + 0 container weight
    MiniLM-L12-v2 (Apache 2.0, 384-d,     − requires paid key
    0.22GB) — 100% Thai recall@10         − breaks air-gapped deploy
  SWAP: paraphrase-multilingual-
    mpnet-base-v2 (Apache 2.0, 768-d,
    1.0GB) — 100% EN/TH/cross recall
+ zero paid keys (OSS promise)
+ works offline / air-gapped
+ Apache 2.0 (safe to bake into image)
```

The application calls one function `embed(text) -> vector`; the dispatch (local vs remote) is config. Embeddings are **off the critical path** — write-side is queued via `embedding.requested`; read-side is one quick call per `memory.search`. **Spike 3 result (`docs/spikes/spike-3-embedding-and-pgvector.md`):** the originally-named candidates (`e5-small`, `embeddinggemma`) are not FastEmbed-supported, so the bake-off pivoted to the two FastEmbed Apache-2.0 multilingual models above. Both hit 100% Thai recall@10 and p95 retrieval ~0.5–1.9ms at 200 rows (~100× under the NFR2 50ms target). **Default locked: MiniLM-L12-v2, `VECTOR(384)`**, fixed-per-deploy (model change = re-embed migration); FastEmbed version pinned; model baked into the distroless image (Spike 8) so cold-boot needs no network. Multilingual (Thai-included) matters because user-zero is a Thai SMB and the v0.8 LINE adapter targets Asian SMBs.

## 8. MCP roles over time

**v0.4 launch: no MCP.** Agents reach CMF via the REST Builder API (`POST /v1/agents/{id}/...`, FR13); CMF reaches OpenAI via the Assistants POLL pattern. MCP is deferred to v0.6 (~Oct 2026) because the locked connector strategy is "largest deployed agent runtime first" (OpenAI Assistants), and one flawless connector beats three half-working ones.

**v0.6+: CMF plays BOTH MCP roles simultaneously**, on different connections:

```
ROLE A — CMF as MCP SERVER (exposes tools to external agents)
  Builder's MCP agent ──MCP──▶ https://your-cmf.com/mcp
                               tools: memory.search, memory.write_rule,
                                      memory.recall_decision, approval.request,
                                      audit.query
                               + Elicitation (HITL approval requests)

ROLE B — CMF as MCP CLIENT (consumes MCP-compliant agent platforms)
  cmf/connectors/mcp_client.py ──MCP──▶ LangGraph / Mastra /
                                        OpenAI Agents SDK / any of
                                        200+ MCP servers
```

### Elicitation HITL flow (Shape 1, the standardized pattern)

The agent (MCP client) calls CMF's `approval.request` tool with `wait: blocking` + an `idempotency_key`. CMF's `/mcp` handler:
1. Checks idempotency (FR14); returns cached response if seen.
2. Writes `approval_requests` row (`state=pending`, `timeout_at=T+4h`).
3. Writes `approval.requested` audit row.
4. Publishes to `approval.pending` so the worker renders Slack Block Kit async (don't block the MCP call on Slack latency).
5. Long-polls Postgres until `state != 'pending'` or `timeout_at` passes (restart-safe — not an in-memory await).
6. Returns `{decision, decided_by, decided_at, decided_via}` over the open MCP response.

Operator's button click resolves the same `approval_id` as in §4. The agent receives the decision and resumes.

**Migration story:** v0.4 REST → v0.6 MCP is zero-friction for builders because both transports share the same auth subsystem (§9). The MCP server is a new file (`cmf/mcp_server.py`) that reuses `cmf/memory/search.py` and the existing approval logic — no backend rewrite.

## 9. Auth model

Shared subsystem for the v0.4 REST Builder API and the v0.6 MCP server.

**Identifier hierarchy:** `tenant_id` → `agent_id` → `api_key`. Every table carries `tenant_id` (single-tenant per deployment at v0.4; the column makes the v1.x+ multi-tenant migration non-breaking).

**Key issuance** (`/agent add`, User Admin only): generate `agent_id` + token (`cmf_live_...`); hash the token with **Argon2id** (`argon2-cffi`, FR18); store only the hash in `agent_api_keys`; show plaintext once. Audit `key.issued`.

**Per-request flow:** extract `Authorization: Bearer`; narrow by a non-secret token prefix index; verify Argon2id (~100ms, intentional — makes offline brute-force on a leaked hash dump expensive); reject if `revoked_at` or `expires_at` passed; inject `tenant_id`/`agent_id`/`scopes`/`key_id` + `correlation_id` into request scope (NFR10).

**Authorization:** each MCP tool / REST endpoint declares required scopes (`memory.read`, `memory.write`, `approval.request`, `audit.read`). **Every tenant-scoped query carries a mandatory `tenant_id` filter.** Memory rules also carry `agent_scope` (`global` vs `agent_specific`) so a key issued to `finance-ai` only sees global rules + its own.

**Key rotation:** `/agent rotate-key` issues a new key, keeps the old valid for a 7-day overlap (`expires_at = now+7d`), audits `key.rotated`. Immediate revocation: `/agent revoke-key` sets `revoked_at = now()`.

**Web UI auth:** Argon2id-hashed passwords + session tokens (HTTP-only secure cookie, also hashed at rest), same primitives.

**Defense:** per-IP rate limit (50 attempts / 15 min) on `auth.rejected.unknown_key` via a Redis sliding window. All auth outcomes audited (`auth.rejected.*`, `authz.denied`, `key.issued/rotated/revoked`); the Audit Log has a Security filter. Threat model documented in `SECURITY.md` at v0.1.0.

## 10. Telemetry — locked design (Tier 1 + 2 + 3)

Two audiences, two surfaces, one data source (`audit_events` + `approval_requests`).

**Tier 1 — token capture in audit rows.** Every `agent.run.completed` audit row carries `tokens.{prompt,completion,total}`, `model`, `duration_seconds`, `tool_call_count` when the agent platform exposes them; missing fields stored as `null`, never estimated (FR22). Captured per-connector starting at Epic 5 MT-02 (OpenAI Assistants).

**Tier 2 — 12 Prometheus metrics** (NFR11), exposed at `cmf-gateway:8080/metrics` for the *operator of the CMF deployment* (Grafana / PromQL):

| # | Metric | Type | Key labels |
|---|---|---|---|
| 1 | `cmf_gateway_request_seconds` | histogram | route, method, status |
| 2 | `cmf_worker_job_seconds` | histogram | stream, event_type |
| 3 | `cmf_agent_call_seconds` | histogram | connector, agent_id, outcome |
| 4 | `cmf_approval_pending_count` | gauge | agent_id |
| 5 | `cmf_audit_events_total` | counter | event_type, agent_id |
| 6 | `cmf_failure_mode_total` | counter | mode, agent_id |
| 7 | `cmf_agent_run_total` | counter | agent_id, connector, outcome |
| 8 | `cmf_agent_tokens_total` | counter | agent_id, model, kind |
| 9 | `cmf_approval_decided_total` | counter | agent_id, decision, decided_via |
| 10 | `cmf_approval_decision_seconds` | histogram | agent_id, decided_via |
| 11 | `cmf_messages_total` | counter | agent_id, direction |
| 12 | `cmf_memory_search_total` | counter | agent_id, hit |

Items 1–6 are infrastructure health; 7–12 are business KPIs. OpenTelemetry traces remain deferred.

**Tier 3 — Web UI Dashboard page** (5th admin page, FR23), for the *User Admin inside the tenant*. Sourced from `audit_events` + `approval_requests` aggregations (not Prometheus). For a configurable time range (7d default / 30d / custom): inbound message volume + sparkline; approval count + decision breakdown (approve/decline/expire); median operator decision time; per-agent activity table (messages, approvals, tokens/run); failure-mode counts with linkback to filtered Audit Log; active-conversation gauge. Built on Epic 11's React/Vite/Tailwind/shadcn scaffold; one new dep (`recharts`). Shipped by Epic 14 (`docs/spec.md` §7).

## 11. The seven named failure modes

Each has an explicit recovery path tested at v0.4 (FR19). Hard guarantee (NFR4): **no operator or agent message is ever silently dropped** — every failure produces a visible operator error or a visible audit event.

| Failure | Recovery |
|---|---|
| Agent platform times out (>2 min) | "Still working" message; offer stop; operator decides |
| Agent platform returns 500 | Retry once with backoff; graceful error if still failing |
| Channel SDK rate-limited | Queue + back off; replay when rate clears |
| Memory layer unreachable | Operate without memory; log degraded state |
| Approval timeout (24h+) | Escalate to backup approver per policy (60s tick) |
| Webhook delivery fails | Exponential backoff; surface to operator after 3 attempts |
| State-machine inconsistency | Postgres conversation state is authoritative; reconcile from it |

## 12. What is NOT in the v0.4 design

Each item has a documented destination; "not now," not "never." Full table in `brief/chatmyfleet/mvp-architecture.md` "Explicit defer list."

- **MCP** (server + client) → v0.6
- **Generic webhook connector** → v0.5
- **Ed25519 audit hash chain + signing** → v0.9 (append-only via role permission is the v0.4 foundation)
- **Second channel** → LINE v0.8, Telegram v0.10
- **Multi-tenant Cloud** → v1.x+ (`tenant_id` columns exist now)
- **KMS abstraction** → post-launch (AESGCM with static `.env` key at v0.4)
- **LISTEN/NOTIFY hot-update** → post-launch (worker restart for config changes at v0.4)
- **Helm chart** → ~v0.12 (Docker Compose only at v0.4)
- **Separate `cmf-scheduler` / `cmf-embeddings` containers** → when throughput justifies (folded into `cmf-worker` at v0.4)
- **BM25 memory fallback** → post-launch
- **OpenTelemetry traces** → post-launch (structlog + `correlation_id` covers v0.4 investigation)

## 13. References

- `docs/user_brief.md` — project intent, success criteria, defer list, constraints
- `docs/research_business.md` — persona, competitor analysis, OSS funding model
- `docs/research_technical.md` — stack picks, build-vs-buy, risk register, implementation strategy
- `docs/spec.md` — FRs, NFRs, 9 spikes, 15 epics with micro-tasks (the implementation contract)
- `brief/chatmyfleet/architecture.md` — public conceptual model + 5 design principles *(reference, do not edit)*
- `brief/chatmyfleet/mvp-architecture.md` — v0.4 build spec, defer list, repo layout *(reference, do not edit)*
- `brief/chatmyfleet/architecture-deep.md` — eventual-state engineering design *(reference, do not edit)*
- `brief/chatmyfleet/ux.md` — UX direction, 5 design principles, scripted operator journeys *(reference, do not edit)*
