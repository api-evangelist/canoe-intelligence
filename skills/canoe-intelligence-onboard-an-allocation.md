---
name: Onboard an LP allocation
description: Build the Canoe record chain — organization, entity, account, fund, term — and create the allocation that Canoe extracts positions against.
api: openapi/canoe-intelligence-api-openapi.yml
operations:
  - RequestingTokensClientCredentials
  - Organizations
  - FindById
  - CreateASingleEntity
  - CreateASingleAccount
  - CreateSingleFund
  - CreateSingleTerm
  - GetTerms
  - CreateSingleAllocation
  - GetAllocation
generated: '2026-08-09'
method: generated
source: https://api.canoesoftware.com/docs
---

# Onboard an LP allocation

An allocation is the join between an investing entity/account and a fund term. Everything Canoe
extracts hangs off it, so the chain has to exist before the allocation does.

Order: **Organization → Entity → Account → Fund → Term → Allocation.**

## Step 0 — Token

`RequestingTokensClientCredentials` — `POST /oauth/token/client-credentials`. See
`conventions/canoe-intelligence-conventions.yml` for the header set. 24-hour token lifetime.

## Step 1 — Resolve the organization (the fund sponsor)

`Organizations` — `GET /v1/organizations` (send `X-API-VERSION: 2023-08-07` to paginate), or
`FindById` — `GET /v1/organizations/{id}`.

Organizations are self-referencing via `parent_id`, so a sponsor and its sub-managers are one tree.
Search the existing tree before creating anything — duplicate sponsors fragment extraction.

## Step 2 — Create the investing entity

`CreateASingleEntity` — `POST /v1/organizations/entities`

This is the LP side. Entities also nest via `parent_id`. Set `aliases` generously: alias matching is
how Canoe reconciles the names that appear on GP statements against your entity.

Record your own identifier in `downstream_ids` (key = downstream system name, value = its id) so you
can map back without maintaining a side table.

## Step 3 — Create the account

`CreateASingleAccount` — `POST /v1/organizations/accounts`

Accounts group positions under an entity. Allocations reference this by `account_id`.

## Step 4 — Ensure the fund and its term exist

- `CreateSingleFund` — `POST /v1/funds`. Set `organization_id` to the sponsor from Step 1, plus
  `name`, `legal_name`, `sponsor`, `start_date`, and `aliases`.
- `GetTerms` — `GET /v1/terms` to check whether the share class already exists.
- `CreateSingleTerm` — `POST /v1/terms` if it does not. The term carries `fund_id`,
  `organization_id`, `designation`, and `date_format`. **The term, not the fund, is the extraction
  unit** — an allocation binds to a term.

## Step 5 — Create the allocation

`CreateSingleAllocation` — `POST /v1/allocations`

Bind `entity_id`, `account_id`, `investment_id` (the fund) and `term_id`, and set `initial_date`
and `initial_amount`. Put your own position identifier in `custom_allocation_id`.

Verify with `GetAllocation` — `GET /v1/allocations` filtered by `custom_allocation_id`. Watch
`extraction_status` on the returned record: that is Canoe telling you whether the position is ready
to receive extracted data.

## Rules

- **There is no idempotency key.** Every step here is a create. If a `POST` times out or you get a
  5xx, **do not resend it** — re-query first (`GET /v1/allocations` by `custom_allocation_id`,
  `GET /v1/funds` by name, `GET /v1/terms` by `fund_id`) and only create if the record is absent.
  A duplicated entity or allocation is a data-quality incident, not a retry.
- **`422` is validation.** Read the body, fix the field, resend. **`403` is entitlement** — the
  write endpoints are gated by purchased services; retrying will not help.
- Set `custom_fields` in the same call where you can; `GetCustomFields` (`GET /v1/custom_fields`)
  lists the definitions your tenant has.
