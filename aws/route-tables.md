---
domain: aws
tags: [aws, system-design]
sources: []
updated: 2026-11-26
---

# Route Tables

## What it is

A **route table** is a list of rules that says where network traffic leaving a subnet should be sent, based on the destination address. It is attached to **subnets**.

## What problem it solves

A network needs a map: "traffic for this range goes here". Without a route, packets have nowhere to go, even if every other piece is in place.

## Notes

- Every table has a **`local` route** (the VPC's own range), so resources inside the VPC can reach each other.
- The route `0.0.0.0/0 → igw-...` means "everything else goes to the [[internet-gateway]]". Its presence makes a subnet **public** (see [[vpc-subnets]]).
- It answers "is there a **path**?", not "is this traffic **allowed**?". The second question is for [[security-groups]].
- The route table is invisible in the instance's page. To check it, open the subnet's route table and look for the IGW route.
- The route table attaches to the **subnet**, so all instances in the subnet share it. You cannot set it per instance (security groups can do that).

## Relations

- [[vpc-subnets]] — the route table is attached to subnets
- [[internet-gateway]] — the target of the internet route
- [[security-groups]] — the allow/deny layer that comes after routing
- [[ec2-reachability-checklist]] — where the route fits as the "road"

## Tags

#aws #system-design
