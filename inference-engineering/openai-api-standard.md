---
domain: inference-engineering
tags: [inference-engineering]
sources: []
updated: 2026-10-08
---

# OpenAI API Standard

## What it is

A JSON wire protocol for sending messages to a language model and receiving responses. The core endpoint is `POST /v1/chat/completions`. OpenAI defined it, but it has become the de facto standard for the entire industry: Ollama, LiteLLM, vLLM, LM Studio, and dozens of other runtimes all implement it, so the same client code◊ works against any of them.

## What problem it solves

Before this standard, every LLM provider had its own API shape. Switching from one model to another required rewriting the client. The OpenAI format solves this by giving the ecosystem a common contract:

- **For application developers** — write once against `/v1/chat/completions`, point `base_url` at any backend.
- **For inference server authors** — implement this spec and every existing tool (LangChain, LlamaIndex, Cursor, etc.) works immediately.
- **For agents** — the `finish_reason` field gives agents a reliable signal for what to do next (respond to user, call a tool, abort).

## Notes

### 1. Request structure

```json
POST /v1/chat/completions
{
  "model": "gpt-4o",
  "messages": [
    {"role": "system",    "content": "You are a helpful assistant."},
    {"role": "user",      "content": "What is the weather in Madrid?"}
  ],
  "tools": [...],
  "tool_choice": "auto",
  "temperature": 0.7,
  "max_tokens": 1024
}
```

**Key request fields:**

| Field | Type | Notes |
|---|---|---|
| `model` | string | Which model to run |
| `messages` | array | Conversation history. Roles: `system`, `user`, `assistant`, `tool` |
| `tools` | array | Optional. List of functions the model can call |
| `tool_choice` | string/object | `"auto"` (default), `"none"`, `"required"`, or force a specific function |
| `temperature` | float | 0–2, controls randomness |
| `max_tokens` | int | Deprecated in favour of `max_completion_tokens` (Sept 2024), still accepted |
| `stream` | bool | Enable SSE streaming |

**Tool definition** inside the `tools` array:

```json
{
  "type": "function",
  "function": {
    "name": "get_weather",
    "description": "Returns current weather for a location.",
    "parameters": {
      "type": "object",
      "properties": {
        "location": {"type": "string", "description": "City, Country"}
      },
      "required": ["location"]
    }
  }
}
```

---

### 2. Response structure

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1699896916,
  "model": "gpt-4o",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": null,
        "tool_calls": [
          {
            "id": "call_xyz789",
            "type": "function",
            "function": {
              "name": "get_weather",
              "arguments": "{\"location\": \"Madrid, Spain\"}"
            }
          }
        ]
      },
      "finish_reason": "tool_calls"
    }
  ],
  "usage": {
    "prompt_tokens": 82,
    "completion_tokens": 17,
    "total_tokens": 99
  }
}
```

The response always wraps the result in `choices[0].message`. When the model calls a tool, `content` is `null` and `tool_calls` is populated.

---

### 3. `finish_reason` — the critical field

`finish_reason` tells you **why the model stopped generating**. For agents, this is the primary dispatch signal.

| Value | Meaning | Agent action |
|---|---|---|
| `"stop"` | Natural end — the model finished its response | Send text to user, loop is done |
| `"tool_calls"` | Model wants to call one or more tools | Execute tools, send results back, loop again |
| `"length"` | Hit `max_tokens` limit before finishing | Handle truncation; may need to increase limit |
| `"content_filter"` | Response blocked by content policy | Surface error to user |
| `"function_call"` | **Deprecated.** Old name for `"tool_calls"` | Treat same as `"tool_calls"` |

**The bug this maps to:** Ollama's native API (non-OpenAI-compatible endpoint) sometimes returns `"stop"` even when the model generated tool calls. An agent checking only `finish_reason == "tool_calls"` will silently skip the tool invocation. The fix: always check **both** `finish_reason` and the presence of `tool_calls` in the message.

```python
# Defensive check — don't rely on finish_reason alone
message = response.choices[0].message
if message.tool_calls or response.choices[0].finish_reason == "tool_calls":
    # execute tools
