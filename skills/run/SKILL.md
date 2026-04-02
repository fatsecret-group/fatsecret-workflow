---
name: run
description: "Build and run the app. Default: booted simulator. Use `device` for physical device. Use `--delete` for clean install. Specify simulator name to target a specific simulator (e.g. `iPhone 16 Pro`)."
disable-model-invocation: true
allowed-tools: Bash
---

Build and run the app on simulator (default) or physical device.

Arguments: `$ARGUMENTS`

## Configuration

Set these per project in `.claude/skills/run/SKILL.md` or CLAUDE.md:

- **Workspace**: `Calorie Counter.xcworkspace`
- **Scheme**: `Calorie Counter`
- **Bundle ID**: `com.fatsecret.caloriecounter`

## Steps

### 1. Determine target

- If `$ARGUMENTS` contains `device` → **device mode**
- Otherwise → **simulator mode**
  - If `$ARGUMENTS` contains a simulator name (e.g. `iPhone 16 Pro`, `iPhone Air`, `iPad Pro`) → use that specific simulator
  - Otherwise → use the currently booted simulator, or boot the first available iPhone

### 2. Simulator mode

1. If a simulator name was specified in `$ARGUMENTS`, find its UDID: `xcrun simctl list devices available | grep "<name>"` and boot it if not already booted: `xcrun simctl boot <UDID>; open -a Simulator`
2. Otherwise, find a booted simulator: `xcrun simctl list devices booted | grep -i "iphone\|ipad"`
3. If none booted, list available simulators and boot the first iPhone: `xcrun simctl list devices available | grep -i iphone` then `xcrun simctl boot <UDID>; open -a Simulator`
4. Use the resolved simulator's UDID as `$SIM_ID`
4. If `$ARGUMENTS` contains `--delete`: `xcrun simctl uninstall $SIM_ID com.fatsecret.caloriecounter 2>/dev/null`
5. Build: `xcodebuild -workspace "Calorie Counter.xcworkspace" -scheme "Calorie Counter" -destination "id=$SIM_ID"`. Check for BUILD SUCCEEDED.
6. Get app path: `xcodebuild -workspace "Calorie Counter.xcworkspace" -scheme "Calorie Counter" -destination "id=$SIM_ID" -showBuildSettings | grep -m1 "BUILT_PRODUCTS_DIR"` then append `/Calorie Counter.app`
7. Install: `xcrun simctl install $SIM_ID "<APP_PATH>"`
8. Launch: `xcrun simctl launch $SIM_ID com.fatsecret.caloriecounter`

### 3. Device mode

1. Find connected device: `xcrun devicectl list devices 2>/dev/null | grep -i connected`
2. Use the device UDID as `$DEVICE_ID`
3. If `$ARGUMENTS` contains `--delete`: `xcrun devicectl device uninstall app --device $DEVICE_ID com.fatsecret.caloriecounter 2>/dev/null`
4. Build: `xcodebuild -workspace "Calorie Counter.xcworkspace" -scheme "Calorie Counter" -destination "id=$DEVICE_ID" -allowProvisioningUpdates`. Check for BUILD SUCCEEDED.
5. Get app path: `xcodebuild -workspace "Calorie Counter.xcworkspace" -scheme "Calorie Counter" -destination "id=$DEVICE_ID" -showBuildSettings | grep -m1 "BUILT_PRODUCTS_DIR"` then append `/Calorie Counter.app`
6. Install: `xcrun devicectl device install app --device $DEVICE_ID "<APP_PATH>"`
7. Launch: `xcrun devicectl device process launch --device $DEVICE_ID com.fatsecret.caloriecounter`
