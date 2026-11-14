# Module spec — Send to device

Module id: `send`. Default: enabled once at least one provider is available.
Permissions: Local Network (LocalSend provider only).

## 1. What it does

Drag a file to the notch, drop it on a device, it arrives on that device.
An AirDrop-like experience for people whose phone is not an iPhone. How the
file travels is a provider the user chooses.

## 2. Why providers

The owner uses Blip today (also on his Windows laptop). Blip is free for
personal use but closed source, and no command-line or URL-scheme interface
is documented. So:

- Keep Blip usable through whatever surface it does expose.
- Offer an open-source path that we control end to end (LocalSend).
- Keep AirDrop available for the day the user owns an iPhone.

Switching ecosystem = toggling providers in settings.

## 3. Providers

### 3.1 `ShareMenuProvider` (Phase 3a) — covers Blip and AirDrop

macOS apps can add themselves to the system share menu. Upstream
boring.notch already enumerates those services
(`ShareServiceFinder.findApplicableServices`, `QuickShareService`) and can
invoke one by title.

- Target list = share services the user ticks in settings (e.g. "Blip",
  "AirDrop").
- `send()` = `NSSharingService.perform(withItems:)` for that service.
  Upstream's `SharingStateManager` (`boringNotch/models/`) keeps the notch
  open while a share session is active; reuse it.
- Limitation: the service's own window usually asks which device. That is
  one extra click, outside our control.

SPIKE (first task of Phase 3) — answer these on the owner's Mac and record
the results in DECISIONS.md:

1. Does "Blip" appear in the share services list for a file?
2. Does invoking it open a device picker, or send to a default device?
3. Does `/Applications/Blip.app/Contents/Info.plist` declare URL schemes
   (`CFBundleURLTypes`)? Try them.
4. Is there a CLI inside the bundle (`Contents/MacOS`, `Contents/Resources`)
   or an AppleScript dictionary (`sdef /Applications/Blip.app`)?
5. Does Blip register Finder Quick Actions / Services
   (`pbs -dump_pboard`, System Settings > Keyboard > Services)?

Outcome decides whether Blip can be a true one-drop target (pinned device)
or stays a two-step target. Do not automate Blip's UI with Accessibility
clicks; that is brittle and out of scope.

### 3.2 `LocalSendProvider` (Phase 3b) — one-drop, open source

LocalSend is an open-source cross-platform transfer app with a documented
protocol (confirm its licence before copying any code; we implement the
protocol ourselves, so normally none is copied). The phone runs the LocalSend app; MyNotch speaks the
protocol natively, so no LocalSend app is needed on the Mac for sending.

Protocol v2 essentials (VERIFIED against
https://github.com/localsend/protocol on 2026-11-14):

- Discovery: UDP multicast `224.0.0.167:53317`. Announcement JSON:
  `alias`, `version` ("2.0"), `deviceModel`, `deviceType`
  (mobile/desktop/web/headless/server), `fingerprint`, `port`, `protocol`
  (http/https), `download`, `announce`. Peers answer an `announce: true`
  message by POSTing their info to `/api/localsend/v2/register` on the
  announcer (or by a UDP reply with `announce: false`).
- Fingerprint: SHA-256 of the TLS certificate in HTTPS mode; a random string
  in HTTP mode. Used to recognise devices and ignore ourselves.
- Send:
  1. `POST /api/localsend/v2/prepare-upload` with `info` (our device fields)
     and `files` (map id -> `{id, fileName, size, fileType, sha256?,
     preview?, metadata?}`). Optional `?pin=`. Response: `sessionId` and
     `files` (id -> token).
     Codes: 204 nothing to send, 401 PIN required/invalid, 403 rejected,
     409 busy with another session, 429 too many requests.
  2. `POST /api/localsend/v2/upload?sessionId=..&fileId=..&token=..` with the
     raw bytes, one request per file (parallel allowed).
     Codes: 403 bad token/IP, 409 busy, 422 checksum mismatch.
  3. `POST /api/localsend/v2/cancel?sessionId=..` to abort.

Design:

- Discovery runs only while the module is enabled and (a) the notch is open
  on the device picker, (b) a drag is in progress, or (c) settings is open.
  Otherwise a slow refresh (every 60 s) keeps pinned devices' addresses
  fresh. No constant multicast chatter.
- **Pinned device** = alias + fingerprint + last known address. Dropping on
  a pinned device sends without further prompts on the Mac. The phone still
  shows LocalSend's own accept prompt unless the user enabled quick-save for
  favourites there; that is the phone's decision and stays there.
- HTTPS peers use self-signed certificates. Accept a peer certificate only
  if its SHA-256 equals the pinned fingerprint (trust on first use when
  pinning, shown to the user). Never disable TLS validation globally.
- Streams files from disk (no loading into memory), reports byte progress,
  supports cancel.
- Needs the macOS Local Network permission; request it only when the
  provider is first enabled, with an explanation.

Out of scope for Phase 3: receiving files on the Mac (the phone can already
send to the LocalSend or Blip app on the Mac). "Receive into the notch
shelf" is a later feature and means running the receiver side of the
protocol (`register`, `prepare-upload`, `upload` endpoints) with an accept
card in the notch (timeSensitive band).

### 3.3 Later provider ideas (parking lot)

- KDE Connect protocol (adds clipboard sync and phone notifications; bigger
  job).
- `adb push` over USB/Wi-Fi debugging for power users.
- Quick Share (Google's protocol; no official Mac API).

## 4. UI

### 4.1 Drag over the notch

Upstream already opens the notch when a drag enters the notch region
(`AppDelegate.setupDragDetectors`, `handleDragEntersNotchRegion`) and shows
the shelf drop zone. We add device targets beside it:

```
+----------------------------------------------------------------+
|  [  Shelf  ]   |  [phone] Pixel      [laptop] Windows   [Blip] |
|  drop to keep  |  LocalSend          LocalSend          share  |
+----------------------------------------------------------------+
```

- Pinned devices first, then other discovered devices, then share-menu
  services.
- Offline pinned devices are shown dimmed; dropping on one gives a clear
  "Pixel is not reachable" result, never a silent failure.
- Dropping on a target does not add the file to the shelf.

### 4.2 Progress and result

- status-band presentation: device name, file name (or "3 files"), progress
  bar, cancel button.
- Success: sneak peek "Sent to Pixel". Failure: message + Retry.
- Several transfers queue per device.

### 4.3 Shelf integration

Each shelf item's context menu gets "Send to > <device>".

### 4.4 Settings

- Providers list with switches (Share menu, LocalSend).
- Share menu: tick which services appear as targets.
- LocalSend: device name shown to others, discovered devices with "Pin",
  pinned devices with rename / unpin / fingerprint, PIN field.
- Show device targets when dragging (on/off).

## 5. Logic placement

`NotchCore/Transfer`: `TransferTarget`, `TransferProgress`, LocalSend message
types + codec, upload client built on `URLSession` (injectable for tests),
fingerprint pinning check, discovery message parsing. Tests use a local stub
server for prepare/upload/cancel and cover each status code.

`NotchModules/SendToDevice`: multicast socket (Network.framework), provider
objects, views, drop handling.

## 6. Acceptance

See ROADMAP.md Phase 3.
