# Prompts for Claude Code — MyNotch

Copy-paste prompts. Use one per session. Start each new session with the
"resume" prompt if you are continuing a phase.

Tip: begin big sessions in plan mode (Shift+Tab until "plan mode") so Claude
Code shows its plan before touching files. Approve the plan, then let it
work.

---

## 0. First ever session — Phase 0A kickoff

```
You are working in my fork of boring.notch. I am turning it into "MyNotch",
a modular notch app. I am new to Swift and Xcode (I know Flutter, React and
Python), so explain what you do in plain language.

Before anything else, read in this order:
CLAUDE.md, docs/PRD.md, docs/ARCHITECTURE.md, docs/ROADMAP.md,
docs/DECISIONS.md, docs/PROGRESS.md, docs/MODULE_SYSTEM.md, docs/SETUP.md.

Then do ONLY Phase 0A from docs/ROADMAP.md ("Fork runs"):
1. Confirm the facts in CLAUDE.md about the upstream codebase still match
   this checkout. Tell me about any mismatch before changing anything.
2. Build the app unchanged with xcodebuild and report the result.
3. Rename the app identity (bundle id com.shravandige.mynotch, display name
   MyNotch) keeping the name in one constants file.
4. Disable the Sparkle auto-updater and its settings UI.
5. Add NOTICE.md.
6. Do the sandbox spike (Q1 / ADR-003): turn the app sandbox off, rebuild,
   and give me a checklist of upstream features to test by hand.

Rules: follow CLAUDE.md exactly. Tag every edit to an upstream file with
// MYNOTCH:. Build after each step. Commit after each step with a clear
message. Do not start 0B. When 0A is done, update docs/PROGRESS.md, then
stop and give me: what changed, which files, and exact steps to test it.

Start by showing me your plan for 0A.
```

## 1. Resume a phase (use at the start of any later session)

```
Read CLAUDE.md and docs/PROGRESS.md, then the docs relevant to the current
phase. Tell me in 5 lines where we are and what you will do this session.
Continue the current phase only. Same rules as always: small steps, build
and test after each, update docs/PROGRESS.md at the end, stop at the next
checkpoint.
```

## 2. Phase 0B + 0C — project structure and module system

```
Checkpoint 0A is approved. Read CLAUDE.md, docs/MODULE_SYSTEM.md,
docs/ARCHITECTURE.md (sections 3-5) and docs/ROADMAP.md Phase 0B and 0C.

Do 0B first:
- Create Packages/NotchCore (library NotchCore, executable notch-hook,
  tests) and get `swift test` passing.
- Spike Q2: add boringNotch/NotchModules as a file-system-synchronised
  folder (copy how boringNotch/private is declared in project.pbxproj) and
  add Packages/NotchCore as a local package of the app target. Prove a file
  in NotchModules that imports NotchCore compiles into the app. If editing
  project.pbxproj by hand is risky, stop and give me the Xcode steps from
  docs/SETUP.md section 6 instead. Back up project.pbxproj before editing.

Stop and show me 0B working. After I confirm, do 0C in this order:
AttentionQueue with tests -> NotchModule + ModuleRegistry -> NotchPresenter
-> upstream touch points -> LegacyFlagModule (spike Q3) ->
ModulesSettingsView -> DemoModule.

Finish with the Phase 0 acceptance list from the roadmap as a manual test
script for me. Update docs/PROGRESS.md and docs/DECISIONS.md. Stop at
Checkpoint 0.
```

## 3. Phase 1A — agent bridge plumbing and spikes

```
Checkpoint 0 is approved. We start the Agents module.
Read CLAUDE.md, docs/modules/AGENTS.md, docs/HOOK_PROTOCOL.md,
docs/SECURITY.md and docs/ROADMAP.md Phase 1.

First fetch the current official Claude Code hooks documentation
(https://code.claude.com/docs/en/hooks) and compare it with
docs/HOOK_PROTOCOL.md section 4. List every difference and update the doc.

Then do ONLY Phase 1A:
- WireMessage + WireCodec with tests.
- notch-hook client. It must exit 0 silently when the app is not running.
- AgentBridgeServer inside a minimal AgentsModule (no UI yet), following
  the socket rules S10-S14.
- Debug event recorder.
- Create a scratch project at ~/Developer/mynotch-scratch with a
  project-level .claude/settings.json containing our hook entries. Do NOT
  touch my global ~/.claude/settings.json in this phase.
- Tell me exactly what to run in that scratch project so we record one
  real payload for every event type. Save scrubbed copies as test fixtures.
- Answer spikes Q4, Q5 and Q6 with evidence and write them in
  docs/DECISIONS.md.

Stop at Checkpoint 1A and show me the recorded event stream and the spike
answers.
```

