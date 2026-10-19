# The F491H image

Read from `dumps/jio/f491h`, ref
`MAD-user-6.0-MRA58K-Jio-F491H-F001-04-13-310823-release-keys`.

## Identity

| field | value | where it comes from |
| --- | --- | --- |
| device | `F491H` (JioPhone 2) | path / model |
| vendor | `LYF` | `defaults/settings.json`, UA build token |
| Android base | 6.0, `MRA58K`, SDK 23 | the image name |
| KaiOS | `2.5.3.2`, name `B2G` | `system/b2g/application.ini` |
| Gecko | `48.0a2` | `system/b2g/platform.ini` |
| build id | `20230831153742` | `platform.ini`, `application.ini` |
| app id | `{3c2e2abc-06d4-11e1-ac3b-374f68613e61}` | `application.ini` |
| crash endpoint | `crash-reports.kaiostech.com` | `crashreporter.ini` |
| screen | 240x320, 1x | JioPhone 2 hardware |
| locale / tz | `en-IN` / `Asia/Kolkata` | settings |

The runtime User-Agent is assembled from the vendor, model and a LYF build
token. KaiLARP-core uses:

```
Mozilla/5.0 (Mobile; LYF/F491H/LYF-F491H-000-04-13-310823; Android; rv:48.0) Gecko/48.0 Firefox/48.0 KAIOS/2.5
```

## Configuration files

| file | size | role |
| --- | --- | --- |
| `defaults/settings.json` | ~486 KiB | the full settings database: every `settings://` key the OS ships with |
| `defaults/deviceconfig.json` | 6.2 KiB | maps operator-tunable feature names (`kaidc.4g.hide`) onto settings keys (`network.4g.ui`), with recover/reboot flags |
| `defaults/customization.json` | 2.7 KiB | launcher and operator customisation |
| `defaults/pref/user.js` | — | Gecko prefs the operator overrides |
| `defaults/pref/dev-pref.js` | — | developer prefs (adb, remote debugging) |
| `ua-update.json` | 3.2 KiB | per-site UA overrides so old sites see a browser they accept |

`deviceconfig.json` is the clearest statement of what the carrier considers a
"feature": it lists 4G UI hiding, ADB remote mode, bluetooth, call forwarding,
data roaming and so on, each with a settings key and whether toggling it needs
a reboot. KaiLARP-core seeds `mozSettings` from the settings sample and keeps
this mapping in the research notes rather than the runtime, since the shim does
not need carrier policy.

## The 64 bundled applications

`system/b2g/webapps/` holds one directory per origin (plus a `webapps.json`
registry). They fall into three groups: 40 Gaia/platform, 19 Jio/carrier and 5
third-party.

### Gaia / platform (about 41)

The OS itself. Notable ones:

- `system.gaiamobile.org` — the UI shell
- `launcher.gaiamobile.org` — homescreen
- `keyboard.gaiamobile.org`, `keypad.gaiamobile.org` — input
- `communications.gaiamobile.org`, `callscreen.gaiamobile.org`,
  `emergency-call.gaiamobile.org`, `callrecording.gaiamobile.org` — telephony UI
- `settings.gaiamobile.org`, `ftu.gaiamobile.org`, `fota.gaiamobile.org` —
  settings, first-time use, firmware update
- `contact`, `calendar`, `clock`, `notes`, `calculator`, `unitconverter`,
  `music`, `video`, `gallery`, `camera`, `fm`, `filemanager`, `download` —
  the user-facing stock apps
- `mmitest.gaiamobile.org`, `engmode.gaiamobile.org`,
  `mdnwatpos.gaiamobile.org`, `agingtest.gaiamobile.org` — factory test and
  engineering modes; these poke at hardware directly and are the most
  hardware-hungry apps in the image

### Jio / carrier (19)

`diagnostics.jio.com` (`FPDD`), `myjio.jio.com`, `jiopay.jio.com`,
`jiomusic.jio.com`, `jiocinema.jio.com`, `jiotv.jio.com`,
`jiovideocall.jio.com`, `jiomessaging.jio.com`, `jiomessages.rjil.com`,
`jioservice.jio.com`, `jiochannels.jio.com`, `jiogames.jio.com`,
`jioxpressnews.jio.com`, `jiohelpine`… actually `jiohelpline.jio.com`,
`jioads.jio.com`, `jiods.jio.com`, `jiopages.jio.com`, `hellojio.jio.com`,
`filecleaner.jio.com`.

`diagnostics.jio.com` is the interesting one for this project: it requests
`nfc`, `camera`, `audio-capture`, `bluetooth`, `wifi-manage`, `fmradio`,
`mobileconnection`, `telephony`, `device-storage:sdcard`, `power`,
`sound-trigger` and `settings`. It is a hardware self-test, so KaiLARP's gate
rejects it — the secure element it wants cannot be faked.

### Third party (5)

| origin | app |
| --- | --- |
| `youtube.com` | YouTube (`1.92.53.2`) |
| `kaios.whatsapp.net` | WhatsApp (`2.2329.7.1`) |
| `m.facebook.com` | Facebook (`2.0.3.46`) |
| `assistant.google.com` | Google Assistant |
| `kaios-plus.kaiostech.com` | KaiOS Plus services |

These five have real `update.webapp` descriptors that KaiLARP-core reads and
gates directly. Copies live in
`KaiLARP-core/fixtures/f491h/real-manifests/`.

## What is reachable and what is not

The dump is a GitLab project, so `/-/raw/<ref>/<path>` serves individual files
and `/-/law`-style APIs list trees. Two practical notes from working with it:

- The **descriptors are small** (`update.webapp` is a few KiB) and fetch fine.
- The **packages are not small**. `application.zip` runs from ~100 KiB
  (calculator) to several MiB (settings, ~4 MiB). Through a constrained proxy
  (see below) they truncate.

### What the truncation looks like, measured

`Content-Length` is honest, the body is not:

| file | `Content-Length` | bytes actually received |
| --- | --- | --- |
| `jiopay.jio.com/update.webapp` | 8 830 | 8 830 |
| `jiopay.jio.com/application.zip` | 3 010 980 | 23 743 |
| `calculator.gaiamobile.org/application.zip` | 104 784 | 23 744 |
| `system/b2g/defaults/settings.json` | ~486 KiB | 19 651 |

So a truncated download is a **valid ZIP prefix**: it has the local file
headers, `unzip` reports `End-of-central-directory signature not found`, and a
naive `curl -o` exits 0. Check the size against `Content-Length` before
believing a package is complete. Plain HTTP is refused outright from this
environment (`connect ... Permission denied`), which is worth knowing before
debugging an app's `http://api.jio.com` call — the failure is the sandbox, not
the app.

### A note for whoever runs this next

In the environment these notes were written in, `dumps.tadiphone.dev`
responses were truncated at 16–20 KiB by the egress proxy, and `git clone
--filter=blob:none` stalled with a few KiB of pack left regardless of
`http.version`, `http.postBuffer` or `lowSpeedTime`. That is a property of the
proxy, not of the dump: the GitLab API and the tree HTML confirm the files
exist at their full sizes. Consequence:

- `KaiLARP-research` ships the real descriptors and config files, plus the
  file sizes, but not the multi-MiB `application.zip` bodies.
- `KaiLARP-core` is therefore written to fetch apps **at run time, on the
  device**, where there is no such proxy. On a phone the same URL the user
  pastes works unchanged.
- Host-side verification uses a synthetic app package plus one real descriptor
  booted on its own. The verification report says which is which rather than
  pretending otherwise.
