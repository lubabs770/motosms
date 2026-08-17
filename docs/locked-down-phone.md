# The locked-down phone stack

Six repos aim at the same problem from different angles: **make a phone do useful
work for a Linux box, headlessly, when the phone's OS is built to prevent exactly
that.** This is what each one is, what approach it takes, and where that approach
runs out of road.

The hardware throughout is a **Motorola Moto G7 Power** (`ocean`, Android 9 / API
28), SIM on RedPocket (an AT&T MVNO), living in a drawer on a USB cable, reached
over adb and Tailscale.

---

## The wall everything runs into

Android does not gate capabilities on *what* you ask. It gates them on **which
uid is asking**, and the three uids available on a non-rooted device each hold a
different, non-overlapping slice:

| uid | is | can | cannot |
|---|---|---|---|
| `shell` (2000) — adb | below the UI | `am`, `pm`, `settings`, `dumpsys`, `input`, `content` provider access | hold `READ_SMS`/`SEND_SMS`; `com.android.shell` does not even *declare* them, so `pm grant` refuses |
| app uid — Termux (10168) | a normal app | hold runtime permissions, run arbitrary Linux userland, sshd | `am`/`pm`/`settings`/`content`; the `content` CLI needs signature-level `ACCESS_CONTENT_PROVIDERS_EXTERNALLY` |
| your own app | a normal app you wrote | anything its manifest declares and you grant | nothing else |

The consequence that shapes every design here: **`shell` has provider access but
no SMS permission; Termux has the SMS permission but no provider access.** Neither
can read the MMS database alone. Verified, not assumed:

    Termux:  content query --uri content://mms
             -> SecurityException at getContentProviderExternal
    adb:     content query --uri content://mms
             -> requires android.permission.READ_SMS
    adb:     pm grant com.android.shell android.permission.READ_SMS
             -> Package com.android.shell has not requested permission

So anything touching MMS requires **an app you wrote**. Anything touching device
policy requires **adb or Device Owner**. Text messages are the lucky exception:
Termux:API declares the SMS permissions, so plain ssh is enough.

---

## motosms — the current text bridge

**Approach: use ssh as both transport and authentication; keep all logic on
Linux; put nothing on the phone.**

`termux-sms-send` out, `termux-sms-list` polled in, over ssh into Termux. No
gateway app, no HTTP server, no token — the ssh key is the credential. Transport
re-resolves per call: **USB (`adb forward`) first**, then tailnet, then LAN.

Why USB first is the load-bearing decision: see burundi below.

Limits it accepts rather than fights:

- **Inbound is poll-only.** Termux:API does not declare `RECEIVE_SMS`, so no
  broadcast exists to hook. Adaptive poll (5s hot, decaying to 60s idle).
- **GSM-7 or nothing.** This carrier cannot send UCS-2 at all; a single emoji
  fails the whole message silently. Everything outbound is transliterated to
  ASCII and capped at 3800 chars.

---

## motomms — pictures, because nothing else can

**Approach: write the smallest possible app that does only the two things an app
is strictly required for, and expose it on loopback.**

`SmsManager.sendMultimediaMessage()` out, `content://mms` read in. A foreground
service serves `127.0.0.1:8099` — loopback *is* the security model, so there is no
token. Linux reaches it either by `adb forward` or by curl inside the existing
ssh session.

Two things worth knowing:

- **The AOSP PDU encoder is vendored** (13 files, Apache-2.0, via
  klinker41/android-smsmms). `sendMultimediaMessage()` wants a serialised
  M-Send.req, and the framework's own `com.google.android.mms.pdu` is hidden API.
  Hand-rolling WSP binary encoding blind, against a 4-minute CI cycle, is not a
  good trade.
- **It is deliberately not part of moto-guard.** That app is Device Owner; a bad
  update to the holder of lock-task can strand the screen.

### Why not Tasker or MacroDroid

Both were seriously considered. Tasker's local HTTP server looks like it solves
ingress — but ingress was never the problem here, and routing through Tasker over
Tailscale would *reintroduce* the tunnel dependency that USB-first exists to
remove. More decisively, Tasker's "Compose MMS" is an intent that opens a
composer and waits for a human tap. This phone lives in a drawer. Plugins read
notifications; they do not build PDUs. The choice was never Tasker vs MacroDroid
— it was *which app do I write*.

---

## moto-guard — Device Owner kiosk

**Approach: make the physical screen useless to anyone holding the phone, without
ever touching the debugging channel.**

Pins the foreground to a whitelist (`Policy.kt` → `lockTaskPackages`), becomes
HOME, disables the status bar, hides Settings, and sets `DISALLOW_FACTORY_RESET`
and friends. The point is that nobody at the glass can revoke USB debugging and
cut the adb pipeline. It never sets `DISALLOW_DEBUGGING_FEATURES` — that would
saw off the branch everything else sits on.

