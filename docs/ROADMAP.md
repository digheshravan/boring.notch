# Roadmap — MyNotch

Rule: one phase per branch, one review checkpoint per phase. Do not start a
phase until the owner has reviewed the previous one. Each task is small
enough to build and verify on its own.

Legend: `SPIKE` = find out, write the answer in DECISIONS.md, no feature
code. `[ ]` = task. **Checkpoint** = stop and hand over.

---

## Phase 0 — Base (branch `phase-0-base`)

Goal: the fork runs under its own identity and has a working module system.
No new user-visible feature except the Modules settings page.

### 0A. Fork runs
- [ ] Follow SETUP.md: fork, clone, remotes, branch.
- [ ] Build upstream unchanged; run it; confirm the notch works. Fix nothing.
- [ ] Rename: bundle id, display name, logger subsystem; put the name in one
      constants file (ADR-010).
- [ ] Disable Sparkle updater and its settings UI (ADR-008).
- [ ] Add `NOTICE.md` (fork of boring.notch, GPL-3.0, list of changes).
- [ ] SPIKE Q1: sandbox off (ADR-003). Test music controls, calendar,
      mirror, HUD replacement, shelf drag-and-drop, battery, lock screen.
      Record results.
- [ ] Set a stable signing identity so macOS permissions persist.

**Checkpoint 0A.** Owner runs the app, confirms it behaves like boring.notch
under the new name, and that boring.notch (if installed) is unaffected.

### 0B. Project structure
- [ ] Create `Packages/NotchCore` (library + empty `notch-hook` executable +
      one passing test). `swift test` green.
- [ ] SPIKE Q2: add `boringNotch/NotchModules` as a synced folder and
      `Packages/NotchCore` as a local package dependency of the app target.
      Prove it: a file in `NotchModules` that imports `NotchCore` compiles
      into the app.
- [ ] `scripts/install-hook-client.sh`, `scripts/dev-build.sh`.

### 0C. Module system
- [ ] `AttentionQueue` + `Priority` + `Presentation` model in `NotchCore`
      with the tests listed in MODULE_SYSTEM.md section 9.
- [ ] `NotchModule` protocol, `ModuleRegistry` (bootstrap, enable/disable,
      persistence).
- [ ] `NotchPresenter`: owns open/close of the notch; renders card,
      sneak-peek and badge styles; applies queue rules.
- [ ] Upstream touch points (ARCHITECTURE.md 3.3): `NotchViews.module`,
      tab bar from registry, `ContentView` switch + presenter overlay.
- [ ] `LegacyFlagModule` wrappers. SPIKE Q3.
- [ ] `ModulesSettingsView` + link in Settings.
- [ ] `DemoModule` (debug only) exercising tab, card, sneak peek, disable.

### Acceptance — Phase 0
1. App builds from a clean clone with the documented commands.
2. `swift test --package-path Packages/NotchCore` passes.
3. Settings > Modules lists built-in features with working switches; each
   switch matches the corresponding upstream setting both ways.
4. Demo module: tab appears/disappears with its switch without restart.
5. Demo critical card opens the notch by itself, stays until a button is
   clicked, then the notch closes.
6. While a critical card is visible, a volume/music sneak peek does not
   replace it; it shows after.
7. Two cards queued: answering the first shows the second.
8. Nothing regressed: music, calendar, shelf, HUD work as in 0A.

**Checkpoint 0.** Owner tests 3-8 by hand. Feedback round. Then delete or
`#if DEBUG` the demo module.

---

## Phase 1 — Agents MVP: Claude Code (branch `phase-1-agents`)

Goal: approve Claude Code permission requests from the notch and see
sessions. Read `modules/AGENTS.md`, `HOOK_PROTOCOL.md`, `SECURITY.md` first,
and re-read the official Claude Code hooks documentation.

### 1A. Plumbing
- [ ] `WireMessage` + `WireCodec` (+ tests: round trip, oversize, garbage).
- [ ] `notch-hook` client (HOOK_PROTOCOL.md section 1). Test by hand with
      `nc -lU` as a fake server and with no server (must exit 0 silently).
- [ ] `AgentBridgeServer`: socket lifecycle, permissions, peer uid check,
      limits, hold/reply, withdraw on disconnect. Starts in
      `AgentsModule.start()`, stops and removes the socket in `stop()`.
- [ ] Debug event recorder (off by default).
- [ ] Hand-add the hook entries to a **test project's**
      `.claude/settings.json` (not the global file yet). Record real
      payloads for every event; scrub; save as fixtures.
- [ ] SPIKE Q4, Q5, Q6.

**Checkpoint 1A.** Show the owner the recorded event stream from a real
session and the spike answers. Confirm the UX assumptions still hold.

### 1B. Logic
- [ ] `ClaudeCodeAdapter` test-first against fixtures (normalise, needsHold,
      encode allow / deny / deny+message / passthrough).
- [ ] `AgentReducer` + `AgentSession` with tests for every transition.
- [ ] `RiskClassifier` with the command table tests (SECURITY.md).
- [ ] Text sanitiser (control chars, ANSI, bidi) with tests.

### 1C. UI
- [ ] `AgentStore` (ObservableObject) fed by the bridge.
- [ ] Agents tab: session list, row, states.
- [ ] `PermissionCardView` with all SECURITY.md approval rules
      (S1-S9), countdown, queue indicator, deny-with-note.
