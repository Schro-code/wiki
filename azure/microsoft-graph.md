---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# Microsoft Graph

## What it is

Microsoft Graph is the **single entry point** to all the data and services of the Microsoft 365 and Entra ID ecosystem. A single REST endpoint that unifies what used to be dozens of separate APIs.

## What problem it solves

Microsoft 365 has many services: Outlook, Teams, SharePoint, OneDrive, Entra ID (users, groups)… Before, each had its own API with its own authentication and format. Graph unifies them:

```
Your app → graph.microsoft.com → Entra ID / Teams / Outlook / SharePoint / OneDrive
```

## Notes

### How it is used — two layers

#### Layer 1: direct REST API (with OData)

The base endpoint is:

```
https://graph.microsoft.com/v1.0/
```

Calls are standard HTTP. Query parameters follow the **OData** standard — hence the `$`:

| Parameter | What it does | Example |
|---|---|---|
| `$filter` | Filters results (like WHERE) | `$filter=surname eq 'Bob'` |
| `$select` | Chooses which fields to return | `$select=displayName,mail` |
| `$top` | Limits the number of results | `$top=10` |
| `$orderby` | Sorts results | `$orderby=surname desc` |
| `$expand` | Includes nested relationships | `$expand=manager` |

**Full example:** get only the displayName of users whose surname is Bob:

```
GET https://graph.microsoft.com/v1.0/users?$filter=surname eq 'Bob'&$select=displayName
```

In a real URL, spaces are encoded as `%20`:

```
https://graph.microsoft.com/v1.0/users?$filter=surname%20eq%20'Bob'&$select=displayName
```

> `eq` is the OData equality operator. There is no `equals()`. Parameters are separated with `&`, not `&&`.

The call requires a Bearer token in the header — it doesn't work in the browser without prior authentication. To try it without code: **Graph Explorer** at `developer.microsoft.com/graph/graph-explorer`.

#### Layer 2: SDK (Python / C#)

The SDK is a typed wrapper over the same REST API. Internally it makes exactly the same HTTP call.

**Python:**

```python
from msgraph import GraphServiceClient
from msgraph.generated.users.users_request_builder import UsersRequestBuilder
from kiota_abstractions.base_request_configuration import RequestConfiguration

query_params = UsersRequestBuilder.UsersRequestBuilderGetQueryParameters(
    filter="surname eq 'Bob'",
    select=["displayName"]
)
config = RequestConfiguration(query_parameters=query_params)
users = await client.users.get(request_configuration=config)
```

**C#:**

```csharp
var users = await graphClient.Users.GetAsync(config =>
{
    config.QueryParameters.Filter = "surname eq 'Bob'";
    config.QueryParameters.Select = new[] { "displayName" };
});
```

### Authentication

Every call to Graph requires an OAuth 2.0 token. It is obtained through **MSAL** (Microsoft Authentication Library) using the credentials of an **App Registration** in Entra ID.

See [[microsoft-entra-id]] for the complete identity map.

### AZ-204 exam keys

- `$select` returns specific fields — there is no `$only`
- The filter operator is `eq`, not `equals()`
- OData parameters are separated with `&`, not `&&`
- No token, no call — Graph Explorer handles it automatically for testing

## Relations

- [[microsoft-entra-id]] — App Registrations and the identity model behind Graph tokens

## Tags

#azure #az204
