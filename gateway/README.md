# gateway/

OpenAI-compatible inference gateway. This is the request-time front door.

## What it must do

| Capability | Notes |
|------------|-------|
| OpenAI-compatible API | `POST /v1/chat/completions`, `GET /v1/models` — drop-in client compatibility |
| Traffic split routing | Route `canaryPercent` of requests to the canary backend (Phase 3) |
| Auth | API keys, hashed at rest (Phase 6) |
| Rate limiting | Redis token bucket per API key (Phase 6) |
| Semantic cache | Milvus: embed prompt, serve cached answer above similarity threshold (Phase 6) |
| Tracing | OTel span per request, context propagated to the backend (Phase 5) |
| Metrics | Prometheus: request count, latency histogram, tokens, cache hit rate, backend label |

## Design constraints

- Python + FastAPI + httpx. Async everywhere; the gateway must never block on a
  model call it could stream.
- Streaming responses (`stream: true`) are a hard requirement, not a nice-to-have.
- Backends are interchangeable behind one interface: `vllm` and `llama.cpp` both
  speak the OpenAI dialect — the gateway must not care which is running.
- Config via env vars, twelve-factor style. No config files baked into the image.

## Definition of done (Phase 6)

- [ ] `pip install`-able package with tests
- [ ] streaming chat completions work end-to-end through the traffic split
- [ ] cache hit-rate and rate-limit metrics visible in Prometheus
