# Module spec — Notification mirror (later phase)

Module id: `mirror`. Default: **disabled**. Permissions: Accessibility
(approach A) or Full Disk Access (approach B).

Status: specified for planning. Not scheduled until Phases 0-3 are reviewed.
Everything here starts with a spike.

## 1. What the owner wants

- WhatsApp messages in the notch, but only from chats and groups he flags as
  important.
- Important mail in the notch instead of as a banner on the right.
- In general: the user decides which apps and which senders reach the notch.

## 2. The hard truth

macOS has no public API for one app to read another app's notifications.
There are two workable routes, each with a cost:

### Approach A — watch notification banners (Accessibility)

Observe the Notification Center UI process with the Accessibility API and
read each banner's app name, title, subtitle and body as it appears.

- Proven by other notch apps (Casement lists mirrored notifications under
  the Accessibility permission).
- Only sees notifications that are actually displayed as banners. If the
  user sets an app's alert style to "None" to stop the banner on the right,
  there is nothing to observe.
- So with approach A the notification appears **both** on the right and in
  the notch. It cannot be "instead of".
- The banner view hierarchy changes between macOS releases. Expect breakage
  on OS updates.
- Upstream keeps Accessibility calls in the unsandboxed XPC helper; with the
  sandbox off (ADR-003) we can call them directly, but keep them behind one
  small service class.

### Approach B — read the Notification Center database (Full Disk Access)

macOS stores delivered notifications in a local database
(`~/Library/Group Containers/group.com.apple.usernoted/` on recent macOS).
With Full Disk Access the app can watch it for new rows.

- Works even when the app's alert style is "None", so the banner on the
  right can be turned off and the notch becomes the only surface. This is
  what the owner actually asked for.
- Full Disk Access is a very broad permission. For a security-minded,
  open-source app this needs a clear explanation and must stay optional.
- The database format is private and has moved before.

### Decision rule

Spike both on the owner's macOS version. Prefer **A as the default**
(narrower permission) and offer **B as an opt-in "replace banners" mode**
with a plain warning. If only one works reliably, ship that one and say so.

## 3. Filtering rules

```
MirrorRule
  app            bundle id (e.g. WhatsApp desktop, com.apple.mail)
  mode           all | onlyMatching | off
  matchers       [String]   chat / group / sender names, case-insensitive
  showBody       Bool       show message text or just "New message from X"
  band           message (default) | timeSensitive
```

- WhatsApp: the banner title is the chat or group name (for groups the
  sender appears in the subtitle or body prefix — confirm in the spike).
  "Flag as important" = add that exact name to `matchers`. A rename breaks
  the match; the settings page shows recently seen names so re-flagging is
  one click.
- Mail: do not reimplement Apple's "important" logic. Mail's own settings
  can restrict notifications (e.g. Primary inbox, VIPs, Contacts). Set that
  in Mail, then mirror everything Mail notifies (`mode: all`). Optional
  extra matchers for senders.
- Any other app can be added with the same rule shape. That is the general
  feature: "which apps and which senders may use my notch".

## 4. UI

- message-band sneak peek beside the notch: app icon, sender, first line.
  Expands on hover to the full text. Auto-dismiss after N seconds (setting).
- Click: activate the source app (cannot deep-link to the exact chat).
- No reply box in v1 (replying would need UI automation of WhatsApp).
- A recent list (last 10, memory only) in a Mirror tab, clearable.

## 5. Privacy rules

- Content lives in memory only. Never logged, never written to disk, never
  included in the debug event log.
- Cleared on lock, on quit, and when the module is switched off.
- Option to hide message bodies while the screen is being shared or
  recorded. SPIKE: how to detect this reliably. (Upstream's
  `SharingStateManager` is unrelated: it tracks share-sheet sessions.)
- Respect macOS Focus: when a Focus mode is silencing the source app, do not
  show it in the notch.

## 6. Phone-side notifications (idea, not planned)

Because the owner's phone is Android, notifications could instead come from
the phone itself (Android notification listener -> Mac), which also covers
apps with no Mac client. That needs an Android companion app or a KDE
Connect-compatible implementation. Parking lot.

## 7. Spike checklist

1. On the owner's macOS version: can the Accessibility API read banner
   title/subtitle/body for WhatsApp and Mail? Record the element hierarchy.
2. Group messages: where do group name and sender appear?
3. With alert style "None": does anything observable still happen (A)?
   Does the database still receive the row (B)?
4. How quickly after delivery can each approach see the notification?
5. What happens on notification stacks / "3 new messages" summaries?
6. Does Focus mode suppress banners before we see them?

Write the findings into DECISIONS.md before any feature code.
