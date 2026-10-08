---
domain: system-design
tags: [system-design]
sources: []
updated: 2026-09-10
---

# IPC Transport Mechanisms

## What it is

IPC (Inter-Process Communication) is the set of mechanisms the OS provides for isolated processes to exchange data. There are only a few real mechanisms at the OS level — everything else is a protocol built on top of them.

## What problem it solves

The OS enforces process isolation: each process has its own memory space and cannot touch another process's data directly. IPC mechanisms are the bridges the kernel provides to cross that boundary.

---

## Foundation: how the OS hands you a channel (file descriptors)

Before comparing mechanisms, it helps to understand how the OS gives you access to any of them.

When a process wants to open a socket, pipe, or file, it asks the OS. The OS creates the real object in kernel space and hands back a number — a **file descriptor (fd)** — that the process uses to refer to it from then on.

```
Process:  "OS, give me a socket"
OS:        creates socket object internally
OS:       "Here's your reference: fd=3"
Process:   saves fd=3, uses it to read/write
```

**For socket-based IPC specifically, there are two kinds of fd:**

- **The listening fd** (the front door) — one per server, stays open forever, receives all incoming connection requests
- **The connection fd** (the private room) — one created per client via `accept()`, dedicated to that client alone

```
server listening on fd=3  ←── client A knocks → accept() → fd=4  (A's private channel)
                          ←── client B knocks → accept() → fd=5  (B's private channel)
                          ←── client C knocks → accept() → fd=6  (C's private channel)
```

This is why the fd limit equals the connection limit: the front door uses one fd; each connected client consumes one more. The OS default is 1024 fds per process (`ulimit -n`), but it can be raised to millions.

→ Full detail: [[file-descriptors]]

---

## Layer 0 — OS mechanisms (what the kernel actually gives you)

The most important property of each mechanism is **locality**: can it reach across two different machines, or does it only work when both processes are on the same machine?

| Mechanism | What the kernel provides | Can it cross machines? |
|---|---|---|
| **Pipe** | A buffer in kernel memory. One end writes, other end reads | No — both processes must be on the same machine |
| **Signal** | OS sends a number to a PID (`SIGINT`, `SIGKILL`) | No — PIDs are local to a machine |
| **Shared memory** | Two processes map the same physical RAM page | No — RAM is physically on one machine |
| **File** | Bytes on disk. Process A writes, B reads | No — same filesystem (unless NFS, a special case) |
| **Unix domain socket** | Socket addressed by a file path on disk | No — file path is local |
| **TCP/IP socket** | Socket addressed by IP + port | Yes — IP works across machines |

**The critical rule:** only TCP/IP socket can cross machines. Everything else is physically bound to one machine.

---

## Layer 1 — Protocols (conventions built on top)

A protocol answers: "given that we have a channel open, what do we write on it?"

Protocols **inherit the locality of whatever they run on**. They cannot transcend the physical limits of the mechanism beneath them.

| Protocol | Typically runs on | Locality it inherits |
|---|---|---|
| **HTTP / HTTPS** | TCP/IP socket | Network-capable |
| **WebSocket** | TCP/IP socket | Network-capable |
| **gRPC** | HTTP/2 → TCP/IP socket | Network-capable |
| **JSON-RPC** | Anything — pipe, HTTP, socket | Depends on what you put it on |
| **Message queue** (Kafka, RabbitMQ) | TCP/IP socket | Network-capable |
| **Unix domain socket** | Local socket | Local only — faster than TCP (no network stack) |
| **RPC over stdin/stdout** | Pipe | Local only |

gRPC is remote because TCP is remote. RPC-over-pipe is local because pipes are local. There is no exception to this.

---

## Mental model

```
┌─────────────────────────────────────────────────────────┐
│                   What you build on                      │
│   HTTP · gRPC · WebSocket · JSON-RPC · message queues   │
└────────────────────┬────────────────────────────────────┘
                     │ built on
┌────────────────────▼────────────────────────────────────┐
│              OS mechanism (the real channel)             │
│                                                          │
│  LOCAL ONLY              │  NETWORK-CAPABLE              │
│  ──────────────────────  │  ───────────────────          │
│  Pipe                    │  TCP/IP socket                │
│  Signal                  │    └─ HTTP, gRPC, WS on top  │
│  Shared memory           │                               │
│  File                    │                               │
│  Unix domain socket      │                               │
└─────────────────────────────────────────────────────────┘
```

---

## System design implications of crossing the network

When you choose a TCP-based mechanism, you unlock horizontal scaling but you pay with:

| Cost | What it means |
|---|---|
| **Latency** | Network roundtrip vs. memory copy — orders of magnitude slower |
| **Partial failure** | The other process can be unreachable mid-operation |
| **Ordering** | Packets can arrive out of order; you cannot assume sequence |

These three costs are the root of the hard problems in distributed systems: CAP theorem emerges from partial failure, Sagas exist because ACID transactions cannot span a network boundary, and eventual consistency is a response to latency + ordering.

**The monolith → microservices transition is fundamentally an IPC change.** Services that communicated via function calls (same process, same memory) now communicate via TCP. The architecture works the same way logically, but the failure model is completely different.

---

## Choosing a mechanism

| Situation | Best choice |
|---|---|
| Same machine, max throughput | Shared memory (zero-copy) or Unix domain socket |
| Same machine, subprocess integration | Pipe (RPC over stdin/stdout) |
| Cross-service, browser-facing | REST over HTTP |
| Cross-service, internal, typed contracts | gRPC |
| Async, decoupled sender/receiver | Message queue (Kafka, RabbitMQ) |

---

## RPC over stdin/stdout (the Pi pattern)

RPC (Remote Procedure Call) is a **pattern**, not a transport. It means: calling a function that lives in another process as if it were local. You can do RPC over HTTP, over a socket, or over a pipe.

[[pi-coding-agent]] uses pipes. When you run `pi --mode rpc`:

1. Pi starts as a subprocess
2. Your program writes JSON commands to Pi's **stdin** (one line = one command)
3. Pi does the work and writes JSON events to its **stdout** (one line = one event)
4. The connection stays open the whole time — it's a stream, not request/response

```
Your program                Pi process
─────────────────────────────────────────
write JSON  ──────────────→  stdin
            ←──────────────  stdout  (streaming events as work happens)
```

You receive each chunk of text the moment Pi produces it. With HTTP you would wait for the full response before getting anything.

---

## Relations

- [[file-descriptors]] — every IPC mechanism is represented as an fd in the process; the fd limit is why connection counts are bounded
- [[ai-agent-harness]] — harnesses like Pi use these transports to expose their agent loop programmatically
- [[pi-coding-agent]] — uses the pipe transport (stdin/stdout) for its RPC mode
- [[grpc]] — gRPC is a protocol built on TCP; understanding IPC layers explains why it can only run across a network
- [[distributed-transactions]] — crossing the network (TCP IPC) is what makes distributed transactions hard
- [[cap-and-pacelc]] — CAP theorem is a consequence of partial failure, which only exists with network IPC
- [[sagas]] — Sagas are the practical answer to distributed atomicity once you've crossed the IPC boundary

## Tags

#system-design
