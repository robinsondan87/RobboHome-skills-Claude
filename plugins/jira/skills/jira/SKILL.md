---
description: Atlassian Cloud Jira (RobboHome workspace) — canonical TODO tracker. SOPS-managed creds, REST v3 quick-reference, ADF description converter, TODO discovery sweep. Replaces the deprecated `plane` skill (migrated 2026-05-06).
---

# Skill: Jira (canonical TODO tracker)

Atlassian Cloud Jira on the free tier — RobboHome workspace at `https://robbohome.atlassian.net`. Replaced self-hosted Plane on 2026-05-06; all 86 open issues from the 6 Plane projects were migrated and Plane was decommissioned. Use this skill anywhere the previous `plane:plane` skill was used.

## Where it lives

| | |
|---|---|
| Cloud site | `https://robbohome.atlassian.net` |
| Service account | RobboHom Bot — `robbohomebot@gmail.com` (accountId `712020:b79cf3ef-d472-45bf-afaa-c1e045d0e4af`, timezone Europe/London) |
| Auth | HTTP Basic, `$JIRA_EMAIL:$JIRA_API_TOKEN` |
| API base | `$JIRA_BASE_URL/rest/api/3/` (note: `/rest/api/3/`, NOT `/api/v1/` like Plane) |
| Free tier limits | 10 users, unlimited projects, 2GB storage |
| Issue types | `Task`, `Epic`, `Subtask` (in every project) |
| Priorities | `Highest`, `High`, `Medium`, `Low`, `Lowest` |
| Default workflow states | `To Do`, `In Progress`, `Done` |

## Projects

| Key | Name | Notes |
|---|---|---|
| `INFRA` | Infrastructure | Servers, networking, monitoring, SOPS, fleet ops |
| `LC` | Loop Coach | Diabetes/insulin profile experiments |
| `HA` | Home Assistant | HA overhaul follow-ons + epics |
| `SCC` | Stafford Camera Club | Site rebuild priorities + UX backlog |
| `GT` | GeekyThings | 3D printing business workflow ops |
| `OC` | OpenClaw | Agent cluster admin + improvements |
| `KAN` | RobboHome (default) | The signup default — leave alone or retire later |

The 6 working projects map 1:1 to the prior Plane keys. Issue numbering does NOT match between Plane and Jira (Plane returned newest-first during migration); each migrated Jira issue has a footer line `Migrated from Plane: <PLANE-KEY>-<seq> on 2026-05-06` for traceability.

## Secrets in SOPS

```
JIRA_BASE_URL    https://robbohome.atlassian.net
JIRA_EMAIL       robbohomebot@gmail.com
JIRA_API_TOKEN   ATATT3…   (generate at id.atlassian.com → Security → API tokens)
```

Load via `source ~/data/config/load-secrets.sh`. Same SOPS file as the rest of the fleet (`~/data/config/.secrets.env`).

The bot account is **NOT** a site admin — it can list/read/create/update issues in projects but cannot create new projects. Project creation must happen in the Jira UI by a site admin.

## API quick-reference

Auth: `-u "$JIRA_EMAIL:$JIRA_API_TOKEN"` for every call.

```bash
source ~/data/config/load-secrets.sh

# Verify auth
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" "$JIRA_BASE_URL/rest/api/3/myself" | jq '.displayName'

# List projects
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" "$JIRA_BASE_URL/rest/api/3/project/search" \
  | jq -r '.values[] | "\(.key)\t\(.name)"'

# List open issues in a project (To Do + In Progress)
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -G "$JIRA_BASE_URL/rest/api/3/search/jql" \
  --data-urlencode 'jql=project=OC AND statusCategory != Done ORDER BY created DESC' \
  --data-urlencode 'fields=summary,priority,status' \
  | jq -r '.issues[]? | "\(.key)\t\(.fields.priority.name)\t\(.fields.status.name)\t\(.fields.summary)"'

# Create an issue (description must be ADF — see converter below)
curl -s -X POST -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -H "Content-Type: application/json" \
  "$JIRA_BASE_URL/rest/api/3/issue" \
  -d '{
    "fields": {
      "project": { "key": "INFRA" },
      "summary": "…",
      "issuetype": { "name": "Task" },
      "priority": { "name": "Medium" },
      "description": {
        "type": "doc", "version": 1,
        "content": [{ "type": "paragraph", "content": [{ "type": "text", "text": "…" }] }]
      }
    }
  }'

# Update an issue
curl -s -X PUT -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -H "Content-Type: application/json" \
  "$JIRA_BASE_URL/rest/api/3/issue/INFRA-12" -d '{"fields":{"priority":{"name":"High"}}}'

# Transition (To Do → In Progress → Done). First list available transition IDs:
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" "$JIRA_BASE_URL/rest/api/3/issue/INFRA-12/transitions" \
  | jq -r '.transitions[] | "\(.id)\t\(.name)"'
# Then POST the transition id:
curl -s -X POST -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -H "Content-Type: application/json" \
  "$JIRA_BASE_URL/rest/api/3/issue/INFRA-12/transitions" -d '{"transition":{"id":"<id>"}}'
```

