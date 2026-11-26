---
domain: aws
tags: [aws]
sources: []
updated: 2026-11-26
---

# AMI and User Data

## What it is

An **AMI (Amazon Machine Image)** is a template of the disk an instance boots from (OS plus whatever is installed). **User data** is a script that runs once when the instance first boots.

## What problem it solves

Servers should be reproducible, not hand-crafted. Both are ways to get a new [[amazon-ec2|instance]] into the same configured state without doing it manually.

## Notes

| | Custom AMI | User data |
|---|---|---|
| Idea | A finished **photo** of a configured server | A **recipe** run at boot |
| Boot speed | Fast, everything is already installed | Slower, installs on every boot |
| Flexibility | Must be rebuilt to change | Edit the script and relaunch |

- User data runs with the instance's network and role. If it downloads from S3 or installs packages, the instance needs outbound access and an IAM role with permission.
- If a user data script fails (e.g. a placeholder value was not replaced, or a bucket is wrong), the app silently never starts. The system log shows why. Instance status checks will still say "passed".
- The labs used the stock Amazon Linux 2023 AMI plus user data (install nvm, Node 20, download `app.zip` from S3, `npm install`, `npm start`).

## Relations

- [[amazon-ec2]] — the instance that boots from the AMI
- [[launch-templates]] — store the AMI choice and user data together
- [[auto-scaling-groups]] — rely on that recipe to create identical instances

## Tags

#aws
