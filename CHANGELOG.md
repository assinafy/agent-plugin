# Changelog

All notable changes to this plugin are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and versions follow
[Semantic Versioning](https://semver.org/).

## [1.2.2] - 2026-10-19

### Changed

- Digital-certificate signers now require virtual assignments. Collect assignments
  and templates with placed fields are rejected before cost estimation or sending
  with `digital_certificate_requires_virtual`.

## [1.2.1] - 2026-10-16

### Changed

- Documented the one-step PDF flow's 10-signer limit and bounded processing time.
- Clarified that template requests use inline contacts only for Email signing and
  validate signing order before creating contacts.
- Binary downloads now keep base64 only in structured tool results instead of
  duplicating it in text.

## [1.2.0] - 2026-10-16

### Added

- Signing and reminder cost previews through `estimate_only` on the existing PDF,
  assignment, template and reminder tools, with balances and blocking reasons.
- Biometric ID and liveness verification, authorized government-ID contact updates,
  and identity-review status guidance in the signing flow.
- Complete request/response schemas and examples for all 27 tools.

### Changed

- Each signer uses one compatible Email or WhatsApp notification channel.
- Plugin listings and both language guides include the supported signing methods
  and cost previews.

## [1.1.0] - 2026-10-13

### Changed

- Each task now has its own tool with a descriptive name, 27 in all: for example
  `assinafy_list_documents`, `assinafy_get_document`, `assinafy_upload_document`,
  `assinafy_send_pdf_for_signature`, `assinafy_create_assignment`,
  `assinafy_create_document_from_template`, `assinafy_resend_notification` and
  `assinafy_update_signer`. Tools no longer take an `action` argument. The skill,
  tool reference and READMEs use the new names.
- The assistant no longer asks for or sends a CPF or CNPJ. For ICP-Brasil digital
  certificate signing, the signer's CPF or CNPJ is managed on the contact in the
  Assinafy app; the assistant checks that it is present before sending.

## [1.0.0] - 2026-10-10

### Added

- Plugin and marketplace `assinafy` for Claude Code, the Claude apps and Cowork, OpenAI
  Codex and ChatGPT, and VS Code with GitHub Copilot, connecting to the hosted Assinafy
  MCP server at `https://mcp.assinafy.com.br/mcp` over Streamable HTTP.
- Portable Agent Plugins manifest (`plugin.json`, `mcp.json`) with ChatGPT listing
  details, review test cases and a Portuguese translation.
- OAuth 2.1 sign-in with PKCE and client metadata documents: choose a workspace
  and approve `account:read`, `documents:read`, `documents:write`, `templates:read`,
  `templates:write` and `offline_access`. No API key or client secret.
- 14 tools covering the signing flow: find, prepare, request signatures from a PDF,
  an existing document or a saved template, review before sending, place fields,
  follow up, manage signers, check fields, download, verify and delete.
- Signature verification by Email, WhatsApp and ICP-Brasil digital certificate
  (A1/A3), with Email and WhatsApp notifications, signing order and copy receivers.
- `signing` skill with the end-to-end flow, channel requirements and costs,
  document statuses and follow-up rules, plus a reference with request and response
  payloads for every tool and action.
- README in Portuguese and English covering installation, sign-in, the complete
  signing flow, signer experience, troubleshooting and other MCP clients.

[1.2.2]: https://github.com/assinafy/agent-plugin/releases/tag/assinafy--v1.2.2
[1.2.1]: https://github.com/assinafy/agent-plugin/releases/tag/assinafy--v1.2.1
[1.2.0]: https://github.com/assinafy/agent-plugin/releases/tag/assinafy--v1.2.0
[1.1.0]: https://github.com/assinafy/agent-plugin/releases/tag/assinafy--v1.1.0
[1.0.0]: https://github.com/assinafy/agent-plugin/releases/tag/assinafy--v1.0.0
