---
domain: harness
tags: [harness]
sources:
  - raw/Pi Coding Agent.md
  - raw/Pi Coding Agent 1.md
  - raw/Pi Coding Agent 2.md
  - raw/Pi Coding Agent 3.md
updated: 2026-08-26
---

# Pi Coding Agent

## What it is

Pi (`@earendil-works/pi-coding-agent`) is a coding agent harness by earendil.works. It provides the full agent loop, a rich set of built-in tools, session management, and multiple run modes. It sits in the thin-to-mid range of the harness spectrum: opinionated about the loop, but not about your domain logic.

## What problem it solves

A raw LLM API takes text and returns text. Pi adds the infrastructure to turn that into a working coding agent: it manages context, runs the tool loop, handles compaction, retries on transient errors, and provides session persistence with branching. You write the domain-specific tools and prompts; Pi handles the rest.

## Notes

### Run modes

Pi can be embedded or spawned in four ways depending on your integration needs:

| Mode | How | Best for |
|---|---|---|
| **TUI** (default) | `pi` | Interactive use in a terminal |
| **JSON** | `pi --mode json "prompt"` | Single-shot, piped output as JSONL events |
| **RPC** | `pi --mode rpc` | Subprocess integration — bidirectional JSON protocol over stdin/stdout |
| **SDK** | `createAgentSession()` in Node.js | In-process TypeScript integration, full type safety |

Choose RPC when integrating from another language or when you want process isolation. Choose the SDK when you're in the same Node.js process and want direct access to session state.

### Built-in tools

Pi ships with coding tools enabled by default: `read`, `bash`, `edit`, `write`. Additional built-ins available: `grep`, `find`, `ls`, `powershell` (Windows).

Configure per session:
```typescript
// Read-only mode
const { session } = await createAgentSession({ tools: ["read", "grep", "find", "ls"] });

// Disable one tool
const { session } = await createAgentSession({ excludeTools: ["bash"] });
```

### Extending Pi — two patterns

**1. Extension (file-based):** A TypeScript file that receives `ExtensionAPI`. Pi discovers it from `~/.pi/agent/extensions/` or `.pi/extensions/`. This is the pattern for the guardian agent.

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function myExtension(pi: ExtensionAPI): void {
  pi.on("session_start", async () => { /* startup */ });

  pi.registerTool({
    name: "my_tool",
    description: "...",
    parameters: MySchema,
    async execute(_toolCallId, args, signal) {
      return { content: [{ type: "text", text: "result" }] };
    },
  });
}
```

**2. Custom tools via SDK:** Pass tools directly when creating a session, without a file.

```typescript
import { defineTool, createAgentSession } from "@earendil-works/pi-coding-agent";

const myTool = defineTool({
  name: "my_tool",
  description: "...",
  parameters: MySchema,
  execute: async (_id, params) => ({ content: [{ type: "text", text: "done" }], details: {} }),
});

const { session } = await createAgentSession({ customTools: [myTool] });
```

### RPC protocol

RPC mode uses strict JSONL over stdin/stdout:
- **stdin** → commands (JSON objects, one per line, `\n`-delimited)
- **stdout** → events (streamed as they occur)

Commands and events are separate streams. A command response has `type: "response"` and correlates to a command by `id`. Events stream asynchronously and have their own types (`agent_start`, `message_update`, `tool_execution_*`, etc.).

Key framing detail: split on `\n` only. Node's `readline` is not compliant here — it also splits on Unicode separators (U+2028, U+2029), which breaks JSON strings.

Core commands: `prompt`, `steer`, `follow_up`, `abort`, `get_state`, `get_messages`, `set_model`, `compact`, `new_session`, `fork`, `clone`.

Events of note:
- `agent_settled` — the run is truly done (no pending retry, compaction, or queued messages)
- `message_update` — streaming deltas only; assemble from `message_start` + deltas, use `message_end` as authoritative
- `tool_execution_update` — accumulated output so far (replace, not append)

Extensions can surface interactive dialogs via the **Extension UI Protocol**: `select`, `confirm`, `input`, `editor` block until the client responds; `notify`, `setStatus`, `setWidget` are fire-and-forget.

### Session management

Sessions are trees, not flat logs. Each entry has `id` / `parentId`, enabling in-place branching:

- `fork(entryId)` — branch from a previous user message, discarding later exchanges
- `clone()` — duplicate the current branch into a new session file
- `get_entries` with `since: lastId` — efficient polling for new entries after a cursor

Sessions persist as `.jsonl` files. `SessionManager.continueRecent()` restores the most recent session for a given cwd.

### What you build yourself

Pi handles the loop, the context, the tools, the sessions. Everything domain-specific is yours:

- External API clients (e.g. Confluence, GitHub)
- State machines (e.g. approval gates, approval states)
- Side-channel servers (e.g. HTTP endpoints the model cannot access, for human-in-the-loop flows)
- Error-handling strategies for non-idempotent external writes (e.g. [[write-ambiguity-reconciliation]])
- Session isolation per user in multi-tenant deployments

### Multi-user deployments

With `--mode rpc`, a session manager can spawn one Pi child process per user and route JSONL over stdin/stdout. Each process owns its own session state. The main practical issue is port conflicts for any side-channel HTTP servers built on top — these need dynamic port assignment when running multiple instances.

The SDK alternative: `createAgentSession()` in a single Node.js process, one session per user, sharing the process but with isolated session state.

## Relations

- [[ai-agent-harness]] — where Pi sits on the harness spectrum
- [[write-ambiguity-reconciliation]] — error-handling pattern for non-idempotent writes built on top of Pi

## Tags

#harness
