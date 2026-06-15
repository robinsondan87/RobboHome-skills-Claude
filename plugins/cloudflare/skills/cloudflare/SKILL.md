---
name: cloudflare
description: cloudflare skill for RobboHome automation.
---

# Skill: Cloudflare Management

## Via the MetaMCP hub (Codex `home` namespace)

Cloudflare's full API is exposed through the hub as the **code-mode** `cloudflare__*` tools (3 tools wrapping ~2500 CF endpoints in ~1k tokens), in the **`home`** namespace (MetaMCP on svr001, Codex MCP server `home`, bearer `HOMELAB_MCP_KEY`) — the hosted `https://mcp.cloudflare.com/mcp` registered with the SOPS `CLOUDFLARE_API_TOKEN` as bearer. **Prefer it for ad-hoc CF queries/changes.** The direct API + curl recipes below stay as the fallback and for scripted bulk ops. See the `metamcp-hub` memory.

## Credentials
SOPS-encrypted in `~/data/config/.secrets.env`. Load with `source ~/data/config/load-secrets.sh`. Keys:
- CLOUDFLARE_API_TOKEN
- CLOUDFLARE_ACCOUNT_ID=e9328cb93f3e347ef118a3b6dfa5678d
- CLOUDFLARE_TUNNEL_ID=02b1a979-319c-48a3-8ec4-29dbcab727d4
- CLOUDFLARE_ZONE_ID=93e554d66c0ed530fbd1387ce14a62a5
- CLOUDFLARE_IDP_ID=0413099f-04a1-45cb-866d-ce3d6bf874c6 (Google IDP)
- ZERO_TRUST_TEAM_DOMAIN=robbohome.cloudflareaccess.com

## Add a tunnel route + DNS record for a new service

```bash
# 1. Add tunnel ingress route (GET config first, append, then PUT back)
curl -X GET "https://api.cloudflare.com/client/v4/accounts/{ACCOUNT_ID}/cfd_tunnel/{TUNNEL_ID}/configurations" \
  -H "Authorization: Bearer {TOKEN}"

# PUT updated config with new entry before the http_status:404 catch-all

# 2. Create DNS CNAME record
curl -X POST "https://api.cloudflare.com/client/v4/zones/{ZONE_ID}/dns_records" \
  -H "Authorization: Bearer {TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"type":"CNAME","name":"SUBDOMAIN.robbohome.com","content":"{TUNNEL_ID}.cfargotunnel.com","proxied":true}'
```

## Protect a service with Zero Trust Google auth

```bash
curl -X POST "https://api.cloudflare.com/client/v4/accounts/{ACCOUNT_ID}/access/apps" \
  -H "Authorization: Bearer {TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Service Name",
    "domain": "SUBDOMAIN.robbohome.com",
    "type": "self_hosted",
    "session_duration": "24h",
    "allowed_idps": ["{IDP_ID}"],
    "auto_redirect_to_identity": true,
    "policies": [{"name": "RobboHome Admins", "decision": "allow",
      "include": [{"email": {"email": "robinsondan87@gmail.com"}}]}]
  }'
```

## Add more emails to an existing Access policy
Zero Trust → Access → Applications → Edit → Policies → add email to include list.

## Notes
- Tunnel runs as Docker container on Unraid (192.168.1.200)
- Restart cloudflared container on Unraid if new routes aren't picked up
- Services must bind to 0.0.0.0 (not 127.0.0.1) for tunnel to reach them from Unraid
- Cockpit requires /etc/cockpit/cockpit.conf with Origins + AllowUnencrypted=true
