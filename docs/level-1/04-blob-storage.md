# 04 · Blob Storage

A **storage account** is Azure's namespace for durable, highly available
storage — inside it you get **Blob Storage** (unstructured object storage:
files, backups, images, static websites), plus Files/Queues/Tables (covered
in [Level 2, Module 02](../level-2/02-storage-deep-dive.md)). This module
covers creating a storage account, containers, blobs, access tiers, and
hosting a static website directly from Blob Storage.

## Create a storage account

Storage account names must be **globally unique** across all of Azure,
lowercase letters and numbers only, 3-24 characters — you'll need to pick
something distinctive.

```bash
az group create --name rg-storage-demo --location eastus

az storage account create \
  --name stlearndemo$RANDOM \
  --resource-group rg-storage-demo \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2
```

- `--sku Standard_LRS` (Locally Redundant Storage) keeps 3 copies within one
  datacenter — the cheapest redundancy option, fine for learning. Other
  options: `Standard_ZRS` (zone-redundant), `Standard_GRS` (geo-redundant
  across regions).
- `--kind StorageV2` is the modern general-purpose account type that
  supports all the features in this module.

Save the account name to a variable for the rest of this module:

```bash
STORAGE_ACCOUNT=$(az storage account list \
  --resource-group rg-storage-demo \
  --query "[0].name" --output tsv)
echo $STORAGE_ACCOUNT
```

## Containers and blobs

A **container** is a namespace within a storage account (like a top-level
folder); a **blob** is any file stored inside a container.

```bash
# Get a connection string (or use --auth-mode login with your RBAC identity)
az storage container create \
  --account-name $STORAGE_ACCOUNT \
  --name photos \
  --auth-mode login

# Upload a file as a blob
echo "hello from azure" > hello.txt
az storage blob upload \
  --account-name $STORAGE_ACCOUNT \
  --container-name photos \
  --name hello.txt \
  --file hello.txt \
  --auth-mode login

# List blobs in a container
az storage blob list \
  --account-name $STORAGE_ACCOUNT \
  --container-name photos \
  --output table \
  --auth-mode login

# Download a blob
az storage blob download \
  --account-name $STORAGE_ACCOUNT \
  --container-name photos \
  --name hello.txt \
  --file downloaded.txt \
  --auth-mode login
```

`--auth-mode login` uses your signed-in RBAC identity (you need the
**Storage Blob Data Contributor** role, either inherited from Owner/
Contributor or assigned explicitly) instead of an account key — the
recommended approach over copying keys around.

## Access levels: public vs. private

Containers are **private** by default. You can make one publicly readable
(useful for a website's assets) at creation or after the fact:

```bash
az storage container create \
  --account-name $STORAGE_ACCOUNT \
  --name public-assets \
  --public-access blob \
  --auth-mode login
```

`--public-access` values: `off` (private, default), `blob` (anonymous read
of blobs, but container listing is not exposed), `container` (anonymous read
of blobs *and* the ability to list them). Use `off` unless you specifically
need anonymous access — prefer **SAS tokens** (below) for controlled,
time-limited sharing instead.

## Shared Access Signatures (SAS)

A **SAS token** grants time-limited, scope-limited access to a storage
resource without sharing your account key or requiring the requester to
have an Azure identity at all — the standard way to hand out a temporary
download/upload link.

```bash
# Generate a read-only SAS valid for 1 hour on a single blob
az storage blob generate-sas \
  --account-name $STORAGE_ACCOUNT \
  --container-name photos \
  --name hello.txt \
  --permissions r \
  --expiry $(date -u -d "1 hour" '+%Y-%m-%dT%H:%MZ') \
  --auth-mode login --as-user \
  --output tsv
```

(On macOS, replace the `date -u -d "1 hour"` GNU-ism with
`date -u -v+1H '+%Y-%m-%dT%H:%MZ'`.) Append the resulting query string to
the blob's URL to get a shareable, expiring link.

## Access tiers

Blob access tiers trade storage cost against retrieval cost/latency for
data you access less often:

| Tier | Best for | Notes |
|---|---|---|
| **Hot** | Frequently accessed data | Highest storage cost, cheapest access. |
| **Cool** | Infrequently accessed, stored ≥30 days | Lower storage cost, small per-GB retrieval fee. |
| **Cold** | Rarely accessed, stored ≥90 days | Lower still, higher retrieval fee than Cool. |
| **Archive** | Rarely accessed, stored ≥180 days | Cheapest storage; retrieval takes hours (must be "rehydrated" first). |

```bash
# Set the default tier for the account
az storage account create ... --access-tier Cool

# Or set the tier of one blob
az storage blob set-tier \
  --account-name $STORAGE_ACCOUNT \
  --container-name photos \
  --name hello.txt \
  --tier Cool \
  --auth-mode login
```

Lifecycle management policies (covered in [Level 2, Module 02](../level-2/02-storage-deep-dive.md))
automate moving blobs between tiers based on age.

## Static website hosting

Blob Storage can serve a static website directly — no VM or App Service
needed for a plain HTML/CSS/JS site.

```bash
az storage blob service-properties update \
  --account-name $STORAGE_ACCOUNT \
  --static-website \
  --index-document index.html \
  --404-document 404.html
```

This creates a special `$web` container and returns a public endpoint like
`https://stlearndemo1234.z13.web.core.windows.net/`. Upload your site's
files into `$web`:

```bash
echo "<h1>Hello from Azure Static Website</h1>" > index.html
az storage blob upload \
  --account-name $STORAGE_ACCOUNT \
  --container-name '$web' \
  --name index.html \
  --file index.html \
  --auth-mode login

az storage account show \
  --name $STORAGE_ACCOUNT \
  --query primaryEndpoints.web --output tsv
```

For a custom domain and CDN caching in front of this endpoint, pair it with
Azure CDN or Azure Front Door — beyond this level's scope, but the storage
side is exactly what you just did.

## Cheat sheet

| Command | Purpose |
|---|---|
| `az storage account create --sku --kind` | Create a storage account. |
| `az storage container create --public-access` | Create a container (private by default). |
| `az storage blob upload/download/list` | Upload, download, or list blobs. |
| `az storage blob generate-sas --permissions --expiry` | Create a time-limited shareable link. |
| `az storage blob set-tier --tier` | Change a blob's access tier. |
| `az storage blob service-properties update --static-website` | Turn on static website hosting. |
| Hot / Cool / Cold / Archive | Access tiers, cheapest storage to cheapest access in that order. |

## Exercise

1. Create a storage account and a private container called `docs`.
2. Upload two text files to it, list them, and download one back down.
3. Generate a 15-minute read-only SAS URL for one of the files and open it
   in a private/incognito browser tab to confirm it works without you being
   signed in.
4. Enable static website hosting on the same account, upload a one-line
   `index.html`, and load the resulting `.web.core.windows.net` URL in a
   browser.
5. Delete the resource group when done.
