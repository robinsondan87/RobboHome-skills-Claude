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

The announcement tools added in SCC `v0.7.42` are:

- `scc__list_announcements` - list current announcements and publication state.
- `scc__create_announcement` - create a manual announcement; defaults to draft.
- `scc__list_announcement_email_imports` - inspect mailbox import history.
- `scc__retry_announcement_email_import` - retry a failed import by record ID.

The news mailbox tools added in SCC `v0.7.44` are:

- `scc__list_news_email_imports` - inspect public-news mailbox import history.
- `scc__retry_news_email_import` - retry a failed import by record ID.

## Announcement email workflow

Purelymail mailbox `announcements@staffordcameraclub.co.uk` is polled by the
production Compose service `announcement_mail` every 120 seconds. The password
is stored in SOPS as `SCC_ANNOUNCEMENT_IMAP_PASSWORD`; never print it or put it
in a command argument.

Trusted senders are `pamelaannbennett3@gmail.com` and
`robbohomebot@gmail.com`. A message is accepted only when its real `From`
address is allow-listed and Purelymail's topmost `Authentication-Results`
reports DKIM or DMARC passing. Accepted mail becomes an immediately published,
members-only announcement. The site does not rebroadcast it by email. The
sender receives a success or failure receipt.

Attachments are Wagtail Documents in the login-restricted `Member announcement
documents` collection. Signed-out `/documents/...` requests redirect to login,
and host nginx blocks direct `/media/documents/...` access with 404. Do not
weaken either control.

Processed mail moves to `Processed`; rejected/untrusted mail moves to
`Needs attention`; failed mail remains in INBOX for correction or MCP retry.
Inspect the worker with:

```bash
ssh scc_contabo 'cd /opt/stacks/scc && docker compose -p scc -f docker-compose.prod.yml ps announcement_mail && docker compose -p scc -f docker-compose.prod.yml logs --tail=100 announcement_mail'
```

## News email workflow

SCC `v0.7.44` deployed the `news_mail` worker for
`news@staffordcameraclub.co.uk`, but it remains disabled until the mailbox
password is supplied and Jira `SCC-72` is completed. Do not enable it or invent
credentials.

Once activated, it applies the same sender allow-list and Purelymail
DKIM/DMARC checks as the announcement worker. The subject becomes the public
news title, safe email formatting becomes the body, the sender display name is
reused as the author, and the email date becomes the post date. The first image
is the featured image, further images form the gallery, and non-image
attachments are public Wagtail document links. Processed, rejected, failed,
receipt and duplicate handling match the announcement workflow.

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

Competition images are permanent records. Never delete a Wagtail image object
or its storage file after it has been attached to a competition, including when
an entry, competition, season or member changes. SCC v0.7.15 protects referenced
images at the database relationship. Internal team-battle images are owned by
the team and intentionally have no individual member link.

Audit live member links without writing:

```bash
ssh scc_contabo 'cd /opt/stacks/scc && docker compose -p scc -f docker-compose.prod.yml exec -T web python manage.py audit_competition_member_links'
```

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
| Announcement worker | `scc-announcement_mail-1`, Purelymail IMAP |
| News worker | `scc-news_mail-1`, deployed disabled pending `SCC-72` |

Every SCC release must bump the top `CHANGELOG.md` version. Merging to `main`
creates the tag, builds the image and deploys web plus MCP through the existing
release workflow.

Related skills: `scc_contabo:svr002`, `jira:jira`, `robbohome-memory`.
