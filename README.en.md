# Assinafy agent plugin

*[Leia em português](README.md) · English*

Send PDFs or saved templates for electronic signature in Brazil from your AI assistant.
Use Email, WhatsApp, biometric ID and liveness, or ICP-Brasil A1/A3 certificates;
track signatures, send reminders, download certified PDFs and manage contract terms.
The plugin connects to `https://mcp.assinafy.com.br/mcp` with OAuth 2.1 and adds the
`signing` skill. No API key, client secret or local server is needed.

## Install

### Claude Code

```text
/plugin marketplace add assinafy/agent-plugin
/plugin install assinafy@assinafy
```

Choose the install scope, then open `/mcp`, select **plugin:assinafy:assinafy** and
sign in. Run `/reload-plugins` if requested. Without a browser, use
`claude mcp login --no-browser plugin:assinafy:assinafy` and follow its instructions.
Update with `claude plugin update assinafy@assinafy`; remove with
`claude plugin uninstall assinafy@assinafy`. The skill is also available as `/assinafy:signing`.

### Claude apps and Cowork

Open **Customize → Plugins → Add → Add marketplace**, enter `assinafy/agent-plugin`
and install **Assinafy**. Open its **Connectors** tab and connect. A Team or Enterprise
Owner may need to allow the source or add the connector. For tools alone, add a custom
connector at the MCP URL above, leave OAuth client credentials empty, then connect.

### Codex

```bash
codex plugin marketplace add assinafy/agent-plugin
codex plugin add assinafy@assinafy
codex mcp login assinafy --scopes account:read,documents:read,documents:write,templates:read,templates:write,offline_access
```

### ChatGPT

Install [Assinafy from the ChatGPT plugin directory](https://chatgpt.com/plugins/plugin_asdk_app_6acbe48b99f881919f32df8e549c1efa),
then connect. For tools alone, add the MCP URL as a remote connector using OAuth;
select CIMD if offered and leave client credentials empty.

### VS Code with GitHub Copilot

Run **Chat: Install Plugin From Source** with `https://github.com/assinafy/agent-plugin`.
Enable `chat.plugins.enabled` and complete the browser sign-in. Alternatively, set
`"chat.plugins.marketplaces": ["assinafy/agent-plugin"]` and install from the marketplace.

## Sign in

The client discovers Assinafy's OAuth server, opens browser consent with PKCE S256,
and exchanges the authorization code for access and refresh tokens. Choose one workspace
and approve `account:read`, `documents:read`, `documents:write`, `templates:read`,
`templates:write` and `offline_access`. The client manages tokens; never put credentials
in prompts or tool arguments. A tool needing an unapproved scope returns `403 insufficient_scope`.
Reconnect to change workspaces. Public signature-hash verification needs no sign-in.

## Complete signing flow

1. **Find or upload.** Resolve an existing document by name, or upload a PDF (up to 25 MiB).
   Uploading alone sends nothing. Wait for `metadata_ready` when placing fields.
2. **Prepare recipients.** Reuse contacts or create authorized ones. Choose verification,
   one compatible notification channel per signer, signing order and optional copy receivers.
   Save an authorized CPF/CNPJ before certificate signing; biometrics requires a valid CPF.
3. **Choose the source.** For a new PDF and up to 10 Email signers, use
   `assinafy_send_pdf_for_signature`. For other methods, order or copies, upload then
   use `assinafy_create_assignment`. For templates, find roles and fields, validate values,
   and use `assinafy_create_document_from_template`. Inline template contacts are Email-only;
   other methods need saved signer IDs. Certificate templates must have no placed fields.
4. **Preview and send.** Set `estimate_only: true` on the PDF, assignment, template or
   reminder tool to see current costs, balances and blocking reasons. PDF previews retain
   the uploaded document; reuse its ID to send. Other previews create no records or messages.
   Estimates reserve no credits. An explicit send request authorizes sending; if you request
   a review, the assistant waits for confirmation. Review and field-placement cards send
   through their controls and do not accept `estimate_only`.
5. **The recipient signs.** The invitation opens Assinafy. The human accepts terms,
   verifies identity with Email/WhatsApp code, their A1/A3 certificate, or ID photos and
   liveness, reviews the document, fills fields and signs or declines. The assistant cannot
   sign, accept terms or enter codes. Later steps are notified after earlier steps finish.
6. **Follow up.** Read document status and each signer's notification history (`sent`,
   `delivered`, `opened`, `failed`); sending is not proof of delivery or signature. WhatsApp
   history and paged activities provide further detail. Remind only a pending signer in the
   active step, once per request. Set or clear the deadline without sending a reminder.
   Contact changes invalidate unverified links; verified channels on unfinished documents
   cannot change. Send another reminder only when requested.
7. **Wait for certification.** After signing, wait through `certificating` to `certificated`.
   Identity checks in `InReview` require account review in Assinafy; after all signatures,
   `pending_identity_review` waits for approval. Rejection cancels with cancellation notices.
8. **Download and verify.** Download the `certificated` artifact when present. Other artifacts
   are `original`, `certificate-page`, `bundle` and `pades`; page images have their own tool.
   Save binary results as files. Verify the printed signature hash publicly; `verified_at`
   is the time of the check, not the signing or certification time.
9. **Recover or delete.** Writes are not retried automatically. If a composite send fails
   after upload, inspect the retained document and continue from it. Delete only on request
   and in a deletable state. If an expired request cannot be extended, prepare a new request.

In ChatGPT Work, ask to watch a document for status changes or the signed copy.
Other clients check when asked. Compatible MCP Apps hosts display status, review,
field-placement and contract cards; other hosts receive text/JSON.

