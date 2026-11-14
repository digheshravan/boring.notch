# CLAUDE.md — MyNotch

MyNotch (working name) is a personal, open-source fork of
[TheBoredTeam/boring.notch](https://github.com/TheBoredTeam/boring.notch)
(Swift / SwiftUI / AppKit, GPL-3.0). It turns the MacBook notch into a modular
productivity surface where **every feature is a module the user can switch on
or off**, with Mac + Android users treated as first-class.

Read these before writing code, in this order:

1. `docs/PRD.md` — what we are building and why
2. `docs/ARCHITECTURE.md` — how it fits into the boring.notch codebase
3. `docs/ROADMAP.md` — phases, tasks, acceptance criteria
4. `docs/DECISIONS.md` — decisions already made (do not silently reverse them)
5. `docs/PROGRESS.md` — where the last session stopped
6. The spec for the module you are working on (`docs/MODULE_SYSTEM.md`,
   `docs/modules/*.md`, `docs/HOOK_PROTOCOL.md`, `docs/SECURITY.md`)

## How we work (non-negotiable)

- **One phase at a time.** Work only on the phase named in the prompt. Do not
  start the next phase, even if the current one finishes early.
- **Small steps, always building.** After every meaningful change: build, run
  the package tests, fix errors before moving on. Never leave the tree broken.
- **Stop at review checkpoints.** Each phase in `docs/ROADMAP.md` ends with a
  checkpoint. Stop there, summarise what changed, list how the owner can test
  it by hand, and wait for feedback.
- **Spikes before features.** Items marked `SPIKE` or `UNVERIFIED` in the docs
  are not facts. Test them on the real machine first, write the result into
  `docs/DECISIONS.md`, then build.
- **Update the docs as you go.** At the end of every session update
  `docs/PROGRESS.md` (done / in progress / next / blockers). If a decision
  changes, add a new ADR; do not edit history.
- **Ask when a decision is the owner's.** Naming, UX trade-offs, anything
  touching security rules, anything that deletes or rewrites the owner's
  config files outside the repo.
- **No scope creep.** Do not add features that are not in the roadmap. Put
  ideas in the "Ideas parking lot" section of `docs/PROGRESS.md`.

## Hard rules

1. **Never auto-approve an agent permission.** Every "allow" is a human click.
   See `docs/SECURITY.md`.
2. **Fail open to the agent's normal prompt.** If MyNotch is not running,
   crashes, or times out, the agent must behave exactly as if MyNotch did not
   exist. Never fail into "allow". Never hang an agent.
3. **Never edit the owner's agent config without a backup.** Before touching
   `~/.claude/settings.json`, `~/.codex/hooks.json`, `~/.cursor/hooks.json`
   or Antigravity config: copy to `<file>.mynotch-backup-<timestamp>`, merge
   (never overwrite), and make uninstall restore cleanly.
4. **A disabled module does nothing.** No timers, sockets, observers, network
   or permission prompts while its switch is off.
5. **No telemetry, no network calls** except those a module needs and
   documents (e.g. LocalSend on the local network).
6. **Keep upstream files as untouched as possible.** New code goes in
   `boringNotch/NotchModules/` and `Packages/NotchCore/`. When an upstream
   file must change, keep the edit minimal and mark it:
   `// MYNOTCH: <why>`. This keeps upstream merges cheap.
7. **GPL-3.0 stays.** Keep `LICENSE`, keep upstream copyright headers, and
   note changes in `NOTICE.md`. Code copied from other projects must be
   GPL-compatible (MIT/Apache-2.0 are fine) and credited in
   `THIRD_PARTY_LICENSES`.

## Repo layout (after Phase 0)

```
boringNotch/                 upstream app sources (touch minimally)
  NotchModules/              OUR UI + module glue (Xcode synced folder)
    Core/                    ModuleRegistry, NotchPresenter, settings UI
    Agents/                  Agents module
    SendToDevice/            Send-to-device module
    NotificationMirror/      Notification mirror module
    Legacy/                  wrappers exposing upstream features as modules
Packages/NotchCore/          OUR logic as a local Swift package (testable)
  Sources/NotchCore/         models, wire protocol, agent adapters, risk
                             classifier, store reducers, arbiter, LocalSend
  Sources/notch-hook/        tiny CLI the agents run as their hook
  Tests/NotchCoreTests/
docs/                        this planning set
scripts/                     build / install helpers
```

Rule of thumb: **if it can be tested without a window, it lives in
`Packages/NotchCore`.** SwiftUI views and AppKit glue live in
`boringNotch/NotchModules`.

## Commands

```bash
# Discover scheme names first (the project ships no shared schemes)
xcodebuild -list -project boringNotch.xcodeproj

# Build the app
xcodebuild -project boringNotch.xcodeproj -scheme boringNotch \
  -configuration Debug -destination 'platform=macOS' build

# Logic tests (fast, run these constantly)
swift test --package-path Packages/NotchCore

# Build + install the hook client for local use
swift build -c release --package-path Packages/NotchCore --product notch-hook
./scripts/install-hook-client.sh      # copies to ~/.mynotch/bin/notch-hook

# Simulate an agent event without running an agent
echo '<json>' | ~/.mynotch/bin/notch-hook --agent claude-code
```

If a build fails, read the full error, fix the cause, rebuild. Do not paper
over errors with `// swiftlint:disable`, force unwraps, or deleted code.

## Facts about the upstream codebase (verified 2026-11-14, v2.7.3, `main`; re-checked against the checkout in Phase 0A)

- Deployment target macOS 14.0; **building needs macOS 15.6+ and Xcode 26+**.
  Confirmed building on macOS 27.2 / Xcode 27.2.
- Two targets: `boringNotch` (app; sandboxed upstream, **sandbox off in the
  fork since 0A**, ADR-003) and `BoringNotchXPCHelper` (unsandboxed XPC
  service: accessibility + brightness). Scheme name: `boringNotch`.
- No test target and no shared schemes.
- Most source folders are classic Xcode groups: **a new `.swift` file is not
  compiled unless it is registered in `project.pbxproj`.** Synced folders
  (`PBXFileSystemSynchronizedRootGroup`): `boringNotch/private` and the
  whole `BoringNotchXPCHelper/` folder. We add `boringNotch/NotchModules`
  as a second synced folder once (Phase 0), then never hand-edit the pbxproj
  for new files again.
- Settings are stored with the `Defaults` package; keys live in
  `boringNotch/models/Constants.swift`.
- Tabs: `NotchViews` enum in `boringNotch/enums/generic.swift`, `tabs` array
  in `boringNotch/components/Tabs/TabSelectionView.swift`, switch in
  `boringNotch/ContentView.swift` (`switch coordinator.currentView`).
- Open/close: `BoringViewModel.open()` / `.close()`; one view model per
  screen in `AppDelegate.viewModels` (`boringNotch/boringNotchApp.swift`).
- Closed-notch "sneak peek" and expanding views:
  `BoringViewCoordinator.toggleSneakPeek` / `toggleExpandingView`.
- Open notch size is fixed at 640x190 (`boringNotch/sizing/matters.swift`).
- Share menu services (AirDrop, and any app that adds itself to the share
  menu): `boringNotch/components/Shelf/Services/QuickShareService.swift` and
  `ShareServiceFinder.swift`.
- Sparkle auto-update pointed at the upstream feed. **Disabled in the fork
  in 0A** (never started, feed removed, UI removed); the package is still
  linked. Do not re-enable it until MyNotch has its own feed.
- Identity: the app name and bundle id live in `Configuration/MyNotch.xcconfig`
  (project-level base config); code-side names in `MyNotchBrand`
  (`boringNotch/models/Constants.swift`). The XPC helper's id is
  `<bundle id>.XPCHelper` and the client derives it at runtime.
- Upstream has no `os.Logger` usage; MyNotch code uses subsystem
  `MyNotchBrand.loggerSubsystem` (`com.mynotch`).
- Debug builds are ad-hoc signed until a signing team is set; Hardened
  Runtime is not applied to ad-hoc Debug builds.

## Code style

- Swift 5 language mode, Swift concurrency (`async/await`, `@MainActor` for
  UI state). Match the surrounding upstream style in upstream files.
- No force unwraps or `try!` in our code. Errors are logged with `os.Logger`
  (subsystem `com.mynotch`, one category per module) and surfaced in the UI
  where the user can act on them.
- Every user-facing string goes through `String(localized:)`.
- Public types in `NotchCore` get a one-line doc comment. No essay comments.
- Tests for every adapter, codec, reducer and classifier in `NotchCore`.
  Use recorded real payloads as fixtures (`Tests/NotchCoreTests/Fixtures`).

## Owner context

The owner knows Flutter/Dart, React and Python well; **Swift and Xcode are new
to him.** When you finish a step, explain what you did in plain language,
name the files, and give exact manual test steps. Prefer the simple design
over the clever one. He is security-focused: call out anything that widens
the attack surface.
