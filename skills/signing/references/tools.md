# Assinafy MCP tool reference

Every tool exposed by `https://mcp.assinafy.com.br/mcp`, with the Assinafy
REST operation it calls, its arguments, a request and a response. Arguments are sent
as the MCP `tools/call` `arguments` object. Successful JSON results arrive both as
`structuredContent` and as the same JSON in a text block; responses below show that
JSON. IDs, names, emails and dates are fictional.

For every complete input and response schema, including optional fields and cost
previews, read [tool-payloads.json](tool-payloads.json). It matches MCP server v3.3.1;
the sections below provide the conversational flow and representative payloads.

`account_id` is optional everywhere it appears. The workspace comes from the OAuth
grant; a value that differs from the grant is rejected.

## Conventions

- **One tool per operation**; each accepts only the arguments listed in its section.
- **Timestamps** are RFC 3339 (`2026-12-31T23:59:59Z`).
- **Lists** return `{"data": [...], "meta": {...}}`; `meta` comes from Assinafy's
  pagination headers and is omitted when absent. Pages start at 1, `per_page` max 100.
- **Errors** return `isError: true` with one text block:
  - API failure: `API error 404: Página não encontrada.` (Assinafy messages are in
    Portuguese). A missing permission appends the challenge:
    `API error 403: ... (Bearer error="insufficient_scope", scope="documents:write")`.
  - Input rejected by the server: the bare message, for example `Document ID is required`.
  - Input rejected by the schema: `validating "arguments": validating root: required: missing properties: ["hash"]`.
  - Plan limits: `API error 403: A notificação via WhatsApp não está disponível para o seu plano atual.`
    or `API error 403: A assinatura com certificado digital não está disponível para o seu plano atual.`
- **Transport**: HTTP `401` with `WWW-Authenticate: Bearer resource_metadata=...` starts
  the OAuth sign-in; `403 insufficient_scope` asks for more scopes; `429` honours
  `Retry-After`; `503` means Assinafy authorization is temporarily unavailable.

## Cost previews

Set `estimate_only: true` on the existing PDF send, assignment, template or reminder
tool. Successful preview responses return `{cost_estimate: {...}}`. Check
`has_sufficient_resources`, `blocking_reason`, balances and `total_credits` before
sending. `blocking_reason` is null when no resource condition blocks the request.
Estimates reserve nothing; Assinafy rechecks eligibility and balances at send time.

| Tool | Preview behavior |
|---|---|
| `assinafy_send_pdf_for_signature` | Uploads and processes the PDF, returning `{document,cost_estimate}`; creates no contacts or invitations. Reuse `document.id` with `assinafy_create_assignment` to send. |
| `assinafy_create_assignment` | Needs `document_id`, `method` and signing choices; signer IDs are optional. Collect also needs `entries`, without reading page geometry. Creates no assignment. |
| `assinafy_create_document_from_template` | Needs `template_id` and one `role_id` per signer; no contact information required. Creates no contacts, documents or invitations. |
| `assinafy_resend_notification` | Needs the normal document, assignment and signer IDs. Sends no notification. |

Request example for an existing document:

```json
{"document_id":"1f0a2b3c4d5e6f708192a3b4c5d6","method":"virtual","signers":[{"verification_method":"Email"}],"estimate_only":true}
```

Response example:

```json
{"cost_estimate":{"documents":1,"credits":0,"needs_extra_document":false,"extra_document_cost":0,"total_credits":0,"breakdown":[{"code":"NotificationEmail","name":"Email Notification","cost":0,"quantity":1,"unit_cost":0}],"document_balance":10,"credit_balance":20,"has_sufficient_resources":true,"blocking_reason":null,"message":null}}
```

To send after a preview, supply real signer IDs or authorized template contacts
and omit `estimate_only` or set it false. Keep a prepared PDF's document ID;
rerunning the PDF upload would create another document. Review and placement
cards accept only sending arguments, without `estimate_only`.

## Shared objects

### Document

Returned by `assinafy_get_document`, `assinafy_list_documents` items,
`assinafy_upload_document`, `assinafy_rename_document` and
`assinafy_create_document_from_template`.

