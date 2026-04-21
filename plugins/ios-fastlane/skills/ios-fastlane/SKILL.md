---
description: ios-fastlane skill for RobboHome automation.
---

# Skill: iOS Fastlane Configuration

Standard Fastlane setup for robbohome iOS projects (sideload / free developer account).

## Standard Appfile
```ruby
app_identifier "com.robbohome.<appname>"
apple_id "3dlabzuk@gmail.com"
```

## Standard Fastfile
```ruby
default_platform(:ios)

PROJECT = "<AppName>/<AppName>.xcodeproj"
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
      output_name: "#{SCHEME}.ipa"
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

## Standard Makefile targets
```makefile
deploy:
	fastlane deploy

resign:
	fastlane resign_and_install

bump-patch:
	@bash scripts/bump-version.sh patch
```

## Standard bump-version.sh addition for iOS
Add after the VERSION file write:
```bash
PLIST="<AppName>/Resources/Info.plist"
if [ -f "$PLIST" ]; then
  /usr/libexec/PlistBuddy -c "Set :CFBundleShortVersionString $NEW_VERSION" "$PLIST"
fi
```

## Troubleshooting
- **"No devices found"** — check `idevice_id -l`, replug iPhone, re-trust on device
- **"Provisioning profile doesn't include device"** — run from Xcode once to re-register
- **"Certificate expired"** — run `make resign` (7-day free account limit)
- **"No signing certificate"** — open Xcode → Settings → Accounts → Manage Certificates → + Developer
