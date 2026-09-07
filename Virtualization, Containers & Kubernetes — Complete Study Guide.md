
---

## PART 1: Hypervisors

🔑 **Core idea:** A hypervisor is software that lets **one physical machine** pretend to be **many machines**, by abstracting/dividing up CPU, memory, disk, network.

```
Physical Hardware → Hypervisor → VM1, VM2, VM3... (each thinks it owns the whole machine)
```

---

### 🔹 Type 1 Hypervisor — "Bare-Metal"

🔑 **Keyword:** _"No landlord — runs straight on hardware"_

- **Definition:** Installed directly on physical hardware. IS the operating system layer itself.
- **Performance:** Best — no middleman OS to slow things down
- **Used for:** Production, datacenters, cloud providers
- **Examples:** Hyper-V, VMware ESXi, Xen
- **Azure:** Uses a custom Type 1 hypervisor (Microsoft Hyper-V based) under every Azure VM
- **Analogy:** Building your house directly on the land — no landlord in between



```
              Server / PC hardware
                    ↓
             UEFI / BIOS firmware
                    ↓
          Type 1 Hypervisor (ESXi, Xen, etc.)
                    ↓
        ┌───────────┼───────────┐
        ↓           ↓           ↓
     Linux VM    Windows VM   Linux VM
        ↓           ↓           ↓
   Linux kernel Windows kernel Linux kernel
        ↓           ↓           ↓
      Apps        Apps        Apps
```


---

### 🔹 Type 2 Hypervisor — "Hosted"

🔑 **Keyword:** _"Runs as an app on top of an OS"_

- **Definition:** Installed like a regular application on top of an existing host OS (Windows/Mac/Linux)
- **Performance:** Lower — has to go through the host OS layer
- **Used for:** Desktop/dev/testing, running a VM on your laptop
- **Examples:** VirtualBox, VMware Workstation, Parallels
- **Analogy:** Renting a room inside someone else's house — the host OS is the house

```
                Laptop hardware
                       ↓
                  UEFI firmware
                       ↓
                  GRUB bootloader
                       ↓
                Linux kernel
                       ↓
                 Ubuntu OS
                       ↓
                  VirtualBox
                (Type 2 hypervisor)
                       ↓
              ┌────────┴────────┐
              ↓                 ↓
          Windows VM         Linux VM
              ↓                 ↓
       Windows kernel      Linux kernel
              ↓                 ↓
            Apps               Apps
```


---

### 📊 Type 1 vs Type 2

||Type 1 (Bare-Metal)|Type 2 (Hosted)|
|---|---|---|
|Runs on|Hardware directly|Existing host OS|
|Performance|High|Lower|
|Use case|Production, cloud|Dev, testing, laptops|
|Examples|Hyper-V, ESXi, Xen|VirtualBox, VMware Workstation|

---

---

## PART 2: Virtual Machines — Resource Allocation

### 🔹 CPU Allocation

🔑 **Keyword:** _"vCPUs carved from real cores"_

- **Definition:** The hypervisor hands out **virtual CPUs (vCPUs)** to each VM and schedules them onto real **physical CPU cores (pCPU)**.
- **Overcommitment:** You can assign MORE total vCPUs than physical cores exist — hypervisor time-slices between VMs.
- **Key terms:** vCPU, pCPU, CPU scheduling, oversubscription
- **Azure:** VM size (e.g., `D4s_v5`) defines exactly how many vCPUs you get

---

### 🔹 Memory Allocation

🔑 **Keyword:** _"RAM sliced and handed out per VM"_

- **Definition:** Hypervisor allocates physical RAM to each VM.
- **Static memory:** Fixed amount reserved, always available
- **Dynamic memory:** Amount can grow/shrink based on VM's actual need (Hyper-V "Dynamic Memory")
- **Memory ballooning:** Hypervisor "reclaims" unused RAM from a VM and gives it to another VM that needs it
- **Overcommitment:** Like CPU, you can promise more RAM than physically exists (risky if all VMs demand max at once)

---

### 🔹 NUMA (Non-Uniform Memory Access)

🔑 **Keyword:** _"Not all RAM is the same distance from the CPU"_

- **Definition:** In multi-socket servers, memory is split into **nodes**, each physically closer to one CPU socket.
- **Local access** (same node) = fast
- **Remote access** (different node) = slower
- **Why it matters:** If a large VM's vCPUs/memory get spread across multiple NUMA nodes, performance can drop due to remote memory calls.
- **Azure:** Larger VM sizes are designed around NUMA node boundaries — Azure tries to fit a VM inside one NUMA node when possible
- **Analogy:** Grabbing a snack from your own kitchen (fast) vs from a neighbor's house (slow)

---

### 🔹 Virtual Disks

🔑 **Keyword:** _"Files that behave like hard drives"_

- **Definition:** Storage presented to a VM as a "disk," but really backed by a file (or block storage) on the host.
- **Formats:** VHD / VHDX (Hyper-V, Azure), VMDK (VMware)
- **Types:** Fixed-size (space reserved upfront) vs Dynamically-expanding (grows as needed)
- **Azure:** Managed Disks — OS disk + optional Data disks; tiers = Standard HDD, Standard SSD, Premium SSD, Ultra Disk

---

---

## PART 3: Containers

