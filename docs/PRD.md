# Product Requirements — MyNotch

Status: draft v1 (2026-11-14). Owner: Shravan Dige. Working name: MyNotch.

## 1. Problem

The MacBook notch is dead space. Several apps put it to use, but:

- **Features are scattered.** One app has agent approvals, another has a file
  shelf, another has notifications. You cannot run two notch apps at once.
- **Some of the best features are paid** or closed source.
- **Every notch app assumes an iPhone.** AirDrop, iPhone Live Activities and
  iMessage are the defaults. People with a Mac and an Android phone get
  nothing built for them.
- **The developer decides what you see.** Toggles exist in places, but there
  is no clean way to say "this whole feature does not exist for me".

The first concrete pain: when a coding agent (Claude Code, Codex, Antigravity,
Cursor) needs permission, the owner has to stop reading, find the right
window, and click Allow. With several agents running in the background this
happens constantly and each one costs a context switch.

## 2. Vision

One open-source notch app where **the user, not the developer, decides what
the notch does.** Every feature is a module. Modules are added one at a time
as real problems are recognised. A switched-off module does not exist: no UI,
no background work, no permissions.

## 3. Who it is for

1. **Primary: the owner.** MacBook Pro (notch) + Android phone + Windows
   laptop. Runs several AI coding agents. Wants fewer context switches.
2. **Secondary: Mac + Android users** who are tired of being second-class.
3. **Tertiary: developers** who want to add their own module without forking
   the whole UI.

## 4. Principles

1. **User in control.** Every module has an on/off switch and its own
   settings. Sensible defaults, nothing forced.
2. **Off means off.** Disabled modules run no code and ask no permissions.
3. **Swappable providers.** A capability (e.g. "send to device") is separate
   from how it is done (Blip, LocalSend, AirDrop). Switching phone ecosystem
   is a settings change, not a new app.
4. **Quiet by default.** The notch stays hidden unless something needs the
   user. One thing at a time, most important first.
5. **Safe by design.** Convenience never weakens a security decision. See
   `SECURITY.md`.
6. **Local and private.** No accounts, no telemetry, nothing leaves the Mac
   unless a module's job is to send it (file transfer).
7. **Open.** GPL-3.0, documented module interface, readable code.

## 5. What makes it different

| | boring.notch | Agent Notch / Casement | MyNotch |
|---|---|---|---|
| Feature on/off | Flags for some features | Some | Every feature is a module; off = not running |
| Agents (approve from notch) | No | Yes | Yes (module) |
| Android phone support | No | No | First-class: send-to-device providers, later notifications |
| Provider swap (Blip / LocalSend / AirDrop) | No | No | Yes |
| Add your own feature | Fork and edit views | Fork | Implement one protocol, register one line |
| What shows when several things want the notch | Ad hoc | Ad hoc | Explicit priority arbiter, user-tunable |

Honest note: approving agents from the notch already exists (Agent Notch,
Casement, CodeIsland, Vibe Notch). MyNotch does not win by having that
feature. It wins by the module system, the provider model and the
Mac + Android focus.

## 6. Scope

### v1 (this plan)

| Phase | Deliverable |
|---|---|
| 0 | Base: fork runs under a new identity, module system, attention arbiter, Modules settings page, existing boring.notch features listed as switchable modules |
| 1 | Agents module: Claude Code sessions in the notch, permission pop-up with Allow / Deny / Deny with note, auto-hide |
| 2 | Agents module: progress detail, jump to window, Codex support, "always allow" using the agent's own suggested rule |
| 3 | Send-to-device module: drag a file to the notch and drop it on a pinned device. Providers: share menu (Blip, AirDrop), then LocalSend |

### Later (specified, not scheduled)

- Agents: Antigravity and Cursor adapters, answering agent questions,
  queued follow-up prompts.
- Notification mirror module: WhatsApp for flagged chats/groups only,
  important mail only.
