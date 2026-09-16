# KaiLARP-research

What a JioPhone/KaiOS image actually is, what its apps expect from the hardware,
and what can honestly be emulated on a phone that is not a JioPhone.

This repository is the notes side of KaiLARP. It exists so that
[KaiLARP-core](../) and KaiLARP-Sandbox do not have to re-derive the platform
every time somebody touches them.

Everything here was read off a public system dump:

| | |
| --- | --- |
| host | `dumps.tadiphone.dev` (a GitLab instance on the Electrosphere) |
| project | `dumps/jio/f491h` (project id `4430`) |
| ref | `MAD-user-6.0-MRA58K-Jio-F491H-F001-04-13-310823-release-keys` |
| device | Jio F491H / JioPhone 2, Android 6.0 `MRA58K` |
| runtime | KaiOS `2.5.3.2`, Gecko `48.0a2`, build `20230831153742` |

The dump is a `system/` tree, so every path below is relative to
`https://dumps.tadiphone.dev/dumps/jio/f491h/-/raw/<ref>/`.

## What is in here

| file | what it covers |
| --- | --- |
| [`docs/b2g-runtime.md`](docs/b2g-runtime.md) | how Boot2Gecko boots, what `b2g`/`libxul`/`omni.ja` are, how an app is packaged and launched |
| [`docs/f491h-dump.md`](docs/f491h-dump.md) | the F491H image layout, config files, the 64 bundled apps, and what each one is |
| [`docs/spoofing-model.md`](docs/spoofing-model.md) | the native/spoof/deny capability model and the line we do not cross |
| [`findings/f491h.json`](findings/f491h.json) | the same facts as data, for tooling |
| [`findings/webapps.json`](findings/webapps.json) | raw `system/b2g/webapps` listing from the dump API |

## The short version

A KaiOS app is a web app. `system/b2g/webapps/<origin>/` holds an
`application.zip` (the app: `manifest.webapp`, HTML, JS, CSS) and an
`update.webapp` (the descriptor the store used to install it). The phone runs
them in Gecko, not Chromium, and hands the page a pile of `moz*` APIs whose
behaviour depends on real hardware: `mozMobileConnections` is a radio,
`mozTelephony` places calls, `mozCameras` is a camera, `mozSecureElement` is
the NFC secure element.

So "run a JioPhone app on an Android phone" is two problems, not one:

1. **Run the web app.** A modern web engine can do that; what it cannot do is
   answer `navigator.mozSettings` or `navigator.mozMobileConnections`. That is
   what KaiLARP-core's shim provides.
2. **Be honest about the hardware.** Some of that surface can be emulated
   faithfully enough for an app to boot and show its UI. Some of it cannot be
   emulated without lying about a security guarantee. The capability gate
   refuses the second kind rather than guessing.

Neucom Sphere would call that last part the difference between a simulation and
a forgery. KaiLARP only ships the first one.
