---
description: init-project skill for RobboHome automation.
---

# Skill: Init New Project

When asked to initialise a new RobboHome project:

1. Ask for: project-name, subdomain, port number, project type (node/python/static)
2. Copy template: ~/data/infrastructure/templates/[type]/
3. Find/replace placeholders:
   - PROJECT_NAME → project-name
   - PORT → chosen port
   - SUBDOMAIN → subdomain.robbohome.com
   - YOUR_GITHUB_USERNAME → actual username
4. Create GitHub repo: gh repo create robbohome-PROJECT_NAME --private
5. Register a new self-hosted runner for the repo (each repo needs its own — personal GitHub accounts can't share runners across repos):
   ```bash
   # Get registration token
   gh api repos/robinsondan87/REPO_NAME/actions/runners/registration-token --method POST --jq '.token'
   # On svr002: create new runner dir, download, configure, install service
   # See ~/data/infrastructure/skills/register-runner/SKILL.md for full steps
   ```
6. Set required GitHub Secrets on the new repo:
   ```bash
   source ~/data/config/.secrets
   gh secret set CLOUDFLARE_API_TOKEN --repo robinsondan87/REPO --body "$CLOUDFLARE_API_TOKEN"
   gh secret set CLOUDFLARE_ACCOUNT_ID --repo robinsondan87/REPO --body "$CLOUDFLARE_ACCOUNT_ID"
   gh secret set CLOUDFLARE_TUNNEL_ID --repo robinsondan87/REPO --body "$CLOUDFLARE_TUNNEL_ID"
   # Plus any app-specific secrets (AUTH_SECRET, API keys, etc.)
   ```
7. Push initial commit → confirm Actions workflow triggers
8. Add Cloudflare tunnel route + DNS + Zero Trust app:
   - See cloudflare skill for full API calls
   - subdomain.robbohome.com → http://192.168.1.17:PORT

## Port allocation (avoid conflicts)

| Port  | Service       |
|-------|---------------|
| 3000  | hello-world   |
| 3001  | open-webui    |
| 3002  | geekythings   |
| 3003+ | new projects  |
| 3847  | gym-coach     |
| 9000  | portainer     |
| 9090  | cockpit       |
| 11434 | ollama        |
