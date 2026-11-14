# Module spec — Agents

Module id: `agents`. Default: enabled. Permissions: none for Claude Code and
Codex (Accessibility only if "jump to exact window" needs it later).

## 1. What it does

- Lists every running AI coding agent session.
- When an agent needs permission, opens the notch with a card: Allow / Deny /
  Deny with note.
- Shows what each session is doing on demand.
- Stays out of the way otherwise.

Inspiration: the "Agent Huddle" floating panel in the owner's reference video
(a list of sessions, each with a title, current step text, an n/m progress
bar, a highlight when a session "asks a question", and buttons to jump to or
dismiss a session). Same information, but in the notch, with approvals.

## 2. Session model

```
AgentSession
  id            agent kind + session_id
  agent         claudeCode | codex | ...
  project       last path component of cwd
  title         session title if the agent provides one, else first prompt
                (truncated), else project
  state         working | waitingForPermission | waitingForInput | done | idle
  currentStep   e.g. "Running: npm test", "Editing auth.swift"
  progress      (done, total)?  from task/todo events
  recent        last 20 tool calls (tool, one-line summary, ok/failed)
  lastMessage   last assistant message (from Stop)
  pending       [PermissionRequest]
  hostApp       bundle id of the terminal / editor, pid
  startedAt, updatedAt
```

State transitions (reducer in `NotchCore`, fully unit tested):

| Event | New state |
|---|---|
| sessionStarted | idle |
| promptSubmitted | working |
| toolStarted / toolFinished | working (update currentStep / recent) |
| permissionRequested | waitingForPermission |
| permission resolved or withdrawn | working |
| notification(idle / needs input) | waitingForInput |
| stopped | done |
| sessionEnded | removed |

Housekeeping: every 30 s remove sessions whose host process (`ppid`) no
longer exists; mark sessions with no events for 10 min as `idle`. `done`
sessions stay listed until dismissed or until the session ends
(setting: auto-dismiss done sessions after N minutes, default 10).

## 3. UI

All sizes must fit the fixed open notch (640x190).

### 3.1 Closed notch

- Nothing pending, no agents: nothing.
- Agents running: small indicator beside the notch — a dot plus count
  ("3"). Dot colour: green working, grey idle/done. Setting to hide it.
- Something waiting: indicator turns amber and pulses once. The card opens
  by itself (see 3.3).

### 3.2 Agents tab (notch open)

Left: scrollable session list. Row = agent icon, title, current step (one
line, truncated), thin progress bar when progress is known, state chip.
Right side of the row: jump-to-window button, dismiss button.
Sort: waiting first, then working, then done.

Click a row -> detail view replaces the list (back button):

- Header: agent, project, state, elapsed time.
- Progress: "5/7" + bar + current task name.
- Recent actions (newest first): tool, summary, ok/failed.
- Last assistant message (scrolls).
- Buttons: Jump to window, Dismiss.

Double-click a row = jump to window.

### 3.3 Permission card (critical presentation)

```
+----------------------------------------------------------------+
| [icon] Claude Code · my-project            wants to run  Bash  |
|----------------------------------------------------------------|
| npm install --save-dev vitest @vitest/coverage-v8              |
| (monospaced, full text, wraps, scrolls inside the card)        |
|----------------------------------------------------------------|
| [ Deny ]  [ Deny with note... ]          [ Always ]  [ Allow ] |
|  1 of 2 waiting                                    0:74 left   |
+----------------------------------------------------------------+
```

- Title line: agent, project, tool.
- Body: the **complete** command. For file edits: path + a compact diff
  (first N lines, scrollable). For MCP tools: server, tool, arguments as
  pretty JSON. Plain text only.
- If the agent supplied a description, show it above the command in a
  secondary colour, labelled "Agent says:" so it is not mistaken for the
  command.
