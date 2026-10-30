---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# Azure API Management

## What it is

Azure API Management (APIM) is a single control layer placed in front of all your APIs. A Gateway receives every request and applies policies (authentication, rate limiting, transformation, caching) before forwarding it to the backend. A Developer Portal lets consumers discover, try and subscribe to the APIs.

> APIM does not replace your backend — it protects it and controls access to it.

## What problem it solves

You have several APIs (backends, microservices, Azure Functions...) exposed directly to the outside world. That means:

- Each API handles its own authentication
- There is no way to limit how many calls a client makes
- There is no centralized documentation
- If you change a backend, the client has to update its URL

```
Without APIM:
  Client → Backend A
  Client → Backend B
  Client → Backend C   (each one with its own auth logic, limits, etc.)

With APIM:
  Client → APIM Gateway → Backend A
                        → Backend B
                        → Backend C
```

## Notes

### Main components

| Component | What it does |
|-----------|--------------|
| **Gateway** | Receives all requests and applies the policies before passing them to the backend |
| **Developer Portal** | Auto-generated portal where consumers explore, test and subscribe to the APIs |
| **Management Plane** | Where you configure everything: APIs, products, policies, users |
| **Policies** | Rules applied at the gateway (auth, transformations, caching, rate limiting) |

---

### Most-used policies

| Policy | Typical section | What it does |
| --- | --- | --- |
| `validate-jwt` | `<inbound>` | Validates a JWT token before reaching the backend |
| `rate-limit` | `<inbound>` | Limits calls per subscription in a time window (e.g. 10 calls / 60 s) |
| `rate-limit-by-key` | `<inbound>` | Same as `rate-limit` but by a custom key (IP, header, JWT claim...) |
| `quota` | `<inbound>` | Cumulative limit over a long period (e.g. 1000 calls/month). Complements `rate-limit` |
| `cache-lookup` / `cache-store` | `<inbound>` / `<outbound>` | Caches responses so the backend isn't hit on every request |
| `set-header` | `<inbound>` / `<outbound>` | Adds or modifies headers in the request or response |
| `rewrite-uri` | `<inbound>` | Changes the URL before sending it to the backend |
| `mock-response` | `<inbound>` | Returns a simulated response without calling the backend. Useful in development |
| `forward-request` | `<backend>` | Forwards the request to the backend (if omitted, no call is made) |

---

### Policy pipeline syntax

Base structure that always wraps all policies:

```xml
<policies>
    <inbound>
        <base />
        <!-- inbound policies go here -->
    </inbound>

    <backend>
        <base />
        <!-- control of the backend call goes here -->
    </backend>

    <outbound>
        <base />
        <!-- outbound policies go here -->
    </outbound>

    <on-error>
        <base />
        <!-- error handling goes here -->
    </on-error>
</policies>
```

> `<base />` inherits the policies of the parent level. If you omit it, nothing inherited from Global → Product → API runs.

#### What goes in each section

| Section | Purpose | Example policies |
|---------|---------|------------------|
| `<inbound>` | Prepare and validate the request before it goes to the backend | `validate-jwt`, `rate-limit`, `set-header`, `cors`, `return-response` |
| `<backend>` | Control how the backend call is executed | `retry`, `forward-request` |
| `<outbound>` | Transform the response before returning it to the client | `set-header` (delete internal headers), `cache-store` |
| `<on-error>` | Consistent error response if something fails | `return-response` with an error code |

> `cors` goes in `<inbound>` even though its headers appear in the response — APIM intercepts the OPTIONS preflight in inbound and answers directly without reaching the backend.

#### `<base />`

Calls the policies inherited from the upper level. Policies have four scope levels:

```
Global (all APIs)
  └── Product (group of APIs)
        └── API (one specific API)
              └── Operation (one specific endpoint)
```

`<base />` inside `<inbound>` means "apply the API-level and Global policies first, before mine". If you remove it, the policies of the upper levels do not run for that operation.

#### Full pipeline example

