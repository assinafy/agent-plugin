---
name: signing
description: Send, track, remind, download and verify Assinafy e-signatures through the Assinafy MCP tools. Use when the user wants a PDF or saved template signed, asks who has signed, wants a reminder sent, needs a signed (certificated) copy, wants a signature hash verified, or manages Assinafy signer contacts, including cost previews, Email, WhatsApp, biometric ID and liveness, and ICP-Brasil digital certificate (A1/A3) signing. Also use for Assinafy contract terms, amendments, renewals and lifecycle reminders (gestão de contratos, aditivos, renovação and vencimento).
---

# Assinafy signing and contract flow

The `assinafy` MCP server (`https://mcp.assinafy.com.br/mcp`) exposes 29 tools. Signing tools keep their individual operations; two CLM tools
group fixed read and write actions separately. Full request and response
payloads for every tool are in [references/tools.md](references/tools.md); read the
relevant section before building a non-trivial call.

## Connection

- One OAuth connection authorizes one Assinafy workspace. If a workspace tool returns
  `401` or the server needs authentication, ask the user to connect or re-authenticate
  Assinafy in their client's MCP or connector settings. Never ask for,
  accept or pass API keys, tokens or `account_id` overrides; the workspace comes from
  the grant.
- `assinafy_verify_document` is public and works before sign-in.
- HTTP `403 insufficient_scope` means the grant lacks a permission; ask the user to
  reconnect and approve the scopes named in the error. Nothing was sent.

## Pick the flow

