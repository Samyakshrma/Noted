
---

## 1. Virtual Machines (IaaS)

VMs give you a full OS you control — patching, networking stack, disk layout, everything. Maximum flexibility, maximum operational responsibility.

### 1.1 VM Families

Every VM size is defined largely by its **vCPU-to-memory ratio**, plus the underlying processor generation and any specialized hardware (GPU, local NVMe, RDMA). The three families you listed cover the "general workload" spectrum:

|Family|vCPU:Memory Ratio|Example Series|Best For|
|---|---|---|---|
|**General Purpose**|Balanced (~1:4 GB)|Dsv5, Dasv5 (AMD), Dpsv5 (Arm/Ampere), B-series (burstable)|Dev/test, small-medium databases, low-medium traffic web/app servers|
|**Compute Optimized**|High CPU, lower memory (~1:2 GB)|Fsv2|Medium-traffic web servers, batch processing, application servers, network appliances, gaming servers|
|**Memory Optimized**|High memory, lower CPU (~1:8 GB or higher)|Esv5, Easv5 (AMD), Mv2 (multi-TB RAM)|Relational databases (SQL Server, SAP HANA), in-memory caches (Redis), large-scale analytics|

**How to actually choose:**

- **General Purpose** is your default starting point — pick it unless a specific bottleneck (CPU-bound or memory-bound) tells you otherwise.
- **Compute Optimized** matters when your workload is CPU-bound relative to memory — e.g., a web tier doing lots of request processing but not holding large datasets in RAM.
- **Memory Optimized** matters when memory is the constraint — large buffer pools/caches, big datasets held in-memory, or database engines that perform better with more RAM per core (SQL Server buffer pool, SAP HANA, large Redis instances).

**Also worth knowing (not asked but adjacent):** Storage Optimized (Lsv3 — high disk throughput/IOPS, NVMe), GPU (NC/ND/NV — ML training/inference, visualization), and HPC (HB/HC — tightly-coupled scientific workloads with InfiniBand/RDMA) round out the full family list, but the three above cover 90% of real-world VM decisions.

**B-series burstable VMs** deserve a special mention within General Purpose: they accrue "CPU credits" during idle periods and spend them during bursts, making them very cost-effective for workloads that are mostly idle with occasional spikes (dev boxes, low-traffic web apps).

### 1.2 VM Scale Sets (VMSS)

A **Scale Set** lets you manage a group of _identical, load-balanced_ VMs as a single logical unit, and automatically grow/shrink that group in response to demand.

**Orchestration modes** (a critical distinction):

||**Uniform**|**Flexible** (current default/recommended)|
|---|---|---|
|VM identity|VMs are near-identical, indexed instances|Each VM is a standard Azure VM resource — individually addressable|
|Mixing VM sizes/types|No|Yes — mix Spot + Regular, different sizes in one scale set|
|Fault domain spread|Simpler, less granular control|Explicit control, works naturally with Availability Zones|
|Typical use|Large homogeneous stateless fleets|Modern workloads needing resilience + flexibility (this is what backs AKS node pools)|

**Autoscaling** triggers on:

- **Metric-based rules** — CPU %, memory, queue length, or custom Azure Monitor metrics (scale out when CPU > 70% for 10 min, scale in when < 30%).
- **Schedule-based rules** — pre-provision capacity ahead of known traffic patterns (e.g., business hours).

**Upgrade policies** control how configuration changes (like a new image version) roll out to running instances:

- **Manual** — you trigger upgrades yourself, instance by instance.
- **Automatic** — all instances upgraded immediately, no control over sequencing (risky for prod).
- **Rolling** — batched, health-check-gated rollout, similar in spirit to a Kubernetes rolling deployment — the production-safe choice.

```bash
# Example: create a Flexible-orchestration scale set
az vmss create \
  --resource-group myRG \
  --name myScaleSet \
  --orchestration-mode Flexible \
  --image Ubuntu2204 \
  --vm-sku Standard_D2s_v5 \
  --zones 1 2 3 \
  --instance-count 3
```

VMSS is the substrate underneath a lot of "managed" Azure compute — AKS node pools, for instance, are just Flexible scale sets with Kubernetes bootstrapped onto them.

### 1.3 Availability Sets

An Availability Set is a **logical grouping** that spreads VMs across:

