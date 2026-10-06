---
name: register-runner
description: register-runner skill for RobboHome automation.
---

# Skill: Register GitHub Actions Runner

For server apps using self-hosted CI, provision a project-specific runner on the
approved deployment host. Apps now span Contabo and Unraid; do not assume every
repository belongs on scc_contabo. Inspect the project's actual workflow, runner
registration and deployment skill before creating or enabling another runner.
**Only needed for server apps (Pattern A). iOS apps do not use runners — see `skills/deployment-patterns/SKILL.md`.**

## Current LoopCoach Boundary (Checked 2026-10-06)

LoopCoach's live runner is `vps-scc-loop-coach-v2`, running as `deploy` on
Contabo with generic self-hosted/Linux/X64 labels. Unraid is the intended
destination, not a completed migration. Its two rehearsal stacks remain stopped.
Do not register/enable a competing runner against a generic live workflow or
start those stacks to validate runner setup.

The prepared branch requires explicit host selection, dedicated runner routing
and encrypted recovery capture before release configuration replacement. Read
LoopCoach's `docs/single-instance-deployment.md` before activation. Actual checks
found Python 3.10.12 and age 1.0.0 on Contabo, and no host `python3` command on
Unraid. Provision the destination runner's required toolchain in its managed
environment; do not assume tools from a developer Mac or bare-host package
installs will be present/persistent. Validate access and ownership of its
explicit production/backup paths before allowing deployment jobs.

## Historical Contabo Procedure

The procedure and inventory below are historical examples, not current host,
user or path discovery. For a Contabo runner, verify the current bootstrap and
project layout first — see `skills/server-bootstrap/SKILL.md`.

## Steps

**1. Get a registration token for the repo:**
```bash
gh api repos/robinsondan87/REPO_NAME/actions/runners/registration-token --method POST --jq '.token'
```

**2. On scc_contabo — create a new runner directory and download the runner:**
```bash
ssh scc_contabo '
RUNNER_VERSION=$(curl -s https://api.github.com/repos/actions/runner/releases/latest | grep "\"tag_name\"" | sed "s/.*\"v\(.*\)\".*/\1/")
mkdir -p /home/robbohomebot/actions-runner-REPO_NAME
cd /home/robbohomebot/actions-runner-REPO_NAME
curl -o actions-runner-linux-x64.tar.gz -L \
  "https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz"
tar xzf actions-runner-linux-x64.tar.gz
chown -R robbohomebot /home/robbohomebot/actions-runner-REPO_NAME
'
```

**3. Register the runner:**
```bash
ssh scc_contabo '
cd /home/robbohomebot/actions-runner-REPO_NAME
sudo -u robbohomebot ./config.sh \
  --url "https://github.com/robinsondan87/REPO_NAME" \
  --token "TOKEN_FROM_STEP_1" \
  --name "scc_contabo-REPO_NAME" \
  --labels "robbohome,homeserver" \
  --unattended
'
```

**4. Install and start as a systemd service:**
```bash
ssh scc_contabo '
cd /home/robbohomebot/actions-runner-REPO_NAME
sudo ./svc.sh install robbohomebot
sudo ./svc.sh start
sudo ./svc.sh status
'
```

**5. Verify runner shows as Idle:**
```bash
gh api repos/robinsondan87/REPO_NAME/actions/runners --jq '.runners[]'
```

## Naming convention
- Directory: `/home/robbohomebot/actions-runner-REPO_NAME/`
- Runner name: `scc_contabo-REPO_NAME`
- Systemd service: `actions.runner.robinsondan87-REPO_NAME.scc_contabo-REPO_NAME.service`

## Existing runners on scc_contabo
| Repo             | Directory                          | Runner name                  |
|------------------|------------------------------------|------------------------------|
| robbohome-hello-world | /opt/runners//           | scc_contabo             |
| gym-coach        | /opt/runners/gym-coach/        | scc_contabo-gym         |
| GeekyThingsProductCatalogue | /opt/runners/geekythings/ | scc_contabo-geekythings |

## When NOT to set up a runner
**iOS apps do not use GitHub Actions runners.** They build and deploy locally on the Mac Mini via Fastlane.
No `.github/workflows/` file, no runner registration, no scc_contabo involvement.
See `skills/ios-sideload/SKILL.md` and `skills/ios-fastlane/SKILL.md` for the iOS deploy pattern.

Repos that follow the local-build pattern (no runner needed):
- `gym-coach-health-sync` — iOS app, Fastlane on Mac Mini
