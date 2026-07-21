# 08 · Azure Monitor & Log Analytics

**Azure Monitor** is the umbrella platform that collects metrics, logs, and
traces from every Azure resource. Its two everyday pieces are **metrics**
(numeric time series, e.g. CPU %) and **Log Analytics** (a queryable store
for text/structured logs, searched with **KQL**, the Kusto Query Language).
This module wires up both, plus an alert that notifies you when something
crosses a threshold.

## Platform metrics (free, automatic)

Every Azure resource emits **platform metrics** automatically, at no extra
cost, for free — no setup required.

```bash
# List available metric definitions for a VM
az monitor metrics list-definitions \
  --resource /subscriptions/<sub-id>/resourceGroups/rg-vm-demo/providers/Microsoft.Compute/virtualMachines/vm-demo \
  --output table

# Read the last hour of CPU percentage, averaged per 5-minute bucket
az monitor metrics list \
  --resource /subscriptions/<sub-id>/resourceGroups/rg-vm-demo/providers/Microsoft.Compute/virtualMachines/vm-demo \
  --metric "Percentage CPU" \
  --interval PT5M \
  --start-time $(date -u -v-1H '+%Y-%m-%dT%H:%MZ') \
  --output table
```

You can view the same data as a chart in the Portal under any resource's
**Monitoring → Metrics** blade — the CLI and Portal read from the same
underlying store.

## Log Analytics workspace

A **Log Analytics workspace** is the destination for logs (Azure resource
diagnostic logs, VM guest-level logs via the Azure Monitor Agent,
Application Insights telemetry, custom logs) and the place you run KQL
queries against them.

```bash
az group create --name rg-monitor-demo --location eastus

az monitor log-analytics workspace create \
  --resource-group rg-monitor-demo \
  --workspace-name law-learn-demo

WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group rg-monitor-demo \
  --workspace-name law-learn-demo \
  --query customerId --output tsv)
```

### Send a resource's logs to the workspace

Diagnostic settings route a resource's logs/metrics to one or more
destinations (Log Analytics, Storage, Event Hubs):

```bash
az monitor diagnostic-settings create \
  --name send-to-law \
  --resource /subscriptions/<sub-id>/resourceGroups/rg-func-demo/providers/Microsoft.Web/sites/<function-app-name> \
  --workspace law-learn-demo \
  --logs '[{"category": "FunctionAppLogs", "enabled": true}]' \
  --metrics '[{"category": "AllMetrics", "enabled": true}]'
```

## Querying with KQL

**KQL (Kusto Query Language)** reads like a pipeline of filters, similar in
spirit to piping through `grep`/`awk`, but structured and typed.

```kusto
// All error-level function app logs from the last 24 hours
FunctionAppLogs
| where TimeGenerated > ago(24h)
| where Level == "Error"
| project TimeGenerated, Message, FunctionName
| order by TimeGenerated desc
```

```kusto
// Count requests per hour over the last day (works against AppRequests
// once Application Insights is wired in -- see the callout below)
AppRequests
| where TimeGenerated > ago(1d)
| summarize RequestCount = count() by bin(TimeGenerated, 1h)
| render timechart
```

Run a query from the CLI:

```bash
az monitor log-analytics query \
  --workspace $WORKSPACE_ID \
  --analytics-query "FunctionAppLogs | where Level == 'Error' | take 20" \
  --output table
```

Key KQL keywords to know starting out:

| Keyword | Purpose |
|---|---|
| `where` | Filter rows by a condition. |
| `project` | Select/rename specific columns. |
| `summarize ... by` | Aggregate (count, avg, sum, ...) grouped by a column. |
| `order by` | Sort results. |
| `ago(1d)` / `ago(30m)` | Relative time filters. |
| `bin(TimeGenerated, 1h)` | Bucket timestamps for time-series aggregation. |
| `render timechart` | Render results as a chart in the Portal's query UI. |

## Alerts

An **alert rule** watches a metric or log query and fires an **action
group** (email, SMS, webhook, Azure Function, ...) when a condition is met.

```bash
# Create an action group that emails you
az monitor action-group create \
  --resource-group rg-monitor-demo \
  --name ag-email-me \
  --short-name emailme \
  --action email admin-alert your-email@example.com

# Alert if a VM's CPU averages above 80% for 5 minutes
az monitor metrics alert create \
  --resource-group rg-monitor-demo \
  --name high-cpu-alert \
  --scopes /subscriptions/<sub-id>/resourceGroups/rg-vm-demo/providers/Microsoft.Compute/virtualMachines/vm-demo \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action ag-email-me
```

Log-based alerts (fired from a KQL query result rather than a metric) use
`az monitor scheduled-query create` instead, letting you alert on things
like "more than 10 error log lines in the last 15 minutes."

## Application Insights (a preview)

**Application Insights** is Azure Monitor's application performance
monitoring (APM) layer — request rates, response times, dependency calls,
and exception traces from inside your app code, typically via a lightweight
SDK or auto-instrumentation. It's built on the same Log Analytics workspace
and queried with the same KQL, just against different tables (`AppRequests`,
`AppExceptions`, `AppDependencies`). Covered in depth in
[Level 4, Module 05](../level-4/05-advanced-observability.md); worth
knowing it exists as you design apps at this level, since wiring it in early
is far easier than retrofitting it later.

## Cheat sheet

| Command | Purpose |
|---|---|
| `az monitor metrics list --metric --interval` | Read a resource's platform metrics. |
| `az monitor log-analytics workspace create` | Create a Log Analytics workspace. |
| `az monitor diagnostic-settings create --workspace --logs --metrics` | Route a resource's logs/metrics to a workspace. |
| `az monitor log-analytics query --analytics-query` | Run a KQL query from the CLI. |
| `az monitor action-group create --action email` | Create a notification target for alerts. |
| `az monitor metrics alert create --condition --action` | Create a metric-based alert rule. |
| `where` / `summarize by` / `ago()` | Core KQL filtering, aggregation, and time-window keywords. |

## Exercise

1. Create a Log Analytics workspace and route a resource's (e.g. a Function
   App's) logs into it via a diagnostic setting.
2. In the Portal's **Logs** blade for that workspace, run a KQL query
   filtering to the last hour and counting entries by category.
3. Create an action group with your email, and a metric alert on any
   resource you still have running (e.g. VM CPU or Function App errors)
   with a low, easy-to-trigger threshold so you can see the notification
   arrive.
4. Delete the alert, action group, and resource group when done.
