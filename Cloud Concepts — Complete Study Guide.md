
---

## PART 1: Service Models

**Big picture spectrum** — how much YOU manage vs how much the PROVIDER manages:

```
On-Prem → IaaS → CaaS → PaaS → FaaS → SaaS
[You manage everything] ────────────► [Provider manages everything]
```

---

### 🔹 IaaS — Infrastructure as a Service

🔑 **Keyword:** _"Rent an empty apartment"_

- **Definition:** You rent raw compute, storage, and networking. It's virtualized hardware — nothing else.
- **You manage:** OS, patches, runtime, apps, data, network config
- **Provider manages:** Physical servers, virtualization/hypervisor, physical network, datacenter
- **Azure examples:** Virtual Machines, Managed Disks, Virtual Network
- **Analogy:** Empty apartment — walls & plumbing exist, you bring your own furniture (OS + apps)

---

### 🔹 PaaS — Platform as a Service

🔑 **Keyword:** _"Rent a furnished apartment"_

- **Definition:** Provider manages the OS + runtime. You just bring your code.
- **You manage:** Application code, app configuration, data
- **Provider manages:** OS, runtime, middleware, scaling infra, physical layers
- **Azure examples:** Azure App Service, Azure SQL Database
- **Analogy:** Furnished apartment — electricity/water work already, just move your stuff in

---

### 🔹 SaaS — Software as a Service

🔑 **Keyword:** _"Stay in a hotel"_

- **Definition:** Fully finished software product. You just use it.
- **You manage:** Your data, your users, basic app settings
- **Provider manages:** Literally everything else — app, runtime, OS, infra
- **Azure/Microsoft examples:** Microsoft 365, Outlook, Teams, Dynamics 365
- **Analogy:** Hotel room — housekeeping, maintenance, everything handled for you

---

### 🔹 FaaS — Function as a Service (serverless compute)

🔑 **Keyword:** _"Vending machine — pay per use"_

- **Definition:** You write small, single-purpose functions triggered by events. No server thinking at all — not even app lifecycle.
- **You manage:** Just the function code + its trigger/config, data
- **Provider manages:** Everything — server, OS, runtime, auto-scaling (even scales to zero)
- **Azure example:** Azure Functions
- **Analogy:** Vending machine — you don't own or run it, you just use it and pay per snack (per execution)
- **Key trait:** Event-driven, stateless, billed per execution time

---

### 🔹 CaaS — Container as a Service

🔑 **Keyword:** _"Shipping company for your containers"_

- **Definition:** You package your app in containers (Docker-style). Provider manages the orchestration engine (scheduling, scaling, healing).
- **You manage:** Container images, app/workload configuration, deployments
- **Provider manages:** Orchestration control plane, clustering infra
- **Azure example:** Azure Kubernetes Service (AKS), Azure Container Instances (ACI)
- **Analogy:** Shipping company — you pack the container (your app), they move & track it
- **Key trait:** Portability — same container runs anywhere (local, cloud, another cloud)

---

### 📊 Quick Comparison Table

|Model|You Manage|Provider Manages|Example|
|---|---|---|---|
|IaaS|OS, runtime, apps, data|Hardware, virtualization|Azure VMs|
|CaaS|Containers, workloads|Orchestration engine|AKS|
|PaaS|App code, data|OS, runtime, scaling|App Service|
|FaaS|Function code only|Everything else|Azure Functions|
|SaaS|Data, users|Entire application|Microsoft 365|

---

---

## PART 2: Deployment Models

### 🌐 Public Cloud

🔑 **Keyword:** _"Shared, internet-based, no hardware ownership"_

- **Definition:** Infrastructure owned & run by a third party (e.g., Microsoft), delivered over the internet, shared across multiple customers (multi-tenant).
- **Example:** A standard Azure subscription
- **Pros:** No upfront hardware cost (CapEx→OpEx), scales fast, pay-as-you-go
- **Cons:** Less control over underlying infra

---

### 🔒 Private Cloud

🔑 **Keyword:** _"Dedicated, single-tenant, more control"_

- **Definition:** Cloud infra used exclusively by ONE organization — can live on-prem or be hosted by a third party, but it's not shared.
- **Example:** Azure Local (formerly Azure Stack HCI), Azure Stack Hub
- **Pros:** Full control, meets strict compliance/regulatory needs
- **Cons:** Higher cost, you manage more of the stack yourself

---

### 🔀 Hybrid Cloud

🔑 **Keyword:** _"On-prem + public cloud, connected"_

- **Definition:** Combines public + private cloud so apps/data can move between them.
- **Example:** Azure Arc connecting an on-prem datacenter to Azure services
- **Use case:** Data residency laws, gradual migration, "cloud bursting" for extra capacity during spikes

---

