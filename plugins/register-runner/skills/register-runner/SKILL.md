---
description: register-runner skill for RobboHome automation.
---

# Skill: Register GitHub Actions Runner

Each repo needs its own self-hosted runner on scc_contabo.
**Only needed for server apps (Pattern A). iOS apps do not use runners — see `skills/deployment-patterns/SKILL.md`.**

**Prerequisites:** scc_contabo must be bootstrapped first — see `skills/server-bootstrap/SKILL.md`. Personal GitHub accounts cannot share runners across repos.

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
