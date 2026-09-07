


---

## 1. Azure Policy

Azure Policy evaluates your resources against rules ("policy definitions") and either reports on or actively enforces compliance — at any scale from a single resource group up to your entire tenant via management groups.

The four effects you listed are the ones that matter most in practice, and they represent an escalating scale of enforcement:

|Effect|When It Acts|What It Does|
|---|---|---|
|**Audit**|After a resource exists (or on evaluation cycle)|Flags non-compliant resources in the compliance report — **doesn't block anything**. The starting point for any new policy: see the blast radius before you enforce it.|
|**Deny**|At request time, before the resource is created/updated|**Blocks the operation outright** if it would violate the rule — e.g., "reject any storage account created without HTTPS-only enabled."|
|**Modify**|During creation/update|**Actively changes** the request — most commonly used to auto-add or auto-correct tags, or add settings a resource is missing, without blocking the deployment.|
|**DeployIfNotExists (DINE)**|After the resource is created|Checks whether a **related, companion resource** exists (diagnostic settings, a monitoring agent extension, a specific role assignment) and, if it doesn't, **triggers a remediation deployment** to create it. This is the "self-healing" effect — and it requires a managed identity on the policy assignment with enough permission to actually perform that remediation deployment.|

A fifth worth knowing: **Append**, an older, more limited cousin of Modify (adds fields but can't remove/overwrite them the way Modify can) — largely superseded by Modify for new work.

**Key structural concepts:**

- **Policy Definition** — a single rule.
- **Initiative (Policy Set)** — a bundle of related policy definitions grouped together. This is exactly what powers the compliance standards from the Security guide (PCI-DSS, ISO 27001, NIST) — each one is an initiative made of dozens of individual definitions, and your "compliance %" in Defender for Cloud is literally derived from initiative compliance state.
- **Assignment scope** — a definition/initiative is _assigned_ at a management group, subscription, or resource group level, and applies to everything inside that scope, with the option to carve out **exemptions** for specific resources that have a legitimate reason not to comply.
- **Remediation** — Audit/Deny only affect resources going forward. For **Modify** and **DeployIfNotExists**, you can run a **remediation task** retroactively against resources that already existed before the policy was assigned, bringing your existing estate into compliance rather than only enforcing on new resources.

### 2. Blueprints — Understand Conceptually (⚠️ Being Retired)

**Current status (verify before building anything new on this):** Azure Blueprints is being retired. The original retirement date of July 11, 2026 has been **extended to January 31, 2027**, with a **phased retirement that began July 31, 2026** — meaning as of today, Blueprints is already in its wind-down phase. **Don't build new governance on Blueprints.** The concept is still worth understanding, because (a) you'll encounter it in existing environments and older exam material, and (b) its replacement is architecturally cleaner and worth knowing _why_ it's better.

**What Blueprints conceptually did:** it packaged together **ARM templates, policy assignments, RBAC role assignments, and resource group structure** into a single versioned, repeatable artifact — the idea being "define a compliant environment once, then stamp it out consistently across many subscriptions," with built-in resource locking so the deployed environment couldn't drift from the intended state.

**Why it's being replaced:** Blueprints bundled two genuinely different jobs into one service — _storing/versioning a template_ and _assigning/managing/locking the deployment of that template_. The replacement splits these into two purpose-built features that you use together:

|Job Blueprints Did|Modern Replacement|
|---|---|
|Store and version the deployable artifact|**Template Specs** (or just a Git repo of Bicep/ARM)|
|Assign, deploy, manage lifecycle, and lock the deployed resources|**Azure Deployment Stacks**|

You'd typically store your environment definition as a Template Spec, then deploy and govern it as a Deployment Stack — which gives you the same "deny unauthorized changes to what I deployed" locking behavior Blueprints provided, but as a supported, forward-looking service rather than one on a retirement countdown.

---

## 3. Resource Locks

A lock is a subscription/resource-group/resource-scoped setting that overrides even **Owner** permissions to prevent a specific category of accidental action.

|Lock Type|Blocks|Still Allows|
|---|---|---|
|**Read-only (CanNotDelete + ReadOnly)**|Any modification to the resource, including its configuration|Reading/listing the resource|
|**Delete (CanNotDelete)**|Deletion of the resource|All other modifications|

- **Inheritance** — locks cascade downward. A lock placed at the resource group level applies to _every resource inside it_, even ones added later.
- **Who can remove a lock** — only a principal with explicit permission on the `Microsoft.Authorization/locks` action (not just generic Owner/Contributor on the resource itself) can remove a lock — this is deliberate, so a lock genuinely protects against a compromised or careless Owner-level account, not just "everyone except the resource creator."
- **The classic gotcha:** a **ReadOnly** lock can break things you wouldn't expect, because some operations that _feel_ like reads are technically implemented as `POST` requests under the hood — the most common example is listing a storage account's **access keys**, which is blocked by a ReadOnly lock even though "reading a key" sounds harmless. Always test what a ReadOnly lock actually breaks on a given resource type before applying it broadly.

---

## 4. Tagging Strategy — Enterprise-Grade Tagging

Tags are key-value metadata attached to resources, and at enterprise scale they're the backbone of **three separate capabilities** that all depend on the same discipline: cost allocation, automation targeting, and operational ownership.

**A practical enterprise taxonomy usually covers:**

|Tag|Purpose|
|---|---|
|`CostCenter` / `BillingCode`|Chargeback/showback to the right budget owner|
|`Environment`|`Prod` / `Staging` / `Dev` / `Test` — drives automation like "only patch non-prod automatically"|
|`Owner`|Who to contact when something breaks or needs decommissioning|
|`Application` / `Workload`|Groups resources belonging to one logical system, even across resource groups|
|`DataClassification`|Feeds directly into the security controls from the previous guide — e.g., trigger stricter policy for `Confidential`-tagged resources|
|`ExpirationDate`|For temporary resources — pairs with an automated cleanup Logic App to fight resource sprawl|

**Making tags actually stick, at scale:**

- Tags **don't automatically cascade** from a resource group down to the resources inside it by default. The standard pattern is an Azure Policy with the **Modify** effect that inherits a specified tag from the parent resource group (or subscription) onto every resource created inside it — this is how "enterprise-grade" tagging usually gets enforced in practice, rather than trusting every engineer to remember.
- Azure Cost Management separately supports a **tag inheritance** setting specifically for _cost data_ — letting subscription/resource-group tags flow into cost/usage reports even for child resources that were never individually tagged, so chargeback reporting stays accurate without 100% tagging compliance on every resource.
- **A common pitfall:** tag _names_ are case-**in**sensitive (`Environment` and `environment` are treated as the same tag), but tag _values_ are case-**sensitive** (`Prod` and `prod` are different values) — inconsistent casing on values is one of the most common ways enterprise tagging strategies quietly break their own cost reports.
- Enforce the taxonomy with a **Deny** policy for required tags on resource creation, combined with a **Modify** policy for inherited/default values — Audit-only tagging policies tend to decay over time without a harder backstop.

---

## 5. Cost Management

### 5.1 Budgets

Spending thresholds set at subscription, resource group, or management group scope. You define one or more **alert thresholds** (e.g., 80% of budget, 100%, 120% _forecasted_) that fire an **Action Group** — email, webhook, or an automated Logic App that can actually respond (deallocate non-critical resources, post to a Teams channel, open a ticket).

- Budgets can alert on **actual spend to date** or on **forecasted spend** — forecasted alerts are the more useful early-warning signal, since they tell you "at this trajectory, you'll blow the budget by month-end" while there's still time to act.

### 5.2 Cost Analysis

An interactive exploration tool for breaking spend down by service, resource group, tag, location, or time period — the primary way to answer "why did the bill jump this month?" or "which team/tag is actually driving this cost?"

- Supports both accumulated and time-series views for spotting trends vs. one-off spikes.
- Cost data can be **scheduled for export** to a storage account, from which it feeds BI tooling (Power BI) or the open-source **FinOps toolkit** for more sophisticated, cross-team reporting than the built-in portal views support.

### 5.3 Reservations

A **1- or 3-year commitment** to a specific amount of usage (VM family, SQL vCores, Cosmos DB throughput, etc.) in exchange for a substantial discount — often up to ~70%+ versus pay-as-you-go pricing, depending on the service and term length.

- Applies **automatically** to any matching running resource — you don't reconfigure the resource itself, the billing system just recognizes the match and applies the discount.
- **Instance size flexibility** — for many compute reservations, the discount can apply across a _range_ of sizes within the same VM family, not just one exact SKU, giving some built-in flexibility as workloads are resized.
- **Scope** — a reservation can be locked to a single subscription, shared across all subscriptions under a billing account, or applied at a management group level.
- **The trade-off:** it's a real financial commitment. If your workload needs shrink or shift away from the reserved SKU/region, you're less flexible than pay-as-you-go (though Azure does allow exchanges/cancellations within limits and sometimes a fee).

### 5.4 Savings Plans

A **newer, more flexible** commitment model: instead of committing to a specific SKU/region, you commit to a **fixed hourly dollar amount** of compute spend for 1 or 3 years, and the discount applies automatically across a broad range of eligible compute services — VMs, Container Instances, App Service, AKS, Premium Functions — **regardless of region, instance family, or OS**.

||Reservations|Savings Plans|
|---|---|---|
|Commitment shape|Specific resource type/SKU, often scoped to region|Fixed **$/hour** spend, service- and region-agnostic within eligible compute|
|Flexibility|Lower — tied to what you reserved|Higher — automatically applies wherever your compute spend lands|
|Discount depth|Generally deeper for the _exact_ matching usage|Slightly shallower than an equivalent, perfectly-matched reservation|
|Best for|Stable, predictable workloads on a known SKU/region|Organizations whose compute footprint shifts across regions/families over time|

**Practical rule of thumb:** Reservations for the workloads you _know_ won't move (a production database on a fixed SKU for the next three years); Savings Plans for your general, evolving compute footprint where you want the discount without locking in the shape of the infrastructure.

---

## 6. FinOps

FinOps isn't an Azure product — it's the **operating discipline** (formalized by the FinOps Foundation) that everything in Section 5 exists to support: bringing financial accountability to cloud's variable-spend model, the same way DevOps brought operational accountability into engineering. It's explicitly **cross-functional** — engineering, finance, and business stakeholders sharing a real-time view of spend, instead of finance discovering the number 30 days later on an invoice with no context.

The FinOps Framework runs as a continuous cycle across three phases:

1. **Inform** — can you actually _see_ and _allocate_ spend accurately? This phase lives or dies on the tagging discipline from Section 4 and the visibility tools from Sections 5.1–5.2 — you cannot optimize what you can't attribute to a team or product.
2. **Optimize** — act on what Inform reveals: rightsizing over-provisioned resources, buying Reservations/Savings Plans for now-predictable workloads, eliminating idle/orphaned resources (unattached disks, forgotten dev VMs left running).
3. **Operate** — the governance loop that keeps the first two phases from decaying: Budgets and alerts (5.1) catch drift early, and **Azure Policy** (Section 1) automatically enforces the tagging and configuration standards that Inform depends on — so accurate cost visibility isn't a one-time cleanup project, it's structurally maintained.

Microsoft's own tooling reflects this framework directly: Cost Management exports increasingly follow **FOCUS** (the FinOps Open Cost and Usage Specification — an open, cross-cloud standard for normalizing billing data), and the open-source **FinOps toolkit** provides Power BI–based reporting built on top of that standardized data, specifically so organizations running multi-cloud aren't stuck reconciling incompatible billing formats team by team.

---

## How It All Connects

Notice the same layering pattern as the Security guide, applied to governance instead of protection:

- **Azure Policy** is the _enforcement engine_ — it's what actually makes tagging standards, required configurations, and remediation happen automatically rather than relying on documentation nobody reads.
- **Resource Locks** are a blunter, more absolute backstop for the handful of resources where "even an Owner shouldn't be able to do this by accident" — a shared VNet, a production database, a domain controller.
- **Tags** are the connective tissue that lets everything else — cost, automation, ownership, security classification — be _queried_ rather than tribal-knowledge.
- **Cost Management** turns that tagged, policy-governed estate into an actual financial control loop: budgets catch overspend early, Reservations/Savings Plans convert predictable patterns into real discounts.
- **FinOps** is the organizational habit that keeps all four of the above running as a cycle instead of a one-time setup project — which is exactly why it's listed last: it's not a tool you configure once, it's the discipline that keeps the tools honest over time.

The Blueprints retirement is a good illustration of the broader lesson in this whole domain: governance tooling itself needs governance — know what's actively supported before you build a multi-year standard on top of it.