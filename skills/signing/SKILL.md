---
name: signing
description: Send, track, remind, download and verify Assinafy e-signatures through the Assinafy MCP tools. Use when the user wants a PDF or saved template signed, asks who has signed, wants a reminder sent, needs a signed (certificated) copy, wants a signature hash verified, or manages Assinafy signer contacts, including Email, WhatsApp and ICP-Brasil digital certificate (A1/A3) signing. Also for Portuguese requests such as assinar, assinatura, enviar para assinatura, lembrete, documento assinado or certificado digital.
---

# Assinafy signing flow

The `assinafy` MCP server (`https://mcp.assinafy.com.br/mcp`) exposes 14 tools. Grouped
tools take an explicit `action` and accept only that action's arguments. Full
request and response payloads for every tool and action are in
[references/tools.md](references/tools.md); read the relevant section before
building a non-trivial call.

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
| Send a new PDF to people by email | `assinafy_request_signatures` `from_pdf` |
| Upload now, send later | `assinafy_prepare_document` `upload` (optionally `rename`) → later `from_document` |
| Signing order, copy receivers, WhatsApp or certificate signers | `assinafy_prepare_document` `upload` (new PDF) → `assinafy_find_signers` / `assinafy_save_signer` → `assinafy_request_signatures` `from_document` |
| Let the user review before sending | `assinafy_review_signature_request` → send only after the user confirms |
| Place signature/initials/date/input fields on pages | `assinafy_place_fields`, or `from_document` with `method: "collect"` and `entries` |
| Use a saved template | `assinafy_find_templates` `list` → `assinafy_check_fields` `list`/`validate` → `assinafy_request_signatures` `from_template` |
| "Has everyone signed?" | `assinafy_find_documents` `list` → `get`, or `assinafy_show_document` |
| Remind a signer | `assinafy_find_documents` `get` (signer not `completed`, already `notified`, in the active `step`) → `assinafy_follow_up_assignment` `resend` |
| Extend the deadline | `assinafy_follow_up_assignment` `set_expiration` |
| Signed copy | `assinafy_find_documents` `get` (status `certificated`, artifact present) → `assinafy_download_document` `artifact` with `artifact: "certificated"` |
| Verify a signature hash | `assinafy_verify_document` |
| Delete a document | `assinafy_find_documents` `get` → `assinafy_delete_document`, only on explicit request |

## Verification and notification channels

Set per signer in `from_document`, `from_template`, `review_signature_request` and
`place_fields`:

| `verification_method` | Requirements | Credits per signer |
|---|---|---|
| `Email` (default) | Signer has an email | Included |
| `Whatsapp` | Signer has `whatsapp_phone_number` (E.164); WhatsApp notification; paid plan | 0.45 (WhatsApp notification) |
| `DigitalCertificate` | ICP-Brasil A1 or A3 certificate; signer has `government_id` (CPF or CNPJ); alone in its `step` (with other signers, set `step` on everyone); Digital Certificate feature enabled | 0.5 plus the notification channel |

Each request also uses one document from the plan allowance, or 1 credit once it is
exhausted. `notification_methods` accepts `Email` and/or `Whatsapp` (WhatsApp is paid); in
`from_template` each signer takes at most one channel. `from_pdf`
always uses Email verification and Email notification; use `from_document` for any
other channel. A CPF signer needs their own e-CPF (or an e-CNPJ naming them as legal
representative); a CNPJ signer needs that company's e-CNPJ. Signers inside documents
omit `government_id`; confirm it with `assinafy_find_signers` `get` and set it with
`assinafy_save_signer` `update` before sending. When the plan lacks a
feature, Assinafy answers `403` naming it; tell the user, do not retry.

## Document status

`uploading` → `uploaded` → `metadata_processing` → `metadata_ready` →
`pending_signature` → `certificating` → `certificated`. Alternatives: `expired` (try
`set_expiration`; if Assinafy refuses, start over), `rejected_by_signer`, `rejected_by_user`, `failed`.

- 100% signed can still be `certificating`. Download the signed PDF only at
  `certificated` with the `certificated` artifact listed.
- Deletable states: `metadata_ready`, `pending_signature`, `expired`,
  `rejected_by_signer`, `rejected_by_user`, `failed`.
- `collect` assignments need `metadata_ready`; `virtual` can start from `uploaded`.

## Rules

1. An explicit instruction to send, remind or delete authorizes that action for the
   named recipients; do not ask again. Ask only when a recipient, template role or
   document match is missing or ambiguous.
2. Resolve names to IDs with the find tools and reuse IDs across turns. Do not ask the
   user for technical identifiers, and keep IDs and base64 out of replies.
3. Inspect before writing: never upload a document that already has an ID, never create
   a second assignment for a document that has one, and confirm a signer is pending
   before a reminder. Each invitation and reminder can consume credits.
4. Writes are not retried automatically and composite calls are not transactional. When
   an error names a retained `document_id`, inspect it and continue with
   `from_document` instead of uploading again.
5. Files: `file_base64` is the PDF's base64 text (max 25 MiB decoded). Where a shell is
   available, produce it with `base64 -i <file>` rather than reading the PDF as pages;
   large files may exceed what fits in one tool call, so ask the user to upload those
   in Assinafy and continue from the document ID. Write downloaded base64 to disk
   (decode with `base64 -d`) instead of printing it.
6. You cannot sign, accept terms or enter verification codes for anyone. Signing links
   and codes in results are sensitive; never use or expose them.
7. Template authoring, webhook configuration and account administration are done in
   Assinafy, not through these tools.
