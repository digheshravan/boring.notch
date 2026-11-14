# Decisions (ADR log) — MyNotch

Append-only. To change a decision, add a new entry that supersedes the old
one. Format: context, decision, consequences.

---

## ADR-001 — Fork boring.notch instead of starting from scratch
*2026-11-14 · accepted*

**Context.** The owner wants a boring.notch-like base with his own features.
boring.notch (GPL-3.0, SwiftUI) already solves the hard notch-window
problems (window level, multi-display, sizing, drag detection, lock screen)
and has music, calendar, HUD and a file shelf with share-menu support.
Alternatives considered: forking Casement or Agent Notch (both MIT, already
have agent approvals) or a new app.

**Decision.** Fork boring.notch `main` at v2.7.3. Keep `upstream` as a git
remote. Study Agent Notch and Casement as references for the Agents module.

**Consequences.** The project is GPL-3.0 and must stay so when published.
Building requires macOS 15.6+ and Xcode 26+. We inherit a large
`ContentView`/`SettingsView`; we touch them minimally.

## ADR-002 — Module system first, upstream features wrapped not ported
*2026-11-14 · accepted*

**Context.** "User chooses what shows" is the core idea. Upstream has flags
scattered through views, not modules.

**Decision.** Phase 0 builds `NotchModule` + `ModuleRegistry` +
`NotchPresenter` before any feature. Upstream features are exposed through
`LegacyFlagModule` wrappers over their existing `Defaults` flags. Porting an
upstream feature into a real module is done only when there is a reason.

**Consequences.** One uniform Modules page from day one. Some legacy
"modules" may only hide UI rather than stop work; documented per feature.

## ADR-003 — Turn the app sandbox off in the fork
*2026-11-14 · accepted, revisit before public release*

**Context.** Upstream's app target is sandboxed. The Agents module needs a
Unix socket reachable by agent hook processes and must edit files in
`~/.claude`, `~/.codex`. A sandboxed app cannot do either without awkward
workarounds (container sockets trigger "access data from other apps"
prompts; config edits would need open-panel grants per folder). Later
modules need Accessibility.

**Decision.** Set `com.apple.security.app-sandbox` to false for the app
target. Keep Hardened Runtime and the entitlements still needed without the
sandbox (camera, calendars, Apple events).

**Consequences.** No App Store. Less containment (see SECURITY.md section
4). Settings move out of the sandbox container (new bundle id, so nothing to
migrate). SPIKE in Phase 0: confirm music control, calendar, camera mirror,
HUD and shelf still work with the sandbox off. Alternative kept in reserve:
host the bridge and installer in the unsandboxed XPC helper.

## ADR-004 — Agent bridge over a Unix domain socket
*2026-11-14 · accepted*

**Context.** Agents run a hook command; it must reach the app. Options:
localhost HTTP (Claude Code even supports `type: "http"` hooks natively),
XPC, Unix socket.

**Decision.** Unix stream socket at `~/.mynotch/bridge.sock`, directory
`0700`, socket `0600`, peer-uid check. Newline-delimited JSON.

**Consequences.** Not reachable from the network or the browser, unlike a
localhost port. Works identically for every agent. Needs a small client
binary (ADR-005).

## ADR-005 — Thin hook client, adapters in the app, fail open
*2026-11-14 · accepted*

**Decision.** `notch-hook` forwards raw stdin plus context and prints back
exactly what the app returns. All agent-specific parsing and output
formatting lives in `NotchCore` adapters. Any failure in the client results
in exit 0 with no output, which Claude Code and Codex treat as "no decision"
and fall back to their own prompt.

**Consequences.** Adapters are unit-testable. The client rarely changes
(good for Codex, which re-asks for trust when a hook changes). Agents whose
"no output" means allow (reported for Cursor) cannot be gated safely and are
status-only until proven otherwise.

## ADR-006 — Logic in a local Swift package, UI in a synced folder
*2026-11-14 · accepted*

**Context.** Upstream has no test target, and new files in classic Xcode
groups must be hand-registered in `project.pbxproj`.

**Decision.** `Packages/NotchCore` (SwiftPM library + `notch-hook`
executable + tests) for everything that needs no window.
`boringNotch/NotchModules` added once as a file-system-synchronised folder
(same mechanism upstream uses for `boringNotch/private`) for views and glue.

