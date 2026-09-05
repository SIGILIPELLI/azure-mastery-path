# 05 · Advanced Observability (App Insights, Tracing)

Basic Azure Monitor metrics tell you a VM's CPU is high; they don't tell
you which downstream call inside a distributed request made an API respond
slowly. This module covers **Application Insights** for distributed
tracing, **Log Analytics** KQL queries for cross-service correlation, and
building alerts on the signals that actually predict incidents.

## Enabling Application Insights on a workload

```bash
az monitor app-insights component create \
  --resource-group rg-observability \
  --app orders-api-insights \
  --location eastus \
  --application-type web \
  --workspace $(az monitor log-analytics workspace show -g rg-observability -n law-platform --query id -o tsv)
```

Workspace-based Application Insights (as above, via `--workspace`) stores
telemetry in a shared Log Analytics workspace rather than its own classic
store — this is the current default and lets you join app telemetry with
infrastructure logs in one KQL query, which the older classic mode couldn't
do.

```bash
az monitor app-insights component show \
  --resource-group rg-observability \
  --app orders-api-insights \
  --query connectionString -o tsv
```

Wire the connection string into the app (via `APPLICATIONINSIGHTS_CONNECTION_STRING`
environment variable, consumed by the Application Insights SDK or, for
many languages, auto-instrumentation with no code changes).

**Gotcha:** the older **instrumentation key** alone is deprecated in favor
of the full **connection string** (which also encodes the ingestion
endpoint) — apps configured with only an instrumentation key can silently
fail to send telemetry if Microsoft changes regional ingestion endpoints,
since the key alone no longer reliably resolves where to send data.

## Distributed tracing across services

Application Insights auto-correlates requests across services using the
**W3C Trace Context** (`traceparent` header) — a request into `orders-api`
that calls `fulfillment-worker` via Service Bus, which calls `payments-api`,
shows up as one **end-to-end transaction** in the portal if all three
services share instrumentation:

```kusto
requests
| where operation_Id == "a1b2c3d4e5f6"
| project timestamp, name, duration, success, cloud_RoleName
| order by timestamp asc
```

```text
timestamp             name                duration  success  cloud_RoleName
2026-08-25T10:00:01Z  POST /orders        842       True     orders-api
2026-08-25T10:00:01Z  ProcessOrder        210       True     fulfillment-worker
2026-08-25T10:00:01Z  POST /charge        615       True     payments-api
```

**Gotcha:** trace correlation breaks across any hop that doesn't propagate
`traceparent` — a message dropped onto a Service Bus queue without copying
the trace context into a message property, or a call through a component
without an Application Insights SDK, **silently truncates** the trace; you
see two separate, uncorrelated transactions instead of one, with no error
telling you correlation failed.

## KQL for cross-cutting queries

```kusto
requests
| where timestamp > ago(1h)
| where success == false
| summarize FailureCount = count() by name, resultCode
| order by FailureCount desc
```

```kusto
dependencies
| where timestamp > ago(1h)
| where type == "Azure Service Bus"
| summarize avg(duration), percentile(duration, 95) by target
```

**Gotcha:** `percentile()` in KQL is computed **approximately** (using a
t-digest algorithm) for performance on large datasets — for exact
percentiles on small, precise datasets use `percentiles_array` with a
smaller sample or validate against `arg_max`/manual sorting; the difference
rarely matters at p95 dashboards but can mislead if you're chasing a tight
SLA at p99.9 on a low-volume endpoint.

## Alerting on the right signals

```bash
az monitor metrics alert create \
  --resource-group rg-observability \
  --name alert-high-latency \
  --scopes $(az monitor app-insights component show -g rg-observability -n orders-api-insights --query id -o tsv) \
  --condition "avg requests/duration > 1000" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2
```

For custom/log-based conditions beyond built-in metrics, use a **scheduled
query alert rule** running KQL on a timer:

