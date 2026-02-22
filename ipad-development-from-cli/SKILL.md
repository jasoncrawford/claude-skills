---
name: ipad-development-from-cli
description: Use when setting up an iOS or iPad project from command line, using xcodegen to generate Xcode projects, managing iOS simulator runtimes, encountering test resource loading issues with Bundle.main, or deploying to physical iPad devices
---

# iPad/iOS Development from CLI

## Overview

Reference for building iPad/iOS apps from the command line using xcodegen, with solutions to common gotchas around resource bundling, simulator setup, test configuration, and device deployment.

## xcodegen Project Setup

Install: `brew install xcodegen`

**Critical resource bundling gotcha:** Don't separate `sources` and `resources` with excludes. Let xcodegen auto-detect JSON/assets as resources:

```yaml
# BAD - resources won't bundle in app
sources:
  - path: MyApp
    excludes: [Resources]
resources:
  - path: MyApp/Resources

# GOOD - auto-detects JSON/assets as resources
sources:
  - MyApp
```

**iPad-only settings:**

```yaml
settings:
  base:
    TARGETED_DEVICE_FAMILY: "2"
    SWIFT_VERSION: "6.0"
    INFOPLIST_KEY_UILaunchScreen_Generation: true
```

**Test targets** need `GENERATE_INFOPLIST_FILE: true` in settings or builds fail with missing Info.plist.

Generate project: `xcodegen generate`

## iOS Simulator Runtime

Xcode 26+ requires downloading the iOS platform separately (~8.4 GB). Start this early:

```bash
xcodebuild -runFirstLaunch          # Initialize CLI tools first
xcodebuild -downloadPlatform iOS    # Long download
```

## CLI Build & Test Commands

```bash
# Build
xcodebuild build -scheme MyScheme -destination 'platform=iOS Simulator,name=iPad (A16)'

# Test
xcodebuild test -scheme MyScheme -destination 'platform=iOS Simulator,name=iPad (A16)'

# Simulator management
xcrun simctl boot 'iPad (A16)' && open -a Simulator
xcrun simctl install 'iPad (A16)' path/to/Build/Products/Debug-iphonesimulator/MyApp.app
xcrun simctl launch 'iPad (A16)' com.example.myapp
xcrun simctl terminate 'iPad (A16)' com.example.myapp
```

## Test Resource Loading

`Bundle.main` and `Bundle(for:)` don't reliably find app resources in hosted test targets. Use `#filePath` to locate files relative to the source tree:

```swift
private func loadResource(_ relativePath: String) throws -> Data {
    let testFile = URL(fileURLWithPath: #filePath)
    let projectRoot = testFile.deletingLastPathComponent().deletingLastPathComponent()
    let url = projectRoot.appendingPathComponent(relativePath)
    return try Data(contentsOf: url)
}

// Usage: let data = try loadResource("MyApp/Resources/data.json")
```

## Physical iPad Deployment

1. Connect iPad via USB, tap "Trust This Computer" on iPad
2. Xcode: target > Signing & Capabilities > Automatically manage signing > select Apple ID team
3. First install: trust dev cert on iPad at Settings > General > VPN & Device Management
4. Free Apple ID works — no paid developer account needed

If iPad shows "unavailable/unpaired": unplug, replug, watch for trust dialog. If still stuck, reset Location & Privacy settings on iPad.

## SwiftUI Readability Patterns

| Problem | Solution |
|---------|----------|
| Poor contrast on colorful backgrounds | Dark scrim: `.background(RoundedRectangle(cornerRadius: 24).fill(.black.opacity(0.4)))` |
| Text jumps during typewriter animation | Left-align and anchor: `.frame(maxWidth: .infinity, alignment: .leading)` |
| Content overflows screen | `ScrollView` with `.scrollBounceBehavior(.basedOnSize)` |
| Centered text shifts as words appear | Use `.multilineTextAlignment(.leading)`, never `.center` with animated text |

## Swift 6 / @Observable

Use `@Observable class` (not ObservableObject) with `@State` in views (not @StateObject). Works cleanly with SwiftUI in iPadOS 26+.
