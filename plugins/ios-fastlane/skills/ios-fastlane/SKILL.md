---
description: ios-fastlane skill for RobboHome automation.
---

# Skill: iOS Fastlane Configuration

Standard Fastlane setup for robbohome iOS projects (sideload / paid developer account with HealthKit or other restricted entitlements).

## Standard Appfile
```ruby
app_identifier "com.robbohome.<appname>"
apple_id "3dlabzuk@gmail.com"
```

## Standard Fastfile

**Critical:** `PROJECT` must point to the `.xcodeproj` at the repo root (not inside a subfolder). XcodeGen places it at root.

```ruby
default_platform(:ios)

PROJECT = "<AppName>.xcodeproj"   # root-level, NOT "<AppName>/<AppName>.xcodeproj"
SCHEME  = "<AppName>"

platform :ios do

  desc "Build and install to connected iPhone"
  lane :deploy do
    increment_build_number(
      build_number: Time.now.strftime("%Y%m%d%H%M"),
      xcodeproj: PROJECT
    )
    build_app(
      scheme: SCHEME,
      project: PROJECT,
      configuration: "Release",
      export_method: "development",
      output_directory: "./build",
      output_name: "#{SCHEME}.ipa",
      xcargs: "-allowProvisioningUpdates"
    )
    install_on_device(ipa: "./build/#{SCHEME}.ipa")
  end

  desc "Re-sign and reinstall existing IPA (7-day free cert renewal)"
  lane :resign_and_install do
    sigh(development: true, force: true)
    resign(
      ipa: "./build/#{SCHEME}.ipa",
      signing_identity: "iPhone Developer",
      provisioning_profile: lane_context[SharedValues::SIGH_PROFILE_PATH]
    )
    install_on_device(ipa: "./build/#{SCHEME}.ipa")
  end

end
```

## Standard project.yml signing config (XcodeGen)

Apps using restricted entitlements (HealthKit, etc.) require a paid team at the target level:

```yaml
settings:
  base:
    CODE_SIGN_STYLE: Automatic
    DEVELOPMENT_TEAM: V49U9A6Q5N  # project-level (personal team)

targets:
  <AppName>:
    settings:
      base:
        CODE_SIGN_IDENTITY: "iPhone Developer"
        DEVELOPMENT_TEAM: 9SF4DS367B  # target-level (org/paid team for restricted entitlements)
```

**Note:** After every `make generate`, signing is preserved because it is in project.yml. No need to re-set in Xcode.

## Standard Makefile targets
```makefile
deploy:
	fastlane deploy

resign:
	fastlane resign_and_install

bump-patch:
	@bash scripts/bump-version.sh patch
```

## Prerequisites
- `brew install ios-deploy` — required for `install_on_device`
- `sudo xcode-select -s /Applications/Xcode.app/Contents/Developer` — must be set before first deploy
- iPhone must have **Developer Mode** enabled: Settings > Privacy & Security > Developer Mode
- Open Xcode > Settings > Accounts and sign in once before first `make deploy` — seeds the keychain so xcodebuild can auto-provision

## Troubleshooting
- **"Could not find Xcode project"** — PROJECT path is wrong; XcodeGen puts `.xcodeproj` at repo root, not inside a subfolder
- **"agvtool requires Xcode"** — run `sudo xcode-select -s /Applications/Xcode.app/Contents/Developer`
- **"Apple Generic Versioning not enabled"** — resolved automatically once xcode-select points to Xcode.app
- **"No Account for Team"** — open Xcode > Settings > Accounts, sign in, then retry `make deploy`
- **"No profiles found / Automatic signing disabled"** — add `xcargs: "-allowProvisioningUpdates"` to `build_app` in Fastfile
- **"No devices found"** — check `ios-deploy --detect`, replug iPhone, re-trust on device
- **"Certificate expired"** — run `make resign` (7-day free account limit)
- **Developer Mode not enabled** — app installs but won't open; enable in Settings > Privacy & Security > Developer Mode, restart phone