---
domain: aws
tags: [aws, system-design]
sources: []
updated: 2026-11-26
---

# Internet Gateway

## What it is

An **internet gateway (IGW)** is the component attached to a [[amazon-vpc|VPC]] that connects it to the internet, in both directions.

## What problem it solves

A VPC is isolated by default. The IGW is the single door that lets traffic enter and leave, and it also performs the mapping between an instance's private IP and its public IP.

## Notes

- It is attached to the **VPC**, not to a subnet. Which subnets use it is decided by their [[route-tables|route tables]].
- Just attaching it does nothing for a subnet: a route `0.0.0.0/0 → igw-...` must exist.
- The IGW only helps instances that have a **public IP**. Without one, the instance has no address the internet can reply to.
- It is horizontally scaled and managed by AWS, so there is nothing to size or maintain.
- Request path into an app: Internet → IGW → VPC → subnet (route table) → [[security-groups|security group]] → instance.

## Relations

- [[amazon-vpc]] — the network the IGW is attached to
- [[route-tables]] — send traffic to the IGW
- [[ec2-reachability-checklist]] — the IGW is the "door"

## Tags

#aws #system-design
