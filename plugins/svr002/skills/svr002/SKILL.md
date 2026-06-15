---
name: svr002
description: RobboHome fleet host (Contabo VPS) — access, layout, and ops. (Formerly svr002; svr002 was retired 2026-05-29 and its workloads migrated here.)
---

# Skill: RobboHome Fleet Host (Contabo VPS)

> **svr002 (the old home Docker server, 192.168.1.17) is RETIRED as of 2026-05-29.**
> Its entire fleet was migrated to the **Contabo VPS** described below. Any older
> reference to `svr002` / `robbohome-server` / `161.97.66.102` for app hosting now
> means **this box**. (The home LAN box at `192.168.1.200` — unraid/NAS — is
> separate and still live.)

## Access
- SSH: `ssh scc_contabo` (alias in `~/.ssh/config`) — Tailscale IP `100.80.48.12`, user `root`. Tailscale machine name and OS hostname are both `contabo-fleet` as of 2026-05-30 (was `vmi3091030`).
- Public IP: `161.97.66.102`
- Reached over Tailscale; the box's own Tailscale SSH ACL is `action: accept` (no per-connection browser re-auth)

## What runs here
The whole RobboHome app fleet plus the Stafford Camera Club site. Each app is a
self-contained Docker Compose stack under `/opt/stacks/<app>/`:

| App | Stack dir | Local port | Public URL |
|-----|-----------|-----------|------------|
| scc (Wagtail) | `/opt/stacks/scc` | 127.0.0.1:8000 | staffordcameraclub.co.uk |
| loop-coach-v2 | `/opt/stacks/loop-coach-v2` | 127.0.0.1:3030 | loopcoach.robbohome.com |
| brickswap (+brickhive) | `/opt/stacks/brickswap` | 127.0.0.1:8080 | brickswap / brickhive.robbohome.com |
| geekythings | `/opt/stacks/geekythings` | 127.0.0.1:3002 | geekythings.robbohome.com |
| gym-coach | `/opt/stacks/gym-coach` | 127.0.0.1:3847 | gymcoach.robbohome.com |
| technews | `/opt/stacks/technews` | 127.0.0.1:3004 | technews.robbohome.com |
| hello-world | `/opt/stacks/hello-world` | 127.0.0.1:3000 | hello.robbohome.com |

## Layout
- `/opt/stacks/<app>/` — `compose.yml` + `.env` + `data/` (bind-mounted persistent data). Owned by the `deploy` user.
- `/opt/runners/<app>/` — one GitHub Actions self-hosted runner per repo, running as `deploy`.
- `/opt/backups/` — local DB dumps + volume tarballs (→ pushed to SVR003).
- `/opt/secrets/`, `/opt/scripts/` — SOPS-encrypted env + maintenance scripts.
- The `deploy` user (uid 1002, in the `docker` group) owns all of `/opt/stacks` & `/opt/runners`.

## Ingress (IMPORTANT — different from the old svr002 tunnel model)
- Host-level **nginx** terminates :80/:443 and reverse-proxies each hostname → `127.0.0.1:<port>`.
- App containers bind **`127.0.0.1:<port>`** only (NOT `0.0.0.0`) — never directly exposed on the public IP.
- Public `*.robbohome.com` apps: **Cloudflare DNS A-record → `161.97.66.102`, proxied**, zone SSL mode **Flexible** (Cloudflare↔origin over HTTP, so no origin cert needed). nginx vhosts are plain `:80`.
- `staffordcameraclub.co.uk` is a separate zone with its own Let's Encrypt cert on the box.
- These apps do **NOT** use the Cloudflare Tunnel (that tunnel still serves the unraid `.200` services + a couple of still-on-svr002... no longer — svr002 is off).

## Deploy model (CI)
- Push to `main` (or tag, per repo) → the repo's runner here builds the image, pushes to GHCR, then `cd /opt/stacks/<app> && docker compose pull && up -d`.
- **GHCR push auth:** workflows log in with a `write:packages` PAT stored as the repo secret **`GHCR_PAT`** (NOT `GITHUB_TOKEN`, which was denied `write_package`).
- The `deploy` user is `docker login`'d to ghcr.io (same PAT) for manual pulls.
- New public app: add an nginx `:80` vhost (`/etc/nginx/sites-available/<host>` → `proxy_pass http://127.0.0.1:<port>`), flip Cloudflare DNS to an A→box record, bind the container to 127.0.0.1.

## Common ops
```bash
ssh scc_contabo 'docker ps'
ssh scc_contabo 'su - deploy -c "cd /opt/stacks/<app> && docker compose ps"'
ssh scc_contabo 'su - deploy -c "cd /opt/stacks/<app> && docker compose logs --tail 100"'
ssh scc_contabo 'nginx -t && systemctl reload nginx'
```

Related skills: [[deploy]], [[deployment-patterns]], [[register-runner]], [[cloudflare-tunnel]], [[secrets]], [[docker-management]]. See also the SCC VPS access notes for the SCC compose project specifically.
