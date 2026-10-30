---
domain: azure
tags: [azure, az204]
sources: []
updated: 2026-10-30
---

# AzCopy

## What it is

A command-line tool for efficiently transferring data to and from Azure Storage. It works with Azure Blob Storage, Azure Files and Azure Data Lake Storage Gen2.

## What problem it solves

Moving large amounts of data between on-premises systems and Azure (mass migrations, backups, synchronization) is slow and fragile when done file by file. AzCopy handles it in bulk, with resumable jobs and tunable network usage.

## Notes

### Authentication

Two main methods:

| Method | When to use it |
|--------|---------------|
| **Azure AD (Entra ID)** | Recommended. Uses `azcopy login` and delegates permissions via RBAC |
| **SAS token** | Appended directly to the resource URL |

```bash
# With a SAS token
azcopy copy 'https://account.blob.core.windows.net/container?<SAS>' './destination'

# With Azure AD
azcopy login
azcopy copy 'https://account.blob.core.windows.net/container' './destination'
```

### Key commands

| Command | Action |
|---------|--------|
| `azcopy copy` | Copies files (source → destination) |
| `azcopy sync` | Synchronizes only modified files |
| `azcopy list` | Lists the contents of a container |
| `azcopy remove` | Deletes blobs or files |
| `azcopy make` | Creates a container or share |
| `azcopy jobs list` | Shows running or completed jobs |
| `azcopy jobs resume` | Resumes a failed or cancelled job |

### Key concepts

#### Jobs and resilience

Every operation generates a **job ID**. If it fails, you can resume it with `azcopy jobs resume <jobID>` without repeating everything.

#### Server-side copy (Blob → Blob)

AzCopy can copy data directly between storage accounts **without passing through the local client**.

```bash
azcopy copy 'https://source.blob.core.windows.net/container?<SAS>' \
            'https://destination.blob.core.windows.net/container?<SAS>' \
            --recursive
```

#### `copy` vs `sync`

- `copy` — always copies everything.
- `sync` — compares and transfers only the differences. With `--delete-destination` it deletes at the destination whatever does not exist at the source.

#### The `--recursive` flag

Required to include subdirectories. Without it, only the root level is copied.

#### Performance

`--cap-mbps` and `--concurrency-value` control network usage so it is not saturated.

### RBAC roles required (with Azure AD)

| Role | Access |
|-----|--------|
| **Storage Blob Data Contributor** | Read and write blobs |
| **Storage Blob Data Reader** | Read-only |
| **Storage Queue Data Contributor** | Queues |

See [[azure-storage-access-and-permissions]] and [[azure-rbac]] for more context on permissions.

### AZ-204 exam keys

- Difference between `copy` (always copies) and `sync` (only differences)
- Jobs can be resumed with `azcopy jobs resume`
- Supports server-side copy between accounts without passing through the client
- Two authentication methods: Azure AD (recommended) and SAS token
- `--recursive` is mandatory for directories

## Relations

- [[azure-blob-storage]] — the main target of AzCopy transfers
- [[azure-storage-access-and-permissions]] — SAS vs Azure AD access models used by AzCopy authentication
- [[azure-rbac]] — the data roles AzCopy needs when authenticating with Azure AD

## Tags

#azure #az204
