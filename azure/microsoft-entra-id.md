---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# Microsoft Entra ID

## What it is

Microsoft Entra ID (formerly Azure Active Directory / Azure AD) is Microsoft's **identity directory**: the central system that knows who you are and what you have access to. This page is the complete mental map of identity in Azure: Entra ID, Tenant, Service Principals, App Registration, Managed Identity and the Microsoft Identity Platform.

## What problem it solves

Azure services need to authenticate to each other, and users need to authenticate to applications. Without a centralized system, every service would manage its own credentials separately. Entra ID is the single place where identities live.

## Notes

### The mental map: three layers

The confusion around identity concepts comes from mixing different layers. They are not alternatives — they are levels built on top of one another:

```
┌──────────────────────────────────────────────────────────────┐
│                        ENTRA ID                              │
│           (the directory — formerly called "Azure AD")       │
│                                                              │
│   Your organization = a TENANT (your instance of the         │
│   directory)                                                 │
│                                                              │
│   Inside the tenant live the IDENTITIES:                     │
│   ┌──────────┐  ┌─────────────────────────────────────────┐  │
│   │  Users   │  │         Service Principals              │  │
│   │ (humans) │  │   (apps, services — non-human)          │  │
│   └──────────┘  │  ┌───────────────┐  ┌───────────────┐  │  │
│                 │  │ App Registr.  │  │   Managed     │  │  │
│                 │  │ (you create   │  │   Identity    │  │  │
│                 │  │  it)          │  │ (Azure creates│  │  │
│                 │  │               │  │  it itself)   │  │  │
│                 │  └───────────────┘  └───────────────┘  │  │
│                 └─────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
        ↑
Microsoft Identity Platform
(the protocols and SDKs developers use to reach Entra ID:
 OAuth 2.0, OpenID Connect, MSAL)
```

### Layer 1 — Entra ID and Tenant

**Entra ID** is Microsoft's identity service. It used to be called Azure Active Directory (Azure AD). It is the directory where all identities live.

**Tenant** = your organization inside Entra ID. When a company signs up for Azure, it gets its own tenant. It is your "private instance" of Microsoft's global directory.

> Every tenant has a unique ID (GUID). In the exam it appears as `tenant-id` or `directory-id`.

### Layer 2 — Service Principals: App Registration vs Managed Identity

**Service Principals** are the core of the confusion — there are two ways to create them.

#### App Registration

You create it. You register your application in Entra ID → Azure creates a Service Principal for it. **You manage the secret or certificate.**

*Use case:* a web or mobile app that needs to authenticate users, or an external app (outside Azure) that needs to access Azure resources.

#### Managed Identity

Azure creates it. It is a special Service Principal that Azure fully manages: it creates the secret, rotates it automatically, and destroys it when the resource is deleted. **You never see the credential.**

*Use case:* your Function needs to read from Service Bus, your App Service needs to write to Cosmos DB — without storing secrets in configuration.

```
App Registration = you say "my app exists"         → you manage the secret
Managed Identity = Azure says "this resource is"   → Azure manages the secret
```

#### The two types of Managed Identity

**System-assigned** — the identity lives and dies with the resource. If you delete the Function, it disappears. One resource = one identity.

**User-assigned** — you create the identity as an independent resource and assign it to several services. Useful when several Functions need the same permissions: you manage a single identity.

| | System-assigned | User-assigned |
|---|---|---|
| **Lifecycle** | Tied to the resource | Independent |
| **Reusable** | No | Yes |
| **When to use** | One service → one resource | Several services with the same permissions |

### Layer 3 — Microsoft Identity Platform

This is the **developer layer**. It is not a new service; it is the set of:

- **Protocols:** OAuth 2.0, OpenID Connect
- **SDKs:** MSAL (Microsoft Authentication Library)
- **Endpoints:** `login.microsoftonline.com/{tenant-id}/...`

It is what you use in code when you want a user to log in or when your app needs to request an access token. It is the "API" on top of Entra ID.

#### Open standards

**OAuth 2.0 does NOT identify the user.** That is the most frequent misunderstanding, and it also shows up in the exam.

| Question | Protocol |
|---|---|
| Can this app access my photos? | **OAuth 2.0** — authorization |
| Who is this user? | **OpenID Connect** — authentication |