| User intent | Calls, in order |
|---|---|
| Send a new PDF to up to 10 people by email | `assinafy_send_pdf_for_signature` |
| Upload now, send later | `assinafy_upload_document` (optionally `assinafy_rename_document`) → later `assinafy_create_assignment` |
| Signing order, copy receivers, WhatsApp, biometric or certificate signers | `assinafy_upload_document` (new PDF) → `assinafy_list_signers` / `assinafy_create_signer` → `assinafy_create_assignment` |
| Preview signing or reminder costs | Set `estimate_only: true` on the matching PDF, assignment, template or reminder tool; [cost preview rules](references/tools.md#cost-previews) |
| Let the user review before sending | `assinafy_review_signature_request` → send only after the user confirms |
| Place signature/initials/date/input fields on pages | `assinafy_place_fields`, or `assinafy_create_assignment` with `method: "collect"` and `entries` |
| Use a saved template | `assinafy_list_templates` → `assinafy_list_fields` / `assinafy_validate_fields` → `assinafy_create_document_from_template` |
| "Has everyone signed?" | `assinafy_list_documents` → `assinafy_get_document`, or `assinafy_show_document` |
| "Tell me when it's signed" | ChatGPT Work watches the document through MCP Events on its own; elsewhere, check when asked and never poll |
| Remind a signer | `assinafy_get_document` (signer not `completed`, already `notified`, in the active `step`) → `assinafy_resend_notification` |
| Find contracts or review upcoming renewals | `assinafy_browse_contracts`; read [contract management](references/tools.md#contract-management) |
| Set terms, link an amendment, renew or terminate | Read with `assinafy_browse_contracts`, then an authorized `assinafy_update_contract`; retain conflict guards |
| Watch contract notices or lifecycle changes | Subscribe to `contract.activity` when Events are available; Assinafy owns scheduled owner emails |
| Extend the deadline | `assinafy_set_assignment_expiration` |
| Signed copy | `assinafy_get_document` (status `certificated`, artifact present) → `assinafy_download_document` with `artifact: "certificated"` |
| Verify a signature hash | `assinafy_verify_document` |
| Delete a document | `assinafy_get_document` → `assinafy_delete_document`, only on explicit request |

## Verification and notification channels

Set per signer in `assinafy_create_assignment`, `assinafy_create_document_from_template`,
`assinafy_review_signature_request` and `assinafy_place_fields`:

| `verification_method` | Requirements | Credits per signer |
|---|---|---|
| `Email` (default) | Signer has an email | Included |
| `Whatsapp` | Signer has `whatsapp_phone_number` (E.164); WhatsApp notification; paid plan | 0.45 (WhatsApp notification) |
| `DigitalCertificate` | `method: "virtual"`; ICP-Brasil A1 or A3 certificate; CPF/CNPJ saved in `government_id`; alone in its `step` (with other signers, set `step` on everyone); Digital Certificate feature enabled | 0.5 plus the notification channel |
| `IdDocumentFaceMatchBasic` | ID photos and liveness; valid CPF saved in `government_id`; ID Document + Face Match feature enabled | Use the cost preview |

Each signature request uses one plan document, or 1 credit once exhausted. Use the
API's cost preview for current costs, balances and blocking reasons; estimates do
not reserve credits. A PDF preview uploads and retains its document; reuse that ID
with `assinafy_create_assignment` to send. Other previews create no records or
messages. `estimate_only` is not accepted by review or placement cards.

Each signer takes exactly one compatible notification channel: Email verification
uses Email, Whatsapp uses Whatsapp, and certificate/biometric verification accepts
either. Omitted settings are inferred. The common PDF tool always uses Email;
other methods use the assignment or template flow.

For a saved template, an inline `full_name` and `email` contact is Email-only.
WhatsApp, certificate and biometric signing require an existing signer `id`; create
or update the authorized contact first. Template signing steps are all-or-none and
contiguous from 1, with a certificate signer alone in its step.

`DigitalCertificate` requires a virtual assignment. Refuse
`assinafy_create_assignment` with any other `method`. Before
`assinafy_create_document_from_template`, inspect the selected template; any placed
field makes it `collect`, so refuse a certificate signer. Both tools reject their
cost-preview and send paths before writing or charging with the stable code
`digital_certificate_requires_virtual`.

Read the contact's `government_id` before certificate or biometric requests. If
missing, save an authorized CPF/CNPJ through `assinafy_create_signer` or
`assinafy_update_signer`, or let the user enter it in Assinafy. Biometrics requires
a valid CPF; a certificate requires the matching person's e-CPF or company's
e-CNPJ. Handle identity data only for the authorized contact operation. The human
completes certificate signing, ID photos and liveness in Assinafy. Report a plan
feature refusal instead of repeating the call.

## Document status

`uploading` → `uploaded` → `metadata_processing` → `metadata_ready` →
`pending_signature` → `certificating` → `certificated`. Alternatives: `expired` (try
`assinafy_set_assignment_expiration`; if Assinafy refuses, start over), `rejected_by_signer`, `rejected_by_user`, `failed`.

- `identity_verification_status: InReview` requires the account to review identity
  in Assinafy. Once all signers finish, that document waits at
  `pending_identity_review` until approval;
  rejection cancels the document. The assistant reports the state and next step.
- 100% signed can still be `certificating` or `pending_identity_review`. Download the signed PDF only at
  `certificated` with the `certificated` artifact listed.
- Deletable states (identity review is not deletable): `metadata_ready`, `pending_signature`, `expired`,
  `rejected_by_signer`, `rejected_by_user`, `failed`.
- `collect` assignments need `metadata_ready`; `virtual` can start from `uploaded`.

## Rules

1. An explicit instruction to send, remind, change contract terms, renew, terminate or delete authorizes that action for the
   named recipients; do not ask again. Ask only when a recipient, template role or
   document match is missing or ambiguous.
2. Resolve names to IDs with the list and get tools and reuse IDs across turns. Do not ask the
   user for technical identifiers, and keep IDs and base64 out of replies.
3. Inspect before writing: never upload a document that already has an ID, never create
   a second assignment for a document that has one, and confirm a signer is pending
   before a reminder. Each invitation and reminder can consume credits.
4. Writes are not retried automatically and composite calls are not transactional. When
   an error names a retained `document_id`, inspect it and continue with
   `assinafy_create_assignment` instead of uploading again.
5. Files: `file_base64` is the PDF's base64 text (max 25 MiB decoded). Where a shell is
   available, produce it with `base64 -i <file>` rather than reading the PDF as pages;
   large files may exceed what fits in one tool call, so ask the user to upload those
   in Assinafy and continue from the document ID. Downloads keep base64 only in
   `structuredContent`; write it to disk (decode with `base64 -d`) instead of printing it.
6. You cannot sign, accept terms or enter verification codes for anyone. Signing links
   and codes in results are sensitive; never use or expose them.
7. Template authoring, webhook configuration and account administration are done in
   Assinafy, not through these tools.

## Contract lifecycle

Read [contract management](references/tools.md#contract-management) before a CLM
write. Reuse the current `updated_at` and, for renewals, `term`. A conflict requires
a fresh read and review; a timeout requires inspecting current terms and activities.
For a signed amendment, set the delta on a separate document linked to the parent
and let Assinafy apply it after certification. Contract dates differ from signing
deadlines. Assinafy schedules owner notices and auto-renewal; Events report actual
activities while the client keeps its subscription refreshed. Read current terms
before acting on an alert, which grants no permission for another write or email.
