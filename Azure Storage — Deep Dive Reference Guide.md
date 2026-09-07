

Storage Accounts are the umbrella container for four distinct services (Blob, File, Queue, Table), each solving a different data-shape problem. This guide goes top-down: the account itself, then a deep dive on Blob Storage specifically (since it's where most of the interesting depth lives — tiers, protection, security), then the shared data-protection and security concepts that apply across the platform.

---

## 1. Storage Accounts

A **Storage Account** is a management, billing, and namespace boundary — it's the "container" that holds Blob, File, Queue, and Table storage underneath it, each exposed through its own endpoint (`<account>.blob.core.windows.net`, `.file.`, `.queue.`, `.table.`).

**Account types:**

|Type|Notes|
|---|---|
|**General-purpose v2 (GPv2)**|The default, recommended choice for almost everything — supports all four services, all access tiers, latest features|
|**Premium Block Blob**|SSD-backed, for high-transaction-rate or low-latency blob workloads|
|**Premium File Shares**|SSD-backed Azure Files, for latency-sensitive file-share workloads|
|**Premium Page Blobs**|SSD-backed, low-latency page blobs (legacy unmanaged VM disk pattern)|
|**Azure Data Lake Storage Gen2**|GPv2 account with **hierarchical namespace** enabled — turns blob storage into a true file-system-like structure with real directories, used for big data/analytics workloads (Spark, Databricks, Synapse)|

**Performance tiers:** **Standard** (magnetic/HDD-backed, cost-optimized, the default for most data) vs. **Premium** (SSD-backed, low and consistent latency, higher cost — used when your workload is latency- or IOPS-sensitive rather than capacity-driven).

The four services living inside an account:

### 1.1 Blob Storage

Object storage for unstructured data — images, video, documents, backups, log files, VM images. The most heavily used of the four; deep dive in Section 2.

### 1.2 File Storage (Azure Files)

Fully managed file shares accessible via the **SMB** and **NFS** protocols — mountable simultaneously from cloud VMs, on-prem servers, and containers, just like a traditional network file share. Common use cases: lift-and-shift of on-prem file servers, shared application config, "home directories" for user profiles. **Azure File Sync** extends this further by caching an Azure file share on an on-prem Windows Server, giving hybrid access with local-speed reads while the cloud copy stays authoritative.

### 1.3 Queue Storage

A simple, lightweight messaging store for decoupling application components — one component drops a message, another picks it up asynchronously. Messages up to 64 KB, **at-least-once delivery** (no strict ordering guarantee). It's intentionally minimal — for scenarios needing ordering guarantees, dead-lettering, sessions, or transactions, **Azure Service Bus** is the appropriate step up; Queue Storage's advantage is simplicity and very low cost for basic decoupling.

### 1.4 Table Storage

A NoSQL key-value store for schemaless structured/semi-structured data, addressed by a **PartitionKey** (groups related rows, defines the scale-out boundary) and **RowKey** (unique within a partition). It's still available and supported, but for new development Microsoft steers most workloads toward the **Azure Cosmos DB Table API** instead — same programming model, but with global distribution, guaranteed low-latency SLAs, and automatic secondary indexing that classic Table Storage lacks.

---

## 2. Blob Storage — Deep Dive

### 2.1 Containers

A **container** is the top-level organizational unit inside a storage account for blobs — conceptually similar to a bucket (AWS S3) or a root folder. Blob Storage uses a genuinely **flat namespace**: there's no real folder hierarchy under the hood. "Folders" you see in tools like Storage Explorer or the portal are a virtual illusion created by using `/` characters inside blob names (e.g., `logs/2026/09/app.log` is really just one blob whose name happens to contain slashes).

**Public access levels** are set per container:

- **Private (no anonymous access)** — default, recommended.
- **Blob** — anonymous read access to individual blobs if you know the full URL, but container listing is not allowed.
- **Container** — anonymous read access to blobs _and_ the ability to list all blobs in the container.

_(If you need a true hierarchical file system rather than a simulated one, that's what enabling the hierarchical namespace / ADLS Gen2 on the account is for — see Section 1.)_

### 2.2 Blobs — Three Types

|Blob Type|Optimized For|Typical Use|
|---|---|---|
|**Block Blob**|Uploading/storing large amounts of discrete data, built from independently-managed blocks|The default choice — documents, media, backups, general file storage|
|**Append Blob**|Append-only operations|Logging scenarios where many sources continuously write to the end of the same blob|
|**Page Blob**|Random read/write access at 512-byte page granularity|VHD/VM disk files (the storage layer underneath _unmanaged_ disks)|

### 2.3 Access Tiers

Access tiers let you balance **storage cost** against **access/retrieval cost and latency**, matching price to how "hot" the data actually is.

|Tier|Access Pattern|Storage Cost|Access/Retrieval Cost|Minimum Retention|Retrieval Latency|
|---|---|---|---|---|---|
|**Hot**|Frequent access|Highest|Lowest|None|Immediate|
|**Cool**|Infrequent (accessed roughly monthly or less)|Lower than Hot|Higher than Hot|30 days (early deletion fee if removed sooner)|Immediate|
|**Cold**|Rarely accessed (roughly quarterly or less)|Lower than Cool|Higher than Cool|90 days|Immediate|
|**Archive**|Rarely accessed, long-term retention|Lowest by far|Highest|180 days|**Offline** — requires rehydration, typically hours, before the blob is readable again|

Key nuance: **Hot/Cool/Cold** are all _online_ tiers — the data is always immediately readable, you're just paying a different balance of storage-vs-access cost. **Archive** is fundamentally different — it's an _offline_ tier; you cannot read an archived blob directly, you must first trigger a **rehydration** (either "Standard" priority, taking up to ~15 hours, or "High" priority, faster but pricier) which copies the blob to Hot or Cool before it becomes readable.

**Lifecycle Management policies** automate tier transitions and expiration so you don't have to move data manually — e.g., "move blobs to Cool after 30 days of no modification, to Archive after 90 days, and delete after 7 years," all rule-based on age since last modified (or last accessed, if access-tracking is enabled).

```json
{
  "rules": [
    {
      "name": "moveToArchiveAndExpire",
      "type": "Lifecycle",
      "definition": {
        "filters": { "blobTypes": ["blockBlob"], "prefixMatch": ["logs/"] },
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 30 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 90 },
            "delete": { "daysAfterModificationGreaterThan": 2555 }
          }
        }
      }
    }
  ]
}
```

---

## 3. Data Protection

### 3.1 Replication

Replication determines **how many copies of your data exist, and where** — this is your durability/disaster-recovery dial.

|Option|Copies|Spread|Protects Against|Read access to secondary?|
|---|---|---|---|---|
|**LRS** (Locally Redundant)|3|Single datacenter|Drive/node/rack-level hardware failure|N/A — no secondary|
|**ZRS** (Zone Redundant)|3|Across Availability Zones in the same region, synchronous|Datacenter-level failure within the region|N/A — still one logical copy, just zone-spread|
|**GRS** (Geo-Redundant)|6 (3+3)|LRS in primary region + async-replicated LRS copy in the paired secondary region|Full regional outage/disaster|No — secondary not readable unless a failover occurs|
|**RA-GRS** (Read-Access Geo-Redundant)|6 (3+3)|Same as GRS|Full regional outage/disaster|**Yes** — secondary is readable at all times via a distinct `-secondary` endpoint|

There are also **GZRS** and **RA-GZRS**, which combine ZRS in the primary region with geo-replication to the secondary — the strongest (and most expensive) combination, giving you both zone-level and region-level protection simultaneously.

**Choosing:** LRS is fine for easily-reproducible or non-critical data; ZRS is the sensible default for production data where regional-disaster protection isn't required but zone resilience is; GRS/RA-GRS/GZRS/RA-GZRS come into play when business continuity requires surviving the loss of an entire Azure region.

### 3.2 Soft Delete

- **Blob soft delete** — when a blob is deleted or overwritten, it's retained (recoverable) for a configurable retention window (1–365 days) instead of being immediately purged.
- **Container soft delete** — the same protection at the container level, letting you recover an entire accidentally-deleted container.

Soft delete is a safety net against **accidental** deletion/overwrite — it is _not_ a substitute for a real backup strategy (it won't help against, say, malicious deliberate purging with sufficient permissions and time, or logical/application-level data corruption written intentionally).

### 3.3 Versioning

When enabled, **every** modification or overwrite of a blob automatically creates a new **version**, each with a unique version ID; previous versions become read-only snapshots you can list, read, or promote back to current. Versioning and soft delete are designed to work together: soft delete protects against a blob/version being deleted, versioning protects against a blob being silently overwritten — combined, you get full point-in-time recoverability without a separate backup pipeline for many scenarios.

### 3.4 Immutable Storage

Implements **WORM (Write Once, Read Many)** semantics — once locked, a blob genuinely cannot be modified or deleted by anyone, including the account owner, until the policy allows it. Two policy types:

- **Time-based retention** — the blob is locked for a fixed period (e.g., 7 years) and cannot be deleted or modified until that period elapses, even by an account owner with full access keys.
- **Legal hold** — an indefinite hold with no fixed expiry, removed only by explicit administrative action — used for litigation/investigation scenarios where the retention period isn't known in advance.

This is the feature that satisfies regulatory compliance requirements like **SEC Rule 17a-4** or **FINRA** record-retention rules, where the whole point is that _no one_, including a compromised admin account, should be able to alter the data early.

---

## 4. Security

Three distinct ways to authenticate/authorize access to storage data — understanding the trade-offs between them matters as much as knowing they exist.

### 4.1 SAS Tokens (Shared Access Signatures)

A SAS is a **signed URL** granting delegated, time-limited, scope-limited access to storage resources without handing out the account keys themselves.

|SAS Type|Signed With|Scope|
|---|---|---|
|**User Delegation SAS**|Microsoft Entra ID credentials|Scoped to what the requesting Entra identity is actually permitted (via RBAC) — **most secure**, recommended default|
|**Service SAS**|Storage account key|Scoped to a single service (blob, file, queue, or table)|
|**Account SAS**|Storage account key|Can span multiple services and grant account-level operations|

Configurable on any SAS: specific **permissions** (read/write/delete/list/etc.), a **start and expiry time**, allowed **IP ranges**, and a requirement for **HTTPS only**.

**Best practices:** prefer **User Delegation SAS** wherever possible (it isn't tied to a long-lived account key and inherits Entra-based revocability), keep expiry windows short, and use **stored access policies** on the container when you need the ability to revoke a whole batch of already-issued SAS tokens at once (a raw SAS otherwise remains valid until it naturally expires — the only way to revoke it early is to rotate the key it was signed with, which invalidates _every_ SAS signed with that key).

### 4.2 Access Keys

Every storage account has **two 512-bit access keys** (`key1`/`key2`) that grant **full administrative control** over every service and every piece of data in the account — equivalent to root access.

- Because they're this powerful, a leaked access key is a serious incident — treat them like the most sensitive secret in your environment.
- **Best practices:** never hard-code them into application source; store them in **Azure Key Vault** if you must use them at all; rotate regularly; and prefer avoiding them entirely in favor of Managed Identity for anything running inside Azure.
- **Why two keys exist:** it enables zero-downtime rotation — update your application to key2 while key1 is still valid, confirm everything works, _then_ regenerate key1, and vice versa next rotation cycle.

### 4.3 Managed Identity Access

The modern, recommended approach: an Azure resource (VM, App Service, Function App, AKS pod via Workload Identity, etc.) gets an **identity in Microsoft Entra ID**, and that identity is granted **RBAC roles directly on the storage data** — no keys, no SAS tokens, no secrets to manage or rotate at all.

- **System-assigned** identity — tied 1:1 to the resource's lifecycle (created/destroyed with it).
- **User-assigned** identity — a standalone identity you create once and attach to multiple resources.
- **Data-plane RBAC roles** matter here and are a common point of confusion: roles like **Storage Blob Data Reader**, **Storage Blob Data Contributor**, **Storage Blob Data Owner**, or **Storage Queue Data Contributor** grant access to the _actual data_. This is distinct from the classic **Contributor** role on the storage account resource itself, which lets you manage the account's _configuration_ (create containers, change replication settings) but — by design, for defense-in-depth — does **not** by itself grant access to read or write the data inside it.

### Comparing the Three

||SAS Token|Access Key|Managed Identity|
|---|---|---|---|
|Credential to manage|A signed URL, self-expiring|Long-lived shared secret|None — identity-based|
|Blast radius if leaked|Limited to what the SAS was scoped to|Entire account, full control|N/A — nothing to leak|
|Revocability|Hard (unless using stored access policy)|Full account key rotation required|Instant, via RBAC role removal|
|Best for|Time-limited access for external/third-party clients|Legacy scenarios, break-glass admin tasks|Anything running as an Azure resource — the default choice today|

**Practical rule of thumb:** if the caller is a first-party Azure resource you control, use **Managed Identity**. If the caller is an external client that needs temporary, narrowly-scoped access (e.g., a browser uploading directly to a blob, or a partner system pulling a specific file), use a **User Delegation SAS**. Treat raw **Access Keys** as a last resort, reserved for scenarios where neither of the above is technically possible.