---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# Azure Logic Apps

## What it is

A **workflow engine** based on triggers and actions. You define a sequence of steps and Azure executes them in order every time the trigger fires.

```
Trigger (what starts it?)
   └── Action 1
       └── Action 2
           └── Action 3
```

| Concept | What it is |
|---|---|
| **Trigger** | Event that starts the flow (HTTP, Service Bus, timer, blob...) |
| **Action** | A step in the flow (call an API, save to a DB, send an email...) |
| **Connector** | Pre-built wrapper for an external service — already has the auth and the format |
| **Workflow** | The complete flow, defined internally as JSON |

## What problem it solves

You have a process that connects several services in sequence — an event arrives, you call an external API, save data in a database, send an email. You could do it with [[azure-functions]], but you would have to write all the plumbing code yourself: authentication with each service, retries, serialization, HTTP clients.

**Logic Apps** solves this by giving you that flow visually, without that infrastructure code:
- 400+ pre-built connectors (Office 365, SQL, Salesforce, Teams, SendGrid...)
- Authentication managed by Azure
- Retries and error handling that are configured, not programmed

```
Problem:
  Connect 4 services → write 4 clients with their own auth, retries and serialization

With Logic Apps:
  Configure 4 steps → Azure manages the connection and resilience of each one
```

## Notes

### How it fits with related services

| Service | Role | When to use it |
|---|---|---|
| [[azure-event-grid]] | Routes events | You only need to notify that something happened |
| [[azure-service-bus]] | Message queue | You need to decouple producer and consumer |
| [[azure-functions]] | Runs code | Complex logic, transformations, calculations |
| **Logic Apps** | Orchestrates steps | Connect several services in sequence without infrastructure code |

**Collaboration example:**

```
Service Bus (a message arrives)
   └── Logic App (orchestrates the flow)
       ├── Calls an external API
       ├── Saves the result in Cosmos DB
       └── Notifies via Teams
```

### Hosting plans

| Plan | Description |
|---|---|
| **Consumption** | Serverless. Pay per execution. No VNET. |
| **Standard** | Dedicated plan. Supports private VNET integration. More control. |

### AZ-204 exam keys

- Logic Apps is **low-code**: flows in a visual designer, JSON internally
- It differs from Azure Functions in that you don't write the connection logic — you use connectors
- It can use a Function as an action within its flow
- **Consumption** → serverless, pay per execution; **Standard** → dedicated plan with VNET support
- The trigger can be HTTP, Service Bus, Event Grid, timer, blob, among others

## Relations

- [[azure-functions]] — code-first alternative; can be called as an action inside a workflow
- [[azure-event-grid]] — event routing; can be a trigger
- [[azure-service-bus]] — messaging broker; can be a trigger

## Tags

#azure #az204
