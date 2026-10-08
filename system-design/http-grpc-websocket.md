---
domain: system-design
tags: [system-design]
sources: []
updated: 2026-09-15
---

# HTTP, gRPC and WebSocket

## What it is

Three protocols for communication between systems. All three run over TCP — the difference is in **who can send first**, how long the connection lasts, and what format data takes.

## What problem it solves

Different communication needs require different models. Sending a form submission is not the same problem as pushing a chat message in real time. This page maps each protocol to its model and the problems it solves.

## Notes

### The core question: who can send first?

| Protocol | Who initiates each message | Connection lifetime |
|---|---|---|
| HTTP | Always the client | Closes after each response |
| gRPC | Always the client | Persistent (HTTP/2), but still request-response |
| WebSocket | Either side | Stays open until explicitly closed |

This is the most important distinction. HTTP and gRPC cannot push to a client without the client asking first. WebSocket can.

---

### HTTP — request → response → close

The client opens a TCP connection, sends a request, the server sends one response, then the connection closes.

```
Client ──GET /messages──▶ Server
Client ◀── {data} ──────── Server
[connection closes]
```

- Client always starts
- Server can only reply
- Stateless — server remembers nothing between requests
- Format: JSON (text, human-readable)
- Used for: public APIs, browser-facing endpoints, any CRUD operation

---

### gRPC — same model, different wire format

gRPC follows the same request-response model as HTTP, but uses:
- **Protocol Buffers** (binary, ~3x smaller than JSON) instead of JSON
- **HTTP/2** (multiplexed, persistent TCP connection) instead of HTTP/1.1

```
Client ──createOrder(args)──▶ Server    # looks like a local function call
Client ◀── {order_id} ──────── Server
```

The connection can persist and carry many request-response pairs, but the server still cannot push without a client request.

- Used for: internal service-to-service communication (microservices)
- Not used in browsers: browsers don't support raw gRPC
- For full gRPC mechanics (proto files, code generation): see [[grpc]]

---

### WebSocket — persistent, bidirectional

Starts as an HTTP request with an `Upgrade` header. After the server accepts, the same TCP connection stays open and both sides can send at any time.

```
Client ──HTTP Upgrade──▶ Server    (handshake, one time)
Client ◀══════════════▶ Server    (full-duplex, both sides send freely)

                    User A sends a message
Client ◀── push ────────Server    (server pushes without client asking)
```

- Either side can send at any time
- No request needed before the server can speak
- Used for: real-time chat, live notifications, multiplayer games, stock tickers

---

### Why HTTP is not enough for real-time push

The problem: User B needs to receive a message without knowing when it will arrive. With HTTP, B must ask — it cannot listen.

Three patterns exist to work around this, or solve it properly:

#### Short polling

B asks repeatedly on a timer:

```
Client ──GET /messages──▶ Server    (every 5 seconds)
Client ◀── []  ──────────Server    (empty — nothing yet)
Client ──GET /messages──▶ Server
Client ◀── []  ──────────Server    (empty again)
Client ──GET /messages──▶ Server
Client ◀── [msg] ────────Server    (message finally arrives)
```

- Simple to implement
- Wasteful: many empty responses, high latency (up to N seconds)

#### Long polling

B asks once and the server **holds the connection open** until a message arrives:

```
Client ──GET /messages──▶ Server
                          (server waits... message arrives from A)
Client ◀── [msg] ────────Server    (responds immediately)
Client ──GET /messages──▶ Server   (client reconnects right away)
```

- Near-real-time
- Server holds many open connections (one per online user)
- Still request-response — client always initiates

#### WebSocket

Server pushes the moment a message arrives, no client request needed:

```
Client ◀── push ────────Server    (instant, any time)
```

- Lowest latency
- Server can push without waiting
- Requires managing persistent connections at scale

#### Comparison

| | Short polling | Long polling | WebSocket |
|---|---|---|---|
| Latency | Up to N seconds | Near-instant | Instant |
| Empty requests | Many | None | None |
| Server state | Stateless | Holds connections | Persistent connections |
| Real-time chat | No | Acceptable | Yes |
| Scalability pattern | Easy (stateless) | Moderate | Needs sticky sessions |

**Typical chat architecture**: sender → HTTP POST → server → WebSocket push to receiver. Long polling as fallback when WebSocket is blocked (corporate firewalls).

---

### Why clients must initiate — NAT

A client cannot receive an unsolicited message from a server because of **NAT** (Network Address Translation).

```
User B's device           Router (NAT)          Internet
192.168.1.5  ────────▶   203.0.113.7   ─────▶  ...
(private IP)             (public IP)
```

- B's device has a private IP — only valid inside the home network
- The router has the public IP but only forwards traffic that B's device initiated
- Even if the server tried to connect to B, the router would drop it
- **Solution**: B must initiate the connection to the server and keep it alive

This is why WebSocket starts with an HTTP request from the client: the client opens the pipe, then both sides can use it.

---

### Sockets and connections

Every network connection is a **socket** — one endpoint of a TCP channel, identified by:

```
(client_IP : client_port)  +  (server_IP : server_port)
192.168.1.5 : 52341              203.0.113.7 : 443
```

The server listens on one fixed port (e.g., 443). Each new client gets its own socket pair — the server port never changes. One server port can hold millions of simultaneous connections.

Each open connection consumes one **file descriptor** (OS handle) and ~20 KB of kernel memory (send/receive buffers). The practical ceiling is RAM and the fd limit, not ports.

For the full fd model: see [[file-descriptors]].

---

### Quick reference: when to use each

| Situation | Protocol |
|---|---|
| Public API, CRUD, browser client | HTTP (REST) |
| Internal microservice calls, high throughput | gRPC |
| Real-time push to clients (chat, notifications) | WebSocket |
| Browser + real-time but WebSocket is blocked | Long polling |

## Relations

- [[rest-api]] — full REST/HTTP reference: statelessness, HTTP verbs, status codes, caching
- [[grpc]] — gRPC implementation detail: proto files, code generation, streaming modes
- [[file-descriptors]] — how the OS manages connections: fd table, socket buffers, fd limits
- [[ip-addresses]] — IP addressing, private vs public IPs, the NAT model

## Tags

#system-design
