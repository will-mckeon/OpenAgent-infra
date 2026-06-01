# openagent-infra

> **OpenAgent model inference infrastructure** — the resilient, dual-model serving layer of the OpenAgent OS ecosystem.

---

## Overview

`openagent-infra` is the inference proxy repository for **OpenAgent**. This repo is solely responsible for proxying requests to the underlying LLMs, authenticating callers, managing dual-model routing for unit economics, and streaming responses via a production-ready REST API.

This repo is intentionally scoped to the model layer only. It has no knowledge of the frontend interfaces or business logic. Those concerns live in separate repositories. The boundary is clean by design: **openagent-infra serves the models, everything else builds on top of it.**

## The Dual-Model Strategy

OpenAgent's inference architecture relies on two models running in parallel on RunPod serverless to balance deep reasoning with low-latency control tasks:

- **primary-120b** — built on **gpt-oss-120b**, the primary reasoning model handling complex analysis and deep conversations.
- **router-20b** — built on **gpt-oss-20b**, the fast, lightweight control layer handling routing, history filtering, and agentic decisions.

---

## Where This Fits

```text
OpenAgent Ecosystem
│
├── openagent-infra      ← YOU ARE HERE
│   └── LLM inference API (port 8002)
│       Open core — model proxy layer
│
├── openagent-frontend   ← separate repo
│   └── The user product experience (port 8000)
│       Owns the system prompt, talks to openagent-infra
│
├── openagent-training   ← future
│   └── Fine-tuning and training pipelines
│
├── openagent-data       ← future
│   └── Data collection and curation for training
│
└── openagent-auth       ← future
    └── API key management and user signup

```

**Port topology:**

```text
User → openagent-frontend (:8000) → api-service (:8002) → RunPod primary-120b         [default]
                                                        → RunPod router-20b           [model="router"]

```

`api-service` is the only service in `openagent-infra`. Both models run on separate RunPod serverless workers.

---

## Architecture

```text
┌─────────────────────────────────────────────────┐
│              Docker Container                   │
│                                                 │
│  ┌───────────────────────────────────────────┐  │
│  │         api-service  (port 8002)          │  │
│  │         FastAPI proxy — src/api/main.py   │  │
│  │                                           │  │
│  │  POST /chat  →  validates X-API-Key       │  │
│  │              →  injects reasoning_effort  │  │
│  │              →  routes by model field     │  │
│  │              →  streams SSE to caller     │  │
│  │  GET  /health → checks proxy + both       │  │
│  │                 RunPod workers            │  │
│  │  Auth: X-API-Key header required on /chat │  │
│  └──────────┬──────────────┬─────────────────┘  │
└─────────────┼──────────────┼────────────────────┘
              │ model="base" │ model="router"
              │ (default)    │
              ▼              ▼
┌─────────────────────┐  ┌─────────────────────────┐
│ RunPod vLLM Worker  │  │ RunPod vLLM Worker      │
│ primary-120b        │  │ router-20b              │
│ gpt-oss-120b/MXFP4  │  │ gpt-oss-20b/MXFP4       │
│ Primary reasoning   │  │ Routing, history,       │
│ All /chat by default│  │ agent control layer     │
└─────────────────────┘  └─────────────────────────┘

```

### Request flow

1. `openagent-frontend` sends `POST /chat` with `X-API-Key`, messages list, optional `reasoning_effort`, and optional `model`.
2. `api-service` validates the API key — returns `401` if missing or invalid.
3. `api-service` injects `Reasoning: <level>` into the system message automatically.
4. `api-service` routes to the correct RunPod worker — primary (120B) by default, router (20B) when `model="router"`.
5. `api-service` forwards via httpx with `Authorization: Bearer RUNPOD_API_KEY`.
6. RunPod generates tokens on dedicated GPU hardware.
7. Tokens stream back through `api-service` to the caller as SSE events.
8. A final `data: [DONE]` event signals end of stream.

### System prompt ownership

The system prompt is owned by **openagent-frontend**, not `openagent-infra`. The frontend sends it as the first message in the OpenAI messages list on every request. `api-service` injects the reasoning effort level into it automatically before forwarding to RunPod. The infra layer never stores or inspects the system prompt content.