```xml
<policies>
    <inbound>
        <base />

        <!-- 1. Validate that the client has a valid JWT -->
        <validate-jwt header-name="Authorization" failed-validation-httpcode="401">
            <openid-config url="https://login.microsoftonline.com/{tenant-id}/v2.0/.well-known/openid-configuration" />
            <required-claims>
                <claim name="aud"><value>api://my-api</value></claim>
            </required-claims>
        </validate-jwt>

        <!-- 2. Limit to 100 calls per minute per subscription -->
        <rate-limit calls="100" renewal-period="60" />

        <!-- 3. Inject the backend API key (hidden from the client) -->
        <set-header name="X-Backend-Key" exists-action="override">
            <value>{{backend-api-key}}</value>
        </set-header>

        <!-- 4. Remove a header the client shouldn't see -->
        <set-header name="X-Internal-Debug" exists-action="delete" />
    </inbound>

    <backend>
        <base />

        <!-- Retry up to 3 times if the backend returns 503 -->
        <retry condition="@(context.Response.StatusCode == 503)" count="3" interval="2">
            <forward-request />
        </retry>
    </backend>

    <outbound>
        <base />

        <!-- Delete internal header from the response before returning it to the client -->
        <set-header name="X-Powered-By" exists-action="delete" />

        <!-- Cache the response for 5 minutes -->
        <cache-store duration="300" />
    </outbound>

    <on-error>
        <base />

        <!-- Uniform error response in case of failure -->
        <return-response>
            <set-status code="500" reason="Internal Server Error" />
            <set-header name="Content-Type" exists-action="override">
                <value>application/json</value>
            </set-header>
            <set-body>{"error": "Internal gateway error"}</set-body>
        </return-response>
    </on-error>
</policies>
```

---

### Named Values — static reusable values

Variables configured in the APIM portal (Settings → Named Values) and referenced in policies with `{{name}}`. Their value never appears in the XML — it is resolved at runtime.

**When to use them:** secrets, backend URLs, API keys, values that change between environments (dev/prod) but are fixed within each environment. They can be linked to Azure Key Vault.

```xml
<!-- In the APIM portal you store:
     "backend-url"     → "https://internal-api.mycompany.com/v2"
     "backend-api-key" → "supersecret123"
-->

<inbound>
    <!-- Change the target backend using a Named Value -->
    <set-backend-service base-url="{{backend-url}}" />

    <!-- Inject the API key to the backend without the client seeing it -->
    <set-header name="X-Api-Key" exists-action="override">
        <value>{{backend-api-key}}</value>
    </set-header>
</inbound>
```

> If you change the value in the portal, all policies that use it are updated automatically. Never hardcode secrets in the policy XML.

---

### set-variable — dynamic pipeline variables

They compute a value at runtime (during the request) and store it for reuse later in the same pipeline. They disappear when that request's lifecycle ends.

**When to use it:** when you need to compute something complex once (read a header, extract a JWT claim, transform a value) and reuse it at several points in the pipeline without repeating the logic.

```xml
<inbound>
    <!-- Extract the role from the header and store it in a variable -->
    <set-variable name="userRole"
        value="@(context.Request.Headers.GetValueOrDefault("X-Role", "guest"))" />

    <!-- Reuse the variable in a condition -->
    <choose>
        <when condition="@((string)context.Variables["userRole"] == "admin")">
            <!-- Only admins get through: add a header the backend recognizes -->
            <set-header name="X-Admin-Access" exists-action="override">
                <value>true</value>
            </set-header>
        </when>
        <otherwise>
            <!-- Everyone else gets 403 without reaching the backend -->
            <return-response>
                <set-status code="403" reason="Forbidden" />
                <set-body>{"error": "Unauthorized access"}</set-body>
            </return-response>
        </otherwise>
    </choose>
</inbound>

<outbound>
    <!-- The variable is still available in outbound -->
    <set-header name="X-Processed-Role" exists-action="override">
        <value>@((string)context.Variables["userRole"])</value>
    </set-header>
</outbound>
```

#### Named Values vs set-variable

| | Named Value `{{...}}` | set-variable `context.Variables` |
|---|---|---|
| **Lifetime** | Permanent (configured in the portal) | Lasts only one request |
| **Value** | Static, the same for all requests | Dynamic, computed on each request |
| **Typical use** | Secrets, URLs, API keys | Data extracted from the JWT, headers, conditional logic |
| **Visible in the XML** | No (resolved at runtime) | Yes, in the `@(...)` expression |

---

### Authorization and access