- **Fault Domains (FDs)** — groups of hardware sharing a power source and network switch. Typically 2–3 FDs available per set. Protects against a rack-level hardware failure.
- **Update Domains (UDs)** — groups of VMs that Azure will not reboot simultaneously during planned host maintenance. Default 5, configurable up to 20.

**What it protects against:** localized hardware failure and planned maintenance _within a single datacenter_. **What it does NOT protect against:** the loss of an entire datacenter.

- **SLA:** 99.95% when you run 2+ VMs in the same Availability Set.
- **Cost:** free — it's just a placement construct, no extra charge.
- Availability Sets are increasingly a "legacy" pattern for new designs — most new architectures reach for Availability Zones instead when the region supports them, since zones give datacenter-level protection at a similarly low cost.

### 1.4 Availability Zones

Availability Zones are **physically separate locations within an Azure region**, each with independent power, cooling, and networking. Regions that support zones guarantee a minimum of **three zones**.

- **SLA:** 99.99% when VMs are deployed across 2+ zones.
- **Zonal vs. zone-redundant:** a _zonal_ resource is pinned to one specific zone (e.g., "this VM lives in Zone 2"); a _zone-redundant_ resource (like a Standard Load Balancer or zone-redundant storage) automatically spans all zones with no pinning needed.
- Inter-zone latency within a region is low enough (typically sub-2ms round trip) that **synchronous replication** across zones is viable — this is what underpins zone-redundant storage and multi-zone SQL/Cosmos DB configurations.
- Best practice: combine **Availability Zones + VM Scale Sets (Flexible mode)** for workloads that need both elastic scale and datacenter-level fault tolerance — this is the modern replacement pattern for "just use an Availability Set."

**Decision rule of thumb:** if the region supports zones, use zones for anything production-critical. Fall back to Availability Sets only in regions without zone support.

### 1.5 Managed Disks

Managed Disks abstract away the underlying storage account management — Azure handles placement, replication, and scaling of the disk storage for you.

|Disk Type|Performance Tier|Typical Use|
|---|---|---|
|**Ultra Disk**|Sub-millisecond latency, independently configurable IOPS/throughput|SAP HANA, top-tier transactional databases|
|**Premium SSD v2**|High performance, granular IOPS/throughput tuning without resizing the disk|Performance-sensitive production workloads, more cost-flexible than Ultra|
|**Premium SSD**|Consistent low latency (SSD-backed)|Production VMs, databases|
|**Standard SSD**|Moderate, consistent performance|Web servers, lightly-used enterprise apps|
|**Standard HDD**|Lowest cost, higher latency|Backup, infrequently accessed data, dev/test|

**Key concepts:**

- **OS disk vs. Data disk vs. Temp disk** — the temp disk is _local, ephemeral_ storage on the physical host (not a managed disk at all) — it's wiped on deallocation/migration and should never hold data you care about (page files, scratch space only).
- **Caching modes** — None, ReadOnly, ReadWrite. ReadOnly caching is typically recommended for data disks holding read-heavy database files; ReadWrite for OS disks.
- **Snapshots & Images** — a snapshot is a point-in-time, read-only copy of a disk (great for backup or cloning); an image is a generalized template (VM deprovisioned/sysprepped) used to stamp out new VMs.
- **Shared Disks** — allow a single managed disk to attach to multiple VMs simultaneously, enabling clustering scenarios (Windows Server Failover Clustering) that need shared storage.
- **Encryption** — Server-Side Encryption (SSE) is on by default with platform-managed keys, or you can bring your own key via Azure Key Vault (customer-managed keys); Azure Disk Encryption additionally offers in-guest encryption (BitLocker on Windows, DM-Crypt on Linux) for defense-in-depth.
- **Bursting** — Premium SSD disks under a certain size get on-demand IOPS/throughput bursting for handling spiky workloads without provisioning for peak permanently.

---

## 2. Azure App Service (PaaS)

App Service removes the OS/patching burden entirely — you deploy code or a container, Azure runs it on a managed compute layer.

### 2.1 Web Apps

- Hosts web applications and REST APIs across multiple language stacks: .NET, Java, Node.js, PHP, Python, Ruby, or a custom container.
- Built-in CI/CD hooks — GitHub Actions, Azure DevOps Pipelines, or simple local Git push-to-deploy.
- Custom domains with free App Service Managed Certificates for TLS.
- Runs on an **App Service Plan**, which is the actual unit of compute (VM size + region + pricing tier) — multiple apps can share one plan.

