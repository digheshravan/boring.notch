# Progress log — MyNotch

Claude Code updates this file at the end of every session. Newest entry on
top. Keep entries short and factual.

## Current status

- **Phase:** 0A done; 0B not started
- **Branch:** `phase-0-base` (pushed to `origin`)
- **Last checkpoint passed:** 0A (2026-11-14)
- **Next action:** Phase 0B (project structure), when the owner starts it
- **Blocked on:** nothing

## Session template (copy for each session)

```
### YYYY-MM-DD — Phase N<letter> — <one-line summary>
Done:
- ...
In progress:
- ...
Spike results (also recorded in DECISIONS.md):
- Qn: ...
Files added / changed:
- ...
Upstream files touched (must be tagged // MYNOTCH:):
- ...
How to test by hand:
1. ...
Known issues:
- ...
Next:
- ...
```

## Sessions

### 2026-11-14 — Pre-0B — Notch hides over full-screen apps on every display
Owner request: notch visible on every display on the normal desktop
(MacBook and extended monitor), hidden on a display while an app is full
screen there, shown on hover.
Done:
- First attempt misread the request (hid the notch on non-notch displays
  all the time); reverted in `6d2058c`.
- Upstream already supports the wanted behaviour through Settings > Media >
  "Full screen behavior". Its default, "Hide for media app only", only hides
  when the full-screen app is the music source. MyNotch's default is now
  "Hide for all apps". One-word change, tagged.
- Built, relaunched; external display in a full-screen browser shows no
  notch bar.
