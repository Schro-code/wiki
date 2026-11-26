---
domain: aws
tags: [aws]
sources: []
updated: 2026-11-26
---

# Application Load Balancer

## What it is

An **Application Load Balancer (ALB)** is a managed Layer 7 (HTTP/HTTPS) load balancer. It gives users **one DNS name** and distributes requests across several instances.

## What problem it solves

With one instance, users hit its own IP, capacity is limited and a failure takes the app down. The ALB hides the instances behind a single entry point, sends traffic only to healthy ones, and lets you add or remove instances without users noticing.

## Notes

- It is deployed across **at least two subnets in different AZs**, so the front door itself survives an AZ failure.
- A **listener** is a rule "listen on port X, then forward to Y". In the lab: port 80 → forward to `lab-app-target-group`. The listener forwards to a [[target-groups-and-health-checks|target group]], never directly to instances.
- It has its **own security group** (`LoadBalancerSG`: internet → port 80), separate from the instances' group. A good setup lets the instances accept traffic only from that group. See [[security-groups]].
- Use `http://` for the lab DNS name: browsers may redirect to `https://` (443), which the lab does not configure.
- The ALB spreads requests across AZs. In Lab 4 you see this by refreshing the app's Configuration page and watching the AZ change.

## Relations

- [[target-groups-and-health-checks]] — the pool of instances the ALB forwards to, and the health checks that decide who receives traffic
- [[auto-scaling-groups]] — register and replace instances behind the ALB
- [[horizontal-scaling-and-high-availability]] — the ALB is the entry point that makes scaling out possible
- [[vpc-subnets]] — the ALB is placed in public subnets in two AZs

## Tags

#aws
