---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# Azure Storage — Permissions and Access

## What it is

The set of mechanisms for authorizing access to an Azure Storage Account: Account Keys, Azure AD / RBAC, SAS tokens and anonymous access. It also covers the three access scenarios for Blob Storage, the difference between data roles and management roles, and Delegated vs Application permissions.

## What problem it solves

A Storage Account holds data that different callers need to reach in different ways — an admin, an application, a user through an app, or a third party with no Azure identity. Each caller needs the right level of access without exposing the master keys, so you must choose the correct authorization method for each scenario.

## Notes

### The 4 ways to authorize access

| Method | When to use | Security |
|--------|-------------|----------|
| **Account Key** | Internal admin — full access, avoid in production | Medium-low |
| **Azure AD / RBAC** | Users and managed identities — the most secure | High |
| **SAS Token** | Delegated, temporary access for third parties | Medium-high |
| **Anonymous access** | Only for explicitly configured public blobs | Very low |

### SAS Token — the most important for the exam

**Temporary and limited** access without exposing the master keys. It defines exactly which resource, which permissions and until when.

#### Three types

| Type | Signed with | Scope |
|------|-------------|---------|
| **Account SAS** | Storage Account Key | One or more services (Blob, Queue, Table, File) |
| **Service SAS** | Storage Account Key | A specific resource of one service |
| **User Delegation SAS** | Azure AD | Blob/Data Lake only — **the most secure** |

A SAS controls: **permissions** (r/w/d/l...), **resource**, **expiration time**, **allowed IPs** and **protocol** (https).

> Exam rule: if the question mentions **Entra ID + RBAC + SAS** → **User Delegation SAS**.

#### User Delegation SAS — works only with Blob

This is an important design limitation for the exam:

| SAS type | Blob | File | Queue | Table |
|-------------|------|------|-------|-------|
| Account SAS | ✅ | ✅ | ✅ | ✅ |
| Service SAS | ✅ | ✅ | ✅ | ✅ |
| **User Delegation SAS** | ✅ | ❌ | ❌ | ❌ |

**Why:** Account SAS and Service SAS are signed with the Storage Account Key, which has access to all services. User Delegation SAS is signed with Entra ID credentials, and Microsoft only implemented that support for Blob Storage (and ADLS Gen2, which is Blob underneath).

> Exam rule: User Delegation SAS + "which service does it protect" → always **Blob**. If the question mentions File, Queue or Table with SAS → Account SAS or Service SAS.

#### SAS token scope — what level it can target

| SAS type | Can target |
|-------------|----------------|
| **Account SAS** | The whole account (Blob + File + Queue + Table) |
| **Service SAS** | Container, a specific blob, file share, queue or table |
| **User Delegation SAS** | Container or a specific blob (Blob only) |

```
Storage Account  ← Account SAS
    └── Container  ← Service SAS / User Delegation SAS
            └── blob.jpg  ← Service SAS / User Delegation SAS
```

#### Stored Access Policy — revoke without regenerating keys

A Service SAS can be linked to a **Stored Access Policy** (defined on the container). This lets you **revoke access before it expires**, without changing the account keys.

- It is associated with the **container**, not with an individual blob or the account
- Maximum **5 policies per container**
- The SAS references the policy by name: `?si=policy-name&sig=...`
- To revoke: delete or modify the policy → every SAS that references it stops working

| Concept | Level |
|----------|-------|
| **SAS token** | Flexible: account, container or individual blob |
| **Stored Access Policy** | Container only (max. 5 per container) |

```
SAS without policy → not revocable until it expires
SAS with policy    → revocable at any time
```

> Exam rule: "revoke a SAS before it expires" → you need a **Stored Access Policy**. "SAP for blobs" → it is associated with the **container**.

### RBAC (Azure AD) — the most secure way

The most secure way. Roles are assigned to users, groups or application identities.

#### Main roles you must know

| Role | Permissions |
|-----|----------|
| `Storage Blob Data Owner` | Full control + ACLs in Data Lake |
| `Storage Blob Data Contributor` | Read, write, delete blobs |
| `Storage Blob Data Reader` | Read-only |
| `Storage Blob Delegator` | Create User Delegation SAS |
| `Storage Queue Data Contributor` | Read, write, delete messages |

#### Data roles vs management roles

Generic Azure roles **do not give access to blob data**. They are management-plane roles (create/delete the account, view configuration).

| Generic role | Equivalent data role | What it allows |
|---|---|---|
| `Owner` | `Storage Blob Data Owner` | Full access + ACLs |
| `Contributor` | `Storage Blob Data Contributor` | Read, write, delete blobs |
| `Reader` | `Storage Blob Data Reader` | Read-only |

