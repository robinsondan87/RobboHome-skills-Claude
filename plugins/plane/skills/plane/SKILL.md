---
description: Self-hosted Plane (project tracker) on svr002 — RobboHome workspace, Infrastructure project, API access via SOPS, and the TODO discovery sweep pattern.
---

# Skill: Plane (canonical TODO tracker)

[Plane](https://plane.so) Community Edition self-hosted on svr002 — Linear/Jira-style issue tracker. Replaces ad-hoc `TODO.md` files across the fleet.

## Where it lives

| | |
|---|---|
| Public URL | https://plane.robbohome.com (CF tunnel via 192.168.1.17:18080) |
| Local URL | `http://192.168.1.17:18080` |
| Stack | `~/data/plane-selfhost/plane-app/` on svr002 (12 containers under `plane-app-*`) |
| Workspace | `RobboHome` (slug: `robbohome`) |
| Primary project | **Infrastructure** (identifier `INFRA`, id `ba6c47c8-8e65-4146-bd75-ac58e27288bb`) |
| Auth | first-run admin: `Robinsondan87@gmail.com` (creds in SOPS as `PLANE_ADMIN_EMAIL` / `PLANE_ADMIN_PASS`) |

Stack management lives in the `docker-management` skill — see that for the deploy + the two startup gotchas (DATABASE_URL fallback, misleading "failed to start").

## Secrets in SOPS

```
PLANE_URL                 https://plane.robbohome.com
PLANE_API_KEY             plane_api_…  (workspace-scoped)
PLANE_WORKSPACE_SLUG      robbohome
PLANE_ADMIN_EMAIL         …
PLANE_ADMIN_PASS          …
```

Load via `source ~/data/config/load-secrets.sh`.

## API quick-reference

Base path: `$PLANE_URL/api/v1/workspaces/$PLANE_WORKSPACE_SLUG/`. Auth header: `x-api-key: $PLANE_API_KEY`.

The path prefix is **`v1`**; root paths like `/api/users/me/` return 401, and `/api/v1/workspaces/` is 404. Work always under the workspace slug.

```bash
source ~/data/config/load-secrets.sh

# List projects
curl -s -H "x-api-key: $PLANE_API_KEY" "$PLANE_URL/api/v1/workspaces/$PLANE_WORKSPACE_SLUG/projects/" | jq .

# Create an issue (priority: urgent | high | medium | low | none)
curl -s -X POST -H "x-api-key: $PLANE_API_KEY" -H "Content-Type: application/json" \
  "$PLANE_URL/api/v1/workspaces/$PLANE_WORKSPACE_SLUG/projects/<project_id>/issues/" \
  -d '{"name":"…","description_html":"<p>…</p>","priority":"high"}'

# List open issues in a project
curl -s -H "x-api-key: $PLANE_API_KEY" \
  "$PLANE_URL/api/v1/workspaces/$PLANE_WORKSPACE_SLUG/projects/<project_id>/issues/?state__group=backlog,unstarted,started" | jq .
```

The Infrastructure project's id is `ba6c47c8-8e65-4146-bd75-ac58e27288bb`. Other projects: list via the projects/ endpoint.

## When to add a Plane issue vs do it inline

**Add to Plane** when the work is:
- Multi-session (won't finish in this conversation)
- Cross-machine / requires coordination
- Worth surfacing to someone else later

**Don't add to Plane** when:
- It's a quick fix you're about to do anyway
- It's a code TODO that belongs in the codebase as a comment
- It's already covered by an existing INFRA-* issue (search before creating)

Default to creating a Plane issue when the user says **"add a TODO"**, **"track this"**, **"come back to this"**, or similar — unless they specifically say `~/.claude/TODO.md` or another file.

## TODO discovery sweep (find work that should be in Plane)

Run this when the user asks to "find all my TODOs" or before a planning session. Surveys, doesn't migrate.

```bash
source ~/data/config/load-secrets.sh

echo "=== GitHub repos owned by you ==="
gh repo list robinsondan87 --limit 200 --json name,isPrivate,isArchived,pushedAt \
  | jq -r '.[] | select(.isArchived | not) | .name' | while read -r r; do
    OPEN=$(gh issue list --repo "robinsondan87/$r" --state open --json number 2>/dev/null | jq length)
    [ "$OPEN" -gt 0 ] && echo "  robinsondan87/$r — $OPEN open issues"
  done

echo "=== Local code-comment TODOs (skip dependency dirs) ==="
for ws in ~/StudioProjects ~/Documents ~/3dPrinting ~/SCC_2026; do
  [ -d "$ws" ] || continue
  N=$(rg --files-with-matches \
       -g '!node_modules' -g '!.git' -g '!dist' -g '!build' -g '!target' \
       -g '!.next' -g '!.cache' -g '!__pycache__' -g '!vendor' -g '!.venv' -g '!venv' \
       -t md -t py -t js -t ts -t go -t rb -t sh \
       'TODO|FIXME|HACK|XXX' "$ws" 2>/dev/null | wc -l)
  echo "  $ws — $N files with markers"
done

echo "=== Standalone TODO.md files anywhere ==="
find ~/Documents ~/StudioProjects ~/data ~ -maxdepth 4 -name 'TODO*.md' \
  -not -path '*/node_modules/*' -not -path '*/.git/*' 2>/dev/null

echo "=== Claude session history with explicit 'todo' references ==="
grep -lE '"text":"[^"]*[Tt]odo[^"]*"' ~/.claude/projects/*/*.jsonl 2>/dev/null | head -10

echo "=== Already in Plane ==="
curl -s -H "x-api-key: $PLANE_API_KEY" \
  "$PLANE_URL/api/v1/workspaces/$PLANE_WORKSPACE_SLUG/projects/ba6c47c8-8e65-4146-bd75-ac58e27288bb/issues/" \
  | jq '.results | length' | xargs -I{} echo "  INFRA project: {} issues"
```

**Triage filters** — most of what surfaces is noise:
- **Forked/upstream repos** (e.g. AndroidAPS) — TODOs there aren't your work, skip.
- **Already-migrated GitHub issues** — `robbohome-config` had #1/#2/#3 mirrored in Plane as `INFRA-5/6/7`. Close the GitHub originals with a Plane link comment, don't double-track.
- **Stale dated TODOs** — if a TODO has a "reassess by 2026-02-03" line and that date has passed, it's probably moot.

## Migration-script template

Useful for bulk import (initial setup, post-survey, etc.). Written in plain `urllib` so no SDK install needed.

```python
import json, os, urllib.request, time

API = os.environ["PLANE_URL"] + "/api/v1"
WS  = os.environ["PLANE_WORKSPACE_SLUG"]
KEY = os.environ["PLANE_API_KEY"]
PROJECT_ID = "ba6c47c8-8e65-4146-bd75-ac58e27288bb"  # Infrastructure

def post_issue(name, html_desc, priority="medium"):
    r = urllib.request.Request(
        f"{API}/workspaces/{WS}/projects/{PROJECT_ID}/issues/",
        data=json.dumps({"name": name, "description_html": html_desc, "priority": priority}).encode(),
        method="POST",
        headers={"x-api-key": KEY, "Content-Type": "application/json"},
    )
    with urllib.request.urlopen(r) as resp:
        return json.loads(resp.read())

# Use description_html, not description_text — the latter is server-rendered from html.
# Convert markdown to simple HTML: paragraphs from \n\n, line breaks from \n.
```

## Adding a new project (for a different scope)

If we add `Apps`, `Personal`, `SCC`, etc., create the project once and store the id alongside the INFRA one in this skill so the migration script can target multiple projects:

```bash
source ~/data/config/load-secrets.sh
curl -s -X POST -H "x-api-key: $PLANE_API_KEY" -H "Content-Type: application/json" \
  "$PLANE_URL/api/v1/workspaces/$PLANE_WORKSPACE_SLUG/projects/" \
  -d '{"name":"GeekyThings","identifier":"GT","description":"…"}' | jq -r '.id'
```

## Related Skills
- `skills/docker-management/SKILL.md` — Plane stack deploy, gotchas
- `skills/secrets/SKILL.md` — where the `PLANE_*` keys live
- `skills/cloudflare/SKILL.md` — how the `plane.robbohome.com` tunnel route was added
