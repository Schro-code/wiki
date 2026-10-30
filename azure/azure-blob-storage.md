---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# Azure Blob Storage

## What it is

Azure's object storage service: scalable, durable and reachable from anywhere. It stores files, images, backups, logs and any other unstructured data. This page covers account and blob types, the four access tiers, the .NET SDK, properties and metadata, lifecycle management, data protection and immutable storage.

## What problem it solves

Storing files, images, backups, logs and any unstructured data in the cloud without managing servers.

## Notes

### Storage Account types

| Type | What for | Supported blobs |
|------|----------|-----------------|
| **Standard GPv2** | General purpose, the most common | Block, Append, Page |
| **Premium Block Blobs** | Low latency, high throughput | Block only |
| **Premium Page Blobs** | VM VHD disks | Page only |
| **Premium File Shares** | Azure Files with low latency | Not applicable |

> For the exam: if it mentions "low latency" or "high performance" on blobs → **Premium Block Blobs**.

### Blob types

| Type | Characteristics | Use case |
|------|----------------|----------|
| **Block blob** | Made of individual blocks. Optimized for parallel uploads. Up to 190.7 TiB. | Files, images, videos, backups |
| **Append blob** | Only allows appending blocks at the end. A block in the middle cannot be modified or deleted. | Logs, event streams |
| **Page blob** | Random access to 512-byte pages. Optimized for random reads/writes. | Azure VM VHD disks |

> Exam rule: "logs that are only written at the end" → **Append blob**. "VM disk" → **Page blob**. Any other case → **Block blob**.

### Access Tiers

Blob Storage offers four tiers for storing data according to how often it is accessed. More access = higher storage cost but lower read cost. Less access = lower storage cost but higher read cost.

| Tier | Type | What for | Minimum retention |
|------|------|----------|---------------------|
| **Hot** | Online | Data that is read or modified frequently. Highest storage cost, lowest access cost. | No minimum |
| **Cool** | Online | Infrequently accessed data. Lower storage cost than Hot, higher access cost. | **30 days** |
| **Cold** | Online | Rarely accessed data that still needs fast retrieval. Cheaper than Cool. | **90 days** |
| **Archive** | Offline | Rarely accessed data with tolerable latency (hours). The cheapest to store. | **180 days** |

> Archive is **offline**: before reading the blob you have to "rehydrate" it to Hot or Cool — it can take hours.

#### Rule for the exam

| Scenario | Tier |
|-----------|------|
| Images of an active website | Hot |
| Monthly backups that are rarely consulted | Cool |
| Audit logs that are almost never read | Cold |
| Historical files / long-term compliance | Archive |

If the question mentions **latency in hours** or **minimum storage cost** → Archive.
If it mentions **frequent access** → Hot.

### Azure Blob Storage hierarchy

Azure Blob Storage has three nested levels:

```
Storage Account
└── Container  (like a folder / bucket)
    └── Blob   (the file itself: image, PDF, .txt...)
```

| Level | Description |
|-------|-------------|
| **Storage Account** | The root resource. Groups all containers and defines the region, redundancy and tier. |
| **Container** | Groups blobs. Equivalent to a folder or bucket. |
| **Blob** | The actual stored file. |

### SDK classes (.NET)

| Class | What it points to | When to use it |
|-------|-------------------|---------------|
| `BlobServiceClient` | The whole Storage Account | List containers, create/delete containers |
| `BlobContainerClient` | A specific container | List blobs, upload files to the container |
| `BlobClient` | A specific blob (the file) | Download, upload, delete or change properties of that file |

#### Essential operations

```csharp
// Get clients
BlobServiceClient service = new BlobServiceClient(connectionString);
BlobContainerClient container = service.GetBlobContainerClient("my-container");
BlobClient blob = container.GetBlobClient("file.txt");

// Create container
await container.CreateIfNotExistsAsync();

// Upload
await blob.UploadAsync(stream, overwrite: true);

// Download
BlobDownloadInfo download = await blob.DownloadAsync();

// List blobs in a container
await foreach (BlobItem item in container.GetBlobsAsync())
{
    Console.WriteLine(item.Name);
}

// Delete
await blob.DeleteAsync();
```

#### Changing HTTP properties

```csharp
await blobClient.SetHttpHeadersAsync(new BlobHttpHeaders
{
    ContentType = "text/plain",
    ContentLanguage = "en-us"
});
```

`SetHttpHeadersAsync` modifies the blob's HTTP properties without touching the file content.

### SDK — detailed reference by operation

#### Instantiation pattern

