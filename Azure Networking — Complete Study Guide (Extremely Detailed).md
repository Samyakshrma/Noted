

🔑 **Mental map — how traffic flows through everything below:**

```
Internet
   │
[Front Door / Traffic Manager]     ← global, optional, Part 5
   │
[Application Gateway + WAF]        ← regional, Layer 7, Part 5
   │
[Load Balancer]                    ← Layer 4, Part 5
   │
[NSG]  →  [VM inside a Subnet, inside a VNet]   ← Parts 1 & 4
   │
[Route Table / UDR]  decides the path           ← Part 2
   │
[VNet Peering / VPN Gateway / ExpressRoute]  →  on-prem or other VNets   ← Part 3
```

> 🆕 A few sections tagged **🆕 Added** cover essential companion topics not in your original list — flagged so you know what's extra.

---

---

## PART 1: Virtual Networks (VNets) Fundamentals

### 🔹 Virtual Network (VNet)

🔑 **Keyword:** _"Your own private, isolated network in Azure"_

- **Definition:** The fundamental building block for private networking in Azure — an isolated network where you control IP ranges, subnets, routing, and security.

---

### 🔹 CIDR

🔑 **Keyword:** _"Notation for defining an IP address range"_

- **Definition:** Classless Inter-Domain Routing — written as `10.0.0.0/16`. The number after the slash = how many bits are **fixed** (the network portion); the rest are usable for hosts.
- **Formula:** Total addresses = `2^(32 - prefix)`
- **Examples:**
    - `/16` → 65,536 addresses (large VNet)
    - `/24` → 256 addresses (typical subnet)
    - `/29` → 8 addresses (tiny — e.g., a gateway subnet)
- **Rule of thumb:** Smaller number after the slash = bigger address range

---

### 🔹 Subnets

🔑 **Keyword:** _"Segments carved out of a VNet's address space"_

- **Definition:** A range of IPs within the VNet, used to logically separate resources (e.g., web tier, app tier, data tier).
- **Rules:**
    - Subnet CIDR must fall **inside** the VNet's CIDR range
    - Subnets **cannot overlap** with each other
- **Example:** VNet `10.0.0.0/16` → Subnet-Web `10.0.1.0/24`, Subnet-DB `10.0.2.0/24`

---

### 🔹 IP Addressing

🔑 **Keyword:** _"How resources get identified on the network"_

- **Private IP:** Internal-only, used within the VNet / peered networks / VPN — not reachable from the internet
- **Public IP:** Internet-routable, needed for a resource to be reached from (or reach out to) the internet directly
- **Allocation method:**
    - **Dynamic** — assigned automatically, can change if the resource is stopped/deallocated
    - **Static** — fixed, never changes

> 🆕 **Added — Public IP SKUs**
> 
> ||Basic|Standard|
> |---|---|---|
> |Security|Open by default|Secure by default (must explicitly allow via NSG)|
> |Zone redundancy|❌ No|✅ Yes|
> |Required by|Legacy resources|Standard Load Balancer, Azure Firewall, App Gateway v2, Bastion|
> |Status|Being retired|Current standard|

> 🆕 **Added — Reserved addresses in every Azure subnet** Azure reserves **5 IPs** per subnet: the network address, the default gateway, two for Azure DNS mapping, and the broadcast address. Example: in `10.0.1.0/24`, `.0`, `.1`, `.2`, `.3`, and `.255` are reserved → 251 usable addresses.

---

### 🔹 DNS

🔑 **Keyword:** _"Translates names to IP addresses"_

- **Azure-provided DNS:** Default, automatic — basic name resolution for resources within the VNet
- **Azure DNS (public zones):** Host your own public domain's DNS records on Azure's infrastructure
- **Custom DNS:** Point the VNet to your own DNS servers instead (common in hybrid setups pointing to on-prem DNS)
- _(Private DNS Zones — for resolving Private Endpoints — covered in full in Part 6)_

---

### 🔹 DHCP

🔑 **Keyword:** _"Automatically hands out private IPs to resources"_

- **Definition:** Azure automatically assigns private IPs to network interfaces (NICs) within a subnet via DHCP — you never run your own DHCP server.
- **Note:** Even a "static" private IP in Azure is technically DHCP-_reserved_ — Azure guarantees that same address is always handed to that NIC.

