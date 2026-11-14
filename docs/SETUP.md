# Setup — MyNotch

Things the owner does once, by hand, before the first Claude Code session.
About 20 minutes plus the Xcode download.

## 1. Requirements

- macOS 15.6 or later (required to build upstream boring.notch).
- Xcode 26 or later from the App Store. Open it once and let it install the
  macOS components.
- Command line tools: `xcode-select -p` should print a path inside
  Xcode.app. If not: `sudo xcode-select -s /Applications/Xcode.app`.
- Git and a GitHub account. `gh` CLI is optional but handy.
- Claude Code installed and signed in.
- Free Apple ID added in Xcode > Settings > Accounts (gives you an "Apple
  Development" signing certificate; no paid developer account needed to run
  on your own Mac).

Check:

```bash
sw_vers -productVersion        # 15.6 or higher
xcodebuild -version            # Xcode 26.x
swift --version
git --version
```

## 2. Fork and clone

1. On GitHub open https://github.com/TheBoredTeam/boring.notch and click
   **Fork**. Leaving "Copy the main branch only" ticked is fine; we only
   need `main`.
2. Then:

```bash
cd ~/Developer            # or wherever you keep projects
git clone https://github.com/<your-username>/boring.notch.git mynotch
cd mynotch
git remote add upstream https://github.com/TheBoredTeam/boring.notch.git
git fetch upstream
git checkout -b mynotch/main
git checkout -b phase-0-base
```

(Renaming the GitHub repo to your final project name can wait.)

## 3. Add the planning files

Copy the contents of the planning zip into the repo root so you get:

```
mynotch/
  CLAUDE.md
  PROMPTS.md
  docs/
    PRD.md  ARCHITECTURE.md  MODULE_SYSTEM.md  HOOK_PROTOCOL.md
    SECURITY.md  DECISIONS.md  ROADMAP.md  SETUP.md  PROGRESS.md
    modules/AGENTS.md  SEND_TO_DEVICE.md  NOTIFICATION_MIRROR.md
```

Upstream already has `README.md`, `CONTRIBUTING.md` and `SECURITY.md` at the
root. Leave them for now; they are replaced before a public release.

```bash
git add CLAUDE.md PROMPTS.md docs
git commit -m "Add MyNotch planning docs"
```

## 4. Build upstream once, unchanged

```bash
open boringNotch.xcodeproj
```

In Xcode:

1. Wait for "Resolving package dependencies" to finish (first time takes a
   few minutes).
2. Select the `boringNotch` scheme and "My Mac".
3. Click the project in the sidebar > target **boringNotch** > **Signing &
   Capabilities** > Team: choose your Personal Team. Do the same for target
   **BoringNotchXPCHelper**.
4. Press Cmd+R.

The app has no Dock icon (it is a menu-bar/notch app). Hover the notch.
If it works, quit it from its menu-bar icon. If you already use the real
boring.notch, quit that first; the two share an identity until Phase 0
renames ours.

If the build fails here, stop and fix this first (usually signing or an old
Xcode). Give Claude Code the exact error text.

## 5. Start Claude Code

```bash
cd ~/Developer/mynotch
claude
```

Paste the Phase 0 kickoff prompt from `PROMPTS.md`.

## 6. One-time Xcode step Claude Code may ask you to do

If Claude Code cannot safely edit the project file to add the new folder
and package (ROADMAP 0B), do it in Xcode:

- **Add the local package:** File > Add Package Dependencies... > Add
  Local... > choose `Packages/NotchCore` > Add Package > tick library
  `NotchCore` for target `boringNotch`.
- **Add the synced folder:** in the sidebar right-click the `boringNotch`
  group > Add Files to "boringNotch"... > select the folder
  `boringNotch/NotchModules` > Action: "Reference files in place", Groups:
  **"Create folders"** (this makes it a synced folder; its icon is blue) >
  Targets: tick `boringNotch` > Finish. (Dialog labels can differ slightly
  between Xcode versions; the goal is a blue folder, not a yellow group.)

After that, any file created inside `boringNotch/NotchModules` is compiled
automatically.

## 7. Permissions you will be asked for (and when)

| Prompt | When | Why |
|---|---|---|
| Accessibility | upstream HUD replacement; later notification mirror | read/replace system HUD; read banners |
| Calendar, Camera, Apple Events | upstream features | calendar view, mirror, music control |
| Local Network | Phase 3, LocalSend provider | find and reach your phone on Wi-Fi |
| Full Disk Access | only if you opt into the "replace banners" mirror mode | read the notification database |

Grant only what the module you are testing needs.

## 8. Phone setup (Phase 3)

- Blip: already installed. Make sure the Mac app is signed in and "Blip"
  shows in Finder's right-click > Share menu.
- LocalSend: install from the Play Store (or F-Droid), open it, note the
  device name. Phone and Mac on the same Wi-Fi; some college/hostel
  networks block device-to-device traffic (client isolation). If the phone
  never appears, test on a home network or the phone's hotspot.

## 9. Keeping up with upstream (optional, later)

```bash
git fetch upstream
git checkout mynotch/main
git merge upstream/main      # resolve conflicts in files tagged // MYNOTCH:
```

Do this between phases, never in the middle of one.
