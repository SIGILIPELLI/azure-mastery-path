# 05 · Azure DevOps & CI/CD Pipelines

[Module 01](01-app-service-deep-dive.md) used GitHub Actions for
continuous deployment. **Azure DevOps** is Microsoft's own alternative:
Repos (Git hosting), Pipelines (YAML-based CI/CD), Boards (work tracking),
and Artifacts (package feeds) — this module covers Repos and Pipelines,
the two you'll use for any real Azure-centric CI/CD setup.

## Organizations, projects, and repos

Azure DevOps is organized as **Organization → Project → Repos/Pipelines**.
Create these from the [Azure DevOps portal](https://dev.azure.com) (there's
no `az` CLI equivalent for creating the organization itself — that's a
one-time manual step), then everything after that can be scripted with the
`az devops` extension:

```bash
az extension add --name azure-devops

az devops configure --defaults organization=https://dev.azure.com/your-org

az devops project create --name capstone-app

az repos create --project capstone-app --name capstone-repo
```

```bash
# Clone it and push existing code
git clone https://dev.azure.com/your-org/capstone-app/_git/capstone-repo
cd capstone-repo
git add -A && git commit -m "Initial commit"
git push origin main
```

**Gotcha:** the first `git push` to an Azure Repos remote prompts for
authentication — use a **Personal Access Token (PAT)** (Azure DevOps →
User Settings → Personal Access Tokens) as the password, not your regular
Azure AD password, especially once your org enforces MFA.

## Pipeline YAML basics

An Azure Pipeline is defined in `azure-pipelines.yml` at the repo root —
stages, jobs, and steps, conceptually identical to a GitHub Actions
workflow file but with its own syntax:

```yaml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: ubuntu-latest

stages:
  - stage: Build
    jobs:
      - job: BuildAndTest
        steps:
          - task: UsePythonVersion@0
            inputs:
              versionSpec: "3.11"
          - script: pip install -r requirements.txt
            displayName: "Install dependencies"
          - script: pytest
            displayName: "Run tests"
          - task: ArchiveFiles@2
            inputs:
              rootFolderOrFile: "$(System.DefaultWorkingDirectory)"
              includeRootFolder: false
              archiveFile: "$(Build.ArtifactStagingDirectory)/app.zip"
          - task: PublishBuildArtifacts@1
            inputs:
              pathToPublish: "$(Build.ArtifactStagingDirectory)"
              artifactName: "drop"

  - stage: Deploy
    dependsOn: Build
    condition: succeeded()
    jobs:
      - deployment: DeployToAppService
        environment: production
        pool:
          vmImage: ubuntu-latest
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: "azure-service-connection"
                    appName: "webapp-demo"
                    package: "$(Pipeline.Workspace)/drop/app.zip"
```

Register and run it:

```bash
az pipelines create \
  --name capstone-ci \
  --repository capstone-repo \
  --repository-type tls \
  --branch main \
  --yml-path azure-pipelines.yml
```

**Stage** = a logical phase (Build, Deploy); **job** = a set of steps that
run on one agent; **step** = a single `task` (a packaged, reusable action)
or inline `script`. The `dependsOn`/`condition` pair on the `Deploy` stage
is what makes deployment wait for — and depend on — a successful build.

## Service connections

A **service connection** is Azure DevOps's stored credential for talking
to an external system — in `AzureWebApp@1` above, `azureSubscription:
"azure-service-connection"` refers to one pointing at your Azure
subscription, created once via the portal (Project Settings → Service
connections → Azure Resource Manager) or `az devops service-endpoint
azurerm create`. It typically backs onto a **service principal** or
workload identity federation, so the pipeline authenticates to Azure
without a long-lived secret sitting in the YAML file.

**Gotcha:** service connections are scoped either to the whole
subscription or to a single resource group — always scope to the
narrowest resource group that makes sense for a given pipeline, so a
compromised or misconfigured pipeline can't touch unrelated resources.

## Variables and secrets

```yaml
variables:
  - name: appName
    value: webapp-demo
  - group: capstone-secrets   # variable group, can hold secrets
```

Secret variables (API keys, connection strings) belong in a **variable
group** (Pipelines → Library) or, better, in **Key Vault** referenced by
that group — never hardcoded in the YAML, which is plaintext in Git
history forever once committed.

```yaml
variables:
  - group: capstone-secrets   # linked to a Key Vault, values pulled at run time
steps:
  - script: echo "Using $(DB_CONNECTION_STRING)"   # value is masked in logs
```

Pipeline logs automatically mask any value matching a secret variable, but
that's a safety net, not a guarantee — avoid `echo`-ing secrets directly
regardless.

## Multi-stage approvals

Add a manual approval gate before a production deploy stage runs (Pipelines
→ Environments → your environment → Approvals and checks, in the portal) —
YAML alone can't fully express this, since it's tied to the **Environment**
object, not the pipeline file. Once configured, the `Deploy` stage above
pauses and emails/notifies the designated approvers before it proceeds.

## How It Actually Works

An Azure Pipelines run isn't executed by the Azure DevOps service itself —
YAML pipeline definitions are compiled into a **job graph**, and each job is
dispatched to an **agent**: either a Microsoft-hosted agent (a fresh,
ephemeral VM image spun up per job from a pool of pre-imaged VMs and
destroyed after, which is why hosted-agent builds always start from a clean
environment) or a self-hosted agent (a long-lived process you register that
polls the Azure DevOps service for queued jobs over an outbound HTTPS
connection — this is why self-hosted agents work behind a firewall with no
inbound ports open at all: the agent always initiates the connection).
Each pipeline stage's artifacts are published to a service-managed blob
store scoped to the pipeline run, and a later stage's `download` step pulls
from that same store — stages don't share a filesystem or VM.

**Approvals and gates** on multi-stage pipelines are implemented as a
pause the orchestration engine inserts before a stage's jobs are dispatched
to any agent: the pipeline run's state machine transitions to "pending
approval" and literally does not queue the stage's job until an authorized
approver acts (or the approval times out), which is why an approval gate
costs nothing in agent minutes while it's waiting. A **service connection**
to Azure is, under the hood, either a stored service-principal client
secret/certificate or an Entra **workload identity federation** trust
(newer, secret-less) — the pipeline's agent exchanges that credential for a
short-lived ARM access token at deploy time using the exact same OAuth
client-credentials flow a service principal uses anywhere else, meaning
deployment permissions are entirely governed by whatever RBAC role that
service principal holds, independent of the human who authored the
pipeline.

## Cheat sheet

| Command / concept | Purpose |
|---|---|
| `az devops project create` / `az repos create` | Create a project and a Git repo. |
| `azure-pipelines.yml` (`trigger`, `pool`, `stages`) | Define the CI/CD pipeline as code. |
| `az pipelines create --yml-path` | Register a pipeline from a YAML file. |
| Service connection | Stored credential a pipeline uses to reach Azure. |
| Variable group / Key Vault link | Store secrets outside the YAML file. |
| Environment approvals | Manual gate before a stage (e.g. production deploy) runs. |

## Exercise

1. Create an Azure DevOps project and repo, and push a small app to it.
2. Write an `azure-pipelines.yml` with a `Build` stage (install deps, run
   tests, publish an artifact) and a `Deploy` stage that depends on it.
3. Create a service connection to your Azure subscription and use it in an
   `AzureWebApp@1` task to deploy to a Web App from
   [Module 01](01-app-service-deep-dive.md).
4. Add a variable group with at least one secret value, and reference it
   in a step without ever printing it directly.
5. Add a manual approval check on the `Deploy` stage's environment and
   confirm a new run pauses for approval before deploying.
