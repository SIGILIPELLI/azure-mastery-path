# 08 · Data Platform at Scale (Synapse, Data Factory)

[Level 2, Module 08](../level-2/08-cosmos-db-deep-dive.md) covered a
transactional NoSQL database for application workloads. This module covers
the **analytical** side: moving data from many sources into a warehouse and
querying it at scale with **Azure Synapse Analytics**, orchestrated by
**Azure Data Factory** pipelines.

## Data Factory: orchestrating pipelines

A **pipeline** groups activities (copy data, run a stored procedure,
trigger a Databricks notebook) into one schedulable, monitorable unit:

```bash
az datafactory create \
  --resource-group rg-data-platform \
  --factory-name adf-orders-platform

az datafactory linked-service create \
  --resource-group rg-data-platform \
  --factory-name adf-orders-platform \
  --linked-service-name ls-sql-orders \
  --properties '{
    "type": "AzureSqlDatabase",
    "typeProperties": { "connectionString": "Server=sql-orders-primary.database.windows.net;Database=db-orders;" }
  }'
```

```bash
az datafactory pipeline create \
  --resource-group rg-data-platform \
  --factory-name adf-orders-platform \
  --pipeline-name pl-copy-orders-to-synapse \
  --pipeline '{
    "activities": [{
      "name": "CopyOrders",
      "type": "Copy",
      "inputs": [{ "referenceName": "ds-orders-sql", "type": "DatasetReference" }],
      "outputs": [{ "referenceName": "ds-orders-synapse", "type": "DatasetReference" }]
    }]
  }'
```

**Gotcha:** Data Factory's **Copy Activity** performance depends heavily on
**Data Integration Units (DIUs)** and parallel copy settings — the default
autodetect setting is fine for most loads, but a large one-time historical
backfill left on default settings can take an order of magnitude longer
than the same job with DIUs and parallelism explicitly tuned; check the
copy activity's monitoring output for "used DIUs" before assuming
performance can't be improved.

## Synapse: dedicated SQL pools vs. serverless

Synapse offers two very different query engines over the same workspace:

```bash
az synapse workspace create \
  --resource-group rg-data-platform \
  --name synw-orders-platform \
  --storage-account stadls2orders \
  --file-system synapsefs \
  --sql-admin-login-user sqladmin \
  --sql-admin-login-password "ReplaceWithARealSecret1!"

az synapse sql pool create \
  --resource-group rg-data-platform \
  --workspace-name synw-orders-platform \
  --name sqlpool-orders \
  --performance-level DW200c
```

A **dedicated SQL pool** (`DW*c` performance levels) is a provisioned,
always-billing MPP warehouse — predictable performance, billed whether
queried or not. A **serverless SQL pool** (no provisioning step, free to
create) queries files directly in the data lake and bills per **data
scanned**, not per hour:

```sql
SELECT TOP 100 *
FROM OPENROWSET(
    BULK 'https://stadls2orders.dfs.core.windows.net/orders/*.parquet',
    FORMAT = 'PARQUET'
) AS orders_data
```

**Gotcha:** serverless SQL pool billing is per **byte scanned**, not per
row returned — a poorly partitioned data lake forcing a full scan of
terabytes to answer a query touching only a small date range can cost far
more per query than expected; partition data by date (or another common
filter column) in the lake's folder structure and always filter on the
partition column so the engine can skip irrelevant files.

## Pausing dedicated pools to control cost

Unlike serverless, a dedicated SQL pool bills continuously while running —
pausing it when not in active use is the primary cost lever:

```bash
az synapse sql pool pause \
  --resource-group rg-data-platform \
  --workspace-name synw-orders-platform \
  --name sqlpool-orders

az synapse sql pool resume \
  --resource-group rg-data-platform \
  --workspace-name synw-orders-platform \
  --name sqlpool-orders
```

**Gotcha:** resuming a paused dedicated pool takes on the order of several
minutes — a pipeline that pauses the pool after each nightly load and tries
to resume it for an ad-hoc query the next morning without accounting for
resume latency produces a confusing "query is just hanging" experience for
the first user of the day; automate resume on a schedule ahead of expected
usage, not on first-query-arrival.

## Data Factory triggers and monitoring

```bash
az datafactory trigger create \
  --resource-group rg-data-platform \
  --factory-name adf-orders-platform \
  --trigger-name trg-nightly \
  --properties '{
    "type": "ScheduleTrigger",
    "typeProperties": { "recurrence": { "frequency": "Day", "interval": 1, "startTime": "2026-08-26T02:00:00Z" } },
    "pipelines": [{ "pipelineReference": { "referenceName": "pl-copy-orders-to-synapse", "type": "PipelineReference" } }]
  }'

az datafactory trigger start \
  --resource-group rg-data-platform \
  --factory-name adf-orders-platform \
  --trigger-name trg-nightly
```

```text
$ az datafactory pipeline-run query-by-factory -g rg-data-platform --factory-name adf-orders-platform --last-updated-after 2026-08-25T00:00:00Z --last-updated-before 2026-08-26T00:00:00Z --query "value[].{status:status, run:pipelineName}"
[{"status": "Succeeded", "run": "pl-copy-orders-to-synapse"}]
```

**Gotcha:** creating a trigger with `az datafactory trigger create` does
**not** start it — it's created in a stopped state, a frequent source of
"the pipeline never ran" confusion; you must explicitly `trigger start` (as
above) before the schedule takes effect.

## Warehouse engine comparison

| | Dedicated SQL Pool | Serverless SQL Pool | Spark Pool |
|---|---|---|---|
| **Billing** | Per hour, provisioned | Per byte scanned | Per node-hour while running |
| **Startup** | Always-on (or paused/resumed, minutes) | Instant, no provisioning | Cluster startup, ~minutes |
| **Best for** | Predictable, heavy BI workloads | Ad-hoc queries over lake files | Large-scale ETL, ML |
| **Cost control** | Pause when idle | Partition data, filter well | Autoscale/auto-terminate |

## Cheat sheet

| Command | Purpose |
|---|---|
| `az datafactory pipeline create` | Define an orchestration pipeline. |
| `az datafactory trigger create` + `trigger start` | Schedule and actually enable a pipeline run. |
| `az synapse sql pool create --performance-level` | Provision a dedicated SQL pool. |
| `az synapse sql pool pause` / `resume` | Control dedicated pool cost by stopping/starting compute. |
| `OPENROWSET(BULK ... FORMAT='PARQUET')` | Query lake files directly via serverless SQL. |
| `az datafactory pipeline-run query-by-factory` | Check pipeline run history and status. |

## Exercise

1. Create a Data Factory pipeline copying data from one linked service to
   another, and check the Copy Activity's monitoring output for DIU usage.
2. Create a Synapse workspace with a serverless SQL pool query against
   partitioned Parquet files in a data lake, filtering on the partition
   column.
3. Provision a dedicated SQL pool, pause it, then resume it and time how
   long resume actually takes.
4. Create a daily schedule trigger for a pipeline and confirm it does
   nothing until you explicitly `trigger start` it.
5. Delete the resource group when finished.