**Consequences.** `swift test` works from the terminal. After the one-time
project edit, new files need no project changes. The one-time edit is the
riskiest step of Phase 0; if hand-editing the pbxproj fails, the owner does
it in Xcode (steps in SETUP.md).

## ADR-007 — No automatic approvals; "always allow" only via the agent's own rules
*2026-11-14 · accepted*

**Decision.** v1 has no auto-approve, no MyNotch-side allowlist. Phase 2
adds "Always allow" for Claude Code only, by returning a rule the agent
itself suggested (`permission_suggestions` -> `updatedPermissions`), after
showing the rule and an explicit click. The rule is stored and enforced by
Claude Code, not by MyNotch.

**Consequences.** MyNotch never becomes a policy engine that can be wrong.
Reducing prompt volume stays the job of the agent's permission settings.

## ADR-008 — New identity; upstream auto-update disabled
*2026-11-14 · accepted*

**Decision.** New bundle identifier and app name (placeholders
`com.shravandige.mynotch`, "MyNotch"). Remove/disable the Sparkle feed
(`SUFeedURL` points at upstream) and the "check for updates" UI.

**Consequences.** MyNotch and boring.notch can be installed side by side.
No update mechanism until the project has its own releases.

## ADR-009 — Send-to-device through providers; Blip via share menu, LocalSend native
*2026-11-14 · accepted*

**Context.** The owner uses Blip (closed source, no documented automation
interface). He is open to an open-source tool if Blip cannot be driven.

**Decision.** `TransferProvider` abstraction. Provider 1: share-menu
services (covers Blip and AirDrop using upstream's existing code).
Provider 2: native LocalSend protocol v2 client for true one-drop sends to
pinned devices. A spike decides how far Blip can go.

**Consequences.** The phone needs the LocalSend app for provider 2.
Switching to an iPhone later is a settings change.

## ADR-010 — Working name
*2026-11-14 · placeholder*

"MyNotch" everywhere until the owner picks a name. Keep the name in as few
places as possible (bundle display name, `~/.mynotch`, logger subsystem,
one constants file) so renaming is a small change.

## ADR-011 — Base branch
*2026-11-14 · accepted*

Upstream asks contributors to branch from `dev`, but `dev` moves fast. The
fork's `main` tracks upstream `main` (released code). Our work happens on
`mynotch/main`, with one branch per phase (`phase-0-base`, `phase-1-agents`,
...) merged after the owner's review.

---

## Open questions (resolve with spikes; then write an ADR)

| # | Question | Phase |
|---|---|---|
| Q1 | Does everything upstream still work with the sandbox off? | 0 |
| Q2 | Can `NotchModules` be added as a synced folder by editing the pbxproj, or must it be done in Xcode? | 0 |
| Q3 | Do the legacy flags fully stop their features or only hide UI? | 0 |
| Q4 | While a Claude Code `PermissionRequest` hook is pending, is the terminal prompt also answerable? | 1 |
| Q5 | Best source for task progress n/m in Claude Code: `TaskCreated`/`TaskCompleted` events or task/todo tool inputs? | 1 |
| Q6 | Does an omitted `matcher` match all tools for Claude Code hooks? | 1 |
| Q7 | Do Codex hooks fire in the Codex IDE extension / desktop app? | 2 |
| Q8 | What automation surface does Blip expose on macOS? | 3 |
| Q9 | Antigravity hook config location, payload and decision format on the owner's version | later |
| Q10 | Cursor: does missing hook output fall back to Cursor's prompt on the owner's version? | later |
| Q11 | Notification mirror: Accessibility vs database on the owner's macOS | later |

## Spike log

Results of spikes as they come in. A spike that changes a decision gets a new
ADR above; this log only records what was found.

### Q1 — Sandbox off (ADR-003) · 2026-11-14 · in progress
- Sandbox flipped off in `boringNotch/boringNotch.entitlements`; app builds
  and the signed entitlements show `app-sandbox = false` with camera,
  calendars and Apple Events still present.
- No upstream code depends on the sandbox container. Shelf files use
  security-scoped bookmarks, which also work unsandboxed (to be confirmed by
  hand).
- Feature results: **pending owner's hand test** (checklist in PROGRESS.md,
  2026-11-14 entry). Record per-feature pass/fail here when done.
