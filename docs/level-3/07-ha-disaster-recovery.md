# 07 · High Availability & Disaster Recovery

**Availability** keeps a single-region deployment resilient to individual
failures (a VM crashes, a datacenter loses power). **Disaster recovery**
handles losing an entire region. They use different Azure mechanisms and
different target metrics — this module covers both, plus the RTO/RPO
vocabulary you need to talk about them precisely.

## Availability Zones vs. Availability Sets

An **Availability Set** spreads VMs across fault domains (racks with
separate power/network) and update domains within a **single datacenter**.
**Availability Zones** spread resources across physically separate
datacenters within a region, each with independent power, cooling, and
networking — a strictly stronger guarantee, and the default choice when
the region supports zones.

```bash
az vm create \
  --resource-group rg-ha \
  --name vm-web-1 \
  --image Ubuntu2204 \
  --zone 1 \
  --vnet-name vnet-ha \
  --subnet subnet-web

az vm create \
  --resource-group rg-ha \
  --name vm-web-2 \
  --image Ubuntu2204 \
  --zone 2 \
  --vnet-name vnet-ha \
  --subnet subnet-web
```

**Gotcha:** not every Azure region supports Availability Zones, and not
every VM SKU is available in every zone within a region that does — check
`az vm list-skus --zone --location <region>` before designing a
zone-redundant architecture, since discovering a SKU isn't zonal in your
target region after the fact means re-architecting.

## Zone-redundant vs. zonal PaaS services

Most PaaS services offer a **zone-redundant** SKU tier that Azure spreads
across zones transparently, no explicit zone pinning needed:

```bash
az sql db create \
  --resource-group rg-ha \
  --server sql-ha-server \
  --name db-ha \
  --zone-redundant true \
  --edition Premium
```

## RTO and RPO

- **RTO (Recovery Time Objective):** how long you can be down before it's
  unacceptable — drives your failover *automation* investment.
- **RPO (Recovery Point Objective):** how much data you can afford to
  lose — drives your *replication frequency*.

A synchronous multi-zone SQL deployment gives near-zero RPO/RTO within a
region; cross-region async replication (Geo-Replication, GRS storage)
typically gives RPO in seconds-to-minutes and RTO in minutes, since
failover isn't automatic.

## Azure Site Recovery (cross-region DR)

```bash
az backup vault create \
  --resource-group rg-dr \
  --name rsv-dr-vault \
  --location westus2

az site-recovery vault create \
  --resource-group rg-dr \
  --name asr-vault \
  --location eastus
# Fabric, protection container, and replication policy setup for
# VM/VMware replication is done via the ASR extension / portal wizard
# for most of the finer-grained config; the CLI covers vault lifecycle.

az backup protection enable-for-vm \
  --resource-group rg-dr \
  --vault-name rsv-dr-vault \
  --vm vm-web-1 \
  --policy-name DefaultPolicy
```

Site Recovery continuously replicates VM disks to a secondary region; a
**recovery plan** groups VMs (e.g. database tier before app tier) and
orchestrates ordered failover with a single trigger, tested via **test
failover** into an isolated network without impacting production.

**Gotcha:** test failover uses an **isolated copy** of the network by
default and does not validate whether the production failover network
(NSGs, DNS, load balancer config in the DR region) is actually correctly
configured — a clean test failover is necessary but not sufficient proof a
real failover will work; periodically also review the actual DR-region
network configuration, not just the VM boot success.

## Geo-redundant storage and SQL failover groups

```bash
az storage account create \
  --resource-group rg-ha \
  --name sthageo \
  --sku Standard_RAGRS

az sql failover-group create \
  --resource-group rg-ha \
  --server sql-ha-server \
  --name fg-orders \
  --partner-server sql-ha-server-secondary \
  --add-db db-ha
```

```bash
az sql failover-group set-primary \
  --resource-group rg-ha \
  --server sql-ha-server-secondary \
  --name fg-orders
```

**Gotcha:** RA-GRS gives you a **read-only** secondary endpoint
(`<account>-secondary.blob.core.windows.net`) — your application code must
explicitly know to read from it during an outage; Azure does not
automatically redirect writes or reads on regional failure, and a full
storage account failover (`az storage account failover`) is a manual,
one-way, non-reversible operation you trigger, not something automatic.

## HA/DR pattern comparison

| | Availability Zones | Availability Set | Geo-replication / ASR |
|---|---|---|---|
| **Protects against** | Datacenter failure | Rack/host failure | Region failure |
| **Scope** | Single region, multi-DC | Single datacenter | Cross-region |
| **RTO** | Seconds (transparent) | Seconds (transparent) | Minutes (failover trigger) |
| **RPO** | Near-zero | Near-zero | Seconds to minutes (async) |
| **Failover** | Automatic | Automatic | Manual or scripted trigger |

## Cheat sheet

| Command | Purpose |
|---|---|
| `az vm create --zone` | Pin a VM to a specific Availability Zone. |
| `az vm list-skus --zone` | Check zone support for a SKU in a region. |
| `az sql db create --zone-redundant true` | Create a zone-redundant SQL database. |
| `az backup protection enable-for-vm` | Enable backup/replication for a VM. |
| `az sql failover-group create` | Set up SQL cross-region failover. |
| `az sql failover-group set-primary` | Trigger a manual failover. |
| `az storage account failover` | Manually fail over RA-GRS storage (irreversible). |

## Exercise

1. Create two VMs pinned to different Availability Zones in a region that
   supports them; verify zone support first with `az vm list-skus --zone`.
2. Create a zone-redundant SQL database and explain, in your own words, the
   RTO/RPO difference between it and a cross-region failover group.
3. Set up a SQL failover group between a primary and secondary server, add
   a database, and perform a manual `set-primary` failover.
4. Create a Recovery Services vault, enable VM backup, and describe what a
   test failover does and does not prove about real DR readiness.
5. Delete the resource groups when finished.