### 2.2 API Apps

Architecturally, API Apps run on **the same underlying App Service infrastructure** as Web Apps — there's no separate compute layer. What differs is the tooling optimized around REST API hosting:

- Built-in Swagger/OpenAPI definition generation and discovery.
- CORS support configured natively (no manual middleware needed for cross-origin browser calls).
- Clean integration path into **API Management** for gateway concerns — throttling, versioning, developer portals, subscription keys.
- "Easy Auth" — built-in authentication/authorization against Microsoft Entra ID, Google, Facebook, etc. without writing auth code into the app.

### 2.3 Deployment Slots

Available from the **Standard** tier upward. A slot is a fully live, separately addressable instance of your app (its own hostname) that you can stage a new version in before it's customer-facing.

- **Slot swap** exchanges the virtual IP/routing between two slots — because both are already warm, this achieves **near-zero-downtime deployment** (no cold start penalty like a fresh deploy would have).
- **Sticky settings** — you can mark specific app settings or connection strings as "slot-specific" so they _don't_ swap (e.g., a staging slot should keep pointing at a staging database even after a swap).
- **Traffic routing / testing-in-production** — you can route a percentage of live traffic to a slot before a full swap, enabling canary releases.
- Classic pattern: `staging` slot → validate → swap into `production`.

### 2.4 Scaling

|Approach|What it changes|Notes|
|---|---|---|
|**Scale Up** (vertical)|App Service Plan tier/size (Free → Shared → Basic → Standard → Premium → Isolated)|Changes CPU/memory/features available per instance|
|**Scale Out** (horizontal)|Instance count|Manual, or **autoscale rules** based on metrics (CPU%, memory, HTTP queue length) or a schedule|

- **Isolated tier** deploys into an **App Service Environment (ASE)** — a single-tenant, VNet-injected deployment of App Service for workloads with strict network isolation, high scale, or compliance requirements.
- Autoscale is configured through Azure Monitor autoscale settings, with min/max/default instance counts and cooldown periods to avoid thrashing.

---

## 3. Azure Functions (Serverless / FaaS)

Functions is event-driven compute where you write only the handler — Azure manages provisioning, scaling (including to zero), and the execution host.

**Core mental model — Trigger vs. Binding:**

- A **trigger** is what _invokes_ the function (exactly one per function).
- **Bindings** (input/output) are a declarative way to read/write data (a Cosmos DB document, a blob, a queue message) _without_ writing SDK boilerplate — you just decorate a parameter and the runtime wires it up.

### 3.1 Triggers

|Trigger|Fires On|Key Details|
|---|---|---|
|**HTTP**|Incoming HTTP request|Lets you build lightweight APIs directly; supports auth levels (`anonymous`, `function` — key required, `admin`)|
|**Blob**|New/updated blob in a storage container|Default implementation polls the container (can have latency up to ~10 min); an **Event Grid–based blob trigger** exists for near-real-time reaction instead|
|**Queue**|New message in an Azure Storage Queue|Supports **poison-message handling** — after a message fails processing a configurable number of times (default 5), it's moved to a `-poison` queue instead of retried forever|
|**Event Hub**|New events in a high-throughput event stream|Designed for telemetry/log ingestion at scale; processes events in **batches**, uses **checkpointing** to track progress per partition, and scales function instances roughly per-partition|

### 3.2 Hosting Plans

||**Consumption Plan**|**Premium Plan**|
|---|---|---|
|Scaling|True scale-to-zero, event-driven scale controller|Same event-driven scaling engine, but with **pre-warmed ("always ready") instances**|
|Cold start|Yes — a real consideration for latency-sensitive workloads|Effectively eliminated via pre-warmed instances|
|VNet integration|Limited/not supported|Fully supported — can reach resources inside a private network|
|Execution time limits|Shorter default/max timeout (function apps typically default to 5 min, configurable up to a hard cap)|Much longer or effectively unbounded execution windows|
|Billing|Pure pay-per-execution (GB-seconds of memory × duration), generous monthly free grant|Billed for pre-warmed instance capacity (core-seconds/memory reserved) _plus_ any additional scale-out — more predictable, higher baseline cost|
|Best for|Spiky, infrequent, latency-tolerant workloads|Latency-sensitive production APIs, workloads needing VNet access or longer execution|

