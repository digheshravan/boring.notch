# Module System — MyNotch

The module system is the product. Everything else plugs into it.

## 1. Goals

- Every feature is one self-contained unit with one on/off switch.
- Off means off: no UI, no background work, no permission prompts.
- Adding a feature = one new folder + one line in the registry.
- A capability can have swappable providers.
- Upstream boring.notch features appear in the same list without being
  rewritten.

## 2. The protocol

Sketch; refine during Phase 0 but keep the shape.

```swift
import SwiftUI

/// A permission a module needs from macOS. Shown in Settings before enabling.
enum ModulePermission: String, CaseIterable {
    case accessibility, localNetwork, fullDiskAccess, automation, notifications
}

/// What a module contributes to the expanded notch tab bar.
struct ModuleTab {
    let title: String
    let symbolName: String          // SF Symbol
    let content: () -> AnyView
}

@MainActor
protocol NotchModule: AnyObject {
    /// Stable identifier, lowercase, e.g. "agents". Used in Defaults keys.
    var id: String { get }
    var displayName: String { get }
    var summary: String { get }      // one line, shown in Settings
    var symbolName: String { get }
    var defaultEnabled: Bool { get }
    var requiredPermissions: [ModulePermission] { get }

    /// Acquire resources. Called when the module becomes enabled.
    func start() async
    /// Release every resource. Must be safe to call repeatedly.
    func stop() async

    /// Optional contributions. Return nil when not applicable.
    var tab: ModuleTab? { get }
    var settingsView: AnyView? { get }
    /// Small view shown beside the closed notch while the module has
    /// something ambient to show (e.g. "3 agents"). Nil hides it.
    var closedIndicator: AnyView? { get }
}
```

Attention (cards, sneak peeks, auto-open) does not go through the protocol.
Modules call `NotchPresenter.shared.present(...)` / `.withdraw(id:)`.

## 3. Registry

```swift
@MainActor
final class ModuleRegistry: ObservableObject {
    static let shared = ModuleRegistry()

    /// THE one place a new module is registered.
    private(set) lazy var all: [NotchModule] = [
        AgentsModule(),
        SendToDeviceModule(),
        // NotificationMirrorModule(),
    ] + LegacyFlagModule.upstreamModules

    @Published private(set) var enabledIDs: Set<String> = []

    func bootstrap() async          // start all enabled modules at launch
    func setEnabled(_ id: String, _ on: Bool) async
    func isEnabled(_ id: String) -> Bool
    var enabledTabs: [(id: String, tab: ModuleTab)] { get }
}
```

- Enabled state: `Defaults` key `module.<id>.enabled`; falls back to
  `defaultEnabled` when unset.
- `setEnabled(true)`: persist, `await start()`, publish.
- `setEnabled(false)`: `NotchPresenter.withdrawAll(module:)`, `await stop()`,
  persist, publish. If the current tab belongs to that module, switch to
  `.home`.
- Enabling a module that needs permissions shows what will be requested and
  why **before** macOS prompts.

## 4. Legacy wrappers

Upstream features already have `Defaults` flags. `LegacyFlagModule` exposes
each as a module whose switch reads and writes that flag. `start()`/`stop()`
are no-ops; upstream code already reacts to the flag.

| Module id | Display name | Upstream flag |
|---|---|---|
| `legacy.music` | Music & now playing | `musicLiveActivityEnabled` (AppStorage) |
| `legacy.calendar` | Calendar | `showCalendar` |
| `legacy.shelf` | File shelf | `boringShelf` |
| `legacy.mirror` | Camera mirror | `showMirror` |
| `legacy.hud` | Volume / brightness HUD | `hudReplacement` |
| `legacy.battery` | Battery indicator | `showBatteryIndicator` |

SPIKE (Phase 0): confirm each flag fully disables its feature. Where a flag
only hides UI but work continues, note it in `PROGRESS.md`; fixing that is a
later task, not a Phase 0 blocker.

The existing per-feature settings pages stay where they are. The Modules
page links to them ("Settings...").

## 5. Presenter API

```swift
struct Presentation: Identifiable {
    enum Style { case card, sneakPeek, badge }
    let id: String                 // stable; re-presenting replaces
    let moduleID: String
    let priority: Priority         // .critical/.timeSensitive/.message/.status/.ambient
    let style: Style
    let requiresAck: Bool          // stays until answered or withdrawn
    let ttl: TimeInterval?         // auto-dismiss
    let content: () -> AnyView
}

@MainActor
final class NotchPresenter: ObservableObject {
    static let shared = NotchPresenter()
    @Published private(set) var current: Presentation?
    @Published private(set) var queuedCount: Int
    func present(_ p: Presentation)
    func withdraw(id: String)
    func withdrawAll(module: String)
    func openNotch(tab: NotchViews? = nil)   // the only code that calls vm.open()
    func closeNotchIfIdle()
}
```

Queue rules: see ARCHITECTURE.md section 5. Card size is constrained by the
fixed open notch (640x190). Cards must fit; long content scrolls inside the
card.

## 6. Settings > Modules

One list. Each row: icon, name, one-line summary, permission chips, switch,
"Settings..." button when the module has a settings view.

Sections: **Productivity** (Agents), **Devices** (Send to device),
**Notifications** (Notification mirror), **Built-in** (legacy wrappers).

Global controls at the top: "Do not disturb", "Modules may open the notch by
themselves" (master switch), tab order (drag to reorder, Phase 2 polish).

## 7. Providers pattern

When a capability has several ways to do it, the module owns a provider
protocol and a provider list, and exposes "which providers are enabled" in
its settings. Example (`SendToDevice`):

```swift
protocol TransferProvider: AnyObject {
    var id: String { get }                 // "sharemenu.blip", "localsend"
    var displayName: String { get }
    var isAvailable: Bool { get async }    // app installed / network ok
    func targets() async -> [TransferTarget]
    func send(_ items: [URL], to target: TransferTarget)
        -> AsyncThrowingStream<TransferProgress, Error>
}
```

Disabled providers are not started. Enabling AirDrop and disabling Blip is a
settings change inside the module.

## 8. Rules for module authors

1. No singletons reaching into another module. Shared services go in `Core`.
2. No direct calls to `BoringViewModel.open()`; use `NotchPresenter`.
3. All background work is created in `start()` and cancelled in `stop()`.
   Keep a `Task` handle bag and cancel it.
4. Module state that should survive restart goes in `Defaults` under
   `module.<id>.*`. Nothing sensitive on disk.
5. Logic goes in `NotchCore` with tests; the module folder holds views and
   glue.
6. A module must work on a Mac without a notch (upstream supports the
   floating style) or hide itself cleanly.

## 9. Phase 0 definition of done

- `NotchModule`, `ModuleRegistry`, `NotchPresenter`, `LegacyFlagModule`,
  `ModulesSettingsView` exist and build.
- A throwaway `DemoModule` proves the path end to end: a tab with a button
  that presents a critical card with two buttons; a second button that
  presents a status sneak peek; switching the module off removes the tab and
  withdraws its presentations.
- `AttentionQueue` unit tests cover: priority order, FIFO within a band,
  hover lock, TTL expiry, withdraw, do-not-disturb.
- `DemoModule` is deleted (or `#if DEBUG`) before Phase 1 starts.
