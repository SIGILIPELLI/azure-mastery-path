# 01 · Setup & Azure CLI/Portal

Microsoft Azure is Microsoft's public cloud platform: virtual machines,
storage, databases, networking, and hundreds of managed services, all billed
by the minute/GB/request and managed either through the web-based **Azure
Portal** or the **`az` command-line interface (CLI)**. This module gets you a
free account, the CLI installed, and your first resource group created —
everything after this builds on those two things.

## Create a free Azure account

1. Go to [azure.microsoft.com/free](https://azure.microsoft.com/free/) and
   sign up with a Microsoft account (or create one).
2. New accounts get a 30-day/$200 USD credit plus a set of **"Always Free"**
   services (a small Linux/Windows VM, 5 GB of Blob Storage, a Cosmos DB
   free tier, Azure Functions' Consumption plan free grant, and more) that
   stay free indefinitely within their usage caps.
3. Signup requires a credit/debit card for identity verification, but Azure
   will not auto-upgrade to a paid plan or charge you without explicit
   action — the free trial simply stops working (or you can upgrade
   manually) once the credit or 30 days is used up.

!!! warning "Cost awareness"
    Every module in this course is written to fit inside the free tier or a
    few cents of usage. Still, get in the habit of deleting resource groups
    you create for practice (see the last section below) — an idle VM or
    database left running is the #1 way a free trial turns into a bill.

## Install the Azure CLI (`az`)

```bash
# macOS (Homebrew)
brew update && brew install azure-cli

# Windows (winget)
winget install -e --id Microsoft.AzureCLI

# Linux (Debian/Ubuntu)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

Verify the install:

```bash
az version
# {
#   "azure-cli": "2.6x.0",
#   ...
# }
```

If you'd rather not install anything locally, the Portal's top bar has a
**Cloud Shell** icon (`>_`) that opens a browser-based terminal with `az`
already installed and authenticated — every command in this course works
there too.

## Log in

```bash
az login
# Opens a browser window to sign in with your Microsoft account.
# On success, prints a JSON array of the subscriptions you can access.
```

On a machine without a browser (e.g. an SSH session), use the device-code
flow instead:

```bash
az login --use-device-code
# To sign in, use a web browser to open https://microsoft.com/devicelogin
# and enter the code XXXXXXXXX to authenticate.
```

## Subscriptions

A **subscription** is the billing + access-management boundary in Azure —
every resource you create belongs to exactly one subscription. A free
account gets one subscription automatically ("Azure subscription 1" or
"Free Trial").

```bash
# List subscriptions you can access
az account list --output table
# Name                  CloudName    SubscriptionId                        State    IsDefault
# --------------------  -----------  -------------------------------------  -------  -----------
# Azure subscription 1  AzureCloud   00000000-0000-0000-0000-000000000000  Enabled  True

# See which one is currently active
az account show --output table

# Switch the active subscription (if you have more than one)
az account set --subscription "Azure subscription 1"
```

## Resource groups

A **resource group** is a logical container that holds related resources
(VMs, storage accounts, databases, ...) for a project. Every resource
belongs to exactly one resource group, and a resource group belongs to one
region for its own metadata (the resources inside can span regions, though
in practice you usually keep them together). Deleting a resource group
deletes everything inside it — this is the single most useful cleanup
command in Azure.

```bash
# Create a resource group named "rg-learn" in the East US region
az group create --name rg-learn --location eastus

# List resource groups
az group list --output table

# Show details of one
az group show --name rg-learn

# Delete it (and everything inside it) when you're done experimenting
az group delete --name rg-learn --yes --no-wait
```

`--no-wait` returns immediately instead of blocking until deletion finishes
(deletion can take a few minutes) — useful when cleaning up at the end of a
session.

## Choosing a region

Pick a region close to you for lower latency, and stay consistent across a
project (most services can only talk to each other efficiently, or at all,
within the same region for private networking). List available regions:

```bash
az account list-locations --output table --query "[].{Name:name, DisplayName:displayName}"
```

Common ones used throughout this course: `eastus`, `westus2`, `westeurope`,
`centralindia`.

## Portal vs. CLI vs. Cloud Shell

| Tool | Best for |
|------|----------|
| **Azure Portal** (portal.azure.com) | Exploring what a service offers, visual dashboards, one-off clicks-based setup, reading docs inline via the "?" panel. |
| **`az` CLI** (local) | Repeatable, scriptable setup; what this course uses for every command so you can copy/paste and re-run. |
| **Cloud Shell** (browser, in the Portal) | Same CLI, zero local install, persists a small file share between sessions — good for a quick session from any machine. |
| **Bicep/ARM templates** | Declarative, version-controlled infrastructure — covered in [Module 9](09-arm-bicep-intro.md). |

This course uses the `az` CLI throughout because commands are copy/pasteable
and reproducible, but everything shown also has a Portal equivalent if you
prefer clicking through — the Portal is a good place to double check what a
command actually created.

## Cheat sheet

| Command | Purpose |
|---------|---------|
| `az login` | Authenticate the CLI to your Azure account. |
| `az account list -o table` | List subscriptions you can access. |
| `az account set --subscription <name-or-id>` | Switch the active subscription. |
| `az group create -n <name> -l <region>` | Create a resource group. |
| `az group list -o table` | List resource groups. |
| `az group delete -n <name> --yes --no-wait` | Delete a resource group and everything in it. |
| `az account list-locations -o table` | List available Azure regions. |
| `az version` | Show the installed CLI version. |

## Exercise

1. Sign up for a free Azure account if you haven't already, and run
   `az login` locally.
2. Create a resource group called `rg-module01` in a region near you.
3. Run `az group show --name rg-module01 --output table` to confirm it
   exists, then find it in the Portal under **Resource groups**.
4. Delete it with `az group delete --name rg-module01 --yes` (drop
   `--no-wait` this time so you can watch it finish) and confirm it's gone
   with `az group list --output table`.
