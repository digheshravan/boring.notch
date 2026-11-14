# Start here — MyNotch planning set

Prepared 2026-11-14. Everything Claude Code needs to build MyNotch step by
step, with you reviewing between steps.

## Tomorrow morning, in order

1. **Do `docs/SETUP.md` sections 1-4** (install Xcode 26, fork, clone, copy
   these files into the repo, build boring.notch once unchanged).
   Do not skip the unchanged build: it proves your toolchain works before
   any of our changes.
2. **Open Claude Code in the repo**, switch to Opus, enter plan mode.
3. **Paste prompt 0 from `PROMPTS.md`** (Phase 0A kickoff).
4. When it stops at a checkpoint, **test by hand** using the steps it gives
   you, then use the "Give feedback after a review" prompt.
5. Move to the next prompt only after you are happy with the checkpoint.

Expect Phase 0 (the base) to take the first day or two. The agent approval
feature is Phase 1. That order is deliberate: every later feature plugs
into the base.

## What each file is for

| File | Purpose | Who reads it |
|---|---|---|
| `CLAUDE.md` | Standing instructions, rules, commands, codebase facts. Loaded automatically by Claude Code every session | Claude Code |
| `PROMPTS.md` | Copy-paste prompts per phase + reusable ones | You |
| `docs/PRD.md` | What we are building, for whom, why it is different, user stories | Both |
| `docs/ARCHITECTURE.md` | How it fits into boring.notch; exact files to touch | Claude Code |
| `docs/MODULE_SYSTEM.md` | The on/off module design: protocol, registry, attention arbiter | Claude Code |
| `docs/HOOK_PROTOCOL.md` | How agents talk to the app; per-agent formats | Claude Code |
| `docs/modules/AGENTS.md` | Agents feature spec (the video idea) | Both |
| `docs/modules/SEND_TO_DEVICE.md` | Mac to Android file sending (Blip, LocalSend, AirDrop) | Both |
| `docs/modules/NOTIFICATION_MIRROR.md` | WhatsApp flagged chats + important mail (later) | Both |
| `docs/SECURITY.md` | Security requirements and review checklist | Both |
| `docs/DECISIONS.md` | Decisions made and open questions | Both |
| `docs/ROADMAP.md` | Phases, tasks, acceptance tests, checkpoints | Both |
| `docs/SETUP.md` | One-time manual setup | You |
| `docs/PROGRESS.md` | Living log; Claude Code updates it each session | Both |

## Decisions I made for you (change any of them)

1. **Name:** "MyNotch" as a placeholder; bundle id `com.shravandige.mynotch`.
2. **Base:** fork of boring.notch `main` (v2.7.3). Licence stays GPL-3.0.
3. **Order:** base and module system first, then agents, then file sending.
4. **App sandbox off** in the fork, because the agents feature needs it.
   This rules out the App Store and is flagged for review before a public
   release.
5. **Never auto-approve.** Every allow is your click. Dangerous commands are
   highlighted and cannot be approved by shortcut.
6. **First agent: Claude Code. Second: Codex.** Antigravity and Cursor come
   after v1 because their hook behaviour could not be confirmed from
   official docs; Cursor starts as view-only.
7. **File sending:** Blip through the Mac share menu first (may need one
   extra click in Blip's window), then a native LocalSend sender for true
   one-drop sends.
8. **Existing boring.notch features** show up as switches on the new Modules
   page straight away, by wrapping their existing settings rather than
   rewriting them.

## Things that are not yet known (the plan tests them first)

- Whether you can still answer in the terminal while the notch card is
  waiting (Claude Code docs do not say).
- How much Blip can be automated on macOS.
- Whether WhatsApp/Mail notifications can replace the system banner rather
  than duplicate it (needs Full Disk Access if so).
- Whether everything in boring.notch still works with the sandbox off.

Each of these is a marked spike in the roadmap with a fallback.

## What I checked and what I did not

Checked on 2026-11-14: the boring.notch source (structure, settings keys,
entitlements, project format), the official Claude Code hooks reference,
the official Codex hooks reference, and the LocalSend protocol document.

Not checked on a real machine: nothing here has been compiled or run. File
paths and hook schemas come from reading source and docs. The first prompt
tells Claude Code to confirm the codebase facts before changing anything.
