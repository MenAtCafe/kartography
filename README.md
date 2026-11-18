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

**The current build, 18 November: 52.2 fps mean over a race session, 49 frames over 100 ms, 4.49 GB peak footprint.**

Three runs of this build, one race each, changing only the settings:

| | First run | Second run | **Third run** |
|---|---:|---:|---:|
| Session mean | 44.1 fps | 47.2 fps | **52.2 fps** |
| Mean after the menus | 42.4 fps | 47.1 fps | **52.6 fps** |
| Five-second windows, median | about 43 fps | about 47.5 fps | **53.1 fps, best 60** |
| Frames over 100 ms | 178 | 103 | **49** |
| Display ticks spent holding | 29% | 22% | **12%** |
| Peak footprint | 4.50 GB | 4.53 GB | **4.49 GB** |

At full 1920x1080 with MetalFX scaling to the window. One setting separates each run: the second turns the upload gate
off, the third turns the clock keeper off as well, and both are the defaults now. The keeper is off because on this chip it
spins work the frames need more than the clock it buys; the gate is off because the GPU upload path measured faster. A
fourth run with the same settings measured 45.7 fps while the machine was busy and warm, which is what a fanless laptop
does after an hour of play, and it is reported rather than hidden.

The **14 November build this replaces** measured 58.3 fps on its third run, 55.2 on its second and 46.7 on its first,
taken over 300-frame windows of a race. That method leaves menus and loading out, so it does not subtract from the table
above; on race windows alone the new build sits at a median of 53.1 against that 58.3, and it is written that way here
because the new build is not faster than the old one yet. What it is, is steadier: fewer frames over 100 ms and half the
display ticks spent holding.

The difference between those three old runs was the pipeline cache and nothing else: the first had none and compiled its
way round the course, the third had 1,283 pipelines in it and compiled nothing for the first six windows of the race.
[PERFORMANCE.md](PERFORMANCE.md) has both methods, the windows, the stutters, the comparison against the 1.0.0 dump, and
an honest list of what is still unmeasured.
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

- **A keyboard map.** `DM_KBD_MAP` and nothing hardcoded: 25 actions, one line, sent upstream as
  [PR #1535](https://github.com/seedds/nxnative/pull/1535).
- **1.8.0's build registered** in the game profile, so the update packs with this profile's settings and every fix for its
  builds rather than as an unknown version.
- **Guest memory in files.** The 72.5 GB guest block lives in 73 sparse, unlinked 1 GB files, out of the footprint, which is
  what makes an 8 GB machine viable.
- **File-backed uploads on the GPU path.** The CPU deswizzle gate is kept, but the shipped settings leave it off
  (`DM_GUEST_FILE_CPU_UP=0`): it measured 47.2 fps against 44.1, and 103 frames over 100 ms against 178.
- **A GPU clock keeper and an upload source ring**, both ported from
  [no-lag-kart](https://github.com/kingbumi31/no-lag-kart) and both off by default. The keeper gained on a 16 GB M2 Pro
  and loses on the Neo (52.6 fps without it, 47.2 with), so the game profile sets `gpu_keep off`. The ring is not
  proven yet and `DM_NVN_UP_COPY` stays at 0.
- **A pipeline archive built at build time** for this Mac's GPU: 9,591 render pipelines and 728 compute ones, so a first
  race compiles dozens rather than thousands.
- **One-command build and play**, with the settings, the icon and the seal handling baked in.

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
- **One shader does not translate.** 11,602 of 11,603 compile; the missing one uses an Ampere instruction that is not
  implemented yet.
- **Two diagnostics repeat once per run**, a slow-path texture upload and one texture type mismatch. Neither has cost a
  frame that we can see.
- **Some frames stall for no accounted reason.** 300 to 400 ms with no pipeline, no upload and no GPU time inside them.
  Open.
- **Single player only.** No online and no local wireless.
- **It gets smoother as you play.** Pipelines compile on first sight and are kept.

## History

| When | What |
|---|---|
| Early November 2026 | Bring-up on the 1.0.0 dump, starting from [plumber-racer-2](https://github.com/cheemu21/plumber-racer-2). 27 to 32 fps at 1920x1080. |
| Mid November | 1.8.0 registered in the profile, guest memory moved into sparse files so an 8 GB Mac stops swapping, and a pipeline archive built for this GPU. Three runs: 46.7, 55.2, 58.3 fps. The repository is renamed Kartography. |
| 15 November | The keyboard map sent upstream as [PR #1535](https://github.com/seedds/nxnative/pull/1535), the 1.8.0 build row as [PR #2621](https://github.com/seedds/nxnative/pull/2621). |
| 18 November | Refreshed onto nxnative main, 254 commits later. Two settings from another fork were measured and settled here: the GPU clock keeper costs 5 fps on this chip and ships off, the CPU upload gate costs 3 fps and ships off. An upload ring was ported and left off, unproven. Session means after the refresh: 52.2 fps, 49 frames over 100 ms, 4.49 GB peak. |

Every number above has its method in [PERFORMANCE.md](PERFORMANCE.md), including the ones that do not compare with each other.
## Credits and license

The engine is [nxnative](https://github.com/seedds/nxnative) by seedds, MIT licensed, and this repository carries that
license. The port started from [plumber-racer-2](https://github.com/cheemu21/plumber-racer-2), a Mario Kart World bring-up
whose memory work this builds on. Shader translation draws on Ryujinx and SPIRV-Cross; packing uses LibHac. The keyboard
map, the profile, the upload gate and the measurements are ours.

Mario Kart World is a trademark of Nintendo. This project ships no game code, no keys and no assets from the game, and is
not affiliated with or endorsed by Nintendo.
