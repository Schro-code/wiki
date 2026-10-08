---
domain: inference-engineering
tags: [#inference-engineering]
sources: []
updated: 2026-10-08
---

# Inference Engineering — Learning Roadmap

Practical path to understand how to load, serve, monitor, and connect a local LLM to an AI agent.
Based on studying `gersonrpq/coding-agent-inference-infra` (H100 + SGLang + LiteLLM + pi coding agent)
and replicating the patterns locally on Apple M5 Pro 48 GB.

---

## The stack we are building

```
pi coding agent
      ↓
LiteLLM :4000       ← proxy + monitoring gateway
      ↓
Ollama :11434        ← local inference server (llama.cpp + Metal)
      ↓
Qwen3-14B Q4_K_M    ← the model
```

The repo uses this same layered pattern, scaled to an H100 with two SGLang workers behind LiteLLM.
The concepts are identical. The hardware changes, the architecture does not.

---

## Phase 1 — Serve a model locally ✅

**Goal:** load a model, verify it works for agentic use, measure baseline performance.

| Concept | Key idea |
|---|---|
| GGUF | Binary format that stores model weights + quantization in a single file |
| Quantization | Compressing weights from 2 bytes (f16) to 1 byte (Q8) or 0.5 bytes (Q4) to save memory |
| Ollama | Model manager + inference server wrapping llama.cpp. Exposes an HTTP API on :11434 |
| Model selection for agents | Four criteria: tool calling support, context window ≥ 32K, quantization quality, decode speed |
| Apple Silicon unified memory | CPU and GPU share the same 48 GB pool — no VRAM limit, can run Q8 models that a 24 GB GPU cannot |
| Tool calling verification | The go/no-go gate: send a tool schema and verify the model returns structured JSON, not prose |
| Thinking mode | Qwen3 generates long chain-of-thought by default. Use `/no_think` to suppress it for faster responses |
| Baseline measurement | Decode speed: 29.6 tok/s (Qwen3-14B Q4_K_M on M5 Pro). Minimum for agent use: ~8 tok/s |

---

## Phase 2 — Monitor the model 🔄

**Goal:** see what happens inside the stack during an agent session — latency, token counts, errors.

| Concept | Key idea |
|---|---|
| LiteLLM | OpenAI-compatible proxy. Sits between the agent and Ollama. Logs every request with latency + tokens |
| Gateway pattern | The agent only knows LiteLLM's URL. The backend (Ollama, OpenAI, Azure…) can swap without touching the agent |
| Request logging | Every call through LiteLLM shows: duration ms, input tokens, output tokens, model, status code |
| Virtual keys | LiteLLM can issue API keys per client, each with its own rate limits and budget. The repo uses this per tenant |
| Prometheus + Grafana | LiteLLM exposes `/metrics`. Scrape with Prometheus, visualize in Grafana. Key metrics: TTFT, tok/s, errors |
| TTFT | Time To First Token — how long until the model starts responding. The repo's SLO is p99 ≤ 20 s |
| Key metrics to watch | TTFT, decode speed, context length per request, error rate (especially tool-call parse failures) |

---

## Phase 3 — Connect a coding agent

**Goal:** connect the pi coding agent to the local stack and observe a real agentic session end to end.

| Concept | Key idea |
|---|---|
| pi-mono | Open-source coding agent by badlogic. Reads files, runs commands, writes code through tool calls |
| Tool calling in practice | The agent sends a message with tools defined. The model returns a tool_call JSON. The harness executes it and feeds the result back |
| Context growth | Each tool call adds tokens. After 5–10 tool calls, the context can be 10–20K tokens. Watch this in LiteLLM |
| Prefix caching | If the system prompt is the same across turns, the inference engine can cache its KV and skip recomputing it |

---

## Phase 4 — Understand the infrastructure (from the repo)

**Goal:** understand the engineering decisions behind the H100 setup and which ones apply locally.

| Concept | Key idea |
|---|---|
| KV cache | GPU memory storing attention tensors from previous tokens. More KV memory = more concurrent sessions |
| KV bytes per token | `2 × attention_layers × KV_heads × d_head × bytes_per_dtype`. For Qwen3.5-9B bf16: 32 KiB/token |
| max_num_seqs | Hard limit on concurrent sequences. The repo runs 6 per worker (12 total) |
| Admission control | The gateway decides to accept or refuse a request based on current KV pressure and queue depth |
| Session affinity (affload) | Route the same conversation to the same worker to reuse its cached KV. +16–35% throughput vs round-robin |
| Priority queuing | Interactive requests (agent turns) get priority over batch jobs |
| Capacity curve | Throughput peaks at ~16–20 concurrent sessions. Beyond that, refusals grow faster than throughput |

---

## Phase 5 — Advanced concepts (for deep understanding)

These require either an NVIDIA GPU or can be studied theoretically.

| Concept | Local? |
|---|---|
| SGLang (alternative to Ollama/vLLM) | Partial — can run locally without MIG |
| Hierarchical KV cache (L1 GPU / L2 RAM / L3 Mooncake) | L1+L2 yes, L3 needs RDMA |
| MIG (Multi-Instance GPU) | No — NVIDIA only |
| Load testing with synthetic + real agent sessions | Yes — adapt `loadgen.py` and `agentgen.py` from the repo |
| Cost modelling ($/token from GPU-hours) | Yes — pure math |
| Pre-registered experiments | Yes — scientific method for A/B routing comparisons |

---

## Key decisions from the repo worth studying

| Decision | Lesson |
|---|---|
| K=2 queue allowance | A longer queue adds latency, not throughput. Minimum queue that reduces refusals |
| affload over least-busy routing | Session affinity beats load balancing when the workload has long shared prefixes |
| No stream timeout | A timeout that fires during legitimate long work (e.g. writing a 300-line file) is worse than no timeout |
| batch_share reservation | Without reserving slots for interactive, batch fills the queue and interactive refusal jumps to 16.5% |
| Callbacks over a separate service | Control plane in-process avoids network hops; the price is single-replica constraint |

---

## Relations

- [[index]] — domain index with all pages in this area
- [[ollama]] — the local inference server
- [[litellm]] — the gateway layer
- [[quantization]] — memory vs quality tradeoff
- [[model-selection-for-agents]] — the four criteria applied in Phase 1

## Tags

#inference-engineering
