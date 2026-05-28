# Spike Results — consolidated index

Per-spike detail lives in `docs/spikes/spike-N-*.md`. Throwaway experiment code lived in `spikes/` (gitignored). Status mirrored in `docs/spec.md` §5.

## Done (credential-free, run 2026-05-28)

## Spike 7 Append-only audit two-role DB — SUCCESS
App role gets INSERT (+ sequence USAGE) but not UPDATE/DELETE; Postgres rejects UPDATE/DELETE on `audit_events` with SQLSTATE 42501. SQLAlchemy 2.0 async works as the restricted role. NFR5 + architectural call #3 validated. Gotcha: IDENTITY PK needs `GRANT USAGE ON SEQUENCE`. → `docs/spikes/spike-7-audit-role-enforcement.md`

## Spike 9 Redis Streams reclaim — SUCCESS
`XAUTOCLAIM` reclaims stale PEL entries after a simulated worker crash; zero loss, PEL drained. NFR9 holds. Key finding: reclaim is **at-least-once** → worker side effects must be idempotent (state-machine guards + Spike 6 keys). → `docs/spikes/spike-9-redis-streams-reclaim.md`

## Spike 6 Builder API idempotency — SUCCESS
`INSERT … ON CONFLICT (key) DO NOTHING RETURNING` claim: 12 concurrent same-key requests → action runs exactly once, all get the same response; new key runs again. FR14 validated. Same guard doubles as the worker side-effect guard for Spike 9. → `docs/spikes/spike-6-idempotency.md`

## Spike 3 Embedding bake-off + pgvector HNSW — SUCCESS
FastEmbed lacks e5-small + embeddinggemma → pivoted (no torch) to MiniLM-L12-v2 (384d, 0.22GB) vs mpnet-base-v2 (768d, 1.0GB). Both: Thai recall@10 = 100%; p95 retrieval @200 rows = 0.46/1.88ms (~100× under target). **Recommend default MiniLM-384**; mpnet-768 documented swap. NFR2 validated. Follow-up: lock FR21 / architecture §7. → `docs/spikes/spike-3-embedding-and-pgvector.md`

## Spike 8 Distroless image + cold-boot — SUCCESS
Distroless + asyncpg + cryptography + argon2 + fastembed/onnxruntime + baked MiniLM works. Build 87s → 558MB; cold-boot to green `/health` 7s (≪ 5-min budget). Gotcha: onnxruntime needs `libgomp.so.1` copied into distroless. amd64 to re-verify in CI. → `docs/spikes/spike-8-distroless-cold-boot.md`

## Spike 1 Slack 3s ack — SUCCESS (core); live send-back deferred
slack-bolt AsyncApp on FastAPI, in-process ASGI, locally-signed requests: ack p95 **0.29ms** (~1700× under the 500ms NFR1 target); signature + replay rejection + url_verification all work. Real Block Kit send-back deferred (needs a Slack bot token); the 3s-ack architectural risk is fully resolved. → `docs/spikes/spike-1-slack-3s-ack.md`

## Spike 2 OpenAI connector — CLOSED (not a viable CMF target)
Research found OpenAI doesn't host a server-invocable agent: Assistants API removed 2026-08-26; Agent Builder `workflow_id` is ChatKit/browser-only; Agents SDK is self-hosted. **OpenAI dropped as an agent-platform connector target.** CMF targets host-agent + callable-endpoint platforms → Spikes 10–13. → `docs/spikes/spike-2-openai-poll.md`

## Deferred (need external credentials)

- **Spike 4** — NL rule extraction (needs OpenAI/Anthropic key) → Epic 8
- **Spike 5** — Email link signing + audit linkback (needs SMTP + Slack) → Epic 10

## Wave 1 platform connection spikes (added 2026-05-28; validate the generic webhook connector — Epic 5)

- **Spike 10** — n8n (Webhook node + Respond to Webhook; HITL callback) → needs n8n account + workflow
- **Spike 11** — Dify (`/v1/chat-messages`, conversation_id; HITL) → needs Dify account + app + API key
- **Spike 12** — Lindy (Webhook Trigger, Bearer secret, two-way/callback; HITL) → needs Lindy account + agent + secret
- **Spike 13** — Flowise (`/api/v1/prediction/{id}`, sessionId, SSE; HITL) → needs Flowise instance + chatflow + API key
