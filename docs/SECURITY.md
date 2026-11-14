# Security — MyNotch

MyNotch sits between a human and tools that can run arbitrary commands on
their machine. A bug here can approve something the user never saw. Treat
this file as requirements, not advice.

## 1. Assets

1. The user's approval decision (integrity: only a real click may allow).
2. The user's agent configuration files (`~/.claude/settings.json`, etc.).
3. Content shown in the notch: commands, file diffs, prompts, later messages
   and mail (confidentiality).
4. Files sent to devices (go to the intended device only).

## 2. Trust boundaries

```
agent process --stdin--> notch-hook --unix socket--> MyNotch.app --click--> human
   (untrusted text)       (same user)   (same user)                (trusted)
```

- Everything arriving from an agent is **untrusted text**: tool names,
  commands, descriptions, file contents, session titles. A prompt-injected
  agent may craft these to mislead the user.
- Other local processes running as the same user can connect to the socket.
  They can already do anything the user can, so the socket adds no new
  privilege — **provided a socket message can never cause an approval by
  itself.** Approval originates only from a UI click in our process.
- Other users on the Mac and anything on the network must have no access.

## 3. Requirements

### Approvals

- **S1** No code path returns `allow` without a click on the Allow control of
  the card for that exact request id. No timers, no defaults, no "remember"
  logic in v1.
- **S2** Timeout, crash, quit, socket error, malformed data -> no decision
  (empty stdout, exit 0). Never `allow`. Never block the agent forever.
- **S3** The card shows the **complete** command / arguments that will run.
  If the preview is truncated for size, say so and classify at least
  `caution`.
- **S4** The agent's own description of the action is labelled as the
  agent's words and is visually secondary to the real command.
- **S5** Render all agent text as plain text. No markdown, no links, no
  attributed strings built from input. Replace control characters, ANSI
  escapes and bidirectional override characters (U+202A-U+202E,
  U+2066-U+2069) with visible placeholders. Show a warning when the command
  contains non-ASCII look-alike characters.
- **S6** New cards are inert for the first 400 ms after they appear and
  after the card content changes, so a click aimed at something else cannot
  approve a request that just popped up.
- **S7** Keyboard approve shortcuts are unassigned by default and never act
  on `danger` requests.
- **S8** "Always allow" (Phase 2) only ever applies a rule the agent itself
  suggested, shows the exact rule text first, and needs its own click.
- **S9** A request is bound to its connection. If the connection closes, the
  card is removed and a late click does nothing.

### Risk classifier (`NotchCore/Agents/RiskClassifier.swift`)

Advisory only — it changes styling and shortcut eligibility, it never
approves or denies.

`danger` when the command (after trimming, across `;`, `&&`, `||`, pipes and
subshells) matches any of:

- `rm` with recursive and force flags; `rm` on `/`, `~`, `$HOME`, `*`
- `sudo`, `su`, `doas`
- `git push` with `--force` / `-f`; `git reset --hard`; `git clean -fd`
- pipe to a shell: `curl|wget ... | sh|bash|zsh`; `eval`; `base64 -d | sh`
- `chmod -R`, `chown -R`, `chmod 777`
- `dd`, `mkfs`, `diskutil erase*`
- writes to `~/.ssh`, `~/.aws`, `~/.gnupg`, keychains, shell rc files,
  `~/.claude`, `~/.codex`, `~/.cursor`, `~/.mynotch`, LaunchAgents
- `launchctl`, `crontab`, `osascript`, `defaults write`
- `npm publish`, `docker push`, `gh repo delete`, `terraform destroy`,
  `kubectl delete`
- reading secrets then sending them out (heuristic: `cat .env` / `printenv`
  combined with `curl`/`nc`)

`caution` when: network tools (`curl`, `wget`, `nc`, `ssh`, `scp`), package
installs, writes or deletes outside the session `cwd`, MCP tools, truncated
previews, commands over 500 characters, anything containing obfuscation
hints (`$(...)` inside quotes, hex/base64 blobs).

