# Performance

Mario Kart World 1.8.0 (v917504) on a MacBook Neo: Apple A18 Pro, 8 GB of unified memory, macOS 27.2. One course,
Mario Bros. Circuit, on the same machine, across three builds of this repository.

This is not a benchmark suite. It is a laptop playing a console game, measured honestly. Two methods appear here, and
every table says which one it used, because they do not subtract from each other:

- **Race windows**, 300 frames each, taken from inside a race. Used for the 14 November and 18 November builds.
- **Display-link sessions**, a five-second summary at a time, from launch to quit, menus and loading included. Used from
  26 November on, because it is what the host logs by itself.

## The current build, 26 November

**51.6 fps mean over a race session, 71 frames over 100 ms, 4.33 GB peak footprint.** Built on nxnative main of
26 November, shader chain `3ed4ea8baaaa`.

| | First run, cold | Second run | **Third run** |
|---|---:|---:|---:|
| Mean, whole session | 42.7 fps | 49.7 fps | **51.6 fps** |
| Mean, after the menus | 41.9 fps | 50.0 fps | **51.3 fps** |
| Race windows, median | 44.1 fps | 51.0 fps | **52.6 fps, best 57, worst 43** |
| Frames over 100 ms | 139 | 85 | **71** |
| Median long frame | 38.0 ms | 32.0 ms | **33.1 ms** |
| Median GPU time | 16.6 ms | 13.0 ms | **13.1 ms** |
| Peak footprint | 3.88 GB | 4.21 GB | **4.33 GB** |

*Method: display-link sessions, one Grand Prix race per run.*

The step from the first run to the second is the pipeline cache, not the build: 16.6 ms of GPU time becomes 13.0 once the
pipelines exist. That is 1.3 ms under the 14.3 ms of the 18 November build on the same scene, so this chain is not slower
per frame.

Against the 18 November build it is level on frame rate, 51.6 against 52.2 and 52.6 against 53.1 on race windows, and
lighter on memory: 4.33 GB at the peak against 4.49, with the median down from 4.22 to 3.89.

## The 18 November build

**52.2 fps mean over a race session, 49 frames over 100 ms, 4.49 GB peak footprint.** Built on nxnative main of
18 November, 254 commits on from the build before it, shader chain `7a2020f13431`.

| | First run, the old defaults | Second run | **Third run** |
|---|---:|---:|---:|
| Mean, whole session | 44.1 fps | 47.2 fps | **52.2 fps** |
| Mean, after the menus | 42.4 fps | 47.1 fps | **52.6 fps** |
| Race windows, median | about 43 fps | about 47.5 fps | **53.1 fps, best 60** |
| Frames over 100 ms | 178 | 103 | **49** |
| Display ticks spent holding | 29% | 22% | **12%** |
| Median long frame | 41.3 ms | 36.8 ms | **29.7 ms** |
| Peak footprint | 4.50 GB | 4.53 GB | **4.49 GB** |

*Method: display-link sessions, one race per run.*

Each run differs by one setting, and those settings are what the app ships now:

1. The defaults as they were.
2. `DM_GUEST_FILE_CPU_UP=0`, which puts file-backed texture uploads on the GPU path: 3 fps and nearly half the frames
   over 100 ms.
3. `gpu_keep off`, which turns off the GPU clock keeper ported from another fork: another 5 fps, and display ticks spent
   holding halved.

## The 14 November build

**58.3 fps mean in a race, at full 1920x1080, with the GPU finishing in 10.5 ms against a 16.67 ms slot.** Built on
nxnative main `4b10c73f`, shader chain `4b57309b93d2`.

| | First run | Second run | **Third run** |
|---|---:|---:|---:|
| Race, mean frame time | 21.4 ms | 18.1 ms | **17.2 ms** |
| That is | 46.7 fps | 55.2 fps | **58.3 fps** |
| 95th percentile | 38.6 ms | 30.8 ms | **30.9 ms** |
| GPU, mean of the per-window medians | 13.0 ms | 11.4 ms | **10.5 ms** |
| Dropped frames, per 300 | 101 | 50 | **29 mean, 3 best, 67 worst** |
| Footprint | 4.3 to 4.5 GB | 3.8 to 4.7 GB | **3.5 to 4.6 GB** |
| Pipelines cached at start | 0 | 947 | **1,283** |
| Queue replay thread | 12.61 ms at 1.44 GHz | 11.00 ms at 1.61 GHz | **10.11 ms at 1.81 GHz** |

*Method: 300-frame race windows.*

Same build, same settings. The only thing that changed between the three columns is what the pipeline cache had already
learned, and it is worth reading as the shape of every first run:

| Frames into the race | 2400 | 2700 | 3000 | 3300 | 3600 | 3900 | 4200 | 4500 |
|---|---|---|---|---|---|---|---|---|
| Pipeline bases compiled | 0 | 0 | 0 | 0 | 0 | 34 | 23 | 6 |

The first run compiled its way round the course. By the third, the first six windows of the race compiled nothing at all,
and the frame time followed: 21.4 ms, then 18.1, then 17.2.

## Against the 1.0.0 dump

Same course, same machine, 300-frame windows.

