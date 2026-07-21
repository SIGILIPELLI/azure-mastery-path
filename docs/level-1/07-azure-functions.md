# 07 · Azure Functions

**Azure Functions** is Azure's serverless compute service: you write a small
function, it runs in response to a **trigger** (an HTTP request, a timer, a
new blob, a queue message, ...), and on the **Consumption plan** you pay only
for the time it actually executes — plus a generous free monthly grant. No
server to provision or patch.

## Triggers and bindings

- A **trigger** is what starts a function run (exactly one per function):
  HTTP request, Timer, Blob Storage, Queue Storage, Cosmos DB change feed,
  Event Grid, Service Bus, and more.
- A **binding** is a declarative way to read input from, or write output to,
  another service without writing SDK boilerplate — e.g. an "output binding"
  to Blob Storage lets you just return a value and have it written as a
  blob.

| Trigger type | Fires when |
|---|---|
| HTTP | A request hits the function's URL. |
| Timer | On a CRON-like schedule. |
| Blob | A blob is created/updated in a watched container. |
| Queue | A message arrives in a Storage Queue. |
| Cosmos DB | A document is inserted/updated (via the change feed). |
| Event Grid | An event is published (resource changes, custom events). |

## The Consumption plan

The **Consumption plan** is the default serverless hosting plan: Azure
allocates compute only while your function runs, scales to zero when idle,
and scales out automatically under load. The free grant is **1 million
requests and 400,000 GB-seconds of execution per month, forever** — most
learning and small production workloads never leave the free tier.

Other plans (Premium, App Service Plan, covered in
[Level 2, Module 09](../level-2/09-azure-functions-advanced.md)) trade the
scale-to-zero cold start for always-warm instances or VNet integration.

## Create a Function App

A **Function App** is the deployable/billable unit that hosts one or more
individual functions.

```bash
az group create --name rg-func-demo --location eastus

# Functions need a storage account for internal state (triggers, logs)
az storage account create \
  --name stfuncdemo$RANDOM \
  --resource-group rg-func-demo \
  --sku Standard_LRS

STORAGE_ACCOUNT=$(az storage account list --resource-group rg-func-demo --query "[0].name" --output tsv)

az functionapp create \
  --resource-group rg-func-demo \
  --name func-learn-demo$RANDOM \
  --storage-account $STORAGE_ACCOUNT \
  --consumption-plan-location eastus \
  --runtime python \
  --runtime-version 3.11 \
  --functions-version 4 \
  --os-type Linux

FUNCTION_APP=$(az functionapp list --resource-group rg-func-demo --query "[0].name" --output tsv)
```

## Write and run a function locally

Install Azure Functions Core Tools, then scaffold a project:

```bash
# macOS
brew tap azure/functions
brew install azure-functions-core-tools@4

func init hello-func --python
cd hello-func
func new --name HttpHello --template "HTTP trigger" --authlevel "function"
```

This generates `HttpHello/__init__.py`:

```python
import azure.functions as func

def main(req: func.HttpRequest) -> func.HttpResponse:
    name = req.params.get("name", "world")
    return func.HttpResponse(f"Hello, {name}! This ran on Azure Functions.")
```

Run it locally to test before deploying:

```bash
func start
# Functions:
#   HttpHello: [GET,POST] http://localhost:7071/api/HttpHello

curl "http://localhost:7071/api/HttpHello?name=Azure"
# Hello, Azure! This ran on Azure Functions.
```

## Deploy to Azure

```bash
func azure functionapp publish $FUNCTION_APP
```

```bash
# Get the deployed function's URL (includes its auth key for "function"-level auth)
az functionapp function show \
  --resource-group rg-func-demo \
  --name $FUNCTION_APP \
  --function-name HttpHello \
  --query invokeUrlTemplate --output tsv

curl "https://<func-app>.azurewebsites.net/api/HttpHello?code=<key>&name=Azure"
```

## A timer-triggered function

```bash
func new --name DailyCleanup --template "Timer trigger"
```

`DailyCleanup/function.json` sets the CRON-style schedule
(`{second} {minute} {hour} {day} {month} {day-of-week}`):

```json
{
  "bindings": [
    {
      "name": "mytimer",
      "type": "timerTrigger",
      "direction": "in",
      "schedule": "0 0 2 * * *"
    }
  ]
}
```

```python
import datetime
import azure.functions as func

def main(mytimer: func.TimerRequest) -> None:
    utc_now = datetime.datetime.utcnow().isoformat()
    print(f"DailyCleanup ran at {utc_now}")
```

`"0 0 2 * * *"` fires once a day at 02:00 UTC — useful for nightly cleanup,
report generation, or polling an external API on a schedule.

## Authorization levels

HTTP-triggered functions support three auth levels, set per function:

| Level | Behavior |
|---|---|
| `anonymous` | No key required — anyone with the URL can call it. |
| `function` | Requires a function-specific or host-wide key as a query param or header. |
| `admin` | Requires the master key — full admin access, rarely used for regular triggers. |

```bash
# List the keys for a function
az functionapp function keys list \
  --resource-group rg-func-demo \
  --name $FUNCTION_APP \
  --function-name HttpHello
```

## Cheat sheet

| Command | Purpose |
|---|---|
| `func init <name> --python` | Scaffold a new local Functions project. |
| `func new --name --template` | Add a function with a given trigger type. |
| `func start` | Run functions locally at `localhost:7071`. |
| `az functionapp create --consumption-plan-location --runtime` | Create a Function App on the Consumption plan. |
| `func azure functionapp publish <app>` | Deploy local code to Azure. |
| `az functionapp function keys list` | Get the key(s) needed to call a `function`-level HTTP trigger. |
| Trigger vs. binding | Trigger starts the run (one per function); bindings read/write data declaratively. |

## Exercise

1. Scaffold a Python Functions project locally with an HTTP trigger that
   accepts a `name` query parameter and returns a JSON greeting
   (`{"message": "Hello, <name>"}`) using `func.HttpResponse` with
   `mimetype="application/json"`.
2. Run it locally with `func start` and test with `curl`.
3. Create a Function App on Azure and deploy it with
   `func azure functionapp publish`.
4. Call the deployed URL (with its `function`-level key) and confirm you
   get the same JSON back.
5. Delete the resource group when finished.
