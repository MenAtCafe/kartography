# Performance

Mario Kart World 1.8.0 (v917504) on a MacBook Neo: Apple A18 Pro, 8 GB of unified memory, macOS 27.2. Build: this
repository's `main`, nxnative main `4b10c73f` plus the changes in the README, packed from a 1.8.0 NUP with the
pipeline archive built for this Mac (`applegpu_g17p`) and carried in the app. Settings are the four in `assets/env.txt`:
`DM_GUEST_FILE=1`, the keyboard map, `DM_PRESENT_UPSCALE=metalfx` and `DM_FPS_HOLD=off`.

One playthrough of Mario Bros. Circuit, first race, on 14 November 2026. One run, one course: everything here is that, and
nothing repeats it yet.

## What it does now

| Metric | This machine |
|---|---|
| Race, mean frame time | 21.4 ms |
| That is | 46.7 fps |
| Race, 95th percentile frame | 38.6 ms |
| GPU, mean of the per-window medians | 13.0 ms |
| Dropped frames, per 300 | 101 mean, 58 best, 138 worst |
| Menus and title | 60 fps, one window with 1 dropped frame of 300 |
| Footprint | 4.3 to 4.5 GB while racing |

The GPU is inside its 16.67 ms slot for the first time. The limiter moved to the CPU: the queue replay thread, which turns
the game's NVN calls into Metal, took 12.61 ms a frame at 1.44 GHz, against a declaration of 15.00 of 16.67 ms.

## Against the 1.0.0 dump

The same course, the same machine, three runs on the older build:

| Build | Size the 3D renders at | Race mean | GPU p50 | Dropped of 300 |
|---|---|---|---|---|
| 1.0.0, first run | 1920x1080 | 36.7 ms, 27.2 fps | 19.8 ms | 229 |
| 1.0.0, third run (warm) | 1920x1080 | 30.8 ms, 32.5 fps | 16.9 ms | 202 |
| 1.0.0, render scale 0.75 | 1440x810 | 21.1 ms, 47.5 fps | 12.0 ms | 96 |
| **1.8.0** | **1920x1080** | **21.4 ms, 46.7 fps** | **13.0 ms** | **101** |

So 1.8.0 at full size matches what the 1.0.0 dump managed only after dropping the 3D to 1440x810, and it is about 45
percent faster than 1.0.0 at the same resolution. The render scale control is no longer needed to get here, which matters
because it was never available on 1.8.0: its stub is registered against 1.0.0's `main`.

The menus improved too. On 1.0.0 they dropped 3 to 50 frames of 300; on 1.8.0 the best window dropped exactly one.

## The stutters

430 frames over the deadline during the race. Of those, 81 involved a pipeline or a shader library and 79 had more than a
megabyte of texture uploads, so compiles are not the main cause. 109 were over 50 ms.

The worst is the race start, which is honest loading:

```
frame 2762   513.0 ms | 11 new pipelines | 107.1 MB of textures, 369 ms of it
```

After that the big ones have almost nothing in NVN accounting for them:

```
frame 3002   429.7 ms | pipelines 0, textures 0
frame 3568   368.7 ms | pipelines 0, textures 0.2 MB
frame 5753   311.1 ms | pipelines 0, textures 0.7 MB
```

Two thirds of a second with no pipeline, no upload, no GPU time inside it. That is the same shape the 1.0.0 build showed,
and it is the open question: it is in the game's own threads or in the host around them.

## Two red lines, neither fatal

`swizzled texture storage is not in a memory pool; falling back to the CPU`, the same once-only diagnostic the older build
printed, where the upload takes the slow path and the picture is right.

`texture type mismatch: shader d92d9b75e2cc7dad_b (stage 1) declares slot 0 as Metal type 2, texture ID 337 is type 7`,
which fires once. A shader samples a texture as a different kind than what is bound there, and the draw still goes out, so
the thing to watch for is one odd-looking surface.

One shader of 11,603 did not translate: `Ampere instruction 481 at 0x0 (020c0481) is not implemented`. The course never asked
for its library, so it did not show up here. Whatever uses it will be missing until the translator learns the instruction.

## The configuration that ships

`DM_GUEST_FILE=1` puts the guest's 72.5 GB block in 73 files of 1 GB, so it is out of the footprint. The Mac's own 4 GB
alias region holds the game's regions, which reach +1.62 GB in the first 15 seconds of `nnMain`. `mem_map_limit on` and
`op_mode handheld` are the profile's; handheld renders 1920x1080 rather than a docked 2560x1440. MetalFX scales that to the
window at 2816x1584. `DM_FPS_HOLD=off` keeps late frames from being held an extra vsync, which on the older build was worth
about 14 fps on its own.

## Heat

This laptop has no fan and the chip governs itself. On the older build the frame rate fell from 30 to 22 fps once the chip
sat near 110 C. On 1.8.0 the replay thread measured 1.44 GHz during the race while the same thread measured 2.77 GHz in the
menus afterwards, so the clock, not the code, is what moves when it is warm. A cool machine run right after a long one has
not been compared.

## What is not measured

- A second course, and a second race on the same one.
- A cool machine against a warm one, which is the only way to separate 1.8.0's gains from the clock's.
- Whether the 300 ms stalls are the game, the host, or the guest files being paged.
- The shader that did not translate: which course or effect asks for it.
- The texture type mismatch above: what it draws.

## Words used here

**Frame time.** How long one picture took, in milliseconds. At 60 fps a frame has 16.67 ms; at 30 fps, 33.34 ms. Every
number in this file is frame time, dropped frames or memory, because those are what you feel.

**Dropped frame.** A frame that missed its slot: the picture on screen lingered while the new one was late. Three dropped
frames in a second looks like a small skip, thirty looks like a slideshow. A window is 300 frames, five seconds at 60 Hz.

**p50 and p95.** The middle frame of a window and the one that only one frame in twenty beats. p95 is where stutter lives.

**The replay thread.** nxnative runs the game's rendering API calls on a thread of its own, translating each into Metal.
When that thread takes longer than the slot, no amount of GPU headroom helps.
