---
description: xcode-setup skill for RobboHome automation.
---

# Skill: Xcode Setup & iOS Project Creation

One-time setup for iOS development on the Mac Mini.

## Generating an Xcode project with XcodeGen (recommended)
If a `project.yml` exists in the repo, generate the `.xcodeproj` without opening Xcode:
```bash
brew install xcodegen
xcodegen generate
```
Then open the generated `.xcodeproj` in Xcode, select Personal Team, connect iPhone, hit Run.
This is faster than creating a project manually and means all project config lives in source control.

## Install Xcode
Open Mac App Store → search Xcode → Get (~7GB download).
Or via CLI (downloads same package):
```bash
mas install 497799835
```

## Post-install setup
```bash
xcode-select --install          # command line tools
sudo xcodebuild -license accept # accept licence non-interactively
brew install fastlane            # Fastlane for build/deploy automation
brew install libimobiledevice   # idevice_id, idevicesyslog tools
```

## Add Apple ID to Xcode
Xcode → Settings → Accounts → + → Apple ID → sign in with 3dlabzuk@gmail.com

## Create a new iOS project (standard pattern)
1. File → New → Project → iOS → App
2. Settings:
   - Interface: SwiftUI
   - Language: Swift
   - Bundle ID: `com.robbohome.<appname>`
   - Save into existing cloned GitHub repo directory
3. Target → Signing & Capabilities:
   - Team: Personal Team (3dlabzuk@gmail.com)
   - Automatically manage signing: ON
4. Add capabilities as needed (HealthKit, Background App Refresh, etc.)
5. Connect iPhone → Product → Run to register device and verify signing

## Wiring pre-written Swift files into a new Xcode project
When Swift files already exist (e.g. scaffolded before Xcode was installed):
1. In Xcode project navigator, right-click the group → Add Files to Project
2. Select all `.swift` files from the existing directories
3. Ensure "Add to target: <AppName>" is checked
4. Build (Cmd+B) to verify no missing file errors

## Key Info.plist entries for capabilities
```xml
<!-- HealthKit -->
<key>NSHealthShareUsageDescription</key>
<string>Reads health data to sync with your gym coach dashboard.</string>

<!-- Background sync -->
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array>
    <string>com.robbohome.<appname>.dailysync</string>
</array>
```

## Related Skills
- `skills/ios-sideload/SKILL.md` — next step: deploy and re-sign the app on device
- `skills/ios-fastlane/SKILL.md` — standard Fastfile/Appfile templates
- `skills/deployment-patterns/SKILL.md` — when to use iOS vs server deployment

## SDLC pattern for iOS projects
Unlike server apps (Docker + GitHub Actions on svr002), iOS projects:
- Build locally on Mac Mini via Fastlane
- Deploy direct to device via USB/WiFi
- No GitHub Actions runner needed
- VERSION file synced to CFBundleShortVersionString via PlistBuddy in bump-version.sh
- See `skills/ios-sideload/SKILL.md` for deploy and re-signing workflow
