---
description: server-setup skill for RobboHome automation.
---

# Skill: Home Server Setup

When asked to set up or bootstrap the RobboHome home server:

1. Reference: ~/data/infrastructure/server-bootstrap.sh
2. Requires:
   - GITHUB_RUNNER_TOKEN (repo → Settings → Actions → Runners → New → copy token)
   - GITHUB_REPO (format: username/repo-name)
3. Run with sudo on Ubuntu Desktop 24.04 home server
4. After running: sudo reboot
5. Verify post-reboot:
   - nvidia-smi shows GPU
   - docker run --rm --gpus all nvidia/cuda nvidia-smi works
   - Cockpit at http://svr002:9090
   - Portainer at http://svr002:9000
   - GitHub runner shows Idle in repo settings
6. Deploy Ollama stack: cd ~/data/ollama && docker compose up -d
7. Add Cloudflare published routes for new services
