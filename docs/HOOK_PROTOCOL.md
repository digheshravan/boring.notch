# Hook Protocol — agents <-> MyNotch

How a coding agent talks to MyNotch. Three layers:

1. **Agent hook config** — tells the agent to run `notch-hook` on events.
2. **`notch-hook`** — a tiny CLI. Transport only. No agent knowledge.
3. **Bridge + adapters** (inside the app) — understand each agent's JSON.

Labels used below: **VERIFIED** = checked against official docs on
2026-11-14. **UNVERIFIED** = from third-party write-ups; confirm against the
official docs and a real payload before building.

Always re-read the official hook docs at the start of an agent phase. These
formats change.

---

## 1. `notch-hook` CLI

```
notch-hook --agent <claude-code|codex|antigravity|cursor> [--wait <seconds>]
```

Behaviour:

1. Read all of stdin (the agent's hook JSON). Do not parse it beyond
   extracting `hook_event_name` (needed only to choose wait vs no-wait when
   `--wait` is not given).
2. Connect to `~/.mynotch/bridge.sock`. If that fails: **exit 0, print
   nothing.**
3. Send one request line (see section 2).
4. If the app answers `"hold": false`: exit 0, print nothing.
5. If the app answers `"hold": true`: keep the connection open and wait for
   the final response, up to `--wait` seconds (default 90).
6. Print the response's `stdout` string exactly, exit with its `exit` code.
7. On timeout, socket error, malformed response, or any signal: **exit 0,
   print nothing.**

"Exit 0 with no output" means "no decision" for Claude Code and Codex: the
agent falls back to its normal prompt. That is the fail-open rule. The client
must never print anything that could be read as a decision unless the app
sent it.

Other requirements:

- No dependencies beyond Foundation. Start-up under ~20 ms.
- No logging to stdout/stderr (agents may read both). Debug logging only to
  `~/.mynotch/logs/hook.log` when env `MYNOTCH_DEBUG=1`.
- Context it adds to the request: parent pid, cwd, and these env vars when
  present: `TERM_PROGRAM`, `__CFBundleIdentifier`, `ITERM_SESSION_ID`,
  `TMUX`, `TMUX_PANE`, `VSCODE_PID`, `TERM_SESSION_ID`. They let the app
  bring the right window forward later.

## 2. Wire protocol (client <-> app)

Unix domain stream socket, UTF-8, newline-delimited JSON, one request per
connection.

Request (client -> app):

```json
{"v":1,"id":"<uuid>","agent":"claude-code","ppid":4242,"cwd":"/path",
 "env":{"TERM_PROGRAM":"iTerm.app"},"payload":{ ...raw hook JSON... }}
```

First response (app -> client), always sent within 200 ms:

```json
{"v":1,"id":"<uuid>","hold":false}
```
or `{"v":1,"id":"<uuid>","hold":true}` when the event needs a human.

Final response (only after `hold:true`):

```json
{"v":1,"id":"<uuid>","stdout":"<exact text for the agent>","exit":0}
```

`stdout` may be empty: that means "no decision, use your normal prompt".

Rules:

- The app treats the socket closing as "request withdrawn" and removes the
  card. This is how the notch learns the agent was answered elsewhere.
- Requests over 1 MB are rejected (connection closed).
- Unknown `v` -> reply `{"hold":false}` and log.
- The app never trusts `agent` or `payload` beyond what is needed to render
  them. All text is displayed as plain text (no markdown, no links).

## 3. Normalised model (inside `NotchCore`)

```swift
enum AgentKind: String { case claudeCode, codex, antigravity, cursor }

enum AgentEvent {
    case sessionStarted(SessionRef, source: String?)
    case promptSubmitted(SessionRef, preview: String)
    case toolStarted(SessionRef, ToolCall)
    case toolFinished(SessionRef, ToolCall, ok: Bool)
    case permissionRequested(SessionRef, PermissionRequest)
    case notification(SessionRef, kind: String, message: String)
    case taskProgress(SessionRef, done: Int, total: Int, current: String?)
    case stopped(SessionRef, lastMessage: String?)
    case sessionEnded(SessionRef, reason: String?)
}

struct PermissionRequest {
    let id: String              // wire id
    let tool: String            // "Bash", "Edit", "mcp__x__y" ...
    let summary: String         // one line for the card title
    let detail: String          // FULL command / diff / arguments
    let risk: Risk              // from RiskClassifier
    let suggestions: [PermissionSuggestion]   // agent-provided "always allow" rules
}

enum PermissionDecision {
    case allow
    case allowAlways(PermissionSuggestion)    // Phase 2
    case deny(message: String?)
    case passthrough                          // let the agent ask itself
}
```

Each adapter implements:

```swift
protocol AgentAdapter {
    static var kind: AgentKind { get }
    func normalise(_ raw: JSONValue, context: WireContext) -> AgentEvent?
    func needsHold(_ raw: JSONValue) -> Bool
    func encode(_ decision: PermissionDecision, for raw: JSONValue) -> HookOutput  // stdout + exit
}
```

Adapters are pure functions and are tested with recorded fixtures.

---

## 4. Claude Code adapter — VERIFIED

Docs: https://code.claude.com/docs/en/hooks

Config file: `~/.claude/settings.json` (user scope; applies to the CLI and
the VS Code extension).

Common input fields on every event: `session_id`, `transcript_path`, `cwd`,
`hook_event_name`, usually `permission_mode`.

| Event | Extra input we use | Hold? | Maps to |
|---|---|---|---|
| `SessionStart` | `source` (startup/resume/clear/compact/fork), `model`, `session_title` | no | sessionStarted |
| `UserPromptSubmit` | `prompt` | no | promptSubmitted |
| `PreToolUse` | `tool_name`, `tool_input`, `tool_use_id` | no | toolStarted (+ taskProgress when the tool is a task/todo tool) |
| `PostToolUse` | `tool_name`, `tool_response`, `duration_ms` | no | toolFinished |
| `PostToolUseFailure` | same | no | toolFinished(ok:false) |
| `PermissionRequest` | `tool_name`, `tool_input`, optional `permission_suggestions`, `mcp_server` | **yes** | permissionRequested |
| `Notification` | `message`, `title`, `notification_type` (`permission_prompt`, `idle_prompt`, `agent_needs_input`, ...) | no | notification |
| `Stop` | `last_assistant_message`, `stop_hook_active`, `background_tasks` | no (Phase 1) | stopped |
| `SubagentStop` | `last_assistant_message` | no | toolFinished-like status |
| `SessionEnd` | `reason` | no | sessionEnded |

Also available and useful for progress: `TaskCreated`, `TaskCompleted`
(fields include `task_id`, `task_subject`). SPIKE: record real payloads and
decide whether progress n/m comes from these events, from the task/todo
tool inputs seen in `PreToolUse`, or both.

Decision output for `PermissionRequest` (print on stdout, exit 0):

```json
{"hookSpecificOutput":{"hookEventName":"PermissionRequest",
  "decision":{"behavior":"allow"}}}
```
```json
{"hookSpecificOutput":{"hookEventName":"PermissionRequest",
  "decision":{"behavior":"deny","message":"<note for Claude>"}}}
```

Other documented decision fields: `updatedInput` (allow), `updatedPermissions`
(allow; array of permission update entries, this is how "always allow" is
expressed, using an entry from `permission_suggestions`), `interrupt` (deny;
stops Claude when true).

Facts that shape the design:

- `allow` does not override the user's deny rules; they are still evaluated.
- No decision from any hook -> normal permission flow (interactive sessions).
  In sessions that cannot show a prompt, no decision means the call is
  denied.
- Exit code 2 does not deny for this event. Deny only through the JSON.
- Command hooks default to a 600 s timeout; a timed-out hook yields no
  decision.
- `"async": true` on a command hook runs it in the background and ignores
  its output. Use it for every status event so MyNotch can never slow the
  agent. Do **not** use it on `PermissionRequest`.
- `SessionEnd` hooks get about 1.5 s in total. The client must send and exit
  immediately.

SPIKE (first task of Phase 1): while a `PermissionRequest` hook is still
running, can the user also answer in the terminal? The docs do not say.
Test it. If yes: the socket-close rule handles it. If the terminal prompt
only appears after the hook returns: keep `--wait` short enough to be
tolerable (default 90 s) and show a countdown on the card.

Hook entries the installer adds (merge into existing `hooks`, never replace):

```json
{
  "hooks": {
    "PermissionRequest": [
      { "hooks": [ { "type": "command",
          "command": "\"$HOME/.mynotch/bin/notch-hook\" --agent claude-code --wait 90",
          "timeout": 120 } ] }
    ],
    "PreToolUse": [
      { "hooks": [ { "type": "command",
          "command": "\"$HOME/.mynotch/bin/notch-hook\" --agent claude-code",
          "async": true } ] }
    ]
  }
}
```

Repeat the async form for `SessionStart`, `UserPromptSubmit`, `PostToolUse`,
`PostToolUseFailure`, `Notification`, `Stop`, `SubagentStop`, `SessionEnd`,
`TaskCreated`, `TaskCompleted`. Confirm in the docs whether an omitted
`matcher` matches all tools or whether `"*"` is required. Every entry we add
is identifiable by the `notch-hook` path so uninstall removes exactly ours.

## 5. Codex adapter — VERIFIED (shape), verify details when building

Docs: https://developers.openai.com/codex/hooks

Config: `~/.codex/hooks.json` (or `[hooks]` in `~/.codex/config.toml`; use
only `hooks.json`). Same nesting as Claude Code: event -> matcher groups ->
`hooks` array of `{ "type": "command", "command": ..., "timeout": ... }`.

Events: `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PermissionRequest`,
`PostToolUse`, `Stop`, `SubagentStart`, `SubagentStop`, `SessionEnd`, plus
compaction/model events we ignore.

Common input: `session_id`, `transcript_path` (may be null), `cwd`,
`hook_event_name`, `model`. `PermissionRequest` adds `turn_id`, `tool_name`
(`Bash`, `apply_patch`, MCP names), `tool_input` (`command`, `description`),
`permission_mode`.

Decision output: same envelope as Claude Code with
`decision.behavior` = `allow` or `deny` (+ `message`). **Do not send
`updatedInput`, `updatedPermissions` or `interrupt` to Codex.** So "always
allow" is Claude Code only.

Differences that matter:

- **Trust.** Codex skips new or changed hooks until the user reviews and
  trusts them with `/hooks` in the CLI. The installer must tell the user to
  do this, and the Agents settings page should show "Codex: waiting for you
  to trust the hook" until the first event arrives.
- If several hooks decide, any deny wins. No decision -> normal prompt.
- `SessionEnd` default timeout is 1 s.
- Whether hooks fire in the Codex IDE extension / desktop app is not
  documented. SPIKE.

## 6. Antigravity adapter — UNVERIFIED (later phase)

Docs to read first: https://antigravity.google/docs/hooks

What third-party sources say:

- Hooks are registered in a `hooks.json` (global under `~/.gemini/config/`,
  or as a plugin folder `~/.gemini/config/plugins/<name>/` containing
  `plugin.json` + `hooks.json`; repo-level under `.agents/hooks.json`).
- Lifecycle events include `PreToolUse`, `PostToolUse`, `PreInvocation`,
  `PostInvocation`, `Stop`.
- There is no separate "permission request" event. `PreToolUse` fires for
  **every** tool call with the tool name and arguments on stdin and can
  return allow / deny / force-ask.
- Running the CLI with `--dangerously-skip-permissions` may bypass hooks.

Design consequence: the adapter must decide which calls deserve a card.
Read-only tools (view/search/list) -> return no decision immediately.
Commands, writes, MCP calls -> hold and show a card, **unless** Antigravity
would not have prompted anyway. Find out whether the hook can see the
agent's own verdict; if not, mirror the user's permission preset from
Antigravity settings. Ship as status-only first, gating second.

## 7. Cursor adapter — status only until proven safe

Docs: https://cursor.com/docs (Hooks). Config: `~/.cursor/hooks.json`:

```json
{ "version": 1,
  "hooks": { "beforeShellExecution": [ { "command": "<path> --agent cursor" } ] } }
```

Events include `sessionStart`, `beforeSubmitPrompt`, `preToolUse`,
`beforeShellExecution`, `beforeMCPExecution`, `beforeReadFile`,
`afterFileEdit`, `stop`. Output shape:
`{"permission":"allow|deny|ask","user_message":"...","agent_message":"..."}`.

Why status only: forum reports through 2026 describe `ask` and `allow` being
ignored in several versions, the allow-list taking precedence over hook
answers, and **malformed or missing hook output being treated as allow**.
That last point breaks our fail-open rule (for Cursor, "no output" may mean
"run it"). Do not gate Cursor commands from the notch until a spike on the
owner's installed version proves that (a) a missing answer falls back to
Cursor's own prompt and (b) deny/allow are honoured. Until then: subscribe
to non-blocking events only and show sessions as read-only.

## 8. VS Code

VS Code is a host, not an agent. Claude Code and Codex running inside it use
the same user-level hook config, so they appear automatically (Codex: SPIKE).
GitHub Copilot's agent mode is out of scope.

## 9. Installer (`HookInstaller` + `NotchCore/Install/ConfigMerger`)

Per agent: `status()` -> notInstalled / installed / outdated / agentNotFound;
`install()`; `uninstall()`.

`install()`:

1. Ensure `~/.mynotch/bin/notch-hook` exists and is executable.
2. Read the agent config (create `{}` if the file does not exist; never
   create the agent's directory if the agent is not installed).
3. Write a backup beside it: `<name>.mynotch-backup-<yyyyMMdd-HHmmss>`.
4. Merge our entries. Preserve every existing key, hook and ordering.
   Idempotent: running twice changes nothing.
5. Write atomically (temp file + rename). Preserve file permissions.
6. Re-read and validate JSON. On any failure restore the backup and report.

`uninstall()` removes only entries whose command contains
`/.mynotch/bin/notch-hook`, and deletes now-empty arrays/objects it created.

`ConfigMerger` is pure (`Data in -> Data out`) and has tests for: empty
file, file with other hooks on the same event, already installed,
malformed JSON (must refuse, not overwrite), comments/trailing commas
(refuse and tell the user), uninstall leaves foreign hooks untouched.

## 10. Test fixtures

Phase 1 starts by turning on the debug event log, running real sessions and
saving one payload per event type to
`Packages/NotchCore/Tests/NotchCoreTests/Fixtures/claude-code/*.json`
(scrub paths and prompts). Adapters are then written test-first against
these files. Do the same per agent.
