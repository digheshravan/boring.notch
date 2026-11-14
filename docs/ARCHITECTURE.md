# Architecture — MyNotch

Verified against upstream `main` (v2.7.3, last commit 2026-10-31) on
2026-11-14. Line numbers drift; search by symbol name.

## 1. Big picture

```
 Agents (Claude Code, Codex, ...)          Phone (Android)
        | run hook command                       ^
        v                                        | LocalSend / Blip
  notch-hook (tiny CLI)                          |
        | Unix socket ~/.mynotch/bridge.sock     |
        v                                        |
 +--------------------------- MyNotch.app ---------------------------+
 |  NotchCore (Swift package: pure logic, tested)                    |
 |    wire protocol, agent adapters, risk classifier, reducers,      |
 |    attention arbiter logic, LocalSend client                      |
 |                                                                   |
 |  NotchModules/Core                                                |
 |    ModuleRegistry  -> which modules exist / are enabled           |
 |    NotchPresenter  -> what the notch shows right now              |
 |    ModulesSettingsView                                            |
 |                                                                   |
 |  NotchModules/Agents | SendToDevice | NotificationMirror | Legacy |
 |                                                                   |
 |  Upstream boring.notch (window, notch shape, music, shelf, HUD)   |
 +-------------------------------------------------------------------+
```

## 2. The upstream app in one page

| Concern | Where |
|---|---|
| App entry, windows per screen, drag detectors | `boringNotch/boringNotchApp.swift` (`DynamicNotchApp`, `AppDelegate`) |
| Notch window classes | `boringNotch/components/Notch/BoringNotchWindow.swift`, `BoringNotchSkyLightWindow.swift` |
| Root view, closed vs open layout, drop handling | `boringNotch/ContentView.swift` |
| Global UI state (current tab, sneak peek, expanding view) | `boringNotch/BoringViewCoordinator.swift` (`BoringViewCoordinator.shared`) |
| Per-screen state (open/closed, size, drop targeting) | `boringNotch/models/BoringViewModel.swift` |
| Tab enum | `boringNotch/enums/generic.swift` (`NotchViews { home, shelf }`) |
| Tab bar | `boringNotch/components/Tabs/TabSelectionView.swift` (`tabs` array), `components/Notch/BoringHeader.swift` |
| Home tab (music, calendar, mirror) | `boringNotch/components/Notch/NotchHomeView.swift` |
| Shelf (files, quick share) | `boringNotch/components/Shelf/**` |
| Settings window | `boringNotch/components/Settings/SettingsView.swift` (one 1,800-line file; sections are structs `GeneralSettings`, `Media`, `Shelf`, ...) |
| Settings keys | `boringNotch/models/Constants.swift` (`Defaults.Keys`) |
| Sizes | `boringNotch/sizing/matters.swift` (`openNotchSize` 640x190) |
| Privileged helper | `BoringNotchXPCHelper/` (unsandboxed; accessibility + brightness) |

Existing feature flags (all `Defaults` Bool keys): `boringShelf`,
`showCalendar`, `showMirror`, `hudReplacement`, `showBatteryIndicator`,
`enableLyrics`, `enableDownloadListener`, `showNotHumanFace`, plus
`musicLiveActivityEnabled` (`@AppStorage` in the coordinator). These are
checked inline all over `ContentView` and `NotchHomeView`.

## 3. Our additions

### 3.1 `Packages/NotchCore` (local Swift package)

Pure Swift, no AppKit/SwiftUI imports in the library target, so `swift test`
runs it from the command line.

```
Sources/NotchCore/
  Bridge/        WireMessage, WireCodec (newline-delimited JSON)
  Agents/        AgentKind, AgentEvent (normalised), AgentSession,
                 PermissionRequest, PermissionDecision
    Adapters/    ClaudeCodeAdapter, CodexAdapter, (later) AntigravityAdapter,
                 CursorAdapter  -- raw hook JSON <-> normalised types
    AgentReducer.swift     (state, event) -> state
    RiskClassifier.swift   command/tool -> .normal | .caution | .danger
  Attention/     Presentation, Priority, AttentionQueue (arbiter logic)
  Transfer/      TransferTarget, TransferProvider protocol, LocalSend client
  Install/       ConfigMerger: merge/unmerge hook entries in agent config
Sources/notch-hook/main.swift
Tests/NotchCoreTests/ (+ Fixtures/ with real recorded payloads)
```

### 3.2 `boringNotch/NotchModules` (synced folder in the app target)

```
Core/
  NotchModule.swift          protocol
  ModuleRegistry.swift       list, enable/disable, lifecycle
  NotchPresenter.swift       connects AttentionQueue to the notch UI
  ModulesSettingsView.swift  Settings > Modules
  ModuleTabHost.swift        renders the selected module tab
Agents/
  AgentsModule.swift, AgentBridgeServer.swift (socket), AgentStore.swift,
  HookInstaller.swift, Views/{AgentListView, AgentRowView, AgentDetailView,
  PermissionCardView, AgentsClosedIndicator, AgentsSettingsView}.swift
SendToDevice/
  SendToDeviceModule.swift, Providers/{ShareMenuProvider, LocalSendProvider},
  Views/{DeviceDropTargets, TransferProgressView, SendSettingsView}.swift
NotificationMirror/   (later)
Legacy/
  LegacyFlagModule.swift     wraps an upstream Defaults flag as a module
```

### 3.3 Touch points in upstream files (keep each edit tiny, tag `// MYNOTCH:`)

