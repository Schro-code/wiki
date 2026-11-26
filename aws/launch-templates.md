---
domain: aws
tags: [aws]
sources: []
updated: 2026-11-26
---

# Launch Templates

## What it is

A **launch template** is a saved recipe for creating an EC2 instance: AMI, instance type, security groups, IAM instance profile, user data, key pair and other settings.

## What problem it solves

An [[auto-scaling-groups|Auto Scaling group]] must create instances **on its own**, e.g. at 3 a.m. when CPU spikes, with nobody to click through the console. It needs the exact recipe. If the only working server were hand-built, nothing could reproduce it.

## Notes

- Contents in Lab 4: Amazon Linux 2023 AMI, `t3.micro`, `Web Security Group`, IAM profile `EmployeeDirectoryAppRole`, no key pair, and the user data script with its three placeholders (images bucket, installation bucket, region) replaced.
- It is the same idea as [[ami-and-user-data]] saved in one place: a "finished photo" (AMI) or a "recipe" (user data), plus the sizing and permissions.
- **Versioned**: editing a template creates a new version. The ASG must be told to use it, and running instances are **not** changed. They must be replaced (e.g. instance refresh, or terminate and let the ASG recreate them).
- Mistakes here propagate to every instance: wrong security group, missing IAM role, or a bad user data value all produce unhealthy instances.

## Relations

- [[auto-scaling-groups]] — consume the template to create instances
- [[amazon-ec2]] — what the template describes
- [[ami-and-user-data]] — the boot image and script stored in it
- [[security-groups]] — chosen in the template

## Tags

#aws