##### OAuth 2.0

**What it is for:** letting a user authorize an app to access a resource on their behalf, without giving it their password.

**Actors:**
- **Resource Owner**: the user
- **Client**: the app that wants access
- **Authorization Server**: whoever issues tokens (in Azure: `login.microsoftonline.com`)
- **Resource Server**: the API that protects the data

**Basic flow (Authorization Code):**

```
User → App → "Sign in with Microsoft"
App → Authorization Server → "Give me a code for this user"
User authenticates with Microsoft
Microsoft → App → code
App → Microsoft → "Here is the code, give me an access token"
Microsoft → App → access token
App → API → "Here is my access token"
API → verifies the token → returns data
```

**JWT in OAuth:** Azure uses JWT for access tokens. It has three parts: `header.payload.signature`

```json
// Decoded payload of an Azure access token
{
  "aud": "https://graph.microsoft.com",   // who it is for
  "iss": "https://login.microsoftonline.com/...",
  "exp": 1753900000,                       // when it expires
  "scp": "User.Read Mail.Send",            // granted permissions
  "oid": "abc-123..."                      // user ID (but this is already OIDC)
}
```

##### OpenID Connect (OIDC)

**What it is for:** adds authentication on top of OAuth. It answers "who is this user?"

**How it works:** it is OAuth 2.0 + an extra **ID Token**. The key is the `openid` scope.

| Token | Contains | For whom |
|---|---|---|
| **Access Token** | Permissions (scopes) | The API |
| **ID Token** | User identity | The client app |

**Flow:**

```
User → App → "I want to sign in"

App → Authorization Server →
  "Give me a code for this user"
  scope=openid profile email          ← this is what activates OIDC

User authenticates with Microsoft

Microsoft → App → code

App → Microsoft → "Here is the code, give me tokens"

Microsoft → App → access token + ID token   ← this is the difference from pure OAuth

App reads the ID token → knows who the user is
App uses the access token → calls APIs on their behalf
```

The ID Token (always a JWT) contains **claims** about the user:

```json
{
  "sub": "abc123",           // unique user ID
  "name": "Jane Doe",
  "email": "jane@example.com",
  "iss": "https://login.microsoftonline.com/...",
  "aud": "my-client-id",
  "exp": 1753900000
}
```

> Summary: OIDC = OAuth 2.0 + ID Token + `/userinfo` endpoint

##### SAML 2.0

**What it is for:** the same as OIDC (identity federation), but older and based on **XML** instead of JSON.

**Typical in:** legacy enterprise systems, corporate SSO.

**Flow:**

```
User → App (Service Provider)
App → Identity Provider (Entra ID)
User authenticates
IdP → sends a signed XML (SAML Assertion) → App
App → verifies the signature → user authenticated
```

The **SAML Assertion** is the equivalent of the OIDC ID Token, but in XML:

```xml
<saml:Assertion>
  <saml:AttributeStatement>
    <saml:Attribute Name="email">
      <saml:AttributeValue>jane@example.com</saml:AttributeValue>
    </saml:Attribute>
  </saml:AttributeStatement>
</saml:Assertion>
```

##### Comparison for the exam

| | OAuth 2.0 | OpenID Connect | SAML 2.0 |
|---|---|---|---|
| **Purpose** | Authorization | Authentication | Authentication |
| **Token format** | JWT (usually) | JWT mandatory | XML |
| **Built on** | — | OAuth 2.0 | Its own |
| **Use case** | API access | Modern login | Legacy/enterprise SSO |
| **In Azure** | Access to Graph, own APIs | Sign in with Microsoft | Old enterprise apps |

### API Permissions in App Registration

When you register an app, you have to declare which external APIs it needs to access. This is configured under **API Permissions** inside the App Registration.

#### The two permission types

| Type | Who acts | When to use |
|------|----------|-------------|
| **Delegated** | The app acts **on behalf of the signed-in user**. The effective permissions are the user's. | A user is logging in — RBAC assigned to users |
| **Application** | The app acts **with its own identity**, with no user. | Daemons, background services, no user login |

> Exam rule: if the scenario says "roles assigned to the users" → **Delegated**. If there is no user → **Application**.

#### The two most important APIs for the exam

