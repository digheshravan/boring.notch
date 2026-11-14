# NOTICE

MyNotch (working name) is a modified version of
[boring.notch](https://github.com/TheBoredTeam/boring.notch) by TheBoredTeam
and contributors, forked from upstream `main` at v2.7.3
(commit `b7611bb`, 2026-10-31).

boring.notch is licensed under the GNU General Public License v3.0, and so is
MyNotch. See [`LICENSE`](LICENSE). Upstream copyright notices in source files
are kept unchanged. Third-party components bundled by upstream keep their own
licences; see [`THIRD_PARTY_LICENSES`](THIRD_PARTY_LICENSES).

MyNotch is not affiliated with or endorsed by TheBoredTeam.

## How changes are marked

- Every edit to an upstream source or plist file carries a `MYNOTCH:` comment
  (`// MYNOTCH:` in Swift, `<!-- MYNOTCH: -->` in plists) explaining why.
- `boringNotch.xcodeproj/project.pbxproj` cannot hold comments that survive
  Xcode saving it, so its changes are listed below instead.
- New MyNotch code lives in its own files (listed below) and, from Phase 0B,
  in `boringNotch/NotchModules/` and `Packages/NotchCore/`.

## Changes from upstream

### 2026-11-14 — Phase 0A

**Identity (ADR-008, ADR-010)**
- New file `Configuration/MyNotch.xcconfig`: app name `MyNotch`, bundle id
  `com.shravandige.mynotch`. Used as the project-level base configuration.
- `project.pbxproj`: app bundle id, product name and display name now come
  from the xcconfig; XPC helper bundle id is `$(MYNOTCH_BUNDLE_ID).XPCHelper`.
- `boringNotch/models/Constants.swift`: added `MyNotchBrand` (code-side names:
  logger subsystem `com.mynotch`, `~/.mynotch`, XPC helper service name).
- `boringNotch/XPCHelperClient/XPCHelperClient.swift`: helper service name is
  derived from the app's bundle id instead of being hard-coded.

**Auto-update disabled (ADR-008)**
- `boringNotch/boringNotchApp.swift`: Sparkle updater is never started;
  "Check for Updates…" removed from the menu bar menu.
- `boringNotch/components/Settings/SettingsView.swift`: "Software updates"
  toggles and the "Check for Updates…" toolbar button removed from About.
- `boringNotch/Info.plist`: upstream feed URL, signing key and sandboxed
  installer keys removed; automatic checks off.
- `boringNotch/boringNotch.entitlements`: Sparkle installer mach-lookup
  exceptions removed.

**App sandbox off (ADR-003)**
- `boringNotch/boringNotch.entitlements`: `com.apple.security.app-sandbox`
  set to false. Other entitlements unchanged.

**Documentation**
- Added `CLAUDE.md`, `PROMPTS.md`, `docs/` (MyNotch planning set) and this
  file.
