# 09 · Azure Functions Advanced

[Level 1, Module 07](../level-1/07-azure-functions.md) covered the basics:
triggers, bindings, the Consumption plan. This module covers what you need
once a single stateless HTTP function isn't enough anymore: **Durable
Functions** (stateful orchestration), custom bindings, and **deployment
slots** for Functions apps.

## Durable Functions: the problem they solve

A plain function is stateless and short-lived — it can't easily express
"call function A, wait for it to finish, then call B and C in parallel,
then wait for both before calling D," especially if that whole sequence
might take minutes or hours (functions have execution time limits on
Consumption). **Durable Functions** is an extension that lets an
**orchestrator function** describe exactly that kind of workflow in
ordinary code, with Azure handling checkpointing and replay behind the
scenes.

```bash
az group create --name rg-func-adv --location eastus

az storage account create \
  --name stfuncadv$RANDOM \
  --resource-group rg-func-adv \
  --sku Standard_LRS
STORAGE_ACCOUNT=$(az storage account list --resource-group rg-func-adv --query "[0].name" --output tsv)

az functionapp create \
  --resource-group rg-func-adv \
  --name func-adv-demo$RANDOM \
  --storage-account $STORAGE_ACCOUNT \
  --consumption-plan-location eastus \
  --runtime python \
  --runtime-version 3.11 \
  --functions-version 4 \
  --os-type Linux

FUNCTION_APP=$(az functionapp list --resource-group rg-func-adv --query "[0].name" --output tsv)
```

`requirements.txt` needs the Durable Functions package:

```
azure-functions
azure-functions-durable
```

## The orchestrator, activity, and client functions

A Durable Functions app has (at minimum) three function types working
together:

`HttpStart/__init__.py` — the **client function** that kicks things off:

```python
import azure.functions as func
import azure.durable_functions as df


async def main(req: func.HttpRequest, starter: str) -> func.HttpResponse:
    client = df.DurableOrchestrationClient(starter)
    instance_id = await client.start_new("ProcessOrderOrchestrator", None, None)
    return client.create_check_status_response(req, instance_id)
```

`ProcessOrderOrchestrator/__init__.py` — the **orchestrator function**,
describing the workflow:

```python
import azure.durable_functions as df


def orchestrator_function(context: df.DurableOrchestrationContext):
    order = context.get_input()

    validated = yield context.call_activity("ValidateOrder", order)
    # Fan out: charge payment and reserve inventory in parallel
    payment_task = context.call_activity("ChargePayment", validated)
    inventory_task = context.call_activity("ReserveInventory", validated)
    yield context.task_all([payment_task, inventory_task])

    result = yield context.call_activity("SendConfirmation", validated)
    return result


main = df.Orchestrator.create(orchestrator_function)
```

`ValidateOrder/__init__.py` — one of several **activity functions**, the
actual units of work:

```python
def main(order: dict) -> dict:
    order["validated"] = True
    return order
```

**Gotcha — orchestrator code must be deterministic:** Azure replays the
orchestrator function's code from the top every time it wakes up after an
`await`/`yield` (to rebuild its in-memory state from the history), so it
can't call `datetime.utcnow()`, generate random numbers, or call an
external API directly inside the orchestrator — those side effects belong
in activity functions, called via `context.call_activity(...)`, never
inline in the orchestrator body.

## Checking status and the pattern this replaces

```bash
curl -X POST https://<func-app>.azurewebsites.net/api/HttpStart

# Returns a statusQueryGetUri you poll:
curl "https://<func-app>.azurewebsites.net/runtime/webhooks/durabletask/instances/<instance-id>?..."
# {"runtimeStatus": "Running", ...}  then eventually "Completed"
```

Without Durable Functions, "wait for three parallel steps then continue"
would need a hand-rolled queue-and-state-table system — an orchestration
table tracking which steps finished, a queue trigger per step, manual
"am I done yet" checks. Durable Functions replaces all of that
boilerplate with ordinary-looking sequential code.

## Fan-out/fan-in and sub-orchestrations

The `payment_task`/`inventory_task` pair above is the **fan-out/fan-in**
pattern — launch N activities concurrently, then continue once all (or
some) complete. It generalizes to a dynamic list:

```python
tasks = [context.call_activity("ProcessItem", item) for item in order["items"]]
results = yield context.task_all(tasks)
```