**Note on search endpoint**: `/rest/api/3/search` was deprecated in favour of `/rest/api/3/search/jql` in 2024. The new endpoint requires `nextPageToken` cursor pagination instead of `startAt` integer offsets. Use `search/jql` for new code.

## Description format: ADF (Atlassian Document Format)

Jira REST v3 requires descriptions as structured ADF JSON, not HTML or markdown. The minimal valid document is:

```json
{ "type": "doc", "version": 1,
  "content": [{ "type": "paragraph", "content": [{ "type": "text", "text": "hello" }] }] }
```

Common block nodes used in TODO/issue descriptions:

```js
// Single paragraph
{ type: "paragraph", content: [{ type: "text", text: "..." }] }

// Hard break inside paragraph (multi-line text)
{ type: "paragraph", content: [
  { type: "text", text: "line 1" },
  { type: "hardBreak" },
  { type: "text", text: "line 2" },
]}

// Ordered/bullet list
{ type: "orderedList",  // or "bulletList"
  content: [
    { type: "listItem", content: [
      { type: "paragraph", content: [{ type: "text", text: "Step 1" }] },
    ]},
  ]
}

// Horizontal rule (between sections)
{ type: "rule" }

// Inline marks for code/strong/em
{ type: "text", text: "code", marks: [{ type: "code" }] }
{ type: "text", text: "bold", marks: [{ type: "strong" }] }
```

For HTML inputs (e.g. importing Plane content), see the migration script template below for a minimal HTML→ADF converter.

## When to add a Jira issue vs do it inline

