---
name: Pull extracted document data for a fund
description: Authenticate to the Canoe API, locate a fund, and retrieve the data Canoe extracted from that fund's documents, including the source PDFs.
api: openapi/canoe-intelligence-api-openapi.yml
operations:
  - RequestingTokensClientCredentials
  - GetFunds
  - GetSingleFunds
  - GetSingleFundsDocumentdata
  - GetSingleFundsDocumentId
  - DownloadSingleFundsDocumentdata
generated: '2026-08-09'
method: generated
source: https://api.canoesoftware.com/docs
---

# Pull extracted document data for a fund

The core read flow: get a token, find the fund, pull the extracted data, optionally pull the source documents.

Base URL: `https://api.canoesoftware.com`

## Before you start

- You need an OAuth client (`client_id` / `client_secret`) created in the Canoe API
  Configuration Dashboard at `https://client.canoesoftware.com/api_configuration`.
- Your tenant may only have access to a subset of endpoints — access depends on the services
  purchased from Canoe and on the API user's permissions. A `403` means authenticated but not
  entitled, not a bad token.
- Your network may need ports `443` and `9443` open, and may need Canoe's IP ranges allowlisted.

## Step 1 — Get an access token

`RequestingTokensClientCredentials` — `POST /oauth/token/client-credentials`

Send `client_id` and `client_secret`. The response is `{token_type, expires_in, access_token}`.
Tokens are valid for **24 hours**. Cache the token for its lifetime; do not request a new one per call.

Send on every subsequent request:

```
Authorization: Bearer {access_token}
Accept: application/json
X-Requested-With: XMLHttpRequest
```

## Step 2 — Find the fund

`GetFunds` — `GET /v1/funds`

To paginate you **must** send `X-API-VERSION: 2022-02-24`. Without it the endpoint returns the
unpaginated legacy shape and `page` / `perPage` are ignored. Page size caps at 100.

Pagination metadata comes back in **response headers** (`total`, `first`, `next`, `prev`, `last`),
not in the body — the body is a bare JSON array.

Use `GetSingleFunds` (`GET /v1/funds/{id}`) once you have an id, or `GetFundsDetailedData`
(`GET /v1/funds/details`) when you also need term and allocation detail on the same record.

## Step 3 — Pull the extracted data

`GetSingleFundsDocumentdata` — `GET /v1/funds/{id}/document-data`

Send `X-API-VERSION: 2024-10-07` to enable `page` / `limit`. Use the `fields` parameter — a
comma-separated allowlist of response field names — to keep payloads small when you only need a few
of the 2,586 possible extraction fields.

Narrow with the date-range pairs the endpoint exposes (`data_date_start` / `data_date_end`,
`file_upload_time_start` / `file_upload_time_end`, `last_modified_time_start` / `last_modified_time_end`)
rather than pulling everything and filtering client-side.

## Step 4 — Get the source documents (optional)

- `GetSingleFundsDocumentId` — `GET /v1/funds/{id}/document-ids` returns `{total, ids}`. Use this
  first to size the job and to diff against what you already hold.
- `DownloadSingleFundsDocumentdata` — `GET /v1/funds/{id}/documents` returns the documents
  themselves, up to 100 per page.

## Rules

- **Do not blind-retry writes.** Canoe publishes no idempotency key. This flow is read-only, so
  retries are safe here — but the same is not true of the create/update skills.
- **Honour `retry_after`.** A `429` on the auth endpoints carries `error_code: RATE_LIMIT_EXCEEDED`
  and a `retry_after` in seconds. Back off exponentially; do not hammer token issuance.
- **Errors are plain JSON**, not RFC 9457. Auth failures carry `error_code`, `error_description`
  and `hint`. Read `error_code`, not the prose.
- `400` is a malformed request, `401` is a missing/invalid token, `403` is entitlement, `404` is a
  fund that does not exist for you. See `errors/canoe-intelligence-problem-types.yml`.
