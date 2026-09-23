---
name: provision-cdn-profile-and-endpoint
description: Stand up an Azure CDN profile and a delivery endpoint from scratch, validating the name before creating anything and polling every long-running step to completion.
api: microsoft-azure-cdn
generated: '2026-09-17'
method: generated
source: openapi/microsoft-azure-cdn-management-openapi.json (api-version 2026-07-01)
operations:
  - CheckNameAvailabilityWithSubscription
  - Profiles_Create
  - Profiles_Get
  - Endpoints_Create
  - Endpoints_Get
  - Origins_ListByEndpoint
---

# Provision a CDN profile and endpoint

Before you start: Azure CDN Standard from Microsoft (classic) is retiring on 2027-09-30 and has not
accepted new instances since 2025. If you are provisioning new delivery, provision Azure Front Door
instead. Continue only if you are operating an existing estate.

Base URL `https://management.azure.com`. Every request needs `api-version=2026-07-01` and
`Authorization: Bearer <Entra ID token>` with audience `https://management.azure.com/`.
You need the **CDN Profile Contributor** role at the resource-group scope.

## 1. Check the name is free

`CheckNameAvailabilityWithSubscription` —
`POST /subscriptions/{subscriptionId}/providers/Microsoft.Cdn/checkNameAvailability`

CDN endpoint hostnames are globally unique. Do this first; a name collision surfaced by
`Endpoints_Create` costs you a failed long-running operation instead of one cheap call.

## 2. Create the profile

`Profiles_Create` —
`PUT /subscriptions/{subscriptionId}/resourceGroups/{rg}/providers/Microsoft.Cdn/profiles/{profileName}`

This is a **PUT with a client-chosen name**, so it is replay-safe: repeating it with the same body
converges rather than creating a second profile. The SKU you send decides the whole resource family —
a `Standard_Microsoft` profile gets the classic Endpoint/Origin/CustomDomain surface, a
`Standard_AzureFrontDoor` or `Premium_AzureFrontDoor` profile gets AFDEndpoints, Routes and RuleSets.
You cannot mix them under one profile.

**Long-running.** Expect `202` with an `Azure-AsyncOperation` header. Poll it to `Succeeded` before
step 3 — the profile does not exist for the next call until then.

## 3. Create the endpoint

`Endpoints_Create` — `PUT .../profiles/{profileName}/endpoints/{endpointName}`

Also PUT, also long-running, also poll. The origin is part of the create body; you do not create an
origin separately for the first one.

## 4. Confirm

`Profiles_Get` and `Endpoints_Get` for the resource state, `Origins_ListByEndpoint` to confirm the
origin landed. Propagation to the edge is not instantaneous after the ARM operation reports success.

## Failure handling

- `409 Conflict` — a long-running operation is still in flight on the same resource. Poll, do not retry.
- `409 MissingSubscriptionRegistration` — run `az provider register --namespace Microsoft.Cdn` once.
- `403 AuthorizationFailed` — the principal lacks `Microsoft.Cdn/*`; assign CDN Profile Contributor.
- `429` — read `Retry-After`. Writes refill at 10/second from a bucket of 200 per subscription per region.
- Errors arrive as `{"error": {"code","message","target","details","additionalInfo"}}`, not RFC 9457.
