---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# Azure RBAC

## What it is

RBAC (Role-Based Access Control) controls **what an identity can do** on an Azure resource, once it is already authenticated. Entra ID answers "who are you?"; RBAC answers "what can you do?".

## What problem it solves

Authentication alone doesn't say what an identity may do. RBAC grants permissions through a uniform model (identity + role + scope) so access can be as narrow as the need requires and inherited predictably down the resource hierarchy.

## Notes

### The three ingredients of an RBAC assignment

Every permission assignment in Azure has exactly three parts:

```
WHO?           WHAT CAN THEY DO?      ON WHAT?
Identity   +       Role            +    Scope
```

| Ingredient | What it is | Examples |
|---|---|---|
| **Identity** | User, group, Service Principal or Managed Identity | Jane Doe, support-team, my-function-app |
| **Role** | Set of allowed actions | Reader, Contributor, Owner, or a custom role |
| **Scope** | The resource or container the assignment applies to | Subscription, resource group, a specific resource |

### Scopes: from largest to smallest

The scope defines how far the permission reaches. Permissions are **inherited downward** — if you assign a role at the subscription, it applies to all resource groups and resources inside it.

```
Management Group
    └── Subscription
            └── Resource Group
                    └── Individual resource
                            └── Sub-resource (blob, queue, table...)
```

**Rule:** always assign at the narrowest scope that covers the need. If the team only needs to see one specific app, don't give them access to the whole subscription.

#### In the portal (IAM)

To assign a role: go to the resource (or subscription, or resource group) → **Access Control (IAM)** → Add role assignment → choose role and identity.

### Action syntax

Every RBAC action follows this format:

```
{provider}/{resource type}/{action}
```

The `*` is a wildcard meaning "everything".

| Action | Meaning |
|---|---|
| `*/read` | Read **any** resource type from **any** provider |
| `Microsoft.Compute/*/read` | Read any Compute resource (VMs, disks, etc.) |
| `Microsoft.Storage/storageAccounts/*` | Any action on Storage Accounts |
| `Microsoft.Insights/diagnosticSettings/*/read` | Only read diagnostic settings |
| `Microsoft.Support/*` | Any action in the support service (includes write) |

> The `read` action never implies write. The `*` wildcard as an action does include write.

### Azure built-in roles

Azure has more than 100 built-in roles. The four fundamental ones for the exam:

| Role | Actions it includes | When to use it |
|---|---|---|
| **Owner** | `*` (everything, including managing permissions) | Full administrator — with the power to delegate access |
| **Contributor** | `*` except managing permissions | Can create and manage resources, but not change who has access |
| **Reader** | `*/read` | Read-only on all resources. Monitoring, support |
| **User Access Administrator** | Only manage RBAC assignments | To delegate access management without giving power over resources |

#### Reader = `*/read`

The **Reader** role is exactly the `*/read` action applied to the scope. It is the right role when:
- A support team needs to see logs and configuration
- Monitoring without the ability to change anything
- Resource auditing

### Data roles vs management roles

This distinction is critical and a source of confusion in the exam:

| Type | Controls | Examples |
|---|---|---|
| **Management roles** (control plane) | Create, delete and configure the resource | Reader, Contributor, Owner |
| **Data roles** (data plane) | Read/write the content inside the resource | Storage Blob Data Reader, Service Bus Data Receiver |

**The Reader role does NOT give access to a blob's data.** It only lets you see that the Storage Account exists and its configuration. To read a blob's content you need `Storage Blob Data Reader`.

```
Reader            → can see the Storage account in the portal
                  → CANNOT download a file from the blob

Storage Blob      → can download files from the blob
Data Reader       → CANNOT delete or create the account
```

See [[azure-storage-access-and-permissions]] for the complete Blob access scenarios.

### Custom roles

If no built-in role fits, you can create a custom role by defining exactly the allowed actions:

```json
{
  "Name": "VM Operator",
  "Actions": [
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/read"
  ],
  "NotActions": [],
  "AssignableScopes": ["/subscriptions/{subscription-id}"]
}
```

- `Actions` — what it can do
- `NotActions` — exceptions within Actions (subtracted)
- `AssignableScopes` — at which scopes this role can be assigned

> `NotActions` is not an explicit deny — it is a subtraction. If another role grants that action, the permission still applies. For explicit denials there are **Deny Assignments** (only assignable by Azure Blueprints/Policies, not manually).

### How a request works internally

When an identity tries to do something in Azure, ARM (Azure Resource Manager) evaluates in this order:

```
1. Does any Deny Assignment block this action? → Denied
2. Does any role assignment grant this action at this scope or a higher one? → Allowed
3. → Denied by default
```

### AZ-204 exam keys

| Question | Answer |
|---|---|
| Team that only needs to see resources | **Reader** (`*/read`) |
| App that needs to write to a blob | **Storage Blob Data Contributor** (not Contributor) |
| Service that needs to read from Service Bus | **Azure Service Bus Data Receiver** (not Reader) |
| Someone who needs to manage who has access | **User Access Administrator** or **Owner** |
| Read-only permission on Compute | `Microsoft.Compute/*/read` |
| Reader on the subscription vs on a resource | Inheritance: subscription applies to everything it contains |

## Relations

- [[microsoft-entra-id]] — answers "who are you?"; RBAC answers what that identity may do
- [[azure-storage-access-and-permissions]] — data-plane roles applied to Blob Storage
- [[azcopy]] — requires data roles when authenticating with Azure AD

## Tags

#azure #az204
