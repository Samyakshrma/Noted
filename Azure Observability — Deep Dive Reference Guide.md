

Everything in the previous four guides — compute, storage, security, governance — eventually needs one question answered: **what is actually happening right now, and why?** Observability is that layer. The mental model that ties this whole guide together: **Azure Monitor is the platform, Log Analytics Workspace is where the detailed data lives and gets queried, and Application Insights is that same backend specialized down to the level of your own application code.** One telemetry platform, one query language, applied at every altitude from "is this VM's CPU spiking" down to "which downstream API call inside my checkout service is timing out."

---

## 1. Azure Monitor

Azure Monitor is the umbrella platform for collecting, analyzing, and acting on telemetry across Azure, hybrid, and even multi-cloud resources. Log Analytics, Application Insights, Alerts, and Dashboards aren't separate products bolted on — they're all facets of this one platform.

### 1.1 Metrics

**Numerical, time-series data** — CPU %, network throughput, request count, queue length — collected at regular intervals and stored in a specialized time-series database purpose-built for fast, near-real-time analysis (typically 1–3 minute latency from emission to availability).

- **Platform metrics** are emitted automatically by most Azure resources with zero configuration required. **Custom metrics** can be emitted by your own application code via the SDK.
- **Metrics Explorer** is the portal tool for charting them — apply aggregations (avg/sum/min/max/count), split by dimension (e.g., break VM CPU down per-instance in a scale set), and pin charts to a dashboard.
- Metrics are cheap to store and fast to query, which is exactly why they're the right tool for **dashboards and low-latency alerting** — but they're intentionally shallow. A CPU metric tells you _that_ something spiked, not _why_. That "why" is what Logs are for.
- Retention for platform metrics is around 93 days by default — long enough for trend analysis, but if you need longer-term metric history, route it into a Log Analytics Workspace.

### 1.2 Logs

**Structured, detailed event records** — richer and slower than metrics, with a latency of a few minutes rather than seconds, but capable of answering "why," not just "what."

|Log Type|What It Captures|Availability|
|---|---|---|
|**Activity Log**|Subscription-level record of **control-plane** operations — who did what, to which resource, when (e.g., "User X restarted VM Y at 14:32")|Always on, no configuration needed|
|**Resource Logs (Diagnostic Logs)**|**Data-plane** operations specific to a resource — storage read/write ops, NSG flow logs, SQL query logs|Must be explicitly enabled via a **Diagnostic Setting** on each resource|
|**Microsoft Entra ID Logs**|Sign-ins, audit logs, provisioning events|Enabled via Entra diagnostic settings|

