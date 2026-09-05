# 06 · Performance Engineering & Load Testing

Observability (Module 05) tells you what's happening in production; **load
testing** tells you what will happen before it does — finding the point
where a service degrades, and confirming that autoscaling (from
[Level 3, Module 03](../level-3/03-advanced-aks.md)) actually reacts in
time to protect it.

## Azure Load Testing basics

```bash
az load create \
  --resource-group rg-perf \
  --name alt-orders-platform \
  --location eastus

az load test create \
  --resource-group rg-perf \
  --load-test-resource alt-orders-platform \
  --test-id orders-api-baseline \
  --load-test-config-file loadtest-config.yaml
```

```yaml
# loadtest-config.yaml
testId: orders-api-baseline
testPlan: orders-api.jmx
engineInstances: 2
failureCriteria:
  - avg(response_time_ms) > 500
  - percentage(error) > 1
```

Azure Load Testing runs Apache JMeter scripts on managed, autoscaled test
engines — no infrastructure to provision yourself, and the same JMX file
works whether you run it locally for a quick check or at scale through the
service.

```bash
az load test-run create \
  --resource-group rg-perf \
  --load-test-resource alt-orders-platform \
  --test-id orders-api-baseline \
  --test-run-id run-$(date +%Y%m%d-%H%M)
```

```text
$ az load test-run show -g rg-perf --load-test-resource alt-orders-platform --test-run-id run-20260825-1000 --query "{status:status, virtualUsers:virtualUsers}"
{
  "status": "DONE",
  "virtualUsers": 50
}
```

**Gotcha:** `failureCriteria` marks a run pass/fail but doesn't stop the
test early on breach by default — a test can run its full duration hammering
an already-failing service, generating cost and noisy data, unless you also
configure `--auto-stop-error-rate` to abort early once error rate crosses a
threshold, which most teams forget until their first expensive dead-run.

## Finding the breaking point

Ramp virtual users incrementally rather than jumping straight to peak load,
so you can identify the actual degradation point, not just pass/fail at one
level:

```yaml
engineInstances: 4
loadPattern:
  rampUp:
    duration: 5m
    startUsers: 10
    endUsers: 200
```

```text
Users   Avg Latency  Error Rate
10      120ms        0.0%
50      180ms        0.1%
100     340ms        0.3%
150     890ms        2.1%    <- degradation starts here
200     3400ms       18%     <- effectively failing
```

**Gotcha:** the load-generation client itself can become the bottleneck
before the service under test does — if `engineInstances` is too low for
the target load, you measure the test tool's ceiling, not the service's;
watch the test engine's own CPU/network metrics (surfaced in the results)
to confirm it wasn't saturated before concluding the service is the
limiting factor.

## Validating autoscaling actually keeps up

Correlate the load test's ramp-up against the HPA/cluster-autoscaler
timeline from [Level 3](../level-3/03-advanced-aks.md):

```kusto
InsightsMetrics
| where Name == "PodReadyCount"
| where TimeGenerated between (datetime(2026-08-25T10:00:00Z) .. datetime(2026-08-25T10:10:00Z))
| project TimeGenerated, Val
| order by TimeGenerated asc
```

```text
TimeGenerated          Val
10:00:00               2
10:03:00               2     <- load ramping, HPA hasn't reacted yet
10:03:30               5     <- HPA scaled pods
10:05:00               8     <- new nodes joined, pods scheduled
```

The gap between load starting to climb and pod count actually increasing
is your **real-world scale-up lag** — if that gap exceeds how long your
users tolerate degraded latency, no amount of eventual autoscaling saves
the experience; the fix is either a higher `--min-count` baseline or a more
aggressive HPA target, not just "autoscaling is enabled."

## Performance-testing environments

**Gotcha:** load-testing a shared staging environment used by other teams
can produce misleading results (their traffic mixes with your synthetic
load) or actively cause an incident for them — always run load tests
against an isolated environment matching production's topology and SKUs as
closely as possible, never directly against production, and never against
a shared non-isolated staging slot without coordinating first.

