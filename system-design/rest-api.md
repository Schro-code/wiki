---
domain: system-design
tags: [system-design]
sources: []
updated: 2026-09-03
---

# REST API

## What it is

A REST API is an interface that follows the REST architectural style (Representational State Transfer) to allow communication between systems over HTTP. Introduced by Roy Fielding in 2000. The fundamental unit is a **resource** — anything that can be named and addressed (a user, an order, a product).

## What problem it solves

Provides a standard, predictable way for clients and servers to communicate: uniform URLs identify resources, HTTP verbs express the intended action, and HTTP status codes convey what happened. Decouples client from server so they can evolve independently.

## Notes

### What "Representational State Transfer" means

Each word carries meaning:

- **State** — the current condition of a resource (a user has a name, email, role)
- **Representation** — a snapshot of that state in a format (JSON, XML, HTML). The server doesn't hand you the resource itself — it hands you a *representation* of it at this moment
- **Transfer** — that representation is moved between client and server over HTTP

`GET /users/123` doesn't give you "the user". It gives you a JSON document that represents what the user looks like right now.

### The 6 constraints

For an API to be RESTful it must follow these constraints:

| Constraint | What it means |
|---|---|
| **Uniform Interface** | Resources are identified by URLs; interactions are standardized across the system |
| **Client-Server** | Client and server are separated — client handles UI, server handles data |
| **Stateless** | The server stores no client session state between requests — each request must carry everything needed to process it |
| **Cacheable** | Responses must declare whether they can be cached and for how long |
| **Layered System** | The client doesn't know if it's talking directly to the server or an intermediary (CDN, proxy, load balancer) |
| **Code on Demand** | Server can return executable code (optional) |

### Stateless — the precise meaning

Stateless means the **server** stores no **client session state** between requests. It does not mean the system has no state at all.

The distinction that matters:

| What | Where | REST says |
|---|---|---|
| Client session state ("user X is logged in, their cart is...") | Server process/memory | ❌ Forbidden |
| Application state (conversation history, orders, users) | Database as a resource | ✅ Fine — that's just a resource |
| Cached responses (result of GET /products for 60s) | Redis / CDN / client | ✅ Fine — per-request, not per-client |

**Practical consequence — horizontal scaling:** if state lives in a specific server process, requests from the same user must always go to that process (sticky sessions). If state lives in a database, any instance can serve any request. Statelessness is what makes instances interchangeable and enables scaling freely.

**Chatbot example:** a chatbot API is stateless when conversation history is stored as a resource in a database keyed by `conversation_id`. Each request includes the ID; the server fetches the history, runs the model, writes the new message back — then forgets everything. Any of 100 instances can serve the next request.

In LangGraph terms: `MemorySaver` (state in RAM) is not stateless — breaks under multiple instances. `PostgresSaver` / `RedisSaver` (state in DB) is stateless — any instance can continue any conversation.

### Stateless enables caching

These constraints reinforce each other. Because the server is stateless, `GET /products/42` means the same thing for everyone — the response is deterministic per URL. That makes it safe to cache at the client, CDN, or reverse proxy layer.

If the server had session state, the same URL might return different data per user. Caching would be unsafe. Statelessness is what makes caching reliable.

**Server-side caching (Redis, in-process) is not a violation.** Caching a response to a URL is caching *computation*, not *client identity*. The cache key is the request, not the client. If the cache is wiped, behavior is unchanged — you just recompute.

### Authentication without session state — JWT

Server-side sessions (storing "user X is authenticated") would violate statelessness. The common solution: **JWT tokens**. The server signs a token containing the user's identity and claims, and gives it to the client. The client sends it with every request. The server verifies the signature and reads the claims — no storage needed. The client carries their own state.

### HTTP verbs

| Verb | Meaning | Idempotent? | Side effects? |
|---|---|---|---|
| `GET` | Fetch a resource or collection | ✅ | No |
| `HEAD` | Same as GET but no response body | ✅ | No |
| `POST` | Create a new resource / trigger an action | ❌ | Yes |
| `PUT` | Replace a resource entirely | ✅ | Yes |
| `PATCH` | Partially update a resource | ❌ (usually) | Yes |
| `DELETE` | Remove a resource | ✅ | Yes |

**Idempotent** means calling it multiple times produces the same result as calling it once. `PUT /users/123` with the same body always leaves the user in the same state. `POST /orders` creates a new order each time.

### HTTP status codes

| Class | Range | Meaning |
|---|---|---|
| Informational | 1xx | Request received, continuing |
| Success | 2xx | Request succeeded (200 OK, 201 Created, 204 No Content) |
| Redirection | 3xx | Further action needed (301 Moved Permanently, 304 Not Modified) |
| Client error | 4xx | Bad request from the client (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found) |
| Server error | 5xx | Server failed to fulfill a valid request (500 Internal Server Error, 503 Service Unavailable) |

### Typical resource URL pattern

| URL | Verb | Action |
|---|---|---|
| `/users` | GET | List all users |
| `/users/{id}` | GET | Get one user |
| `/users` | POST | Create a user |
| `/users/{id}` | PUT / PATCH | Update a user |
| `/users/{id}` | DELETE | Delete a user |

### Trade-offs

**Advantages:** simple and universally understood; stateless design enables horizontal scaling; good caching support; client and server are decoupled.

**Disadvantages:** over-fetching (the response includes more data than needed); under-fetching (multiple round trips needed to get related data — see GraphQL as an alternative); no built-in streaming or push.

## Relations

- [[cap-and-pacelc]] — statelessness and caching connect to the AP position: prefer availability and performance over strict consistency
- [[sagas]] — Sagas coordinate distributed operations across services that expose REST APIs; each step and compensation is typically an HTTP call
- [[acid-and-base]] — the database behind a REST API chooses ACID or BASE; the API layer is agnostic but the consistency model shapes what the API can guarantee

## Tags

#system-design
