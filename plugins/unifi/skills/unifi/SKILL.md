---
description: UniFi specialist context — UDM at 192.168.1.1, sirkirby/unifi-network-mcp via OpenClaw, dedicated mcp-beacon admin account, Phase 2 write scope.
---

# Skill: UniFi (📡 Beacon)

Beacon is the OpenClaw specialist agent for the home UniFi network. It runs locally on this Mac and talks to the UDM via the `sirkirby/unifi-network-mcp` MCP server, spawned on-demand by OpenClaw.

## Controller

| | |
|---|---|
| Hardware | UniFi Cloud Gateway (UDM) |
| LAN IP | `192.168.1.1` |
| Mac LAN IP | `192.168.1.91` |
| TLS | self-signed cert (verify_ssl = `false`) |
| Firmware | (filled in once verified during Phase 1) |
| Network app version | (filled in once verified during Phase 1) |

## MCP server

| | |
|---|---|
| Project | [`sirkirby/unifi-network-mcp`](https://github.com/sirkirby/unifi-network-mcp) |
| Install | `uvx unifi-network-mcp@latest` (Python, no Docker) |
| Transport | stdio, on-demand spawn by OpenClaw |
| Mutation gate | preview-then-confirm via MCP-returned `confirm:<id>` token |

### OpenClaw registration

```bash
openclaw mcp set unifi-network '{
  "transport": "stdio",
  "command": "uvx",
  "args": ["unifi-network-mcp@latest"],
  "env": {
    "UNIFI_NETWORK_HOST":      { "ref": "UNIFI_NETWORK_HOST" },
    "UNIFI_NETWORK_USERNAME":  { "ref": "UNIFI_NETWORK_USERNAME" },
    "UNIFI_NETWORK_PASSWORD":  { "ref": "UNIFI_NETWORK_PASSWORD" },
    "UNIFI_NETWORK_VERIFY_SSL":{ "ref": "UNIFI_NETWORK_VERIFY_SSL" }
  }
}'
```

### Claude Code registration (for dev sessions)

```bash
claude mcp add --transport stdio --scope user unifi-network -- \
  uvx unifi-network-mcp@latest
# then set the four env vars from the same source the plain shell can reach
```

## Dedicated admin account

Service account (NOT the user's personal admin):

| | |
|---|---|
| Username | (set during Phase 1 setup — distinct from personal admin) |
| Auth scope | "Restrict to local access only" — not linked to any ui.com account |
| Roles | UniFi OS = **Limited Admin**, Network = **Admin**. No Protect/Access/Talk. |
| 2FA | disabled (headless flow can't satisfy TOTP) |
| Audit | Every MCP call is logged in UDM under this username — distinct from personal admin trail |

## Secrets

Stored as SecretRefs in OpenClaw (`openclaw secrets configure`):

| Ref | Value |
|---|---|
| `UNIFI_NETWORK_HOST` | `192.168.1.1` |
| `UNIFI_NETWORK_USERNAME` | (the dedicated admin username) |
| `UNIFI_NETWORK_PASSWORD` | (strong generated password) |
| `UNIFI_NETWORK_VERIFY_SSL` | `false` |

Rotation: update the SecretRef + `openclaw secrets reload`. No MCP/agent restart.

## Tool inventory

(Filled during Phase 1 step 6 by Beacon listing tools and grouping by read/write — kept here for cross-session reference.)

## Phase 2 write scope (live)

**Allowed writes:**
- Rename client
- Add/edit/remove static DHCP reservation
- Add/edit/remove local DNS record

**Refused (manual UDM UI only):**
- VLAN / network create / delete
- Firewall rules (legacy or ZBF)
- Admin user changes
- Subnet / gateway / WAN config
- Factory reset, firmware update

## Gotchas

- **Self-signed cert** — `UNIFI_NETWORK_VERIFY_SSL=false`. We trust it because we're on the LAN.
- **2FA must be disabled** on the dedicated admin — TOTP can't be satisfied headlessly.
- **Account scope = "Local access only"** — not linked to ui.com to avoid OAuth/MFA flow.
- **UDM firmware updates** can rotate the controller's session/cookie format. After a UDM upgrade, run `openclaw doctor` and re-test Beacon. If MCP auth fails, no creds rotation needed — just first-call retry.
- **Never echo creds.** SOUL.md item 6 — Beacon redacts MCP env in any quoted output.
- **`unifi-network-mcp` API_KEY mode is read-only** — for write coverage we use username+password.

## Related skills

- `home-assistant` — sister automation skill (HA + Unraid MCPs, similar wiring patterns)
- `svr002` — primary home server (where other secrets live, but **not** UniFi creds — those are on this Mac in OpenClaw's store)