For workflows big enough to warrant their own sub-workflow (e.g. "process
one line item" is itself multi-step), `context.call_sub_orchestrator(...)`
nests an orchestrator inside another, keeping each one readable.

## Custom bindings

Beyond the built-in HTTP/Timer/Blob/Queue/Cosmos DB triggers, Azure
Functions supports bindings for Event Grid, Event Hubs, Service Bus, and
SignalR, declared the same declarative way. An Event Grid trigger, for
example:

```json
{
  "bindings": [
    {
      "type": "eventGridTrigger",
      "name": "event",
      "direction": "in"
    }
  ]
}
```

```python
import azure.functions as func

def main(event: func.EventGridEvent):
    print(f"Event type: {event.event_type}, subject: {event.subject}")
```

**Gotcha:** each binding extension (Event Grid, Service Bus, Durable
Functions itself) needs its extension bundle registered in
`host.json` — a common "why isn't my trigger firing" cause is a missing
or outdated extension bundle version, not a code bug:

```json
{
  "version": "2.0",
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.*, 5.0.0)"
  }
}
```

## Deployment slots for Function Apps

Function Apps support the same slot mechanism as
[Module 01](01-app-service-deep-dive.md)'s Web Apps — deploy to `staging`,
verify, then swap:

```bash
az functionapp deployment slot create \
  --name $FUNCTION_APP \
  --resource-group rg-func-adv \
  --slot staging

func azure functionapp publish $FUNCTION_APP --slot staging

az functionapp deployment slot swap \
  --name $FUNCTION_APP \
  --resource-group rg-func-adv \
  --slot staging \
  --target-slot production
```

**Gotcha:** Function App slots are only available on **Premium** or
**Dedicated (App Service Plan)** hosting — not on the Consumption plan.
If you need zero-downtime function deploys, that alone can be the deciding
factor to move off Consumption despite losing scale-to-zero.

## How It Actually Works

The **Durable Functions** extension implements long-running orchestrations
without keeping a process alive the whole time, using a pattern called
**event sourcing**: every action an orchestrator function takes (calling an
activity, waiting for an external event, a timer) is appended as an event
to a history table (backed, on the default storage provider, by Azure
Table Storage plus queues for dispatch), and whenever the orchestration
needs to resume, the Durable Task Framework doesn't restart your
orchestrator function from scratch conceptually — it **replays** your
orchestrator code from the top, feeding it the recorded history so each
previously-completed `await` returns its cached result instantly rather
than re-executing, and only new code past the last recorded point actually
runs. This replay mechanism is exactly why orchestrator functions must be
deterministic (no `DateTime.Now`, no random, no direct I/O) — any
non-deterministic call would diverge from the recorded history on replay
and corrupt the orchestration's state.

**Deployment slots** for Function Apps work identically to the App Service
mechanism in Module 1 of this level — separate site directories and worker
processes swapped via Kudu — but with an added wrinkle for Consumption-plan
apps: because Consumption instances are ephemeral (spun up per invocation
by the scale controller), a "swap" there really just repoints which
deployment package's storage container the scale controller pulls from
when it next cold-starts an instance, rather than swapping a live running
process the way a dedicated-plan slot does. **Premium plan** ("Elastic
Premium") pre-warms a configurable number of instances specifically to
eliminate the cold-start problem described in Module 7 — those pre-warmed
instances sit idle with the language worker already loaded, and the scale
controller promotes them to serving traffic before provisioning any
additional cold instance, which is the concrete mechanism behind Premium's
"no cold start" guarantee for traffic within your pre-warmed capacity.

## Cheat sheet

| Concept | Purpose |
|---|---|
| Orchestrator function | Describes multi-step workflow logic; must be deterministic (no I/O, no `datetime.now()`, no random). |
| Activity function | Does the actual work (I/O, external calls); called via `context.call_activity`. |
| Client function | Starts a new orchestration instance, returns a status-check URL. |
| `context.task_all([...])` | Fan-out/fan-in: run tasks concurrently, wait for all. |
| `context.call_sub_orchestrator` | Nest one orchestration inside another. |
| `host.json` extensionBundle | Registers non-core trigger/binding types (Event Grid, Durable, Service Bus). |
| `az functionapp deployment slot create/swap` | Slots for Function Apps — Premium/Dedicated plans only. |

## Exercise

1. Build the `ProcessOrderOrchestrator` example above with at least one
   fan-out/fan-in step (`ValidateOrder` → `ChargePayment` +
   `ReserveInventory` in parallel → `SendConfirmation`).
2. Start an instance via the HTTP client function and poll its
   `statusQueryGetUri` until it reports `Completed`.
3. Add an Event Grid–triggered function and confirm the `host.json`
   extension bundle registers it correctly (check
   `func start`'s startup log for the function being listed).
4. Create a `staging` slot for the Function App (requires switching to a
   Premium/EP1 plan), deploy a change to it, and swap it into production.
5. Delete the resource group when finished.