```

---

### 4. The tool-calling agentic loop — step by step

The full round-trip when an agent uses a tool:

**Step 1 — Initial request**
```json
{
  "model": "gpt-4o",
  "messages": [
    {"role": "user", "content": "What is the weather in Madrid?"}
  ],
  "tools": [{ "type": "function", "function": { "name": "get_weather", ... } }],
  "tool_choice": "auto"
}
```

**Step 2 — Model responds with a tool call** (`finish_reason: "tool_calls"`)
```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": null,
      "tool_calls": [{
        "id": "call_xyz789",
        "type": "function",
        "function": {
          "name": "get_weather",
          "arguments": "{\"location\": \"Madrid, Spain\"}"
        }
      }]
    },
    "finish_reason": "tool_calls"
  }]
}
```

**Step 3 — Client executes the tool locally**
```python
args = json.loads(tool_call.function.arguments)
result = get_weather(args["location"])
# result = {"temp": 22, "unit": "C", "condition": "sunny"}
```

**Step 4 — Client sends results back**

Append the assistant's tool-call message AND a `tool` role message for each result:
```json
{
  "model": "gpt-4o",
  "messages": [
    {"role": "user", "content": "What is the weather in Madrid?"},
    {
      "role": "assistant",
      "content": null,
      "tool_calls": [{ "id": "call_xyz789", ... }]
    },
    {
      "role": "tool",
      "tool_call_id": "call_xyz789",
      "content": "{\"temp\": 22, \"unit\": \"C\", \"condition\": \"sunny\"}"
    }
  ],
  "tools": [...]
}
```

**Critical:** `tool_call_id` must exactly match the `id` from the model's tool call. The assistant message with `tool_calls` must also be in the history — do not strip it.

**Step 5 — Model returns the final answer** (`finish_reason: "stop"`)
```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "The weather in Madrid is currently 22°C and sunny."
    },
    "finish_reason": "stop"
  }]
}
```

The loop ends. Text goes to the user.

---

### 5. Parallel tool calls

The model can request multiple tools in a single turn — `tool_calls` is an array. Execute all of them, then send all results back in a single request before the next inference call.

```json
"tool_calls": [
  { "id": "call_001", "function": { "name": "get_weather", "arguments": "..." } },
  { "id": "call_002", "function": { "name": "get_forecast", "arguments": "..." } }
]
```

---

### 6. Who implements this standard

| Runtime | Role | How to point at it |
|---|---|---|
| **OpenAI** | Origin | `https://api.openai.com/v1` |
| **Ollama** | Local inference (llama.cpp) | `http://localhost:11434/v1` |
| **LiteLLM** | Proxy/gateway for 100+ providers | `http://localhost:4000/v1` |
| **vLLM** | High-throughput GPU server | `http://localhost:8000/v1` |
| **LM Studio** | Desktop GUI with local models | `http://localhost:1234/v1` |
| **llama.cpp server** | Raw C++ inference | `http://localhost:8080/v1` |

To switch backends, change only `base_url` (and `api_key` if required). Application code is unchanged.

---

### 7. Compatibility caveats

"OpenAI-compatible" does not mean "identical." Known gaps:

- **Ollama** — tool calling works on the `/v1/` endpoint but fidelity varies by model. The native `/api/generate` endpoint returns `"stop"` even for tool calls, which breaks agents expecting `"tool_calls"`.
- **Arguments are a JSON string, not an object** — `tool_calls[].function.arguments` is always a raw JSON string. Always parse with `json.loads()` before use. The model can hallucinate keys not in your schema — validate before calling.
- **`content` is `null` on tool calls** — this is expected. Do not treat `null` content as an error.
- **Streaming tool calls** — arguments arrive in deltas. Accumulate them before parsing.

## Relations

- [[ollama]] — local inference server that implements this standard at `localhost:11434/v1`
- [[litellm]] — proxy that exposes a single OpenAI-compatible endpoint in front of many backends
- [[tool-calling-verification]] — how to verify a model actually honours the protocol before connecting an agent
- [[inference-monitoring]] — observing token usage (`usage` field) and latency from the same endpoint

## Tags

#inference-engineering
