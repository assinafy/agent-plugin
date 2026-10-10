# Changelog

All notable changes to this plugin are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and versions follow
[Semantic Versioning](https://semver.org/).

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

[1.0.0]: https://github.com/assinafy/agent-plugin/releases/tag/assinafy--v1.0.0
