# Spike 1 — Slack signature verify + ack inside the 3-second window

**Status:** Done (core) — **SUCCESS**; live Block Kit send-back deferred (needs Slack token)
**Date:** 2026-05-28
**Target epic:** Epic 4 — Slack Adapter + First @agent Round-Trip
**Validates:** the gateway/worker split premise (NFR1 <500ms ack); architectural call #1
**Environment:** `slack-bolt` 1.28.0 `AsyncApp` + `AsyncSlackRequestHandler` on FastAPI, in-process ASGI (httpx), Python 3.11 — **no credentials** (requests signed locally with a chosen signing secret)

## Question

Does `slack-bolt` `AsyncApp` mounted on FastAPI verify Slack's request signature and acknowledge well within the 3-second webhook timeout, while the handler does only fast work (enqueue)?

## Result

| Check | Result |
|---|---|
| `url_verification` challenge handshake | ✅ echoes challenge |
| Valid signed `event_callback` → 200 + enqueued | ✅ |
| Invalid signature → rejected | ✅ 401 |
| Stale timestamp (>5 min) → rejected (replay) | ✅ 401 |
| Ack latency p95 over 200 events | **0.29 ms** (p50 0.24 / p99 0.32 / max 0.46) |
| p95 < 500ms (NFR1) | ✅ by ~1700× |

## Key findings

1. **Ack is sub-millisecond** — signature verify + dispatch + enqueue is ~0.3ms p95, ~1700× under the 500ms NFR1 target and ~10,000× under Slack's 3s hard timeout. The gateway will never be the ack bottleneck; the gateway/worker split (architectural call #1) is structurally sound.
2. **Signature verification + replay protection are built in.** `request_verification_enabled=True` rejects bad HMACs; slack-bolt automatically rejects timestamps older than 5 minutes. No extra code.
3. **Construction requires a token or an `authorize` callable.** For credential-free testing, a custom `authorize` returning an `AuthorizeResult` bypasses Slack's `auth.test`. A *dummy token* instead triggers a real `auth.test` network call per event — avoid. The real gateway supplies the bot token.

## Deferred (needs Slack bot token)

- Real `chat.postMessage` Block Kit send-back to a live channel.
- Events API delivery over a public tunnel.

These complete the round-trip; the **3s-ack core risk is fully resolved**. Confirm the live send-back in Loop 4 Epic 4 once a Slack app is provisioned.

## Decision

The architectural premise holds: verify + ack ≪ 3s, hand slow work to the worker via the queue. Epic 4 proceeds. Use `authorize`/real token per environment; keep `request_verification_enabled=True`.
