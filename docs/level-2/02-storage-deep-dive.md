# 02 · Storage Deep Dive (Files, Queues, Tables)

[Level 1, Module 04](../level-1/04-blob-storage.md) covered Blob Storage —
one of four data services that live inside every storage account. This
module covers the other three: **Azure Files** (network file shares),
**Queue Storage** (simple messaging), and **Table Storage** (NoSQL
key-value data), plus **lifecycle management** policies that apply across
all of them.

## Azure Files

An Azure Files **share** is an SMB (and optionally NFS) network file share
— mountable from Windows, Linux, or macOS like any other network drive,
and usable as a drop-in replacement for an on-prem file server.

```bash
az group create --name rg-storage-deep --location eastus

az storage account create \
  --name ststoragedeep$RANDOM \
  --resource-group rg-storage-deep \
  --sku Standard_LRS \
  --kind StorageV2

STORAGE_ACCOUNT=$(az storage account list --resource-group rg-storage-deep --query "[0].name" --output tsv)

az storage share create \
  --account-name $STORAGE_ACCOUNT \
  --name shared-config \
  --quota 10 \
  --auth-mode login
```

`--quota 10` caps the share at 10 GiB — shares are provisioned capacity,
billed whether or not you fill it (unlike Blob's pay-for-what-you-use
model). Mount it from Linux:

```bash
STORAGE_KEY=$(az storage account keys list \
  --account-name $STORAGE_ACCOUNT --resource-group rg-storage-deep \
  --query "[0].value" --output tsv)

sudo mkdir -p /mnt/shared-config
sudo mount -t cifs "//$STORAGE_ACCOUNT.file.core.windows.net/shared-config" /mnt/shared-config \
  -o username=$STORAGE_ACCOUNT,password=$STORAGE_KEY,serverino,nosharesock,actimeo=30
```

**Gotcha:** SMB mounts over the public endpoint require port 445 outbound,
which many corporate and ISP networks block — Azure's own VMs and VNets
work fine, but mounting from a random coffee-shop Wi-Fi often doesn't.
Private endpoints ([Module 03](03-networking-deep-dive.md)) sidestep this
entirely by keeping traffic off the public internet.

## Queue Storage

A **queue** holds messages (up to 64 KB each) for asynchronous processing
— a producer enqueues work, one or more consumers dequeue and process it
at their own pace, decoupling the two sides.

```bash
az storage queue create \
  --account-name $STORAGE_ACCOUNT \
  --name orders \
  --auth-mode login

az storage message put \
  --account-name $STORAGE_ACCOUNT \
  --queue-name orders \
  --content '{"orderId": 1042, "action": "ship"}' \
  --auth-mode login

# A consumer peeks/dequeues, processes, then explicitly deletes
az storage message get \
  --account-name $STORAGE_ACCOUNT \
  --queue-name orders \
  --auth-mode login
# Returns the message plus a "pop receipt" needed to delete it

az storage message delete \
  --account-name $STORAGE_ACCOUNT \
  --queue-name orders \
  --id <message-id> \
  --pop-receipt <pop-receipt-from-get> \
  --auth-mode login
```

A dequeued message becomes **invisible** to other consumers for a
visibility timeout (default 30s), not deleted — if the consumer crashes
before calling delete, the message reappears and gets retried. This is
the core reliability guarantee: **at-least-once** delivery, so consumers
must be idempotent (safe to process the same message twice).

**Gotcha:** Storage Queues are intentionally simple — no topics, no
dead-lettering, no ordering guarantees across messages. For anything
needing those, reach for **Service Bus** instead (covered in
[Level 3, Module 04](../level-3/04-event-driven-architecture.md)); Storage
Queues are the right choice when "cheap, simple, at-least-once" is enough.

## Table Storage

A **table** stores schemaless entities addressed by a **partition key**
and **row key** pair — extremely cheap, extremely fast for key-based
lookups, but not a relational database (no joins, limited query
flexibility).

```bash
az storage table create \
  --account-name $STORAGE_ACCOUNT \
  --name Devices \
  --auth-mode login

az storage entity insert \
  --account-name $STORAGE_ACCOUNT \
  --table-name Devices \
  --entity PartitionKey=building-a RowKey=sensor-001 Temperature=21.4 \
  --auth-mode login

az storage entity query \
  --account-name $STORAGE_ACCOUNT \
  --table-name Devices \
  --filter "PartitionKey eq 'building-a'" \
  --auth-mode login
```

**Partition key design is the whole game:** all entities sharing a
partition key are stored (and scaled) together, so queries filtered by
partition key are fast, while queries that scan across partitions are
slow and expensive at scale. A common pattern is `PartitionKey` = a
natural grouping (building, tenant, date), `RowKey` = the unique item
within that group — exactly what the example above does.

## Lifecycle management policies

A **lifecycle management policy** automatically moves or deletes blobs
based on age rules, applied at the storage account level — the automation
layer on top of the access tiers from
[Level 1, Module 04](../level-1/04-blob-storage.md).

```json
{
  "rules": [
    {
      "name": "archive-old-logs",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["logs/"]
        },
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 30 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 90 },
            "delete": { "daysAfterModificationGreaterThan": 365 }
          }
        }
      }
    }
  ]
}
```

```bash
az storage account management-policy create \
  --account-name $STORAGE_ACCOUNT \
  --resource-group rg-storage-deep \
  --policy @lifecycle-policy.json
```

This rule cools logs after 30 days, archives them after 90, and deletes
them after a year — with zero code and zero cron jobs. **Gotcha:** the
policy scans and applies changes once per day, not instantly, and
retrieving an **Archive**-tier blob (via `az storage blob set-tier --tier
Hot`, i.e. rehydration) can take hours, so don't apply aggressive archive
rules to anything you might need back on short notice.

## Cheat sheet

| Command | Purpose |
|---|---|
| `az storage share create --quota` | Create an Azure Files share with a provisioned size. |
| `mount -t cifs //<account>.file.core.windows.net/<share>` | Mount a share over SMB. |
| `az storage queue create` | Create a Queue Storage queue. |
| `az storage message put/get/delete` | Enqueue, dequeue (peek+lock), and delete a message. |
| `az storage table create` | Create a Table Storage table. |
| `az storage entity insert/query` | Insert or query entities by partition/row key. |
| `az storage account management-policy create --policy` | Apply a lifecycle rule (tier/delete by age). |

## Exercise

1. Create an Azure Files share, mount it locally (or on a test VM), and
   write a file to it through the mount.
2. Create a queue, enqueue two messages, dequeue one, and delete it using
   its pop receipt — confirm the second message is still there.
3. Create a table, insert three entities sharing one `PartitionKey` with
   different `RowKey`s, then query them back filtered by that partition
   key.
4. Write a lifecycle policy that moves blobs under a `temp/` prefix to
   Cool after 7 days and deletes them after 30, and apply it to your
   storage account.
5. Delete the resource group when finished.