---

---

## PART 2: Routing

### 🔹 System Routes

🔑 **Keyword:** _"Default routes Azure creates automatically"_

- **Definition:** Azure automatically creates default routes for every subnet, enabling traffic to flow: within the VNet, VNet-to-Internet, across peered VNets, and to on-prem via VPN/ExpressRoute.
- **Key fact:** Can't be edited directly, but **can be overridden** using User Defined Routes.

---

### 🔹 User Defined Routes (UDR)

🔑 **Keyword:** _"Custom routes YOU define to override the defaults"_

- **Definition:** Manually created routes in a route table, forcing traffic through a specific path instead of the Azure default — most commonly through a firewall or network virtual appliance (NVA).
- **Example:** Force all outbound traffic (`0.0.0.0/0`) through Azure Firewall's private IP for inspection before it leaves the VNet.

---

### 🔹 BGP (Border Gateway Protocol)

🔑 **Keyword:** _"Dynamic route exchange, used with VPN/ExpressRoute"_

- **Definition:** A routing protocol that automatically exchanges routes between your on-prem network and your Azure VNet over a VPN Gateway or ExpressRoute connection — routes update automatically as the network changes, no manual entry needed.
- **Used for:** Optional with Site-to-Site VPN; **required** with ExpressRoute

---

### 🔹 Route Tables

🔑 **Keyword:** _"Container holding a set of routes, attached to subnets"_

- **Definition:** A collection of routes (system + user-defined) associated with one or more subnets — determines exactly where traffic from that subnet is sent.
- **Each route has:**
    - **Address prefix** — the destination
    - **Next hop type** — Virtual Network Gateway, Virtual Network, Internet, Virtual Appliance, or None (blackhole/drop)

---

---

## PART 3: Connectivity

### 🔹 VNet Peering

🔑 **Keyword:** _"Connects two VNets directly, privately, over Microsoft's backbone"_

- **Definition:** Connects two VNets so resources in each communicate using **private IPs**, over Microsoft's backbone — never touching the public internet.
- **Key trait — non-transitive:** If VNet A peers with VNet B, and VNet B peers with VNet C, **A cannot automatically talk to C**. Each peering relationship is independent.
- **Default scope:** Same region

---

### 🔹 Global Peering

🔑 **Keyword:** _"VNet peering, but across different Azure regions"_

- **Definition:** The exact same concept as VNet Peering, but works **across regions** — still uses Microsoft's private backbone, never the public internet.
- **Example:** Peer a VNet in East US with a VNet in West Europe.

---

### 🔹 VPN Gateway

🔑 **Keyword:** _"Encrypted tunnel over the public internet, connecting Azure to somewhere else"_

- **Definition:** A virtual network gateway that sends **encrypted (IPsec)** traffic over the **public internet** between an Azure VNet and an on-prem location (or another VNet).

**🔸 Site-to-Site VPN** 🔑 _"Connects your entire on-prem network to Azure"_

- Persistent encrypted tunnel between an on-prem VPN device and the Azure VPN Gateway
- **Use case:** Permanently connecting a corporate office network to Azure

**🔸 Point-to-Site VPN** 🔑 _"Connects a single device to Azure"_

- An individual client machine connects to the VNet over an encrypted tunnel using client software
- **Use case:** A remote employee securely reaching Azure resources from their laptop, without a full site connection

---

### 🔹 ExpressRoute

🔑 **Keyword:** _"Private, dedicated connection — never touches the public internet"_

- **Definition:** A dedicated private connection between on-prem and Azure, provisioned through a connectivity provider — completely bypasses the public internet.
- **Benefits:** More reliable, faster, lower and more consistent latency, higher security & bandwidth than VPN
- **Use case:** Enterprises with mission-critical, high-volume, or highly sensitive on-prem ↔ Azure traffic

> 🆕 **Added — Azure Virtual WAN** 🔑 _"Hub bringing together VPN, ExpressRoute, and VNets into one managed architecture"_ A networking service that unifies VPN, ExpressRoute, and VNet connections into a single hub-and-spoke topology, centrally managed — used for large-scale, multi-region enterprise networks.

