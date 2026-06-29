# lumen-settings

The system Settings app for **AspisOS**, a capability-based,
no-ambient-authority x86-64 operating system built on the from-scratch
[Aegis](https://github.com/AspisOS/Aegis) kernel.

settings is a two-pane System Settings application — a left sidebar of
categories and a right content pane rendered as cards. It is the **privileged**
control surface of the Lumen desktop: it edits the hostname, restarts and powers
off the machine, toggles autologin and NTP, picks the desktop accent, and
launches the network and lock helpers. It is a standalone component of the Lumen
desktop, distributed as a [herald](https://github.com/AspisOS/AspisOS) package,
and runs as an **external client** of the
[lumen](https://github.com/AspisOS/lumen) compositor — it connects to
`/run/lumen.sock` over the Lumen window protocol rather than being an in-process
compositor built-in.

## Where settings fits

AspisOS is decomposed into independent repositories. settings is one client of
the graphical stack:

| Repo | Role |
|------|------|
| [`AspisOS/Aegis`](https://github.com/AspisOS/Aegis) | The kernel. Backs every live pane: `/proc`, `uname`, `sys_netcfg`, `sys_sethostname`, `sys_reboot`, `sys_set_autologin`, `sys_set_ntp`, and the capability model that gates them. |
| [`AspisOS/lumen`](https://github.com/AspisOS/lumen) | The compositor / display server. Owns the framebuffer; settings is one of its clients and uses `lumen_invoke` to launch the network manager and lock screen. |
| [`AspisOS/glyph`](https://github.com/AspisOS/glyph) | The GUI toolkit settings links against: the renderer and TTF fonts (`draw_*`, `font_*`), the theme/accent API (`glyph_theme_*`), and the client side of the Lumen protocol (`lumen_client.h`). |
| `AspisOS/lumen-settings` | **This repo.** The system Settings app. |

## What it does

Grounded in `src/main.c`. settings opens a **720x600** window and presents a
sidebar of twelve categories, each a card-based pane that mixes controls Aegis
can actually drive (rendered live) with greyed **"(Future Development)"** rows
for capabilities the system does not expose yet — so the UI maps the intended
surface without pretending to control things it cannot.

Every value shown comes from a **real source** — nothing is invented:
`/proc/{version,meminfo,cpuinfo,uptime,mounts}`, `uname`, `getuid`,
`clock_gettime`, `sys_netcfg` (syscall 500), and the `LUMEN_FB_W`/`LUMEN_FB_H`
environment hints.

| Category | Live today | Stubbed (Future Development) |
|----------|------------|------------------------------|
| System | edition, kernel, processor, cores, memory, uptime, arch; **hostname editor** (`sys_sethostname`) | — |
| Display | resolution / color depth (from `LUMEN_FB_*`) | scaling, brightness |
| Appearance | **accent picker** (`glyph_theme_set_accent`) | theme mode, wallpaper, font |
| Sound | — | output / input (HDA exists, no mixer interface yet) |
| Network | live status from `sys_netcfg`; **Open Network Manager** (`lumen_invoke "netman"`) | Wi-Fi, VPN, proxy |
| Date & Time | live clock + date; **NTP toggle** (`sys_set_ntp`, persisted to `/etc/aegis/ntp`) | timezone |
| Input | — | keyboard / mouse tuning |
| Users | current user / uid / hostname | user management |
| Storage | mounted filesystems from `/proc/mounts` | usage / quotas |
| Power | **Restart** + **Power Off** (`sys_reboot`); **Lock Screen** (`lumen_invoke "lock"`) | sleep, battery |
| Privacy | the capability security model (real facts); **Automatic Login** toggle (`sys_set_autologin`, persisted to `/etc/aegis/autologin`) | per-app permissions |
| About | Aegis / version / hardware summary | — |

The privileged, system-mutating operations it drives are the hostname editor,
Restart / Power Off, and the autologin and NTP toggles; the accent picker, the
network-manager launch, and the lock screen round out the interactive surface.

## Capabilities

AspisOS has **no ambient authority**: a process can mutate the system only
through capabilities declared for it at exec time. settings' policy
(`pkg/etc/aegis/caps.d/settings`) is:

```
admin POWER
```

This is a **privileged** app — the only one of the Lumen leaf apps that holds a
capability. It runs in the `admin` profile and holds `POWER`, which is what lets
it request the system-mutating operations above: `sys_reboot` (Restart / Power
Off) and the hostname / autologin / NTP editors. On a capability-based system
this is deliberate and explicit — this policy file is the *only* thing that
grants those operations, and nothing else in the bundle can widen it. Holding
`POWER` is also why the package must be `class=system`.

## Building

settings builds with a musl cross-compiler against a **pinned**
[glyph](https://github.com/AspisOS/glyph) toolkit artifact (the GUI libraries it
links), then packs a signed herald package.

```sh
make MUSL_CC=/path/to/musl-gcc HERALD_KEY=/path/to/signing.key
```

- `make` runs `tools/fetch-glyph.sh $(GLYPH_VERSION)` to download and unpack the
  pinned toolkit into `toolkit/`, compiles `src/*.c` against it, then packs.
- `MUSL_CC` is the musl cross-compiler (defaults to `musl-gcc` on `PATH`; the
  only toolchain assumption — point it at an Aegis-native `cc` to build on-device
  in the future).
- `HERALD_KEY` is the ECDSA-P256 key that signs the `.hpkg`.
- `GLYPH_VERSION` pins the toolkit release; `VERSION` is this app's own version.

Output: `lumen-settings.hpkg` (a `class=system` herald package) +
`lumen-settings.hpkg.sig`.

## Package payload

`lumen-settings.hpkg` is a **herald `class=system` package**: a manifest-first
uncompressed POSIX `ustar` archive with a detached ECDSA-P256/SHA-256 signature
(`tools/pack.sh`). Its herald id (`lumen-settings`) deliberately differs from the
bundle/exec name (`settings`), it installs across two trees, and it ships a
capability policy — each of which makes it `class=system` (first-party,
signature-trusted, installed verbatim) rather than an ordinary single-prefix
package:

```
/apps/settings/settings         the app binary
/apps/settings/app.ini          the bundle descriptor (name=Settings, exec=settings)
/etc/aegis/caps.d/settings       its capability policy (admin POWER)
```

## Repository layout

```
src/        settings source (main.c)
pkg/        install-tree skeleton shipped verbatim (apps bundle + caps.d)
tools/      fetch-glyph.sh (pinned toolkit fetch) + pack.sh (build the signed .hpkg)
Makefile    fetch toolkit -> build -> pack
VERSION         this app's version
GLYPH_VERSION   the pinned glyph toolkit version it builds against
```

## Dependencies

`depends=lumen` — settings is an external client of the compositor, so
installing it pulls [lumen](https://github.com/AspisOS/lumen). lumen also ships
the desktop fonts (Inter, JetBrains Mono), so settings inherits them
transitively; there is no separate font package.
