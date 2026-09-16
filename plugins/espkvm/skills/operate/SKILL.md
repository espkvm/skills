---
name: operate
description: Drive a machine through an ESP-KVM over its HTTP API - read the BIOS screen as text, press keys, move the pointer, swap the virtual disk, run a saved runbook, cut the power.
when_to_use: Use when a task means operating a computer that has no working operating system, or one reachable only through its KVM - a BIOS or UEFI setup screen, a boot menu, an installer, a machine that will not boot, or anything behind an ESP-KVM or espkvm.local.
---

# Operating a machine through ESP-KVM

An [ESP-KVM](https://espkvm.io) sits between a computer and its monitor,
keyboard and mouse. Everything its web console can do is a plain HTTP call, so a
script or an agent can do the same: watch a machine boot, read its firmware
menu, press keys in it, hand it a disk image, and power-cycle it.

The part worth knowing about: when the target is in a **text mode** - a BIOS
setup screen, a boot menu, memtest, a Linux console - the device hands back the
screen as **exact characters**, not a picture. Read the menu, decide, press the
key.

## Before anything else

The device answers on HTTPS with a certificate it made itself. Fetch it once and
trust it, rather than turning verification off:

```sh
curl -sk https://espkvm.local/cert.pem -o espkvm-ca.pem
curl --cacert espkvm-ca.pem https://espkvm.local/api/v1/auth/session
```

`/cert.pem` and `/api/v1/auth/session` are the only two things that answer
before you sign in. Everything else, `GET /api/capabilities` included, needs a
session - and capabilities is the call to make first once you have one: it says
what this device can do and why it cannot do the rest. A board with no ATX
wiring has no power button; a board with no card has no virtual media.

## Sign in

One POST, then keep the cookie. Sessions last 12 hours.

```sh
curl -sk -c jar.txt -X POST https://espkvm.local/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"user":"admin","password":"..."}'
# -> {"mustChange":false}   and a kvm_session cookie
```

- There is **no Basic auth**. A 401 is a JSON body, not a browser challenge.
- `mustChange: true` means the device still has its default password. Until it
  is changed, every path outside `/api/v1/auth/` answers 401.
- Sessions are re-used, not stacked: **only four exist at a time**, and a fifth
  login silently evicts the oldest. Sign in once and hold the cookie. Do not log
  in per request - it will throw a person out of their console.
- After three wrong passwords the device starts sleeping before it answers, up
  to 15 seconds. Do not retry a bad password in a loop.
- `GET /api/v1/auth/session` tells you whether you are signed in.

## Two rules that will otherwise return 403

**1. A write must say it is meant.** Every POST, PUT and DELETE has to carry
either `Content-Type: application/json` or the header `X-ESP-KVM: 1`. JSON calls
get this for free; the **binary uploads** (`/storage/upload`, `/storage/rescue`,
`/system/update`, `PUT /tls/cert`) send octet-stream, so those must set
`X-ESP-KVM: 1` by hand. Otherwise:
`{"error":"a write must say it is meant: ..."}`.

**2. The agent API is off by default.** Screenshots and the keyboard and pointer
endpoints answer
`{"error":"the agent API is off (enable it in Settings > Security)"}`
until someone turns on the `agent_api` setting. It is off because it hands a
program the same control over the target that the console has. It can be turned
on over the API itself - `PUT /api/v1/settings` with `{"agent_api":true}` - but
that is the operator's decision to make, so ask before flipping it.

Gated by it: `video/frame.jpg`, `hid/move`, `hid/click`, `hid/key`, `hid/type`.
Not gated: reading the screen as text, runbooks, power, storage, settings.

## Read the screen

```sh
curl -sk -b jar.txt https://espkvm.local/api/v1/screen/text
```

200 gives `{"cols","rows","text","confidence","ageMs", ...}` - `text` is the
screen, rows separated by newlines. `highlight` marks the selected row as
`[row, firstCol, length]` triples, which is how you tell where the cursor is in
a menu. `alert` appears when the screen watch matched a phrase.

**204 with an empty body means there is no text reading, not that something
failed.** It is the normal answer on a desktop, a graphical boot splash or a
lost signal.

Asking is what starts the reading. A narrow mode is read continuously anyway,
but a 1080p console is only read while somebody wants it, so the request wakes
the scan and then waits about 1.2 s for the first result - measured at 1.5 s
end to end. Nothing else has to be watching; a viewer is not needed. A second
204 straight after the first means the screen really is a picture, so poll on a
timer rather than in a tight loop.

For screens that are pictures, take a JPEG:

```sh
curl -sk -b jar.txt https://espkvm.local/api/v1/video/frame.jpg -o screen.jpg
```

This needs the MJPEG codec. While H.264 is selected it answers **409** - an
H.264 stream cannot be snapshotted. `GET /api/v1/video/status` reports `codec`,
`signal`, `width`, `height`, `inputHz` and `textMode` (whether this mode *could*
be read as characters).

## Keyboard and pointer

These speak raw USB HID, not key names. `POST /api/v1/hid/key`:

```sh
curl -sk -b jar.txt -X POST https://espkvm.local/api/v1/hid/key \
  -H 'Content-Type: application/json' -d '{"keys":[59]}'          # F2
curl -sk -b jar.txt -X POST https://espkvm.local/api/v1/hid/key \
  -H 'Content-Type: application/json' -d '{"keys":[76],"modifier":5}'  # ctrl+alt+del
```

`modifier` is a bitmask: ctrl `1`, shift `2`, alt `4`, gui/win `8` (the
right-hand pair are `16`/`32`/`64`/`128`). `keys` holds up to six usage codes;
the device sends one press and one release.

Usage codes: `a`-`z` are `4`-`29`, `1`-`9` are `30`-`38`, `0` is `39`,
`F1`-`F12` are `58`-`69`. The ones worth having by name:

| key | code | key | code | key | code |
|---|---|---|---|---|---|
| enter | 40 | space | 44 | insert | 73 |
| esc | 41 | capslock | 57 | home | 74 |
| backspace | 42 | printscreen | 70 | pageup | 75 |
| tab | 43 | pause | 72 | delete | 76 |
| minus | 45 | end | 77 | pagedown | 78 |
| equal | 46 | right | 79 | left | 80 |
| menu | 101 | down | 81 | up | 82 |

Typing text:

```sh
curl -sk -b jar.txt -X POST https://espkvm.local/api/v1/hid/type \
  -H 'Content-Type: application/json' -d '{"text":"root"}'
```

Two traps. **It stops at 80 characters** and says nothing about the rest, so
send long text in chunks. And a KVM sends key *positions*, not characters: what
arrives depends on the layout the target has active. Set the device's
`kbd_layout` setting to match the target, or a password with punctuation in it
will land wrong.

The pointer takes absolute coordinates, **0..32767 on both axes**, not pixels
and not fractions. Scale from `width`/`height` in `/api/v1/video/status`.
Out-of-range values are clamped, not refused.

```sh
curl -sk -b jar.txt -X POST https://espkvm.local/api/v1/hid/click \
  -H 'Content-Type: application/json' -d '{"x":16384,"y":16384,"button":"left"}'
```

All four answer **409 `no USB target attached`** when the target's USB is not
connected or the machine is off. `POST /api/v1/hid/reattach` re-presents the
keyboard and mouse, as if the cable had been pulled and put back - useful when a
target enumerated the KVM before the KVM was ready.

## Prefer a runbook for anything multi-step

A runbook is a script that runs **on the device**, and it can wait for the
screen: `wait Press F2` holds until that phrase appears, `gone Loading` until it
disappears. It keeps running with no browser and no agent connected, which makes
it the right tool for "get into the BIOS and change the boot order" - a fixed
sequence of arrow keys does not survive a different firmware version, and a wait
does.

```sh
curl -sk -b jar.txt -X POST https://espkvm.local/api/v1/runbooks/run \
  -H 'Content-Type: application/json' -d '{"name":"boot-from-usb"}'   # -> 202
curl -sk -b jar.txt https://espkvm.local/api/v1/runbooks/status
curl -sk -b jar.txt -X POST https://espkvm.local/api/v1/runbooks/stop \
  -H 'Content-Type: application/json'
```

`run` answers **202**, not 200. `status` reports `state`
(`idle|running|done|failed|stopped`), which step it is on and why it ended. Only
one runs at a time; a second answers 409. Runbooks are stored in the
`runbooks_json` setting, so they can be written through `PUT /api/v1/settings`.
Hak5 DuckyScript payloads run unchanged.

## Virtual media

```sh
curl -sk -b jar.txt https://espkvm.local/api/v1/storage/images
```

Reports the card (`mounted`, `totalBytes`, `freeBytes`, bus speed), the built-in
rescue image, the `images` on the card, and `active` - which one the target sees.
Two reserved names: `@rescue` is the image in the device's own flash, `@wholesd`
hands the target the whole card, read-write.

Pick one by writing settings - `msc_enable` on and `msc_image` set - and the
target sees the new disk without replugging anything:

```sh
curl -sk -b jar.txt -X PUT https://espkvm.local/api/v1/settings \
  -H 'Content-Type: application/json' -d '{"msc_enable":true,"msc_image":"ubuntu.iso"}'
```

Uploading an image is a raw body, so it needs the header:

```sh
curl -sk -b jar.txt -X POST -H 'X-ESP-KVM: 1' \
  --data-binary @ubuntu.iso 'https://espkvm.local/api/v1/storage/upload?name=ubuntu.iso'
```

The name goes in the query string, not the body. An image that is currently
mounted cannot be replaced or deleted - eject it first (`msc_enable: false`).
Uploads run at a few MB/s; **video streaming competes with the card for the same
bus**, so pause the stream if an upload matters more than watching.

`/storage/delete` and `/storage/rescue` answer with the whole images document,
not `{"ok":true}`.

## Power

```sh
curl -sk -b jar.txt -X POST -H 'Content-Type: application/json' \
  https://espkvm.local/api/v1/power/click     # tap the power button
```

`click`, `hold` (a hard off), `reset`, and `wake` (Wake-on-LAN to the configured
MAC). No body. They need ATX wiring, so a board without it answers 409. A second
action while one is in flight answers 429.

## Settings, and the rest of the device

- `GET /api/v1/settings` returns every value as one flat object; secrets are
  omitted, never readable.
- `GET /api/v1/settings/schema` describes every setting: type, range, choices,
  help text, which capability it needs and whether it wants a reboot. Read the
  schema before writing a key rather than guessing its range. An `enum` is
  written as its **index into `choices`**, not as the word: `vid_codec` is
  `0` for mjpeg and `1` for h264, so switching the codec to take a screenshot is
  `{"vid_codec":0}`.
- `PUT /api/v1/settings` takes any subset and applies it **all or nothing**;
  a bad value returns 400 naming the key, and nothing is written. Bodies must
  stay under 4 KB.
- `GET /api/v1/system/info` - version, uptime, memory, temperature, network,
  OTA slots, ATX state.
- `GET /api/v1/system/log` - the device's own log as plain text, which is the
  first place to look when something behaved oddly.
- `POST /api/v1/system/update` - a firmware image into the spare slot
  (octet-stream, so `X-ESP-KVM: 1`).

## Traps, collected

- `screen/text` 204 = the screen is a picture. Normal, not an error.
- `frame.jpg` 409 = H.264 is running; set `vid_codec` to 0 (mjpeg) to snapshot.
- `hid/type` silently stops at 80 characters and silently drops anything the US
  layout cannot produce.
- Pointer coordinates are 0..32767, not pixels.
- `runbooks/run` returns 202.
- `hid/key` takes numbers, never key names.
- Four sessions exist at once; logging in repeatedly evicts real people.
- `/storage/delete` and `/storage/rescue` answer with the whole images document,
  not `{"ok":true}`.
- `hold_ms` appears in a comment on `/hid/key` but is not implemented.

## A worked example: get into the BIOS

```sh
D=https://espkvm.local; C="curl -sk -b jar.txt"
$C -X POST -H 'Content-Type: application/json' $D/api/v1/power/click   # power on
while :; do
  s=$($C -w '%{http_code}' -o /tmp/screen.json $D/api/v1/screen/text)
  [ "$s" = 200 ] && grep -q 'F2' /tmp/screen.json && break
  sleep 1
done
$C -X POST -H 'Content-Type: application/json' -d '{"keys":[59]}' $D/api/v1/hid/key
```

Then read `screen/text` again, look at `highlight` to see which row is selected,
and press `up`/`down`/`enter` (82/81/40) from there. If this is a thing you will
do more than once, write it as a runbook and let the device do the waiting.