### 📊 VPN Gateway vs ExpressRoute

||VPN Gateway|ExpressRoute|
|---|---|---|
|Path|Public internet (encrypted)|Private dedicated circuit|
|Setup speed|Fast (minutes–hours)|Slower (days–weeks, needs a provider)|
|Reliability/Latency|Variable, internet-dependent|Consistent, low-latency|
|Cost|Lower|Higher|
|Best for|Small–medium scale, quick setup, DR backup|Large-scale, mission-critical workloads|

---

---

## PART 4: Security

### 🔹 NSG (Network Security Group)

🔑 **Keyword:** _"Firewall rules attached to a subnet or NIC"_

- **Definition:** A set of allow/deny rules filtering traffic to/from Azure resources — can be attached at the subnet level and/or individual NIC level.

**Inbound rules** 🔑 _"Controls traffic COMING IN"_ — evaluated for traffic arriving at the resource

**Outbound rules** 🔑 _"Controls traffic GOING OUT"_ — evaluated for traffic leaving the resource

**Priorities** 🔑 _"Lower number = evaluated first, and wins"_

- Each rule has a priority (100–4096). Azure evaluates from **lowest to highest**; the first matching rule applies and the rest are skipped for that traffic.
- **Example:** Priority 100 "Allow 443 from Internet" is checked _before_ Priority 200 "Deny all from Internet" → HTTPS gets through, everything else gets denied by the later rule.
- **Default rules:** NSGs ship with built-in defaults (Allow VNet in/outbound, Allow Azure Load Balancer inbound, Deny all other inbound) at very high priority numbers (65000+) — your own lower-numbered rules can override them.

---

### 🔹 ASG (Application Security Group)

🔑 **Keyword:** _"Group VMs by role, reference the group in NSG rules instead of IPs"_

- **Definition:** Lets you logically group VMs/NICs (e.g., "WebServers", "DBServers") and reference that group **name** in NSG rules instead of hardcoding IP addresses.
- **Benefit:** Rules stay valid as VMs are added/removed from the group — zero IP management.
- **Example:** NSG rule → _"Allow traffic from ASG-WebServers to ASG-DBServers on port 1433"_

---

### 🔹 Azure Firewall

🔑 **Keyword:** _"Managed, stateful, cloud-native firewall for the whole VNet"_

- **Definition:** Fully managed Firewall-as-a-Service — filters traffic centrally across VNets/subscriptions, with built-in high availability and auto-scaling.

**Firewall Policies** 🔑 _"The rule sets controlling how Azure Firewall behaves"_

- Centralized config object defining NAT rules, network rules, application rules, threat intelligence settings, TLS inspection — can be **shared** across multiple firewall instances.

**DNAT (Destination NAT)** 🔑 _"Translate a public IP+port to a private IP+port — INBOUND"_

- Lets internet traffic reach a private resource by translating the destination address.
- **Example:** Forward the Firewall's public IP `:3389` to a specific VM's private IP `:3389` — publishing an internal service through the firewall.

**SNAT (Source NAT)** 🔑 _"Translate a private IP to a public IP — OUTBOUND"_

- Translates the source address of outbound traffic from private → public, so return traffic routes back correctly.
- **Example:** A VM with only a private IP needs internet access — the Firewall SNATs its traffic using the Firewall's own public IP.

> 🆕 **Added — NAT Gateway** 🔑 _"Dedicated, scalable outbound-only internet connectivity for a subnet"_ A fully managed NAT service giving VMs in a subnet reliable outbound internet access — avoids the SNAT port exhaustion issues a Load Balancer can hit at scale. Use it when you need outbound-only connectivity without standing up a full Firewall or Load Balancer.

> 🆕 **Added — Azure Bastion** 🔑 _"Jump box as a managed service — secure browser-based RDP/SSH"_ PaaS service deployed in your VNet giving secure RDP/SSH access to VMs directly through the Azure Portal over TLS — no public IP or open RDP(3389)/SSH(22) ports on the VM ever needed.

> 🆕 **Added — Azure DDoS Protection** 🔑 _"Absorbs/mitigates Distributed Denial of Service attacks"_
> 
> - **Basic:** Free, automatically enabled for every Azure resource
> - **Standard/IP Protection:** Enhanced mitigation tuned to your VNet's actual traffic patterns, cost-protection guarantee, richer telemetry & alerting

