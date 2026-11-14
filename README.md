# Mario Kart World on an 8 GB Mac

Mario Kart World, the Switch 2 game, running natively on a MacBook Neo: Apple A18 Pro, 8 GB of unified memory, macOS 27.
There is no emulator here. The game's own ARM64 code runs, Horizon's services are reimplemented under it and NVN2 is
translated to Metal, which is what [nxnative](https://github.com/seedds/nxnative) does.

This build is Mario Kart World **1.8.0** (v917504), packed from a 1.8.0 NUP with nxnative main `4b10c73f` and the changes
below. The 1.0.0 dump is still what the profile's shader tables were made from, so 1.8.0 is packed from a profile derived
from it, with its own shaders.

Nothing from the game is in here. No ROM, no keys, no built app. You need your own copy of the game to build it.

## What it does here

| Metric | This machine |
|---|---|
| Race, mean frame time | 21.4 ms (46.7 fps) |
| Race, 95th percentile | 38.6 ms |
| GPU, mean of the per-window medians | 13.0 ms |
| Dropped frames, per 300 | 101 mean, 58 best, 138 worst |
| Menus | 60 fps, one window with 1 dropped of 300 |
| Footprint | 4.3 to 4.5 GB while racing |
| Render | 1920x1080, upscaled to the window by MetalFX |

One playthrough of Mario Bros. Circuit, first race. On the 1.0.0 dump the same course ran at 32.5 fps at 1080p, or 47.5 fps
once the 3D was dropped to 1440x810. 1.8.0 gets that frame rate at full size, which is why the render scale control this
repository added for 1.0.0 is not needed any more, and not offered on 1.8.0 at all.
[PERFORMANCE.md](PERFORMANCE.md) has the runs, the stutters and what is still unmeasured.

## What this repository adds to nxnative

**A keyboard map, `DM_KBD_MAP`, in `dm/host/pad_kbd.h`.** The Mac layout is J K U I for A B X Y and nothing for the camera,
which is unusable without a controller. The map is data, 25 actions, and it is the change we sent upstream as
[PR #1535](https://github.com/seedds/nxnative/pull/1535). See [keyboard-layout.md](keyboard-layout.md).

**1.8.0's build is registered** in `dm/games/0400C3F00006E000/game.tsv`, so the update packs with this profile's settings and
every fix for its builds instead of as an unknown version. Upstream's row, from [PR #2621](https://github.com/seedds/nxnative/pull/2621).

**Packaged textures upload on the CPU when the guest memory is in files.** A no-copy Metal buffer over file pages is read
into memory and wired when it enters the residency set. `DM_GUEST_FILE_CPU_UP=0` turns that gate off, to compare with
upstream's buffer path.

**A render scale stub for the 1.0.0 build** in `patches.py`, which multiplies the six 3D entries of the game's resolution
table and leaves the HUD alone. It is registered against 1.0.0's `main`, so 1.8.0 does not get it.

**A profile fitted to this machine**: `mem_map_limit on`, handheld mode, and `fps_target 60`.

**A squircle icon**, through `MAC_ICON` in `dm/mac/pack_app.py`, instead of the game's own icon.

**One-command build and play**: `scripts/build.sh` bakes the settings, the icon and the pipeline archive and drops the bundle
seal; `scripts/play.sh` launches the app. `DM_FPS_HOLD=off` is part of the build because it is what made the race feel steady.

`tex_soft` is off in this profile. Our 1.0.0 build found one level down tiles Mario Kart World's terrain UVs wrongly, since its
terrain shaders read a texture's size to lay out those UVs; plumber-racer-2 runs it on and reports no broken textures.
Untested here, and not needed: the footprint is already 4.4 GB.

## How to play

Build the app, then double-click it or run `scripts/play.sh`. Hold **Option** at launch for the launcher: frame-rate cap,
hold, console mode, upscaling and language. There is no Render scale row on 1.8.0. [docs/playing.md](docs/playing.md) has
every switch and the controller notes.

| Key | Pad |
|---|---|
| C X V Z | A B X Y |
| Q O | ZL ZR |
| E U | L R |
| F H | click left / right stick |
| WASD | left stick |
| I J K L | right stick (camera) |
| arrows | D-pad |
| Shift with an arrow | the same direction on the right stick |
| = - | + - |
| Space | shake |

That is what `assets/env.txt` bakes in. Without it the default is Nintendo's layout, which is J K U I for A B X Y and no
camera at all; [keyboard-layout.md](keyboard-layout.md) has the map format, so a layout of your own is one line.

## Building

You need the game's **1.8.0** NUP (a `.nup` holding the base game and its update, or a folder dump with `base/` and
`updates/`), Xcode 27.2, Homebrew, an arm64 Python 3.12 and the toolchain nxnative lists. Then:

```sh
DUMP="/path/to/Mario Kart World + Update 1.8.0.nup" scripts/build.sh
```

`scripts/build.sh` packs v917504 by default; `PACKAGE=v0` packs another version if your dump has one.

The first build is the long one, about 45 minutes: the shader split walks the whole RomFS and all 11,603 of 1.8.0's
shaders are translated and compiled, then the Mac host, the pipeline archive (about 9 minutes) and the app. Later builds
of the same version reuse the caches and take about five minutes.

The pipeline archive is made for this Mac's GPU and carried in the app, so its first start opens it instead of compiling
its way through a course: 776.8 MB, 9,591 render pipelines and 728 compute ones when this was written. `GPU=none` skips it.
`--no-prebake` drops the pipelines the archive leaves to the device.

`DUMP` has no default on purpose: it is your path, never the repository one. `OUT` and `WORK` default to `~/Documents`, and
all of them can come from `scripts/local.conf`, which is not tracked. `DM_GUEST_FILE` needs about 10 GB free on the temp volume
(a 2 GB floor plus 8 GB, since a Mac can swap); without it the game falls back to anonymous memory and the footprint
climbs to 8 GB. A build started without a terminal writes its console to `$WORK/build.log`.

## Running

```sh
scripts/play.sh
```

Double-click the app, or run the script if you want a terminal. The host keys its save, its pipeline cache and its temp
files by title ID, in `~/Library/Application Support/nxnative/0400C3F00006E000/`; the log is at
`~/Library/Logs/nxnative/0400C3F00006E000.log`. The save is the same one the 1.0.0 build used, because it is keyed by title ID
rather than by version.

## Measurements

[PERFORMANCE.md](PERFORMANCE.md) has the 1.8.0 numbers, the comparison against the 1.0.0 dump, the stutters and what is
still unmeasured.

## Where this came from

Four builds, two weeks.

**The first, early November 2026.** A fork of [plumber-racer-2](https://github.com/cheemu21/plumber-racer-2), itself a port
of nxnative to this game. It carried the memory work that first got Mario Kart World running on an 8 GB laptop: the guest
memory in a file mapping, texture levels dropped to save space, tiles released once uploaded, and a cap on how much memory
the game could map. 40 to 57 fps, 4.48 GB footprint, at 1.0.0.

**The second, a week later.** 3.7 to 4.0 GB, 0 to 10 dropped frames of 300 on a warm course, a bundled pipeline archive,
and the keyboard map, the texture gate and the profile work. It reported 60 fps in races, which was true in a narrow sense:
when frames ran long, the game stretched its own clock, so the counter held while the world slowed.

**The third.** nxnative main `c4ffb9df`, which carries `DM_GUEST_FILE` and the host frame step for this game. 1.000 real speed,
about 52 fps in a race at 1080p, 4.4 GB. That is the build the render scale and the 1.0.0 measurements belong to.

**This one.** The refresh onto nxnative main `4b10c73f`, 245 commits further on, plus the render scale stub, the pipeline
archive in the build and 1.8.0's build registration. 1.8.0 at full 1920x1080 runs the same course as the third build did at
1440x810: 46.7 fps against 47.5, at a sharper size. [REFRESH.md](REFRESH.md) has what the refresh took from upstream, what
it deliberately did not take, and why.

## Credits

The engine is [nxnative](https://github.com/seedds/nxnative) by seedds. This port started from
[plumber-racer-2](https://github.com/cheemu21/plumber-racer-2), a Mario Kart World port of nxnative whose bring-up and
memory work this build builds on; the render scale stub is its find, re-ported here. The keyboard map, the texture upload
gate, the profile changes and the measurements are ours. Ryujinx, LibHac, SPIRV-Cross and MacNativeGames are in that stack
too.

No game files, no keys, no built application. The project is not affiliated with Nintendo.
