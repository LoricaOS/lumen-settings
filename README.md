# lumen-settings

The system Settings app for **AspisOS**, a capability-based,
no-ambient-authority operating system built on the from-scratch
[Aegis](https://github.com/AspisOS/Aegis) kernel.

settings is a two-pane System-Settings application: a left sidebar of
categories and a right content pane rendered as cards. It is a standalone
component of the Lumen desktop, distributed as a
[herald](https://github.com/AspisOS/AspisOS) package, and runs as an external
client of the [lumen](https://github.com/AspisOS/lumen) compositor (it connects
to `/run/lumen.sock` over the external window protocol rather than being an
in-process compositor built-in).

## Role in the system

- A `/apps` bundle app: launched from the desktop via its `app.ini` descriptor
  (`name=Settings`, `exec=settings`), it opens a 720x600 window over the Lumen
  external window protocol.
- Categories: System, Display, Appearance, Sound, Network, Date & Time, Input,
  Users, Storage, Power, Privacy, About. Each pane mixes controls Aegis can
  actually drive (rendered live) with greyed "(Future Development)" rows for
  capabilities the system does not expose yet, so the UI maps the intended
  surface without pretending to control things it cannot.
- Real data sources only — nothing is invented: `/proc/{version,meminfo,
  cpuinfo,uptime,mounts}`, `uname`, `getuid`, `clock_gettime`, `sys_netcfg`
  (syscall 500), and the `LUMEN_FB_W/H` environment hints.
- Privileged operations it can drive: the hostname editor (`sys_sethostname`),
  Restart / Power Off (`sys_reboot`), autologin and NTP toggles
  (`sys_set_autologin`, `sys_set_ntp`), and the accent-color picker.

## Capabilities

settings' cap policy (`pkg/etc/aegis/caps.d/settings`) is:

```
admin POWER
```

This is a **privileged** app. It runs in the `admin` profile and holds the
`POWER` capability, so it can request power/administrative operations —
restart and power-off (`sys_reboot`), and the system-mutating editors
(hostname, autologin, NTP). On a capability-based system this is deliberate and
explicit: the policy file is the only thing that grants those operations, and
nothing else in the bundle can widen it.

Because its herald package id (`lumen-settings`) intentionally differs from the
bundle/binary name (`settings`), and it installs across `/apps` and
`/etc/aegis/caps.d`, settings is a `class=system` package: first-party and
signature-trusted, installed verbatim by herald.

## Building

settings fetches a pinned [glyph](https://github.com/AspisOS/glyph) toolkit
artifact (the GUI libraries it links) and builds against it, then packs a signed
herald package.

```sh
make MUSL_CC=/path/to/musl-gcc HERALD_KEY=/path/to/signing.key
```

- `GLYPH_VERSION` pins the toolkit release fetched by `tools/fetch-glyph.sh`.
- `MUSL_CC` is the musl cross-compiler (the only toolchain assumption — point it
  at an Aegis-native `cc` to build on-device in the future).
- `HERALD_KEY` signs the `.hpkg`.

Output: `lumen-settings.hpkg` (a `class=system` herald package) +
`lumen-settings.hpkg.sig`.

## Package payload

```
/apps/settings/settings         the app binary
/apps/settings/app.ini          the bundle descriptor (launcher metadata)
/etc/aegis/caps.d/settings      its capability policy
```

## Repository layout

```
src/        settings source
pkg/        install-tree skeleton shipped verbatim (apps bundle + caps.d)
tools/      fetch-glyph.sh (toolkit fetch) + pack.sh (build the signed .hpkg)
Makefile    fetch toolkit -> build -> pack
VERSION         this component's version
GLYPH_VERSION   the pinned glyph toolkit version it builds against
```

## Dependencies

`depends=lumen` — settings is an external client of the compositor, so
installing it pulls [lumen](https://github.com/AspisOS/lumen) (which also
supplies the desktop fonts).
