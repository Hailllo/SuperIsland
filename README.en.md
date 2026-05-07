<p align="center">
  <img src="assets/logo.png" width="96" height="96" alt="SuperIsland" />
</p>

<h1 align="center">SuperIsland</h1>

<p align="center">
  Turn your Mac's notch into a live, interactive island.<br />
  Now Playing · Battery · Weather · Calendar · Notifications · Extensions
</p>

<p align="center">
  <a href="README.md">中文</a> · English
</p>

---

## Overview

SuperIsland is a macOS desktop app that turns your Mac's notch into a live, interactive island for system status, media playback, notifications, and extensions.

This repository is forked from [`shobhit99/SuperIsland`](https://github.com/shobhit99/SuperIsland), with localization and feature adjustments applied in this fork.

## Requirements

- macOS 14 Sonoma or later
- Xcode 15+
- XcodeGen: `brew install xcodegen`
- Node.js 18+: only needed when developing some extensions

## Setup

```bash
git clone https://github.com/Hailllo/SuperIsland.git
cd SuperIsland
xcodegen generate
open SuperIsland.xcodeproj
```

Select the `SuperIsland` scheme in Xcode, choose your Mac as the destination, then click Run or press `⌘R`.

On first launch, the app will ask for Accessibility, Calendar, Location, and other permissions. These are required for the relevant modules to work.

## Building a release DMG

A release build requires a Developer ID certificate and notarization credentials. Copy `.env.template` to `.env` and fill in:

```env
APPLE_ID=you@example.com
APP_SPECIFIC_PASSWORD=xxxx-xxxx-xxxx-xxxx
TEAM_ID=XXXXXXXXXX
SIGNING_IDENTITY=Developer ID Application: Your Name (TEAMID)
```

Then run:

```bash
./scripts/build-and-release.sh
```

This archives, exports, notarizes, and produces a signed `build/SuperIsland.dmg`.

For a quick unsigned local build:

```bash
./scripts/build-dmg.sh
```

## Project structure

```text
SuperIsland/
  App/              AppDelegate, AppState, app entry point
  Modules/          Built-in modules such as Battery, Now Playing, Weather, etc.
  Settings/         Settings window views
  Utilities/        Update, permissions, launch-at-login, and helpers
  Views/            Compact, expanded, and full-expanded island views
ExtensionHost/      JavaScript extension runtime, manager, and bridge layer
Extensions/         Bundled extensions
scripts/            Build and release scripts
```

## Extensions

Extensions are JavaScript packages that run inside a sandboxed JavaScriptCore context. See:

- [EXTENSIONS.md](EXTENSIONS.md)
- [EXTENSIONS-API.md](EXTENSIONS-API.md)

Currently bundled extensions include:

- Pomodoro Timer
- AI Usage Rings
- Agents Status

## Updates

SuperIsland checks for updates automatically on launch. When a new version is available, an update dialog appears. Click **Update** to download and install it.

The update checker uses GitHub Releases from this repository.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).