| Build | 3D renders at | Race mean | GPU p50 | Dropped of 300 |
|---|---|---|---|---|
| 1.0.0, first run | 1920x1080 | 36.7 ms, 27.2 fps | 19.8 ms | 229 |
| 1.0.0, third run, warm | 1920x1080 | 30.8 ms, 32.5 fps | 16.9 ms | 202 |
| 1.0.0, render scale 0.75 | 1440x810 | 21.1 ms, 47.5 fps | 12.0 ms | 96 |
| **1.8.0, third run** | **1920x1080** | **17.2 ms, 58.3 fps** | **10.5 ms** | **29** |

1.8.0 at full resolution is twenty-six frames ahead of the best the 1.0.0 dump managed at the same size, and eleven ahead
of what 1.0.0 needed a 25 percent resolution cut to reach, with a fifth of the dropped frames.

That is why the render scale control this repository built for 1.0.0 is not offered on 1.8.0: its stub is registered
against 1.0.0's `main`. The update did the job the control was invented for.

## Where the ceiling is

At 10.5 ms the GPU has six milliseconds of daylight inside a 16.67 ms slot, so the GPU is not what limits this game. What
limits it is the **queue replay thread**, which turns the game's NVN calls into Metal and took 10.11 ms in that same
third race. When that thread is the long pole, resolution does not help and neither does a faster GPU.

The clock matters more. The same thread measured 2.77 GHz in the menus after a race and 1.81 GHz during it. On a laptop
with no fan, several frames of difference come from warm against cool, not from one build against another.

## The hiccups

On the current build, 71 frames over 100 ms in a race session, and the shape of them does not change:

```
704 ms   at  10.7 s | the window goes fullscreen at boot
444 ms   at  36.5 s
399 ms   at  12.6 s
360 ms   at  57.3 s | no pipeline, no upload, no GPU time inside it
354 ms   at 117.3 s | no pipeline, no upload, no GPU time inside it
```

Two problems in one list. The scene cut is textures arriving at once: 130 to 150 MB through the upload path in a single
frame, which is where the 350 to 450 ms ones cluster. The other is the game flipping its **present interval** when it cannot hold 60, which costs a frame with nothing
in it at all. Neither is the GPU.

The pipeline cache is no longer part of it. In the 14 November build 439 frames ran late and only 25 of them involved a
pipeline, and the archive the app now carries was built for this GPU at build time, 919 MB of it.

## The settings that ship

| Setting | What it does | Why |
|---|---|---|
| `DM_GUEST_FILE=1` | the guest's 72.5 GB block in sparse files, out of the footprint | 3.9 to 4.3 GB instead of 8 to 9, which is what an 8 GB machine survives |
| `DM_GUEST_FILE_CPU_UP=0` | file-backed uploads take the GPU path | 3 fps, and nearly half the frames over 100 ms |
| `gpu_keep off` | no GPU clock keeper | the keeper cost 5 fps on this chip |
| `DM_PRESENT_UPSCALE=metalfx` | MetalFX lifts 1080p to the window | the picture, not the frame rate |
| `DM_FPS_HOLD=off` | a late frame does not lengthen the next | worth about 14 fps on the 1.0.0 dump |
| `mem_map_limit on`, `op_mode handheld` | handheld renders 1920x1080 rather than a docked 2560x1440 | the resolution this file measures at |

## Two red lines, neither fatal

`swizzled texture storage is not in a memory pool; falling back to the CPU`, once per run. The upload takes the slow
path and the picture is correct.

`texture type mismatch: shader ... declares slot 0 as Metal type 2, texture ID 337 is type 7`, also once. A shader
samples a texture as a different kind than what is bound, the draw still goes out, and what to watch for is one
odd-looking surface.

Neither appears in the long-frame lines.

## One shader out of 11,601

`Ampere instruction 481 at 0x0 (020c0481) is not implemented`. The translator skipped it and the build went on.
Mario Bros. Circuit never asked for that library, so nothing here missed it. Whatever needs it will be missing until the
instruction lands upstream.

## Heat

No fan. The chip governs itself and the frame rate follows. On 1.0.0 the same course fell from 30 to 22 fps once the chip
sat near 110 C. A warm machine after an hour of play measured 45.7 fps where the same build had done 52.6 cold. Charged,
hard surface, short sessions, lid open to cool.

## What is not measured

- **Another course.** Mario Bros. Circuit is the first track in the game, and quite possibly the lightest. A 24-kart race
  on a heavier one is the number that would matter.
- **The 350 to 450 ms frames with nothing in them.** Which thread is waiting, and whether the guest files are being paged
  at that moment.
- **The scene cut.** Whether it can be made cheaper, and by how much.
- **The chips this was not run on.** M1 through M6, base or Pro.
- **The missing shader.** Which course or effect asks for the one that did not translate.

## Words used here

**Frame time.** How long one picture took, in milliseconds. At 60 fps a frame has 16.67 ms; at 30 fps, 33.34 ms.

**Dropped frame.** A frame that missed its slot. Three in a second is a small skip, thirty is a slideshow.

**p50 and p95.** The middle frame of a window, and the one only one frame in twenty beats. p95 is where stutter lives.

**Pipeline.** A shader plus the settings around it: blending, depth, layout, the target it draws into. Change one and the
GPU wants a different compiled program. Compiling one mid-race is the hitch you feel on a new course.

**The replay thread.** nxnative runs the game's rendering calls on a thread of its own, translating each into Metal. When
that thread is slower than the slot, GPU headroom buys nothing.
