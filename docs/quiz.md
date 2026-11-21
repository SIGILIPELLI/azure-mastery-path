---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Azure Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "A new Azure free account is created. What happens when the credit or 30 days runs out?",
    "options": [
      "Everything is deleted at once",
      "Nothing changes",
      "It silently bills your card",
      "The trial stops working unless you manually upgrade; it won't auto-charge"
    ],
    "correctIndex": 3,
    "explanation": "A card is only for identity verification."
  },
  {
    "question": "An idle test VM is left running for a month. What's the likely result?",
    "options": [
      "Azure stops it",
      "A bill: idle VMs and databases are the #1 way a free trial costs money; delete practice resource groups",
      "Azure refunds it",
      "Nothing, it's free"
    ],
    "correctIndex": 1,
    "explanation": "Delete the resource group to clean up everything."
  },
  {
    "question": "A user needs read access to every resource in one resource group, and nothing else. Which RBAC setup?",
    "options": [
      "Reader at tenant root",
      "Owner at the subscription",
      "Assign Reader to the user (or a group) at that resource group's scope",
      "Contributor at management group"
    ],
    "correctIndex": 2,
    "explanation": "Narrow scope and least privilege; roles inherit downward."
  },
  {
    "question": "Which VM family suits a low-traffic dev box eligible for the free tier?",
    "options": [
      "F-series",
      "B-series (burstable), e.g. Standard_B1s",
      "E-series",
      "D-series"
    ],
    "correctIndex": 1,
    "explanation": "F is compute-optimised, E memory-optimised, D general purpose."
  },
  {
    "question": "A blob upload command should avoid account keys. Which option?",
    "options": [
      "--auth-mode login with an RBAC identity",
      "Share the key in code",
      "Disable auth",
      "Use anonymous access"
    ],
    "correctIndex": 0,
    "explanation": "Data-plane RBAC avoids key sprawl."
  },
  {
    "question": "A website needs routing /api to one backend pool and /images to another. Which service?",
    "options": [
      "Load Balancer (layer 4)",
      "Traffic stops",
      "A VNet",
      "Application Gateway (layer 7)"
    ],
    "correctIndex": 3,
    "explanation": "Load Balancer works on IP and port."
  },
  {
    "question": "A burst of orders must not overwhelm a slow processor. What decouples them?",
    "options": [
      "A tag",
      "A bigger VM",
      "A load test",
      "A queue (Storage Queue or Service Bus) between them"
    ],
    "correctIndex": 3,
    "explanation": "Messages wait until the consumer is ready."
  },
  {
    "question": "Event: 'a blob was created; notify five systems'. Event Grid or Service Bus?",
    "options": [
      "Service Bus queue",
      "Event Grid: reactive fan-out notifications",
      "Key Vault",
      "Table storage"
    ],
    "correctIndex": 1,
    "explanation": "Service Bus fits durable, ordered business messages."
  },
  {
    "question": "Your Cosmos DB uses a constant partition key. What goes wrong?",
    "options": [
      "Nothing",
      "It's cheaper",
      "A hot partition: one physical partition takes all traffic and is throttled even if total RU/s is available",
      "It's faster"
    ],
    "correctIndex": 2,
    "explanation": "Choose a high-cardinality key. Verified with a small distribution check."
  },
  {
    "question": "A Function App needs a secret. How should it get it?",
    "options": [
      "In a blob",
      "Through a managed identity reading Key Vault, with no secret in code",
      "Hard-coded in source",
      "In an app setting committed to Git"
    ],
    "correctIndex": 1,
    "explanation": "Grant the identity access via RBAC."
  },
  {
    "question": "Orchestrations must survive restarts and be replayed. Which Functions feature?",
    "options": [
      "Timer trigger",
      "Durable Functions (orchestrator, activity, client), with deterministic orchestrator code",
      "Blob trigger",
      "HTTP trigger"
    ],
    "correctIndex": 1,
    "explanation": "I/O belongs in activities."
  },
  {
    "question": "Which App Service tier is the first with deployment slots?",
    "options": [
      "Basic (B1)",
      "Free (F1)",
      "Shared",
      "Standard (S1+), with 5 slots"
    ],
    "correctIndex": 3,
    "explanation": "Premium v3 offers 20 slots."
  },
  {
    "question": "A deployment is previewed before applying. Which command?",
    "options": [
      "az group delete",
      "az vm start",
      "az deployment group what-if (Bicep/ARM)",
      "az login"
    ],
    "correctIndex": 2,
    "explanation": "Run it before every apply."
  },
  {
    "question": "You need a private, dedicated connection from on-prem to Azure with predictable performance. Which?",
    "options": [
      "Cloud Shell",
      "Site-to-site VPN over the internet",
      "ExpressRoute",
      "Public IP"
    ],
    "correctIndex": 2,
    "explanation": "VPN is encrypted over the public internet."
  },
  {
    "question": "An RTO of 1 hour and RPO of 5 minutes mean:",
    "options": [
      "Back in 5 minutes, 1 hour of loss",
      "Cost limits",
      "No loss, no downtime",
      "Service must be back within an hour, with at most five minutes of data loss"
    ],
    "correctIndex": 3,
    "explanation": "RTO is time to recover; RPO is data loss window."
  },
  {
    "question": "Which policy effect blocks non-compliant resource creation?",
    "options": [
      "Deny (Audit only reports)",
      "Disabled",
      "AuditIfNotExists",
      "Audit"
    ],
    "correctIndex": 0,
    "explanation": "Start with audit, then deny."
  },
  {
    "question": "Where should most guardrails be assigned so many subscriptions inherit them?",
    "options": [
      "In DNS",
      "At a management group",
      "In each app",
      "On each VM"
    ],
    "correctIndex": 1,
    "explanation": "Policy and RBAC inherit downward."
  },
  {
    "question": "Where do you analyse a request's path across microservices in Application Insights?",
    "options": [
      "NSG flow logs",
      "Cost analysis",
      "Policy",
      "Distributed tracing via operation_Id, with cloud_RoleName identifying each service"
    ],
    "correctIndex": 3,
    "explanation": "KQL with where operation_Id == ..."
  },
  {
    "question": "Costs must be attributed to teams. What foundation do you need?",
    "options": [
      "Larger VMs",
      "More regions",
      "A VPN",
      "A consistent tagging strategy enforced by Azure Policy"
    ],
    "correctIndex": 3,
    "explanation": "Tags drive showback and chargeback."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Plan non-overlapping subnets

Allocate subnets from a 10.0.0.0/16 VNet for web (120 hosts), app (250), db (60) and gateway (20). Azure reserves 5 addresses per subnet, so each needs hosts + 5. Use the smallest power-of-two block, largest first, aligned and never overlapping. Print each subnet and its usable count, then whether any overlap.

**Starter code**

```python
import ipaddress

VNET = ipaddress.ip_network("10.0.0.0/16")
WANTED = {"web": 120, "app": 250, "db": 60, "gateway": 20}   # hosts needed per subnet

def plan(vnet, wanted):
    # largest first; each subnet needs hosts + 5 (Azure reserves 5 addresses);
    # pick the smallest power-of-two block, align it, and never overlap
    pass

result = plan(VNET, WANTED)
for name in WANTED:
    n = result[name]
    print(f"{name}: {n} usable={n.num_addresses - 5}")
nets = list(result.values())
print("overlaps:", any(a.overlaps(b) for i, a in enumerate(nets) for b in nets[i + 1:]))
```

**Expected output**

```text
web: 10.0.1.0/25 usable=123
app: 10.0.0.0/24 usable=251
db: 10.0.1.128/25 usable=123
gateway: 10.0.2.0/27 usable=27
overlaps: False
```

<details><summary>Show solution and explanation</summary>

<pre><code>import ipaddress

VNET = ipaddress.ip_network(&quot;10.0.0.0/16&quot;)
WANTED = {&quot;web&quot;: 120, &quot;app&quot;: 250, &quot;db&quot;: 60, &quot;gateway&quot;: 20}   # hosts needed per subnet

def plan(vnet, wanted):
    &quot;&quot;&quot;Allocate the smallest subnet that fits each request (Azure reserves 5 addresses per subnet).&quot;&quot;&quot;
    cursor = int(vnet.network_address)
    out = {}
    for name, hosts in sorted(wanted.items(), key=lambda kv: -kv[1]):
        needed = hosts + 5
        prefix = 32
        while 2 ** (32 - prefix) &lt; needed:
            prefix -= 1
        size = 2 ** (32 - prefix)
        cursor = (cursor + size - 1) // size * size      # align to the subnet size
        net = ipaddress.ip_network((cursor, prefix))
        assert net.subnet_of(vnet)
        out[name] = net
        cursor += size
    return out

result = plan(VNET, WANTED)
for name in WANTED:
    n = result[name]
    print(f&quot;{name}: {n} usable={n.num_addresses - 5}&quot;)
nets = list(result.values())
print(&quot;overlaps:&quot;, any(a.overlaps(b) for i, a in enumerate(nets) for b in nets[i + 1:]))
</code></pre>

<p>Azure reserves five addresses in each subnet, so a /24 gives 251 usable. Planning address space up front avoids overlaps that block peering later. Verified by running the solution.</p>
</details>

### Spot a hot partition

Write hot_partitions(keys, partitions=4, threshold=0.4): hash each key to a partition (sum of character codes modulo partitions) and return {partition: share} for any partition holding more than the threshold of requests. Compare a well-distributed key set with a tenant-skewed one.

**Starter code**

```python
from collections import Counter

def hot_partitions(keys, partitions=4, threshold=0.4):
    # hash each key to a partition (sum of char codes % partitions),
    # return {partition: share} for partitions holding more than `threshold` of requests
    pass

good = [f"user-{i}" for i in range(100)]
bad = ["tenant-a"] * 70 + [f"tenant-{i}" for i in range(30)]
print("good:", hot_partitions(good))
print("bad:", hot_partitions(bad))
```

**Expected output**

```text
good: {}
bad: {0: 0.78}
```

<details><summary>Show solution and explanation</summary>

<pre><code>from collections import Counter

def hot_partitions(keys, partitions=4, threshold=0.4):
    &quot;&quot;&quot;Hash each partition key to a physical partition and flag any holding more than `threshold` of requests.&quot;&quot;&quot;
    def h(k):  # simple stable hash
        return sum(ord(c) for c in k) % partitions
    counts = Counter(h(k) for k in keys)
    total = len(keys)
    return {p: round(c / total, 2) for p, c in sorted(counts.items()) if c / total &gt; threshold}

good = [f&quot;user-{i}&quot; for i in range(100)]
bad = [&quot;tenant-a&quot;] * 70 + [f&quot;tenant-{i}&quot; for i in range(30)]
print(&quot;good:&quot;, hot_partitions(good))
print(&quot;bad:&quot;, hot_partitions(bad))
</code></pre>

<p>A partition key with low cardinality or skew concentrates traffic on one physical partition, which gets throttled even when total RU/s is available. Verified by running the solution.</p>
</details>
