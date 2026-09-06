---
name: Screen a DNA order with Aclid
description: Submit customer sequences to Aclid for biosecurity screening, wait for the result, and read the per-framework regulatory findings that decide whether the order can ship.
api: openapi/aclid-openapi.yml
operations:
  - handle_v2_screen_fasta_v2_screen_fasta_post
  - Initiate_Screen__Inline__v2_screen_inline_post
  - handle_v2_screen_csv_v2_screen_csv_post
  - Retrieve_Screen_Summary_v2_screens__id__get
  - Retrieve_Screen_Details_v2_screens__id__details_get
  - Stream_Screen_v2_screens__id__stream_get
generated: '2026-09-06'
method: generated
source: openapi/aclid-openapi.yml + https://api.aclid.bio/docs
---

# Screen a DNA order with Aclid

Use this when a customer has placed a synthesis order and you need to know, before you ship,
whether any sequence in it is a controlled toxin, a select agent, a regulated pathogen or an
export-controlled element.

## Before you start

- Base URL is `https://api.aclid.bio`. HTTPS only.
- Send your API key as the **raw** value of the `Authorization` header. No `Bearer` prefix.
  `Authorization: <your key>`
- The key decides live vs test mode. Aclid publishes no key prefix, so **check which key you were
  handed before you run this in anger** — you cannot tell by looking at it.
- The published OpenAPI declares no `securitySchemes`. If you generated a client from it, the client
  will not send the header. Add it yourself.

## Steps

1. **Pick the right submission shape.**
   - A FASTA or FASTQ file → `handle_v2_screen_fasta_v2_screen_fasta_post`
     (`POST /v2/screen_fasta`, `multipart/form-data`).
   - A CSV of `name,sequence` rows → `handle_v2_screen_csv_v2_screen_csv_post`
     (`POST /v2/screen_csv`, `multipart/form-data`).
   - Sequences already in memory → `Initiate_Screen__Inline__v2_screen_inline_post`
     (`POST /v2/screen_inline`, `application/json`, body `{ "name": ..., "sequences": [ { "name": ..., "sequence": ... } ] }`).
   - Do **not** use `POST /v2/screen` or `POST /v2/screen_file`. Both are marked `deprecated: true`
     in the contract and point at `/v2/screen_fasta`.

2. **Always send an `idempotence_key`.** It is a field in the request body, not a header. Max 60
   characters; use a UUID derived from your own order id so a retry produces the same key.
   Within 24 hours a repeat of the same key is **not** re-assessed — Aclid answers `303` and
   redirects to the original screen's summary. Treat a `303` as success, not as an error, and
   follow it to the existing screen rather than submitting again.

3. **Set `name` to something you can find later.** It is a display label up to 100 characters that
   Aclid itself does not use — the docs' own example is `"Order #123456"`. It is the only
   caller-controlled correlation field the API has, so put your order id in it.

4. **Respect the size ceilings.** Total base pairs across all sequences in one screen may not exceed
   1,000,000,000. Each FASTA sequence must be at least 30 bp (FASTQ may be any non-zero length).
   Invalid FASTA/FASTQ is rejected. Split oversized orders into multiple screens, each with its own
   idempotence key.

5. **Poll, do not wait on a callback.** Set `asynchronous` if you do not want to block. There are no
   webhooks and no event stream. Call `Retrieve_Screen_Summary_v2_screens__id__get`
   (`GET /v2/screens/{id}`) until `ScreenStatus` reaches `succeeded` or `failed`. Intermediate
   values are `pending_upload`, `queued`, `running`. A `204` on the detail or stream read means
   "not ready yet" — back off and retry; it is not an error.

6. **Read the findings.** `Retrieve_Screen_Details_v2_screens__id__details_get`
   (`GET /v2/screens/{id}/details`) returns `ReportMetadata`. `findings` is a **map keyed by
   compliance framework id**, each value a `Finding` with `reason_code`, `regulatory_status`
   (`controlled` | `not_controlled`) and `material`. The framework ids are
   `us_ccl_export_control`, `eu_dual_use_export_control`, `us_select_agent`,
   `us_screening_framework`, `nih_recombinant_dna_guidelines`, `eu_directive_2000_54_ec`,
   `zkbs_oncogenes` and `usda_vs`. Look each `reason_code` up in
   `errors/aclid-compliance-reason-codes.yml` for the meaning and the compliance action.

7. **For a large result, stream it.** `Stream_Screen_v2_screens__id__stream_get`
   (`GET /v2/screens/{id}/stream`) returns `{ "url": ... }` — a pre-signed URL to a JSON or CSV
   file. Fetch that URL rather than paging the details endpoint.

## Rules

- **There is no undo.** The API is GET and POST only. No delete, cancel or archive operation exists.
  Once a screen is initiated you cannot withdraw it through the API, so validate the payload before
  you submit.
- **Do not retry blindly on error.** No rate limit is documented — no 429, no `RateLimit-*` headers,
  no `Retry-After` — so there is no signal telling you when to back off. Use your own bounded
  exponential backoff and a retry ceiling.
- **A `422` is your fault, not Aclid's.** The body is a FastAPI `HTTPValidationError`: `detail[]`
  entries with `loc`, `msg` and `type` pointing at the offending field. Fix the field; do not retry
  the same payload.
- **A `403` means the key was missing or wrong.** The body carries `request_id` — quote it to
  support. It is the only correlation identifier Aclid exposes, and only on errors.
- **Exemption codes are not clearances.** `housekeeping`, `unspecific` and
  `match_window_below_threshold` explain why a match was not escalated. They do not certify that an
  order is shippable; a human still owns that decision.
