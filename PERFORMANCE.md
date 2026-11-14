# Performance

Mario Kart World 1.8.0 (v917504) on a MacBook Neo: Apple A18 Pro, 8 GB of unified memory, macOS 27.2. The build is this
repository's `main`, nxnative main `4b10c73f` plus the changes in the README, packed from a 1.8.0 NUP with the pipeline
archive made for this Mac.

Two runs on 1.8.0 and, for comparison, three on the 1.0.0 dump. One course, Mario Bros. Circuit, raced once per run. This is
not a benchmark suite. It is a laptop playing a console game, measured honestly, and it is the best evidence there is right
now that the port is in good shape.

## The headline

**55.2 fps mean in a race, at full 1920x1080, with the GPU inside its budget for the first time.**

| | Second run | First run |
|---|---|---|
| Race, mean frame time | **18.1 ms** | 21.4 ms |
| That is | **55.2 fps** | 46.7 fps |
| 95th percentile | **30.8 ms** | 38.6 ms |
| GPU, mean of the per-window medians | **11.4 ms** | 13.0 ms |
| Dropped frames, per 300 | **50 mean, 24 best, 90 worst** | 101 mean, 138 worst |
| Footprint | 3.8 to 4.7 GB | 4.3 to 4.5 GB |
| Queue replay thread | **11.00 ms at 1.61 GHz** | 12.61 ms at 1.44 GHz |

The difference between the two runs is not the build. It is the pipeline cache.

## The cache, and why the second run is faster

Before the first run the runtime cache was empty, because it had been cleared on purpose when the 1.8.0 app replaced the
1.0.0 one. By the second run it had something in it:

```
pipeline archive .../pso_archive: 16 files (16 with an index of 947 pipelines), 169.2 MB
```

947 pipelines already compiled and indexed, loaded before the first frame. You can watch the rest arrive during the race,
because the log counts bases compiled in each 300-frame window:

| Frames | 300 | 1500 | 2400 | 3300 | 3900 | 4800 | 5400 | 6300 | 6600 |
|---|---|---|---|---|---|---|---|---|---|
| Bases compiled | 15 | 28 | 23 | 36 | 12 | 7 | 0 | 0 | 8 |

The first third of the race is still hiring, 25 to 36 bases every five seconds. By the last third it is idle, then it wakes
up for eight more when the track shows something new. Every run moves that frontier forward, and the frame time follows it:
the early windows sit at 17.4 ms with 33 frames dropped, the late ones at 17.0 to 18.6 with 24 to 40.

This is the single most valuable thing your play produces, and it is why the second time is always better than the first.

## Against the 1.0.0 dump

Same course, same machine.

| Build | 3D renders at | Race mean | GPU p50 | Dropped of 300 |
|---|---|---|---|---|
| 1.0.0, first run | 1920x1080 | 36.7 ms, 27.2 fps | 19.8 ms | 229 |
| 1.0.0, third run, warm | 1920x1080 | 30.8 ms, 32.5 fps | 16.9 ms | 202 |
| 1.0.0, render scale 0.75 | 1440x810 | 21.1 ms, 47.5 fps | 12.0 ms | 96 |
| **1.8.0, second run** | **1920x1080** | **18.1 ms, 55.2 fps** | **11.4 ms** | **50** |

Read that table twice. 1.8.0 at full resolution is nearly seventeen frames ahead of the best the 1.0.0 dump could do at the
same size, and eight ahead of what 1.0.0 needed a 25 percent resolution cut to reach.

The render scale control this repository built for 1.0.0 is therefore unused on 1.8.0, and not offered: its stub is
registered against 1.0.0's `main`. The update did the job the control was invented for.

The menus improved too. On 1.0.0 they dropped 3 to 50 frames of 300; on 1.8.0 the best window dropped one.

## Where the ceiling is now

The GPU is no longer the interesting number. At 11.4 ms it has five milliseconds of daylight inside a 16.7 ms slot, and the
work of a frame is set somewhere else: the queue replay thread, which turns the game's NVN calls into Metal, spends 11.00 ms
at 1.61 GHz. When that thread is the long pole, resolution does not matter and neither does the GPU.

What does matter is the clock. The same thread measured 2.77 GHz in the menus after the race. On a laptop with no fan, the
difference between 55 and 60 fps is often the difference between warm and cool, not between one build and another.

## The hiccups

367 frames over the deadline in the race, of which 28 involved a pipeline, 51 pulled more than a megabyte of textures, and 68
were over 50 ms. The worst are not compiles:

```
frame 2414   409.5 ms | pipelines 1 | textures 0.0 MB     the race start
frame 4227   376.5 ms | pipelines 0 | textures 0.1 MB
frame 4495   289.3 ms | pipelines 0 | textures 0.7 MB
```

Three hundred milliseconds with no pipeline, no upload and no GPU time inside it. Same shape on 1.0.0, so it did not arrive
with the update. It lives in the game's own threads or in the host around them, and it is the open item that would move the
1 percent worst frames the most.

## Two red lines, neither fatal

`swizzled texture storage is not in a memory pool; falling back to the CPU`, once per run. The upload takes the slow path and
the picture is correct.

`texture type mismatch: shader d92d9b75e2cc7dad_b (stage 1) declares slot 0 as Metal type 2, texture ID 337 is type 7`, also
once. A shader samples a texture as a different kind than what is bound, the draw still goes out, so the thing to watch for
is one odd-looking surface.

Both appeared in both runs and neither correlates with a dropped frame.

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
why a first race at all compiles dozens rather than thousands.

## Heat

No fan. The chip governs itself and the frame rate follows it. On 1.0.0 the same course fell from 30 to 22 fps once the chip
sat near 110 C. Here the replay thread read 1.61 GHz during the race and 2.77 GHz in the menus after it. Charged, hard
surface, short sessions, lid open to cool.

## What is not measured

- A second course, so whether Mario Bros. Circuit is typical is unknown.
- A cool machine against a warm one, which is the only way to separate 1.8.0's gains from the clock's.
- Whether the 300 ms stalls are the game, the host, or the guest files being paged.
- Which course or effect asks for the shader that did not translate.
- What the texture type mismatch draws.

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
