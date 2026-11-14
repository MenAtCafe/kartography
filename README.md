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
30 fps into 55.

## Performance

One course, Mario Bros. Circuit, two runs on the same build:

| | Second run | First run |
|---|---|---|
| Race, mean frame time | **18.1 ms** | 21.4 ms |
| That is | **55.2 fps** | 46.7 fps |
| 95th percentile | **30.8 ms** | 38.6 ms |
| GPU p50 | **11.4 ms** | 13.0 ms |
| Dropped frames per 300 | **50** | 101 |
| Footprint | 3.8 to 4.7 GB | 4.3 to 4.5 GB |

At full 1920x1080 with MetalFX scaling to the window. The difference between the two runs is the pipeline cache: the
first had none, the second had 947 pipelines in it. [PERFORMANCE.md](PERFORMANCE.md) has the windows, the stutters, the
comparison against the 1.0.0 dump, and an honest list of what is still unmeasured.

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
- **A CPU upload gate**, with `DM_GUEST_FILE_CPU_UP=0` to compare against the GPU path.
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
- **One shader does not translate.** 11,602 of 11,603 compile; the missing one uses an Ampere instruction that is not
  implemented yet.
- **Two diagnostics repeat once per run**, a slow-path texture upload and one texture type mismatch. Neither has cost a
  frame that we can see.
- **Some frames stall for no accounted reason.** 300 to 400 ms with no pipeline, no upload and no GPU time inside them.
  Open.
- **No online, no local wireless, no GameChat.** Single player and same-machine multiplayer only.
- **It gets smoother as you play.** Pipelines compile on first sight and are kept.

## Credits and license

The engine is [nxnative](https://github.com/seedds/nxnative) by seedds, MIT licensed, and this repository carries that
license. The port started from [plumber-racer-2](https://github.com/cheemu21/plumber-racer-2), a Mario Kart World bring-up
whose memory work this builds on. Shader translation draws on Ryujinx and SPIRV-Cross; packing uses LibHac. The keyboard
map, the profile, the upload gate and the measurements are ours.

Mario Kart World is a trademark of Nintendo. This project ships no game code, no keys and no assets from the game, and is
not affiliated with or endorsed by Nintendo.
