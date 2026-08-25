# 02 · Infrastructure as Code at Scale (Bicep/Terraform)

[Level 1, Module 09](../level-1/09-arm-bicep-intro.md) introduced a single
Bicep file deploying a handful of resources. At scale you're managing dozens
of environments and hundreds of resources — this module covers **modules**,
**remote state**, and **what-if/plan** review, in both Bicep and Terraform,
since real organizations run one or the other (sometimes both).

## Bicep modules

A **module** is a Bicep file referenced from another, letting you split a
big deployment into reusable, independently-testable pieces:

```bicep
// modules/storage.bicep
param location string
param storageAccountName string

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}

output storageAccountId string = storageAccount.id
```

```bicep
// main.bicep
param location string = 'eastus'
param envName string

module storage 'modules/storage.bicep' = {
  name: 'deploy-storage'
  params: {
    location: location
    storageAccountName: 'st${envName}${uniqueString(resourceGroup().id)}'
  }
}

output storageId string = storage.outputs.storageAccountId
```

```bash
az deployment group create \
  --resource-group rg-iac-scale \
  --template-file main.bicep \
  --parameters envName=prod
```

**Gotcha:** each `module` block creates a **nested deployment** visible in
the deployment history (`az deployment group list`) — a failure inside a
module shows up as a generic failure on the parent deployment unless you
drill into `az deployment operation group list` for the nested deployment
name, which trips up people expecting the top-level error message to be
specific.

## Bicep parameter files per environment

```json
// params.prod.json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "envName": { "value": "prod" },
    "location": { "value": "eastus" }
  }
}
```

```bash
az deployment group create \
  --resource-group rg-iac-scale \
  --template-file main.bicep \
  --parameters params.prod.json
```

## what-if before every apply

```bash
az deployment group what-if \
  --resource-group rg-iac-scale \
  --template-file main.bicep \
  --parameters params.prod.json
```

```text
Resource and property changes are indicated with these symbols:
  + Create
  ~ Modify
  - Delete

The deployment will update the following scope:

Scope: /subscriptions/xxxx/resourceGroups/rg-iac-scale

  ~ Microsoft.Storage/storageAccounts/stprodabc123
    ~ sku.name: "Standard_LRS" => "Standard_GRS"

Resource changes: 1 to modify.
```

**Gotcha:** `what-if` can under-report changes for properties Azure Resource
Manager treats as write-only or that get normalized server-side (some
network security rules, certain identity blocks) — treat it as a strong
signal, not a guarantee, and still review the deployment's actual result.

## Terraform equivalent

```hcl
# main.tf
terraform {
  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = "~> 3.0" }
  }
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "sttfstateshared"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}

provider "azurerm" {
  features {}
}

module "storage" {
  source              = "./modules/storage"
  location            = "eastus"
  storage_account_name = "st${var.env_name}${random_string.suffix.result}"
}

variable "env_name" {
  type = string
}
```

```bash
terraform init
terraform plan -var="env_name=prod" -out=prod.tfplan
terraform apply prod.tfplan
```

## Remote state in an Azure Storage backend

Terraform state must live somewhere shared and lockable so two engineers
never apply concurrently against stale state:

```bash
az group create --name rg-tfstate --location eastus

az storage account create \
  --resource-group rg-tfstate \
  --name sttfstateshared \
  --sku Standard_LRS \
  --encryption-services blob

az storage container create \
  --account-name sttfstateshared \
  --name tfstate
```

The `azurerm` backend block (above) points at this container; Terraform
uses a **blob lease** on the state file as its lock, so a crashed `apply`
can leave a stale lease — recover with:

```bash
terraform force-unlock <LOCK_ID>
```

**Gotcha:** never use `terraform force-unlock` on a lock you haven't
confirmed is actually stale (check with your team first) — forcing it while
another apply is genuinely in flight causes concurrent writes and a
corrupted state file, which is a much worse afternoon than waiting.

## Bicep vs. Terraform

| | Bicep | Terraform |
|---|---|---|
| **Scope** | Azure only | Multi-cloud |
| **State** | None — ARM tracks it | Explicit state file you manage |
| **Preview** | `what-if` | `plan` |
| **Syntax** | Declarative, ARM-native | HCL, provider-based |
| **Modules** | `.bicep` files | Registry or local modules |
| **Drift detection** | `az deployment group what-if` | `terraform plan` shows drift |

## Cheat sheet

| Command | Purpose |
|---|---|
| `az deployment group create --template-file main.bicep` | Deploy a Bicep template. |
| `az deployment group what-if` | Preview changes before applying. |
| `az deployment operation group list` | Inspect a failed nested deployment. |
| `terraform init` | Initialize providers and backend. |
| `terraform plan -out=plan.tfplan` | Preview and save a plan. |
| `terraform apply plan.tfplan` | Apply a previously reviewed plan. |
| `terraform force-unlock <ID>` | Clear a stale state lock (confirm first). |

## Exercise

1. Split a single-file Bicep deployment into a `modules/` folder with at
   least one module taking parameters and returning an output consumed by
   `main.bicep`.
2. Run `az deployment group what-if` against an existing resource group and
   read the diff before applying.
3. Set up a Terraform `azurerm` backend pointed at a storage container, run
   `terraform plan -out=plan.tfplan`, inspect it, then `apply` the saved
   plan (never apply without a reviewed plan file).
4. Deliberately interrupt a `terraform apply` (Ctrl+C) and practice
   diagnosing whether the resulting lock is safe to `force-unlock`.
5. Delete the resource group when finished.
