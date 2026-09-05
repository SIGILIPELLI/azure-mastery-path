# 01 · App Service & Web Apps Deep Dive

**Azure App Service** is Azure's fully managed platform for hosting web
apps, REST APIs, and mobile backends — you hand it code or a container, and
it handles the VM, OS patching, and web server underneath. Level 1's
capstone used a Static Web App and a Function App; this module is your
first real look at App Service **Web Apps** proper, and goes straight to
the features you need for anything beyond a toy deployment: deployment
slots, scaling, custom domains, and continuous deployment.

## App Service plans and pricing tiers

Every Web App runs on an **App Service Plan** — the underlying compute
(VM size, OS, region) that one or more apps share. The plan's **tier**
determines which features are available:

| Tier | Use case | Slots? | Autoscale? | Custom domain? |
|---|---|---|---|---|
| **F1 (Free)** | Learning/demos only, capped daily CPU quota | No | No | No |
| **B1/B2/B3 (Basic)** | Small production apps, manual scale-out | No | No | Yes |
| **S1/S2/S3 (Standard)** | Most production workloads | Yes (5) | Yes | Yes |
| **P1v3/P2v3/P3v3 (Premium v3)** | Higher scale, more slots, VNet integration | Yes (20) | Yes | Yes |

Create a plan and a Linux web app on it:

```bash
az group create --name rg-appsvc-demo --location eastus

az appservice plan create \
  --name asp-demo \
  --resource-group rg-appsvc-demo \
  --sku S1 \
  --is-linux

az webapp create \
  --name webapp-demo$RANDOM \
  --resource-group rg-appsvc-demo \
  --plan asp-demo \
  --runtime "PYTHON:3.11"

WEBAPP=$(az webapp list --resource-group rg-appsvc-demo --query "[0].name" --output tsv)
az webapp show --name $WEBAPP --resource-group rg-appsvc-demo --query defaultHostName --output tsv
# webapp-demo12345.azurewebsites.net
```

**Gotcha:** the app name becomes part of a globally unique
`*.azurewebsites.net` hostname — same uniqueness rule as storage accounts.
Also, moving *between* Linux and Windows plans isn't a simple config
change; you'd create a new plan and app.

## Deployment slots

A **deployment slot** is a fully separate, independently deployable copy of
your app running on the same plan — a `staging` slot lets you deploy and
smoke-test new code at its own URL
(`webapp-demo--staging.azurewebsites.net`) with zero impact on production,
then **swap** it in.

```bash
az webapp deployment slot create \
  --name $WEBAPP \
  --resource-group rg-appsvc-demo \
  --slot staging

# Deploy new code to staging only
az webapp deploy \
  --name $WEBAPP \
  --resource-group rg-appsvc-demo \
  --slot staging \
  --src-path ./app.zip \
  --type zip

# Swap staging into production once it looks good
az webapp deployment slot swap \
  --name $WEBAPP \
  --resource-group rg-appsvc-demo \
  --slot staging \
  --target-slot production
```

A swap is a **hot** operation — App Service warms up staging's instances
first, then swaps the routing so there's no cold start and (ideally) no
downtime. If something's wrong post-swap, swap again to instantly roll
back; the old production code is now sitting in the `staging` slot.

**Gotcha — "sticky" slot settings:** app settings and connection strings
swap by default, which is usually *wrong* for things like a
`ENVIRONMENT=production` flag or a slot-specific database connection
string. Mark settings that should **stay** with their slot (not follow the
code) as slot-specific:

```bash
az webapp config appsettings set \
  --name $WEBAPP \
  --resource-group rg-appsvc-demo \
  --slot-settings ENVIRONMENT=staging
```

## Scaling

**Manual scale-out** (Basic tier and up) sets a fixed instance count:

```bash
az appservice plan update \
  --name asp-demo \
  --resource-group rg-appsvc-demo \
  --number-of-workers 3
```

**Autoscale** (Standard tier and up) adds/removes instances based on rules
you define against a metric, via Azure Monitor:

```bash
az monitor autoscale create \
  --resource-group rg-appsvc-demo \
  --resource asp-demo \
  --resource-type Microsoft.Web/serverfarms \
  --name autoscale-demo \
  --min-count 1 \
  --max-count 5 \
  --count 1

az monitor autoscale rule create \
  --resource-group rg-appsvc-demo \
  --autoscale-name autoscale-demo \
  --condition "CpuPercentage > 70 avg 10m" \
  --scale out 1

az monitor autoscale rule create \
  --resource-group rg-appsvc-demo \
  --autoscale-name autoscale-demo \
  --condition "CpuPercentage < 30 avg 10m" \
  --scale in 1
```

**Gotcha:** autoscale rules react over a *window* (`avg 10m` above), not
instantly — a traffic spike lasting seconds won't trigger a scale-out
before it's already over. For genuinely spiky traffic, pair a lower CPU
threshold with a shorter window, and expect a few minutes of lag either
way; App Service isn't built for sub-minute reaction times the way
serverless (Functions Consumption) is.

## Custom domains and TLS

Mapping your own domain requires Basic tier or higher, and — for a root
(apex) domain — an `A` record plus a `TXT` record for domain ownership
verification (a subdomain like `www` just needs a `CNAME`):