**Diagnostic Settings** are the routing mechanism — for any given resource, you configure where its logs/metrics should flow: a **Log Analytics Workspace** (for querying), a **Storage Account** (for cheap long-term archive), or an **Event Hub** (for streaming out to a third-party SIEM or external tool — this is also how logs reach Microsoft Sentinel from the Security guide when Sentinel itself isn't the primary workspace).

### 1.3 Alerts

The layer that turns telemetry from "data you could look at" into "something that actively notifies or remediates you."

|Alert Type|Trigger|Typical Use|
|---|---|---|
|**Metric alert**|A metric crosses a static threshold, or deviates from an ML-derived **dynamic threshold** (learns normal range automatically)|Near-real-time — "CPU > 90% for 5 minutes"|
|**Log alert**|A scheduled **KQL query** runs periodically; alert fires if the result meets a condition|Anything requiring correlation/complex logic a simple metric can't express|
|**Activity Log alert**|A specific control-plane event occurs|"Alert whenever anyone deletes a production VM," or Azure **Service Health** events (planned maintenance, outages)|
|**Smart Detection**|Application Insights' built-in ML anomaly detection|Automatically flags failure-rate or performance anomalies without you writing a rule at all|

**Anatomy of an alert rule:** target scope → condition/signal → evaluation frequency → **Action Group**.

**Action Groups** are deliberately decoupled from the alert rule itself — a reusable bundle of responses (email, SMS, push notification, webhook, Logic App, Azure Function, ITSM connector, Automation Runbook) that many different alert rules can share, so you define "how the on-call team gets notified" exactly once and reuse it everywhere.

- **Severity levels** run Sev 0 (Critical) through Sev 4 (Verbose), letting routing/urgency differ by how bad the signal actually is.
- **Alert Processing Rules** let you suppress or reroute alerts temporarily — e.g., silence non-critical alerts during a planned maintenance window without disabling the underlying alert rules.

---

## 2. Log Analytics Workspace

This is the actual **data store** logs get collected into — a regional container organized into **tables** (schema), like `AzureActivity`, `Heartbeat`, `Perf`, resource-specific diagnostic tables, and custom tables you define yourself.

**Data Collection Rules (DCRs)** are the modern, general-purpose ingestion pipeline: they define _what_ gets collected from a source (a VM via the Azure Monitor Agent, a custom app log file, etc.), any **transformation** applied to it in-flight (filtering, reshaping columns before storage — reducing cost by dropping noise before it's even ingested), and _where_ it's routed. This replaced the older pattern of per-resource-type collection configuration with one consistent model.

**Workspace design — a real architectural decision, not a formality:**

|Approach|Trade-off|
|---|---|
|**One centralized workspace**|Simplest cross-resource querying, but coarser access control — anyone with workspace access can potentially see everyone's data unless you configure table-level RBAC|
|**Multiple workspaces** (per team/subscription/region)|Cleaner access boundaries and easier data-residency/sovereignty compliance, but cross-workspace KQL queries are possible yet a bit more complex, and per-workspace cost/retention has to be managed individually|

**Retention & Archive** — interactive retention defaults to 30 days and is configurable up to 2 years; beyond that, a much cheaper **Archive tier** retains the data (queryable via an on-demand search job rather than instantly) for long-term compliance needs without paying full interactive-tier storage cost.

**Pricing** is primarily pay-as-you-go per GB ingested, with **commitment tiers** (reserved daily ingestion capacity) available at a discount for high-volume workspaces — directly analogous to the Reservations/Savings Plans trade-off from the Governance guide, just applied to log ingestion instead of compute.

### 2.1 KQL — Kusto Query Language

KQL is a **read-only**, pipe-based query language purpose-built for exploring large volumes of log/telemetry data. The mental model: data flows left to right through a chain of tabular operators, each one reshaping the result before handing it to the next — closer to a Unix pipeline or LINQ than to SQL's declarative style.

**Core operators worth actually knowing:**

|Operator|Purpose|
|---|---|
|`where`|Filter rows|
|`summarize`|Aggregate — `count()`, `avg()`, grouped `by` a dimension (the rough equivalent of SQL's `GROUP BY`)|
|`project` / `project-away`|Select or drop specific columns|
|`extend`|Add a computed column|
|`join`|Combine rows across two tables (`kind=inner`, `leftouter`, etc.)|
|`bin()`|Bucket a timestamp into fixed intervals — the standard way to build time-series aggregations|
|`render`|Visualize the result directly as a chart, inline in the query tool|

**A representative real query** — top 5 slowest failing dependency calls in the last hour, from an Application Insights-backed workspace:

```kql
dependencies
| where TimeGenerated > ago(1h)
| where Success == false
| summarize FailureCount = count(), AvgDuration = avg(DurationMs)
    by Name, Target
| top 5 by FailureCount desc
```

- Nearly every table uses `TimeGenerated` as its standard timestamp column, and `ago(1h)`, `ago(7d)`, etc. are the idiomatic way to express relative time windows.
- **KQL is the single unifying skill across this entire series** — the exact same language queries a Log Analytics Workspace, Application Insights (Section 3), and **Microsoft Sentinel** from the Security guide, and it's also the native query language of Azure Data Explorer. Learning it once pays off in every one of those contexts.
- **Workbooks** build on top of KQL queries (plus metrics) to assemble reusable, interactive, shareable dashboards/reports — the natural next step once you have a query you'll want to check regularly.

---

## 3. Application Insights

Application Insights is Azure Monitor's **Application Performance Monitoring (APM)** capability — where Metrics and platform Logs tell you about _infrastructure_, Application Insights instruments your actual **code**, capturing requests, outbound dependency calls, exceptions, and custom traces, either via an SDK or increasingly through codeless/agent-based auto-instrumentation.

Critically, modern Application Insights is **workspace-based** — its data lands in a Log Analytics Workspace right alongside your infrastructure logs, in tables like `requests`, `dependencies`, `exceptions`, `traces`, and `customEvents`. This means every KQL skill from Section 2.1 applies directly, with no separate query language to learn for application-level telemetry.

### 3.1 Distributed Tracing

**The problem it solves:** in a microservices architecture, one user request might touch a dozen services — an API gateway, three backend services, a database call, a queue publish, a downstream partner API. Without correlation, an error shows up as a dozen unrelated log lines across a dozen services, and you're left manually guessing which ones belong to the same failed request.

Distributed tracing solves this by propagating a **correlation ID** (`operation_Id`) across every service boundary — standardized today via **W3C Trace Context** headers (`traceparent`/`tracestate`) — so every span of work across every service, for one single user request, can be reassembled into one coherent **end-to-end transaction view** in the portal.

- **The critical practical caveat:** the trace is only as complete as your instrumentation. If one service in the middle of the chain isn't instrumented (or drops the trace headers instead of forwarding them), the trace has a gap at exactly that point — this is the single most common reason distributed tracing "doesn't work" in practice: it's not broken, it's incomplete.
- This capability is what makes Application Insights genuinely essential once you've adopted Container Apps or AKS from the Compute guide — the more a request fans out across small independent services, the more distributed tracing goes from "nice to have" to "the only realistic way to debug a slow request."

### 3.2 Dependency Mapping

The **Application Map** is an auto-generated visual topology of your system — built directly from the distributed tracing data, not manually drawn — showing how your application's components call each other and any external dependencies (databases, downstream APIs, queues, caches).

- Each node on the map surfaces health at a glance: call volume, average duration, failure rate — so during an incident you can visually spot which single node in the chain is the actual bottleneck or point of failure, rather than manually correlating logs across every service one at a time.
- It has a genuine secondary benefit beyond troubleshooting: it's a **living architecture diagram**. New engineers can see what actually calls what, right now, instead of trusting a design doc that was accurate eighteen months ago and has silently drifted since.

### 3.3 Telemetry

The categories of data Application Insights collects, automatically or via light instrumentation:

|Telemetry Type|What It Captures|
|---|---|
|**Requests**|Incoming HTTP requests to your app — duration, response code, success/failure|
|**Dependencies**|Outgoing calls your app makes — SQL queries, HTTP calls to other APIs, Storage/Service Bus calls — tracked individually for duration and success|
|**Exceptions**|Unhandled (and optionally handled) exceptions, with full stack traces|
|**Traces**|Your own custom log statements — whatever you already emit via `ILogger`, Serilog, log4net, etc., funneled into the same pipeline|
|**Custom Events / Metrics**|Business-specific telemetry you explicitly emit — e.g., a `CheckoutCompleted` event with custom properties — for product analytics, not just technical health|
|**Page Views / Browser telemetry**|Client-side data from the JavaScript SDK — page load time, AJAX call performance, browser-side exceptions|

**Sampling** matters at real production volume: collecting 100% of telemetry from a high-traffic app gets expensive fast and can even throttle ingestion. Application Insights supports **adaptive sampling** (automatically dials volume down to hit a target ingestion rate while preserving statistically accurate aggregates) as well as fixed-rate sampling — understanding this trade-off matters, because misconfigured sampling is a common reason "the trace I need just isn't there" during an incident.

**Live Metrics Stream** is a separate, sub-second-latency view of a small set of key signals (request rate, failure rate, CPU, memory) — distinct from the few-minutes latency of regular ingestion — purpose-built for watching a deployment roll out in real time and catching an immediate regression before the normal telemetry pipeline would even show it.

---

## How It All Connects

The layering here mirrors every prior guide in this series, and it's worth being explicit about it:

- **Metrics** are fast, cheap, and shallow — the right tool for "is something wrong right now" and low-latency alerting.
- **Logs + Log Analytics Workspace + KQL** are slower but deep — the right tool for "why" once metrics tell you "something." One query language, one backend, used for infrastructure logs, security data (Sentinel, from the Security guide), and application telemetry alike.
- **Application Insights** takes that exact same backend and specializes it down to your own code — and adds **distributed tracing + dependency mapping** specifically to solve the problem that infrastructure-level logs and metrics structurally can't: "of my 30 microservices, which one actually caused this."
- **Alerts + Action Groups** are what convert all of the above from passive data into active response — and this is the same thread that ran through the Security guide's "Assume Breach → continuous monitoring" principle and the Governance guide's "Operate" phase of FinOps (budget alerts). Observability isn't a separate concern from security or cost governance — it's the shared nervous system all three depend on.

The practical takeaway: invest in learning **KQL** properly before anything else in this guide. It's the one skill that compounds across literally every other Azure domain you've covered in this series — infrastructure logs, security investigation, and application debugging all run through the same query language.