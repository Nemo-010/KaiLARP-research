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
