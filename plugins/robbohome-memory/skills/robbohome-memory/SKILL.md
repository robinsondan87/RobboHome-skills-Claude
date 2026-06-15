---
name: robbohome-memory
description: Persistent memory of Dan's (RobboHome) projects, infra, hosts, and preferences. Use whenever working on RobboHome infrastructure, the SCC site, Gym Coach, GeekyThings, LoopCoach, OpenClaw, the MetaMCP hub, servers (svr001/2/3, Contabo VPS, Macs), Home Assistant, or any home-lab task — to recall non-obvious facts and to append durable new ones. Consult this before assuming, and write to it when you learn something the code/git history doesn't already record.
---

# Skill: RobboHome memory

A persistent, file-based knowledge store of **non-obvious facts** about Dan's
world that aren't derivable from a repo, git history, or `CLAUDE.md`/`AGENTS.md`.
It is shared with Claude Code (this is the same store both agents read/write),
so keeping it current benefits every agent on the machine.

## Where it lives

```
/Users/robbohomebot/.claude/projects/-Users-robbohomebot-Projects-OpenClaw-SCC/memory/
```

- `MEMORY.md` — the **index**: one line per fact (`- [Title](file.md) — hook`).
  This is the live source of truth for "what do I know about?". Read it first.
- One `*.md` file per fact, with frontmatter (`name`, `description`,
  `metadata.type`) and the fact in the body.

The path lives under `.claude/` for historical reasons — it is just a plain
directory of markdown files, equally usable from Codex. Don't move it (Claude
Code reads the same path); only its contents matter.

## Recall protocol (reading)

1. When a task touches anything RobboHome, **Read `MEMORY.md`** at the path above
   for the current index.
2. Skim the one-line hooks; for any that look relevant, **Read the full fact
   file** (`<name>.md` in the same dir) before acting.
3. Treat each fact as a **point-in-time observation**, not live state — if a fact
   names a file, flag, port, or host, verify it still exists before relying on it.

## Capture protocol (writing)

Append a new fact when you learn something **durable and non-obvious** —
a gotcha, a "why" behind a decision, an access detail, an ongoing constraint.
Don't record what the repo, git history, or an `AGENTS.md` already states, and
don't record fleeting in-session detail.

1. **Check for an existing file** that already covers it — update that file rather
   than create a duplicate. Delete a fact that turns out to be wrong.
2. Write one fact per file, `<short-kebab-slug>.md`, with frontmatter:
   ```markdown
   ---
   name: <short-kebab-case-slug>
   description: <one-line summary — used to judge relevance during recall>
   metadata:
     type: user | feedback | project | reference
   ---

   <the fact. For feedback/project, follow with **Why:** and **How to apply:** lines.
   Link related facts with [[their-slug]].>
   ```
   - `user` — who Dan is (role, expertise, preferences).
   - `feedback` — how Dan wants you to work (corrections, confirmed approaches); include the why.
   - `project` — ongoing work, goals, constraints not derivable from code/git. Convert relative dates to absolute.
   - `reference` — pointers to external resources (URLs, dashboards, tickets).
3. **Add a one-line pointer to `MEMORY.md`** (`- [Title](file.md) — hook`). The
   index is loaded as context each session; keep it to one line per fact, never
   put fact bodies in `MEMORY.md`.
4. Link related facts in the body with `[[slug]]`. A `[[slug]]` that doesn't exist
   yet is fine — it marks something worth writing later.

## Current index (snapshot — read MEMORY.md for the live version)

- **gymcoachshared-canonical** — gymcoachshared is the canonical Gym Coach product; three-app topology, CI data-wipe gotcha, Dan's seeded account id.
- **scc-vps-access** — Contabo VPS, ssh alias `scc_contabo`, compose project `scc`, postgres DB, manage.py command pattern.
- **annual-dpi-print-upload** — comprehensive CSV approach: walk folder tree + parse Dicentra RTFs, then drive Playwright on v2 admin portal.
- **competitions-v2-backport** — POST to /api/v1/competitions/imports with bearer auth; per-season schema; chunk at 80MB.
- **scc-release-pipeline** — CHANGELOG-driven tag from `main`, ~5 min build+deploy, self-hosted runner pulls; LocMemCache is per-worker.
- **scc-site-config-knobs** — CONTACT_EMAIL / CLUB_VENUE / CLUB_MEETING_TIME settings, en_GB date overrides, NR Browser toggle.
- **scc-competitions-app-state** — `competitionsv2` is the only comp model; legacy retired v0.7.0; only 2 upload endpoints remain.
- **scc-admin-credentials** — Wagtail admin login in SOPS as `SCC_ADMIN_USERNAME` / `SCC_ADMIN_PASSWORD`.
- **svr002-to-vps-migration** — retire svr002 to a Contabo VPS; ollama dropped, Syncthing→svr003 backups, CI runners reinstall; email-hosting an open want.
- **mac-disk-cleanup** — 256GB Mac fills recurringly; safe purge order; /private/tmp is a red herring.
- **openclaw-cron-ops** — manage crons via `openclaw cron`; hollow agents have no shell tool so shell crons fail — re-point to `main`.
- **family-william-agent** — William's WhatsApp ChatGPT agent; pending WhatsApp re-link.
- **metamcp-hub** — central MCP host on svr001:12008 (MetaMCP); namespace→endpoint model; the hub behind the themed Codex MCP servers.
- **contabo-backup-runbook** — nightly restic to svr003:/mnt/backup/contabo-restic, ssh `svr003` port 2223, retention 7d/4w/6m.

## Related

- `jira` skill — the canonical TODO tracker (memory is for *facts*, Jira is for *work items*).
- The MetaMCP hub and themed MCP namespaces are documented in the `metamcp-hub` fact; the per-area skills (`grafana`, `home-assistant`, `unifi`, `scc-content`, …) now prefer those hub tools.
