---
name: purge-cached-content
description: Evict stale content from Azure CDN edge caches safely, knowing that purge is neither idempotent nor reversible.
api: microsoft-azure-cdn
generated: '2026-09-17'
method: generated
source: openapi/microsoft-azure-cdn-management-openapi.json (api-version 2026-07-01)
operations:
  - Endpoints_ListByProfile
  - Endpoints_PurgeContent
  - Endpoints_LoadContent
  - AFDEndpoints_PurgeContent
---

# Purge cached content

## Know this before you call

Purge is the most common Azure CDN automation and it is the operation with the sharpest edges:

- It is a **POST action**, so it carries **no replay protection**. There is no `Idempotency-Key`
  header anywhere in this API. Firing it twice fires the purge twice.
- It is **not reversible**. Purged content repopulates only on the next request that reaches the
  origin, so a broad purge is a deliberate origin-load event.
- It is **long-running**. A `202` means accepted, not evicted.

## 1. Find the endpoint

`Endpoints_ListByProfile` — `GET .../profiles/{profileName}/endpoints`. Paged: follow `nextLink`
until it is absent.

## 2. Purge

Classic CDN: `Endpoints_PurgeContent` — `POST .../endpoints/{endpointName}/purge`
Front Door Standard/Premium: `AFDEndpoints_PurgeContent` — `POST .../afdEndpoints/{endpointName}/purge`

Which one you call depends on the profile SKU, not on preference. Send the narrowest content path
list that fixes the problem. `/*` purges everything under the endpoint and sends every subsequent
request to the origin.

**Poll the `Azure-AsyncOperation` URL to a terminal state.** Do not report success on the 202.

## 3. Optional pre-warm

`Endpoints_LoadContent` — `POST .../endpoints/{endpointName}/load` pre-loads content back into the
cache. The contract states this is **available for Verizon profiles** only, so check the profile SKU
before assuming it is there.

## Guardrails for an agent

- Never purge `/*` without an explicit instruction naming the endpoint.
- Never retry a purge on a timeout without first polling the operation — you cannot tell a slow purge
  from a failed one by the status code, and a blind retry doubles origin load.
- Watch `x-ms-ratelimit-remaining-subscription-writes`; purge counts against the write bucket.
