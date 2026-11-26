---
domain: aws
tags: [aws, system-design]
sources: []
updated: 2026-11-26
---

# Horizontal Scaling and High Availability

## What it is

**Horizontal scaling** means handling more load by adding more identical machines. **High availability** means the application keeps working when a component or a whole [[aws-regions-and-availability-zones|AZ]] fails. They use the same building blocks.

## What problem it solves

One instance in one AZ is both a **capacity limit** and a **single point of failure**. Making the machine bigger (vertical scaling) has a ceiling, costs more and is still a single point of failure.

## Notes

- **Scalability** = handle more load (add instances). **High availability** = survive failures (several AZs, replace unhealthy instances). Both come from the same stack: [[application-load-balancer]] + [[target-groups-and-health-checks]] + [[auto-scaling-groups]] + [[launch-templates]].
- **Instances must be interchangeable (stateless).** Employee Directory keeps photos in S3 and records in DynamoDB, not on the instance's disk. If state lived on the disk, each instance would show different data and terminating one would lose it. Access to those services uses an IAM role, not stored passwords.
- Reproducibility is a requirement: an instance can only be added at will if it can be built the same way every time ([[ami-and-user-data]]).
- Every layer needs its own door: the ALB's security group faces the internet, the instances' group should accept only the ALB.
- Redundancy is not free: min and max instances are a cost vs availability decision.

```
Users → ALB (2 AZs) → target group → instances in AZ-a and AZ-b
                                      ↑ created by ASG from the launch template
                                      ↑ ELB health check → ASG replaces unhealthy
```

## Relations

- [[application-load-balancer]] — the single entry point
- [[auto-scaling-groups]] — adds, removes and replaces instances
- [[launch-templates]] — the repeatable recipe
- [[aws-regions-and-availability-zones]] — AZs are the unit of failure isolation

## Tags

#aws #system-design
