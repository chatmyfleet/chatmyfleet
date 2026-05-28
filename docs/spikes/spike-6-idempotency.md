# Spike 6 — Idempotency keys on the blocking Builder API endpoint

**Status:** Done — **SUCCESS**
**Date:** 2026-05-28
**Target epic:** Epic 15 — Failure Hardening + Builder API Idempotency + Launch Polish
**Validates:** FR14; the blocking `approval_request` retry-safety
**Environment:** `postgres:16`, asyncpg 0.31.0, Python 3.11, 12 concurrent connections (local Docker, no external credentials)

## Question

Does an idempotency-key scheme on the blocking (long-poll) Builder API endpoint execute the action exactly once under a concurrent network-retry storm, returning the same response to every caller — while a new key executes again?

## Result

Yes. With 12 concurrent requests sharing one `Idempotency-Key`:

| Check | Result |
|---|---|
| Side effect executed for the shared key | 1 (exactly once) — ✅ |
| All 12 callers received identical response | yes — ✅ |
| A different key executed again | yes (2 total) — ✅ |

## Scheme (validated)

A Postgres `idempotency` table keyed by the idempotency key:

```sql
INSERT INTO idempotency (key, status) VALUES ($1, 'running')
ON CONFLICT (key) DO NOTHING RETURNING key;
```

- **Winner** (got a row back): runs the side effect once, writes the result into `idempotency.response`, sets `status='done'`, returns it.
- **Losers** (conflict, no row): poll `SELECT status, response WHERE key` until `status='done'`, return the stored response.

Postgres serialises the `ON CONFLICT` so exactly one of N concurrent connections wins — the guarantee lives in the database, so it holds across gateway replicas, not just in one process. The winner's long-poll (the approval wait) is held while retries queue behind the row.

## Design decisions for Epic 15

- Key on `(agent_id, idempotency_key)` to scope per agent/tenant.
- Store the response as JSONB so retries are answered without re-running.
- Loser strategy: ~100ms polling at MVP; LISTEN/NOTIFY is a later upgrade (same defer as the timer tick).
- Add a TTL janitor to evict old `idempotency` rows (unbounded-growth guard) — not measured here, but required.

## Cross-spike link

This Postgres-row claim is the same guard that makes **`cmf-worker` side effects idempotent under Redis Streams reclaim** (Spike 9's at-least-once finding). One pattern covers both the Builder API and the worker. Combined with the conversation/approval **state-machine guards** (`UPDATE ... WHERE state='pending'`), the system is safe under re-delivery end to end.

## Decision

FR14 validated. Epic 15 ships the idempotency-key scheme on the blocking Builder API; the claim pattern is reused as the worker's side-effect guard (Spike 9).
