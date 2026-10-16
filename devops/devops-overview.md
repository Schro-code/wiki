---
domain: devops
tags: [devops]
sources: []
updated: 2026-10-16
---

# DevOps — Overview

## What it is

A cultural and technical movement that breaks down the wall between software development (who writes code) and IT operations (who deploys and runs it). The goal is to ship software faster, more reliably, and with tighter feedback loops between production and the next development cycle.

## What problem it solves

Traditionally, dev teams threw code "over the wall" to ops teams who deployed it. Releases were infrequent, risky, and manually intensive. When something broke in production, dev said "it works on my machine" and ops said "you gave us broken code." DevOps collapses that wall by automating handoffs, sharing ownership, and measuring the whole pipeline.

## Notes

### The Three Ways (Gene Kim)

The three principles behind DevOps, from *The Phoenix Project* and *The DevOps Handbook*:

| Way | Direction | Core idea |
|---|---|---|
| **Flow** | Dev → Ops → Customer | Maximize throughput of work moving left to right. Small batches, automate everything, eliminate waste. |
| **Feedback** | Customer → Ops → Dev | Amplify feedback moving right to left. Fast test runs, real-time monitoring, alerts, post-mortems. |
| **Continuous Learning** | Across the org | Build a culture of experimentation. Learn from failure. Share knowledge. |

### CALMS framework

A common model for assessing DevOps maturity:

- **C**ulture — shared ownership, blame-free post-mortems, collaboration between dev and ops
- **A**utomation — eliminate manual steps in the pipeline (tests, deploys, infrastructure provisioning)
- **L**ean — small batches, reduce WIP, minimize handoff queues
- **M**easurement — if you can't measure it, you can't improve it
- **S**haring — knowledge, tooling, on-call rotation, runbooks

### DORA metrics

The four metrics from the DevOps Research and Assessment (DORA) program that predict software delivery performance. Elite performers vs low performers differ by orders of magnitude:

| Metric | What it measures | Elite performer |
|---|---|---|
| **Deployment Frequency** | How often you deploy to prod | Multiple times per day |
| **Lead Time for Changes** | Time from commit to running in prod | < 1 hour |
| **Change Failure Rate** | % of deployments that cause a prod incident | 0–15% |
| **MTTR** (Mean Time to Restore) | How long to recover from a prod failure | < 1 hour |

High deployment frequency does NOT mean high failure rate — the opposite is true. Small, frequent deployments are safer than large, infrequent ones.

### DevOps vs SRE vs Platform Engineering

Related disciplines that often overlap:

| Discipline | Focus |
|---|---|
| **DevOps** | Culture + practices: CI/CD, automation, shared ownership |
| **SRE** (Site Reliability Engineering) | Reliability through engineering. Error budgets, SLOs, SLAs, toil reduction. Google's implementation of DevOps. |
| **Platform Engineering** | Building internal developer platforms (IDPs) so dev teams can self-serve infrastructure without depending on ops. |

## Relations

- [[ci-cd-pipeline]] — the main technical artifact of DevOps
- [[branching-strategies]] — how teams organize parallel work to enable fast flow
- [[environments]] — the deployment targets a pipeline moves artifacts through
- [[deployment-strategies]] — how a new version replaces an old one in production

## Tags

#devops