```bash
az monitor scheduled-query create \
  --resource-group rg-observability \
  --name alert-deadletter-growth \
  --scopes $(az monitor log-analytics workspace show -g rg-observability -n law-platform --query id -o tsv) \
  --condition "count 'AzureDiagnostics | where Category == \"DeadletterMessages\"' > 10" \
  --window-size 15m \
  --evaluation-frequency 5m \
  --severity 1
```

**Gotcha:** alerting on **symptoms** (CPU high, latency up) instead of
**causes/leading-indicators** (queue depth trending up, connection pool
exhaustion) means you find out about incidents at the same time your users
do. A dead-letter queue depth alert (from
[Level 3, Module 04](../level-3/04-event-driven-architecture.md)) or a
connection-pool-exhaustion alert typically fires *before* the customer-
facing latency alert, giving actual lead time to react.

## Observability signal types

| Signal | Source | Best for |
|---|---|---|
| **Metrics** | Azure Monitor platform metrics | Resource-level health (CPU, throughput) |
| **Traces** | Application Insights | Following one request across services |
| **Logs** | Log Analytics (KQL) | Ad-hoc investigation, cross-cutting queries |
| **Dependency tracking** | Application Insights SDK | Finding which downstream call is slow |
| **Custom events** | Application Insights SDK (`trackEvent`) | Business-level signals (checkout completed) |

## How It Actually Works

**Application Insights**' distributed tracing works by having its SDK
inject a `traceparent` header (the W3C Trace Context standard) into every
outbound HTTP call your instrumented service makes, and by reading that
same header on every inbound request — each service in a call chain then
reports its own span (start time, duration, dependency calls) tagged with
a shared trace/operation ID, which Application Insights' backend later
stitches together into one **end-to-end transaction view** purely by
grouping all spans sharing that operation ID, not through any special
network-level correlation. This is architecturally the same telemetry
pattern a service mesh's sidecars produce automatically (Module 3) — the
difference is Application Insights instruments at the application/SDK
layer, so it captures logical spans your code defines (a specific method,
a specific SQL call) rather than only proxy-visible HTTP boundaries.

Under the hood, all of this — platform metrics, resource logs, App
Insights traces — converges on the same **Log Analytics workspace /ADX
Kusto engine** from Level 1's Monitor module; **KQL cross-resource queries**
work because every ingested record, regardless of source, is tagged with
its originating resource ID and stored in the workspace's shared column
store, letting one query correlate an AKS pod's logs with the Application
Gateway's access logs and a Cosmos DB's request-unit metrics by joining on
timestamp and correlation fields. **Workbooks and dashboards** are not
separate data stores — they're saved KQL query definitions plus
visualization metadata, re-executed against the live workspace/metrics
store each time they're opened, which is why a workbook's data is always
current as of query time rather than a cached snapshot.

## Cheat sheet

| Command | Purpose |
|---|---|
| `az monitor app-insights component create --workspace` | Create workspace-based App Insights. |
| `az monitor app-insights component show --query connectionString` | Get the connection string to wire into an app. |
| `requests \| where operation_Id ==` | Pull one full distributed trace by ID. |
| `dependencies \| summarize percentile(duration, 95)` | Get p95 latency for downstream calls. |
| `az monitor metrics alert create` | Alert on a built-in platform metric. |
| `az monitor scheduled-query create` | Alert on a custom KQL condition. |

## Exercise

1. Create a workspace-based Application Insights resource and wire its
   connection string into a small test app.
2. Trigger a request that calls at least one downstream dependency, then
   pull the full trace with `operation_Id` and confirm both hops appear.
3. Deliberately break trace propagation (skip forwarding `traceparent`
   across one hop) and observe the transaction split into two.
4. Write a KQL query surfacing the top 5 failing request names in the last
   hour by count.
5. Create one metric alert and one scheduled-query alert, and explain which
   of your alerts is a leading indicator versus a symptom.