- [ ] Closed-notch indicator; "done" sneak peek; optional sound.
- [ ] Auto-open on request, auto-close after the last answer, do not close
      if the user opened the notch by hand.
- [ ] Session housekeeping (dead process cleanup, idle, auto-dismiss).

### 1D. Installer
- [ ] `ConfigMerger` with its tests (HOOK_PROTOCOL.md section 9).
- [ ] `HookInstaller` for Claude Code + Agents settings view (status,
      Install, Remove, "Show what will change", wait time, toggles).

### Acceptance — Phase 1 (manual script)
1. Install from settings. `~/.claude/settings.json` has our entries, a
   backup exists, other content is untouched.
2. Start two Claude Code sessions in two projects: two rows, correct
   project names, state changes as they work.
3. Ask one to run a command that needs permission: the notch opens with
   the full command. **Allow** -> it runs. Repeat, **Deny** -> blocked and
   Claude says so. Repeat, **Deny with note** -> Claude quotes the note.
4. Ask for `rm -rf` on a temp folder: card is red with a reason.
5. Trigger requests in both sessions at once: "1 of 2", answered in order.
6. Answer a request in the terminal instead (if Q4 allows): card vanishes.
7. Quit MyNotch while a card is open: the terminal prompt works normally.
   Start a session with MyNotch not running: no delay, no errors.
8. Leave a request unanswered: after the wait time it falls back to the
   terminal prompt.
9. Switch the Agents module off: tab and indicator gone, socket file gone,
   agents unaffected. On again: works without restarting agents.
10. Remove from settings: `settings.json` back to its original content.
11. Run the SECURITY.md section 5 checklist.
12. `swift test` green.

**Checkpoint 1.** Owner uses it for real work for at least a day before
Phase 2. Collect feedback in PROGRESS.md.

---

## Phase 2 — Agents: detail, Codex, polish (branch `phase-2-agents-plus`)

- [ ] Session detail view: progress n/m, recent actions, last message.
- [ ] Jump to window (activate host app; exact tab where cheap).
- [ ] "Always allow" for Claude Code via suggested rules (ADR-007, S8).
- [ ] `CodexAdapter` + fixtures + installer for `~/.codex/hooks.json` +
      "trust the hook with /hooks" guidance. SPIKE Q7.
- [ ] Keyboard shortcuts (unassigned by default).
- [ ] Embed `notch-hook` in the app bundle; installer copies from there.
- [ ] Feedback items from Checkpoint 1.

### Acceptance — Phase 2
1. Clicking a session shows live step, progress and recent actions.
2. Jump brings the right app forward.
3. "Always" shows the rule, applies it, and the same command does not ask
   again in that project.
4. A Codex session appears and can be approved/denied from the notch;
   Codex without the hook trusted shows a clear hint.
5. Claude Code and Codex requests queue together correctly.
6. Security checklist re-run.

**Checkpoint 2.**

---

## Phase 3 — Send to device (branch `phase-3-send`)

Read `modules/SEND_TO_DEVICE.md`.

### 3A. Share-menu provider
- [ ] SPIKE Q8 (Blip automation surface). Write the result down first.
- [ ] `TransferProvider` protocol, `SendToDeviceModule`, settings.
- [ ] `ShareMenuProvider` reusing upstream `QuickShareService`.
- [ ] Device/service drop targets beside the shelf during a drag.

**Checkpoint 3A.** Owner sends a file to his phone through Blip from the
notch and reports how many clicks it took.

### 3B. LocalSend provider
- [ ] Protocol types + codec + upload client in `NotchCore` with a stub
      server test suite (all status codes, cancel, checksum).
- [ ] Discovery (multicast + register), device list, pinning with
      fingerprint (S23, S24), Local Network permission flow.
- [ ] Progress presentation, cancel, retry, queue.
- [ ] Shelf context menu "Send to".

### Acceptance — Phase 3
1. Dragging a file over the notch shows shelf + device targets.
2. Drop on Blip target: Blip's flow starts with the file attached.
3. With LocalSend open on the phone, the phone appears; pin it.
4. Drop on the pinned phone: progress shows; the file arrives intact
   (compare size / hash); success message.
5. Phone offline: clear failure, Retry works when it is back.
6. Large file (1 GB+): memory stays flat; cancel works mid-transfer.
7. A device presenting the pinned name with a different fingerprint is not
   treated as pinned.
8. Providers and the whole module can be switched off; discovery stops.

**Checkpoint 3.** v1 complete. Owner decides what comes next.

---

## Later (not scheduled; each needs its own spike and a roadmap entry)

- Agents: Antigravity adapter (Q9), Cursor status-only adapter (Q10),
  answering agent questions, queued follow-up prompts.
- Notification mirror (Q11): WhatsApp flagged chats, important mail.
- Receive files into the notch shelf (LocalSend receiver).
- Port the shelf into a real module; tab reordering; per-module shortcuts.
- Public release: name, icon, own update feed, README, contribution guide,
  root `SECURITY.md`, decide on re-sandboxing (ADR-003).
- Loadable third-party modules.

## Estimated effort (owner new to Swift, Claude Code doing most typing)

| Phase | Sessions |
|---|---|
| 0 | 2-3 |
| 1 | 3-4 |
| 2 | 2-3 |
| 3 | 3-4 |

These are guesses. The spikes decide.
