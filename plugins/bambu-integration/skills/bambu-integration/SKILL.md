---
description: Reference for integrating with Bambu Studio URL schemes — and why direct browser→slicer integration is not possible for non-MakerWorld domains.
---

# Bambu Studio Integration — Final Findings

**READ THIS BEFORE attempting any Bambu Studio integration from a custom domain. This has been exhaustively investigated.**

## Direct "Open in Bambu" from a custom domain is NOT possible

Both URL schemes are restricted to trusted (Bambu/MakerWorld) domains:

| Scheme | Behaviour on custom domain |
|--------|---------------------------|
| `bambustudioopen://HOST/path` | Bambu has a domain allowlist — only MakerWorld and Bambu-owned domains work. Custom domains silently fail. |
| `bambustudio://open?file=ENCODED_URL` | Does not work on macOS at all. Works on Windows but gives "cancelled" for non-trusted domains. |
| `bambu-connect://` | For **printer control** (sending jobs to printers), NOT for opening files in the slicer. Wrong scheme entirely. |

## What was tried (exhaustively)

- `bambustudio://` — scheme is registered on macOS but `open?file=` command not handled
- `bambustudioopen://HOST/path` — domain restriction blocks non-MakerWorld hosts
- `bambustudioopen://https://HOST/path` — Chrome normalises nested `https://` to `https//` before passing to OS; Bambu opens blank
- `window.location.href` vs `<a href>` — both normalise the nested URL; no difference
- Token-based auth bypass (`/files-token/`) — tokens work server-side but Bambu still can't fetch due to domain restriction
- Disabling server auth entirely — still blocked by domain restriction in Bambu

## Workaround: Download button

Use the **Download** button to save the `.3mf` file locally, then open it in Bambu Studio manually.
macOS will open `.3mf` files with Bambu Studio if it is the registered handler for that extension.

The "Open in Bambu" button was removed from the GeekyThings product page at v1.0.35.

## Server infrastructure notes (kept for reference)

- `POST /api/file_token` → `{ token }` (TTL: 300s default, `FILE_TOKEN_TTL_SECONDS` env var)
- File served at: `GET /files-token/<token>/<any_filename>.3mf` (no session auth required)
- Cloudflare Zero Trust bypass created for `geekythings.robbohome.com/files-token` (App ID: `45000b6f-89c1-4583-8e10-6c305815a4ac`) — no longer needed but left in place
- `do_HEAD` implemented in server.py for `/files-token/` paths (since v1.0.32, kept)
- Username/password auth is **enabled** — `AUTH_DISABLED` is NOT set in docker-compose.prod.yml