Two escapes, both un-provisioning cleanly: a PIN in the app, and an adb broadcast
carrying a shared secret. A Device Owner cannot be removed by
`dpm remove-active-admin` or `pm uninstall`, so losing both means a recovery wipe
— which also erases the trusted adb keys.

**Its lock-task has real side effects on everything else here.** Launching an
activity for a non-whitelisted package fails with `error code 101`, which is why
motomms is started by an exported broadcast rather than by opening its screen:

    am broadcast -a com.lubabs770.motomms.START \
      -n com.lubabs770.motomms/.BootReceiver -f 0x20

(`-f 0x20` is `FLAG_INCLUDE_STOPPED_PACKAGES`; Android drops broadcasts to an app
that has never been launched, which is the same trap that once kept Termux:Boot
from autostarting.)

---

## burundi — the retired predecessor, and the lesson

**Approach: a third-party HTTP gateway app on the phone (capcom sms-gate),
adaptive-polled over Tailscale, with an adb watchdog to revive the tunnel.**

It worked, and it was more featureful than what replaced it: a full `.` command
router with sticky shell modes (`!cmd`, `.sh`, `.aish`) driving a live per-sender
tmux session, per-thread models, `.ccstatus`. The `.` prefix convention and the
threaded-convo model in motosms are both inherited from it directly.

**It was retired because the watchdog killed it.** A battery warning was
misdiagnosed as a dead pipeline, so the watchdog relaunched Tailscale on the phone
every two minutes, bouncing the tunnel and blackholing inbound texts — producing
the maddening symptom of "it only replies after a second text." The relay also got
OOM-killed once.

Two design rules in the current stack come straight from that post-mortem:

1. **Prefer a transport that cannot be blackholed.** The cable is not a tunnel.
2. **A watchdog that can restart the transport is a liability** unless its health
   check is exactly the thing it fixes. There is no watchdog now.

Also inherited: the gateway app delivered inbound MMS as `contentPreview` with
`attachments:[]`, which led to a long-held and **wrong** belief that this phone
could not download MMS media at all. Logcat later showed
`DownloadRequest HTTP 200, response size=78341` — the phone was fine; the gateway
just never fetched the parts. Worth remembering how long a wrong inference
survived because nobody checked the layer below.

---

## mindgatephoneway — the road not taken

**Approach: skip the SIM entirely; drive Google Voice through a persistent stealth
Chrome session, sniffing Google's own JSON responses rather than scraping DOM.**

Event-driven off DOM mutations, sinks to jsonl/sqlite/webhook, with a heartbeat
and a re-auth alert. Runs on a computer, not the phone.

Never deployed, for two structural reasons: it assumes a Google account with **no
2FA** (every re-login otherwise throws an interactive challenge that cannot be
automated), and it is capture-only — the sender was never built. Google Voice is
SIM-free, which is its appeal, but no official API exists for personal Voice, so
every approach rides a logged-in web session and inherits that fragility.

Kept as a working reference for the browser-session approach, should a SIM-free
channel ever be needed again.

---

## adb-keys — the boring one that unblocks everything

**Approach: pre-seed the device's adb trust so a headless machine never needs the
"Allow USB debugging?" dialog.**

A headless box has no display, so it can never answer that prompt. Restoring
`adbkey`/`adbkey.pub` into `~/.android/` means any device that already trusted the
key authorises silently. Private repo, necessarily — the key authenticates to
every device that trusts it.

---

## What actually works today

    text out    motosms send        termux-sms-send over ssh
    text in     motosms watch       termux-sms-list, adaptive poll
    pic out     motosms mms         -> motomms -> sendMultimediaMessage()
    pic in      motosms watch       -> motomms -> content://mms -> base64 over ssh
    screen      moto-guard          Device Owner lock-task
    adb trust   adb-keys            pre-seeded keypair

Both directions of both media types are proven end to end. The assistant reads
inbound pictures directly and can send pictures it fetches or generates.

## Standing hazards

- **Play Store auto-update force-stops the default SMS app**, silently dropping
  text and pictures for minutes. Blocked via
  `cmd netpolicy add restrict-background-blacklist <play-uid>`.
- **Battery.** The relay must stay on the cable. There is no second channel.
- **The allow-list is the entire security boundary.** The handler runs Claude with
  `--dangerously-skip-permissions`, so an allow-listed text has full control of
  the box, and SMS sender IDs are spoofable.
- **`adb tcpip` does not survive a reboot** on Android 9. If the phone reboots,
  drops off the tailnet, and has no cable, nothing can reach it.