> Exam rule: plain `Owner` or `Contributor` → **do not access blob data**.

#### The three access scenarios for Blob Storage

```
Who needs access?
        │
        ├── User accesses directly ────────────→ RBAC on the user
        │
        ├── Azure app/service accesses alone ──→ Managed Identity + RBAC
        │
        └── User accesses through an app ──────→ RBAC + user_impersonation
```

**Scenario 1 — User accesses directly**

The user authenticates with Entra ID and Azure checks their RBAC roles on the container.

```
User (Entra ID) → authenticates → Azure checks their RBAC on the container → applies permissions
```

**Scenario 2 — App/service accesses alone (no user)**

The app has a Managed Identity. An RBAC role is assigned to that identity. There are no explicit credentials in code.

```csharp
// No explicit credential — DefaultAzureCredential detects the MI
var client = new BlobServiceClient(
    new Uri("https://myaccount.blob.core.windows.net"),
    new DefaultAzureCredential()
);
```

**Scenario 3 — User accesses through an app**

The app acts on behalf of the user. The token carries the user's identity. Azure applies the user's RBAC roles, not the app's.

```
User → login in the app → app requests token with user_impersonation
     → app calls Storage with that token
     → Azure sees the user's identity → checks their RBAC
```

It requires the `user_impersonation` permission on the Azure Storage API as a **Delegated** permission in the App Registration.

| Scenario | Mechanism | RBAC assigned to |
|-----------|-----------|-----------------|
| User accesses directly | RBAC | User (Entra ID) |
| App accesses alone | Managed Identity | The app's Managed Identity |
| User accesses through the app | RBAC + `user_impersonation` | User (Entra ID) |

#### Delegated vs Application (in App Registration)

| Type | Who acts | When to use |
|------|-------------|---------------|
| **Delegated** | The app acts on behalf of the signed-in user | A user is logging in |
| **Application** | The app acts with its own identity, no user | Daemons, background services |

| API | Permission | Type | What for |
|-----|---------|------|----------|
| Microsoft Graph | `User.Read` | Delegated | Read the signed-in user's profile |
| Azure Storage | `user_impersonation` | Delegated | Access the blob as the user |

> Exam rule: "roles assigned to the users" → **Delegated**. No user → **Application**.

### Storage Account Keys (Shared Key)

The account has **two master keys** (key1, key2). With them you have full access to the whole Storage Account.

```
DefaultEndpointsProtocol=https;AccountName=...;AccountKey=<key>
```

- Equivalent to the root password — full access to all containers and services
- If a key is compromised → the whole account is exposed
- Rotate it with key2 without downtime: first migrate to key2, then regenerate key1

### Anonymous (public) access

No authentication — anyone with the URL can read the content.

```
https://myaccount.blob.core.windows.net/container1/photo.jpg
```

- Configured at the container level
- Read-only, Blob Storage only
- Not recommended for sensitive data

### Security priority rule

```
User Delegation SAS > RBAC > Service SAS > Account SAS > Account Key
```

> For the exam: if you are asked "what is the most secure way to give temporary access" → **User Delegation SAS**.

### Relationship between the modes

```
Entra ID + RBAC ──────────────────→ direct access (most secure)
       │
       └──→ User Delegation SAS ──→ delegated access with token

Storage Account Keys ─────────────→ direct access (high risk)
       │
       └──→ Account SAS / Service SAS → delegated access with token

No auth ──────────────────────────→ public access (read-only)
```

### AZ-204 exam keys

- Entra ID + RBAC + SAS → **User Delegation SAS**; it only works with **Blob** (not File, Queue or Table).
- Revoke a SAS before it expires → **Stored Access Policy** (container level, max. 5 per container).
- Plain `Owner` / `Contributor` do **not** access blob data — you need the `Storage Blob Data *` roles.
- User accesses through an app → RBAC + `user_impersonation` (Delegated).
- "Roles assigned to the users" → Delegated; no user → Application.
- Most secure way to give temporary access → User Delegation SAS.

## Relations

- [[microsoft-entra-id]] — the identity model (Delegated vs Application, Managed Identity, SAS types) these scenarios build on
- [[azure-blob-storage]] — the storage service whose access is being controlled
- [[azure-rbac]] — general RBAC model (scopes, roles, data vs management plane)
- [[azcopy]] — uses Azure AD or SAS to authenticate against Storage

## Tags

#azure #az204
