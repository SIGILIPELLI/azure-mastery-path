# 08 · Cosmos DB Deep Dive

[Level 1's capstone](../level-1/10-capstone-project.md) used Cosmos DB as
a black box: create account, create container, read/write items. This
module opens the box — **partitioning strategy**, **consistency models**,
**request units (RUs)**, and the **change feed** — the four concepts that
determine whether a Cosmos DB design scales cheaply or falls over (or gets
expensive) under real load.

## Partitioning strategy

Every container has a **partition key** — the property Cosmos DB uses to
distribute data (and throughput) across physical partitions. Choosing it
well is the single most consequential Cosmos DB design decision.

```bash
az group create --name rg-cosmos-deep --location eastus

az cosmosdb create \
  --resource-group rg-cosmos-deep \
  --name cosmos-deep$RANDOM \
  --locations regionName=eastus failoverPriority=0 \
  --default-consistency-level Session

COSMOS_ACCOUNT=$(az cosmosdb list --resource-group rg-cosmos-deep --query "[0].name" --output tsv)

az cosmosdb sql database create \
  --account-name $COSMOS_ACCOUNT \
  --resource-group rg-cosmos-deep \
  --name ordersdb

az cosmosdb sql container create \
  --account-name $COSMOS_ACCOUNT \
  --resource-group rg-cosmos-deep \
  --database-name ordersdb \
  --name orders \
  --partition-key-path "/customerId" \
  --throughput 400
```

A good partition key spreads both **storage** and **request volume**
evenly, and matches how you actually query (queries filtered by partition
key are cheap; cross-partition **fan-out queries** are expensive and slower
at scale). `/customerId` above is reasonable if most queries are "give me
this customer's orders" — every order for one customer lands together,
and that's also the common access pattern.

**Gotcha — hot partitions:** a partition key with low cardinality (few
distinct values) or skewed access (one value gets 90% of traffic — e.g.
partitioning by `/tenantId` when one tenant is 100x bigger than the rest)
creates a **hot partition**: Cosmos DB throttles that partition's requests
long before your account-level RU budget is exhausted, even though the
account "has RUs to spare" overall. Once chosen, a partition key **cannot
be changed** without migrating to a new container — get this right early.

## Request Units (RUs)

Every operation costs a number of **Request Units** — a normalized measure
of the CPU/memory/IO Cosmos DB spends on it. A 1 KB point read costs
roughly 1 RU; writes, larger documents, and queries scanning more data
cost more.

```bash
# Check current throughput
az cosmosdb sql container throughput show \
  --account-name $COSMOS_ACCOUNT \
  --resource-group rg-cosmos-deep \
  --database-name ordersdb \
  --name orders

# Scale up (manual/provisioned throughput)
az cosmosdb sql container throughput update \
  --account-name $COSMOS_ACCOUNT \
  --resource-group rg-cosmos-deep \
  --database-name ordersdb \
  --name orders \
  --throughput 1000
```

`--throughput 1000` provisions 1000 RU/s, billed whether or not you use
it — the always-on tradeoff of **provisioned throughput**. The
alternative, **autoscale throughput**, scales automatically between 10%
and 100% of a max you set, billed for whatever tier it actually used that
hour:

```bash
az cosmosdb sql container throughput update \
  --account-name $COSMOS_ACCOUNT \
  --resource-group rg-cosmos-deep \
  --database-name ordersdb \
  --name orders \
  --max-throughput 4000
```

Or, for spiky/unpredictable workloads, **serverless** mode (an account-level
choice at creation time, `--capabilities EnableServerless`) bills purely
per-RU-consumed with no provisioning at all — the best fit for dev/test or
low, unpredictable traffic, but it's mutually exclusive with provisioned
throughput on the same account.

**Gotcha:** exceeding provisioned RU/s doesn't fail the request outright —
the SDK gets a `429 Too Many Requests` with a `Retry-After` header and (in
most SDKs) automatically retries after that delay. If you're seeing lots of
429s, that's a signal to raise throughput, fix a hot partition, or both —
not necessarily a bug in your query.

## Consistency models

Cosmos DB offers five consistency levels, a spectrum between "always
perfectly consistent, slower/costlier" and "eventually consistent, fastest,
cheapest":

| Level | Guarantee |
|---|---|
| **Strong** | Reads always see the latest committed write. Highest latency/cost. |
| **Bounded Staleness** | Reads lag writes by at most K versions or T time. |
| **Session** (default) | Within one client "session," always read your own writes. Good default. |
| **Consistent Prefix** | Reads never see out-of-order writes (but may be stale). |
| **Eventual** | No ordering guarantee at all. Lowest latency/cost. |

```bash
az cosmosdb update \
  --resource-group rg-cosmos-deep \
  --name $COSMOS_ACCOUNT \
  --default-consistency-level BoundedStaleness \
  --max-staleness-prefix 100000 \
  --max-interval 300
```

**Session** (the default, and the right choice for most apps) guarantees a
single client always sees its own writes immediately, while different
clients might briefly see slightly different states — the sweet spot for
things like "user submits a comment, then their own page refresh shows it"
without paying Strong consistency's global-coordination cost.

**Gotcha:** the account-level setting is the *default* and *weakest*
allowed for that account; individual requests can request a **stronger**
level per-call (never weaker) via the SDK — useful for the rare read that
genuinely needs Strong without paying that cost on every request.

## The change feed

The **change feed** is a persistent, ordered log of every insert/update to
a container (not deletes, by default) — the mechanism behind "run this
function whenever a document changes," without polling.

```python
from azure.cosmos import CosmosClient

client = CosmosClient(url=COSMOS_ENDPOINT, credential=COSMOS_KEY)
container = client.get_database_client("ordersdb").get_container_client("orders")

response = container.query_items_change_feed(is_start_from_beginning=True)
for item in response:
    print(f"Changed: {item['id']}")
```

In practice, the change feed is most often consumed via an **Azure
Functions Cosmos DB trigger** (mentioned in
[Level 1, Module 07](../level-1/07-azure-functions.md)'s trigger table),
which manages the "read from where I left off" checkpointing for you — a
common pattern is fan-out: order written → change feed trigger fires →
Function sends a notification, updates a search index, and writes an audit
record, all without the original write path knowing any of that happens.

**Gotcha:** the change feed shows the **current state** of a changed
document, not a diff of what fields changed — if you need to know
specifically *which* field changed, you must compare against a
previously stored version yourself; Cosmos DB doesn't hand you a delta.

## How It Actually Works

The **change feed** is not a message queue Cosmos maintains separately —
it's a persistent, ordered log view derived directly from each physical
partition's internal write log: every insert/update (deletes are not
captured by default) is appended in the order it was committed to that
partition, and a change feed processor client keeps a **lease** (itself
stored as documents in a separate "lease container") recording which
partition and which point in that log it has consumed up to — this is why
scaling out change-feed consumers works by having multiple processor
instances each claim a different partition's lease, and why a Cosmos DB
container's partition count is the real ceiling on change-feed read
parallelism.

Cosmos's **multi-region writes** (multi-master) work by having every region
you enable act as an independent write replica running the same
consensus/replication protocol described in Module 6 within its own set of
physical partitions, with **conflict resolution policies** (last-writer-
wins by a system or custom timestamp property, or a merge procedure you
supply) executed by Cosmos itself when two regions concurrently write
conflicting versions of the same item — this happens asynchronously as
part of cross-region replication, after each region's own write has
already been locally acknowledged, which is the actual mechanism behind
multi-region writes' low write latency (you're never blocked waiting on a
remote region) at the cost of eventual, resolved consistency across
regions. Composite/range indexes are maintained automatically by Cosmos's
write path — every insert updates an inverted index structure per
indexed property in the same partition write, which is why Cosmos DB
indexes everything by default and why excluding paths from indexing is
purely a write-cost and storage optimization, not a correctness concern.

