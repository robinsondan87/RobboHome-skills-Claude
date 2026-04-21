---
description: Build and deploy GymCoachHealthSync iOS app to connected iPhone via Fastlane.
allowed-tools: Bash(git*) Bash(make*) Bash(fastlane*) Bash(xcodebuild*)
---

# Deploy GymCoachHealthSync

## Patch release
```bash
cd ~/Projects/gym-coach-health-sync
make bump-patch
make deploy
```

## Re-sign only (7-day free cert expiry — no rebuild needed)
```bash
make resign
```

## Check device is connected
```bash
idevice_id -l
```

## View app logs from device
```bash
idevicesyslog 2>/dev/null | grep GymCoach
```

## Test background task in Xcode debugger
```
1. Run app on device via Xcode
2. In Xcode console (paused):
   e -l objc -- (void)[[BGTaskScheduler sharedScheduler] _simulateLaunchForTaskWithIdentifier:@"com.robbohome.gymcoachhealthsync.dailysync"]
3. Resume — task fires immediately
```

---

## Key reference

| Item | Value |
|------|-------|
| Project path | `~/Projects/gym-coach-health-sync` |
| Repo | `robinsondan87/gym-coach-health-sync` (private) |
| Bundle ID | `com.robbohome.gymcoachhealthsync` |
| Background task ID | `com.robbohome.gymcoachhealthsync.dailysync` |
| Apple ID | `3dlabzuk@gmail.com` (free account, 7-day cert) |
| API endpoint | `POST https://gymcoach.robbohome.com/api/health/sync/day` |
| Auth | Bearer token: `c6aef2431dd8d45a19322b0eb0056d4c420f5e91f234a25012a51fb06e39177a` |

## Notes
- Free Apple Developer account: certs expire every 7 days — run `make resign` to renew
- Monday 9am openclaw cron is set as a 7-day renewal reminder
- No GitHub Actions runner — builds locally on Mac via Fastlane
- See `skills/ios-fastlane/SKILL.md` for Fastfile/Appfile templates
- See `skills/xcode-setup/SKILL.md` for first-time Xcode project setup
