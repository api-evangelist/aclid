---
name: Verify an Aclid customer and record the decision
description: Create a customer (which runs a sanctions and watchlist screen), send them through Aclid's hosted or embedded verification flow, and record an approve/reject/escalate decision as a note.
api: openapi/aclid-openapi.yml
operations:
  - Create_customer_v2_customers_post
  - List_customers_v2_customers_get
  - Retrieve_customer_v2_customers__requester_id__get
  - Create_Verification_URL_v2_verification_url__post
  - Retrieve_Verification_v2_verifications__screen_id__get
  - Create_or_Update_a_Note_v2_notes_post
  - List_Screen_Notes_v2_screens__screen_id__notes_get
  - List_Customer_Notes_v2_customers__requester_id__notes_get
generated: '2026-09-06'
method: generated
source: openapi/aclid-openapi.yml + https://api.aclid.bio/docs
---

# Verify an Aclid customer and record the decision

Sequence screening answers "is this material controlled". This skill answers the other half —
"is this buyer legitimate" — and leaves an auditable decision behind.

## Before you start

Same auth as every Aclid call: the raw API key in the `Authorization` header, HTTPS only, and the
key selects live vs test mode. See `authentication/aclid-authentication.yml`.

## Steps

1. **Create the customer.** `Create_customer_v2_customers_post` (`POST /v2/customers`) takes
   `name`, `company`, `address`, `country` and a `screen_id`. Creating the customer **also
   initiates a sanctions and watchlist screen** — this one call has a side effect beyond the record
   it creates, so treat it as a write with consequences.

   **This operation accepts no `idempotence_key`.** Unlike the screen endpoints it has no replay
   protection, and there is no delete-customer operation to clean up a duplicate. Guard it on your
   side: check `List_customers_v2_customers_get` (`GET /v2/customers`, filter with `search_str`)
   before you create, and make the call exactly once.

2. **Mint a verification URL.** `Create_Verification_URL_v2_verification_url__post`
   (`POST /v2/verification_url/`) takes the `screen_id` from step 1 of the screening flow and an
   optional `redirect_url`. It returns a URL you hand to the customer. Two ways to use it:

   - **Hosted** — redirect the customer to the URL. Aclid hosts the flow; branding and theming are
     customisable, and `redirect_url` brings them back to you afterwards.
   - **Embedded** — load `<script src="https://verify.aclid.bio/widget.js"></script>` and call
     `Aclid.showEmbeddedVerification({ verificationUrl, onSuccess })`. Your domain must be
     allow-listed by the Aclid team first; this is a manual step, not self-serve.

   Inside the flow the customer can link an ORCID iD, which Aclid says expedites the biosecurity
   review.

3. **Read the verification result.** `Retrieve_Verification_v2_verifications__screen_id__get`
   (`GET /v2/verifications/{screen_id}`). `VerificationStatus` is one of `submitted`,
   `partially_submitted`, `missing_verification`, `not_required`. Only `submitted` means the
   customer completed it; `partially_submitted` is not a pass.

4. **Record the decision as a note.** `Create_or_Update_a_Note_v2_notes_post` (`POST /v2/notes`)
   takes `screen_id`, `content` and `decision_status` — `awaiting`, `approved`, `rejected` or
   `escalated`. This is the operation that writes the human judgement into the audit trail.

5. **Read the trail back.** `List_Screen_Notes_v2_screens__screen_id__notes_get`
   (`GET /v2/screens/{screen_id}/notes`) and
   `List_Customer_Notes_v2_customers__requester_id__notes_get`
   (`GET /v2/customers/{requester_id}/notes`). Each `Note` carries `note_id`, `created_by`,
   `created_at` (unix seconds), `note_content` and `decision_status`.

## Rules

- **The note is the only reversible write in this API.** Passing an existing `note_id` back to
  `POST /v2/notes` updates that note instead of creating a new one, so a `decision_status` can be
  moved back to `awaiting`. Aclid publishes **no time window** for this — do not assume one, and do
  not tell a user a decision can be reversed "within N days". Nothing else here can be undone:
  there is no delete-customer, no revoke-verification-URL and no cancel-screen operation.
- **Never auto-approve.** `decision_status` encodes a compliance judgement about export control and
  biosecurity. An agent may gather the evidence, summarise the findings and propose a status; a
  named human sets `approved` or `rejected`. `escalated` is the correct value when you are unsure.
- **Do not paste customer identity data into logs or prompts you do not control.** Name, company,
  address and country are the payload here, and the whole point of the screen is that these people
  are being checked against sanctions and watchlists.
- **Errors:** `422` is a FastAPI `HTTPValidationError` with `detail[].loc` naming the bad field;
  `404` means the customer, screen, note or verification does not exist; `403` means the key is
  missing or wrong (quote the `request_id` from the body to support).