- Risk styling from `RiskClassifier` (see SECURITY.md): caution = amber
  border; danger = red border, a one-line reason ("deletes files
  recursively"), the Allow button requires a click (no shortcut) and is not
  the default focus.
- "Deny with note": inline text field; Enter sends deny with the note as
  the message to the agent.
- "Always" (Phase 2, Claude Code only): shown only when the agent sent a
  suggested rule; clicking reveals the exact rule text and a confirm button.
- Countdown: time left before the request falls back to the agent's own
  prompt.
- Queue indicator when more than one request is waiting. Requests are shown
  oldest first. Answering one shows the next without closing the notch.
- After the last answer: card closes; notch closes unless the user opened it
  by hand.
- If the request is withdrawn (answered elsewhere, agent stopped): card
  disappears with a brief "answered in terminal" note.

### 3.4 Other presentations

| Situation | Band / style |
|---|---|
| Agent finished (`stopped`) | status / sneak peek: "Claude Code · my-project is done" |
| Agent idle and needs input | timeSensitive / sneak peek, amber indicator |
| Hook not trusted yet (Codex) | shown in settings only |

Optional sound on permission request (setting, default on, respects system
mute/Focus).

### 3.5 Keyboard shortcuts (Phase 2, via the existing `KeyboardShortcuts` package)

- Toggle Agents tab.
- Allow oldest request / Deny oldest request — disabled for `danger`
  requests, user-assignable, **unassigned by default**.

## 4. Settings (module settings view)

- Connections: one row per supported agent with status
  (not installed / connected / needs attention), Install and Remove buttons,
  and "Show what will change" (prints the JSON diff before installing).
- Wait time before falling back to the agent's own prompt (30-300 s).
- Closed-notch indicator on/off. Sound on/off.
- Auto-dismiss finished sessions after N minutes.
- Debug: record raw events to `~/.mynotch/logs/events.jsonl` (off by
  default; warns that prompts and commands are written to disk).

## 5. Jump to window

Phase 2, best effort:

1. Activate the host app via `NSRunningApplication` (bundle id from
   `__CFBundleIdentifier`, or walk up from `ppid` to the first GUI app).
2. Later, optional: exact tab/pane for iTerm2 (`ITERM_SESSION_ID` +
   AppleScript), Terminal.app (tty + AppleScript), tmux (`TMUX_PANE`),
   VS Code (`code --reuse-window <cwd>`). Each is its own small task; do not
   block Phase 2 on them.

## 6. Talking back to the agent

| Capability | Mechanism | Phase |
|---|---|---|
| Deny with a reason | `decision.message` on deny | 1 |
| Always allow a rule | `updatedPermissions` with the agent's suggestion (Claude Code) | 2 |
| Answer the agent's multiple-choice question | SPIKE: how Claude Code exposes its question tool to hooks; Agent Notch (MIT) does this for Claude Code and is the reference | later |
| Send a follow-up prompt | Queue text for a session in the notch; when that session's `Stop` hook fires, return a "continue with this instruction" decision immediately. No waiting at stop time. Claude Code caps consecutive continuations (8); honour `stop_hook_active` | later |
| Free typing into a live terminal session | Not possible through hooks. Out of scope | — |

## 7. Edge cases to handle and test

- App starts while agents are already running: sessions appear on their next
  event (no history). Fine.
- App quits with a card open: connections drop; clients exit 0; agents fall
  back to their own prompt.
- Same session sends a second request while the first is pending: queue.
- Two agents in the same project: separate rows (id includes session id).
- Very long commands / huge diffs: truncate the *preview* at 20k characters
  with a visible "+N more characters — open in terminal to review" line, and
  treat truncated requests as `caution` at minimum.
- Subagents: Codex subagents share the parent session id; Claude Code
  provides agent fields on some events. Phase 1 shows them inside the parent
  session's recent actions.
- Non-interactive agent runs (`claude -p`): no decision means deny there. A
  card still appears; that is useful, keep it.
- Screen locked or display asleep: do not auto-open; let requests time out
  to the agent's prompt.
- Multiple displays: open on the screen the coordinator has selected.

## 8. Acceptance (Phase 1)

See ROADMAP.md Phase 1. Manual test script lives there.

## 9. Reference implementations (read, do not copy blindly)

- Agent Notch — MIT — https://github.com/Forichok/Agent-Notch
  (Claude Code + Codex, hooks installer, shortcuts, risky-command styling)
- Casement — MIT — https://github.com/aviralgarg05/casement
  (Claude Code, Codex, Cursor integrations under `integrations/`)

MIT code may be reused in this GPL-3.0 project with attribution in
`THIRD_PARTY_LICENSES`.