## Cheat sheet

| Command / concept | Purpose |
|---|---|
| `--partition-key-path` | Choose the property Cosmos DB partitions on (immutable after creation). |
| `az cosmosdb sql container throughput update --throughput` | Set manual/provisioned RU/s. |
| `--max-throughput` | Switch to autoscale between 10-100% of this max. |
| `EnableServerless` capability | Per-RU billing, no provisioning, mutually exclusive with provisioned. |
| `--default-consistency-level` | Account-wide default consistency (Strong/BoundedStaleness/Session/ConsistentPrefix/Eventual). |
| `query_items_change_feed()` | Read the ordered log of inserts/updates on a container. |
| `429` response | RU budget exceeded for that partition/account; back off and retry. |

## Exercise

1. Create a Cosmos DB account and container with a partition key that
   matches a realistic query pattern (e.g. `/customerId` for "get this
   customer's orders").
2. Insert enough items to see RU charges vary — check the `x-ms-request-charge`
   response header (or the SDK's `request_charge` property) on a point
   read vs. a cross-partition query.
3. Switch the container from manual to autoscale throughput with
   `--max-throughput`, and read back its current tier with `throughput show`.
4. Write a small script using `query_items_change_feed` that prints every
   item as it's inserted, and confirm it picks up new writes made from a
   second terminal.
5. Delete the resource group when finished.
