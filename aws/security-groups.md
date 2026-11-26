---
domain: aws
tags: [aws, system-design]
sources: []
updated: 2026-11-26
---

# Security Groups

## What it is

A **security group** is a stateful, allow-only virtual firewall attached to a resource's **network interface (ENI)**, such as an EC2 instance or a load balancer. It belongs to a [[amazon-vpc|VPC]].

## What problem it solves

Without it, anything that can reach a resource over the network could talk to it on any port. A security group says exactly which traffic is allowed in and out, per resource.

## Notes

- **Attached to the resource, not the subnet.** Two instances in the same subnet can have different security groups. Route tables are the opposite: shared by the whole subnet.
- **Allow-only.** There are no deny rules. Anything not allowed is denied. Defaults: **inbound denied, outbound allowed**.
- **Stateful.** If a connection is allowed in one direction, the reply is allowed automatically.
  - Inbound rules govern who may **start** a conversation with you.
  - Outbound rules govern whom **you** may start one with.
  - So a web server with an inbound rule for port 80 does not need an outbound rule for the reply.
- **A rule's source can be another security group.** E.g. instances allow port 80 only from the load balancer's group instead of `0.0.0.0/0`. It keeps working when IPs change.
- The subnet-level, stateless alternative is the **Network ACL**, which supports deny rules.
- Typical layering from Lab 4: `LoadBalancerSG` (internet → ALB on 80) and `Web Security Group` (ALB → instances on 80). In the lab the Web group allowed `0.0.0.0/0` for simplicity.
- A permissive source (`0.0.0.0/0`) never causes a connection to be **blocked**; if traffic fails, look at routing, public IP or whether the app is listening.

## Relations

- [[route-tables]] — answers "is there a path"; the security group answers "is it allowed"
- [[application-load-balancer]] — has its own security group, separate from the instances'
- [[ec2-reachability-checklist]] — the "permission" check
- [[amazon-vpc]] — the scope of a security group

## Tags

#aws #system-design
