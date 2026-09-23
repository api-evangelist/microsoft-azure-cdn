---
name: audit-profiles-and-usage
description: Inventory every Azure CDN and Front Door profile in a subscription read-only, including quota headroom and WAF policy coverage, without touching a single write operation.
api: microsoft-azure-cdn
generated: '2026-09-17'
method: generated
source: openapi/microsoft-azure-cdn-management-openapi.json (api-version 2026-07-01)
operations:
  - Profiles_List
  - Profiles_ListByResourceGroup
  - Profiles_ListResourceUsage
  - Endpoints_ListByProfile
  - AFDEndpoints_ListByProfile
  - CustomDomains_ListByEndpoint
  - AFDCustomDomains_ListByProfile
  - Policies_List
  - ResourceUsage_List
  - Operations_List
---

# Audit CDN profiles and usage

Every operation in this skill is a `GET` or a read-only list. Nothing here mutates anything, so it is
the right skill to run first against an unfamiliar subscription. The **CDN Profile Reader** role is
sufficient.

## 1. Inventory

`Profiles_List` — `GET /subscriptions/{subscriptionId}/providers/Microsoft.Cdn/profiles` for the whole
subscription, or `Profiles_ListByResourceGroup` to scope it.

The response mixes both families. Read each profile's SKU:

- `Standard_Microsoft`, `Standard_Verizon`, `Premium_Verizon`, `Standard_Akamai` — classic CDN, and
  the Microsoft one is on the 2027-09-30 retirement path.
- `Standard_AzureFrontDoor`, `Premium_AzureFrontDoor` — Front Door.

**Paged.** These operations declare `x-ms-pageable`; follow `nextLink` until absent. There is no page
size parameter.

## 2. Walk the children by family

- Classic: `Endpoints_ListByProfile`, then `CustomDomains_ListByEndpoint` per endpoint.
- Front Door: `AFDEndpoints_ListByProfile` and `AFDCustomDomains_ListByProfile`.

Calling the classic list against a Front Door profile returns nothing rather than an error, which
reads as "no endpoints" and is the most common way this audit silently under-reports.

## 3. Quota headroom

`Profiles_ListResourceUsage` and `ResourceUsage_List` report current-versus-limit counts per resource
type. This is the signal that a profile is about to stop accepting new endpoints or rules.

## 4. Security coverage

`Policies_List` — `GET /subscriptions/{subscriptionId}/providers/Microsoft.Cdn/cdnWebApplicationFirewallPolicies`
lists WAF policies. Cross-reference against the endpoints from step 2 to find delivery surfaces with
no WAF policy attached.

## 5. What the API can do here

`Operations_List` — `GET /providers/Microsoft.Cdn/operations` returns the resource provider's own
operation catalog. Useful for confirming which actions the current api-version exposes before you
plan a change.

## Rate budget

Reads refill at 25/second from a bucket of 250 per subscription per region. A large estate walk will
hit that. Read `x-ms-ratelimit-remaining-subscription-reads` on every response and slow down before
the 429 rather than after it.
