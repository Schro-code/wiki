---
domain: harness
tags: [harness]
sources: []
updated: 2026-08-25
---

# AI Agent Harness

## What it is

A harness is the scaffolding layer that wraps a language model and makes it capable of acting: it runs the interaction loop, manages context, executes tools, and handles I/O. The model is the brain; the harness is the body.

## What problem it solves

A raw LLM API only takes text in and returns text out — it has no memory, no tools, no ability to act on the world. A harness adds the infrastructure to turn that stateless model into an agent that can read files, run commands, manage sessions, and complete multi-step tasks.

## Notes

**The core loop a harness manages:**

1. Receive user input
2. Build context (conversation history + project instructions + relevant files)
3. Call the model
4. Parse the model's response for tool calls
5. Execute those tool calls (read file, run command, search, etc.)
6. Feed results back to the model
7. Repeat until the task is done or the model stops

**What a harness typically provides:**

| Responsibility | Examples |
|---|---|
| Context management | What the model sees: history, file contents, system prompts |
| Tool execution | File I/O, shell commands, web fetch, search |
| Session state | Conversation memory, branching, persistence |
| Permissions | What the agent is allowed to do, confirmation prompts |
| Model routing | Which model to call, fallback logic, switching |

**The harness spectrum:**

From least to most opinionated:

- **Raw API** — no harness, you build everything
- **SDK loops** — e.g. Anthropic's `tool_runner`, minimal scaffolding
- **Thin harness** — e.g. [[pi-coding-agent]] — minimal defaults, fully customizable
- **Opinionated harness** — e.g. [[claude-code]] — rich defaults, limited reconfiguration
- **Full framework** — e.g. LangGraph — harness + orchestration + state machines

**Key design tension:**

A harness that does more for you is faster to start but harder to bend. A harness that does less forces more setup but allows full control. The choice depends on how standard your workflow is.

## Relations

- [[pi-coding-agent]] — an example of a thin, customizable harness for AI coding

## Tags

#harness