---

---

## PART 5: Load Balancing — Understand When to Use Each ⭐

### 🔹 Azure Load Balancer — Layer 4

🔑 **Keyword:** _"Distributes TCP/UDP traffic by IP + port"_

- **Definition:** Operates at Layer 4 (Transport) — distributes traffic across backend VMs based on IP/port, without looking at the actual content.
- **Types:** Public (internet-facing) vs Internal (private traffic only)
- **Use case:** Fast, simple distribution for non-HTTP(S)-aware traffic (a custom TCP service, database cluster, etc.)

---

### 🔹 Application Gateway — Layer 7

🔑 **Keyword:** _"Web traffic router — understands HTTP(S) content"_

- **Definition:** Operates at Layer 7 (Application) — routes based on URL path, host headers, cookies; also handles SSL/TLS termination.
- **Features:** Path-based routing, host-based routing, SSL offload, session affinity, autoscaling
- **Use case:** Web apps needing smart HTTP routing (e.g., `/images/*` → Pool A, `/api/*` → Pool B)

**WAF (Web Application Firewall)** 🔑 _"Add-on protecting web apps from common exploits"_

- Enabled on Application Gateway (or Front Door) — protects against OWASP Top 10 threats (SQL injection, XSS, etc.) via managed rule sets.
- **Use case:** Any public-facing web app needing protection from common web attacks.

---

### 🔹 Azure Front Door

🔑 **Keyword:** _"Global entry point — Layer 7, at the EDGE, closest to users"_

- **Definition:** A global Layer 7 load balancer + CDN + WAF combined — routes users to the closest/healthiest backend across regions using Microsoft's global edge network (anycast).
- **Features:** Global HTTP(S) load balancing, SSL offload, caching (CDN), WAF, cross-region path-based routing
- **Use case:** Multi-region web apps needing global load balancing, fast content delivery, and regional failover

---

### 🔹 Traffic Manager

🔑 **Keyword:** _"DNS-based global routing — never touches the actual traffic"_

- **Definition:** A DNS-level load balancer — resolves the client to the right endpoint's IP based on a routing method, then the client connects **directly** to that endpoint (Traffic Manager never proxies the data itself).
- **Routing methods:** Priority, Weighted, Performance (lowest latency), Geographic, Multivalue, Subnet
- **Use case:** Any protocol (not just HTTP) needing global routing, or DNS-only failover without proxying traffic through Microsoft's network

---

### 📊 "When to Use Each" — Decision Table

|Service|OSI Layer|Scope|Best For|
|---|---|---|---|
|Load Balancer|Layer 4 (TCP/UDP)|Regional|Simple, fast, non-HTTP traffic|
|Application Gateway|Layer 7 (HTTP/S)|Regional|Smart HTTP routing within a region, WAF|
|Front Door|Layer 7 (HTTP/S)|Global|Multi-region web apps, CDN + global WAF|
|Traffic Manager|DNS (any protocol)|Global|Non-HTTP global routing, DNS-only failover|

**Quick decision logic:**

- Regional + non-HTTP traffic → **Load Balancer**
- Regional + HTTP with path/host routing → **Application Gateway**
- Global + web app + CDN/caching needed → **Front Door**
- Global + any protocol + no traffic proxying wanted → **Traffic Manager**
- Commonly layered together: **Front Door** (global) → **Application Gateway** (regional, WAF) → **Load Balancer** (VM pool)

---

---

## PART 6: Private Connectivity

### 🔹 Service Endpoints

🔑 **Keyword:** _"Extend your VNet's identity to a PaaS service, over Microsoft's backbone"_

- **Definition:** Extends the VNet's private address space to a PaaS service (like Storage or SQL) — traffic takes Microsoft's backbone instead of the public internet, and the PaaS service can be locked to accept traffic only from that specific subnet.
- **Key limitation:** The PaaS service **still has a public IP endpoint** — you're just restricting _which_ VNets/subnets can reach it and giving it a private path. It does **not** put the resource on a private IP inside your VNet.

---

### 🔹 Private Endpoint

