<p align="center">
  <img src="assets/kartography-icon.png" width="180" alt="Kartography">
</p>

<h1 align="center">Kartography</h1>

<p align="center"><em>Mario Kart World, running natively on an 8 GB Mac. No emulator, no JIT.</em></p>

<p align="center">
  <a href="PERFORMANCE.md">Performance</a> ·
  <a href="#build">Build</a> ·
  <a href="#how-to-play">How to play</a> ·
  <a href="#how-it-works">How it works</a> ·
  <a href="#limitations">Limitations</a>
</p>

<p align="center">
  <img alt="license MIT" src="https://img.shields.io/badge/license-MIT-blue.svg">
  <img alt="platform macOS arm64" src="https://img.shields.io/badge/platform-macOS%20arm64-lightgrey.svg">
  <img alt="game version 1.8.0" src="https://img.shields.io/badge/game-1.8.0-informational.svg">
</p>

---

## What this is

Kartography runs Mario Kart World **1.8.0** on Apple silicon as a native program. The game's own ARM64 code executes
directly on your chip. What sits underneath it is not a console and not a virtual machine: Horizon's services are
reimplemented in plain C, and the game's NVN2 rendering calls are translated into Metal before they reach the GPU.

The engine is [nxnative](https://github.com/seedds/nxnative) by seedds. This repository is the Mario Kart World port on
top of it: its profile, the fixes this game needs, a one-command build, and the measurements that turned a stuttering
30 fps into almost 60.

## Performance

One course, Mario Bros. Circuit, on a MacBook Neo (Apple A18 Pro, 8 GB of unified memory, macOS 27.2).

**The current build, 26 November: 51.6 fps mean over a race session, 71 frames over 100 ms, 4.33 GB peak footprint.**

Three runs of this build, one Grand Prix race each, the caches filling between them:

| | First run, cold | Second run | **Third run** |
|---|---:|---:|---:|
| Session mean | 42.7 fps | 49.7 fps | **51.6 fps** |
| After the menus | 41.9 fps | 50.0 fps | **51.3 fps** |
| Race windows, median | 44.1 | 51.0 | **52.6, best 57, worst 43** |
| Frames over 100 ms | 139 | 85 | **71** |
| Median long frame | 38.0 ms | 32.0 ms | **33.1 ms** |
| Median GPU time | 16.6 ms | 13.0 ms | **13.1 ms** |
| Peak footprint | 3.88 GB | 4.21 GB | **4.33 GB** |

At full 1920x1080 with MetalFX scaling to the window. The first run's higher GPU time is the cache filling: once the
pipelines are made, the same scene costs 13.0 ms against 14.3 ms on the 18 November build.

Against that build the third run is 51.6 against 52.2 on the session mean and 52.6 against 53.1 on race windows, level within
a run's noise. Memory is where the refresh shows: 4.33 GB at the peak against 4.49, and a median of 3.89 against 4.22.
[PERFORMANCE.md](PERFORMANCE.md) has all three builds' methods, the windows, the stutters, the comparison against the 1.0.0
dump, and an honest list of what is still unmeasured.
## How it works

Three layers, stacked:

1. **The game, patched.** The binary is rewritten only where it insists on Nintendo's kernel: thread-local storage, SVCs,
   timers, the memory ordering Apple's cores require.
2. **A reimplemented operating system.** Horizon's file system, memory pools, HID, audio, save transactions and the SDK
   services the game calls, answered by the host.
3. **A reimplemented graphics driver.** NVN2 objects and commands become Metal. Shaders are translated ahead of time, not
   at runtime, which is what makes this viable where a JIT would not be.

Because the same instruction set is on both ends, nothing simulates a CPU. That is the line between this and an emulator.

## What this repository adds

- **A keyboard map.** `DM_KBD_MAP` and nothing hardcoded: 25 actions, one line, and 44 checks behind it. Proposed
  upstream as [PR #1535](https://github.com/seedds/nxnative/pull/1535); upstream closed it in favour of
  [PR #3141](https://github.com/seedds/nxnative/pull/3141), a general controls file, so the map stays here until that
  lands.
- **1.8.0 as the package.** Upstream registers the build now too; this profile packs 1.8.0 unless you say otherwise, and
  the 1.0.0 dump is kept for comparison.
- **Guest memory in files.** The guest block lives in sparse, unlinked 1 GB files, out of the footprint: 3.9 to 4.3 GB
  in a race, which is what makes an 8 GB machine viable.
- **File-backed uploads on the GPU path.** The CPU deswizzle gate is kept, and the shipped settings leave it off
  (`DM_GUEST_FILE_CPU_UP=0`): worth 3 fps and nearly half the frames over 100 ms.
- **A render scale stub for the 1.0.0 build**, which scales its 3D resolution table. The 1.8.0 build does not need it.
- **Two ideas from [no-lag-kart](https://github.com/kingbumi31/no-lag-kart), measured and not kept.** A GPU clock keeper,
  which gains on a 16 GB M2 Pro and costs 5 fps here, so the profile sets `gpu_keep off`; and an upload source ring,
  which crashed twice and was removed rather than shipped switched off.
- **A pipeline archive built at build time** for this Mac's GPU, 919 MB, so a first race opens compiled pipelines
  instead of making them.
- **One-command build and play**, with the settings, the icon and the seal handling baked in.
- **The measurements.** Three builds, three runs each, each with its method, in [PERFORMANCE.md](PERFORMANCE.md).

## Requirements

- An Apple silicon Mac. Measured on a MacBook Neo: A18 Pro, 8 GB, macOS 27.2.
- Xcode 27.2, Homebrew, an arm64 Python 3.12, and the toolchain nxnative lists.
- **Your own Mario Kart World 1.8.0 NUP.** No game files are in this repository, no keys either.

## Build

```sh
git clone git@github.com:MenAtCafe/kartography.git
cd kartography

DUMP="/path/to/your/mario-kart-world.nup" scripts/build.sh
```

**Build 1.8.0.** Everything here was measured on it, it carries the update's own fixes, and it is the package the profile
registers and the build script packs. The 1.0.0 dump still builds, because the profile registers that build as well, but
it is kept for comparison rather than for playing: 27 to 32 fps at 1920x1080 on the same course against the figures
above, and a render scale control that the 1.8.0 build does not need.

About 45 minutes the first time, most of it translating 11,603 shaders and building the pipeline archive. About five
minutes after that, because the caches are reused. The app lands at `OUT`, `~/Documents/Mario Kart World.app` by default, with the
settings, the keyboard map and the icon already inside it.

`scripts/build.sh` packs 1.8.0 by default (`PACKAGE=v0` for another version). `DUMP`, `OUT`, `WORK` and the rest can
live in `scripts/local.conf`, which is not tracked. A build started without a terminal writes its console to
`$WORK/build.log`.

## How to play

```sh
scripts/play.sh
```

Or double-click the app. Hold **Option** at launch for the launcher: frame-rate cap, console mode, upscaling, language.
The save and the pipeline cache live in `~/Library/Application Support/nxnative/0400C3F00006E000/`, keyed by title ID, so
they survive rebuilds and version changes; the log is in `~/Library/Logs/nxnative/`.

### Controls

| Key | Pad | Key | Pad |
|---|---|---|---|
| C X V Z | A B X Y | WASD | left stick |
| Q O | ZL ZR | I J K L | right stick |
| E U | L R | arrows | D-pad |
| F H | click sticks | Shift + arrow | right stick |
| = - | + - | Space | shake |

**Drift is O.** O is ZR, the button you hold to hop and drift; Q beside it is ZL, and E and U are L and R.

The map is data, not code. [keyboard-layout.md](keyboard-layout.md) documents the format, and `assets/env.txt` is the
shipped layout.

## Project layout

| Path | What lives there |
|---|---|
| `dm/games/0400C3F00006E000/` | the game's profile: settings, shader tables, rewriter patches |
| `dm/gfx/nvn/` | NVN2 on Metal: textures, pipelines, present pacing, shaders |
| `dm/host/` | Horizon's services, threads, memory, input, and the Mac front end |
| `dm/tools/` | the rewriter that makes the binaries run on Apple's kernel |
| `scripts/` | `build.sh`, `play.sh`, `build_ipa_from_nsp.py` |
| `docs/` | the engine's reference notes, symptom index and subsystem guides |
| `assets/` | the project art and its `.icns`, the app icon, the shipped environment |

## Limitations

- **One course measured.** Mario Bros. Circuit, first track in the game. Heavier tracks are untested.
- **Tested on one Mac.** An Apple A18 Pro with 8 GB. Nothing from the M1 to the M6, base or Pro, has been run here, and
  those chips may show graphical issues: the flickering scenery reported on M-series Macs is still open upstream.
- **One shader does not translate.** 11,600 of 11,601 compile; the missing one uses an Ampere instruction that is not
  implemented yet.
- **Scene cuts stall.** A frame costs 350 to 444 ms when 130 to 150 MB of textures arrive at once, such as the cut into a
  race. The worst frame of a session is 704 ms at boot, when the window goes fullscreen.
- **The present interval flips.** The game switches its interval when it cannot hold 60, and frames of 400 ms and more
  turn up with no pipeline, no upload and no GPU time inside them.
- **Two diagnostics repeat once per run**, a slow-path texture upload and one texture type mismatch. Neither has cost a
  frame that we can see.
- **Single player only.** No online and no local wireless.
- **It gets smoother as you play.** 42.7 fps on the first run of a build and 51.6 on the third, and the difference is the
  pipeline cache filling, not the machine getting faster.

## History

| When | What |
|---|---|
| Early November 2026 | Bring-up on the 1.0.0 dump, starting from [plumber-racer-2](https://github.com/cheemu21/plumber-racer-2). 27 to 32 fps at 1920x1080. |
| Mid November | 1.8.0 registered in the profile, guest memory moved into sparse files so an 8 GB Mac stops swapping, and a pipeline archive built for this GPU. Three runs: 46.7, 55.2, 58.3 fps. The repository is renamed Kartography, and the keyboard map goes upstream as [PR #1535](https://github.com/seedds/nxnative/pull/1535). |
| 18 November | Refreshed onto nxnative main, 254 commits later. Two settings from another fork were measured and settled here: the GPU clock keeper costs 5 fps on this chip and ships off, the CPU upload gate costs 3 fps and ships off. Session means after the refresh: 52.2 fps, 49 frames over 100 ms, 4.49 GB peak. |
| 26 November | Refreshed again, 527 commits on. The shader chain moved, so every shader and the archive were rebuilt. The upload ring was dropped after two crashes. Upstream now registers 1.8.0 itself, and brings fixes for archive writes during play, sparse tile uploads and the replay's texture registrations. Three runs after it: 42.7, 49.7 and 51.6 fps, the last at 4.33 GB. |

Every number above has its method in [PERFORMANCE.md](PERFORMANCE.md), including the ones that do not compare with each other.

## Credits and license

The engine is [nxnative](https://github.com/seedds/nxnative) by seedds, MIT licensed, and this repository carries that
license. The port started from [plumber-racer-2](https://github.com/cheemu21/plumber-racer-2), a Mario Kart World bring-up
whose memory work this builds on. Shader translation draws on Ryujinx and SPIRV-Cross; packing uses LibHac. The keyboard
map, the profile, the upload gate and the measurements are ours.

Mario Kart World is a trademark of Nintendo. This project ships no game code, no keys and no assets from the game, and is
not affiliated with or endorsed by Nintendo.
