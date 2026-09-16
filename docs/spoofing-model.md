# Spoofing model

KaiLARP pretends to be a JioPhone so that JioPhone apps can run somewhere else.
That is a compatibility goal, not a licence to lie about security. The model
below is what keeps the two apart.

Every capability an app can require is classified before the app boots.

## The three modes

**`native`** — the host really has it.

- `compute`, `network` — Chromium's engine and network stack.
- `crypto` — real `crypto.subtle`. Not replaced, not wrapped, not faked.

**`spoof`** — the runtime emulates it, faithfully enough for the app surface,
and says so in the boot report.

- `telephony`, `sim`, `sms`, `cellbroadcast` — the RIL object model is built,
  a call transitions `dialing → active`, SMS calls resolve with a message
  object. Nothing leaves the process. The runtime log states "no real modem"
  and "never transmitted".
- `gps` — a fixed mock coordinate, flagged with `__kailarpMocked: true` on the
  position object so an app (or an audit) can tell it is not a real fix.
- `battery`, `sensors`, `bluetooth`, `wifi` — static profiles and synthetic
  scan results.
- `storage` — in-memory device storages. Volatile by design.
- `camera`, `microphone` — enabled by the engine's fake-device switch, which
  produces a synthetic test pattern. Disclosed in the report. Turned off with
  `--no-fake-media`, at which point they become `deny` and any app needing them
  is rejected.

**`deny`** — the runtime will not provide it, and the app does not boot.

- `secureelement` — NFC/eSE. Faking it would tell an app that a secret is
  protected by hardware that is not there.
- `drm` — EME/CDM licences (the `gmp-clearkey` / Widevine path). Fabricating a
  licence or a key is the definition of circumvention.
- `attestation` — hardware key attestation. Same reason as `secureelement`.
- `camera` / `microphone` when fake media is disabled or unavailable.

Refused API surfaces, which reject loudly instead of returning a plausible
lie: `mozPay`, `mozSecureElement`, `mozId`, `mozTCPSocket`, `mozUDPSocket`,
`mozMobileId`, `mozKeyAttestation`, and `requestMediaKeySystemAccess` when the
engine cannot honestly answer it.

## Why camera and microphone are in the "spoof" bucket at all

Because there is a real, non-deceptive way to provide them: the engine's
synthetic capture device (`--use-fake-device-for-media-stream`).
The app receives a stream, the video shows a test pattern, and the report says
the tracks are synthetic. No claim is made that a lens exists. That is
emulation.

The same reasoning does **not** apply to a secure element. A secure element is
only meaningful if the key it holds cannot be extracted. A software stand-in
can be extracted by definition, so presenting one would be a false statement
about a security property. Hence `deny`.

## The gate

`KaiLARP-core/src/capabilities.js` performs the classification.
`KaiLARP-core/src/manifest.js` maps an app's requested `permissions` onto
capabilities:

```
"nfc"      -> secureelement   deny   -> app rejected
"drm"      -> drm             deny   -> app rejected
"camera"   -> camera          spoof  -> allowed, reported
"telephony"-> telephony, sim  spoof  -> allowed, reported
"mobileid" -> sim             spoof  -> allowed, reported
```

Permissions the runtime has no model for are reported as `unmodelled` and do
not block anything — they do not correspond to a guarantee being fabricated.
Only a known `deny` capability stops a boot.

The outcome is a per-app report like this:

```
FPDD:       fake-media=REJECTED secureelement
JioMusic:   fake-media=runnable
WhatsApp:   fake-media=runnable   no-fake-media=REJECTED microphone,camera
Facebook:   fake-media=runnable   no-fake-media=runnable
```

Facebook runs even with fake media off, because it asks for comms, storage,
network and geolocation — none of which need a fabricated guarantee.

## What "pretending to be a JioPhone" means here, concretely

- User-Agent `…(Mobile; LYF/F491H/…; rv:48.0) Gecko/48.0 Firefox/48.0 KAIOS/2.5`
- a 240x320 viewport at 1x, portrait, touch
- `screen` and its `mozOrientation` overridden
- `navigator.hardwareConcurrency = 2`, `deviceMemory = 0.5` (512 MiB class)
- `navigator.platform = Linux armv7l`, `language = en-IN`
- the `moz*` object graph above
- `mozL10n`, `mozSettings`, `mozApps`, `mozAlarms`, device storages, alarms

It does **not** mean forging a serial number, a SIM ICCID that belongs to
somebody, an attestation quote, or a DRM licence. The SIM identity used is a
clearly-labelled fixture value and the mock GPS fix is flagged on the object.
