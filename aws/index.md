---
domain: aws
type: index
updated: 2026-11-26
---

# AWS — Domain Index

Foundations of AWS networking and compute, learned hands-on through the AWS training labs (Lab 2: VPC and a web app on EC2; Lab 4: high availability with a load balancer and Auto Scaling). Each page explains what a service is, why it exists and how it relates to the others.

## Pages

| Page | When to look | Sources |
|---|---|---|
| [[aws-regions-and-availability-zones]] | Look when you confuse Region, AZ and country, or need to know what is regional vs zonal | AWS training labs |
| [[amazon-vpc]] | Look when you want to understand what a VPC is and why it is private by default | AWS training labs |
| [[vpc-subnets]] | Look when deciding what makes a subnet public or private, and how subnets tie resources to an AZ | AWS training labs |
| [[route-tables]] | Look when a subnet should reach the internet or a path seems to be missing | AWS training labs |
| [[internet-gateway]] | Look when you need to know what connects a VPC to the internet and what else is needed | AWS training labs |
| [[security-groups]] | Look when studying stateful firewalls, what they attach to, inbound vs outbound, and security group chaining | AWS training labs |
| [[amazon-ec2]] | Look when launching an instance: public IP behaviour, status checks, IAM role, system log | AWS training labs |
| [[ami-and-user-data]] | Look when choosing between a custom image and a boot script to configure servers | AWS training labs |
| [[ec2-reachability-checklist]] | Look when an instance's page does not load: the four things (door, road, address, permission) plus the app | AWS training labs |
| [[application-load-balancer]] | Look when you want one DNS name in front of several instances across AZs | AWS training labs |
| [[target-groups-and-health-checks]] | Look when targets are unhealthy, or to understand thresholds and how to read the status details | AWS training labs |
| [[launch-templates]] | Look when an Auto Scaling group needs a repeatable recipe for instances | AWS training labs |
| [[auto-scaling-groups]] | Look when studying min/desired/max, target tracking, self-healing and the ALB vs ASG split | AWS training labs |
| [[horizontal-scaling-and-high-availability]] | Look when you want the big picture: why stateless instances, multiple AZs and the load balancer fit together | AWS training labs |

## Related

- [[ip-addresses]] — CIDR and private IP ranges used by VPCs (system-design)