| API | What it is for | Specific permission | Type |
|-----|----------------|---------------------|------|
| **Microsoft Graph** | Read/write user data (profile, email, groups...) | `User.Read` | Delegated |
| **Azure Storage** | Access Blob Storage on behalf of the user (their RBAC applies) | `user_impersonation` | Delegated |

These are two different APIs — when an app needs both, it registers the two permissions separately.

#### Most common Microsoft Graph permissions

| Permission | What it allows |
|---------|-------------|
| `User.Read` | Read the profile of the signed-in user |
| `User.ReadWrite` | Read and write your own profile |
| `User.ReadWrite.All` | Read and write the profile of any user in the tenant |

### Client → APP and APP → APP: Delegated vs Application

Both permission types answer one simple question: **is there a user in the flow?**

| | Delegated | Application |
|---|---|---|
| **OAuth flow** | Authorization Code / PKCE | Client Credentials |
| **User present** | Yes | No |
| **Token represents** | "user X acting through app Y" | "app Z acting on its own" |
| **Claim in the token** | `scp` — scopes the user granted | `roles` — application roles |

```json
// Delegated token
{ "scp": "User.Read Mail.Send", "oid": "user-id" }

// Application token
{ "roles": ["User.ReadWrite.All"], "oid": "service-principal-id" }
```

#### Case 1 — User → App → Resource (Delegated)

The user authenticates and the app acts on their behalf. The effective permissions are the user's, not the app's.

```
User logs in → Authorization Code flow → access token (with scp)
App → Storage → "I want to read blob X" + token
Storage → checks that the user has RBAC on that blob → OK
```

Typical in: web apps, SPAs, mobile apps where the user is present.

#### Case 2 — App → Service, no user (Application / Client Credentials)

There is no user. The app authenticates with its own identity and acts on its own.

**Client Credentials flow:**

```
App → Entra ID →
  "I am client_id X, my secret is Y, give me a token to call this API"

Entra ID → App → access token

App → API → "Here is my access token"
API → verifies → responds
```

No redirect, no user, no ID Token.

##### Who is the "app"

| Situation | Identity |
|---|---|
| App running **inside Azure** (App Service, Functions, VM...) | **Managed Identity** — Azure manages the secret |
| App running **outside Azure** (local, on-prem, another cloud) | **App Registration** — you manage client_id + secret |

Managed Identity also uses Client Credentials internally, but Azure makes it invisible.

##### RBAC — how you grant access

Having an identity is not enough. The identity needs a **role** on the target resource.

```
Target resource → IAM → Add role assignment
  → Role: "Storage Blob Data Reader"
  → Assign to: your Managed Identity or Service Principal
```

| Role | Resource | What it allows |
|---|---|---|
| `Storage Blob Data Reader` | Storage | Read blobs |
| `Storage Blob Data Contributor` | Storage | Read and write blobs |
| `Key Vault Secrets User` | Key Vault | Read secrets |
| `Azure Service Bus Data Receiver` | Service Bus | Consume messages |
| `Azure Service Bus Data Sender` | Service Bus | Send messages |

##### DefaultAzureCredential — the code pattern

An SDK abstraction that tries methods in order until one works:

```
1. Environment variables (AZURE_CLIENT_ID, AZURE_CLIENT_SECRET...)
2. Workload Identity
3. Managed Identity     ← in production Azure uses this one
4. Visual Studio
5. Azure CLI            ← in local dev it uses this one
6. Azure PowerShell
7. Interactive browser
```

The same code works locally and in production without changing anything:

```csharp
var credential = new DefaultAzureCredential();

var secretClient = new SecretClient(new Uri("https://my-vault.vault.azure.net/"), credential);
var blobClient = new BlobServiceClient(new Uri("https://account.blob.core.windows.net/"), credential);
var busClient = new ServiceBusClient("namespace.servicebus.windows.net", credential);
```

##### Complete pattern for the exam

```
1. Your resource (App Service, Function...) → enable Managed Identity
2. Target resource (Storage, Key Vault, Service Bus...) → IAM → assign role to the MI
3. In code → new DefaultAzureCredential() → the SDK does the rest
```

### Alternatives to Entra ID

Not every access to Azure resources requires an identity. There is a second family based on keys and static credentials — no Service Principal, no RBAC.

