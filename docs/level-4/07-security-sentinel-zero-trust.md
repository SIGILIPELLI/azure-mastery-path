# 07 · Security Deep Dive (Sentinel, Zero Trust)

[Level 3, Module 06](../level-3/06-defender-governance.md) covered
Defender for Cloud (posture and per-resource threat detection) and Policy
(enforcement). This module covers **Microsoft Sentinel** (SIEM/SOAR —
correlating signals *across* resources to find attacks that no single
resource's alerts would reveal) and the **Zero Trust** principles that
should shape how you design access in the first place.

## Onboarding Sentinel

Sentinel is Log Analytics with a security layer on top — it needs a
workspace as its data store:

```bash
az sentinel workspace create \
  --resource-group rg-security \
  --workspace-name law-sentinel

az monitor log-analytics workspace table create \
  --resource-group rg-security \
  --workspace-name law-sentinel \
  --name SecurityEvent
```

Data connectors ingest signal from Azure AD sign-ins, Defender for Cloud
alerts, firewall logs, and third-party sources:

```bash
az security data-connector create \
  --resource-group rg-security \
  --name aad-connector \
  --kind AzureActiveDirectory
```

**Gotcha:** Sentinel bills on **data ingestion volume**, and verbose data
sources (raw firewall/NSG flow logs at debug verbosity) can dominate cost
without proportionally improving detection — most teams tune ingestion to
security-relevant log categories and use **Basic Logs** tier (cheaper,
limited query capability) for high-volume, low-value-per-event data instead
of ingesting everything at full analytics tier by default.

## Analytics rules and correlation

A single failed sign-in is noise; the same account failing sign-in from
five countries within ten minutes is a signal. Sentinel's **analytics
rules** express this correlation in KQL:

```kusto
SigninLogs
| where ResultType != "0"
| summarize FailedAttempts = count(), Countries = make_set(LocationDetails.countryOrRegion) by UserPrincipalName, bin(TimeGenerated, 10m)
| where FailedAttempts > 5 and array_length(Countries) > 2
```

```bash
az sentinel alert-rule create \
  --resource-group rg-security \
  --workspace-name law-sentinel \
  --rule-id impossible-travel-failures \
  --kind Scheduled \
  --display-name "Multiple countries failed sign-in" \
  --query-frequency PT10M \
  --query-period PT10M \
  --severity High
```

**Gotcha:** a scheduled analytics rule only evaluates data that has
**already landed** in the workspace — ingestion latency for some connectors
runs several minutes behind real-time, so a `PT10M` (10-minute) query
period rule effectively has 10-15+ minutes of end-to-end detection lag, not
near-instant; size incident response expectations accordingly rather than
assuming SIEM detection is real-time.

## Automated response with playbooks

A **playbook** (a Logic App triggered by an incident) automates the
response — disable a user, isolate a VM, open a ticket — without a human
manually working every incident:

```bash
az logic workflow create \
  --resource-group rg-security \
  --name playbook-disable-user \
  --definition playbook-disable-user.json

az sentinel automation-rule create \
  --resource-group rg-security \
  --workspace-name law-sentinel \
  --automation-rule-name auto-respond-impossible-travel \
  --display-name "Auto-disable on impossible travel" \
  --order 1 \
  --triggering-logic '{"triggersOn":"Incidents","triggersWhen":"Created"}' \
  --actions '[{"actionType":"RunPlaybook","logicAppResourceId":"/subscriptions/xxxx/resourceGroups/rg-security/providers/Microsoft.Logic/workflows/playbook-disable-user"}]'
```

**Gotcha:** a playbook that automatically disables accounts on a
high-sensitivity/high-severity trigger is powerful but risky if the
detection rule has false positives — a legitimate traveling executive
triggering "impossible travel" and getting auto-disabled generates its own
incident. Start automation on lower-blast-radius actions (create a ticket,
tag an incident) and only automate destructive responses once a rule's
false-positive rate is proven low over time.

## Zero Trust principles applied

Zero Trust means: **never trust based on network location alone; verify
every request explicitly.** Concretely on Azure:

- **Verify explicitly** — Conditional Access requiring MFA + compliant
  device, not just "inside the corporate VPN."
- **Least privilege access** — PIM (Privileged Identity Management) for
  just-in-time elevation instead of standing `Owner` roles.
- **Assume breach** — network segmentation (the hub-and-spoke + NSGs from
  earlier levels) so a compromised workload can't reach everything.

```bash
az rest --method POST \
  --uri "https://graph.microsoft.com/v1.0/identityGovernance/privilegedAccess/azureResources/roleAssignmentScheduleRequests" \
  --body '{
    "action": "selfActivate",
    "principalId": "<user-object-id>",
    "roleDefinitionId": "<owner-role-def-id>",
    "resourceId": "<subscription-id>",
    "scheduleInfo": { "expiration": { "type": "afterDuration", "duration": "PT8H" } },
    "justification": "Emergency prod investigation"
  }'
```

**Gotcha:** PIM's just-in-time elevation is only as strong as the approval
workflow behind it — a PIM role configured with **no approval required**
and a generic justification field that isn't reviewed provides an audit
trail but not actual access control; the security value comes from
requiring approval and MFA at activation time, not from PIM's mere
existence.

## Sentinel vs. Defender for Cloud

| | Defender for Cloud | Sentinel |
|---|---|---|
| **Scope** | Per-resource posture and threat detection | Cross-resource correlation, SIEM/SOAR |
| **Data model** | Resource-specific recommendations/alerts | Log Analytics tables, KQL-queried |
| **Response** | Recommendations, some auto-remediation | Playbooks (Logic Apps), automation rules |
| **Best for** | "Is this VM/storage account configured securely?" | "Is there a coordinated attack across my environment?" |

## How It Actually Works

**Microsoft Sentinel** is built directly on top of the Log Analytics
workspace and Kusto engine already covered in Level 1 and Module 5 above —
enabling Sentinel on a workspace doesn't add a new data store, it adds a
**detection and orchestration layer** on top: scheduled **analytics rules**
are, mechanically, KQL queries the Sentinel engine runs against the
workspace on a fixed interval, and a rule's match becomes an "incident"
object tracked in Sentinel's own incident data model, which is distinct
from Defender for Cloud's alerts (Module 6, Level 3) even though both
ultimately correlate signals in the same workspace — Sentinel additionally
supports **automation rules and playbooks**, where a matched incident can
trigger a Logic App (the same Logic Apps engine used for general workflow
automation) to auto-remediate, e.g. calling the Entra API to disable a
compromised account, closing the loop from detection to response inside
one platform.

**Zero Trust** as implemented via Entra Conditional Access is not a single
feature but a **policy evaluation gate inserted into the OAuth token-
issuance flow** itself: when a user or app requests a token from Entra
(the same `/authorize` and `/token` endpoints from Module 1's `az login`),
Entra evaluates every applicable Conditional Access policy's conditions
(user risk score from Entra ID Protection's ML-based sign-in risk model,
device compliance state reported by Intune, network location, requested
resource) *before* issuing the token, and can deny, require MFA step-up, or
grant a token restricted to a session-bound scope — this is the mechanical
reason Zero Trust policies apply uniformly across every Azure/Microsoft 365
service: they're enforced at the shared token-issuance chokepoint every
one of those services relies on, not re-implemented per application.

## Cheat sheet

| Command | Purpose |
|---|---|
| `az sentinel workspace create` | Onboard Sentinel onto a Log Analytics workspace. |
| `az security data-connector create` | Ingest a data source (Azure AD, Defender, etc.). |
| `az sentinel alert-rule create --kind Scheduled` | Create a KQL-based correlation/detection rule. |
| `az sentinel automation-rule create` | Auto-trigger a playbook when an incident is created. |
| `az logic workflow create` | Create the Logic App playbook itself. |
| PIM `roleAssignmentScheduleRequests` (Graph API) | Request just-in-time privileged role activation. |

## Exercise

1. Onboard Sentinel on a Log Analytics workspace and enable the Azure AD
   sign-in data connector.
2. Write a KQL analytics rule detecting more than N failed sign-ins for one
   user within a short window, and create it as a scheduled rule.
3. Create a low-risk automation rule (tag/comment an incident, not a
   destructive action) triggered on incident creation.
4. Configure a PIM role assignment requiring approval and MFA for
   activation, and explain why "no approval required" defeats the purpose.
5. Delete the resource group when finished.
