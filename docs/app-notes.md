# Notes on individual apps

Things learned by running the real packages, not just reading descriptors.

## `youtube.com` — YouTube `1.92.53.2`

The package is **31 KiB**, not multi-megabyte: `index.html`, `style.css`,
`src/innopia_rcu_src.js` (19 KiB), `src/main.js` (38 KiB), a few icons and a
D-pad SVG.

It is **a shell, not the app**. `main.js` builds an `<iframe>` and points it at

```
https://m.youtube.com/?client=kaios
```

`index.html` contains the remote-control overlay (D-pad, minimise/close), and
the JavaScript wires key events, a `dongleManager` Jio accessory listener and
`mozbrowser` load events to that frame. `showMweb()` sets the iframe `src`; the
whole YouTube experience lives on the remote page.

What that means for running it off-device:

- The package itself boots fine. It calls `mozApps.getSelf()` and
  `mozSetMessageHandler('activity')`, and nothing in it needs a fabricated
  hardware guarantee.
- Whether you see anything depends entirely on whether `m.youtube.com`
  still serves a page for the `kaios` client that permits being framed. It used
  to; the header it sends decides it. This is a YouTube-side policy, not a
  hardware question.
- Nothing about this app is DRM. It never asks for `drm` or `eme`; playback
  happens inside the framed page, and that is where a CDM would be needed.

Two real shim gaps this app found, both now fixed in KaiLARP-core:

1. `screen.orientation` was frozen. The app does
   `screen.orientation.onchange = ...` and threw
   `Cannot add property onchange, object is not extensible`. It is a mutable
   object now.
2. `navigator.dongleManager` (Jio's USB-dongle API) was missing entirely. It is
   now present and reports "no dongle attached", which is a truthful state for a
   device that has the API and nothing plugged in.

## `jiopay.jio.com` — JioPay `2.4.1.0` (packaged) / `2.7.8.1` (dump descriptor)

The package that found the most runtime gaps, because it is not a shell like
YouTube and not a hardware test like FPDD: it is a full app that boots, runs
its own platform detection and then *decides* what to do.

What it is: `application.zip` is 1.7 MiB — `index.html` (41 KiB), jQuery 1.10,
RequireJS, `forge.min.js`, a `pgsdk/` payment SDK (card entry, OTP, UPI,
NetBanking), a `KaiOSSDK/` ads SDK, 24 localisation bundles and a service
worker. The manifest declares `role: system`, `type: certified`, origin
`app://jiopay.jio.com`, and two activities (`JioPay`, `jiopay_cardActivity`).
It reads IMEI, ICCID, IMSI and MSISDN from the `jio_service_store` datastore
before it will talk to its backend.

Two different descriptors exist for the same app and they do **not** agree:

| | packaged `manifest.webapp` | dump `update.webapp` |
| --- | --- | --- |
| version | `2.4.1.0` | `2.7.8.1` |
| permissions | 9 | 23 |
| asks for `nfc` | no | yes |
| asks for `audio-capture`, `video-capture` | no | yes |
| gate result | runnable | **rejected: secureelement** |

That is not a bug in the gate: the installed package on this dump is older than
the descriptor the store would install today. Gating the descriptor and booting
the package are both correct, and the difference is worth stating out loud
rather than quietly using one for the other.

Running it end to end on a host:

1. It boots, paints its splash, and calls `navigator.hasFeature(
   "device.capability.qwerty")` to pick a form factor.
2. It reads `jio_service_store` rows 5–10 and dereferences row 6 as
   `.osv`/`.platform_version`. Without the store it shows "Something went
   wrong"; without row 6 it throws on `undefined.osv`.
3. It POSTs to its backend (`rtss-prod.jioconnect.com`, `api.jio.com`). The
   ZLA handshake fails on this host, and the app then shows "Your device should
   have a Jio SIM to automatically sign you in" and, when that alert is
   dismissed, **closes itself**.

Four shim bugs it found, all fixed in KaiLARP-core:

1. `navigator.hasFeature` did not exist → `window.navigator.hasFeature is not a
   function` on the first line of `js/main.js`.
2. `navigator.getDataStores` did not exist, and there was no `jio_service_store`
   at all → the app could never reach its own device-data path.
3. `mozL10n` was defined non-writable, but the package ships mozilla's own
   `l10n.min.js` which does `navigator.mozL10n = {...}` → `Cannot assign to
   read only property 'mozL10n' of object '#<Navigator>'`. Every `moz*` surface
   is writable now; an app replacing one is the app's business.
4. The profile claimed `en-IN`. The dump says otherwise:
   `system/build.prop` has `ro.product.locale=en-US` and
   `system/b2g/defaults/settings.json` ships `"language.current":"en-US"`. The
   app carries `en-IN` translations, but the platform language it launches
   under is `en-US`, and its l10n layer reads `navigator.language` back. The
   profile now reports what the dump reports.

What it still does on a host, and why that is not a runtime failure: with no
backend it reaches its own error path, shows the Jio-SIM alert, and quits. That
path is now *reported* (`closed: true`, `window.close()`) instead of looking
like a page that hung for fifteen seconds.

The one page error that remains is the package's own, not a shim gap.
`js/lib/blockui.js` never declares `isBlockUi`: `blockUI()` assigns it
(sloppy-mode script, so that creates a global) and `unblockUI()` reads it
unguarded. `showAlertPopup()` ends with `unblockUI()`, and on the path JioPay
takes here nothing has called `blockUI()` first, so the read throws
`ReferenceError: isBlockUi is not defined`. Gecko 48 throws on that read exactly
as Chromium does; the difference is only *when* the global gets created. The
throw happens after the popup has been built, appended and set to
`display: block`, which is why the alert is on screen regardless — only the
tail of the function is skipped. It is worth knowing before "fixing" it in the
runtime: doing so would mean defining a global the app never defines.

Also worth knowing: 12 of the 18 images `index.html` references are **not in
the package** (`images/Jio_logo.png`, `images/icon_un_*.png`, `icon_sim.png`,
`jioapps_image.png` …). They are fetched from Jio's CDN at run time, which is
why the launcher tiles look empty off-device. The splash and the JioPay mark
are local and do paint.

## `diagnostics.jio.com` — `FPDD` `1.0.11.3`

Requests `nfc`, `camera`, `audio-capture`, `bluetooth`, `wifi-manage`,
`fmradio`, `mobileconnection`, `telephony`, `device-storage:sdcard`, `power`,
`sound-trigger` and `settings`. It is the hardware self-test, so it is the app
the capability gate refuses first: it asks for the secure element, and a
software stand-in for a secure element is a false statement about hardware.

## Getting a package out of the dump from a restricted network

`KaiLARP-core/.github/workflows/fetch-dump-app.yml` downloads one app from the
dump on a runner and hands back `update.webapp` and `application.zip` as an
artifact. Useful because some egress proxies truncate responses from the dump
(the packages are small — tens of KiB to a few MiB — but a proxy that stops at
16 KiB makes even a 31 KiB package unobtainable).
