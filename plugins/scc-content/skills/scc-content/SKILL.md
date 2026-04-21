---
description: Stafford Camera Club competition import agent — process OneDrive results and upload to the SCC Wagtail website.
---

# SCC Content Agent

Automates getting competition results from OneDrive onto staffordcameraclub.co.uk.
Previously a fully manual Wagtail admin process.

## Architecture

- OneDrive synced locally at `~/OneDrive/SCC/Competition/Entries/`
- Agent scripts at `/Users/robbohomebot/scc-content/scripts/`
- Python venv at `/Users/robbohomebot/scc-content/.venv` (has `requests`)
- Website repo cloned at `/Users/robbohomebot/scc-content/SCC_V2/`
- Credentials at `/Users/robbohomebot/scc-content/.env` (never commit)

## Running the import

```bash
cd /Users/robbohomebot/scc-content
.venv/bin/python3 scripts/process_competition.py "SEASON" "COMPETITION NAME" --date YYYY-MM-DD --judge "Judge Name"
```

Example:
```bash
.venv/bin/python3 scripts/process_competition.py "2024 - 2025" "DPI 1" --date 2024-10-01 --judge "Jane Smith"
```

Add `--dry-run` to parse and validate without uploading.

## Website API endpoint

```
POST /admin/import-agent/
```
Registered at Wagtail URL: `competitions/import-agent/`

Uses existing helpers:
- `_ensure_competition_category_page`
- `_create_competition_entry_image`
- `_find_member_profile_by_name`

## Competition name mapping (OneDrive folder → website)

| OneDrive folder | Website competition name |
|-----------------|--------------------------|
| Radu Trophy | Radu Handoca - Nature |
| 3 of a kind / Panel | Panel (3 of a Kind) |
| DPI 1 / DPI 2 / DPI 3 | DPI 1 / DPI 2 / DPI 3 |
| Print 1 / Print 2 / Print 3 | Print 1 / Print 2 / Print 3 |
| Annual DPI | Annual DPI |
| Annual Print | Annual Print |

## CHANGELOG rule for SCC_V2

Every commit pushed directly to main on `robinsondan87/SCC_V2` **must** include a version bump in `CHANGELOG.md` (e.g. `## 0.4.0 - YYYY-MM-DD`). The "Release Tag From CHANGELOG" workflow reads the top version and fails if it's already tagged.

Always add a new `## X.Y.Z - YYYY-MM-DD` section in the same commit. Increment patch for fixes, minor for features.

## Status

- Website code committed and pushed to `robinsondan87/SCC_V2` main
- Scripts tested via dry-run against 2024-2025 DPI 1 (92 entries parsed, 24 awards matched)
- Live upload not yet tested — needs website to be redeployed with the new endpoint

## Key reference

| Item | Value |
|------|-------|
| Scripts dir | `/Users/robbohomebot/scc-content/scripts/` |
| OneDrive source | `~/OneDrive/SCC/Competition/Entries/` |
| Website repo | `robinsondan87/SCC_V2` |
| API endpoint | `POST /admin/import-agent/` |
| Website | staffordcameraclub.co.uk |
