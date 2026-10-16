---
domain: devops
tags: [devops]
sources: []
updated: 2026-10-16
---

# CI/CD Pipeline

## What it is

A CI/CD pipeline is an automated sequence of steps that runs every time code changes, taking it from a commit all the way to a running deployment. It's the main technical artifact of DevOps — the automation that enforces quality and removes manual handoffs.

## What problem it solves

Without a pipeline, shipping software involves manual steps: someone runs tests locally (maybe), someone deploys manually via SSH (maybe), someone checks logs (maybe). Manual steps are slow, inconsistent, and don't scale. The pipeline makes every step mandatory, reproducible, and fast.

## Notes

### CI vs CD — the distinction

**Continuous Integration (CI):** Automatically integrate every commit into the shared codebase, run tests, and report the result. The goal is to find integration problems within minutes, not days.

**Continuous Delivery (CD):** The pipeline always produces a deployable artifact. Deploying to production is a deliberate human decision, but it can happen at any time with a button press.

**Continuous Deployment:** Same as Continuous Delivery, but the last step (deploy to prod) is also automated — every green build goes live automatically. Requires very high confidence in the test suite.

Most teams practice Continuous Delivery, not Continuous Deployment.

---

### Full pipeline stages

Your current model (tests + linter) covers only two of the roughly 10 stages a mature pipeline has:

```
Trigger
  └─→ Checkout
       └─→ Install dependencies
            └─→ Lint
                 └─→ Test
                      └─→ Build / Compile
                           └─→ Security scan
                                └─→ Publish artifact
                                     └─→ Deploy to staging
                                          └─→ Integration / E2E tests
                                               └─→ Deploy to production ← (manual gate or auto)
                                                    └─→ Smoke tests
```

#### Stage breakdown

| Stage | What it does | Tools |
|---|---|---|
| **Trigger** | Starts the pipeline on PR open, push, merge, or schedule | GitHub Actions, GitLab CI, Jenkins |
| **Checkout** | Gets the code at that commit | git |
| **Install deps** | Restores packages from cache or registry | npm ci, pip install, uv sync, mvn |
| **Lint** | Checks style, formatting, and syntax errors without running code | ESLint, Ruff, Flake8, Pylint, Prettier |
| **Test** | Unit tests, integration tests, coverage thresholds | pytest, Jest, JUnit |
| **Build / Compile** | Produces an artifact from source: transpile JS, compile Java, build Docker image, bundle | tsc, webpack, Docker build, Maven |
| **Security scan** | SAST (static analysis of code), dependency vulnerability scan, secret detection | Semgrep, SonarQube, Trivy, Snyk, Gitleaks |
| **Publish artifact** | Pushes the artifact to a registry so it can be deployed | Docker Hub, ECR, Artifact Registry, PyPI, npm |
| **Deploy to staging** | Installs the artifact in a staging environment | Helm, Terraform, kubectl, ArgoCD |
| **Integration / E2E tests** | Tests the running system end-to-end, not individual units | Playwright, Cypress, Postman/Newman |
| **Deploy to production** | Promotes the artifact from staging to prod | Same tools as staging deploy |
| **Smoke tests** | Quick sanity checks that prod is healthy after deploy | curl health endpoint, synthetic test |

---

### Pipeline-as-code

The pipeline definition lives in a file **inside the repo**. This is non-negotiable in modern DevOps:

| Tool | File location |
|---|---|
| GitHub Actions | `.github/workflows/*.yml` |
| GitLab CI | `.gitlab-ci.yml` |
| Jenkins | `Jenkinsfile` |
| CircleCI | `.circleci/config.yml` |
| Azure Pipelines | `azure-pipelines.yml` |

Storing the pipeline in the repo means: it's versioned alongside the code, reviewed in PRs, and every branch can have a different pipeline if needed.

---

### The artifact principle

**Build once, deploy many.** The pipeline produces one immutable artifact (a Docker image, a compiled binary, a wheel) that is tagged with a version or commit SHA. That exact artifact is deployed to staging — and if it passes — promoted to production.

Never build separately per environment. Environment-specific behavior comes from **environment variables or config**, not from a different build.

---

### Branch-pipeline mapping

Different branches usually trigger different pipeline stages:

| Branch | Typical pipeline |
|---|---|
| `feature/*` (PR) | Lint → Test → Build (no deploy) |
| `develop` / `dev` | Lint → Test → Build → Security scan → Deploy to staging |
| `main` | Full pipeline → Deploy to production |

---

### Fast feedback principle

The stages should be ordered so that the **fastest and cheapest checks run first**. If lint fails, there's no point running a 10-minute integration test suite. Fail fast, fail cheap.

Typical ordering by cost:
1. Lint (seconds)
2. Unit tests (seconds to minutes)
3. Build (minutes)
4. Integration/E2E tests (minutes to tens of minutes)
5. Security scan (can be parallelized)
6. Deploy (minutes)

## Relations

- [[branching-strategies]] — which branch triggers which pipeline stages
- [[environments]] — the deployment targets (staging, prod) a pipeline pushes to
- [[deployment-strategies]] — what happens at the deploy step
- [[devops-overview]] — pipelines are the main implementation of the Flow principle

## Tags

#devops
