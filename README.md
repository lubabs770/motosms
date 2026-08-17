# motosms

Claude over SMS and MMS, through a headless Moto G7 relay, from a Linux box.

Text the phone; the Linux box pulls the message, hands it to Claude, and texts the
reply back. Pictures work in both directions. Nothing touches a cloud relay.

See [docs/locked-down-phone.md](docs/locked-down-phone.md) for how this fits with
the other repos aimed at the same phone, and why each one exists.

## Shape

    ~/motosms/motosms        the bridge (bash)   -> symlinked to ~/.local/bin/motosms
    ~/motosms/on-received    inbound handler: . commands, convos, Claude
    ~/motosms/env            device-specific config   (gitignored)
    ~/motosms/allow          allow-listed numbers     (gitignored)

Config deliberately does **not** live in `~/.config`. The systemd unit is the one
exception, because `systemd --user` only reads units from
`~/.config/systemd/user/`.

    state  ~/.local/state/motosms/    last-id, last-mms-id, threads/, ssh control sockets
    data   ~/.local/share/motosms/    received.log, inbox.jsonl, media/, *.log

## Transport

ssh into Termux on the phone, re-resolved on every call:

1. **USB** — `adb forward tcp:18022 tcp:8022`, then ssh to localhost
2. **tailnet**
3. **LAN**

USB first is the whole point. The previous incarnation of this pipeline died
because a watchdog kept bouncing Tailscale on the phone, which blackholed inbound
texts. A cable cannot be blackholed. ssh multiplexes over a ControlMaster socket,
so polling costs no handshakes.

There is **no daemon on the phone** for text — `termux-sms-send` out,
`termux-sms-list` polled in. Pictures need an app (see below), and that app
listens on loopback only.

## Commands

    motosms send <number> [text...]          # text; also reads stdin
    motosms mms  <number> <file> [caption]   # picture, via the motomms app
    motosms list [n]                         # recent inbox as JSON
    motosms watch                            # poll + dispatch (systemd runs this)
    motosms status                           # transport, battery, ids
    motosms gsm7 [text]                      # show what the carrier will actually get
    motosms tail [n]

## What you text the phone

Plain text goes to Claude. A leading `.` is a command — one tap on a phone
keyboard, unlike `/`. Matching is strict on the first token, so `./build.sh`,
`.bashrc` and `...` still reach Claude as ordinary text.

    .help              command list
    .status            convo, model, watcher, transport, battery
    .usage             Claude usage for the 5h and 7d blocks
    .model [alias]     per-convo model: opus | sonnet | haiku | fable
    .list              your convos (* = active)
    .new [name]        start a convo, named or auto (c1, c2, ...)
    .resume <name>     switch convo (bare .resume lists)
    .ping              liveness, no Claude call

Commands answer without spending a Claude turn, so they are instant and free.

Each convo is its own Claude session under
`~/.local/state/motosms/threads/<last10digits>/<name>.session`, so a long thread
can be parked rather than thrown away. If a session outgrows its context window
the handler blanks it, retries once fresh, and says
`(thread was full, started fresh)` — the predecessor went permanently silent in
that situation, which is the failure this exists to avoid.

## Pictures

Outbound and inbound both go through **[motomms](https://github.com/lubabs770/motomms)**,
a small app on the phone exposing a loopback HTTP API. An app is unavoidable:
every CLI path to MMS is permission-blocked in both directions, and headless MMS
send has exactly one public API. The detail is in
[docs/locked-down-phone.md](docs/locked-down-phone.md).

Outbound shrinks anything over 1 MB (the carrier's own reported ceiling).
Inbound parts are pulled to `~/.local/share/motosms/media/<mmsid>/` and the paths
handed to Claude, which reads the images directly rather than being told one
exists. Binary rides base64 over ssh, because ssh is a text channel.

## Text is GSM-7 or nothing

This carrier (RedPocket, an AT&T MVNO) **cannot send UCS-2 at all**. One emoji,
em-dash, curly quote, ellipsis or backtick fails the *entire* message silently —
the recipient gets nothing, and delivery state stops at "Sent" either way. So
`send` transliterates to ASCII first and caps at 3800 chars (the measured
ceiling is around 4076).

Check before sending anything unusual:

    motosms gsm7 'smart “quotes” and an em—dash'    # -> smart "quotes" and an em-dash

Inbound emoji is fine. Only outbound is constrained.

## Install

    cp env.example env && chmod 600 env    # fill in device details
    printf '+15551234567\n' > allow && chmod 600 allow
    ln -s "$PWD/motosms" ~/.local/bin/motosms
    systemctl --user enable --now motosms-watch.service

The phone side needs Termux with `openssh`, `termux-api` and `curl`, sshd
autostarted via Termux:Boot, and the Termux:API app granted `SEND_SMS` /
`READ_SMS` (grant over adb — `pm grant com.termux.api android.permission.SEND_SMS`).

## Gotchas worth keeping

- **Do not edit `motosms` while `watch` is running.** Bash reads scripts lazily by
  byte offset, so a live edit can corrupt the running loop. Restart the unit.
- **Inbound is poll-only and cannot be push.** Termux:API does not declare
  `RECEIVE_SMS`, so there is no broadcast to hook. The poll is adaptive: 5s after
  activity, decaying to 60s idle.
- **No desktop notifications, anywhere.** This channel exists for when you are
  away from the machine; a popup on an unattended box is noise.
- **Play Store auto-update is a hazard.** It force-stopped the default SMS app
  for minutes mid-test, which silently drops text and pictures. Blocked with
  `cmd netpolicy add restrict-background-blacklist <play-uid>`.
