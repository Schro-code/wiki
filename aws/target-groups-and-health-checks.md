---
domain: aws
tags: [aws]
sources: []
updated: 2026-11-26
---

# Target Groups and Health Checks

## What it is

A **target group** is the list of resources (here, EC2 instances) that can receive traffic from a load balancer listener. It also runs the **health checks** that decide which targets are fit to receive it.

## What problem it solves

The [[application-load-balancer]] must know *who* to send requests to and *whether they work*. The target group is the dynamic membership list, and health checks keep broken instances out of rotation automatically.

## Notes

- The ALB asks each target "are you alive?" every **interval** seconds on a path/port (default: the traffic port, path `/`).
- Thresholds: **healthy threshold** = successes in a row before traffic is sent; **unhealthy threshold** = failures in a row before removal; **timeout** = how long to wait for one answer. Lab 4 used healthy 2, unhealthy 5, timeout 20 s, interval 30 s.
- Only healthy targets receive requests. Unhealthy ones are skipped, not restarted: the repair is done by the [[auto-scaling-groups|Auto Scaling group]], if ELB health checks are enabled on it.
- An [[amazon-ec2|EC2 status check]] and a load balancer health check measure different things: the machine vs the **app**.

**Reading "unhealthy"** (Targets tab → *Health status details*):
- `Health checks failed with these codes: [5xx]` → the app answered with an error.
- Plain `Health checks failed` (no code) → no answer: timeout or connection refused.
- `Target is in the initial state` → still warming up; wait.

**Troubleshooting ladder for "no answer":**
1. Wait for user data to finish (3–5 minutes after launch).
2. Check the instance security group allows the health-check port from the ALB.
3. Open `http://<instance public IP>` directly. If it does not load, the app is not running.
4. Read the instance's system log for the failed step (e.g. wrong bucket name or an unreplaced placeholder). See [[ami-and-user-data]].
5. A fix to the launch template needs a **new version** and replacing the instances, since existing instances are not changed. See [[launch-templates]].

## Relations

- [[application-load-balancer]] — forwards to the target group
- [[auto-scaling-groups]] — registers new instances here and replaces unhealthy ones
- [[ec2-reachability-checklist]] — the same layers apply when a target is unreachable
- [[security-groups]] — must allow the health-check traffic

## Tags

#aws
