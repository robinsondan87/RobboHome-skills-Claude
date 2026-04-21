---
description: ios-sideload skill for RobboHome automation.
---

# Skill: iOS Sideload (Free Developer Account)

Sideloading iOS apps to a personal iPhone using a free Apple Developer account via Fastlane.
No paid account needed — but apps expire every 7 days and must be re-signed.

## Prerequisites
- Xcode installed (Mac App Store, free, ~7GB)
- `brew install fastlane`
- iPhone plugged in via USB and trusted on Mac
- Free Apple ID registered at developer.apple.com (use 3dlabzuk@gmail.com)

## First-time device registration
Open Xcode, plug in iPhone, go to Window → Devices and Simulators.
Xcode will automatically register the device UDID with your Apple account.
Then do a first "Product → Run" from Xcode to verify signing works.

## Standard deploy (build + install)
```bash
cd ~/Projects/<app-name>
make deploy
# or directly:
fastlane deploy
```

## Re-sign only (7-day cert renewal — no rebuild)
Every 7 days the free developer certificate expires. Re-sign without rebuilding:
```bash
make resign
# or directly:
fastlane resign_and_install
```

## Check device is connected
```bash
idevice_id -l
# or
xcrun xctrace list devices 2>&1 | grep -v Simulator
```

## View live device logs
```bash
idevicesyslog 2>/dev/null | grep <AppName>
```

## 7-day renewal reminder
A Monday 9am openclaw cron is set as a reminder to re-sign apps before they expire.
Run `make resign` when prompted.

## Upgrade to paid account
When ready to remove the 7-day limit:
1. Enroll at developer.apple.com/programs ($99/year)
2. In Xcode → Settings → Accounts, upgrade team to paid
3. Re-build once — app will no longer expire

## Related Skills
- `skills/xcode-setup/SKILL.md` — first-time Xcode install and project creation (do this first)
- `skills/ios-fastlane/SKILL.md` — standard Fastfile/Appfile templates
- `skills/deployment-patterns/SKILL.md` — iOS vs server app pattern overview

## Existing iOS projects
| App | Repo | Bundle ID | Build path |
|-----|------|-----------|------------|
| GymCoachHealthSync | robinsondan87/gym-coach-health-sync | com.robbohome.gymcoachhealthsync | ~/Projects/gym-coach-health-sync/build/ |
