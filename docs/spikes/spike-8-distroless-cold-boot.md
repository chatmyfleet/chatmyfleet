# Spike 8 — Distroless image + native deps + `docker compose up` cold-boot

**Status:** Done — **SUCCESS**
**Date:** 2026-05-28
**Target epic:** Epic 1 — Project Skeleton + CI Foundation
**Validates:** the five-minute-install launch criterion (FR1); the distroless runtime decision
**Environment:** `gcr.io/distroless/python3-debian12`, multi-stage build, arm64 (host native — see caveat)

## Question

Can a distroless `linux/amd64` image (no shell, no package manager) bundle the v0.4 native deps — `asyncpg`, `cryptography`, `argon2-cffi`, **plus** the Spike-3 embedding stack (`fastembed` + `onnxruntime` + baked MiniLM-384 model) — and cold-boot to a green `/health` inside the five-minute budget?

## Result

Yes. Build **87s → 558MB**; `docker compose up` cold-boot to green `/health` in **7s**. Inside distroless, `/health` returns:

```json
{"status":"ok","checks":{"postgres":true,"aesgcm":true,"argon2":true,"fastembed_dim":384}}
```

— proving asyncpg (real Postgres), cryptography (AESGCM), argon2-cffi, and fastembed/onnxruntime all work in the distroless runtime.

## Key findings / gotchas (Epic 1)

1. **`libgomp.so.1` must be copied into distroless.** `onnxruntime` (via fastembed) links against it and distroless doesn't ship it. `COPY --from=builder /usr/lib/*-linux-gnu/libgomp.so.1 /usr/lib/` — without it, `import onnxruntime` fails. The `*-linux-gnu` glob keeps it arch-portable. **This is the load-bearing gotcha.**
2. **Install to a target dir, not a venv.** `pip install --target=/deps` + `ENV PYTHONPATH=/deps` copies cleanly into distroless; a venv's `bin/python` symlink points at the absent builder interpreter.
3. **Bake the model into the image** (`cache_dir=/models` in builder, copied to runtime) — cold-boot needs no HuggingFace network call, removing a dependency + rate-limit risk from the install path. +~0.2GB.
4. **Shell-less healthcheck** uses the bundled `python3` + `urllib` (no curl/sh in distroless).
5. **Entrypoint** = `["python3","-m","uvicorn","app:app",...]` (distroless entrypoint is python3).
6. **Compose** `depends_on: postgres: condition: service_healthy` gives clean cold-boot ordering.

## Reusable Dockerfile pattern

Multi-stage: `python:3.11-slim` builder (`pip install --target=/deps`, bake model) → `gcr.io/distroless/python3-debian12` runtime (COPY /deps + /models + libgomp + app; `PYTHONPATH=/deps`; uvicorn module entrypoint). Captured in `spikes/spike-8-distroless-cold-boot/Dockerfile`.

## Caveat

Built/run on **arm64** (host native) for speed. v0.4 ships **linux/amd64**; structural findings are arch-independent but amd64 wheels + the libgomp path must be re-verified in Epic 1 CI (GitHub Actions amd64 runners).

## Decision

Five-minute install is comfortably achievable (one-time build 87s, cold-boot 7s). Distroless + full native stack + baked model works. Epic 1 adopts the multi-stage pattern; CI re-verifies on amd64.