| File | Change |
|---|---|
| `enums/generic.swift` | add `case module(String)` to `NotchViews` (make it `Hashable`) |
| `components/Tabs/TabSelectionView.swift` | build `tabs` from `ModuleRegistry.enabledTabs` appended to the upstream two |
| `ContentView.swift` | in `switch coordinator.currentView` add `case .module(let id): ModuleTabHost(id:)`; add one overlay hook where `NotchPresenter` can show a closed-notch indicator / card |
| `boringNotchApp.swift` | in `applicationDidFinishLaunching` call `ModuleRegistry.shared.bootstrap()`; expose a small helper to open/close the notch on the active screen |
| `components/Settings/SettingsView.swift` | add one `NavigationLink("Modules")` and its detail view |
| `boringNotch.entitlements`, `Info.plist` | sandbox off (ADR-003), updater off (ADR-008) |
| `project.pbxproj` | new bundle id / name; add `NotchModules` synced folder; add local package `Packages/NotchCore` |

Opening the notch programmatically: `AppDelegate.viewModels` maps screen UUID
to `BoringViewModel`; call `open()` on the one for
`coordinator.selectedScreenUUID` (or all, if "show on all displays" is on).
Wrap this in `NotchPresenter` so modules never touch view models directly.

## 4. Module lifecycle

```
app launch -> ModuleRegistry.bootstrap()
           -> for each registered module: if enabled -> await module.start()
user flips switch on  -> persist -> await start() -> tab/indicator appear
user flips switch off -> persist -> await stop()  -> everything disappears
app quit -> stop() all
```

`start()` acquires resources (socket, observers, timers). `stop()` releases
all of them. A module must be restartable any number of times.
Details and the Swift protocol: `MODULE_SYSTEM.md`.

## 5. Attention arbiter

Modules never open the notch themselves. They submit a `Presentation` to
`NotchPresenter`; the arbiter decides.

Priority bands (higher wins):

| Band | Value | Examples | May auto-open the notch |
|---|---|---|---|
| critical | 100 | agent permission request | yes |
| timeSensitive | 80 | incoming transfer needs accept, agent asks a question | yes |
| message | 60 | mirrored WhatsApp / mail | sneak peek only (setting) |
| status | 40 | agent finished, transfer complete | sneak peek only |
| ambient | 20 | music, upstream sneak peeks | never |

Rules:

1. Highest band shows; ties are first-in-first-out.
2. A card the user is hovering or typing in is never replaced; the newcomer
   queues and a small "+1" badge shows.
3. `requiresAck` presentations stay until answered, withdrawn by the module,
   or expired (TTL).
4. Lower bands resume after higher ones clear.
5. Per-module setting "may open the notch by itself" (default on only for
   Agents). A global "Do not disturb" switch downgrades everything except
   critical to badge-only.
6. Upstream sneak peeks (volume, brightness, music) keep working as they do
   today and are treated as `ambient`: they are suppressed while a higher
   band is visible.

The queue logic is pure and lives in `NotchCore/Attention` with tests; the
SwiftUI glue lives in `NotchModules/Core/NotchPresenter.swift`.

## 6. Agents data flow

```
agent hook fires
  -> notch-hook --agent <kind>   (reads stdin JSON, adds env context)
  -> socket -> AgentBridgeServer -> Adapter.normalise(raw) -> AgentEvent
  -> AgentReducer -> AgentStore (ObservableObject, @MainActor)
       status event      -> UI updates, reply sent immediately (empty)
       permission event  -> PermissionRequest stored, Presentation(critical)
                            submitted, connection held open
  user clicks Allow/Deny -> Adapter.encode(decision) -> exact stdout string
  -> socket -> notch-hook prints it, exits 0 -> agent continues
```

If the socket closes before a decision (agent resolved it elsewhere, hook
killed, timeout), the pending request is withdrawn and its card removed.
Full protocol: `HOOK_PROTOCOL.md`.

## 7. Persistence

| Data | Where | Notes |
|---|---|---|
| Module enabled flags, module settings | `Defaults` (UserDefaults) | keys `module.<id>.enabled`, `module.<id>.<setting>` |
| Hook client binary | `~/.mynotch/bin/notch-hook` | copied by the installer |
| Socket | `~/.mynotch/bridge.sock` | dir `0700`, socket `0600` |
| Debug event log (opt-in) | `~/.mynotch/logs/events.jsonl` | off by default; rotates at 5 MB |
| Config backups | next to the original file | `<name>.mynotch-backup-<timestamp>` |
| Pinned devices | `Defaults` | name, provider id, provider-specific address/fingerprint |
| Agent sessions, requests, notification content | memory only | never written to disk |

## 8. Build and signing notes

- Sign debug builds with a **stable identity** (the owner's free "Apple
  Development" certificate, set once in Xcode > Signing). Ad-hoc signatures
  change every build and macOS forgets Accessibility / Local Network grants.
- Bundle identifier differs from upstream so both apps can be installed and
  settings never collide.
- The hook client is built by SwiftPM and installed by
  `scripts/install-hook-client.sh` during development. Embedding it in the
  app bundle (Copy Files build phase to `Contents/Helpers`) is a Phase 2
  polish task; the installer then copies it from the bundle.

## 9. What we deliberately do not build yet

- Dynamic plug-in loading (bundles/dylibs). Modules are compiled in. The
  protocol is designed so loading from bundles can be added later without
  changing modules.
- A rewrite of upstream features. They are wrapped (`LegacyFlagModule`), not
  ported.
- Multi-Mac or remote approval.
