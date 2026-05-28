# Spike 9 — Redis Streams consumer-group reclaim / restart-safety

**Status:** Done — **SUCCESS** (with a load-bearing design finding)
**Date:** 2026-05-28
**Target epic:** Epic 3 — Redis Streams Event Bus + Worker Loop
**Validates:** NFR9 (worker restart preserves in-flight state); `docs/architecture.md` §6.3
**Environment:** `redis:7`, `redis-py` 7.4.0 async, Python 3.11 (local Docker, no external credentials)

## Question

After a worker crashes mid-processing, does the Redis Streams consumer-group pattern (`XREADGROUP` + `XACK` + `XAUTOCLAIM`) recover with no message loss and no double-processing across the event bus?

## Result

`XAUTOCLAIM` works as the architecture assumes. Simulated crash: worker A read 12 events and never ACKed them (left in the Pending Entries List). Worker B reclaimed exactly those 12 via `XAUTOCLAIM`, processed + ACKed them, and drained the never-read 8 normally.

| Check | Result |
|---|---|
| Events seen (unique) | 20/20 — ✅ **no loss** |
| PEL fully drained | 0 pending — ✅ |
| A's 12 unacked events re-delivered to B | yes (processed twice) — **at-least-once**, see finding |

## Key findings

1. **NFR9 holds.** No event is lost when a worker dies mid-processing; another worker reclaims the stale PEL entries via `XAUTOCLAIM` and completes them. Combined with Postgres-as-source-of-truth, in-flight state survives a full worker restart.
2. **Reclaim is at-least-once, NOT exactly-once.** A's events were processed once before its crash and again after B reclaimed them. This is correct Redis Streams behaviour — the consequence is that **`cmf-worker` side effects must be idempotent.**
3. **Design constraint for Epic 3 (load-bearing):** every worker side effect must be safe under re-delivery:
   - agent calls / approval resolution → guard with the **state-machine check** already in the design (`UPDATE ... WHERE state='pending'` is a no-op on a second delivery)
   - Builder API actions → the **idempotency keys** from Spike 6
   - audit writes → dedupe on `correlation_id` + event_type
   This ties Spike 9 → Spike 6 and the conversation/approval state machine.
4. **`min_idle_time` tuning:** must exceed the longest legitimate processing time (≥ the 60s worker tick, likely a few minutes) so a slow-but-alive worker's in-flight entries aren't reclaimed prematurely.

## Reusable reclaim loop (Epic 3)

```python
cursor = "0-0"
while True:
    cursor, claimed, _ = await r.xautoclaim(
        STREAM, GROUP, consumer, min_idle_time=RECLAIM_MS, start_id=cursor, count=100)
    for entry_id, fields in claimed:
        await process_idempotently(fields)   # MUST be idempotent
        await r.xack(STREAM, GROUP, entry_id)
    if cursor == "0-0":
        break
```

## Decision

NFR9 restart-safety validated. Epic 3 ships the reclaim loop **paired with idempotent side effects** (state-machine guards + Spike 6 keys). Recorded as an explicit worker design constraint.
