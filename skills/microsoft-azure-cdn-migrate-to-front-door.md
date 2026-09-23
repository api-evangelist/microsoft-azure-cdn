---
name: migrate-to-front-door
description: Move a retiring Azure CDN classic profile to Azure Front Door Standard or Premium using the validate / migrate / commit sequence, with abort available until commit.
api: microsoft-azure-cdn
generated: '2026-09-17'
method: generated
source: >-
  openapi/microsoft-azure-cdn-management-openapi.json (api-version 2026-07-01),
  https://learn.microsoft.com/en-us/azure/frontdoor/front-door-cdn-comparison
operations:
  - Profiles_Get
  - Profiles_CanMigrate
  - Profiles_CdnCanMigrateToAfd
  - Profiles_Migrate
  - Profiles_CdnMigrateToAfd
  - Profiles_MigrationCommit
  - Profiles_MigrationAbort
---

# Migrate a classic CDN profile to Azure Front Door

Azure CDN Standard from Microsoft (classic) retires on **2027-09-30** and has not accepted new
instances since 2025. Microsoft ships the migration as a first-class API, which is unusual and worth
using rather than rebuilding by hand.

The sequence has a **staging step and a commit step, and it is reversible in between**. That property
is what makes this safe to automate.

## 1. Read the profile

`Profiles_Get` — confirm the SKU is a classic CDN SKU. An `AzureFrontDoor` SKU is already migrated.

## 2. Validate — this changes nothing

`Profiles_CanMigrate` — `POST .../resourceGroups/{rg}/providers/Microsoft.Cdn/canMigrate`
or `Profiles_CdnCanMigrateToAfd` — `POST .../profiles/{profileName}/cdnCanMigrateToAfd`

Both are long-running dry runs. Resolve every reported incompatibility here. The documented blockers
are custom domains without HTTPS and inconsistent session-affinity settings across domains sharing an
origin group.

## 3. Stage the migration

`Profiles_Migrate` or `Profiles_CdnMigrateToAfd`. The contract is explicit about what this does and
does not do: *"This step prepares the profile for migration and will be followed by Commit to
finalize the migration."* Traffic has not moved. Nothing is final.

## 4. Decide

- **Abort** — `Profiles_MigrationAbort`, `POST .../profiles/{profileName}/migrationAbort`. Valid at
  any point **before** commit. This is the reversal window and it is stated in the contract itself.
- **Commit** — `Profiles_MigrationCommit`. After this the migration is final and there is no reverse
  operation in the contract.

## 5. Cut over DNS

Migration does not change DNS. Update your records to the new Front Door endpoint only after commit,
per https://learn.microsoft.com/en-us/azure/frontdoor/tier-migration.

## Agent rules

- Never call `Profiles_MigrationCommit` on the same turn as `Profiles_Migrate`. Commit is the
  irreversible step and it should be a separate, explicitly confirmed decision.
- Every step here is long-running. Poll each one to a terminal state before the next.
