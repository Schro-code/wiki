---
domain: aws
tags: [aws]
sources: []
updated: 2026-11-26
---

# AWS Regions and Availability Zones

## What it is

A **Region** is a geographic area where AWS runs infrastructure (e.g. `us-east-1`, `eu-south-2`). An **Availability Zone (AZ)** is one or more isolated data centers inside a Region (e.g. `us-east-1a`, `us-east-1b`).

## What problem it solves

A single data center can fail (power, network, fire). AZs are physically separate but connected by fast links, so you can spread an application across them and survive the loss of one. Regions let you place workloads close to users and satisfy data-residency rules.

## Notes

- A Region is **not** a country, and an AZ is **not** a Region: AZ ⊂ Region.
- Most resources are **regional** (a VPC), but some are **zonal** (a [[vpc-subnets|subnet]] lives in exactly one AZ, an [[amazon-ec2|EC2 instance]] runs in one AZ).
- Resilience is not automatic: one instance in one AZ is still a single point of failure. You need instances in at least two AZs plus something that routes between them (see [[application-load-balancer]] and [[auto-scaling-groups]]).
- Both labs used `us-east-1` (N. Virginia) with AZs `us-east-1a` and `us-east-1b`.

## Relations

- [[amazon-vpc]] — a VPC spans all AZs of one Region
- [[vpc-subnets]] — a subnet is the unit that pins resources to one AZ
- [[horizontal-scaling-and-high-availability]] — multiple AZs are the basis of high availability

## Tags

#aws