```csharp
var serviceClient = new BlobServiceClient(connectionString);
var containerClient = serviceClient.GetBlobContainerClient("my-container");
var blobClient = containerClient.GetBlobClient("my-blob.txt");
```

#### Containers

```csharp
await containerClient.CreateIfNotExistsAsync();
await containerClient.DeleteAsync();

// List all containers in the account
await foreach (var container in serviceClient.GetBlobContainersAsync())
    Console.WriteLine(container.Name);

// List blobs inside a container
await foreach (BlobItem blob in containerClient.GetBlobsAsync())
    Console.WriteLine(blob.Name);
```

#### Upload — three variants

```csharp
// From a Stream (the most common)
await blobClient.UploadAsync(stream, overwrite: true);

// With options: ContentType and metadata in the same Upload
var options = new BlobUploadOptions
{
    HttpHeaders = new BlobHttpHeaders { ContentType = "application/json" },
    Metadata = new Dictionary<string, string> { { "author", "jdoe" } }
};
await blobClient.UploadAsync(stream, options);

// Directly from a file path
await blobClient.UploadAsync("/path/to/file.txt", overwrite: true);
```

> `UploadAsync` without `overwrite: true` throws an exception if the blob already exists.

#### Download — three variants

All three download the same content; they differ in **where they leave the result**:

| Method | Where it leaves the result | When to use it |
|--------|---------------------------|---------------|
| `DownloadAsync` | Gives you the raw Stream — you handle it | Full control, streaming |
| `DownloadToAsync` | Writes directly to a Stream or file | Large files |
| `DownloadContentAsync` | Everything in memory as `BinaryData` | JSON, small text |

```csharp
// 1. DownloadAsync — raw stream
BlobDownloadInfo download = await blobClient.DownloadAsync();
await download.Content.CopyToAsync(myStream);

// 2. DownloadToAsync — to a file or an existing Stream
await blobClient.DownloadToAsync("/path/destination.txt");
await blobClient.DownloadToAsync(myStream);

// 3. DownloadContentAsync — everything in RAM as BinaryData
BlobDownloadResult result = await blobClient.DownloadContentAsync();
string text         = result.Content.ToString();
byte[] bytes        = result.Content.ToArray();
MyClass obj         = result.Content.ToObjectFromJson<MyClass>();
```

> **Stream** = a flow of data processed in chunks without loading everything into memory. Useful for large files. `DownloadContentAsync` does not use a Stream: it loads everything into RAM at once.

#### Delete

```csharp
// Throws an exception if the blob does not exist
await blobClient.DeleteAsync();

// Safe: does not fail if it does not exist
await blobClient.DeleteIfExistsAsync();

// If the blob has snapshots, you must say what to do with them
await blobClient.DeleteAsync(snapshotsOption: DeleteSnapshotsOption.IncludeSnapshots);
await blobClient.DeleteAsync(snapshotsOption: DeleteSnapshotsOption.OnlySnapshots);
```

> If the blob has snapshots and you don't specify `snapshotsOption`, `DeleteAsync` throws an exception.

#### Copy — server-side copy

```csharp
var source = new Uri("https://account.blob.core.windows.net/container/blob.txt");

// Asynchronous: returns immediately; poll GetPropertiesAsync() to see whether it finished
await blobClient.StartCopyFromUriAsync(source);

// Synchronous: waits for the copy to finish before continuing
await blobClient.SyncCopyFromUriAsync(source);
```

> The copy is **server-side**: Azure moves the data between accounts without anything passing through your machine.

### Properties and metadata

These are two different concepts that the exam often mixes up:

| Concept | Managed by | Editable | Examples |
|----------|---------------|-----------|---------|
| **System properties** | Azure | Some | `ContentType`, `ETag`, `LastModified`, `ContentLength` |
| **User metadata** | The developer | Yes | Any key-value pair |

**Editable properties:** `ContentType`, `ContentEncoding`, `ContentLanguage`, `CacheControl`
**Read-only properties:** `ETag`, `LastModified`, `CreationTime`, `ContentLength`

```csharp
// Read properties and metadata (a single call)
BlobProperties props = (await blobClient.GetPropertiesAsync()).Value;

string contentType = props.ContentType;       // system property
DateTimeOffset modified = props.LastModified; // system property
IDictionary<string,string> meta = props.Metadata; // user metadata

// Set user metadata
var metadata = new Dictionary<string, string>
{
    { "author", "john" },
    { "environment", "prod" }
};
await blobClient.SetMetadataAsync(metadata);
```

> Metadata keys must be valid HTTP header names and lowercase is recommended. Replacing the metadata overwrites all existing metadata — it is not a merge operation.