```bash
# Add the domain (after you've created the DNS records at your registrar)
az webapp config hostname add \
  --webapp-name $WEBAPP \
  --resource-group rg-appsvc-demo \
  --hostname www.example.com

# Free, auto-renewing App Service Managed Certificate
az webapp config ssl create \
  --resource-group rg-appsvc-demo \
  --name $WEBAPP \
  --hostname www.example.com

az webapp config ssl bind \
  --resource-group rg-appsvc-demo \
  --name $WEBAPP \
  --certificate-thumbprint <thumbprint-from-previous-output> \
  --ssl-type SNI
```

**Gotcha:** the managed certificate can't cover root/apex domains on
every DNS provider (it needs to complete an HTTP validation challenge over
your `A` record), and it won't issue until the hostname is already
correctly mapped and resolving — do the DNS step first, wait for
propagation, *then* request the certificate.

## Continuous deployment

Point the app at a GitHub repo and App Service (via GitHub Actions) builds
and deploys on every push:

```bash
az webapp deployment github-actions add \
  --resource-group rg-appsvc-demo \
  --name $WEBAPP \
  --repo "your-org/your-repo" \
  --branch main \
  --login-with-github
```

This writes a `.github/workflows/*.yml` file into your repo that builds
the app and deploys it using a publish profile stored as a repo secret —
the same mechanism [Level 2, Module 05](05-azure-devops-cicd.md) covers
for Azure Pipelines instead of GitHub Actions. For a quick one-off deploy
without wiring up CI at all, `az webapp up` or `az webapp deploy --src-path
app.zip --type zip` push local code directly.

**Gotcha:** continuous deployment normally targets a single slot (often
`production` directly). For zero-downtime releases, target the `staging`
slot with CI and swap manually (or via a pipeline gate) rather than
letting every push go straight to production.

## How It Actually Works

An App Service Plan is a **pool of worker VM instances (a "scale set" under
the hood, though Azure never exposes it as one)** that your Web App is
deployed onto — multiple apps on the same plan share those same worker
instances, isolated from each other by process/container boundaries and
IIS's (or, on Linux, a per-app container's) app-pool sandboxing rather than
by separate VMs, which is exactly why apps on the same plan compete for the
plan's CPU/memory and why moving to a Premium/Isolated tier that gives you
dedicated workers is the fix for noisy-neighbor problems. Deployment slots
are not separate App Service Plans — they're **additional site directories
and app-pool processes on the same underlying worker instances**, each with
its own hostname and site-extension state; a **slot swap** doesn't copy
files or restart cold — it performs a coordinated warm-up of the target
slot's worker process and then atomically swaps the routing rules (and, for
non-sticky settings, the app settings) between the two slots' virtual IPs,
which is why swap is near-instant and why "sticky" app settings are
implemented as a flag that tells the swap engine to leave that specific
setting bound to the slot rather than following the code.

Autoscale rules are evaluated by a separate **Azure Monitor Autoscale**
service polling your plan's metrics on its own schedule (not a background
thread inside your app) — when a rule's threshold is crossed over its
configured window, Autoscale calls the same ARM `Microsoft.Web` scale API
you'd call manually to add/remove worker instances, then waits out its
cooldown period before evaluating again. Continuous deployment via GitHub
Actions or Azure DevOps works by triggering a **Kudu deployment**: Kudu
(App Service's built-in deployment engine, itself a separate site
alongside your app) receives the push, runs your build/deploy script inside
an isolated deployment container, and finally syncs the output into
`/home/site/wwwroot` on the worker's shared file storage (Azure Files-backed),
which is also why all instances of a scaled-out app see the same deployed
files without you copying anything per-instance.

## Cheat sheet

| Command | Purpose |
|---|---|
| `az appservice plan create --sku --is-linux` | Create the compute plan behind one or more web apps. |
| `az webapp create --plan --runtime` | Create a web app on a plan. |
| `az webapp deployment slot create --slot` | Create a staging (or other named) slot. |
| `az webapp deployment slot swap --slot --target-slot` | Swap a slot into production (or roll back). |
| `az webapp config appsettings set --slot-settings` | Mark a setting as slot-specific (doesn't swap). |
| `az appservice plan update --number-of-workers` | Manual scale-out to a fixed instance count. |
| `az monitor autoscale create` / `rule create` | Rule-based autoscale on a metric. |
| `az webapp config hostname add` | Map a custom domain. |
| `az webapp config ssl create/bind` | Issue and bind a free managed certificate. |
| `az webapp deployment github-actions add` | Wire up GitHub Actions CI/CD. |

## Exercise

1. Create an S1 App Service Plan and a Linux web app on it, and confirm you
   can reach the default `*.azurewebsites.net` hostname.
2. Create a `staging` slot, deploy a trivially different version of your
   app to it (e.g. change the homepage text), and confirm both slots serve
   different content at their own URLs.
3. Swap `staging` into production, confirm the change is now live at the
   production URL, then swap again to roll it back.
4. Create an autoscale profile with a scale-out rule at `CpuPercentage >
   70 avg 10m` and a scale-in rule at `< 30 avg 10m`, and read back the
   profile with `az monitor autoscale show`.
5. Delete the resource group when finished.
