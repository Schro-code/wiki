---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# Azure App Configuration

## What it is

A centralized Azure service for managing application configuration and feature flags, separate from the code. It acts as a **central configuration repository**: instead of storing settings in files (`appsettings.json`), environment variables or code, you store them in App Config and all your apps read them from there.

It complements Key Vault: App Config stores non-sensitive settings; Key Vault stores secrets.

```
appsettings.json  →  App Configuration Store (in Azure)
     ↑                        ↑
 (local, fixed)         (central, dynamic)
```

## What problem it solves

Imagine you have an app with this configuration:

```json
{
  "DatabaseUrl": "Server=dev-server;Database=mydb",
  "MaxRetries": 3,
  "ShowBetaFeature": false
}
```

What happens when you have these problems?

- You need to change `MaxRetries` from 3 to 5 **without redeploying** the app
- You have 3 environments (dev, staging, prod) with different values — how do you manage them?
- You want to enable `ShowBetaFeature` only for 10% of users
- Your team has 5 microservices that share the same config — do you duplicate it in each one?

Azure App Configuration solves all of that.

## Notes

### Concepts

#### 1. The Store

It is the resource you create in Azure. Think of it as a configuration database.

```
Azure Portal
└── App Configuration
    └── my-store-dev          ← your store
        ├── key: DatabaseUrl  → value: "Server=..."
        ├── key: MaxRetries   → value: "3"
        └── key: Theme        → value: "dark"
```

Each store has its own URL: `https://my-store-dev.azconfig.io`

---

#### 2. Key-Value pairs

The basic unit. Each setting is a **key → value** pair.

**Keys** — use `:` or `/` to create hierarchy (it is only a naming convention):

```
Database:Host        →  "prod-server.database.windows.net"
Database:Port        →  "5432"
WebApp:Theme         →  "dark"
WebApp:MaxRetries    →  "5"
```

This lets you later read the whole `Database:*` group at once.

**Values** — are always strings. If you need JSON, you store it as a string and your app parses it.

---

#### 3. Labels — the key to multiple environments

The problem without labels: how do I store the value of `Database:Host` for dev and for prod?

With labels, the same key can have **several values depending on the label**:

```
Key: Database:Host | Label: Development  →  "dev-server"
Key: Database:Host | Label: Staging      →  "staging-server"
Key: Database:Host | Label: Production   →  "prod-server"
```

When your app starts, you tell it "give me the config with label `Production`" and it gets only the production values.

> **No label** = label `null` (default value if you specify nothing)

---

#### 4. Feature Flags

They let you enable or disable a feature **without touching code or redeploying**.

Imagine you have a "Beta" section on your website. With a feature flag:

```
Feature: "NewDashboard"
State:   enabled = false   ← disabled for everyone
```

In your code:

```csharp
if (await featureManager.IsEnabledAsync("NewDashboard"))
{
    // show the new dashboard
}
else
{
    // show the old dashboard
}
```

When you want to enable it, you go to the Azure Portal, change `enabled = true`, and **without redeploying** your app starts showing it.

##### Feature Filters — conditional activation

| Filter | When it enables the feature | Consistency |
|--------|-----------------------------|-------------|
| `PercentageFilter` | For a % of evaluations (random per call) | No — varies per request |
| `TimeWindowFilter` | Between two dates/times | Yes — time-based |
| `TargetingFilter` | For specific groups or users | Yes — hash of the user ID |

**How does 20% work in practice?**

- `PercentageFilter`: generates a random number on each call to `IsEnabledAsync`. If it falls within the 20%, it returns `true`. The same user can see different results on different requests.
- `TargetingFilter`: uses the **user ID** to compute a deterministic hash. The same user *always* gets the same result — always inside or always outside the 20%. It is the right choice when you need per-user consistency.

```csharp
// Configure TargetingFilter with user context
services.AddFeatureManagement()
        .WithTargeting<MyTargetingContextAccessor>();
```

---

#### 5. Dynamic configuration (without restarting)

Without this: you change a value in App Config → you have to restart your app for it to read it.

