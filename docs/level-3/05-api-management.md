# 05 · API Management

Once you have more than a couple of backend APIs, exposing each one
directly to consumers means every client hardcodes URLs, every backend
handles its own auth/rate-limiting, and versioning becomes chaos. **Azure
API Management (APIM)** sits in front as a single gateway: one façade,
consistent policies (auth, throttling, transformation), and a developer
portal for API consumers.

## Creating an APIM instance

```bash
az group create --name rg-apim --location eastus

az apim create \
  --resource-group rg-apim \
  --name apim-platform-demo \
  --publisher-name "Platform Team" \
  --publisher-email platform@example.com \
  --sku-name Developer \
  --no-wait
```

**Gotcha:** APIM provisioning takes **30-45 minutes** even for the
`Developer` SKU — always `--no-wait` and poll, and never design a CI/CD
pipeline that blocks on APIM creation inline; provision it once per
environment and treat it as long-lived infrastructure, not something
recreated per deployment.

## Importing a backend API

```bash
az apim api import \
  --resource-group rg-apim \
  --service-name apim-platform-demo \
  --api-id orders-api \
  --path orders \
  --specification-format OpenApi \
  --specification-url https://api-orders.azurewebsites.net/swagger/v1/swagger.json \
  --display-name "Orders API"
```

This creates the routing (`https://apim-platform-demo.azure-api.net/orders/*`
→ your backend) and imports every operation from the OpenAPI spec as a
manageable APIM operation.

## Policies: rate limiting, auth, transformation

Policies are XML applied at product, API, or operation scope. A rate limit
plus a subscription-key requirement on a product:

```xml
<policies>
  <inbound>
    <base />
    <rate-limit-by-key calls="100" renewal-period="60"
      counter-key="@(context.Subscription.Id)" />
    <validate-jwt header-name="Authorization" failed-validation-httpcode="401">
      <openid-config url="https://login.microsoftonline.com/{tenant-id}/v2.0/.well-known/openid-configuration" />
      <audiences>
        <audience>api://orders-api</audience>
      </audiences>
    </validate-jwt>
  </inbound>
  <backend>
    <base />
  </backend>
  <outbound>
    <base />
    <set-header name="X-Powered-By" exists-action="delete" />
  </outbound>
</policies>
```

```bash
az apim api operation update \
  --resource-group rg-apim \
  --service-name apim-platform-demo \
  --api-id orders-api \
  --operation-id get-orders
# policy XML applied via the portal, ARM/Bicep, or
# `az rest` against the policy sub-resource — the CLI has no
# dedicated "set policy" verb, so most teams manage policy XML as code
# and deploy it through Bicep/ARM or the Azure DevOps APIM extension.
```

**Gotcha:** `<base />` inside a policy section re-runs the parent scope's
policy (global → product → API → operation) at that point — omitting it
silently **skips** any policy defined at a broader scope (like a
global rate limit or CORS policy), which is a frequent cause of "why isn't
my global policy applying to this one operation" confusion.

## Products and subscription keys

**Products** bundle one or more APIs behind a single subscription key and
usage quota — most orgs expose a "Free" product (low rate limit, no
approval) and a "Partner" product (higher limits, requires approval):

```bash
az apim product create \
  --resource-group rg-apim \
  --service-name apim-platform-demo \
  --product-id partner-tier \
  --product-name "Partner Tier" \
  --subscription-required true \
  --approval-required true \
  --state published

az apim product api add \
  --resource-group rg-apim \
  --service-name apim-platform-demo \
  --product-id partner-tier \
  --api-id orders-api
```

**Gotcha:** a newly created product defaults to `state=notPublished` and
`subscriptionsLimit=1` per user — teams that forget to `--state published`
find their API "works in the test console" (which uses admin credentials)
but returns 404 for every external subscriber, since unpublished products
aren't visible in the developer portal or accessible via a real
subscription key.

## Versions and revisions

- A **revision** is a non-breaking change to an existing API version
  (fix a bug, update a policy) — test it at `;rev=2` before making it
  current.
- A **version** is a breaking change exposed as a separate path/header
  (`/v1/orders` vs `/v2/orders`) that old consumers can keep using.

```bash
az apim api revision create \
  --resource-group rg-apim \
  --service-name apim-platform-demo \
  --api-id orders-api \
  --api-revision 2
```

## Cheat sheet

| Command | Purpose |
|---|---|
| `az apim create --sku-name Developer --no-wait` | Provision an APIM instance (30-45 min). |
| `az apim api import --specification-format OpenApi` | Import a backend API from its OpenAPI spec. |
| `az apim product create --subscription-required true` | Create a product bundling APIs with a subscription key. |
| `az apim product api add` | Attach an API to a product. |
| `az apim api revision create` | Create a testable, non-breaking revision. |
| `<rate-limit-by-key>` policy | Throttle calls per subscription/IP/custom key. |
| `<validate-jwt>` policy | Require and validate a bearer token before the backend. |

## Exercise

1. Create a Developer-tier APIM instance and import a public OpenAPI spec
   (or a mock backend) as an API.
2. Create a product with `--approval-required true`, add the API to it, and
   publish it.
3. Apply a `<rate-limit-by-key>` policy of 10 calls/60s at the product
   scope and confirm the 11th call in a minute gets throttled.
4. Create revision 2 of the API, test it via `;rev=2`, then promote it to
   current without breaking existing consumers on the unversioned path.
5. Delete the resource group when finished.