## 4. Phase 1B-1D — logic, UI, installer

```
Checkpoint 1A is approved. Continue Phase 1 with 1B, then 1C, then 1D from
docs/ROADMAP.md.

1B is test-first: ClaudeCodeAdapter against the fixtures, AgentReducer,
RiskClassifier (table of at least 60 commands), text sanitiser.
1C: AgentStore, Agents tab, PermissionCardView (every rule S1-S9 in
docs/SECURITY.md), closed-notch indicator, auto-open and auto-close.
1D: ConfigMerger with its tests, HookInstaller, Agents settings view.

After 1C, stop and let me try the permission card with the scratch project
before you build the installer. The installer must show me the exact change
and make a backup before it writes to ~/.claude/settings.json.

End with the Phase 1 acceptance script and the security checklist from
docs/SECURITY.md section 5, run what you can yourself, and list what I
must test by hand. Update docs/PROGRESS.md. Stop at Checkpoint 1.
```

## 5. Phase 2 — detail, Codex, polish

```
Checkpoint 1 is approved. My feedback from using it is in
docs/PROGRESS.md under "Owner feedback log"; handle those items first.

Then do Phase 2 from docs/ROADMAP.md. For Codex, first fetch the current
official Codex hooks documentation and reconcile it with
docs/HOOK_PROTOCOL.md section 5, then record real Codex payloads as
fixtures before writing CodexAdapter. "Always allow" must follow ADR-007
and rule S8. Stop at Checkpoint 2 with the acceptance script.
```

## 6. Phase 3A — send to device via share menu (Blip)

```
Checkpoint 2 is approved. Read docs/modules/SEND_TO_DEVICE.md and
docs/ROADMAP.md Phase 3.

Start with spike Q8 only: investigate what automation surface Blip exposes
on my Mac (share menu service, URL schemes in its Info.plist, any CLI in
the bundle, AppleScript dictionary, Services). Run the checks, show me the
evidence, and write the result in docs/DECISIONS.md. Do not automate Blip's
UI with Accessibility.

Then build 3A: TransferProvider protocol, SendToDeviceModule, settings,
ShareMenuProvider (reuse upstream QuickShareService / ShareServiceFinder),
and device/service drop targets beside the shelf during a drag.
Stop at Checkpoint 3A.
```

## 7. Phase 3B — LocalSend provider

```
Checkpoint 3A is approved. Build Phase 3B from docs/ROADMAP.md.
First fetch https://github.com/localsend/protocol and reconcile it with
docs/modules/SEND_TO_DEVICE.md section 3.2.

Order: protocol types + codec + upload client in NotchCore with a stub
server test suite -> discovery -> pinning with fingerprint (S23, S24) ->
Local Network permission flow -> progress / cancel / retry UI -> shelf
"Send to" menu. Stop at Checkpoint 3 with the acceptance script.
```

---

## Reusable prompts

### Give feedback after a review

```
I tested <checkpoint>. Feedback:
1. <what I saw> — expected <what I wanted>
2. ...
Log this in docs/PROGRESS.md under "Owner feedback log". For each item say
whether it is a bug, a design change (needs my decision) or a new idea
(parking lot). Fix the bugs now, one at a time, building and testing after
each. Ask me about the design changes before doing them.
```

### Something broke

```
<paste the exact error or describe what happened and what you did>
Do not guess. Reproduce it first, find the root cause, explain it to me in
plain language, then fix it and add a test that would have caught it.
```

### Explain the code to me

```
Explain <file or feature> as if I know Flutter/Dart but not Swift. Map the
Swift concepts to their Flutter equivalents where that helps (for example
ObservableObject vs ChangeNotifier, @MainActor vs the UI isolate).
Then show me which 3 files I should read first to understand it.
```

### Add a new module later

```
I want a new module: <problem I noticed and what I want the notch to do>.
Do not write code yet. Read docs/MODULE_SYSTEM.md and the existing module
specs, then write docs/modules/<NAME>.md in the same format (what it does,
how, UI, settings, security, spikes, acceptance) and add a phase to
docs/ROADMAP.md. Ask me the questions you need answered. Wait for my
approval of the spec.
```

### Security review before merging a phase

```
Act as a security reviewer who did not write this code. Review the diff of
this phase branch against docs/SECURITY.md. For every requirement S1-S27
that applies, say pass / fail / not applicable with the file and line that
proves it. Then try to break it: list concrete attacks through the socket,
the hook payload text and the config installer, and test the ones you can.
Report findings before fixing anything.
```

### Sync with upstream

```
Between phases only: fetch upstream, merge upstream/main into mynotch/main,
resolve conflicts keeping our // MYNOTCH: edits, rebuild, run tests, and
run the Phase 0 acceptance list again. Tell me what upstream changed that
affects us.
```
