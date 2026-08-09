---
name: Upload and tag a document
description: Push a statement or notice into Canoe, attach metadata and tags so it routes and extracts correctly, then read back the extracted data.
api: openapi/canoe-intelligence-api-openapi.yml
operations:
  - RequestingTokensClientCredentials
  - UploadDocuments
  - GetDocumentTags
  - GetDocumentAllocationTags
  - SetDocumentMetadata
  - BulkSetDocumentMetadata
  - updateAllocationTags
  - GetASingleDocument
  - GetDocuments
generated: '2026-08-09'
method: generated
source: https://api.canoesoftware.com/docs
---

# Upload and tag a document

For documents Canoe cannot retrieve itself from a GP portal — anything that arrives by email, from a
custodian feed, or from your own archive.

## Step 0 — Token

`RequestingTokensClientCredentials` — `POST /oauth/token/client-credentials`. 24-hour lifetime.

## Step 1 — Upload

`UploadDocuments` — `POST /v1/documents`, `multipart/form-data`.

Set `client_document_id` to **your** identifier for the file. It is the only handle you control, and
it is your duplicate guard — see the retry rule below. The response is
`{id, name, client_document_id}`; keep the Canoe `id`.

## Step 2 — Read the tag vocabularies before you tag

- `GetDocumentTags` — `GET /v1/documents/tags` — tenant-defined document tags.
- `GetDocumentAllocationTags` — `GET /v1/documents/allocation-tags` — allocation-scoped tags.

Both paginate with `page` / `limit`, capped at 100. Tag ids are tenant-specific; never hard-code them.

## Step 3 — Set metadata

- `SetDocumentMetadata` — `PUT /v1/documents/{id}/metadata` for one document.
- `BulkSetDocumentMetadata` — `POST /v1/documents/metadata` for a batch.

Settable: `document_status`, `approval_status`, `comment`, `tags`, `client_document_id`.
`ready_for_extract` on the document record is what gates extraction — a document sitting untouched
is usually one that was uploaded but never marked ready.

## Step 4 — Attach allocation tags

`updateAllocationTags` — `PATCH /v1/documents/{allocationId}/allocation-tags`

This is the one `PATCH` in the API. It binds the document to the allocation whose position the
document reports against.

## Step 5 — Read back

- `GetASingleDocument` — `GET /v1/documents/{id}` for the document record and its `allocations`.
- `GetDocuments` — `GET /v1/documents/data` for the extracted data. Send
  `X-API-VERSION: 2022-09-19` to enable `page` / `limit`, and use `fields` to trim the response.
  You can filter by `downstream_ids` here — but only when the version header is set **and**
  pagination is supplied.

## Rules

- **No idempotency key exists.** If an upload times out, do **not** resend. Query
  `GET /v1/documents/data` filtered by your `client_document_id` first; upload only if it is absent.
  A re-sent upload creates a second document and a second extraction.
- `422` on upload is a rejected file (type, size, or a metadata field). `403` is entitlement.
- Documents can be locked (`is_locked`); a locked document will not accept metadata writes.
- Deletes are destructive and unbatched-safe: `DeleteDocument` (`DELETE /v1/documents/{id}`) and
  `DeleteMultipleDocuments` (`POST /v1/documents/delete-documents`). Require human confirmation
  before calling either.
