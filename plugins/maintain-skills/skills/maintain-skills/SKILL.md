---
description: Create or update skills in RobboHome-skills-Claude so that everything done across sessions is captured and stays current.
allowed-tools: Bash(git*) Bash(gh*) Bash(mkdir*) Bash(cat*)
---

# Maintain RobboHome Skills

Use this skill whenever something significant has been built, configured, or discovered in any workspace or session — so the knowledge persists into future sessions.

## When to use this skill
- After completing a new project or feature (create a deploy/context skill)
- After discovering something that took effort to figure out (e.g. Bambu integration dead-end)
- After a config change that affects how future work should be done
- After a session where you notice a gap: "I had to rediscover X from scratch"
- When a skill is stale or has incorrect information

## Repo structure

```
robinsondan87/RobboHome-skills-Claude
├── .claude-plugin/
│   └── marketplace.json       ← list of all plugins
└── plugins/
    └── <skill-name>/
        ├── .claude-plugin/
        │   └── plugin.json    ← plugin metadata
        └── skills/
            └── <skill-name>/
                └── SKILL.md   ← the actual skill content
```

## Creating a new skill

### 1. Write the SKILL.md
Good skills contain:
- **What it is / does** — one-paragraph context so the reader starts oriented
- **How to use it** — commands, workflows, decision trees
- **Key reference table** — paths, URLs, ports, IDs, repo names
- **Gotchas / notes** — things that took effort to discover; what NOT to do
- **Related skills** — links to other skills in the repo

Bad skills:
- Just repeat what's obvious from the code
- Have no commands or concrete details
- Are too generic to be useful without the project open

### 2. Create the files

```bash
SKILL=<skill-name>
REPO=/tmp/RobboHome-skills-Claude

# Clone if not already present
cd /tmp && gh repo clone robinsondan87/RobboHome-skills-Claude 2>/dev/null || true

mkdir -p $REPO/plugins/$SKILL/.claude-plugin
mkdir -p $REPO/plugins/$SKILL/skills/$SKILL

# Write plugin.json
cat > $REPO/plugins/$SKILL/.claude-plugin/plugin.json << 'JSON'
{
  "name": "<skill-name>",
  "version": "1.0.0",
  "description": "<one-line description>",
  "author": { "name": "robinsondan87" }
}
JSON

# Write SKILL.md (add content here)
cat > $REPO/plugins/$SKILL/skills/$SKILL/SKILL.md << 'MD'
---
description: <same as plugin.json description>
---

# Skill: <Title>

...
MD
```

### 3. Register in marketplace.json

Add a line to `.claude-plugin/marketplace.json` in the `plugins` array:
```json
{ "name": "<skill-name>", "source": "./plugins/<skill-name>" }
```

Keep the list alphabetically sorted by name.

### 4. Commit and push

```bash
cd /tmp/RobboHome-skills-Claude
git add -A
git commit -m "add <skill-name> skill: <one-line summary>"
git push origin main
```

---

## Updating an existing skill

```bash
cd /tmp/RobboHome-skills-Claude

# Pull latest first
git pull

# Edit the skill
$EDITOR plugins/<skill-name>/skills/<skill-name>/SKILL.md

git add plugins/<skill-name>/skills/<skill-name>/SKILL.md
git commit -m "update <skill-name>: <what changed and why>"
git push origin main
```

---

## Checking what skills exist

```bash
gh api "repos/robinsondan87/RobboHome-skills-Claude/git/trees/HEAD?recursive=1" \
  --jq '.tree[] | select(.path | endswith("SKILL.md")) | .path'
```

Or locally:
```bash
ls /tmp/RobboHome-skills-Claude/plugins/
```

---

## Key reference

| Item | Value |
|------|-------|
| Repo | `robinsondan87/RobboHome-skills-Claude` |
| Local clone | `/tmp/RobboHome-skills-Claude` (re-clone if stale) |
| Marketplace index | `.claude-plugin/marketplace.json` |
| Plugin format | `.claude-plugin/plugin.json` per plugin |
| Skill format | `skills/<name>/SKILL.md` per plugin |

## Current skill inventory

See marketplace.json for the authoritative list. As of April 2026:

**Infrastructure:** cicd, cloudflare, cloudflare-tunnel, deploy, deployment-patterns, docker-management, gpu, init-project, ollama, openclaw-update, portainer, register-runner, server-bootstrap, server-setup, svr002

**iOS:** ios-fastlane, ios-sideload, xcode-setup

**Projects:** deploy-gym, deploy-geekythings, deploy-ios, bambu-integration, geekythings-business, geekythings-listings, scc-content, william-agent

**Meta:** maintain-skills (this skill)
