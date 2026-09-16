# How the Boot2Gecko / KaiOS runtime works

Boot2Gecko (B2G) is Firefox OS's runtime with the UI removed. KaiOS is a
continuation of it: same Gecko, same `webapps` model, plus carrier and Jio
services. The F491H image is KaiOS 2.5.3.2 on Gecko 48.0a2.

## The shape of the runtime on disk

`system/b2g/` in the dump:

```
application.ini        Vendor=KaiOS  Name=B2G  Version=2.5.3.2  BuildID=20230831153742
platform.ini           Milestone=48.0a2  BuildID=20230831153742
b2g                    the launcher executable
libxul.so              Gecko itself (everything below is linked from it)
libmozglue.so          allocator, logging, crash plumbing
libmozsandbox.so       seccomp sandbox for content processes
libnss3.so             NSS: TLS and certificate handling
libfreebl3.so          NSS freebl (crypto primitives)
libsoftokn3.so         NSS softoken (software key store)
libnssckbi.so          built-in root certificates
icudt56l.dat           ICU 56 data (i18n, locale collation)
omni.ja                packed JS/HTML/CSS: the platform front end, defaults, prefs
plugin-container       out-of-process plugin/content helper
xpcshell               JS shell for platform scripts
crashreporter.ini      KaiOS crash endpoint (crash-reports.kaiostech.com)
dependentlibs.list     the link order Gecko uses when it starts
ua-update.json         per-site User-Agent overrides
defaults/              device prefs, settings, operator configuration
webapps/               65 installed applications, one directory per origin
```

`omni.ja` is a zip. Prefs and the front-end modules live inside it; KaiOS adds
`defaults/pref/user.js` and `defaults/pref/dev-pref.js` on the outside so the
operator can override without repacking.

## Boot order

1. `b2g` starts, reads `application.ini` and `platform.ini`.
2. Gecko reads `omni.ja` defaults, then `defaults/pref/*.js`, then
   `defaults/settings.json` and `defaults/deviceconfig.json`.
3. The **Gaia** front end (also inside `omni.ja`) launches the homescreen,
   which is its own web app: `webapps/launcher.gaiamobile.org`.
4. Everything the user sees is a web app loaded from `webapps/<origin>`:
   `system.gaiamobile.org` is the UI shell, `keyboard.gaiamobile.org` is the
   keyboard, `communications.gaiamobile.org` handles calls and SMS, and so on.
5. `koost/externalapi` (`ApiDaemonWhitelistedPort=80` in `application.ini`)
   exposes a small privileged daemon to whitelisted pages.

## How an app is packaged

Two descriptors are involved and they are not the same file:

| file | where | what it is |
| --- | --- | --- |
| `update.webapp` | `webapps/<origin>/` | the **store descriptor**: `name`, `display`, `version`, `origin`, `package_path`, `permissions`, `icons`, `developer`. This is what the marketplace had. |
| `application.zip` | `webapps/<origin>/` | the **app package**. Contains `manifest.webapp` (the packaged manifest, no `origin`) plus HTML/JS/CSS. |
| `webapps.json` | `webapps/` | registry: which origins are installed and where their packages live. |

`permissions` is the interesting field. It is an object, not a list, and each
entry is a request for a privileged API:

```json
"permissions": {
  "telephony":  { "access": "readwrite" },
  "geolocation":{ "access": "readonly"  },
  "audio-capture": { "access": "readwrite" }
}
```

The platform checks these before handing out the corresponding `moz*` object.
An app that never asked for `telephony` sees no `mozTelephony` at all, or sees
a version whose calls reject. That means an app's manifest tells you exactly
which hardware it believes it can rely on — which is what the capability gate
in KaiLARP-core keys off.

## The privileged API surface apps actually use

Grouped by what they need underneath:

**Radio / identity** — `mozMobileConnections`, `mozMobileConnection` (legacy),
`mozIccManager`, `mozTelephony`, `mozMobileMessage`, `mozCellBroadcast`,
`mozVoicemail`, `mozNetworkStats`, `mozMobileId`.

**Sensing** — `mozCameras` and `getUserMedia`, `mozFMRadio`, `geolocation`,
`mozBattery`/`getBattery`, `mozBluetooth`, `mozWifiManager`, device orientation
and motion.

**Storage** — `getDeviceStorage(name)` / `getDeviceStorages(name)` for
`apps`, `pictures`, `videos`, `music`, `sdcard`, `crashes`; `mozIndexedDB`;
`mozContacts`.

**Platform** — `mozSettings` (`createLock`, observers), `mozAlarms`,
`mozApps`, `mozPermissionSettings`, `mozInputMethod`, `mozL10n`,
`mozNotification`, message handlers (`mozSetMessageHandler`,
`mozHasPendingMessage`).

**Refused by KaiLARP** — `mozPay` (money), `mozSecureElement` / NFC, `mozId`
(dead Persona), raw `mozTCPSocket`/`mozUDPSocket`, `mozMobileId` tokens,
hardware attestation, and EME/CDM DRM licences.

See [`spoofing-model.md`](spoofing-model.md) for why each of those is in the
"refuse" bucket.

## Crypto, specifically

There are two different things people mean by "the crypto parts":

1. **Web crypto.** `crypto.subtle` in the page. This is not spoofed in any
   way: the engine has a real, audited implementation and KaiLARP leaves it
   alone. `KaiLARP-core`'s demo app proves this by hashing with it and checking
   the digest.
2. **Platform crypto and DRM.** `libnss3.so`, `libfreebl3.so`,
   `libsoftokn3.so` on the device; `gmp-clearkey`, `gmp-fake` and
   `gmp-fakeopenh264` in `system/b2g/` for Encrypted Media Extensions. A CDM,
   a hardware-backed key, or a secure element is a **promise about hardware**.
   Returning a fake one would tell an app a protection exists when it does not.
   KaiLARP refuses those and says so in the boot report.

`gmp-clearkey` is the W3C Clear Key demo CDM, `gmp-fakeopenh264` is a
software H.264 shim. Their presence tells you the phone was expected to play
DRM-gated media. That is precisely the case the gate declines.

## Why Gecko 48 matters

Gecko 48 is from 2016. Apps were written against it, so they use APIs a modern
engine has removed (`mozSettings`, the `moz*` names above, `-moz-` CSS) and
they do not use APIs a modern engine assumes (modules, modern event targets).
KaiLARP-core therefore does not try to run them in "a browser". It boots the
app under an injected compatibility layer that reinstates the B2G surface,
with the F491H User-Agent and a 240x320 viewport, and treats anything the layer
cannot honestly provide as a capability failure.
