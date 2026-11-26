---
domain: aws
tags: [aws, system-design]
sources: []
updated: 2026-11-26
---

# VPC Subnets

## What it is

A **subnet** is a slice of a [[amazon-vpc|VPC's]] IP range that lives in **exactly one Availability Zone**. Resources such as EC2 instances are launched into a subnet.

## What problem it solves

It lets you organize and place resources: choose which AZ they run in, and separate parts of the network that need different internet exposure (e.g. web servers vs databases).

## Notes

- **Public vs private is a property of the subnet's route table**, not of the subnet itself. A subnet is public only if its [[route-tables|route table]] has `0.0.0.0/0 → igw-...` (an [[internet-gateway]]). Otherwise it is private.
- Subnets are **not** for "prod vs dev". That would be separate VPCs or accounts. Subnets are mostly about AZ placement and internet exposure.
- Being in a public subnet is not enough for an instance to be reachable: it also needs a public IP and a [[security-groups|security group]] that allows the traffic. See [[ec2-reachability-checklist]].
- To survive an AZ failure you need subnets (and instances) in at least two AZs. An [[application-load-balancer]] and an [[auto-scaling-groups|Auto Scaling group]] are both told which subnets to use.
- Lab 2: `10.0.0.0/24` and `10.0.1.0/24` in two AZs. Lab 4: Public Subnet 1 and Public Subnet 2, one per AZ.

## Relations

- [[amazon-vpc]] — the parent network the subnet is carved from
- [[route-tables]] — attached to the subnet; decides whether it is public
- [[aws-regions-and-availability-zones]] — the AZ a subnet is pinned to
- [[amazon-ec2]] — instances are launched into a subnet

## Tags

#aws #system-design
