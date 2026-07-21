# 10 · Capstone Project — End-to-End Web App

Time to combine everything from this level into one working system: a static
frontend, a serverless API backend, and a managed database — deployed with
the `az` CLI, using only free-tier services, and **fully torn down** at the
end so nothing keeps billing after you're done.

## Architecture

```
Browser
  │
  ▼
Azure Static Web App  (HTML/JS frontend, free tier)
  │  fetch("/api/notes")
  ▼
Azure Function (Consumption plan)  ── linked backend of the Static Web App
  │
  ▼
Azure Cosmos DB (free tier, Core/SQL API)  — stores notes as JSON documents
```

You'll build a tiny "notes" app: a page with a form that posts a note, and a
list that fetches and displays all notes, backed by a Function App reading
and writing Cosmos DB. **Static Web Apps** can link directly to a Function
App as its "managed backend," which handles CORS and routing (`/api/*`) for
you automatically — no separate hosting or reverse proxy needed.

## Step 1 — Resource group and database

```bash
az group create --name rg-capstone --location eastus

az cosmosdb create \
  --resource-group rg-capstone \
  --name cosmos-capstone$RANDOM \
  --locations regionName=eastus failoverPriority=0 \
  --default-consistency-level Session \
  --enable-free-tier true

COSMOS_ACCOUNT=$(az cosmosdb list --resource-group rg-capstone --query "[0].name" --output tsv)

az cosmosdb sql database create \
  --account-name $COSMOS_ACCOUNT \
  --resource-group rg-capstone \
  --name notesdb

az cosmosdb sql container create \
  --account-name $COSMOS_ACCOUNT \
  --resource-group rg-capstone \
  --database-name notesdb \
  --name notes \
  --partition-key-path "/id" \
  --throughput 400

COSMOS_KEY=$(az cosmosdb keys list \
  --name $COSMOS_ACCOUNT --resource-group rg-capstone \
  --query primaryMasterKey --output tsv)
COSMOS_ENDPOINT=$(az cosmosdb show \
  --name $COSMOS_ACCOUNT --resource-group rg-capstone \
  --query documentEndpoint --output tsv)
```

## Step 2 — The Function App backend

```bash
az storage account create \
  --name stcapstonefn$RANDOM \
  --resource-group rg-capstone \
  --sku Standard_LRS
STORAGE_ACCOUNT=$(az storage account list --resource-group rg-capstone --query "[0].name" --output tsv)

az functionapp create \
  --resource-group rg-capstone \
  --name func-capstone$RANDOM \
  --storage-account $STORAGE_ACCOUNT \
  --consumption-plan-location eastus \
  --runtime python \
  --runtime-version 3.11 \
  --functions-version 4 \
  --os-type Linux

FUNCTION_APP=$(az functionapp list --resource-group rg-capstone --query "[0].name" --output tsv)

# Store the Cosmos DB connection as app settings (not hardcoded in source)
az functionapp config appsettings set \
  --resource-group rg-capstone \
  --name $FUNCTION_APP \
  --settings COSMOS_ENDPOINT="$COSMOS_ENDPOINT" COSMOS_KEY="$COSMOS_KEY"
```

Scaffold and write the function locally:

```bash
func init capstone-func --python
cd capstone-func
func new --name notes --template "HTTP trigger" --authlevel "anonymous"
```

`notes/__init__.py`:

```python
import json
import os
import uuid
import azure.functions as func
from azure.cosmos import CosmosClient

client = CosmosClient(
    url=os.environ["COSMOS_ENDPOINT"],
    credential=os.environ["COSMOS_KEY"],
)
container = client.get_database_client("notesdb").get_container_client("notes")


def main(req: func.HttpRequest) -> func.HttpResponse:
    if req.method == "POST":
        body = req.get_json()
        item = {"id": str(uuid.uuid4()), "text": body.get("text", "")}
        container.create_item(item)
        return func.HttpResponse(
            json.dumps(item), status_code=201, mimetype="application/json"
        )

    # GET: return all notes
    items = list(container.read_all_items())
    return func.HttpResponse(json.dumps(items), mimetype="application/json")
```

`requirements.txt`:

```
azure-functions
azure-cosmos
```

Test locally (using the real Cosmos DB endpoint via a local `local.settings.json`
with the same `COSMOS_ENDPOINT`/`COSMOS_KEY` values), then deploy:

```bash
func azure functionapp publish $FUNCTION_APP
```

## Step 3 — The Static Web App frontend

`www/index.html`:

