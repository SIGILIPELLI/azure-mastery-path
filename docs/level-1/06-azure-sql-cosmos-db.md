# 06 · Azure SQL Database / Cosmos DB Basics

Azure offers fully managed databases so you don't patch, back up, or
provision hardware yourself. This module covers the two most common
starting points: **Azure SQL Database** (managed relational SQL Server) and
**Azure Cosmos DB** (managed multi-model NoSQL) — and how to connect an app
to each.

## Azure SQL Database

Azure SQL Database is a managed, single-database or elastic-pool offering
built on the SQL Server engine — no OS or SQL Server instance to patch, with
built-in high availability and automated backups.

### Create a logical server and database

A **logical server** is just an administrative/connection endpoint (`.database.windows.net`);
it's not a VM you manage, but every database needs one as its parent.

```bash
az group create --name rg-db-demo --location eastus

az sql server create \
  --resource-group rg-db-demo \
  --name sql-learn-demo$RANDOM \
  --location eastus \
  --admin-user sqladmin \
  --admin-password "ChangeThisP@ssw0rd123!"

SQL_SERVER=$(az sql server list --resource-group rg-db-demo --query "[0].name" --output tsv)

# Create a database on the free-tier-eligible General Purpose serverless tier
az sql db create \
  --resource-group rg-db-demo \
  --server $SQL_SERVER \
  --name db-learn \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 1 \
  --compute-model Serverless \
  --use-free-limit \
  --free-limit-exhaustion-behavior AutoPause
```

`--use-free-limit` applies Azure SQL's free-tier allowance (one free
database per subscription, up to 100,000 vCore-seconds and 32 GB/month) —
`AutoPause` means it simply pauses rather than starts charging once the
monthly free allowance runs out.

### Firewall rules

Azure SQL denies all connections by default — you allow specific IPs (or
Azure services) explicitly:

```bash
# Allow your current machine's IP
az sql server firewall-rule create \
  --resource-group rg-db-demo \
  --server $SQL_SERVER \
  --name allow-my-ip \
  --start-ip-address $(curl -s ifconfig.me) \
  --end-ip-address $(curl -s ifconfig.me)

# Allow other Azure services (e.g. an App Service or Function) to connect
az sql server firewall-rule create \
  --resource-group rg-db-demo \
  --server $SQL_SERVER \
  --name allow-azure-services \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0
```

The `0.0.0.0`-`0.0.0.0` range is a special case Azure recognizes as "allow
traffic from Azure resources," not a literal wildcard.

### Connecting from an app

```bash
# Get the ADO.NET-style connection string as a template
az sql db show-connection-string \
  --server $SQL_SERVER \
  --name db-learn \
  --client ado.net
```

A Python app would connect with `pyodbc` using a string shaped like:

```python
import pyodbc

conn = pyodbc.connect(
    "Driver={ODBC Driver 18 for SQL Server};"
    f"Server=tcp:{SQL_SERVER}.database.windows.net,1433;"
    "Database=db-learn;"
    "Uid=sqladmin;"
    "Pwd=ChangeThisP@ssw0rd123!;"
    "Encrypt=yes;TrustServerCertificate=no;Connection Timeout=30;"
)
cursor = conn.cursor()
cursor.execute("SELECT @@VERSION;")
print(cursor.fetchone())
```

In production, replace the username/password with an Entra ID managed
identity connection — covered in [Level 2, Module 06](../level-2/06-managed-identities-key-vault.md) —
so no password lives in your app's config at all.

## Cosmos DB

Cosmos DB is Azure's globally distributed, multi-model database — it can
speak the **Core (SQL) API**, MongoDB API, Cassandra API, Table API, and
Gremlin (graph) API, all backed by the same underlying engine. It's the
natural choice when you need single-digit-millisecond latency at any scale,
flexible schema, or multi-region writes.

### Create an account and database

```bash
az cosmosdb create \
  --resource-group rg-db-demo \
  --name cosmos-learn-demo$RANDOM \
  --locations regionName=eastus failoverPriority=0 \
  --default-consistency-level Session \
  --enable-free-tier true
```

`--enable-free-tier true` applies Cosmos DB's Always-Free allowance (1000
RU/s of throughput and 25 GB storage) — only one free-tier account is
allowed per subscription.

```bash
COSMOS_ACCOUNT=$(az cosmosdb list --resource-group rg-db-demo --query "[0].name" --output tsv)

az cosmosdb sql database create \
  --account-name $COSMOS_ACCOUNT \
  --resource-group rg-db-demo \
  --name db-learn

az cosmosdb sql container create \
  --account-name $COSMOS_ACCOUNT \
  --resource-group rg-db-demo \
  --database-name db-learn \
  --name items \
  --partition-key-path "/category" \
  --throughput 400
```

The **partition key** (`/category` here) is the field Cosmos DB uses to
distribute data and requests across physical partitions — choosing one with
high cardinality and even access patterns matters a lot at scale, covered
in depth in [Level 2, Module 08](../level-2/08-cosmos-db-deep-dive.md).

### Connecting from an app

```bash
az cosmosdb keys list \
  --name $COSMOS_ACCOUNT \
  --resource-group rg-db-demo \
  --type connection-strings
```

```python
from azure.cosmos import CosmosClient

client = CosmosClient(
    url="https://cosmos-learn-demo1234.documents.azure.com:443/",
    credential="<primary-key-from-above>",
)
database = client.get_database_client("db-learn")
container = database.get_container_client("items")

container.create_item({"id": "1", "category": "fruit", "name": "apple"})
for item in container.read_all_items():
    print(item)
```

## Azure SQL vs. Cosmos DB — which one?

| | Azure SQL Database | Cosmos DB |
|---|---|---|
| **Data model** | Relational (tables, joins, transactions) | Document/multi-model, flexible schema |
| **Query language** | T-SQL | SQL-like (Core API), or Mongo/Cassandra/Gremlin dialects |
| **Scaling model** | Vertical (DTUs/vCores), read replicas | Horizontal via partitioning, multi-region |
| **Best for** | Existing relational data, strong consistency, complex joins/transactions | High-scale, low-latency, globally distributed apps, semi-structured data |
| **Free tier** | 1 free database (100K vCore-seconds/mo) | 1000 RU/s + 25 GB, always free |

## Cheat sheet

| Command | Purpose |
|---|---|
| `az sql server create --admin-user --admin-password` | Create a logical SQL server. |
| `az sql db create --edition --use-free-limit` | Create an Azure SQL database. |
| `az sql server firewall-rule create --start-ip --end-ip` | Allow an IP range to connect. |
| `az sql db show-connection-string --client` | Print a connection string template. |
| `az cosmosdb create --enable-free-tier true` | Create a Cosmos DB account on the free tier. |
| `az cosmosdb sql database create` / `sql container create` | Create a Cosmos database and container. |
| `az cosmosdb keys list --type connection-strings` | Get connection strings/keys. |

## Exercise

1. Create an Azure SQL server + free-tier database, add a firewall rule for
   your own IP, and connect with `sqlcmd` or a GUI tool (Azure Data Studio)
   to run `CREATE TABLE notes (id INT PRIMARY KEY, text NVARCHAR(200));`.
2. Separately, create a free-tier Cosmos DB account with a `notes` database
   and `items` container, and insert one JSON document via the CLI's
   `az cosmosdb sql container` commands or the Python SDK snippet above.
3. Write one sentence for yourself on when you'd reach for each — check it
   against the comparison table.
4. Delete the resource group when done (note: Cosmos DB accounts can take a
   few minutes to fully delete).