```json
{
  "resource": "document",
  "id": "1f0a2b3c4d5e6f708192a3b4c5d6",
  "account_id": "1a2b3c4d5e6f708192a3b4c5d6e7",
  "name": "Contrato de Servicos.pdf",
  "status": "pending_signature",
  "artifacts": {
    "original": "https://api.assinafy.com.br/v1/documents/1f0a2b3c4d5e6f708192a3b4c5d6/download/original",
    "thumbnail": "https://api.assinafy.com.br/v1/documents/1f0a2b3c4d5e6f708192a3b4c5d6/thumbnail"
  },
  "pages": [
    {
      "id": "2a0b1c2d3e4f5061728394a5b6c7",
      "number": 1,
      "height": 1651,
      "width": 1275,
      "download_url": "https://api.assinafy.com.br/v1/documents/1f0a2b3c4d5e6f708192a3b4c5d6/pages/2a0b1c2d3e4f5061728394a5b6c7/download"
    }
  ],
  "is_closed": false,
  "signing_url": "https://app.assinafy.com.br/sign/1f0a2b3c4d5e6f708192a3b4c5d6",
  "assignment": {
    "id": "3b0c1d2e3f405162738495a6b7c8",
    "sender_email": "sender@example.com",
    "method": "virtual",
    "expires_at": "2026-12-31T23:59:59Z",
    "message": "Please sign",
    "signers": [
      {
        "id": "4c0d1e2f30415263748596a7b8c9",
        "full_name": "Ana Example",
        "email": "ana@example.com",
        "has_accepted_terms": false,
        "verification_method": "Email",
        "notification_methods": ["Email"],
        "step": 1,
        "notified": true,
        "completed": false,
        "notification_history": [
          {
            "event": "signature_request",
            "status": "sent",
            "sent_at": "2026-10-10T13:42:38Z"
          }
        ]
      }
    ],
    "summary": {
      "signer_count": 1,
      "completed_count": 0,
      "signers": [
        {"id": "4c0d1e2f30415263748596a7b8c9", "full_name": "Ana Example", "email": "ana@example.com", "completed": false}
      ]
    },
    "signing_urls": [
      {"signer_id": "4c0d1e2f30415263748596a7b8c9", "url": "https://app.assinafy.com.br/sign/1f0a2b3c4d5e6f708192a3b4c5d6?email=ana%40example.com"}
    ]
  },
  "created_at": "2026-10-10T13:42:23Z",
  "updated_at": "2026-10-10T13:42:38Z"
}
```

- In `assinafy_get_document`, `assinafy_list_documents`,
  `assinafy_create_document_from_template` and `assinafy_set_assignment_expiration`
  results, empty or null fields are omitted (`assignment` until signatures are
  requested, `template_id` unless generated from a template, empty `tags`,
  `copy_receivers`, `items`). Signers preserve `government_id` and
  `identity_verification_status` when supplied; read the contact before a certificate
  or biometric request. `InReview` requires manual review in Assinafy.
  `assinafy_create_assignment` and `assinafy_rename_document` return Assinafy's
  response unchanged, including nulls, `government_id`, `items` and `signing_urls`.
- `artifacts` gains `certificated`, `certificate-page` and `bundle` (and `pades` where
  enabled) after certification. `is_closed` becomes `true`.
- For `collect` assignments, `items` lists each placement: `page`, `signer`, `field`,
  `display_settings`, `value`, `completed`.
- `decline_reason` and `declined_by` are set when a signer rejects.
- `signing_urls` and verification codes are sensitive. Do not use or show them.

### Statuses

| Status | Meaning | Deletable |
|---|---|:---:|
| `uploading`, `uploaded`, `metadata_processing` | Upload received, pages being processed | no |
| `metadata_ready` | Pages ready; can be prepared or sent | yes |
| `pending_signature` | Signature request sent | yes |
| `pending_identity_review` | All signed; account identity review required | no |
| `certificating` | All signed; certificate being applied | no |
| `certificated` | Signed and certified; artifacts available | no |
| `expired` | Assignment expired | yes |
| `rejected_by_signer`, `rejected_by_user` | Declined | yes |
| `failed` | Processing failed | yes |

### Signer in a request

Used by `assinafy_create_assignment`, `assinafy_review_signature_request` and `assinafy_place_fields`:

| Field | Required | Values |
|---|:---:|---|
| `id` | yes for sending | Workspace signer ID; optional for an assignment cost preview |
| `verification_method` | no | `Email` (default), `Whatsapp`, `DigitalCertificate`, `IdDocumentFaceMatchBasic` |
| `notification_methods` | no | Exactly one compatible channel: `Email` or `Whatsapp`; inferred when omitted |
| `step` | no | Signing order from 1, contiguous; set on every signer or none. Same step signs in parallel |

- Every request uses one document from the plan allowance, or 1 credit once exhausted.
- `Whatsapp` verification needs the WhatsApp channel and a paid plan; the WhatsApp
  notification costs 0.45 credits per signer.
- `DigitalCertificate` signs with the signer's ICP-Brasil A1 or A3 certificate. It needs
  the Digital Certificate feature, a CPF or CNPJ already registered on the signer's
  contact (`government_id` in `assinafy_get_signer`, saved through an authorized
  create/update contact call or in Assinafy), and the signer alone in its step, so give every signer a
  `step` when others sign too. 0.5 credits per signer plus its notification channel.
  A CPF needs that person's e-CPF (or an e-CNPJ naming them as legal representative);
  a CNPJ needs that company's e-CNPJ.
- `IdDocumentFaceMatchBasic` requires its account feature and a valid CPF in
  `government_id`. The signer provides ID photos and liveness in Assinafy. Use
  `estimate_only` for its per-signer cost. `InReview` is resolved by the account
  in Assinafy; `pending_identity_review` holds certification until approval.
