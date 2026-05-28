# Spike 2 — OpenAI as a CMF agent-platform target  →  **CLOSED**

**Status:** **CLOSED (2026-05-28, user decision) — OpenAI is not a viable CMF agent-platform target.**

## Why closed (the conclusion)

CMF connects to a builder's **hosted agent** by invoking a callable endpoint (by id/URL + key). OpenAI does not offer that:

1. **Assistants API** (the brief's original target) — deprecated; **removed 2026-08-26**, which is 7 days *after* the planned v0.4 launch. Building on it is a non-starter.
2. **Agent Builder** (`workflow_id`) — the only ways to run a workflow are **ChatKit** (a browser chat-widget — a competing surface to Slack, with **no headless server-side message/response API**: the server can only mint a session + read threads) or **export to Agents SDK code and self-host it**. Neither is "a hosted agent CMF invokes by id."
3. **Agents SDK / Responses API + dashboard Prompts** — these are either *self-hosted code* or *raw LLM calls*, not a hosted agent platform in CMF's sense.

**Decision:** OpenAI is dropped as a CMF agent-platform connector target. CMF targets platforms that **host the agent AND expose a callable endpoint** — the Wave 1 affiliates **n8n, Dify, Lindy, Flowise** (see Spikes 10–13). Builders who use OpenAI's Agents SDK (incl. Agent Builder exports) self-host their agent and connect later via the **generic webhook / MCP** connector like any custom agent.

This reopens the **v0.4 launch-connector decision** (Epic 5 was "OpenAI Assistants connector").

---

## Research trail (preserved)

> The investigation below is kept for the record — it's how we reached the close decision.

**Original target:** Epic 5 — OpenAI Assistants Connector + Conversation State Machine
**Environment:** openai SDK 2.38.0, live API (user's key)

## Question

Validate the connector mechanism: Assistants run → POLL → `requires_action` → `submit_tool_outputs` resume. And: is the Assistants API still the right target?

## Result

### 1. The Assistants API is DEPRECATED (decisive)

Every Assistants call emits `DeprecationWarning: The Assistants API is deprecated in favor of the Responses API`. The run also ended in `failed`. **Conclusion: the v0.4 connector must NOT be built on the Assistants API.** It should target the **OpenAI Responses API**.

### 2. Live validation blocked: `insufficient_quota`

The provided key returns `429 insufficient_quota`. The live round-trip (timing, token usage, function-call resume, double-submit idempotency) could not be completed on either API. The Responses-API harness (`spikes/spike-2-openai-poll/run_responses.py`) is ready and will validate it once a funded key is available.

## Recommended connector pattern — Responses API HITL (designed; live validation pending)

```
turn 1: responses.create(model, input, tools=[refund])
        → output contains a function_call item   ← the HITL interception point
        → CMF renders Block Kit; operator approves
turn 2: responses.create(previous_response_id=resp.id,
                         input=[{type: function_call_output, call_id, output}], tools=[...])
        → final assistant text
```

- **Simpler than Assistants:** the function_call returns **synchronously** in turn 1 — no `requires_action` poll loop for the basic case. (Spike 2's original "POLL pattern" premise is replaced.)
- Long-running agent work: Responses supports `background=True` + polling/streaming.
- HITL maps cleanly: `function_call` = pause/intercept (Block Kit); `function_call_output` = resume after approval.

## Architectural impact (needs user decision)

The spec/architecture currently say **"OpenAI Assistants connector"** (FR, architecture §6.5, Epic 5, the v0.4 launch scope, multiple roadmap references). This spike recommends changing the v0.4 connector to the **OpenAI Responses API**. The Slack/HITL/memory/audit design is unaffected — only the connector's internal mechanism changes (and it gets simpler: no poll loop).

## Blocked / next

- Provide a **funded OpenAI key** (billing/credits) → run `run_responses.py` to validate latency, tokens, and the function_call→approve→resume round-trip live.
- On user approval, update spec.md + architecture.md: connector target Assistants → Responses API.