Two more hosting options worth knowing exist, even though not asked in depth: the **Dedicated (App Service) Plan** runs Functions on a regular App Service Plan you already own (useful to consolidate billing/scale with an existing Web App), and Functions can also run as a **container inside Azure Container Apps**, blending the serverless-functions programming model with Container Apps' KEDA-based scaling engine.

---

## 4. Azure Container Apps

### 4.1 Containerized Workloads

Container Apps is a **fully managed serverless container platform** built on Kubernetes, KEDA, Dapr, and Envoy under the hood — but you never touch the Kubernetes API directly. It sits deliberately _between_ App Service/Functions and full AKS: you get container flexibility without cluster operations.

- Ideal for microservices, event-driven processing, and background jobs.
- Supports **multiple containers per app** (sidecar pattern) within a single revision — e.g., your app container plus a logging/proxy sidecar.
- Native **Dapr integration** for service-to-service invocation, pub/sub messaging, state management, and external service bindings — without writing that plumbing yourself.
- Built-in ingress (HTTP or raw TCP), configurable as external (public internet) or internal (VNet-only) only.

### 4.2 KEDA Scaling

Container Apps uses **KEDA (Kubernetes Event-Driven Autoscaling)** as its scaling engine.

- **HTTP scaling** — scale based on concurrent request count per replica.
- **Custom scalers** — 50+ built-in KEDA scalers cover Azure Storage Queue length, Service Bus queue/topic depth, CPU/memory thresholds, Kafka lag, and more.
- **Scale to zero** — a true serverless property: an idle app can scale down to zero replicas and incur no compute cost, then scale back up on the next incoming event/request.
- Min/max replica counts are configurable per app to bound cost and guarantee minimum availability where needed.

### 4.3 Revisions

A **revision** is an immutable snapshot of your Container App's configuration at a point in time. Any change to a "revision-scoped" setting (like the container image or scale rules) creates a brand-new revision.

- **Revision modes:**
    - **Single** — only the latest revision is active; older ones are deactivated automatically on each new deploy.
    - **Multiple** — several revisions can run concurrently.
- In Multiple mode, you can assign **traffic-weight percentages** across revisions — enabling canary releases and blue-green deployments natively, conceptually similar to App Service Deployment Slots but built into the container model itself rather than being a separate feature.

---

## 5. Azure Kubernetes Service (AKS) — Master Reference

AKS is Azure's managed Kubernetes offering: Azure operates the **control plane**, you operate the **worker nodes** (which live in your own subscription and you pay for directly).

### 5.1 Architecture Overview

Two logical halves:

1. **Control Plane** — fully managed by Azure, hidden from you, no VMs you can see or SSH into.
2. **Node Pools** — Virtual Machine Scale Sets running in your subscription, where your workloads (pods) actually execute.

This split is the single most important AKS concept: it's what makes AKS "managed" — you never patch or scale the API server yourself, but you're fully responsible for the nodes.

### 5.2 Control Plane

Contains the standard Kubernetes control-plane components: **API server, etcd, scheduler, controller manager**.

- **Pricing tiers:**
    - **Free** — no SLA guarantee (though generally reliable), suitable for dev/test.
    - **Standard** — financially backed **99.95% SLA** when combined with Availability Zones, the baseline for production.
    - **Premium** — adds **Long-Term Support (LTS)** Kubernetes versions for extended support windows.
- **API server exposure** — public by default, or deployable as a **private cluster**, where the API server endpoint is only reachable from inside your VNet (via Private Link/Private DNS) — a meaningful security hardening step.

### 5.3 Worker Nodes