- Email verification pairs with Email notification, Whatsapp with Whatsapp;
  certificate and biometric verification accept either channel, one per signer.

### Collect entries

Required for `method: "collect"`. One object per page:

```json
{
  "page_id": "2a0b1c2d3e4f5061728394a5b6c7",
  "fields": [
    {
      "signer_id": "4c0d1e2f30415263748596a7b8c9",
      "field_id": "5d0e1f2031425364758697a8b9ca",
      "display_settings": {"left": 386, "top": 1400, "width": 421, "height": 46, "fontSize": 22, "fontFamily": "Arial", "backgroundColor": "#d7f4e0"}
    }
  ]
}
```

Geometry is in pixels of Assinafy's 150-DPI page image, from the top-left corner; keep
each rectangle inside the page `width` and `height` from `assinafy_get_document`.
`field_id` comes from `assinafy_list_fields` with `include_standard: true` (types
`signature`, `initial`, `signatureDate`, plus input fields).

---

## Find and inspect documents

### `assinafy_list_documents`

API: `GET /v1/accounts/{accountId}/documents`

Required: none
Optional: `page`, `per_page`, `search`, `sort`, `status`, `account_id`, `method`, `tags`

`search` matches document name or signer. `sort` is `name` or `updated_at`, prefixed
with `-` for descending. `status` takes one status code. `method` filters `virtual`
or `collect`; `tags` is a comma-separated list of tag IDs that must all match.

Request:

```json
{"status": "pending_signature", "search": "Contrato", "sort": "-updated_at", "per_page": 20}
```

Response:

```json
{
  "data": [
    {
      "id": "1f0a2b3c4d5e6f708192a3b4c5d6",
      "account_id": "1a2b3c4d5e6f708192a3b4c5d6e7",
      "name": "Contrato de Servicos.pdf",
      "status": "pending_signature",
      "artifacts": {"original": "https://api.assinafy.com.br/v1/documents/1f0a2b3c4d5e6f708192a3b4c5d6/download/original"},
      "signing_url": "https://app.assinafy.com.br/sign/1f0a2b3c4d5e6f708192a3b4c5d6",
      "created_at": "2026-10-10T13:42:23Z",
      "updated_at": "2026-10-10T13:42:38Z"
    }
  ],
  "meta": {"current_page": 1, "last_page": 1, "per_page": 20, "total": 1}
}
```

