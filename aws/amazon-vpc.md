---
domain: aws
tags: [aws, system-design]
sources: []
updated: 2026-11-26
---

# Amazon VPC

## What it is

A **Virtual Private Cloud (VPC)** is your own isolated private network inside an AWS Region, defined by an IP range (CIDR block) such as `10.0.0.0/16`.

## What problem it solves

Without isolation, every customer's resources would share one flat network. A VPC gives you a private space where you decide the address range, which parts can reach the internet, and who can talk to what.

## Notes

- A VPC is **regional**: it spans every [[aws-regions-and-availability-zones|AZ]] in its Region. You carve it into [[vpc-subnets]].
- A new VPC is private by default. Reaching the internet needs three extra pieces: an [[internet-gateway]], a [[route-tables|route]] to it, and a public IP on the resource. See [[ec2-reachability-checklist]].
- [[security-groups]] belong to a VPC (a security group can only be used by resources in the VPC it was created in).
- CIDR notation and private ranges are explained in [[ip-addresses]].
- Lab 2 built `lab-2-vpc` (`10.0.0.0/16`) with subnets `10.0.0.0/24` and `10.0.1.0/24`. Lab 4 started from a pre-built "Lab VPC" with two public subnets in different AZs.

## Relations

- [[vpc-subnets]] — the slices of the VPC range, each in one AZ
- [[internet-gateway]] — the VPC's door to the internet
- [[route-tables]] — decide where subnet traffic goes
- [[security-groups]] — firewalls scoped to the VPC
- [[ip-addresses]] — CIDR and private IP ranges (system-design)

## Tags

#aws #system-design
