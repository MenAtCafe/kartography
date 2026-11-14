# Performance

Mario Kart World 1.8.0 (v917504) on a MacBook Neo: Apple A18 Pro, 8 GB of unified memory, macOS 27.2. The build is this
repository's `main`, nxnative main `4b10c73f` plus the changes in the README, packed from a 1.8.0 NUP with the pipeline
archive made for this Mac.

Three runs on 1.8.0 and, for comparison, three on the 1.0.0 dump. One course, Mario Bros. Circuit, raced once per run. This
is not a benchmark suite. It is a laptop playing a console game, measured honestly, and it is the best evidence there is
right now that the port is in good shape.

## The headline

**58.3 fps mean in a race, at full 1920x1080, with the GPU finishing in 10.5 ms against a 16.67 ms slot.**

| | Third run | Second run | First run |
|---|---|---|---|
| Race, mean frame time | **17.2 ms** | 18.1 ms | 21.4 ms |
| That is | **58.3 fps** | 55.2 fps | 46.7 fps |
| 95th percentile | **30.9 ms** | 30.8 ms | 38.6 ms |
| GPU, mean of the per-window medians | **10.5 ms** | 11.4 ms | 13.0 ms |
| Dropped frames, per 300 | **29 mean, 3 best, 67 worst** | 50 mean, 24 best, 90 worst | 101 mean, 138 worst |
| Footprint | 3.5 to 4.6 GB | 3.8 to 4.7 GB | 4.3 to 4.5 GB |
| Pipelines cached at start | **1,283** | 947 | 0 |
| Queue replay thread | **10.11 ms at 1.81 GHz** | 11.00 ms at 1.61 GHz | 12.61 ms at 1.44 GHz |

Same build, same course, same settings. The only thing that changed between those three columns is what the pipeline cache
had already learned.

## The cache, and why every run is faster

The first run started with an empty cache, cleared on purpose when the 1.8.0 app replaced the 1.0.0 one. It bought 46.7
fps and compiled its way through the track. By the third run the cache held 1,283 pipelines in 169 MB, and the first six
windows of the race compiled **nothing at all**:

| Frames | 2400 | 2700 | 3000 | 3300 | 3600 | 3900 | 4200 | 4500 |
|---|---|---|---|---|---|---|---|---|
| Bases compiled | 0 | 0 | 0 | 0 | 0 | 34 | 23 | 6 |

Then a burst of 34 when the track showed something new, and a trickle after it, 23, 6, 16, 4, 19, seven, three, two, until
it reached zero again. Every run pushes that frontier further out, and the frame time follows: 21.4 ms, then 18.1, then
17.2.

This is the single most valuable thing your play produces. It is also why the first race after a rebuild always looks worse
than the machine actually is.

## Against the 1.0.0 dump

Same course, same machine.

| Build | 3D renders at | Race mean | GPU p50 | Dropped of 300 |
|---|---|---|---|---|
| 1.0.0, first run | 1920x1080 | 36.7 ms, 27.2 fps | 19.8 ms | 229 |
| 1.0.0, third run, warm | 1920x1080 | 30.8 ms, 32.5 fps | 16.9 ms | 202 |
| 1.0.0, render scale 0.75 | 1440x810 | 21.1 ms, 47.5 fps | 12.0 ms | 96 |
| **1.8.0, third run** | **1920x1080** | **17.2 ms, 58.3 fps** | **10.5 ms** | **29** |

Read that table twice. 1.8.0 at full resolution is twenty-six frames ahead of the best the 1.0.0 dump could do at the same
size, and eleven ahead of what 1.0.0 needed a 25 percent resolution cut to reach, with a fifth of the dropped frames.

The render scale control this repository built for 1.0.0 is therefore unused on 1.8.0, and not offered: its stub is
registered against 1.0.0's `main`. The update did the job the control was invented for.

The menus improved too. On 1.0.0 they dropped 3 to 50 frames of 300; on 1.8.0 the best window dropped three.

## Where the ceiling is now

The GPU is no longer the interesting number. At 10.5 ms it has six milliseconds of daylight inside a 16.7 ms slot, and the
work of a frame is set somewhere else: the queue replay thread, which turns the game's NVN calls into Metal, spends 10.11 ms
at 1.81 GHz. When that thread is the long pole, resolution does not matter and neither does the GPU.

What does matter is the clock. The same thread measured 2.77 GHz in the menus after a race. On a laptop with no fan, the
difference between 58 and 60 fps is often the difference between warm and cool, not between one build and another.