#### 1. Subscription Keys (the most common)

The client includes a key in every request:

```
Header:      Ocp-Apim-Subscription-Key: <key>
Query param: ?subscription-key=<key>
```

Subscriptions can be scoped to:

| Scope | Description |
|-------|-------------|
| **All APIs** | Global access (not recommended) |
| **Product** | Groups several APIs — the client subscribes to the product |
| **Single API** | Access only to one specific API |

The typical flow: the developer goes to the Developer Portal → requests access to a product → gets their key.

> The subscription key identifies **which application** is calling, not **who the user is**. It can be the only security mechanism on internal APIs or APIs without user data.

#### 2. OAuth 2.0 / JWT (`validate-jwt` policy)

OAuth 2.0 (the protocol) produces JWT tokens. APIM does not validate the token automatically — you do it with a policy in `<inbound>`:

```xml
<validate-jwt header-name="Authorization" failed-validation-httpcode="401">
    <openid-config url="https://login.microsoftonline.com/{tenant}/..." />
    <required-claims>
        <claim name="aud">
            <value>api://my-api</value>
        </claim>
    </required-claims>
</validate-jwt>
```

#### Possible combinations

| Configuration | When to use it |
|---------------|----------------|
| **Subscription key only** | Internal APIs or APIs without user data |
| **Subscription key + JWT** | The most common — identifies the app AND the user |
| **JWT only** | When the token already identifies the client sufficiently |

Subscription key and JWT are not alternatives — they are complementary layers that protect different things.

---

### Products and Subscriptions

A **Product** groups one or more APIs and defines the terms of use. Consumers subscribe to the product, not directly to the API.

**Product types:**

| Type | Description |
|------|-------------|
| **Open** | No subscription required — anyone can call the API |
| **Protected** | Requires a subscription key — the consumer must subscribe first |

A subscription generates two keys (primary and secondary) that the consumer includes in every HTTP call in the header:

```
Ocp-Apim-Subscription-Key: <your-key>
```

There are three subscription scopes:

| Scope | Access it grants |
|-------|------------------|
| **All APIs** | All APIs published on the gateway |
| **Product** | Only the APIs grouped in a specific product |
| **Single API** | One specific API |

Products can be configured with automatic approval (`Requires subscription approval: No`), which lets the developer get access without publisher intervention.

---

### Tiers (service levels)

| Tier | Key characteristics |
|------|---------------------|
| **Consumption** | Serverless, pay per call, no VNet integration. Ideal for quick tests |
| **Developer** | No production SLA, for development and testing only. Do not use in production |
| **Basic** | Basic SLA, no VNet |
| **Standard** | Production SLA, no VNet |
| **Premium** | VNet integration (internal/external), multi-region deployment, higher scale |

---

### Additional concepts

**Revisions**
Allow changes to an API without affecting consumers. A revision is not public until you promote it to *current*. Useful for iterating and testing before publishing. Previous revisions remain accessible and you can roll back.

**Versions**
Allow exposing multiple versions of the same API at the same time (`/v1`, `/v2`). The consumer explicitly chooses the version. Versions coexist; it is not a rollback, they are different contracts in parallel.

| Concept | What it is for |
|----------|----------------|
| **Revision** | Iterate on an API without interrupting current consumers |
| **Version** | Publish versions with breaking changes that coexist on the gateway |

---

### AZ-204 exam keys

- The most-tested point on tiers is the distinction between **Consumption** (no VNet, serverless), **Premium** (VNet + multi-region) and **Developer** (no SLA, dev/test only).
- `<base />` is what inherits Global → Product → API policies; omit it and nothing inherited runs.
- `cors` belongs in `<inbound>` (APIM answers the OPTIONS preflight there).
- APIM does not validate JWTs on its own — you need the `validate-jwt` policy in `<inbound>`.
- Subscription key identifies the app, JWT identifies the user; they are complementary, not alternatives.
- Secrets and environment-specific values go in Named Values `{{...}}` (optionally backed by Key Vault), never hardcoded in policy XML.

## Relations

- [[azure-functions]] — a typical backend placed behind the APIM gateway
- [[microsoft-entra-id]] — issues the OAuth 2.0 / JWT tokens that `validate-jwt` checks

## Tags

#azure #az204
