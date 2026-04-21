---
description: cloudflare-tunnel skill for RobboHome automation.
---

# Skill: Cloudflare Tunnel & Zero Trust

Exposes self-hosted services on svr002 publicly via Cloudflare Tunnel without opening ports on the router.

## Credentials
Stored in `~/data/config/.secrets`:
```
CLOUDFLARE_API_TOKEN=...
CLOUDFLARE_ACCOUNT_ID=e9328cb93f3e347ef118a3b6dfa5678d
CLOUDFLARE_TUNNEL_ID=02b1a979-319c-48a3-8ec4-29dbcab727d4
CLOUDFLARE_ZONE_ID=93e554d66c0ed530fbd1387ce14a62a5
ZERO_TRUST_TEAM_DOMAIN=robbohome.cloudflareaccess.com
GOOGLE_OAUTH_CLIENT_ID=...
GOOGLE_OAUTH_CLIENT_SECRET=...
```

## How it works
1. `cloudflared` daemon runs on svr002 and maintains an outbound tunnel to Cloudflare
2. Cloudflare routes `*.robbohome.com` DNS to the tunnel
3. Cloudflare Zero Trust Access (optional) adds email-based auth in front of an app
4. The app itself handles its own auth (Bearer tokens etc.)

## Adding a new public subdomain
1. In Cloudflare dashboard → Zero Trust → Networks → Tunnels → select tunnel → Public Hostname
2. Add hostname: `<subdomain>.robbohome.com` → Service: `http://localhost:<PORT>`
3. DNS record is created automatically

## Cloudflare Access (Zero Trust auth)
Adds an email login gate in front of any app. Useful for admin tools (Portainer, Cockpit).
**Not used on gym-coach** — the app has its own Bearer token auth instead.

### Add Access protection to an app
```bash
source ~/data/config/.secrets
# Create application via API
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/apps" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"App Name","domain":"subdomain.robbohome.com","type":"self_hosted","session_duration":"24h"}'
```

### Remove Access protection from an app
```bash
source ~/data/config/.secrets
# List apps to find ID
curl -s "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/apps" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" | python3 -c "
import json,sys
for app in json.load(sys.stdin).get('result', []):
    print(app['id'], app['name'], app['domain'])
"
# Delete by ID
curl -X DELETE "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/access/apps/<APP_ID>" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN"
```

## Current protected apps
| App | Domain | Access? |
|-----|--------|---------|
| Portainer | portainer.robbohome.com | Yes |
| Cockpit | cockpit.robbohome.com | Yes |
| Open WebUI | ai.robbohome.com | Yes |
| Unraid | unraid.robbohome.com | Yes |
| Gym Coach | gymcoach.robbohome.com | **No** (own auth) |

## Troubleshooting
- **502/tunnel error**: Check `cloudflared` is running on svr002 — `ssh robbohomebot@192.168.1.17 'systemctl status cloudflared'`
- **Still showing Access login after removing**: Cloudflare propagation takes ~30s, or clear browser cookies
- **API returns 302 on requests**: Cloudflare Access is blocking — remove the Access app or add a bypass policy for `/api/*`