- Receiving files from the phone into the notch shelf.
- Clipboard sync with the phone.
- Third-party modules loaded without rebuilding the app.

### Non-goals

- App Store distribution (the app is not sandboxed; see ADR-003).
- Windows or Linux versions.
- Replacing the agents' own UIs. MyNotch is a remote control and a status
  light, not a chat client.
- Auto-approval rules, "YOLO mode", or anything that approves without a click.
- Cloud sync or accounts.

## 7. User stories and acceptance

### Base

- **B1** As a user I open Settings > Modules and see every feature with a
  switch, a one-line description and the permissions it needs.
  *Accept:* toggling a switch takes effect without restarting the app.
- **B2** When I switch a module off, its tab, closed-notch indicator and
  background activity disappear.
  *Accept:* Activity Monitor / logs show no work from that module.
- **B3** When two modules want the notch at once, the more important one
  shows first and the other waits.
  *Accept:* a permission request interrupts a music sneak peek, never the
  other way round.

### Agents

- **A1** I can see every running agent session (agent, project, state).
- **A2** When an agent needs permission, the notch opens by itself and shows
  the full command or file change with Allow and Deny. I never have to find
  the agent's window.
  *Accept:* Allow runs the tool; Deny blocks it; the agent continues or
  stops accordingly; the notch closes afterwards.
- **A3** I can deny with a short note that the agent receives as the reason.
- **A4** If I answer in the agent's own window instead, the notch card
  disappears by itself.
- **A5** If MyNotch is closed or frozen, agents behave exactly as before.
- **A6** I can click a session to see what it is doing: current step, task
  progress (n/m), recent actions, last message.
- **A7** Double-clicking a session brings its window to the front.
- **A8** Dangerous commands look dangerous (red, cannot be approved by
  keyboard shortcut).
- **A9** When nothing is pending, the notch shows at most a small indicator.

### Send to device

- **S1** I drag a file to the notch and see my pinned devices as drop targets
  next to the shelf.
- **S2** Dropping on a device sends the file with visible progress and a
  clear success or failure.
- **S3** In settings I choose the provider (Blip via share menu, LocalSend,
  AirDrop) and pin devices. If I buy an iPhone I enable AirDrop and disable
  the others; nothing else changes.

### Notification mirror (later)

- **N1** I flag WhatsApp chats and groups as important; only those appear in
  the notch.
- **N2** Mail that Apple Mail considers important appears in the notch.
- **N3** Message content is never written to disk.

## 8. Success measures (personal, checked after two weeks of use)

- I approve at least 80% of agent permissions from the notch.
- I can tell what every agent is doing without switching windows.
- Sending a file to my phone takes one drag and no extra clicks.
- I have switched at least one module off and forgotten it exists.
- No incident where MyNotch caused an agent to hang or approved something I
  did not click.

## 9. Risks

| Risk | Mitigation |
|---|---|
| "All features" never ships | Strict phases, review checkpoint after each, parking lot for ideas |
| One-click approval leads to rubber-stamping | Full command always visible, risk highlighting, no auto-approve |
| Agent hook formats change | Adapters isolated in `NotchCore` with fixture tests; verify docs at the start of each agent phase |
| Upstream boring.notch diverges | Minimal, marked edits to upstream files; our code in separate folders |
| Blip has no automation surface | Provider model; LocalSend as open-source fallback |
| Notification mirroring is fragile | Marked as a spike; shipped last; rules match on visible names only |
| Owner is new to Swift | Logic in a testable package, small steps, plain-language explanations |

## 10. Open questions for the owner

1. Final name and bundle identifier (placeholder: MyNotch,
   `com.shravandige.mynotch`).
2. Should the closed notch show a permanent small agent indicator while
   agents run, or stay fully hidden until something needs attention?
3. Sound on permission request: on by default?
4. Which Android phone model (affects LocalSend / notification testing)?
