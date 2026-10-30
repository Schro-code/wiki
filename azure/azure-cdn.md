---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# Azure CDN

## What it is

Azure Content Delivery Network is a globally distributed network of cache servers that reduces latency by delivering static content from the point closest to the user.

## What problem it solves

Without a CDN, every request travels from the user to the origin server (Azure Web App, Storage Account, etc.), regardless of distance:

```
User in Tokyo → Server in West Europe → response takes ~200ms
```

With a CDN, content is cached at **PoPs (Points of Presence)** spread across the world. The first request goes to the origin, but later ones are served by the nearest PoP:

```
User in Tokyo → PoP in Japan → response takes ~10ms
```

**Typical use cases:** video on demand, images, JS/CSS files, software downloads. Any content that does not change with every request.

## Notes

### How caching works

1. User requests `cdn.myapp.com/video.mp4`
2. The CDN checks whether it has it cached at the regional PoP
3. **Cache HIT** → returns it directly
4. **Cache MISS** → requests it from the origin server, caches it and returns it

Content stays in cache until the TTL (Time To Live) expires or it is invalidated manually.

### Caching behavior rules

When you configure a **caching rule** in Azure CDN, you choose how the CDN should honor (or ignore) the `Cache-Control` headers sent by the origin server.

| Behavior | What the CDN does | When to use it |
|---|---|---|
| **Bypass cache** | Caches nothing. Every request goes to the origin | Dynamic content that changes on every call (APIs, real-time data) |
| **Default** | Honors the origin's `Cache-Control` headers. If the origin sends no headers, uses the CDN's default TTL (can be up to 7 days) | When the origin server already manages its own cache headers correctly |
| **Override** | Ignores the origin's headers and always applies the TTL you define in the rule | When you need to guarantee a specific TTL without depending on what the origin does |
| **Set if missing** | Applies your TTL only if the origin does **not** send `Cache-Control`. If the origin does send them, it honors them | As a fallback: "if the origin says nothing, use this duration" |

#### Decision diagram

```
Does the origin send Cache-Control headers?
├── Yes and they are correct → Default (honors them as-is)
├── Yes but I want to force my own TTL → Override (ignores them)
└── It sends no headers
    ├── I want a fallback → Set if missing
    └── I want it to never cache → Bypass cache
```

### AZ-204 exam trick

When the question mentions a specific TTL that must be met ("the content must expire in 1 hour"):

- If the origin **already sends** `Cache-Control: max-age=3600` → **Default** works
- If you want to **guarantee** it regardless of the origin → **Override**
- **Bypass** discards the caching requirement
- **Set if missing** only applies if the origin sends nothing

> The answer is almost always **Override** when the statement demands a specific time without mentioning that the origin already manages it.

## Relations

- [[azure-blob-storage]] — the origin is usually a Storage Account or a Web App
- [[azure-container-services]] — App Service (a Web App) is a common CDN origin

## Tags

#azure #az204
