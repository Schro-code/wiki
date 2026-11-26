---
domain: aws
tags: [aws, system-design]
sources: []
updated: 2026-11-26
---

# EC2 Reachability Checklist

## What it is

A mental checklist for "I launched an instance and the page does not load". An instance on the public internet needs **four things**, each provided by a different component.

## What problem it solves

Reachability fails silently: nothing says *which* piece is missing. Naming the four layers turns guessing into a short list of checks.

## Notes

| # | Need | Provided by | How to check |
|---|---|---|---|
| 1 | **Door** | [[internet-gateway]] attached to the VPC | VPC → Internet gateways → state *Attached* |
| 2 | **Road** | Route `0.0.0.0/0 → igw-...` in the subnet's [[route-tables|route table]] | Subnet → Route table → Routes |
| 3 | **Address** | Public IPv4 (set at launch) | Instance details → Public IPv4 address |
| 4 | **Permission** | [[security-groups|Security group]] inbound rule, e.g. HTTP 80 | Instance → Security tab |

Path of a request: Internet → IGW → VPC → subnet (route table) → security group → app.

A fifth layer lives **inside** the instance: the **app must be listening** on that port. If 1–4 are fine and the page still does not load, check the system log: the [[ami-and-user-data|user data]] script may have failed.

Common slips:
- Typing `https://` when only port 80 is open. Use `http://`.
- Testing before user data finishes (3–5 minutes after launch).
- Trusting the console: a public IP or "3/3 checks passed" proves neither routing nor that the app is running.

Quick test to separate "app" from "load balancer": open `http://<instance public IP>` directly. If it does not load but the security group allows port 80, nothing is listening.

## Relations

- [[internet-gateway]] — the door
- [[route-tables]] — the road
- [[security-groups]] — the permission
- [[amazon-ec2]] — the public IP and the app inside
- [[target-groups-and-health-checks]] — the same ladder applies when a load balancer reports unhealthy targets

## Tags

#aws #system-design