---

## Tech Stack

| Layer | Technology |
| --- | --- |
| Base image | `python:3.12-slim` |
| Model serving | RunPod vLLM serverless — two independent workers |
| primary-120b | gpt-oss-120b by OpenAI (`YourOrg/primary-120b-worker`) |
| router-20b | gpt-oss-20b by OpenAI (`YourOrg/router-20b-worker`) |
| Model precision | MXFP4 (MoE layers) — both models |
| API proxy | FastAPI + uvicorn |
| Streaming | SSE via httpx async proxy from RunPod |
| Auth | `X-API-Key` header (caller) + `RUNPOD_API_KEY` Bearer (RunPod) |

---

## Setup

### 1. Clone the repo

```bash
git clone <repo-url>
cd openagent-infra

```

### 2. Deploy RunPod workers

Deploy two separate RunPod vLLM serverless endpoints — one per model. Set the following environment variables on each worker:

**primary-120b worker:**

| Variable | Value |
| --- | --- |
| `MODEL_NAME` | `YourOrg/primary-120b-worker` |
| `HF_TOKEN` | Your HuggingFace token |
| `MAX_MODEL_LEN` | `16384` |
| `REASONING_PARSER` | `openai_gptoss` |
| `ENFORCE_EAGER` | `true` |

**router-20b worker:**

| Variable | Value |
| --- | --- |
| `MODEL_NAME` | `YourOrg/router-20b-worker` |
| `HF_TOKEN` | Your HuggingFace token |
| `MAX_MODEL_LEN` | `16384` |
| `REASONING_PARSER` | `openai_gptoss` |
| `ENFORCE_EAGER` | `true` |

Copy the endpoint ID from each worker at `https://www.runpod.io/console/serverless`.

### 3. Create your `.env` file

```bash
cp .env.example .env

```

Edit `.env` and fill in your values:

```env
API_KEY=your_long_random_secret_key_here
PRIMARY_BASE_URL=[https://api.runpod.ai/v2/your_primary_endpoint_id_here/openai](https://api.runpod.ai/v2/your_primary_endpoint_id_here/openai)
ROUTER_BASE_URL=[https://api.runpod.ai/v2/your_router_endpoint_id_here/openai](https://api.runpod.ai/v2/your_router_endpoint_id_here/openai)
RUNPOD_API_KEY=your_runpod_api_key_here
REASONING_EFFORT=medium

```

### 4. Build and Start

```bash
docker-compose up --build -d api-service

```

---

## Design Decisions

### Why dual-model routing?

Using a 120B model for every API call destroys unit economics. By routing simple control tasks, history filtering, and agent decisions to a 20B model, we reduce VRAM requirements, lower latency, and drastically cut inference costs while preserving deep reasoning for the user-facing responses.

### Why MXFP4?

Both models utilize native MXFP4 quantization of the MoE layers. It reduces VRAM requirements significantly without quality degradation — the 120B fits on a single 80 GB GPU and the 20B fits within 16 GB.

### Why RunPod serverless?

RunPod serverless provides dedicated GPU hardware for both models, eliminates local GPU and CUDA requirements, scales each worker to zero when idle, and allows developers to focus on orchestration rather than bare-metal infrastructure.

### Why two separate API keys?

`API_KEY` authenticates the caller to the proxy. `RUNPOD_API_KEY` authenticates the proxy to the infrastructure backend. These concerns are deliberately separated to compartmentalize financial exposure. If the caller key is compromised, it can be rotated or revoked without altering the infrastructure configuration.

---

## License

Apache License 2.0
Copyright 2026 William McKeon

```

***

### `DATASHEET.md`

```markdown
# openagent-infra — Datasheet

> Reference document for building on top of openagent-infra.
> Intended audience: **openagent-frontend** and any other service that consumes the OpenAgent inference API.

---

## Quick Reference

