---
description: home-assistant — HA VM on Unraid, zigbee2mqtt, and the HA/Unraid MCP servers.
---

# Skill: Home Assistant & MCP Servers

Home Assistant OS runs as an Unraid VM (`Home Assistant`), not inside a Docker container. The `HomeAssistant_inabox` container on Unraid is just a helper — HA itself is the KVM VM.

## Access
| | |
|---|---|
| HA UI | http://192.168.1.151:8123 (VM IP, DHCP) |
| Unraid VM name | `Home Assistant` |
| VM host | svr001 (Unraid) |

## VM management from svr001
```bash
ssh svr001 "virsh list --all"
ssh svr001 "virsh start 'Home Assistant'"
ssh svr001 "virsh destroy 'Home Assistant'"        # hard-stop (use when VM is stuck)
ssh svr001 "virsh shutdown 'Home Assistant'"       # graceful shutdown
ssh svr001 "virsh edit 'Home Assistant'"           # edit persistent XML
```

Persistent XML: `/etc/libvirt/qemu/Home Assistant.xml`. Runtime XML: `/run/libvirt/qemu/Home Assistant.xml` (changes here don't persist).

## USB passthrough
Zigbee dongle (Sonoff ZBDongle Plus V2, USB `10c4:ea60` Silicon Labs CP210x) is passed through via `<hostdev>`:

```xml
<hostdev mode='subsystem' type='usb' managed='yes'>    <!-- managed='yes' is the safe setting -->
  <source startupPolicy='optional'>
    <vendor id='0x10c4'/>
    <product id='0xea60'/>
  </source>
  <address type='usb' bus='0' port='2'/>
</hostdev>
```

**Why `managed='yes'`:** with `managed='no'`, libvirt does not unbind the host's `cp210x` driver before QEMU grabs the device. If the dongle power-glitches while the VM is running, cp210x rebinds on the host → QEMU keeps a stale usbfs handle → dmesg floods with `usbfs: process … (qemu-system-x86) did not claim interface 0 before use` → z2m sees `Failed to start EZSP layer with status=HOST_FATAL_ERROR`. `managed='yes'` makes libvirt always detach cp210x first, so the race can't happen.

## zigbee2mqtt
### Symptom: `Failed to start EZSP layer with status=HOST_FATAL_ERROR`
Two independent causes — check both:

1. **Stuck QEMU USB handle** (after a power glitch / replug while VM running). Soft reboots from inside HA **don't** fix it. Fix:
   ```bash
   ssh svr001 "virsh destroy 'Home Assistant' && virsh start 'Home Assistant'"
   ```
   Permanent fix: set `managed='yes'` on the hostdev (above).

2. **z2m 2.7.1 regression** (GitHub issue Koenkk/zigbee2mqtt#30062). Add to `/config/zigbee2mqtt/configuration.yaml`:
   ```yaml
   serial:
     port: /dev/serial/by-id/usb-Itead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_V2_e6b482990174ef11bafddf1e313510fd-if00-port0
     adapter: ember
     rtscts: false    # required for 2.7.1+
   ```
   Must be edited via File Editor addon directly — UI save does not persist reliably.

## MCP servers

Two MCP servers are registered with Claude Code (user scope, in `~/.claude.json`):

| Name | Purpose | Endpoint | Auth |
|---|---|---|---|
| `home-assistant` | HA state/control via HAOS add-on | `http://192.168.1.151:9583/<secret-path>` | secret path in URL |
| `unraid` | Unraid GraphQL wrapper | `http://192.168.1.200:6970/mcp` | Bearer token |

### home-assistant MCP
[`ha-mcp` HAOS add-on](https://github.com/homeassistant-ai/ha-mcp).
- Secret path stored at `/data/secret_path.txt` inside the add-on container
- Treat the secret path like a password — anyone with the URL has full HA control

### unraid MCP
[`jmagar/unraid-mcp`](https://github.com/jmagar/unraid-mcp), Docker container on Unraid.

Deployment (run from a host with secrets loaded — Mac or svr002):
```bash
source ~/data/config/load-secrets.sh

ssh svr001 docker run -d --name=unraid-mcp --restart=unless-stopped \
  -p 6970:6970 \
  -e UNRAID_API_URL=http://192.168.1.200/graphql \
  -e UNRAID_API_KEY="$UNRAID_API_KEY" \
  -e UNRAID_MCP_BEARER_TOKEN="$UNRAID_MCP_BEARER_TOKEN" \
  -e UNRAID_MCP_TRANSPORT=streamable-http \
  ghcr.io/jmagar/unraid-mcp:latest
```

- Unraid API key is generated in Unraid UI → Settings → Management Access → API Keys
- Bearer token is `openssl rand -hex 32`, used by the MCP client (Claude Code) to authenticate to this server

### Adding either to Claude Code
```bash
claude mcp add --transport http --scope user home-assistant "http://192.168.1.151:9583/<secret-path>"
claude mcp add --transport http --scope user unraid "http://192.168.1.200:6970/mcp" --header "Authorization: Bearer <bearer>"
claude mcp list   # verify ✓ Connected
```
Restart Claude Code after adding so tools load.

### Remote access
Both endpoints are LAN-only. For use from the Mac off-network:
- Tailscale (install inside HAOS for HA MCP; Unraid already has Tailscale)
- Cloudflare Tunnel via the existing svr001 tunnel (auth via Cloudflare Access)

## Secrets
Stored SOPS-encrypted in `~/data/config/.secrets.env`. Load via `source ~/data/config/load-secrets.sh` (see `skills/secrets/SKILL.md`). Relevant keys for this skill: `UNRAID_API_KEY`, `UNRAID_MCP_BEARER_TOKEN`.

## Related Skills
- `skills/svr002/SKILL.md` — primary home server, secrets live here
- `skills/docker-management/SKILL.md` — Unraid container management
- `skills/cloudflare-tunnel/SKILL.md` — for exposing MCP remotely