- **Node pools** are groups of nodes sharing identical VM size/configuration.
- **System node pools** run critical cluster add-ons (CoreDNS, `metrics-server`, tunnel components); **user node pools** run your application workloads. Best practice: **always separate system and user pools** so app workloads can't starve cluster-critical pods.
- Under the hood, each node pool **is** a VM Scale Set (Flexible mode).
- You can run **multiple node pools** with different VM sizes for different needs — e.g., a GPU pool for ML inference alongside a general-purpose pool for the web tier.
- **OS options** — Linux (Ubuntu, or Azure Linux/Mariner — Microsoft's lightweight container-optimized distro) and Windows Server node pools (2019/2022) for workloads requiring Windows containers.

### 5.4 Networking

|Plugin|Pod IP Source|Notes|
|---|---|---|
|**kubenet**|Overlay network, NAT'd out to the VNet|Simple, conserves VNet IP addresses, but limits some advanced scenarios (e.g., direct pod-to-pod routing across VNets)|
|**Azure CNI**|Each pod gets a real, routable VNet IP|Required for scenarios needing direct pod addressability from outside the cluster; consumes VNet IP space fast at scale|
|**Azure CNI Overlay**|Pods get IPs from a separate overlay space, _not_ consuming VNet IPs|Newer default recommendation — combines Azure CNI's capability with kubenet's IP efficiency|
|**Azure CNI powered by Cilium**|eBPF-based dataplane|Higher performance/scale, replaces kube-proxy with eBPF for networking and enforces network policy|

- **Network Policy** — restrict pod-to-pod traffic (namespace isolation, deny-by-default patterns) via **Azure Network Policy Manager** or **Calico**.
- **Service types** — `ClusterIP` (internal only), `NodePort`, `LoadBalancer` (auto-provisions an Azure Load Balancer, public or internal).
- **DNS** — CoreDNS handles in-cluster service discovery; for private clusters, integration with Azure Private DNS zones resolves the API server internally.

### 5.5 Storage

Persistent storage is provided through **CSI (Container Storage Interface) drivers**:

|CSI Driver|Backing Service|Access Mode|Use Case|
|---|---|---|---|
|**Azure Disk CSI**|Managed Disks|ReadWriteOnce (single-pod block storage)|Databases, stateful single-instance workloads|
|**Azure Files CSI**|Azure Files (SMB/NFS)|ReadWriteMany (shared)|Shared config, content shared across many pods|
|**Azure Blob CSI**|Blob Storage (NFS/blobfuse)|Varies|Large unstructured data accessed like a filesystem|

- **StorageClasses** define the provisioner + parameters (disk SKU, replication) and support both **dynamic** (auto-created PersistentVolume on demand) and **static** (pre-created, manually bound) provisioning.
- **Volume snapshots** are supported for backup/restore workflows on Azure Disk-backed volumes.
- **Ephemeral OS disks** — an option to place the node's OS disk on local VM cache/temp storage instead of remote managed disk storage, trading persistence for speed and lower cost — appropriate since node OS state should be disposable anyway in Kubernetes.

### 5.6 Ingress

Ingress in Kubernetes always follows the **Ingress resource + Ingress Controller** pattern — the resource declares routing rules, the controller is the actual proxy that implements them. Options on AKS:

- **Application Gateway Ingress Controller (AGIC)** — uses Azure Application Gateway (an L7 load balancer with built-in WAF) as the ingress, giving you a managed Azure PaaS component instead of a controller running as cluster pods.
- **NGINX Ingress Controller** — the most common open-source choice; runs as pods inside the cluster.
- **Web Application Routing add-on** — an AKS-managed option that wraps NGINX + Azure DNS integration + automatic TLS certificate management (via cert-manager/Let's Encrypt), reducing manual setup.
- **Istio-based ingress** — available if you enable the Istio service mesh add-on, useful when you already need mesh-level traffic management (mTLS, fine-grained routing) beyond basic ingress.

### 5.7 Scaling

AKS scaling operates at two layers — pods and nodes:

- **Horizontal Pod Autoscaler (HPA)** — adjusts pod _replica count_ based on CPU/memory or custom metrics.
- **Vertical Pod Autoscaler (VPA)** — adjusts a pod's resource _requests/limits_ rather than replica count (right-sizing).
- **Cluster Autoscaler** — adds or removes _nodes_ in a node pool based on whether pods are pending due to insufficient capacity, or nodes are underutilized.
- **KEDA add-on** — the same event-driven scaling engine used by Container Apps is available as a managed AKS add-on, letting pods scale off queue depth, event hub lag, etc., not just CPU/memory.
- **Node Auto Provisioning (Karpenter-based)** — a newer, more efficient node-provisioning approach that picks optimal VM sizes/types on the fly rather than scaling a fixed-shape node pool.

### 5.8 Upgrades

- **Kubernetes version upgrades** happen in two stages: the **control plane** is upgraded first (Azure-managed, no action needed beyond triggering it), then **node pools** are upgraded — and node pools can be upgraded independently and on their own schedule.
- **Surge upgrade settings** let you temporarily add extra ("surge") nodes during a node pool upgrade so workloads can be drained/rescheduled with less disruption, rather than upgrading nodes strictly in place.
- **Node image upgrades** are a separate, lighter-weight operation from full Kubernetes version upgrades — they patch the underlying OS/security updates on nodes without changing the Kubernetes minor/patch version.
- **Auto-upgrade channels** (`none`, `patch`, `stable`, `rapid`, `node-image`) automate how aggressively the cluster keeps itself current.
- **Planned Maintenance windows** let you constrain _when_ Azure is allowed to apply upgrades/updates, so they land during low-traffic periods.
- An alternative pattern to in-place upgrades entirely is the **blue-green node pool** strategy: stand up a new node pool on the target version, cordon and drain the old pool, then delete it — trading some cost/complexity for a cleaner rollback path.

### 5.9 Security

- **Identity & access** — Microsoft Entra ID (Azure AD) integrates with the cluster for _authentication_; Kubernetes RBAC (or **Azure RBAC for Kubernetes Authorization**, which lets you manage K8s permissions via Azure role assignments instead of `kubectl`-applied RoleBindings) handles _authorization_.
- **Managed Identities** — both the control plane and the kubelet use system-assigned managed identities by default, eliminating the need to manage service principal secrets/rotation.
- **Microsoft Entra Workload Identity** — the current mechanism (replacing the deprecated AAD Pod Identity) for federating a Kubernetes service account with an Azure AD identity, so individual pods can securely call Azure services (Key Vault, Storage, etc.) without embedding credentials.
- **Private clusters** — as noted above, hides the API server from the public internet entirely.
- **Network Policy** — pod-to-pod traffic segmentation, enforce deny-by-default zero-trust networking inside the cluster.
- **Azure Policy for AKS** — a Gatekeeper (OPA)–based admission control layer letting you enforce governance rules cluster-wide (e.g., "no privileged containers," "all pods must define resource limits").
- **Microsoft Defender for Containers** — runtime threat detection across images, running containers, and the cluster control plane.
- **Secrets management** — the **Azure Key Vault Provider for Secrets Store CSI Driver** mounts secrets from Key Vault directly as volumes inside pods, avoiding native Kubernetes Secrets (which are only base64-encoded, not encrypted, by default).
- **Supply chain security** — tight integration with **Azure Container Registry** (private image storage, vulnerability scanning) and support for admission control that only allows signed/trusted images to run.

---

## 6. Choosing the Right Compute Service

A quick decision reference across everything above:

|Dimension|VMs|App Service|Functions|Container Apps|AKS|
|---|---|---|---|---|---|
|**Abstraction level**|IaaS|PaaS|Serverless/FaaS|Serverless containers|Managed container orchestration|
|**You manage**|OS, patching, networking, everything|Just your code/container|Just your function code|App config + container image|Node pools, workload configuration, cluster policy|
|**Scale-to-zero**|No|No (lowest tier still running)|Yes (Consumption)|Yes|No (nodes always running, though can scale pool to 0)|
|**Cold start risk**|N/A (always on)|Minimal|Yes on Consumption|Minimal (varies by config)|Minimal (pods reschedule fast if capacity exists)|
|**Best fit**|Legacy apps, full OS control, licensing needs, specialized hardware|Standard web apps/APIs with minimal ops|Event-driven, bursty, short-lived logic|Microservices/event-driven apps wanting container packaging without K8s ops|Complex multi-service systems needing fine-grained orchestration control, custom scheduling, or a portable Kubernetes footprint|
|**Kubernetes access**|N/A|N/A|N/A|No direct API access|Full API access|

**A simple heuristic:** start as far right on this table (AKS) as your requirements truly demand, not as far right as you _could_ go. Most teams overestimate how much orchestration control they need and end up carrying Kubernetes operational cost they didn't have to. If App Service, Functions, or Container Apps satisfy the requirement, they're almost always the lower-total-cost choice in engineering time, even if AKS looks more "impressive" on paper.