| Item | Value |
|---|---|
| Base URL | `http://localhost:8002` |
| Protocol | HTTP/1.1 |
| Streaming | Server-Sent Events (SSE) |
| Auth | `X-API-Key` header (required on `/chat`) |
| Content type in | `application/json` |
| Content type out | `text/event-stream` |
| Request format | OpenAI messages format |
| Reasoning effort | `low` / `medium` / `high` (optional field, default: `medium`) |
| Model selection | `base` (default) / `router` (optional field) |
| Chat endpoint | `POST /chat` |
| Health endpoint | `GET /health` |

---

## Authentication

Every `POST /chat` request must include a valid API key in the `X-API-Key` header. Requests with a missing or invalid key receive `401 Unauthorized`. 

```http
X-API-Key: your_api_key_here

```

---

## API Reference

### `POST /chat`

Sends a full OpenAI messages list to the proxy and receives a token-by-token streamed response via SSE. Optionally controls the reasoning effort level and model routing per request.

#### Request

```json
{
  "messages": [
    {"role": "system",    "content": "You are a helpful OpenAgent..."},
    {"role": "user",      "content": "Analyze the tradeoffs between SSE and WebSockets"}
  ],
  "reasoning_effort": "high",
  "model": "base"
}

```

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `messages` | array | Yes | Full OpenAI messages list including system prompt |
| `reasoning_effort` | string | No | `low`, `medium`, or `high`. Defaults to server `REASONING_EFFORT` env var. |
| `model` | string | No | `base` (default) or `router`. Routes to primary (120B) or router (20B). |

**Important:** `openagent-frontend` is responsible for constructing the full messages list including the system prompt as the first `system` message. The proxy injects `Reasoning: <level>` into the system message automatically.

#### Reasoning effort guidance

| Level | Latency | Use for |
| --- | --- | --- |
| `low` | Fastest | Lightweight tooling calls, simple lookups, routing decisions |
| `medium` | Balanced | Standard interactions, general questions (default) |
| `high` | Slowest | Complex analysis, multi-step reasoning, hard problems |

#### Response

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream; charset=utf-8
Cache-Control: no-cache
Transfer-Encoding: chunked

```

Tokens are streamed as individual SSE events and always end with a `[DONE]` sentinel:

```text
data: Hello
data: !
data: [DONE]

```

#### Error responses

| Status | Condition | Body |
| --- | --- | --- |
| `400` | Messages list is empty or contains no user message | `{"detail": "Messages list cannot be empty"}` |
| `401` | X-API-Key header missing or invalid | `{"detail": "Invalid or missing API key"}` |
| `422` | Request body malformed or missing | FastAPI validation error JSON |
| `503` | RunPod endpoint not reachable | proxy error message |

---

### `GET /health`

Lightweight health check. No authentication required. Checks both the proxy and the RunPod backend.

**Fully ready:**

```json
{"status": "ok", "proxy": "ok", "primary": "ok", "router": "ok"}

```

**Primary unreachable (Cold Start):**

```json
{"status": "degraded", "proxy": "ok", "primary": "unreachable", "router": "ok"}

```

---

## Technical Specifications

### Startup timing

| Phase | Approximate duration |
| --- | --- |
| api-service startup | < 10 seconds |
| RunPod worker cold start | 8–10 minutes (model loading) |
| RunPod worker warm | < 2 seconds |

Both workers scale to zero independently. Design frontends with loading states for both scenarios.

### Generation timing

| Scenario | Reasoning | Approximate duration |
| --- | --- | --- |
| Simple greeting | low | 5–15 seconds |
| Short factual question | medium | 15–45 seconds |
| Complex reasoning task | high | 1–3 minutes |

RunPod continuous batching means concurrent requests do not queue — they are processed together.

---

## Environment Variables Reference

| Variable | Type | Default | Description |
| --- | --- | --- | --- |
| `API_KEY` | string | — | Secret key validated against X-API-Key header. Required. |
| `PRIMARY_BASE_URL` | string | — | RunPod endpoint for primary (120B). Required. |
| `ROUTER_BASE_URL` | string | — | RunPod endpoint for router (20B). Required. |
| `RUNPOD_API_KEY` | string | — | RunPod API key for Bearer auth on both endpoints. Required. |
| `REASONING_EFFORT` | string | `medium` | Server default reasoning level. |

```

```
