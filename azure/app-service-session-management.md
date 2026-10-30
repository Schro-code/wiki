---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# App Service Session Management

## What it is

How an Azure Web App keeps user state (session state) and reuses generated responses (output caching) when it runs as several instances. It covers shared session stores (Azure Cache for Redis, SQL Server, Table Storage) and Application Request Routing (ARR) for disk-based output caching.

## What problem it solves

HTTP is stateless. Nothing is remembered between requests: each request reaches the server as if it were the first. To keep user data between requests (cart, login, preferences) you use **session state**: a store that associates data with a specific session through an identifier (session cookie).

The real problem appears when an Azure Web App **scales out** (multiple instances):

```
Without a shared session:
  User → Instance A  (stores "cart = 3 items" in its local memory)
  User → Instance B  (knows nothing about the cart — the session "disappears")
  User → Instance A  (works again)
```

An in-memory local session only works with a single instance. As soon as there is more than one, users randomly lose their state.

## Notes

### Solution: centralized shared session

For all instances to share the same state, the session store must be **external and centralized**.

#### Options available in Azure

| Solution | How it works | When to use it |
|---------|--------------|---------------|
| **Azure Cache for Redis** | Distributed in-memory store, external to the instances | Recommended — high speed, low latency |
| **SQL Server Session State** | The session is persisted in a relational database | When you already have SQL Server and prioritize durability |
| **Azure Table Storage** | Session in an Azure Storage table | Cheap option; more latency than Redis |

#### Azure Cache for Redis (recommended option)

Redis is an **in-memory key-value store** that all instances query the same way:

```
Instance A ──┐
Instance B ──┼──► Azure Cache for Redis ◄── same state for all
Instance C ──┘
```

**Why Redis for the exam:**
- Supports **multiple readers and a single writer** concurrently — exactly what the question statement asks for
- Configured in ASP.NET with the `Microsoft.Web.RedisSessionStateProvider` package
- Sessions survive restarts and scaling
- Being in-memory, latency is minimal (< 1ms)

### Output caching: storing complete HTTP responses

Besides session state, an application may need to cache the **already generated HTML** so it doesn't have to regenerate it on every request. This is called **output caching**.

The problem without output cache:

```
Without cache:
  Request → App → generates HTML (queries DB, renders template) → responds
  Request → App → generates HTML (same query, same result)      → responds  (duplicated work)

With output cache:
  Request → Cache → hit → returns stored HTML (without touching the app)
```

#### Application Request Routing (ARR)

ARR is an IIS module that acts as a **reverse proxy and load balancer** in front of the app instances. One of its capabilities is **disk-based output caching**: it stores complete HTTP responses on disk and serves them directly without reaching the instances.

```
Without ARR:
  Client → Instance A (generates HTML) → responds

With ARR:
  Client → ARR → cache hit → returns stored HTML  (the instance is not touched)
              └── cache miss → Instance A → ARR stores the response → responds
```

**What ARR does exactly:**
- Receives HTTP requests before they reach the Web Apps
- If the response is already in cache (disk-based), returns it directly
- If it is not in cache, forwards it to an instance, stores the response and returns it

**When to enable ARR:** when the requirement is to store and reuse complete HTTP responses (HTML output) without regenerating them on each request.

### Comparison summary

| Concept | What it solves | Technology |
|---------|-------------|-----------|
| **Shared session state** | Instances that don't share memory when scaling out | Azure Cache for Redis / SQL Server |
| **Output caching** | Regenerating the same HTML repeatedly | Application Request Routing (ARR) |

## Relations

- [[app-service-tiers-and-scaling]] — scaling out is what creates the shared-session problem
- [[azure-cdn]] — edge caching of static content, a different layer from ARR output caching

## Tags

#azure #az204