Everything else `normal`. Tests: a table of at least 60 commands with the
expected class, including evasions (extra spaces, quoting, `\rm`, env var
prefixes, `command rm`).

### Socket and files

- **S10** `~/.mynotch` is created `0700`; the socket `0600`; verify
  ownership and mode at startup and refuse to listen otherwise. Remove a
  stale socket file only if it is a socket owned by the current user.
- **S11** On connect, check the peer's uid (`getpeereid`) equals ours; drop
  otherwise.
- **S12** Limit message size (1 MB), concurrent connections (64) and pending
  requests per session (20). Excess is dropped with a log line.
- **S13** Unix socket only. No TCP listener for the agent bridge.
- **S14** The hook client never executes anything it receives and prints
  only the `stdout` field from the app.

### Config installer

- **S15** Back up before every write; merge, never overwrite; write
  atomically; validate after writing; restore on failure.
- **S16** Refuse to modify a file that does not parse as strict JSON.
- **S17** Show the exact change before applying it the first time.
- **S18** Uninstall removes only our entries.

### Data handling

- **S19** Sessions, commands, prompts, diffs, messages: memory only.
- **S20** The debug event log is off by default, clearly warned, local,
  size-capped, and deletable from settings. Never enabled by a hidden flag.
- **S21** No analytics, crash reporting or update checks to third-party
  servers. The upstream Sparkle updater is disabled (ADR-008).
- **S22** Clear in-memory content on screen lock.

### Send to device

- **S23** LocalSend HTTPS peers: accept a certificate only when its SHA-256
  matches the pinned fingerprint. No blanket "allow any certificate".
- **S24** Pinning a device shows its name and fingerprint and needs a click.
  A pinned name appearing with a different fingerprint is shown as a
  **different, unpinned** device with a warning.
- **S25** Only send files the user dropped or picked. No background sends.
- **S26** Discovery traffic only while needed (see module spec).

### Notification mirror

- **S27** Memory only, cleared on lock/quit/disable; hidden during screen
  sharing by default; Full Disk Access mode is opt-in with a plain warning.

## 4. Sandbox

Upstream's main app is sandboxed with an unsandboxed XPC helper. The fork
turns the app sandbox off (ADR-003) because the Agents module must listen on
a socket outside the container and edit agent config in the home directory,
and later modules need Accessibility. Consequences to accept knowingly:

- A bug in MyNotch is no longer contained by the sandbox. Keep the attack
  surface small: one local socket, strict parsing, no scripting, no web
  views fed with untrusted content.
- Hardened Runtime stays on. Do not add `disable-library-validation`,
  `allow-dyld-environment-variables` or `get-task-allow` to release builds.
- Revisit before any public release: moving the bridge and installer into
  the helper would let the main app be sandboxed again.

## 5. Security review checklist (run at every phase checkpoint)

- [ ] Kill the app with a card open: agent falls back to its own prompt.
- [ ] Send garbage, oversized and truncated JSON to the socket: no crash,
      no card, no decision.
- [ ] Connect from a second local user: refused.
- [ ] Request text containing ANSI escapes, RTL overrides, newlines, 100k
      characters: rendered safely, flagged.
- [ ] A `danger` command: red, reason shown, no shortcut works.
- [ ] Click exactly where a card is about to appear: nothing approved
      (inert window).
- [ ] Install -> uninstall on a config file with other hooks: file is
      byte-identical to the original apart from whitespace normalisation.
- [ ] Module switched off: socket file gone, no listener
      (`lsof -U | grep mynotch` is empty).
- [ ] `grep -R` the logs directory after a session with the debug log off:
      no prompts, commands or message text.

## 6. Reporting

Until the project is public: issues go to the owner. When public, add a
`SECURITY.md` at the repo root with a private reporting address (upstream's
existing file must be replaced so reports do not go to the upstream team).