With dynamic refresh: the app **periodically checks** whether something changed and updates the values in memory.

**The Sentinel pattern** — if your app checks 50 keys every 30 seconds, that is a lot of calls to Azure. The solution:

```
Special key:  AppConfig:Sentinel  →  "v1"
```

Your app only checks that key every 30 seconds (cheap). If it changes from `v1` to `v2`, it then reloads **all** the configuration. You control when to "trigger" the reload by changing that value.

```csharp
options.ConfigureRefresh(refresh =>
{
    refresh.Register("AppConfig:Sentinel", refreshAll: true)
           .SetCacheExpiration(TimeSpan.FromSeconds(30));
});
```

And in the middleware: `app.UseAzureAppConfiguration()`

---

#### 6. Key Vault integration

App Configuration **does not store secrets**. That is what Key Vault is for.

The integration: App Config stores a **reference** to the secret, not the secret itself.

```
App Configuration
└── Key: Database:Password
    Value: reference → https://my-vault.vault.azure.net/secrets/DbPassword
                                          ↓
                                    Key Vault
                                    └── DbPassword: "<secret-value>"
```

Your app reads from App Config, and when it finds a Key Vault reference, it resolves it automatically with `.ConfigureKeyVault(kv => kv.SetCredential(...))`.

---

#### 7. Connecting from the app (.NET)

```csharp
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect("<connection-string>")
           .Select("WebApp:*", "Production")  // prefix + label
           .UseFeatureFlags();
});
```

**Recommended authentication in production** — Managed Identity with `DefaultAzureCredential`, not a connection string:

```csharp
options.Connect(new Uri("https://mystore.azconfig.io"), new DefaultAzureCredential())
```

---

### Complete flow

```
1. Your app starts
        ↓
2. Reads configuration from App Config (filtering by label "Production")
        ↓
3. Resolves Key Vault references if there are any
        ↓
4. The app runs with that config in memory
        ↓
5. Every 30s it checks the Sentinel Key
        ↓
6. If it changed → reloads config without restarting
        ↓
7. If a feature flag changes → the app detects it on the next check
```

---

### Authentication and RBAC

Two ways to connect:

| Method | When to use |
|--------|-------------|
| **Connection String** | Local development |
| **Managed Identity** (`DefaultAzureCredential`) | Production — no credentials in the code |

RBAC roles:

| Role | Permissions |
|------|-------------|
| `App Configuration Data Reader` | Read-only (what the app needs) |
| `App Configuration Data Owner` | Read and write |

---

### Tiers

| | **Free** | **Standard** |
|--|----------|--------------|
| Stores | 1 per subscription | Unlimited |
| Storage | 10 MB | 1 GB |
| Requests/day | 1,000 | Unlimited |
| SLA | No | 99.9% |
| Geo-replication | No | Yes |

---

### Visual summary

```
                    ┌─────────────────────────────┐
                    │     App Configuration Store  │
                    │                              │
                    │  Database:Host  (Dev)  → ... │
                    │  Database:Host  (Prod) → ... │
                    │  WebApp:Theme           → ... │
                    │                              │
                    │  Feature: NewDashboard       │
                    │    enabled: true             │
                    │    filter: TargetingFilter   │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ↓              ↓              ↓
              App Service    Container App    Azure Function
              (label=Prod)   (label=Prod)    (label=Prod)
```

---

### AZ-204 exam keys

Most frequently asked:

1. Difference between App Config and Key Vault — App Config stores settings, Key Vault stores secrets
2. **Labels** for environments — same key, different value depending on the label
3. **Feature flags** and when to use `TargetingFilter` vs `PercentageFilter`
4. **Sentinel key** pattern for dynamic refresh without restarting
5. Authentication with **Managed Identity** in production
6. How to reference Key Vault secrets from App Config

## Relations

- [[microsoft-entra-id]] — Managed Identity and `DefaultAzureCredential` used to authenticate against the store
- [[azure-rbac]] — App Configuration Data Reader / Data Owner roles control access to the store

## Tags

#azure #az204
