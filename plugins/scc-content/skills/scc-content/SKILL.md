---
name: scc-content
description: Stafford Camera Club content and competition import workflows through the native SCC MCP and Wagtail fallback.
---

# SCC content

Use the native SCC MCP for news, programme and competition operations on
staffordcameraclub.co.uk. It runs as the separate `scc-mcp-1` service on the
Contabo host and calls the canonical Django/Wagtail domain logic.

## MetaMCP access

The MetaMCP hub exposes the native server in the `scc` namespace. Authentication
is configured in the connection. Never place bearer values in prompts, query
strings, filenames or tool arguments.

The competition tools added in SCC `v0.7.14` are:

- `scc__list_competitions` - list stable IDs and metadata, optionally by season.
- `scc__preview_competition_import` - validate an image plus CSV/RTF/XLSX import
  without database or storage writes.
- `scc__write_competition_import` - perform the same import idempotently after
  preview approval.

News and programme tools remain available for list/create/edit/publish flows.
They use Wagtail revisions; default to a draft unless Dan explicitly asks to
publish.

## Competition workflow

1. Work on one competition folder at a time. Prefer `Images/` plus `Results/`.
2. Match each image filename stem to the results `Title`, ignoring case and
   repeated whitespace. Preserve original titles in filenames.
3. Call `scc__list_competitions` for the season. If a matching page exists, put
   its ID in `payload.competition_id`; otherwise omit the ID.
4. Build the canonical payload with competition type, name, season, ISO date,
   source, round where required, and lifecycle `JUDGED`. External entries also
   require `host_club`.
5. Send each file as `{kind, filename, content_base64}`. Text CSV/RTF results may
   use `content_text`; images and XLSX use base64.
6. Call preview first. Stop on errors and review all warnings and
   `unknown_members`. Never guess an ambiguous member.
7. After approval, call write with exactly the same payload and files.
8. Verify the returned competition in Wagtail. Publishing is a separate human
   decision.

Supported images: JPG/JPEG, PNG, WebP, TIFF. Supported result formats: CSV/TSV
(UTF-8/UTF-16), RTF and XLSX. Use XLSX rather than old binary XLS files.

Repeated writes use `(competition, image_filename)` and do not duplicate
entries. Failed writes roll back database changes and clean files created by
that attempt. Preview reports images without result rows, result rows without
images, and unmatched members.

Full operator documentation lives in the SCC repository at
`docs/competition-agent-workflow.md`.

## Manual fallback

- **Wagtail > Upload competition** for a new competition import.
- **Wagtail > Competition entries** for existing entry edits, image replacement,
  bulk results, member matching and confirmed deletion.

## Source and deployment

| Item | Value |
|------|-------|
| Repository | `robinsondan87/SCC_V2` |
| Local checkout | `/Users/robbohomebot/Projects/scc-content/SCC_V2` |
| OneDrive source | `~/OneDrive/SCC/Competition/Entries/` |
| Website | `https://staffordcameraclub.co.uk` |
| MCP service | `scc-mcp-1`, port 8765, bearer plus firewall allow-list |

Every SCC release must bump the top `CHANGELOG.md` version. Merging to `main`
creates the tag, builds the image and deploys web plus MCP through the existing
release workflow.

Related skills: `scc_contabo:svr002`, `jira:jira`, `robbohome-memory`.