## The hiccups

439 frames over the deadline in the third race, of which only **25** involved a pipeline. 97 pulled more than a megabyte of
textures and 123 were over 50 ms. The worst are dominated by uploads:

```
frame 3484   779.7 ms | submit 766.9, GPU 13.5 | 154.2 MB of textures
frame 3766   573.9 ms | submit 564.7, GPU  9.5 |  99.9 MB
frame 4092   468.8 ms | submit  10.4, GPU 22.2 |   0.0 MB
frame 3102   457.6 ms | submit   2.7, GPU  4.5 |   0.0 MB
```

Two different problems in one list. The top two are texture streams landing in a single frame, and almost all of the time is
in submit: those 154 and 100 megabytes go through the CPU upload gate, which exists because a no-copy Metal buffer over the
guest's file pages would be read in and wired whole. `DM_GUEST_FILE_CPU_UP=0` would put them back on the GPU path, which is
the experiment to run.

The bottom two are the other thing: 450 ms with single-digit submit and GPU inside it. That is the guest-side stall that has
been there since the 1.0.0 build, and it is still open.

## Two red lines, neither fatal

`swizzled texture storage is not in a memory pool; falling back to the CPU`, once per run. The upload takes the slow path and
the picture is correct.

`texture type mismatch: shader d92d9b75e2cc7dad_b (stage 1) declares slot 0 as Metal type 2, texture ID 337 is type 7`, also
once. A shader samples a texture as a different kind than what is bound, the draw still goes out, so the thing to watch for
is one odd-looking surface.

Both appeared once in each run, and neither shows up in the long-frame lines.

## One shader out of 11,603

`Ampere instruction 481 at 0x0 (020c0481) is not implemented`. The translator skipped it and the build went on. Mario Bros.
Circuit never asked for that library, so nothing here missed it. Whatever needs it will be missing until the instruction is
implemented upstream.

## The configuration that ships

`DM_GUEST_FILE=1` puts the guest's 72.5 GB block in 73 files of 1 GB, out of the footprint. The game's address regions reach
+1.62 GB into a 4 GB alias region in the first 15 seconds, so there is room. `mem_map_limit on` and `op_mode handheld` are the
profile's: handheld renders 1920x1080 rather than a docked 2560x1440, and MetalFX lifts that to the window at 2816x1584.
`DM_FPS_HOLD=off` keeps late frames from being given an extra vsync, which on the 1.0.0 dump was worth about 14 fps by itself.

The app also carries 776.8 MB of pipelines built at build time for this GPU: 9,591 render, 728 compute, none failed. That is
why the first race at all compiles dozens rather than thousands.

## Heat

No fan. The chip governs itself and the frame rate follows it. On 1.0.0 the same course fell from 30 to 22 fps once the chip
sat near 110 C. Here the replay thread read 1.81 GHz during the third race and 2.77 GHz in the menus after it. Charged, hard
surface, short sessions, lid open to cool.

## What is not measured

- **Cold against warm.** Every run so far came after a long session. Nothing here separates 1.8.0's gains from the clock's, and
  a run on a cold machine could be several frames higher.
- **Another course.** Mario Bros. Circuit is the first track in the game and quite possibly the lightest. A 24-kart race on a
  heavier one is the number that would matter.
- **The 300 ms stalls.** Which thread, and whether the guest files are being paged at that moment.
- **The upload hitches.** Whether `DM_GUEST_FILE_CPU_UP=0` turns the 779 ms frames into something better or worse.
- **The missing shader.** Which course or effect asks for the one that did not translate.

## Words used here

**Frame time.** How long one picture took, in milliseconds. At 60 fps a frame has 16.67 ms; at 30 fps, 33.34 ms. Every
number in this file is frame time, dropped frames or memory, because those are what you feel.

**Dropped frame.** A frame that missed its slot: the picture lingered while the new one was late. Three dropped frames in a
second is a small skip, thirty is a slideshow. A window is 300 frames, five seconds at 60 Hz.

**p50 and p95.** The middle frame of a window, and the one only one frame in twenty beats. p95 is where stutter lives.

**Pipeline.** A shader plus the settings around it: blending, depth, layout, the target it draws into. Change one and the GPU
wants a different compiled program. Compiling one mid-race is the hitch you feel on a new course.

**The replay thread.** nxnative runs the game's rendering calls on a thread of its own, translating each into Metal. When
that thread is slower than the slot, GPU headroom buys nothing.