## Profiling application-level bottlenecks

Beyond infrastructure load testing, **Application Insights Profiler**
samples actual code execution under load to show which method is
consuming time inside a slow request:

```bash
az monitor app-insights component update \
  --resource-group rg-perf \
  --app orders-api-insights \
  --set properties.SamplingPercentage=100
```

```text
Hot path: OrdersController.CreateOrder
  └─ 62%  OrderValidator.ValidateInventory  (blocking synchronous DB call)
  └─ 28%  PaymentClient.Charge
  └─ 10%  other
```

**Gotcha:** Profiler and Snapshot Debugger both add measurable CPU/memory
overhead when enabled — leaving 100% sampling on indefinitely in production
is itself a performance risk; use it to capture a targeted window during a
load test or incident, then reduce sampling back down.

## Load testing checklist

| Check | Why |
|---|---|
| Isolated, prod-like environment | Avoid contaminating shared staging or hitting real prod |
| Ramped (not instant) load pattern | Find the actual degradation point, not just pass/fail |
| `--auto-stop-error-rate` configured | Avoid a long, expensive, already-failed run |
| Test engine metrics reviewed | Confirm the client isn't the bottleneck |
| Autoscaling timeline correlated | Confirm scale-up lag is within tolerance |
| Profiler sampling reduced after use | Avoid leaving overhead on indefinitely |

## How It Actually Works

**Azure Load Testing** doesn't generate load from a single client machine —
it runs your Apache JMeter test plan across a fleet of managed, ephemeral
**engine instances** provisioned specifically for the test run, each
executing a share of the configured virtual users in parallel and
streaming results back to the Load Testing service's aggregation layer,
which is why the service can realistically simulate tens of thousands of
concurrent users: the load is genuinely distributed across many
short-lived compute instances rather than one machine's network/CPU
ceiling. Real-time metrics during a run (response time percentiles, error
rate) are computed by that aggregation layer incrementally as engine
instances report batched results, not by waiting for the run to finish and
post-processing a log file — this is what lets you abort a test early when
you see it already failing thresholds.

A load test's results are only as informative as what's actually being
measured under the hood at the target: for an App Service or AKS-hosted
app, the same autoscale mechanisms from Levels 2–3 (Azure Monitor
Autoscale polling metrics, the Kubernetes HPA/cluster autoscaler) are
reacting to the injected load in real time on their own evaluation
cadence, which is why a load test's ramp-up profile matters mechanically —
a too-fast ramp can outpace the autoscaler's polling interval and cooldown
period, causing throttling/errors that reflect the autoscaler's reaction
lag rather than the application code's actual capacity ceiling. This is
also why performance test result interpretation has to separate "the app
code is slow" from "the platform hadn't finished scaling yet" — two
different bottlenecks with completely different underlying causes.

## Cheat sheet

| Command | Purpose |
|---|---|
| `az load create` | Create an Azure Load Testing resource. |
| `az load test create --load-test-config-file` | Define a test from a JMeter plan and config. |
| `az load test-run create` | Execute a test run. |
| `az load test-run show --query status` | Check run status/results. |
| `az monitor app-insights component update --set properties.SamplingPercentage` | Adjust Profiler sampling rate. |

## Exercise

1. Create an Azure Load Testing resource and run a basic JMeter-based test
   against a sample API with `failureCriteria` on latency and error rate.
2. Configure a ramping load pattern and identify the user-count point
   where latency and error rate visibly degrade.
3. Correlate the load ramp against `PodReadyCount` in Log Analytics and
   measure the real scale-up lag for your HPA/cluster-autoscaler config.
4. Enable Profiler at 100% sampling during one test run, capture a hot
   path, then reduce sampling back down afterward.
5. Delete the resource group when finished.