Upstream files touched (tagged // MYNOTCH:):
- `boringNotch/models/Constants.swift`
How to test by hand:
1. Settings > General: "Show on all displays" on.
2. Normal desktop: notch visible on the MacBook and on the external monitor.
3. Make any app full screen on the external monitor (green button): the
   notch disappears there; hover the top centre: it opens; move away: it
   hides again. The MacBook screen still shows its notch.
4. Leave full screen: the notch comes back on the external monitor.
5. Repeat on the MacBook screen with a full-screen app.
6. Settings > Media > "Full screen behavior" shows "Hide for all apps".
Known issues:
- Upstream marks full-screen detection as Beta.
- Upstream draws a 1-pixel black line where the hidden notch sits on a
  non-notch display.

### 2026-11-14 — Phase 0A — Fork runs under the MyNotch identity
Done:
- Forked to `digheshravan/boring.notch`; repo set up in this folder with
  `origin` (fork) and `upstream`; branches `mynotch/main`, `phase-0-base`.
  Planning docs moved to the repo root and committed.
- Checked CLAUDE.md upstream facts against the checkout. All held; small
  corrections written into CLAUDE.md: `BoringNotchXPCHelper/` is also a
  synced folder; upstream has no `os.Logger`; the XPC helper's service name
  was hard-coded in `XPCHelperClient.swift`; Sparkle UI is in 3 Swift files
  (`SparkleView.swift` is only a particle effect, not the updater).
- Built upstream unchanged: BUILD SUCCEEDED (93 upstream warnings, not
  fixed). First package resolve failed on a corrupt Sparkle/Lottie download;
  a retry fixed it.
- Renamed identity: `com.shravandige.mynotch`, "MyNotch", helper
  `com.shravandige.mynotch.XPCHelper`.
- Disabled Sparkle (never started, feed removed, menu and settings UI removed).
- Added `NOTICE.md`.
- Sandbox off; rebuilt; entitlements verified in the signed app.
- Stable signing: app and XPC helper signed "Apple Development", team
  `5ZA2QGQH96` (owner's Personal Team), set in `MyNotch.xcconfig`. Verified
  with `codesign --verify --deep --strict` after a clean build.
- Checkpoint 0A passed: owner ran the full hand test, everything passed.
In progress:
- Nothing.
Spike results (also recorded in DECISIONS.md):
- Q1: passed. Every upstream feature on the checklist works with the
  sandbox off; ADR-003 stands.
Files added / changed:
- Added: `Configuration/MyNotch.xcconfig`, `NOTICE.md`, `CLAUDE.md`,
  `PROMPTS.md`, `docs/`.
Upstream files touched (must be tagged // MYNOTCH:):
- `boringNotch.xcodeproj/project.pbxproj` (cannot hold lasting comments;
  listed in NOTICE.md instead)
- `boringNotch/models/Constants.swift`
- `boringNotch/XPCHelperClient/XPCHelperClient.swift`
- `boringNotch/boringNotchApp.swift`
- `boringNotch/components/Settings/SettingsView.swift`
- `boringNotch/Info.plist` (XML comment)
- `boringNotch/boringNotch.entitlements` (XML comment)
How to test by hand (Checkpoint 0A + spike Q1):
1. Quit the real Boring Notch (menu bar icon > Quit) so two notches do not
   overlap while testing.
2. Run MyNotch: open `boringNotch.xcodeproj` in Xcode, scheme `boringNotch`,
   My Mac, Cmd+R. Or run the last build from Terminal (command in the
   session summary).
3. Identity: onboarding appears (new identity = fresh settings). Activity
   Monitor shows a process named "MyNotch".
4. Updater: menu bar icon menu has Settings, Restart, Quit, and no
   "Check for Updates…". Settings > About has no "Software updates"
   section and no update button.
5. Q1 checklist, mark each pass / fail / partly:
   - [ ] Notch opens on hover and closes when the pointer leaves
   - [ ] Music: play in Apple Music, then Spotify if installed. Title, art
         and play/pause/next work from the notch (Apple Events permission)
   - [ ] Music sneak peek when the track changes with the notch closed
   - [ ] Calendar: enable in Settings, grant access, events show
   - [ ] Reminders appear in the calendar view (if enabled)
   - [ ] Camera mirror: enable in Settings, grant camera, preview shows
   - [ ] Volume / brightness HUD replacement: enable, grant Accessibility
         (System Settings > Privacy & Security > Accessibility), press the
         volume and brightness keys
   - [ ] Shelf: drag a file from Finder onto the notch; it appears in the
         shelf; drag it back out to the Desktop
   - [ ] Shelf persistence: Restart MyNotch from the menu; the shelf item
         is still there
   - [ ] Shelf AirDrop / share: share a shelf item via the share button
   - [ ] Battery indicator: unplug and replug the charger; indicator updates
   - [ ] Lock screen: lock the Mac (Ctrl+Cmd+Q); notch behaves as before
         (no crash, live activity shows if enabled); unlock
   - [ ] Download indicator: download a file in Safari; activity shows
   - [ ] Lyrics (if enabled): lyrics load for a playing song
   - [ ] Launch at login toggle can be switched on and off
   - [ ] Real Boring Notch: start it again afterwards; its settings and
         shelf are unchanged
Known issues:
- After changing signing settings, do a clean build (Xcode: Product >
  Clean Build Folder, Shift+Cmd+K); an incremental build kept the old
  helper signature inside the app.
- The XPC helper target has no Hardened Runtime (upstream setting; only
  matters for notarised releases).
- Permissions granted to earlier ad-hoc builds do not carry over; grant
  them once more to the signed build.
- In-app text still says "Boring Notch" / "boring.notch" in places (menu
  bar menu title, "Restart Boring Notch", Settings window title, onboarding
  text; about 108 matches in Swift files). Only the bundle identity was in
  scope for 0A. Owner to decide when to rename visible text (it goes through
  localisation).
- With the sandbox off, the shelf saves to
  `~/Library/Application Support/boringNotch/Shelf/items.json` (folder name
  hard-coded upstream). It does not clash with the real Boring Notch, which
  is sandboxed and stores its data in its container.
- Sandbox-only entitlement keys left in place (inert) to keep the diff small.
Next:
- Owner: hand test above; record Q1 results in DECISIONS.md spike log.
- Claude Code: Phase 0B after Checkpoint 0A is approved.

### 2026-11-14 — Planning
Done:
- Planning set written (PRD, architecture, module system, hook protocol,
  security, decisions, roadmap, setup, module specs, prompts).
- Upstream boring.notch v2.7.3 inspected; integration points recorded in
  ARCHITECTURE.md and CLAUDE.md.
- Claude Code hooks, Codex hooks and LocalSend protocol v2 checked against
  official documentation; Antigravity and Cursor left as UNVERIFIED.
Next:
- Owner: SETUP.md steps 1-4.
- Claude Code: Phase 0A.

## Owner feedback log

Record each review round here: date, phase, what the owner said, what was
changed in response.

| Date | Phase | Feedback | Action |
|---|---|---|---|
| 2026-11-14 | 0A | Hand test: everything passed. | Q1 recorded as passed; Checkpoint 0A closed. |
| 2026-11-14 | 0A | Notch should be visible on every display on the desktop, and hidden on a display while an app is full screen there (show on hover). First read wrongly as "always hide on non-notch displays". | Wrong change reverted (`6d2058c`). Default "Full screen behavior" set to "Hide for all apps" (`ca27972`). |

## Ideas parking lot

Anything that comes up mid-phase and is not in the roadmap goes here, not
into the code.

- KDE Connect-compatible provider (clipboard sync, phone notifications).
- Android companion app for phone-side notification mirroring.
- Usage / plan-limit display for agents.
- Per-project agent grouping.
- Model/effort router integration (owner's separate project idea).