## Methods, costs and statuses

| Verification | Requirements | Cost |
|---|---|---|
| Email | Email address; Email notification | Use the current estimate |
| WhatsApp (`Whatsapp`) | E.164 phone; WhatsApp notification; paid plan | Use the current estimate |
| Certificate (`DigitalCertificate`) | Virtual assignment; feature enabled; matching CPF/CNPJ; alone in its signing step | Use the current estimate |
| Biometrics (`IdDocumentFaceMatchBasic`) | Feature enabled; valid CPF; ID photos and liveness | Use the current estimate |

Certificate and biometric verification accept either Email or WhatsApp notification.
All steps must be set or all omitted; numbered steps are contiguous from 1. A CPF signer
needs their e-CPF or an e-CNPJ naming them as representative; a CNPJ needs that company's
e-CNPJ. Costs depend on the plan and notification channel. Null balances mean unlimited;
check `has_sufficient_resources` and `blocking_reason`, not just the displayed balance.

| Status | Meaning | Deletable |
|---|---|:---:|
| `uploading`, `uploaded`, `metadata_processing` | Processing | no |
| `metadata_ready` | Ready | yes |
| `pending_signature` | Waiting for signatures | yes |
| `pending_identity_review` | Waiting for account identity review | no |
| `certificating`, `certificated` | Certifying / certified | no |
| `expired`, `rejected_by_signer`, `rejected_by_user`, `failed` | Expired, declined or failed | yes |

## Contract management

Use `assinafy_browse_contracts` to list, get terms or inspect related amendments.
For upcoming renewals, use `status: active` and `ending_within_days: 30`.
Authorized writes use `assinafy_update_contract`: `set`, `renew` or `terminate`.
Read first and reuse `updated_at`; renew also requires the current `term`. After a conflict
or timeout, inspect current terms and activities before continuing. Partial updates change
only supplied fields; external references replace the whole list. Signed amendments use a
separate document linked by `parent_document_id` and `relation_type`, applied after certification.

Assinafy runs lifecycle checks hourly in Brasília time and sends configured owner reminders.
This is separate from signature deadlines. `contract.activity` Events deliver new activities
while the client refreshes OAuth and its subscription; they do not authorize another action.
See [the complete contract flow and payloads](skills/signing/references/tools.md#contract-management).

## Tools and payloads

All 29 tools have [request/response examples and API routes](skills/signing/references/tools.md)
and [complete input and response schemas](skills/signing/references/tool-payloads.json).

| Purpose | Tools (`assinafy_` prefix) |
|---|---|
| Documents | `list_documents`, `get_document`, `list_document_activities`, `list_whatsapp_notifications`, `show_document` |
| Preparation | `upload_document`, `rename_document`, `review_signature_request`, `place_fields` |
| Send | `send_pdf_for_signature`, `create_assignment`, `create_document_from_template` |
| Follow up | `resend_notification`, `set_assignment_expiration` |
| Contacts | `list_signers`, `get_signer`, `create_signer`, `update_signer` |
| Templates and fields | `list_templates`, `get_template`, `list_fields`, `get_field`, `validate_fields` |
| Files and verification | `download_document`, `download_document_page`, `verify_document`, `delete_document` |
| Contracts | `browse_contracts`, `update_contract` |

Lists normally return 20 records, capped at 50 by the API; follow returned pagination metadata.
Template authoring, envelopes, embedded signing, spending ceilings, webhooks, identity review
and account administration are [REST API / Assinafy features](https://api.assinafy.com.br/v1/docs),
not additional tools or arguments in this plugin. Paid plans support up to three account
webhook endpoints; save signing secrets at creation. Account webhooks are independent of MCP Events.

## Examples

- “Send contract.pdf to Ana Example, ana@example.com, for signature.”
- “Preview the cost of biometric signing without sending.”
- “Use the NDA template, then let me review before sending.”
- “Has everyone signed? Remind Ana if she is still pending in the active step.”
- “Download the signed contract and check its signature hash.”

## Connect without the plugin

For tools without the skill, use a remote OAuth MCP connector with the same URL.
Claude Code: `claude mcp add --transport http assinafy https://mcp.assinafy.com.br/mcp`, then `/mcp`.
Codex/IDE: `codex mcp add assinafy --url https://mcp.assinafy.com.br/mcp`, then the login command above.
VS Code MCP settings: `{"servers":{"assinafy":{"type":"http","url":"https://mcp.assinafy.com.br/mcp"}}}`.

## Troubleshooting and privacy

| Symptom | Action |
|---|---|
| Authentication required, `401`, wrong workspace | Disconnect/clear authentication and reconnect in the client |
| `403 insufficient_scope` | Reconnect and approve the permission named in the error |
| Plan or balance refusal | Check the estimate and feature entitlement; do not repeat the send |
| `429` / `503` | Respect `Retry-After` / wait for service recovery; inspect state before retrying writes |
| Signed PDF unavailable | Wait for certification or resolve identity review in Assinafy |
| Retained-document error | Continue from that document instead of uploading again |
| Missing Claude Code tools | `/reload-plugins`; inspect `/plugin` errors |

OAuth limits operations to the workspace and permissions you approve. Revoke access in
Assinafy or disconnect the client. Signing links, verification codes and base64 stay out of chat.
[Privacy](https://www.assinafy.com.br/politica-de-privacidade) · [Terms](https://www.assinafy.com.br/termos-de-uso) · [Support](https://www.assinafy.com.br/contato)

[MIT license](LICENSE)
