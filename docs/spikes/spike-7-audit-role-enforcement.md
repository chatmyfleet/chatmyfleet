# Spike 7 — Append-only audit enforcement via Postgres role permission

**Status:** Done — **SUCCESS**
**Date:** 2026-05-28
**Target epic:** Epic 2 — Postgres Schema + Audit Append-Only Enforcement
**Validates:** non-negotiable architectural call #3 (`docs/architecture.md` §3), NFR5
**Environment:** `pgvector/pgvector:pg16`, SQLAlchemy 2.0.50 async + asyncpg, Python 3.11 (local Docker, no external credentials)

## Question

Can a two-role Postgres setup — a privileged migration role for DDL + a restricted app role with `INSERT` but no `UPDATE`/`DELETE` on `audit_events` — be enforced **at the database layer** (not in application code), while SQLAlchemy 2.0 async connects cleanly as the restricted role?

## Result

Yes. As the app role:

| Operation | Outcome |
|---|---|
| `INSERT` into `audit_events` | ✅ succeeds |
| `SELECT` from `audit_events` | ✅ succeeds |
| `UPDATE audit_events` | ✅ rejected by Postgres — SQLSTATE **42501** (`insufficient_privilege`) |
| `DELETE FROM audit_events` | ✅ rejected by Postgres — SQLSTATE **42501** |

`CREATE EXTENSION vector` succeeds as the migration role, confirming extension creation belongs to the privileged role.

## Key findings

1. **Omission = denial.** The app role gets `GRANT SELECT, INSERT` only; never granting `UPDATE`/`DELETE` is sufficient — Postgres rejects them. No triggers or rules needed.
2. **The NFR5 test asserts on SQLSTATE 42501.** SQLAlchemy raises `ProgrammingError`; check `e.orig.sqlstate == "42501"`.
3. **Sequence USAGE gotcha (must be in the Epic 2 migration):** an `IDENTITY`/serial PK uses an implicit sequence. The app role's `INSERT` fails unless it also receives `GRANT USAGE ON SEQUENCE <seq>` — resolve the sequence via `pg_get_serial_sequence('audit_events','id')`.
4. **Two connection strings:** migration role (Alembic DDL + `CREATE EXTENSION` + GRANTs) vs `cmf_app` (gateway + worker runtime). Only `audit_events` is INSERT-only for `cmf_app`; mutable tables (`conversations`, `approval_requests`, `memory_*`) get full DML grants.

## Reusable GRANT pattern (Epic 2)

```sql
-- as the privileged migration role:
CREATE EXTENSION IF NOT EXISTS vector;
-- CREATE TABLE audit_events (id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY, ...);
CREATE ROLE cmf_app LOGIN PASSWORD :pw;
GRANT CONNECT ON DATABASE :db TO cmf_app;
GRANT USAGE  ON SCHEMA public TO cmf_app;
GRANT SELECT, INSERT ON audit_events TO cmf_app;            -- intentionally no UPDATE/DELETE
GRANT USAGE ON SEQUENCE <audit_events id sequence> TO cmf_app;
```

## Decision

Non-negotiable architectural call #3 and NFR5 are validated. Epic 2 proceeds with the two-role setup. The append-only guarantee is real and DB-enforced. (Ed25519 hash-chain signing at v0.9 builds on this; no change to that plan.)
