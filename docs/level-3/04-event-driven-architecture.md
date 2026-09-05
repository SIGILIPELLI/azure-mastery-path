# 04 · Event-Driven Architecture (Event Grid, Service Bus)

Synchronous request/response (App Service calling App Service, or an API
calling a database directly) couples services tightly — if the downstream
is slow or down, the caller blocks or fails. This module covers Azure's two
main asynchronous messaging services: **Event Grid** (discrete events,
push, fan-out) and **Service Bus** (durable queues/topics, pull, ordered
delivery), and when to reach for each.

## Event Grid: reactive, fan-out events

Event Grid delivers discrete **events** ("a blob was created", "a resource
was deleted") to multiple subscribers with at-least-once delivery and
built-in retry/dead-lettering. It's push-based — Event Grid calls your
endpoint, you don't poll.

```bash
az group create --name rg-events --location eastus

az eventgrid topic create \
  --resource-group rg-events \
  --name egt-orders \
  --location eastus

az eventgrid event-subscription create \
  --source-resource-id $(az eventgrid topic show -g rg-events -n egt-orders --query id -o tsv) \
  --name sub-order-processor \
  --endpoint https://func-order-processor.azurewebsites.net/runtime/webhooks/eventgrid?functionName=ProcessOrder \
  --endpoint-type webhook
```

Publishing an event to the topic:

```bash
ENDPOINT=$(az eventgrid topic show -g rg-events -n egt-orders --query endpoint -o tsv)
KEY=$(az eventgrid topic key list -g rg-events -n egt-orders --query key1 -o tsv)

curl -X POST $ENDPOINT \
  -H "aeg-sas-key: $KEY" \
  -H "Content-Type: application/json" \
  -d '[{
    "id": "1",
    "eventType": "Orders.Created",
    "subject": "orders/12345",
    "eventTime": "2026-08-25T10:00:00Z",
    "data": { "orderId": 12345, "amount": 79.99 },
    "dataVersion": "1.0"
  }]'
```

**Gotcha:** an Event Grid webhook subscriber must respond to the initial
**validation handshake** (echoing a `validationCode`) within a short window
or the subscription is created in an unvalidated state and never receives
events — Azure Functions' Event Grid trigger handles this automatically,
but a hand-rolled HTTP endpoint must implement the handshake explicitly or
subscription creation silently produces a dead subscription.

## Service Bus: durable queues and topics

Service Bus is **pull-based** (consumers fetch messages) with guaranteed
FIFO ordering (with sessions), transactions, and dead-lettering — suited to
workflows where messages represent work that must not be lost or
duplicated-processed, unlike Event Grid's fire-and-forget events.

```bash
az servicebus namespace create \
  --resource-group rg-events \
  --name sb-orders-ns \
  --sku Standard

az servicebus queue create \
  --resource-group rg-events \
  --namespace-name sb-orders-ns \
  --name q-order-fulfillment \
  --max-delivery-count 5 \
  --lock-duration PT1M
```

```text
$ az servicebus queue show -g rg-events --namespace-name sb-orders-ns -n q-order-fulfillment --query "{count:messageCount,size:sizeInBytes}"
{
  "count": 0,
  "size": 0
}
```

**Topics** add pub/sub with server-side filtering — multiple subscriptions
each get a copy, and a **SQL filter** decides which messages a subscription
actually receives:

```bash
az servicebus topic create \
  --resource-group rg-events \
  --namespace-name sb-orders-ns \
  --name topic-orders

az servicebus topic subscription create \
  --resource-group rg-events \
  --namespace-name sb-orders-ns \
  --topic-name topic-orders \
  --name sub-high-value

az servicebus topic subscription rule create \
  --resource-group rg-events \
  --namespace-name sb-orders-ns \
  --topic-name topic-orders \
  --subscription-name sub-high-value \
  --name rule-high-value \
  --filter-sql-expression "amount > 500"
```

**Gotcha:** a message that fails processing repeatedly (exceeding
`--max-delivery-count`) goes to the **dead-letter sub-queue**, not deleted —
if nothing monitors `$DeadLetterQueue`, poisoned messages accumulate
silently and the real failure (a bug processing a specific message shape)
goes unnoticed for weeks. Always alert on dead-letter queue depth, not just
the main queue.

## Event Grid vs. Service Bus vs. Storage Queues

| | Event Grid | Service Bus | Storage Queues |
|---|---|---|---|
| **Model** | Push, pub/sub | Pull, queue/topic | Pull, simple queue |
| **Ordering** | Not guaranteed | FIFO with sessions | Not guaranteed |
| **Delivery** | At-least-once | At-least-once (or exactly-once with sessions) | At-least-once |
| **Max message size** | 1 MB | 256 KB (Standard) / 100 MB (Premium) | 64 KB |
| **Use case** | React to Azure resource/system events, fan-out | Transactional workflows, ordered processing | Simple, cheap decoupling |

## How It Actually Works

**Event Grid** is a publish-subscribe *routing* system, not a message
store: when an event source (a resource provider, or your own custom
topic) publishes an event, Event Grid's intake immediately fans it out to
every matching subscription's endpoint via an HTTP POST (or an Azure-native
handler), retrying with exponential backoff on failure for up to 24 hours
by default before dead-lettering — critically, Event Grid does not hold
events for a subscriber to pull later, so a subscriber that's down when an
event fires only gets it via that retry window, not on reconnect. **Service
Bus**, by contrast, is a genuine broker with durable storage: messages sent
to a queue or topic are persisted (backed internally by a partitioned,
replicated log store) until a consumer explicitly receives and completes
them, which is what enables Service Bus's session support, FIFO ordering
within a session, and true competing-consumer semantics that Event Grid's
fire-and-forget push model cannot offer.

**Storage Queues** sit architecturally between the two: messages are
stored as blob-like entities in the storage account's queue service (same
partitioned storage stamp as Blob/Table storage), and consumers **poll**
for them — a dequeued message becomes invisible for a configurable
visibility timeout rather than being deleted, and if the consumer crashes
before calling delete, the message automatically reappears after that
timeout for another consumer to pick up, which is the actual mechanism
behind Storage Queue's at-least-once delivery guarantee and why duplicate
processing is possible and must be handled idempotently by the consumer.
Choosing between the three in practice comes down to which of these
underlying mechanisms — push fan-out with no storage (Event Grid), a
persistent ordered broker (Service Bus), or simple poll-based durable
storage (Storage Queues) — actually matches the delivery and ordering
guarantee your architecture needs.

## Cheat sheet

| Command | Purpose |
|---|---|
| `az eventgrid topic create` | Create a custom Event Grid topic. |
| `az eventgrid event-subscription create --endpoint-type webhook` | Subscribe an endpoint to a topic. |
| `az servicebus namespace create` | Create a Service Bus namespace. |
| `az servicebus queue create --max-delivery-count` | Create a queue with a dead-letter threshold. |
| `az servicebus topic subscription rule create --filter-sql-expression` | Filter which messages a subscription receives. |
| `az servicebus queue show --query messageCount` | Check current queue depth. |

## Exercise

1. Create an Event Grid topic and subscribe a webhook endpoint; publish a
   test event via `curl` with the topic's SAS key.
2. Create a Service Bus queue with `--max-delivery-count 5`, send a message
   that intentionally fails processing 5 times, and confirm it lands in
   `$DeadLetterQueue`.
3. Create a Service Bus topic with two subscriptions, each with a different
   SQL filter, and confirm messages route correctly based on a property.
4. Write down, for your own future project, one scenario that should use
   Event Grid and one that should use Service Bus, and justify each choice.
5. Delete the resource group when finished.