Each item has the [Document](#document) fields except `resource`; `is_closed` appears only when `true`.

### `assinafy_get_document`

API: `GET /v1/documents/{documentId}`

Required: `document_id`
Optional: none

Request:

```json
{"document_id": "1f0a2b3c4d5e6f708192a3b4c5d6"}
```

Response: a [Document](#document).

### `assinafy_list_document_activities`

API: `GET /v1/documents/{documentId}/activities`

Required: `document_id`
Optional: none

Request:

```json
{"document_id": "1f0a2b3c4d5e6f708192a3b4c5d6"}
```

Response (newest first; `payload` varies by event, `origin` is `null` for system events):

```json
{
  "data": [
    {
      "id": 37854,
      "event": "signature_requested",
      "message": "Solicitação para assinar enviada para Ana Example <ana@example.com>.",
      "payload": {"signer_email": "ana@example.com", "signer_full_name": "Ana Example", "notification_method": "email", "signer_whatsapp_phone_number": null},
      "origin": null,
      "created_at": "2026-10-10T13:42:38Z"
    },
    {
      "id": 37849,
      "event": "document_uploaded",
      "message": "Documento criado.",
      "payload": [],
      "origin": {"ip": "203.0.113.10", "user-agent": "Assinafy-MCP"},
      "created_at": "2026-10-10T13:42:23Z"
    }
  ]
}
```

### `assinafy_list_whatsapp_notifications`

API: `GET /v1/documents/{documentId}/assignments/{assignmentId}/whatsapp-notifications`

Required: `document_id`, `assignment_id`
Optional: none

WhatsApp delivery history only; email history is in each assignment signer's
`notification_history`.

Request:

```json
{"document_id": "1f0a2b3c4d5e6f708192a3b4c5d6", "assignment_id": "3b0c1d2e3f405162738495a6b7c8"}
```

Response (`sent_at` is a Unix timestamp; in sandbox, messages are simulated):

```json
{
  "data": [
    {
      "signer_id": "6e0f102132435465768798a9bacb",
      "phone_number": "+5511900000000",
      "sent_at": 1791639758,
      "header": "Documento para assinatura: Contrato de Servicos.pdf",
      "body": "Olá, Bruno Example. Você recebeu um documento para assinar.",
      "buttons": [{"text": "Abrir documento"}]
    }
  ]
}
```

An assignment without WhatsApp notifications returns `{"data": []}`. Button URLs may
carry signer access codes; never show or use them.

### `assinafy_show_document`

API: `GET /v1/documents/{documentId}`

Required: `document_id`
Optional: none

Hosts that support MCP Apps render a status card with **Remind** and **Refresh**; other
hosts show the text summary.

Request:

```json
{"document_id": "1f0a2b3c4d5e6f708192a3b4c5d6"}
```

Response text: `Contrato de Servicos.pdf: pending_signature. 0 of 1 signed. Waiting for Ana Example. Expires 2026-12-31T23:59:59Z.`

Response:

```json
{
  "view": "status",
  "document_id": "1f0a2b3c4d5e6f708192a3b4c5d6",
  "name": "Contrato de Servicos.pdf",
  "status": "pending_signature",
  "assignment_id": "3b0c1d2e3f405162738495a6b7c8",
  "expires_at": "2026-12-31T23:59:59Z",
  "signed": 0,
  "signers": [{"id": "4c0d1e2f30415263748596a7b8c9", "name": "Ana Example", "signed": false, "step": 1}]
}
```

---

## Prepare a document

### `assinafy_upload_document`

API: `POST /v1/accounts/{accountId}/documents`

Required: `file_name`, `file_base64`
Optional: `account_id`

Uploads without inviting anyone. `file_name` ends in `.pdf`; the decoded PDF is at most
25 MiB and must contain `%PDF-` within its first 1 KiB.

Request:

```json
{"file_name": "Contrato de Servicos.pdf", "file_base64": "<BASE64_PDF>"}
```

Response: a [Document](#document) with `status` `uploaded` and no `assignment`. It
reaches `metadata_ready` within seconds; check with `assinafy_get_document`.

### `assinafy_rename_document`

API: `PATCH /v1/documents/{documentId}`

Required: `document_id`, `name`
Optional: none

Allowed before an assignment exists. `name` is 1–255 characters; Assinafy removes
diacritics and replaces unsupported characters with dashes.

Request:

```json
{"document_id": "1f0a2b3c4d5e6f708192a3b4c5d6", "name": "Contrato de Servicos v2.pdf"}
```

Response: the updated [Document](#document).

---

## Request signatures

### `assinafy_send_pdf_for_signature`

API: `POST /v1/accounts/{accountId}/documents`, `GET /v1/documents/{documentId}`,
`GET /v1/accounts/{accountId}/signers`, `POST /v1/accounts/{accountId}/signers`,
`POST /v1/documents/{documentId}/assignments`
With `estimate_only: true`, also uses `POST /v1/documents/{documentId}/assignments/estimate-cost`; see [cost previews](#cost-previews).

Required: `file_name`, `file_base64`, `signers`
Optional: `message`, `expires_at`, `max_wait_secs`, `poll_secs`, `account_id`, `estimate_only`

Uploads, waits for processing, creates or reuses each signer by exact email, and sends
a `virtual` request with Email verification and Email notification. `signers` items
are `{full_name, email}`; duplicate emails are rejected. `max_wait_secs` is 1–600
(default 30); `poll_secs` defaults to 2.

Request:

```json
{
  "file_name": "Contrato de Servicos.pdf",
  "file_base64": "<BASE64_PDF>",
  "signers": [
    {"full_name": "Ana Example", "email": "ana@example.com"},
    {"full_name": "Bruno Example", "email": "bruno@example.com"}
  ],
  "message": "Please review and sign",
  "expires_at": "2026-12-31T23:59:59Z"
}
```

Response:

```json
{
  "document": {
    "id": "1f0a2b3c4d5e6f708192a3b4c5d6",
    "account_id": "1a2b3c4d5e6f708192a3b4c5d6e7",
    "name": "Contrato de Servicos.pdf",
    "status": "uploaded",
    "artifacts": {"original": "https://api.assinafy.com.br/v1/documents/1f0a2b3c4d5e6f708192a3b4c5d6/download/original"},
    "pages": [],
    "is_closed": false,
    "created_at": "2026-10-10T13:42:23Z",
    "updated_at": "2026-10-10T13:42:23Z"
  },
  "assignment": {
    "id": "3b0c1d2e3f405162738495a6b7c8",
    "method": "virtual",
    "expires_at": "2026-12-31T23:59:59Z",
    "message": "Please review and sign",
    "signers": [
      {"id": "4c0d1e2f30415263748596a7b8c9", "full_name": "Ana Example", "email": "ana@example.com", "verification_method": "Email", "notification_methods": ["Email"], "step": 1, "notified": true, "completed": false},
      {"id": "6e0f102132435465768798a9bacb", "full_name": "Bruno Example", "email": "bruno@example.com", "verification_method": "Email", "notification_methods": ["Email"], "step": 1, "notified": true, "completed": false}
    ],
    "summary": {"signer_count": 2, "completed_count": 0}
  },
  "signer_ids": ["4c0d1e2f30415263748596a7b8c9", "6e0f102132435465768798a9bacb"]
}
```

`document` is the upload response, so its `status` and `pages` are from upload time; call
`assinafy_get_document` for current status and pages. A failure after upload names the retained `document_id`; continue with `assinafy_create_assignment`.
If the access token expires within `max_wait_secs` plus two minutes, nothing is uploaded
and the error says how many seconds remain.

### `assinafy_create_assignment`

API: `POST /v1/documents/{documentId}/assignments`
With `estimate_only: true`, also uses `POST /v1/documents/{documentId}/assignments/estimate-cost`; see [cost previews](#cost-previews).

Required: `document_id`, `method`, `signers`
Optional: `entries`, `message`, `expires_at`, `copy_receivers`, `estimate_only`

`method` is `virtual` (sign the whole document) or `collect` (sign placed fields; needs
`metadata_ready` and [entries](#collect-entries)). `signers` uses
[Signer in a request](#signer-in-a-request). `copy_receivers` is a list of signer IDs
that receive the signed copy without signing. Inspect the document first; if it already
has an `assignment`, follow up instead.

Request (signing order, WhatsApp, certificate and a copy receiver):

```json
{
  "document_id": "1f0a2b3c4d5e6f708192a3b4c5d6",
  "method": "virtual",
  "signers": [
    {"id": "4c0d1e2f30415263748596a7b8c9", "verification_method": "Email", "notification_methods": ["Email"], "step": 1},
    {"id": "6e0f102132435465768798a9bacb", "verification_method": "Whatsapp", "notification_methods": ["Whatsapp"], "step": 2},
    {"id": "7f10213243546576879809bacbdc", "verification_method": "DigitalCertificate", "notification_methods": ["Email"], "step": 3}
  ],
  "copy_receivers": ["80213243546576879809a1cbdced"],
  "message": "Please sign in order",
  "expires_at": "2026-12-31T23:59:59Z"
}
```

Response: Assinafy's assignment, unchanged (abridged here). It also contains `items` and
`signing_urls`; signing URLs are sensitive, never show or use them.

```json
{
  "resource": "assignment",
  "id": "3b0c1d2e3f405162738495a6b7c8",
  "sender_email": "sender@example.com",
  "method": "virtual",
  "expires_at": "2026-12-31T23:59:59Z",
  "message": "Please sign in order",
  "signers": [
    {"id": "4c0d1e2f30415263748596a7b8c9", "full_name": "Ana Example", "email": "ana@example.com", "verification_method": "Email", "notification_methods": ["Email"], "step": 1, "notified": true, "completed": false},
    {"id": "6e0f102132435465768798a9bacb", "full_name": "Bruno Example", "email": "bruno@example.com", "whatsapp_phone_number": "+5511900000000", "verification_method": "Whatsapp", "notification_methods": ["Whatsapp"], "step": 2, "notified": false, "completed": false},
    {"id": "7f10213243546576879809bacbdc", "full_name": "Carla Example", "email": "carla@example.com", "verification_method": "DigitalCertificate", "notification_methods": ["Email"], "step": 3, "notified": false, "completed": false}
  ],
  "copy_receivers": [{"id": "80213243546576879809a1cbdced", "full_name": "Finance Example", "email": "finance@example.com"}],
  "summary": {"signer_count": 3, "completed_count": 0}
}
```

Later steps are notified when the previous step completes.

### `assinafy_create_document_from_template`

API: `POST /v1/accounts/{accountId}/templates/{templateId}/documents`
(plus signer lookup/creation for entries given by email)
With `estimate_only: true`, also uses `POST /v1/accounts/{accountId}/templates/{templateId}/documents/estimate-cost`; see [cost previews](#cost-previews).

Required: `template_id`, `signers`
Optional: `name`, `message`, `expires_at`, `tags`, `editor_fields`, `account_id`, `estimate_only`

One `signers` entry per template role:

| Field | Required | Values |
|---|:---:|---|
| `role_id` | yes | Role from `assinafy_list_templates` |
| `id` | one of | Existing signer ID |
| `full_name`, `email` | one of | New or reused contact (exact email match is reused) |
| `verification_method` | no | `Email`, `Whatsapp`, `DigitalCertificate`, `IdDocumentFaceMatchBasic` |
| `notification_methods` | no | At most one: `Email` or `Whatsapp` |
| `step` | no | Contiguous from 1; all or none |

`editor_fields` is `[{field_id, value}]` with the placement's `field_id` and a string
value (validate first with `assinafy_validate_fields`). `tags` are tag names merged with
the template's defaults.

Request:

```json
{
  "template_id": "91324354657687980a1b2cdcedfe",
  "name": "NDA - Acme Ltda",
  "signers": [
    {"role_id": "a2435465768798091b2c3dedfe0f", "full_name": "Carla Example", "email": "carla@example.com"},
    {"role_id": "b3546576879809a12c3d4efe0f10", "id": "4c0d1e2f30415263748596a7b8c9", "verification_method": "Whatsapp", "notification_methods": ["Whatsapp"]}
  ],
  "editor_fields": [{"field_id": "c46576879809a1b23d4e5f0f1021", "value": "Acme Ltda"}],
  "tags": ["nda"],
  "message": "Please sign the NDA"
}
```

Response: the generated [Document](#document) with `template_id` set and its
`assignment`.

### `assinafy_review_signature_request`

API: `GET /v1/documents/{documentId}`, `GET /v1/accounts/{accountId}/signers/{signerId}`

Required: `document_id`, `method`, `signers`
Optional: `entries`, `message`, `expires_at`, `copy_receivers`

The sending arguments of `assinafy_create_assignment`, without `estimate_only`; sends nothing. MCP Apps hosts show a card whose
**Send** button rechecks the document and sends exactly the reviewed request.

Request:

```json
{"document_id": "1f0a2b3c4d5e6f708192a3b4c5d6", "method": "virtual", "signers": [{"id": "4c0d1e2f30415263748596a7b8c9"}], "expires_at": "2026-12-31T23:59:59Z"}
```

Response text: `Ready to send Contrato de Servicos.pdf (virtual) to Ana Example <ana@example.com>. Expires 2026-12-31T23:59:59Z. Nothing has been sent: ...`

Response:

```json
{
  "view": "review",
  "document_id": "1f0a2b3c4d5e6f708192a3b4c5d6",
  "name": "Contrato de Servicos.pdf",
  "status": "metadata_ready",
  "already_sent": false,
  "method": "virtual",
  "expires_at": "2026-12-31T23:59:59Z",
  "signers": [{"id": "4c0d1e2f30415263748596a7b8c9", "name": "Ana Example", "email": "ana@example.com"}],
  "request": {"document_id": "1f0a2b3c4d5e6f708192a3b4c5d6", "method": "virtual", "signers": [{"id": "4c0d1e2f30415263748596a7b8c9"}], "expires_at": "2026-12-31T23:59:59Z"}
}
```

`verification` is shown only for signers whose request sets `verification_method`.
When `already_sent` is `true`, the document already has a request; do not send again.

### `assinafy_place_fields`

API: `GET /v1/documents/{documentId}`, `GET /v1/accounts/{accountId}/signers/{signerId}`,
`GET /v1/accounts/{accountId}/fields`

Required: `document_id`, `signers`
Optional: `message`, `expires_at`, `copy_receivers`

`assinafy_create_assignment` arguments without `method`, `entries` and `estimate_only`; the document must be
`metadata_ready` with no assignment. MCP Apps hosts open a fullscreen editor that sends
with `method: "collect"`. Other clients get pages and field IDs to build
[entries](#collect-entries).

Request:

```json
{"document_id": "1f0a2b3c4d5e6f708192a3b4c5d6", "signers": [{"id": "4c0d1e2f30415263748596a7b8c9"}]}
```

Response text: `Contrato de Servicos.pdf has 1 pages ready for field placement. ... pages: 1 id 2a0b1c2d3e4f5061728394a5b6c7 1275x1651. Fields: Assinatura (signature, field_id 5d0e1f2031425364758697a8b9ca); ... Nothing has been sent.`

---

## Follow up

### `assinafy_resend_notification`

API: `PUT /v1/documents/{documentId}/assignments/{assignmentId}/signers/{signerId}/resend`
With `estimate_only: true`, also uses `POST /v1/documents/{documentId}/assignments/{assignmentId}/signers/{signerId}/estimate-resend-cost`; see [cost previews](#cost-previews).

Required: `document_id`, `assignment_id`, `signer_id`
Optional: `estimate_only`

Sends one reminder on the signer's channels; may consume notification credits.

Request:

```json
{"document_id": "1f0a2b3c4d5e6f708192a3b4c5d6", "assignment_id": "3b0c1d2e3f405162738495a6b7c8", "signer_id": "4c0d1e2f30415263748596a7b8c9"}
```

Response (`is_sent` means delivered to the channel, not signed):

```json
{"is_sent": true, "document_id": "1f0a2b3c4d5e6f708192a3b4c5d6", "signer_id": "4c0d1e2f30415263748596a7b8c9"}
```

### `assinafy_set_assignment_expiration`

API: `PUT /v1/documents/{documentId}/assignments/{assignmentId}/reset-expiration`

Required: `document_id`, `assignment_id`, `expires_at`
Optional: none

Request:

```json
{"document_id": "1f0a2b3c4d5e6f708192a3b4c5d6", "assignment_id": "3b0c1d2e3f405162738495a6b7c8", "expires_at": "2026-12-31T23:59:59Z"}
```

Response: the updated assignment (as in `assinafy_create_assignment`, omitting null or
empty optional fields) with the new
`expires_at`. No reminder is sent.

---

## Contacts

### `assinafy_list_signers`

API: `GET /v1/accounts/{accountId}/signers`

Required: none
Optional: `search`, `page`, `per_page`, `account_id`

Request:

```json
{"search": "ana@example.com"}
```

Response:

```json
{
  "data": [
    {"id": "4c0d1e2f30415263748596a7b8c9", "full_name": "Ana Example", "email": "ana@example.com", "whatsapp_phone_number": null, "government_id": null, "has_accepted_terms": false}
  ],
  "meta": {"current_page": 1, "last_page": 1, "per_page": 20, "total": 1}
}
```

### `assinafy_get_signer`

API: `GET /v1/accounts/{accountId}/signers/{signerId}`

Required: `signer_id`
Optional: `account_id`

Request:

```json
{"signer_id": "4c0d1e2f30415263748596a7b8c9"}
```

Response:

```json
{"resource": "signer", "id": "4c0d1e2f30415263748596a7b8c9", "full_name": "Ana Example", "email": "ana@example.com", "whatsapp_phone_number": null, "government_id": null, "has_accepted_terms": false}
```

### `assinafy_create_signer`

API: `POST /v1/accounts/{accountId}/signers`

Required: `full_name`
Optional: `email`, `whatsapp_phone_number`, `account_id`, `government_id`

Sends no invitation. Search first to avoid duplicate contacts; Assinafy may reject an
email that already exists in the workspace.
`whatsapp_phone_number` is E.164. `government_id` accepts an authorized CPF or CNPJ,
normalized and validated by Assinafy.

Request:

```json
{"full_name": "Bruno Example", "email": "bruno@example.com", "whatsapp_phone_number": "+5511900000000"}
```

Response:

```json
{"resource": "signer", "id": "6e0f102132435465768798a9bacb", "full_name": "Bruno Example", "email": "bruno@example.com", "whatsapp_phone_number": "+5511900000000", "government_id": null, "has_accepted_terms": false}
```

### `assinafy_update_signer`

API: `PUT /v1/accounts/{accountId}/signers/{signerId}`

Required: `signer_id` and at least one of `full_name`, `email`, `whatsapp_phone_number`, `government_id`
Optional: `full_name`, `email`, `whatsapp_phone_number`, `account_id`, `government_id`

`government_id` saves an authorized CPF or CNPJ on this contact. The API normalizes
and validates it. Use a valid CPF before biometric verification, or CPF/CNPJ before
certificate signing; handle the value as sensitive contact data.
`email` and `whatsapp_phone_number` cannot change while that channel is verified on an
in-flight document (`400` naming the documents). Changing an unverified channel
invalidates earlier links and codes; resend only after the correction succeeds.

Request:

```json
{"signer_id": "7f10213243546576879809bacbdc", "whatsapp_phone_number": "+5511911111111"}
```

Response:

```json
{"resource": "signer", "id": "7f10213243546576879809bacbdc", "full_name": "Carla Example", "email": "carla@example.com", "whatsapp_phone_number": "+5511911111111", "government_id": null, "has_accepted_terms": false}
```

---

## Templates and fields

### `assinafy_list_templates`

API: `GET /v1/accounts/{accountId}/templates`

Required: none
Optional: `search`, `page`, `per_page`, `account_id`

Request:

```json
{"search": "NDA"}
```

Response:

```json
{
  "data": [
    {
      "id": "91324354657687980a1b2cdcedfe",
      "name": "NDA",
      "document_name": "nda.pdf",
      "status": "Ready",
      "pages": [
        {
          "id": "d576879809a1b2c34e5f60102132",
          "number": 1,
          "height": 1755,
          "width": 1240,
          "fields": [
            {
              "id": "e6879809a1b2c3d45f6071021324",
              "field_id": "5d0e1f2031425364758697a8b9ca",
              "role_id": "a2435465768798091b2c3dedfe0f",
              "label": "Assinatura",
              "display_settings": {"top": 562, "left": 386, "width": 421, "height": 46, "fontSize": 22, "fontFamily": "Arial", "backgroundColor": "#d7f4e0"}
            }
          ]
        }
      ],
      "roles": [
        {"id": "a2435465768798091b2c3dedfe0f", "name": "Contratante", "assignment_type": "Signer"},
        {"id": "b3546576879809a12c3d4efe0f10", "name": "Testemunha", "assignment_type": "Signer"}
      ],
      "created_at": "2026-10-02T21:03:39Z",
      "updated_at": "2026-10-02T21:10:08Z"
    }
  ]
}
```

Roles with `assignment_type` `Signer` need a `assinafy_create_document_from_template` entry. Placement `field_id`
values are what `editor_fields` uses.

### `assinafy_get_template`

API: `GET /v1/accounts/{accountId}/templates/{templateId}` (compatibility route, not in
the public OpenAPI document; prefer `assinafy_list_templates`)

Required: `template_id`
Optional: `account_id`

Request:

```json
{"template_id": "91324354657687980a1b2cdcedfe"}
```

Response: one template object, as in `assinafy_list_templates`.

### `assinafy_list_fields`

API: `GET /v1/accounts/{accountId}/fields`

Required: none
Optional: `include_standard`, `include_inactive`, `account_id`

Request:

```json
{"include_standard": true}
```

Response:

```json
{
  "data": [
    {"id": "5d0e1f2031425364758697a8b9ca", "name": "Assinatura", "type": "signature", "regex": null, "is_pre_defined": true, "is_active": true, "is_required": true, "is_standard": true, "is_read_only": false, "is_visible": true},
    {"id": "f798a9bacbdcedfe0f1021324354", "name": "Data de Assinatura", "type": "signatureDate", "regex": null, "is_pre_defined": true, "is_active": true, "is_required": true, "is_standard": true, "is_read_only": true, "is_visible": true},
    {"id": "08a9bacbdcedfe0f102132435465", "name": "CNPJ", "type": "cnpj", "regex": null, "is_pre_defined": true, "is_active": true, "is_required": false, "is_standard": false, "is_read_only": false, "is_visible": true}
  ]
}
```

### `assinafy_get_field`

API: `GET /v1/accounts/{accountId}/fields/{fieldId}`

Required: `field_id`
Optional: `account_id`

Request:

```json
{"field_id": "08a9bacbdcedfe0f102132435465"}
```

Response: one field definition, as in `assinafy_list_fields`.

### `assinafy_validate_fields`

API: `POST /v1/accounts/{accountId}/fields/validate-multiple`

Required: `values`
Optional: `account_id`

`values` is `[{field_id, value}]`. Creates nothing. Keep identifiers with leading zeros
as strings.

Request:

```json
{"values": [{"field_id": "08a9bacbdcedfe0f102132435465", "value": "11.222.333/0001-81"}, {"field_id": "19bacbdcedfe0f10213243546576", "value": "not-an-email"}]}
```

Response:

```json
{
  "data": [
    {"field_id": "08a9bacbdcedfe0f102132435465", "type": "cnpj", "success": true, "error_message": ""},
    {"field_id": "19bacbdcedfe0f10213243546576", "type": "email", "success": false, "error_message": "\"o valor de entrada\" não é um endereço de e-mail válido."}
  ]
}
```

---

## Download, verify, delete

### `assinafy_download_document`

API: `GET /v1/documents/{documentId}/download/{artifactName}`

Required: `document_id`
Optional: `artifact`

`artifact` is `original` (default), `certificated`, `certificate-page`, `pades` or
`bundle`. Download `certificated` only when the status is `certificated` and the
document's `artifacts` lists it. Binary limit 64 MiB.

Request:

```json
{"document_id": "1f0a2b3c4d5e6f708192a3b4c5d6", "artifact": "certificated"}
```

Response:

```json
{"document_id": "1f0a2b3c4d5e6f708192a3b4c5d6", "artifact": "certificated", "base64": "<BASE64_PDF>"}
```

### `assinafy_download_document_page`

API: `GET /v1/documents/{documentId}/pages/{pageId}/download`

Required: `document_id`, `page_id`
Optional: none

Request:

```json
{"document_id": "1f0a2b3c4d5e6f708192a3b4c5d6", "page_id": "2a0b1c2d3e4f5061728394a5b6c7"}
```

Response:

```json
{"document_id": "1f0a2b3c4d5e6f708192a3b4c5d6", "page_id": "2a0b1c2d3e4f5061728394a5b6c7", "base64": "<BASE64_IMAGE>"}
```

### `assinafy_verify_document`

API: `GET /v1/documents/{documentSignatureHash}/verify` (public; no sign-in needed)

Required: `hash`
Optional: none

`hash` is the SHA-1 signature hash printed on the signed document.

Request:

```json
{"hash": "FE32EDDADE7CBDDCBB934E7402047450B0E59C02"}
```

Response (valid):

```json
{
  "id": "1f0a2b3c4d5e6f708192a3b4c5d6",
  "hash": "FE32EDDADE7CBDDCBB934E7402047450B0E59C02",
  "is_valid": true,
  "status": "certificated",
  "agreement_code": "550E8400-E29B-41D4-A716-446655440000",
  "page_count": "1",
  "signer_count": "2",
  "completed_count": 2,
  "completed_at": "2026-10-11T09:15:00Z",
  "verified_at": "2026-10-12T10:00:00Z",
  "message": ""
}
```

Response (unknown or unsigned; always HTTP 200):

```json
{"id": null, "hash": "0000000000000000000000000000000000000000", "is_valid": false, "status": null, "agreement_code": null, "signer_count": null, "completed_count": null, "page_count": null, "completed_at": null, "verified_at": "2026-10-10T13:41:52Z", "message": "Documento não assinado ou não encontrado."}
```

### `assinafy_delete_document`

API: `DELETE /v1/documents/{documentId}`

Required: `document_id`
Optional: none

Only on the user's explicit request, and only in a deletable [status](#statuses).

Request:

```json
{"document_id": "1f0a2b3c4d5e6f708192a3b4c5d6"}
```

Response text: `Document deleted successfully`