| Mechanism             | How it works                                              | Where it applies                            |
| --------------------- | --------------------------------------------------------- | ------------------------------------------- |
| **Connection string** | String with account + key embedded in configuration       | Storage, Service Bus, Event Hubs, Cosmos DB |
| **Access keys**       | Primary/secondary key of the storage account              | Storage Account                             |
| **SAS token**         | Signed URL with permissions and an expiration time        | Blob Storage                                |
| **API key**           | Fixed key to authenticate HTTP requests                   | APIM, Cognitive Services                    |
| **Admin account**     | Fixed username/password tied to the resource              | Azure Container Registry                    |

> Exam rule: if the question mentions security, least privilege or avoiding hardcoded credentials → Entra ID + Managed Identity. Keys are for legacy scenarios or compatibility.

### SAS (Shared Access Signatures)

SAS **is not OAuth or Entra ID**. It is a completely different mechanism for granting access to Azure Storage.

```
Access to Azure resources
│
├── VIA ENTRA ID (OAuth / OIDC)
│   ├── With user      → Delegated → Authorization Code → access token
│   └── Without user   → Application → Client Credentials → access token
│                         RBAC decides what you can do on the resource
│
└── VIA KEY / SIGNATURE (no Entra ID)
    ├── Access keys        → master key of the Storage account
    ├── Connection strings → the key embedded as text
    └── SAS token          → signed URL with limited permissions and expiration
```

SAS exists for one specific case: **giving someone temporary access without giving them an Entra ID identity or an RBAC role.**

> A customer needs to download a PDF contract. They have no Azure account. You don't want to give them permanent access. You generate a SAS URL that expires in 1 hour.

```
https://account.blob.core.windows.net/contracts/doc.pdf?sv=...&se=2026-07-30T12:00Z&sig=...
```

That URL already carries the permissions inside it, cryptographically signed. Storage verifies it without any Entra ID token.

#### The three SAS types

| Type | Signed with | Scope |
|---|---|---|
| **Account SAS** | Account key | Multiple services/containers |
| **Service SAS** | Account key | One specific resource |
| **User Delegation SAS** | Entra ID credentials | Blob Storage only — the most secure |

#### User Delegation SAS — the exception

The only SAS that touches Entra ID. Instead of being signed with the master key, it is signed with the credentials of an Entra ID user.

```
You (with your Entra ID identity) → ask Storage to sign a SAS on your behalf
Storage → signs the URL with your credentials → SAS with your context
```

> Exam rule: if the question says "without exposing the account key" → **User Delegation SAS**.

#### Stored Access Policies

They let you **revoke a SAS without regenerating the key**. They are defined on the container and the SAS references the policy by name. Very useful for controlling the lifecycle.

Without a Stored Access Policy: to revoke a SAS you have to rotate the account key (affects everyone).
With a Stored Access Policy: you delete the policy and every SAS that references it becomes invalid.

### AZ-204 exam keys

- OAuth 2.0 = authorization; OpenID Connect = authentication (adds the ID Token via the `openid` scope); SAML 2.0 = XML-based legacy SSO.
- "Roles assigned to the users" → **Delegated**; no user → **Application** (Client Credentials).
- Inside Azure → **Managed Identity**; outside Azure → **App Registration**.
- Security / least privilege / no hardcoded credentials → Entra ID + Managed Identity.
- "Without exposing the account key" → **User Delegation SAS**.
- Revoke a SAS before it expires → **Stored Access Policy**.
- Pattern: enable MI on the resource → assign a role on the target → `new DefaultAzureCredential()`.

## Relations

- [[azure-rbac]] — what an authenticated identity is allowed to do (Entra ID answers "who are you?", RBAC "what can you do?")
- [[azure-storage-access-and-permissions]] — the storage-specific access scenarios built on these identity concepts
- [[azure-blob-storage]] — the resource most often accessed with Delegated, Managed Identity and SAS patterns
- [[microsoft-graph]] — the API reached with the `User.Read` Delegated permission and MSAL tokens
- [[azure-functions]] — typical consumer of Managed Identity to reach other services
- [[azure-service-bus]] — accessed via Managed Identity and the Data Receiver/Sender roles
- [[azure-cosmos-db]] — accessed from App Service/Functions via Managed Identity without stored secrets

## Tags

#azure #az204