```html
<!DOCTYPE html>
<html>
<head><title>Capstone Notes</title></head>
<body>
  <h1>Notes</h1>
  <form id="note-form">
    <input id="note-text" placeholder="Write a note..." required>
    <button type="submit">Add</button>
  </form>
  <ul id="note-list"></ul>

  <script>
    const list = document.getElementById("note-list");
    const form = document.getElementById("note-form");

    async function loadNotes() {
      const res = await fetch("/api/notes");
      const notes = await res.json();
      list.innerHTML = notes.map(n => `<li>${n.text}</li>`).join("");
    }

    form.addEventListener("submit", async (e) => {
      e.preventDefault();
      const text = document.getElementById("note-text").value;
      await fetch("/api/notes", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ text }),
      });
      document.getElementById("note-text").value = "";
      loadNotes();
    });

    loadNotes();
  </script>
</body>
</html>
```

Create the Static Web App and push this folder to a GitHub repo (Static Web
Apps deploy via a GitHub Actions workflow that `az` sets up for you):

```bash
az staticwebapp create \
  --resource-group rg-capstone \
  --name swa-capstone \
  --source https://github.com/<your-username>/capstone-notes \
  --location eastus2 \
  --branch main \
  --app-location "www" \
  --login-with-github
```

This command creates the GitHub Actions workflow file in your repo
automatically and triggers the first deploy on push.

### Link the Function App as the managed backend

```bash
az staticwebapp backends link \
  --name swa-capstone \
  --resource-group rg-capstone \
  --backend-resource-id $(az functionapp show \
      --name $FUNCTION_APP --resource-group rg-capstone --query id --output tsv) \
  --backend-region eastus
```

Linking a backend makes every request to `/api/*` from the Static Web App's
domain automatically route to your Function App, with CORS handled for you
— the frontend's `fetch("/api/notes")` just works with no extra
configuration.

## Step 4 — Verify it end-to-end

```bash
az staticwebapp show \
  --name swa-capstone --resource-group rg-capstone \
  --query defaultHostname --output tsv
```

Open `https://<hostname>/` in a browser, add a note through the form, refresh,
and confirm it persists (it's coming from Cosmos DB, not browser state).

## Step 5 — Teardown (do this — it stops all billing)

!!! warning "Don't skip this section"
    Cosmos DB's free tier is one account per subscription, Static Web Apps'
    free tier is generous but not unlimited, and a Function App on
    Consumption scales to zero but its Storage Account still holds data.
    None of this is expensive left running for a day, but the whole point
    of a capstone is to practice the full lifecycle — creation **and**
    clean shutdown — like you would with any real project.

The single command that undoes everything in this capstone:

```bash
az group delete --name rg-capstone --yes --no-wait
```

Confirm it's actually gone (don't just trust `--no-wait` and walk away):

```bash
# Poll until this returns an empty list / error "not found"
az group show --name rg-capstone --output table
```

If you also connected a custom domain or created a GitHub Actions secret
tied to this Static Web App, remove the now-broken GitHub Actions workflow
file/secret from your repo too, since the resource group deletion doesn't
touch anything living in GitHub:

```bash
# From your local clone of the capstone-notes repo
rm .github/workflows/azure-static-web-apps-*.yml
git add -A && git commit -m "Remove Azure Static Web Apps deploy workflow (resources torn down)"
git push
```

Finally, double-check nothing else is left in the subscription from earlier
modules in this level, since it's easy to forget a resource group from
Module 3 or 6:

```bash
az group list --output table
```

Delete anything you don't recognize as intentionally kept, the same way.

## Cheat sheet

| Command | Purpose |
|---|---|
| `az cosmosdb create --enable-free-tier true` | Free-tier Cosmos DB account. |
| `az functionapp create --consumption-plan-location --runtime` | Serverless Function App backend. |
| `az functionapp config appsettings set --settings` | Store connection info as app config, not hardcoded. |
| `az staticwebapp create --source --login-with-github` | Create a Static Web App wired to a GitHub repo + Actions deploy. |
| `az staticwebapp backends link --backend-resource-id` | Attach a Function App as the SWA's `/api/*` backend. |
| `az group delete --name --yes --no-wait` | Tear down every resource in the project at once. |
| `az group list -o table` | Sanity-check nothing else is still running. |

## Exercise

1. Build the notes app exactly as described, end to end, and confirm a note
   you add survives a page refresh (proving it round-trips through Cosmos
   DB, not just local JS state).
2. Add a `DELETE /api/notes/{id}` route to the Function App and a delete
   button per note in the frontend — this exercises writing a second HTTP
   route in the same function app and updating the container with a
   partition-key-aware delete call.
3. Run the full teardown sequence above, and verify with
   `az group show --name rg-capstone` that the group is gone.
4. As a final check, run `az group list --output table` and confirm your
   subscription is back to a clean state before moving on to Level 2.