**Add to Jira** when the work is:
- Multi-session (won't finish in this conversation)
- Cross-machine / requires coordination
- Worth surfacing to someone else later

**Don't add to Jira** when:
- It's a quick fix you're about to do anyway
- It's a code TODO that belongs in the codebase as a comment
- It's already covered by an existing INFRA-* issue (search before creating)

Default to creating a Jira issue when the user says **"add a TODO"**, **"track this"**, **"come back to this"**, or similar — unless they specifically say `~/.claude/TODO.md` or another file.

Keep names imperative and short (under ~70 chars). Description in ADF; lead with a "Why:" paragraph and end with **Apply** + **Verify** sections (mirrors the Plane convention).

## TODO discovery sweep (find work that should be in Jira)

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
for ws in ~/StudioProjects ~/Documents ~/3dPrinting ~/SCC_2026 ~/openclaw-work; do
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

echo "=== Already in Jira (open count per project) ==="
for k in INFRA LC HA SCC GT OC; do
  total=$(curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" -G "$JIRA_BASE_URL/rest/api/3/search/jql" \
    --data-urlencode "jql=project=$k AND statusCategory != Done" \
    --data-urlencode "fields=summary" --data-urlencode "maxResults=0" \
    | jq -r '.total // (.issues | length // "?")')
  printf "  %-6s open=%s\n" "$k" "$total"
done
```

**Triage filters** — most of what surfaces is noise:
- **Forked/upstream repos** — TODOs there aren't your work, skip.
- **Already-tracked issues** — search the relevant project (`jql=project=X AND text~"keyword"`) before creating a duplicate.
- **Stale dated TODOs** — if a TODO has a "reassess by 2026-02-03" line and that date has passed, it's probably moot.

## Migration-script template (HTML → ADF)

Used for the Plane → Jira migration on 2026-05-06. Keep this for any future bulk import from an HTML source. Plain Node, no SDK install needed.

```javascript
// Minimal HTML → ADF: preserves <p>, <ol>/<ul>/<li>; flattens inline marks to plain text.
function stripTags(html) {
  return (html || '')
    .replace(/<br\s*\/?>/gi, '\n')
    .replace(/<\/p>/gi, '\n')
    .replace(/<[^>]+>/g, '')
    .replace(/&lt;/g, '<').replace(/&gt;/g, '>').replace(/&amp;/g, '&')
    .replace(/&quot;/g, '"').replace(/&#39;/g, "'").replace(/&nbsp;/g, ' ')
    .replace(/[\r\n]+\s*[\r\n]+/g, '\n').trim();
}
function paragraphFromText(text) {
  const parts = String(text || '').split(/\n+/).filter(Boolean);
  return parts.length === 0
    ? { type: 'paragraph', content: [] }
    : { type: 'paragraph', content: parts.flatMap((t, i) =>
        i === 0 ? [{ type: 'text', text: t }] : [{ type: 'hardBreak' }, { type: 'text', text: t }]) };
}
function htmlToAdf(html) {
  if (!html) return [paragraphFromText('')];
  const blocks = [];
  const blockRe = /<(p|ol|ul)([^>]*)>([\s\S]*?)<\/\1>/gi;
  let m, lastEnd = 0;
  while ((m = blockRe.exec(html)) !== null) {
    if (m.index > lastEnd) {
      const between = stripTags(html.slice(lastEnd, m.index));
      if (between) blocks.push(paragraphFromText(between));
    }
    const [, tag, , inner] = m;
    if (tag.toLowerCase() === 'p') {
      const txt = stripTags(inner);
      if (txt) blocks.push(paragraphFromText(txt));
    } else {
      const items = [];
      const liRe = /<li[^>]*>([\s\S]*?)<\/li>/gi;
      let lim;
      while ((lim = liRe.exec(inner)) !== null) {
        const liText = stripTags(lim[1]);
        if (liText) items.push({ type: 'listItem', content: [paragraphFromText(liText)] });
      }
      if (items.length) blocks.push({ type: tag.toLowerCase() === 'ol' ? 'orderedList' : 'bulletList', content: items });
    }
    lastEnd = m.index + m[0].length;
  }
  if (lastEnd < html.length) {
    const tail = stripTags(html.slice(lastEnd));
    if (tail) blocks.push(paragraphFromText(tail));
  }
  if (blocks.length === 0) blocks.push(paragraphFromText(stripTags(html)));
  return blocks;
}

const JIRA_AUTH = 'Basic ' + Buffer.from(`${process.env.JIRA_EMAIL}:${process.env.JIRA_API_TOKEN}`).toString('base64');
async function jiraCreate(projectKey, summary, htmlDesc, priorityName) {
  const fields = {
    project: { key: projectKey },
    summary: summary.slice(0, 250),
    issuetype: { name: 'Task' },
    description: { type: 'doc', version: 1, content: htmlToAdf(htmlDesc) },
  };
  if (priorityName) fields.priority = { name: priorityName };
  const r = await fetch(`${process.env.JIRA_BASE_URL}/rest/api/3/issue`, {
    method: 'POST',
    headers: { authorization: JIRA_AUTH, 'content-type': 'application/json', accept: 'application/json' },
    body: JSON.stringify({ fields }),
  });
  if (!r.ok) throw new Error(`${r.status} ${await r.text()}`);
  return r.json();
}
// Plane priority → Jira priority mapping:
const PRIORITY_MAP = { urgent: 'Highest', high: 'High', medium: 'Medium', low: 'Low' };
```

## Adding a new project

Project creation requires **site-admin** permissions, which the bot service account does NOT have. Either:
1. Promote the bot to site-admin in https://admin.atlassian.com (granted role), or
2. Create the project manually in the Jira UI: Projects → Create project → Kanban → Team-managed → set the key explicitly (Jira's auto-suggest is usually wrong).

After creation, store the new key + its `id` in this skill's table so future scripts can target it.

## Migrating away from Plane (historical reference)

The 2026-05-06 migration:
1. Verified Jira API auth + listed Plane projects.
2. Created the 6 Jira projects manually (UI) with matching keys: `INFRA`, `LC`, `HA`, `SCC`, `GT`, `OC`.
3. Ran a Node migration script (HTML → ADF, priority map, footer with original Plane key) — see template above.
4. 86/86 issues migrated, 0 failures.
5. Decommissioned Plane: `docker compose down --volumes` on svr002 (4.3GB reclaimed), removed `plane.robbohome.com` from Cloudflare tunnel ingress + DNS.
6. Marked the `plane:plane` skill as deprecated.

## Gotchas

- **Project creation needs site-admin** — bot account doesn't have it; do it via UI.
- **Project key must be set explicitly** in the create-project UI flow; Jira's auto-suggest gives weird short keys.
- **Description is ADF, not HTML** — REST v3 hard-rejects HTML/markdown strings. Use the converter above or a single-paragraph wrapper for plain text.
- **`/rest/api/3/search` is deprecated** — use `/rest/api/3/search/jql` with `nextPageToken` cursor pagination.
- **Priorities must use exact names** — `Highest/High/Medium/Low/Lowest`. Plane's `none` has no Jira equivalent; omit the field for "no priority".
- **Issue numbering does NOT correlate** with the original Plane sequence_id after migration — see the per-issue footer for the link back.
- **Free tier**: 10 users, 2GB storage. Don't add more service accounts than needed.

## Related skills

- `secrets:secrets` — where the `JIRA_*` keys live (`~/data/config/.secrets.env`)
- `plane:plane` — DEPRECATED predecessor; kept for historical reference of the Plane → Jira migration only
