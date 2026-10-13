# CLAUDE.md

Guidance for working in this repository.

## What this is

**SqueezePlay** (binary name `jive`) is the Lua/C user interface and audio player that runs on
the Logitech Squeezebox hardware players, built with **SqueezeOS** firmware. It talks to a
Lyrion Music Server (LMS, formerly Logitech Media Server / SqueezeCenter). It also builds as a
desktop player for Linux, macOS and Windows.

This is a fork of the community-maintained `ralph-irving/squeezeos-squeezeplay`. The default
branch is `public/9.0`, and the version is in `src/squeezeplay.version` (currently 9.0.2).
Upstream bug reports and history live on the ralph-irving repository. The OS image itself,
including wpa_supplicant, busybox and the `/etc/network` scripts, is built from the separate
`ralph-irving/squeezeos` repo, not from this one.

Hardware code names used throughout the tree:

| Code name | Product                | Screen     | Notes                                |
|-----------|------------------------|------------|--------------------------------------|
| `jive`    | Squeezebox Controller  | 240x320    | Handheld remote, wireless only        |
| `baby`    | Squeezebox Radio       | 320x240    | Atheros wifi (`eth1`), wpa_supplicant 2.9 in recent firmware |
| `fab4`    | Squeezebox Touch       | 480x272    | Marvell wifi + ethernet               |
| desktop   | SqueezePlay on a PC    | any        | `squeezeplay_desktop`                 |

## Layout

```
assets/                 Skin artwork sources (Fab4Skin, QVGASkin, Lyrion logos, visualizers)
src/
  squeezeplay/          Core, shared by every platform
    src/                C core: jive.c (main), ui/ (SDL widgets + tolua++ bindings
                        lua_jiveui.pkg), audio/ (decode/ = flac, mad, vorbis, aac, alac, pcm,
                        alsa/portaudio output, visualizers), net/
    share/jive/         Lua framework: JiveMain.lua, AppletManager.lua, System.lua,
                        ui/ (Framework, Window, Menu, Task, ...), net/ (SlimProto, Comet,
                        Socket*, NetworkThread), slim/ (Player, SlimServer, LocalPlayer),
                        audio/ (Playback.lua), utils/
    share/applets/      Platform-independent applets (NowPlaying, SlimBrowser, skins, ...)
  squeezeplay_squeezeos/  Code shared by all SqueezeOS devices (jive/baby/fab4)
    src/                wireless.c, wpa_ctrl.c (wpa_supplicant control socket), squeezeos.c
    share/jive/net/Networking.lua   All network/wifi configuration (wpa_cli, ifup/ifdown,
                                    /etc/network/interfaces editing, health checks)
    share/applets/      SetupNetworking, Diagnostics, SetupFirmwareUpgrade, SetupSSH, ...
  squeezeplay_baby/     Radio: BSP C code (baby_bsp.c, ir.c) + device applets
  squeezeplay_fab4/     Touch: BSP C code + device applets (MediaManager, WirelessBridging...)
  squeezeplay_jive/     Controller: BSP C code + device applets
  squeezeplay_desktop/  Desktop-only applets + launcher script
  squeezeplay_contrib/  Example/fun applets (Snake, Invaders, ...)
  squeezeplay_test/     Test applets (SkinTest, WireFrame, ...)
  <third-party dirs>    Vendored libraries: lua-5.1.1, SDL-1.2.13 (+ image/ttf/gfx),
                        tolua++, luasocket, flac, libmad, Tremor, portaudio, expat, zlib,
                        jpeg, libpng, freetype, axTLS, ...
  platforms/debian/     Debian packaging
  Makefile.linux / Makefile.osx* / Makefile.squeezeos / Jive.sln   Top-level builds
test/SmokeTestAutomation/   Old Perl-driven smoke test scripts (not a unit test suite)
```

Platform directories layer on top of `squeezeplay/`. At install time their `share/` trees are
merged, so a device can add or override applets and Lua modules.

## Applets

Each applet is a directory `share/applets/<Name>/` that contains:
- `<Name>Meta.lua`: always loaded. It declares `jiveVersion`, `defaultSettings`,
  `registerApplet` and `configureApplet`, and registers menu items and services with
  `jiveMain` and `appletManager`.
- `<Name>Applet.lua`: loaded lazily when the applet is first used.
- `strings.txt`: localisation. A token on its own line, then tab-indented
  `<ISO-639-1 code><TAB>text` lines. Keep `EN` and add other languages when you can.
- Optional skin images and other assets.

Applets talk to each other through `appletManager:callService(...)` and
`registerService` in the Meta file.

## Code conventions

- Lua 5.1 with the `module(...)` / `oo.class(_M, Base)` (loop.simple) pattern. Copy any
  globals and builtins a module needs into locals before `module(...)`. After that call they
  are no longer visible inside the module.
- Use **tabs** for indentation. Functions are documented with `--[[ =head2 ... =cut --]]`
  POD blocks.
- Use `jive.utils.log` loggers (`log:debug/info/warn/error`). Do not use `print`.
- Long-running or blocking work, including anything that calls `Networking:request()`
  (wpa_cli), must run inside a `jive.ui.Task`. Many functions assert `Task:running()`,
  and `t_`-prefixed functions expect to be called from a task.
- Keep changes compatible with Lua 5.1 and the old gcc 4.x ARM toolchain. Firmware users run
  a mix of old and new SqueezeOS builds, so for example wifi code must work with both
  older wpa_supplicant releases (0.6.x/0.7.x) and 2.9, which recent Radio firmware ships.

## Building and testing

- No CI or automated unit tests exist in this repo. Lua changes are usually checked by
  syntax-checking (`luac -p file.lua`, using Lua 5.1 if available) and then testing on a
  device or the desktop build.
- Desktop Linux: `cd src && make -f Makefile.linux` (see `src/README.linux` for the old
  package list). The output goes to `../../build/linux`.
- macOS: `Makefile.osx`, see `src/README.osx`. Windows: `Jive.sln`, see
  `src/README.windows.txt`.
- Device firmware: `Makefile.squeezeos` is driven by the squeezeos/poky build, which expects
  `../../squeezeos/build` and the `arm-926ejs-linux-gnueabi` toolchain. On a device you can
  also copy changed `.lua` files into `/usr/share/jive/...` (or the applet directory) and
  restart SqueezePlay.

## Wifi/network notes

`src/squeezeplay_squeezeos/share/jive/net/Networking.lua` drives `wpa_supplicant` through its
control socket (`self:request("SCAN")`, `ADD_NETWORK`, `SET_NETWORK`, ...). It brings
interfaces up and down with busybox `ifup`/`ifdown`, using `iface=<ssid>` mappings in
`/etc/network/interfaces` where spaces in the SSID become `_`. Many wpa_cli calls are wrapped
in `assert(... == "OK\n")`, so an unexpected `FAIL` from a newer wpa_supplicant crashes the
network task. The wifi setup UI is
`src/squeezeplay_squeezeos/share/applets/SetupNetworking/`. See upstream issue
ralph-irving/squeezeos-squeezeplay#5 for the wpa_supplicant 2.9 behaviour changes.
