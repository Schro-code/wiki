---
domain: inference-engineering
type: index
updated: 2026-10-08
---

# Inference Engineering — Domain Index

The discipline of loading, serving, and operating language models so that agents and applications can reliably use them.
Covers model formats, quantization, local inference servers, gateway layers, observability, and hardware constraints.

This domain is being built progressively while setting up a local LLM stack (Ollama → LiteLLM → pi coding agent).

## Pages

| Page | When to look | Sources |
|---|---|---|
| [[roadmap]] | Look first — learning path across all phases, key concepts per phase, decisions from the reference repo | — |
| [[gguf]] | Look when you want to understand how model weights are stored on disk and what a GGUF file contains | — |
| [[quantization]] | Look when choosing between Q4 / Q8 / f16 or understanding the memory–quality tradeoff | — |
| [[ollama]] | Look when setting up local model serving or understanding how Ollama works under the hood (llama.cpp + Metal) | — |
| [[model-selection-for-agents]] | Look when deciding which model to use for an agentic coding task — the four criteria that matter | — |
| [[hardware-for-inference]] | Look when evaluating whether your hardware can run a model (unified memory, VRAM, memory bandwidth, decode speed) | — |
| [[litellm]] | Look when adding a gateway / proxy layer in front of your inference server | — |
| [[inference-monitoring]] | Look when you want to observe model behaviour: TTFT, throughput, context usage, error rates | — |
| [[thinking-mode]] | Look when deciding whether to enable or suppress chain-of-thought generation, and what the token cost is | — |
| [[tool-calling-verification]] | Look when verifying that a model correctly implements the tool-call protocol before connecting an agent | — |
| [[openai-api-standard]] | Look when you want to understand the OpenAI API format, finish_reason values, and the tool calling protocol | — |