🔑 **Core idea vs VMs:** VMs virtualize the **hardware** (each VM has its own full OS). Containers virtualize the **OS** (all containers share the host's kernel) → much lighter and faster.

```
VM:        Hardware → Hypervisor → Guest OS (full) → App
Container: Hardware → Host OS → Container Engine → App (shares host kernel)
```

---

### 🔹 Docker

🔑 **Keyword:** _"Package the app + everything it needs, in one box"_

- **Definition:** Platform to build, ship, and run containers. A Docker **image** = app code + dependencies + config, bundled together.
- **Why it's popular:** "Works on my machine" problem solved — same image runs identically anywhere
- **Key parts:** Dockerfile (recipe to build image), Image (packaged app), Container (running instance of an image)

---

### 🔹 OCI Standard (Open Container Initiative)

🔑 **Keyword:** _"The universal rulebook for containers"_

- **Definition:** Open industry standard defining how container **images** and **runtimes** should behave — so any OCI-compliant tool can run any OCI-compliant image.
- **Why it matters:** Prevents vendor lock-in — an image built with Docker can run on containerd, Podman, CRI-O, etc.
- **Main specs:** Image spec, Runtime spec, Distribution spec

---

### 🔹 Container Runtime

🔑 **Keyword:** _"The engine that actually starts/stops the container"_

- **Definition:** The low-level software that pulls images, sets up isolation (namespaces/cgroups), and runs the container process.
- **Examples:** `containerd`, `CRI-O`, Docker Engine (uses containerd underneath)
- **Kubernetes connection:** Kubernetes talks to any runtime via **CRI** (Container Runtime Interface) — pluggable by design

---

### 🔹 Container Networking

🔑 **Keyword:** _"How containers reach each other and the outside world"_

- **Docker networking modes:**
    - `bridge` (default) — private internal network on the host
    - `host` — container shares host's network directly
    - `none` — fully isolated, no networking
    - `overlay` — connects containers across multiple hosts
- **Kubernetes networking model:** Every **Pod** gets its own unique IP; flat network so all pods can talk to each other directly
- **CNI (Container Network Interface) plugins:** Azure CNI, Calico, Flannel — plug into Kubernetes to implement this networking

---

---

## PART 4: Kubernetes Basics

🔑 **Core idea:** Kubernetes = an **orchestrator**. It doesn't run containers itself — it decides _where_, _how many_, and _keeps them healthy_.

**Object hierarchy (top to bottom):**

```
Deployment → ReplicaSet → Pod → Container(s)
```

---

### 🔹 Pods

🔑 **Keyword:** _"Smallest deployable unit in Kubernetes"_

- **Definition:** Wraps one (or more) containers that share the **same IP address** and storage volumes.
- **Usually:** 1 container per pod — extra "sidecar" containers sometimes added for logging/proxying
- **Analogy:** An apartment (pod) — usually one tenant (container), but roommates (sidecars) share the same address

---

### 🔹 ReplicaSets

🔑 **Keyword:** _"Keeps exactly N copies of a pod alive"_

- **Definition:** Ensures a specified number of identical pod replicas are running at all times. If a pod dies, ReplicaSet creates a new one.
- **Note:** You rarely create these directly — Deployments manage them for you

---

### 🔹 Deployments

🔑 **Keyword:** _"Manages ReplicaSets + gives you rolling updates & rollbacks"_

- **Definition:** Higher-level object that manages ReplicaSets, and adds:
    - Declarative updates (change the image version, it rolls out gradually)
    - Rollback (undo a bad release)
    - Scaling up/down
- **Use case:** The **standard way** to run stateless apps in Kubernetes

---

### 🔹 Services

🔑 **Keyword:** _"A stable address for pods that keep changing"_

- **Problem it solves:** Pods die and get recreated with new IPs constantly — apps need a stable way to find them
- **Definition:** Gives a fixed IP/DNS name + load balances traffic across a group of matching pods
- **Types:**
    - `ClusterIP` — internal-only (default)
    - `NodePort` — exposes the app on a port on every node
    - `LoadBalancer` — provisions a cloud load balancer (e.g., Azure Load Balancer)
    - `ExternalName` — maps to an external DNS name

---

### 🔹 Ingress

🔑 **Keyword:** _"Smart traffic router for HTTP(S) into the cluster"_

- **Definition:** Manages external HTTP/HTTPS access with **routing rules** — path-based (`/api` → service A) or host-based (`shop.site.com` → service B) — plus SSL termination.
- **Why not just use Services?** A `LoadBalancer` Service = one public IP per service (expensive, unmanaged). Ingress = **one entry point** for many services.
- **Requires:** An **Ingress Controller** actually doing the routing — e.g., NGINX Ingress, Azure Application Gateway Ingress Controller (AGIC)

---

### 📊 Kubernetes Objects at a Glance

|Object|Purpose|Analogy|
|---|---|---|
|Pod|Runs container(s), gets an IP|An apartment|
|ReplicaSet|Keeps N pod copies alive|Building maintenance staff|
|Deployment|Manages ReplicaSets + updates|Property manager|
|Service|Stable address + load balancing|Building's front desk/reception|
|Ingress|Smart external traffic routing|The city's road signs directing visitors to the right building|

---

---

## 🧠 Quick Recall Cheat Sheet

|Term|One-Line Keyword|
|---|---|
|Type 1 Hypervisor|Runs directly on hardware|
|Type 2 Hypervisor|Runs on top of a host OS|
|CPU Allocation|vCPUs carved from physical cores|
|Memory Allocation|RAM sliced per VM, can overcommit|
|NUMA|Not all memory is equal distance from CPU|
|Virtual Disk|A file that behaves like a hard drive|
|Docker|Packages app + dependencies together|
|OCI Standard|Universal container format rulebook|
|Container Runtime|Engine that actually runs the container|
|Container Networking|How containers reach each other/outside|
|Pod|Smallest deployable K8s unit|
|ReplicaSet|Keeps N pod copies running|
|Deployment|Manages ReplicaSets + rolling updates|
|Service|Stable IP/DNS for a group of pods|
|Ingress|Smart HTTP(S) router into the cluster|