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

**Verified live 2026-04-25** — full categorised list in [`~/.openclaw/workspace/agents/homelab-unifi-agent/TOOLS.md`](../../../../.openclaw/workspace/agents/homelab-unifi-agent/TOOLS.md).

The MCP exposes only **5 top-level entry points** — the 167 underlying tools are accessed via a meta-tool pattern:

- `unifi_tool_index` — discover tools (call first, optionally with a filter)
- `unifi_execute` — execute one tool by name with arguments
- `unifi_batch` — execute several in parallel (good for one-shot diagnose)
- `unifi_batch_status` — poll a batch operation
- `unifi_load_tools` — advanced direct-load (avoid)

### Counts (167 total)

| Category | Count | Highlights |
|---|---|---|
| Read | 81 | `list_clients`, `list_devices`, `list_networks`, `list_dns_records`, `list_alarms`, `get_dashboard`, `get_network_health`, `get_port_stats`, `get_speedtest_results`, `get_system_info`, `get_wan_status` (via `get_gateway_stats`) |
| Phase 2 allowed (5) | 5 | `unifi_rename_client`, `unifi_set_client_ip_settings` (fixed-IP + local-DNS), `unifi_create_dns_record`, `unifi_update_dns_record`, `unifi_delete_dns_record` |
| Phase 2 refused | 81 | firewall, VLAN/network create/update, port profiles, WLAN, admin users, devices (adopt/forget/restart), VPN, QoS, traffic rules, alarms (archive/forget), backups, clients (block/unblock/forget/authorize) |

### Important UDM modeling note

**Static DHCP reservations are client properties, not separate entities.** UDM does not expose a separate "DHCP reservation" tool — `unifi_set_client_ip_settings` sets a fixed IP (and optionally a local DNS record) on a client by MAC. Beacon should treat this as the static-lease tool.

### Mutation flow

Every Phase 2 write is gated by a preview→confirm token:
1. Beacon calls `unifi_execute(name=<tool>, arguments={..., confirm: false})` → MCP returns a preview
2. Beacon shows the user the diff + an exact `confirm:<id>` token. The MAC (or another unique identifier of the target) is a fine token — Beacon picked this convention live and it works well.
3. User replies the **exact** `confirm:<id>` token. Bare "Confirm" is rejected with *"Need the exact `confirm:<id>` token."* (negative-path verified live 2026-04-25)
4. Beacon calls `unifi_execute(..., confirm: true)` with the token → MCP applies → Beacon logs to `memory/incident-log.md`

### Known performance gotchas

- **Meta-tool indirection cost:** every Phase 2 write requires 3-5 model round-trips (`unifi_tool_index` → reason → `unifi_execute(preview)` → user confirm → `unifi_execute(apply)`). Plan for ~30-60s per write under good conditions.
- **Use `unifi_batch` for any diagnose needing 2+ reads** — sequential `unifi_execute` calls multiply latency.
- **TOOLS.md must stay under 12k chars** — OpenClaw truncates bootstrap files above that threshold, leaving the agent under-informed and increasing round-trip count. Keep TOOLS.md focused on highlights + meta-tool guidance; use `unifi_tool_index` filter for the long tail.
- **Codex auth 401s** — observed during the first live rename. Eventually recovers via auto-refresh; if it persists, run interactive auth refresh.

## Phase 2 write scope (live since 2026-04-25)

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