🔑 **Keyword:** _"Gives the PaaS service an actual private IP inside YOUR VNet"_

- **Definition:** A network interface with a private IP address from your VNet's own address space, connected directly to a specific PaaS resource — the service now appears to live **inside** your VNet.
- **Key benefit over Service Endpoint:** Fully private, zero public IP exposure, and works across peered VNets and on-prem (via VPN/ExpressRoute) too.

---

### 🔹 Private Link

🔑 **Keyword:** _"The underlying framework that POWERS Private Endpoints"_

- **Definition:** The Azure platform capability enabling private connectivity to a PaaS (or your own) service via a Private Endpoint — all traffic stays on Microsoft's private network, never the public internet.
- **Relationship:** Private Link = the underlying framework/service. Private Endpoint = the actual NIC/connection object you create in your VNet using that framework.

---

### 🔹 Private DNS Zones

🔑 **Keyword:** _"DNS resolution for private resources, scoped to your VNet(s)"_

- **Definition:** A DNS zone linked to one or more VNets, resolving domain names to **private** IP addresses — essential for Private Endpoints, since the PaaS service's public DNS name needs to resolve to its new private IP when queried from inside your VNet.
- **Example:** `mystorageaccount.privatelink.blob.core.windows.net` resolves to the Private Endpoint's private IP instead of the public one.

---

### 📊 Service Endpoint vs Private Endpoint

||Service Endpoint|Private Endpoint|
|---|---|---|
|Gets a private IP in your VNet?|❌ No|✅ Yes|
|PaaS service still has a public IP?|✅ Yes (access restricted)|❌ No — fully private|
|Works from on-prem/peered VNet?|❌ No (VNet-local only)|✅ Yes|
|Needs DNS changes?|No|Yes (Private DNS Zone)|
|Best for|Quick, simple VNet-only restriction|Full isolation, hybrid/cross-network access|

---

---

## 🧠 Quick Recall Cheat Sheets

**VNet Fundamentals**

|Term|One-Line Keyword|
|---|---|
|CIDR|Notation defining an IP range|
|Subnet|Segment carved from a VNet|
|Public IP|Internet-routable address|
|Private IP|Internal-only address|
|DNS|Translates names to IPs|
|DHCP|Auto-assigns private IPs|

**Routing**

|Term|One-Line Keyword|
|---|---|
|System Routes|Default routes Azure creates automatically|
|UDR|Custom route overriding the default|
|BGP|Dynamic route exchange for VPN/ExpressRoute|
|Route Table|Container of routes attached to a subnet|

**Connectivity**

|Term|One-Line Keyword|
|---|---|
|VNet Peering|Direct private link between 2 VNets, same region|
|Global Peering|VNet peering across regions|
|Site-to-Site VPN|Connects your whole on-prem network|
|Point-to-Site VPN|Connects a single device|
|ExpressRoute|Private dedicated circuit, no public internet|
|Virtual WAN|Hub unifying VPN/ExpressRoute/VNets|

**Security**

|Term|One-Line Keyword|
|---|---|
|NSG|Allow/deny rules on subnet or NIC|
|Priority|Lowest number evaluated first, wins|
|ASG|Group VMs by role for NSG rules|
|Azure Firewall|Managed stateful firewall for the VNet|
|DNAT|Public IP:port → private IP:port (inbound)|
|SNAT|Private IP → public IP (outbound)|
|NAT Gateway|Dedicated outbound-only connectivity|
|Azure Bastion|Browser-based RDP/SSH, no public IP|
|DDoS Protection|Mitigates denial-of-service attacks|

**Load Balancing**

|Term|One-Line Keyword|
|---|---|
|Load Balancer|Layer 4, regional, IP+port based|
|Application Gateway|Layer 7, regional, HTTP-aware|
|WAF|Blocks common web exploits|
|Front Door|Layer 7, global, edge + CDN|
|Traffic Manager|DNS-only, global, any protocol|

**Private Connectivity**

|Term|One-Line Keyword|
|---|---|
|Service Endpoint|Private path, PaaS keeps public IP|
|Private Endpoint|PaaS gets a private IP in your VNet|
|Private Link|Framework powering Private Endpoints|
|Private DNS Zone|Resolves names to private IPs|