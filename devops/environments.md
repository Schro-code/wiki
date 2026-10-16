---
domain: devops
tags: [devops]
sources: []
updated: 2026-10-16
---

# Environments

## What it is

An environment is an isolated deployment of the same application at a specific stage of the release pipeline. Each environment has a distinct purpose, audience, and confidence level. Code moves through environments as it gains validation before reaching real users.

## What problem it solves

Without separate environments, developers test against production data, QA teams can't validate before users see it, and a broken deploy instantly breaks the live product. Environments create safety buffers: you find problems before real users do.

## Notes

### Standard environments

| Environment | Common aliases | Purpose | Who uses it |
|---|---|---|---|
| **Local** | dev, machine | Developer's own machine. No shared infrastructure. | Individual developer |
| **Development** | dev, integration | Shared integration target. All feature branches merged here. Fast, not always stable. | Dev team |
| **Staging** | QA, pre-prod, UAT, test | Near-mirror of production. Final validation before release. Should be as close to prod as possible. | QA, product managers, sometimes clients |
| **Production** | prod, live | Real users, real data. Every deploy here carries risk. | Everyone |

Some organizations add:
- **Hotfix / sandbox** — ephemeral environment for quick experiments
- **Performance / load testing** — isolated environment for stress tests that would impact staging

---

### Environment parity

The most important principle: **staging should be as close to production as possible.**

Common parity violations that cause "works on staging, breaks on prod" failures:

| What's different | How it bites you |
|---|---|
| Different OS or runtime version | Library behaves differently |
| Smaller database on staging | Performance queries look fine, explode on prod data volume |
| Mocked third-party services on staging | Real API has rate limits, different error formats, auth requirements |
| Different config values | Connection strings, timeouts, feature flags set differently |
| Different infrastructure (bare metal vs containers) | File system behavior, network topology, memory limits |

The fix: use the same infrastructure-as-code (Terraform, Helm) for both, differ only in scale and access credentials.

---

### Environment-specific configuration

The application binary (Docker image, JAR, wheel) should be **identical across environments**. What changes is configuration, injected at runtime via:

- **Environment variables** — the standard approach (12-factor app)
- **Secrets manager** — for credentials: AWS Secrets Manager, Azure Key Vault, HashiCorp Vault
- **Config maps** — for non-secret config in Kubernetes

Never bake environment-specific values into the artifact. Never commit secrets to the repo.

---

### Ephemeral environments (preview environments)

Some teams create a throwaway environment per pull request, automatically:

1. PR is opened
2. Pipeline builds the artifact, spins up a fresh environment
3. QA and product review in isolation (no other PRs interfere)
4. PR is merged → environment is destroyed

**Trade-offs:**
- Pro: complete isolation, test each change independently
- Con: infrastructure cost, slower pipeline, complexity in managing many short-lived envs

Tools: Vercel (frontend), Render, Railway, Argo CD ApplicationSets, GitHub Environments.

---

### Promotion flow

Code (as an artifact) moves through environments, not re-deployed from source:

```
[build] → artifact (Docker image:sha-abc123)
              ↓
         staging deploy
              ↓
         staging tests pass
              ↓
         production deploy ← same image:sha-abc123
```

"Promote to production" means: deploy the artifact that already passed staging. Not: rebuild from source and hope it's the same.

---

### Your model mapped

```
feature/* branches  →  local / dev
dev branch          →  development / staging environment
main branch         →  production environment
```

The gap in most simple setups: `dev` environment is often not a true mirror of prod. Teams add a dedicated staging environment (separate infrastructure, prod-like data, real third-party API keys) when they start having "works on dev, broken on prod" incidents frequently.

## Relations

- [[ci-cd-pipeline]] — the pipeline drives promotion between environments
- [[deployment-strategies]] — how a new version is installed in each environment
- [[devops-overview]] — environments are the stages of the Flow principle

## Tags

#devops