### 🌍 Multi-Cloud

🔑 **Keyword:** _"More than one cloud provider"_

- **Definition:** Using 2+ cloud providers at once (e.g., Azure + AWS + GCP).
- **Example:** Compute on Azure, storage on AWS S3
- **Use case:** Avoid vendor lock-in, use best-of-breed services, redundancy
- **Note:** Azure Arc can manage multi-cloud AND hybrid resources from one control plane

---

### 📊 Quick Comparison Table

|Model|Ownership|Tenancy|Best For|
|---|---|---|---|
|Public|Provider|Multi-tenant|Cost efficiency, scale|
|Private|Org (or dedicated host)|Single-tenant|Compliance, control|
|Hybrid|Mixed|Mixed|Data residency, phased migration|
|Multi-Cloud|Multiple providers|Mixed|Avoiding lock-in, redundancy|

---

---

## PART 3: Shared Responsibility Model

🔑 **Core rule:** _Responsibility shifts toward the provider as you move from IaaS → PaaS → SaaS — but some things NEVER shift._

### ✅ Always YOURS (no matter the model)

- Data
- Endpoints/devices
- Accounts & identities
- Access management

### ✅ Always the PROVIDER's (no matter the model)

- Physical datacenter
- Physical network
- Physical hosts

### 🔄 What shifts in between

- Operating system
- Network controls
- Applications/middleware
- Identity infrastructure (partially)

---

### 📊 General Responsibility Table

|Layer|On-Prem|IaaS|PaaS|SaaS|
|---|---|---|---|---|
|Data & classification|You|You|You|You|
|Endpoints|You|You|You|You|
|Accounts & access mgmt|You|You|You|You|
|Identity infrastructure|You|You|Shared|Shared|
|Application|You|You|Shared|Provider|
|Network controls|You|Shared|Provider|Provider|
|Operating system|You|You|Provider|Provider|
|Physical hosts|You|Provider|Provider|Provider|
|Physical network|You|Provider|Provider|Provider|
|Physical datacenter|You|Provider|Provider|Provider|

---

## 🎯 Applied to the 5 Azure Services

### 1️⃣ Azure Virtual Machine (IaaS)

🔑 _"You patch the OS"_

- **Microsoft manages:** Physical datacenter, physical network, physical hosts, the hypervisor
- **You manage:** OS patching/updates, runtime, applications, data, network security groups, identity/access

### 2️⃣ AKS — Azure Kubernetes Service (CaaS)

🔑 _"Microsoft runs the control room, you run the floor"_

- **Microsoft manages:** Kubernetes **control plane** (API server, scheduler) at no extra cost, physical infra
- **You manage:** **Worker node pools** (VMs — OS patching, though Azure offers auto-upgrade tooling), container images, workload/app configuration, in-cluster RBAC, scaling rules

### 3️⃣ Azure SQL Database (PaaS)

🔑 _"You just manage your data & queries"_

- **Microsoft manages:** Underlying OS, SQL engine patching, automated backups, high availability, physical infra
- **You manage:** Database schema, the data itself, access/firewall rules, query & performance tuning

### 4️⃣ Azure App Service (PaaS)

🔑 _"You just deploy your code"_

- **Microsoft manages:** OS, runtime/framework patching, load balancing, scaling infrastructure, physical infra
- **You manage:** Application code, app settings/configuration, custom domains, data

### 5️⃣ Azure Functions (FaaS)

🔑 _"You write the function, nothing else"_

- **Microsoft manages:** Server, OS, runtime, automatic scaling (down to zero), physical infra
- **You manage:** Function code, triggers/bindings, data

---

### 📊 Side-by-Side Summary

|Service|Model|Microsoft Manages|You Manage|
|---|---|---|---|
|Virtual Machine|IaaS|Hardware, hypervisor|OS, runtime, apps, data, network|
|AKS|CaaS|K8s control plane, infra|Node OS, containers, workload config|
|Azure SQL DB|PaaS|OS, DB engine, backups, HA|Schema, data, access, tuning|
|App Service|PaaS|OS, runtime, scaling|App code, config, data|
|Azure Functions|FaaS|Everything except code|Function code, triggers|

---

---

## 🧠 Quick Recall Cheat Sheet

|Term|One-Line Keyword|
|---|---|
|IaaS|Rent an empty apartment|
|PaaS|Rent a furnished apartment|
|SaaS|Stay in a hotel|
|FaaS|Vending machine, pay per use|
|CaaS|Shipping company for containers|
|Public Cloud|Shared, no hardware ownership|
|Private Cloud|Dedicated, single-tenant|
|Hybrid Cloud|On-prem + public, connected|
|Multi-Cloud|2+ providers at once|
|Shared Responsibility|Data & identity = always yours; physical layer = always provider's|