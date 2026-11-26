---
domain: aws
tags: [aws]
sources: []
updated: 2026-11-26
---

# Auto Scaling Groups

## What it is

An **Auto Scaling group (ASG)** keeps a number of EC2 instances running, created from a [[launch-templates|launch template]] and spread across the subnets (AZs) you choose. It adds instances under load, removes them when load drops, and replaces broken ones.

## What problem it solves

Capacity and failures should not need a human. The ASG gives you scaling (handles more traffic) and self-healing (replaces failed instances) with one definition.

## Notes

- **Min / desired / max**: Lab 4 used 2 / 2 / 4. It never goes below 2, aims for 2, and never exceeds 4.
  - **Min 2** across two AZs means one AZ can fail and one instance keeps serving.
  - **Max** is the ceiling and protects against runaway cost. If it is reached and load is still high, the ASG does nothing more, and users may see slowness.
- Registers new instances in the [[target-groups-and-health-checks|target group]] automatically, and the [[application-load-balancer]] then sends them traffic.
- **Target tracking policy**: "keep average CPU near 30%". Above target the ASG scales **out** (adds instances); when load disappears it scales **in**, back down to the minimum. *Warmup* (300 s) stops it reacting before a new instance is ready. The metrics come from CloudWatch.
- **ELB health checks**: when enabled on the ASG, it also replaces instances the load balancer marks unhealthy. Without them it only watches EC2 status checks, sees "running", and leaves a crashed app in place (the ALB stops sending it traffic, but nothing replaces it).
- Unhealthy instances are **terminated and replaced**, not restarted.
- The ASG only manages instances it created. The original hand-built instance in Lab 4 was not part of it, so the lab terminates it.
- Notifications via SNS tell you when instances launch or terminate.

**Division of work:** the ALB decides **which** healthy instance gets each request; the ASG decides **how many** instances exist and **where**. The ALB's health check detects a failure and the ASG fixes it.

## Relations

- [[launch-templates]] — how the ASG builds an instance
- [[application-load-balancer]] — distributes traffic to the ASG's instances
- [[target-groups-and-health-checks]] — membership list and health signal
- [[horizontal-scaling-and-high-availability]] — the design goal the ASG implements
- [[vpc-subnets]] — the ASG chooses subnets in two AZs

## Tags

#aws