### Lifecycle Management — lifecycle policies

JSON rules that automate moving blobs between tiers and deleting them. They are configured on the Storage Account and evaluated daily.

#### Rule structure

```json
{
  "rules": [{
    "name": "move-logs-to-archive",
    "enabled": true,
    "type": "Lifecycle",
    "definition": {
      "filters": {
        "blobTypes": ["blockBlob"],
        "prefixMatch": ["logs/"]
      },
      "actions": {
        "baseBlob": {
          "tierToCool":    { "daysAfterModificationGreaterThan": 30 },
          "tierToArchive": { "daysAfterModificationGreaterThan": 90 },
          "delete":        { "daysAfterModificationGreaterThan": 365 }
        }
      }
    }
  }]
}
```

#### Available conditions

| Condition | When it is evaluated |
|-----------|-----------------|
| `daysAfterModificationGreaterThan` | Since the last modification |
| `daysAfterCreationGreaterThan` | Since creation |
| `daysAfterLastAccessTimeGreaterThan` | Since the last access (requires enabling access tracking) |

#### Available actions

| Action | On |
|--------|-------|
| `tierToCool` / `tierToCold` / `tierToArchive` | `baseBlob`, `snapshot`, `version` |
| `delete` | `baseBlob`, `snapshot`, `version` |
| `enableAutoTierToHotFromCool` | `baseBlob` |

> Exam rule: "automatically move data when it has gone X days without modification" → **Lifecycle Management Policy**.

### Data protection: soft delete, versioning and snapshots

| Mechanism | What it does | Who enables it |
|-----------|----------|----------------|
| **Soft delete** | Retains deleted blobs for N days (1–365). They can be recovered in that period. | Account configuration |
| **Versioning** | Automatically keeps a previous version on every overwrite or delete. | Account configuration |
| **Snapshot** | Manual point-in-time copy of a blob. It does not change without explicit action. | The developer via SDK/portal |

> Soft delete and versioning are **account configuration** — enabled once, they cover all blobs. Snapshots are **per blob** and manual.

### Immutable Storage — non-modification policy

Lets you configure blobs as **WORM** (Write Once, Read Many): once written, **nobody can modify or delete them** during the defined period — not even the account owner.

Designed for **regulatory compliance**: audits, financial records, tariff plans with an integrity requirement.

#### Two policy types

| Type | How it works | When to use it |
|------|--------------|---------------|
| **Time-based Retention** | The blob is immutable for X days. After that period it can be deleted but not edited. | Periodic audits with a known deadline (e.g. quarterly) |
| **Legal Hold** | Immutable indefinitely until the hold is explicitly lifted. | Legal investigations with no defined end date |

#### How it is applied

It is configured at the **Container** level, not on an individual blob:

```
Container → Policies → Add policy
  Type: Time-based retention
  Interval: 90 (days)
```

During that period, Azure rejects any `PUT`, `DELETE` or `Set Metadata` on the container's blobs — even if the user has the Owner role.

#### Rule for the exam

> If the question mentions **compliance**, **audit**, **records that must not be edited** or **regulatory requirement** → **Immutable Storage + Time-based Retention**.

The access tier (Archive, Cool, Hot) does not protect against modification — it only controls cost and read latency.

### AZ-204 exam keys

| Typical question | Answer |
|----------------|-----------|
| Logs that are only appended | **Append blob** |
| VM disk | **Page blob** |
| Low latency on blobs | **Premium Block Blobs** |
| Minimum access + latency of hours | **Archive** + rehydration |
| Automatically move blobs after X days | **Lifecycle Management Policy** |
| Don't modify data for compliance | **Immutable Storage (WORM)** |
| Recover a deleted blob | **Soft delete** |
| Automatic version history | **Blob versioning** |
| Manual point-in-time copy | **Snapshot** |
| Code to upload a file | `BlobClient.UploadAsync()` |
| Code to read metadata | `BlobClient.GetPropertiesAsync()` |
| Change a blob's Content-Type | `SetHttpHeadersAsync()` |
| Temporary and secure access for third parties | **SAS Token** → see [[azure-storage-access-and-permissions]] |

## Relations

- [[azure-storage-access-and-permissions]] — how to authorize access to blobs (SAS, RBAC, account keys, anonymous)
- [[microsoft-entra-id]] — identity model behind Managed Identity and User Delegation SAS access to Blob Storage
- [[azcopy]] — CLI tool for bulk transfer to and from Blob Storage, including server-side copy
- [[azure-cdn]] — a Storage Account is a common CDN origin for static content

## Tags

#azure #az204
