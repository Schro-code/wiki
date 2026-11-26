---
domain: aws
tags: [aws]
sources: []
updated: 2026-11-26
---

# Amazon EC2

## What it is

**Amazon Elastic Compute Cloud (EC2)** provides virtual servers ("instances") that you launch from an image ([[ami-and-user-data|AMI]]), with a chosen instance type (CPU/memory, e.g. `t3.micro`).

## What problem it solves

You get a server in minutes, pay for the time used, and do not buy or maintain hardware.

## Notes

- An instance runs in **one subnet, hence one AZ** (see [[vpc-subnets]]). It is not highly available by itself.
- **Public IPv4** is assigned **at launch**, driven by the subnet/instance setting *Auto-assign public IP*. It cannot be added to a running instance. Relaunch with it enabled, or associate an Elastic IP.
- The console showing a public IP does **not** prove the instance is reachable. See [[ec2-reachability-checklist]].
- **Status checks (3/3 passed)** only say the machine and its host are healthy. They say nothing about whether your application is running. That is what [[target-groups-and-health-checks|load balancer health checks]] are for.
- An **IAM role** (instance profile) lets the instance call other AWS services (e.g. S3, DynamoDB) without storing credentials. Without it, scripts like `aws s3 cp` fail on boot.
- Troubleshooting a boot script: *Actions → Monitor and troubleshoot → Get system log* shows what the user data did and where it failed.
- A hand-built instance is hard to reproduce. For several identical instances use a [[launch-templates|launch template]].

## Relations

- [[ami-and-user-data]] — what the instance boots from and how it is configured
- [[security-groups]] — firewall on the instance's network interface
- [[launch-templates]] — repeatable definition of an instance
- [[auto-scaling-groups]] — keeps a number of instances running

## Tags

#aws
