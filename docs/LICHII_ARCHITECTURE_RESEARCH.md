# Lichii architecture research

**Date:** 2026-11-22
**Subject:** [zortos293/Lichii](https://github.com/zortos293/Lichii), the official repository only
**Revision examined:** `66c555ad9a21b1caaf57a5d9f8f1fea6f90c2508`
**Branch:** `nx2`
**Commit date:** 2026-11-22T22:42:09Z
**Commit subject:** `lichii: clarify per-game shader precache in What's New (#2473)`
**Version file:** `0.1.0-rc1`
**License:** GPL-3.0 (repository `LICENSE.txt`); the mechanisms below carry
`SPDX-License-Identifier: GPL-3.0-or-later`
**Local checkout:** `X:\REA\work\lichii-research` (read-only research copy)

This report is a source and documentation investigation. Lichii was not built,
not run with a game, and no keys, firmware, game files or third-party emulator
packages were obtained. Its performance claims are therefore quoted from the
project's own notes and tests and are separated from what was independently
measured (nothing, for Lichii).

## 1. What Lichii is

Lichii describes itself as an experimental Nintendo Switch and Switch 2 emulator
in C++, building on Eglantine, Eden, yuzu and Sudachi. The `nx2` branch extends
the Switch 1 GPU emulation to Switch 2 titles (NVN2, Ampere classes, Turing
MME64 macros, QMD 3.0 compute). Most of its performance notes are about a single
Switch 2 title (*Mario Kart World*) on one host.

This matters for adaptation: the parts that are generic C++/Vulkan scheduling
can travel; the parts that depend on NVN2 macros, QMD layout, Switch 2 GOB
tiling, or a specific game's submission pattern cannot.

## 2. GPU thread split (`gpu_split.h`, `gpu_split.cpp`)

### 2.1 Architecture

The GPU work is split at the **engine method call** boundary, not at the Vulkan
call boundary:

- **Front (the existing GPU thread):** takes the guest CPU's command queue,
  parses GPFIFO entries and command lists (`DmaPusher`), runs macros on the
  compiled/JIT path, and keeps copies of the registers and macro data the
  macros read.
- **Back (second thread):** executes the resulting ordered stream of method
  writes: Maxwell3D / KeplerCompute / Fermi2D / DMA `CallMethod`, register
  state, dirty flags, constant-buffer and inline uploads, draws, dispatches,
  queries, fences, and the rasterizer and its caches. Only the back touches the
  rasterizer.

The stream is a single-producer/single-consumer ring of
`{engine, method, argument}` words plus copied payloads. Multi-word methods
(which often read guest command memory in place) are copied into the ring;
CPU-side commands such as flushes, invalidations, `RunInOrder` actions and
presentation are forwarded through the same ring so ordering is preserved.

Source:
- `src/video_core/gpu_split.h:181-220`: the class contract.
- `NX2_GPU_THREAD_SPLIT.md`: the design note (split point, front state,
  sync points, stages, measured status).

### 2.2 Ring mechanics

- **Capacity:** 8 MiB (`RingWords = 1 << 20` 64-bit words,
  `gpu_split.h:1147-1148`).
- **Entries never wrap.** A `Wrap` record fills the remainder of the ring and
  the producer continues at the start (`gpu_split.cpp:501-512`).
- **Reserve** checks the cached consumer position before touching the consumer
  cache line; only when the ring looks full does it re-read the consumer
  (`gpu_split.h:1224-1233`).
- **Backpressure:** `WaitForSpace` publishes, spins up to 4096 `PAUSE`
  iterations, then sleeps on the back's wake primitive
  (`gpu_split.cpp:515-536`).
- **Publishing:** normal publish every 256 words
  (`PublishEvery`, `gpu_split.h:1150`), but after only 8 unpublished words when
  the back is known idle (`PublishAfterWait`, `:1152`;
  `PublishIfDue`, `:1240-1257`). Publish uses a sequentially consistent store
  and wakes the back only if it is waiting (`gpu_split.cpp:538-550`).
- **Consumption:** the back reads the published position with acquire, executes
  records in place, and stores its consumed position with release once per
  batch (`gpu_split.cpp` worker loop). The front's waits read the consumed
  position with acquire.
- **Wake/sleep:** the front's waits spin, then sleep; the back has its own
  idle hook so deferred GPU submission still happens when the ring is empty.
  `SettleIdleBack` asks for that hook without draining.

This is a conventional but carefully tuned SPSC design. Its important property
is not “lock-free” as a label: it is that the consumer position is read rarely,
publishes are batched, and the producer never allocates while sending.

### 2.3 Ordering and guest memory

The split is only correct because the front keeps enough state to answer what
it needs without asking the back:

- **Register mirroring.** Every register write the front sends is mirrored;
  back-side changes the front must reproduce are modelled explicitly
  (constant-buffer offset, `shadow_scratch[0]`, shadow RAM control, macro upload
  pointers, `native_registers`). `EDEN_NX2_SPLIT_VERIFY=1` compared the mirrors
  against the back's registers.
- **Copied payloads.** Command lists are frequently guest memory read in place,
  which the guest reuses. Multi-word methods are copied into the ring. The
  front also copies list words it must read after the guest may have rewritten
  them.
- **Write tracking.** The front records the position of writes by 256-byte
  granule (64 KiB for copies and uploads, up to 16 MiB) in small hashed sets
  (`gpu_split.h:1174-1195`). A read waits only for the last write to its
  granule, not for the whole stream. The sets keep an `evicted` position so a
  collision can make a wait longer but cannot make it miss a write.
- **Drain sites.** The front waits for the back only where a result is needed:
  semaphore acquires, macro DMA reads, syncpoint waits, command-list writers,
  GPU maps, and a full ring. `DrainSite` enumerates them
  (`gpu_split.h:43-75`) and every wait is counted and timed by site.

The design note estimates the front at ~35-40% of the work and the back at
~60-65%, yielding a ceiling of ~1.4-1.6x. It also states the back is still ~92%
busy, ~53% of that in draw setup, so the next limit is the back itself.

### 2.4 Lifetime and failure handling

- `MaybeStart` creates the ring and thread only for Switch 2 titles with the
  Vulkan backend; `EDEN_NX2_SPLIT=0/1/2` selects off, methods-on-back, or
  macros-on-front (`gpu_split.h:675`, `gpu_split.cpp:173`).
- `Stop` joins the back (`gpu_split.h:681`, `gpu_split.cpp:421`).
- A debug switch (`EDEN_NX2_SPLIT_CHECK`) reports any other thread using
  back-owned state, with a stack.
- The code has explicit handling for the render follower being parked or lent,
  and for posted operations, because the back can post work to the Vulkan
  render thread in the more advanced modes.

The split is not a small patch: `GpuSplit` is referenced from 53 source files
and `GpuSplit::Get()` appears 82 times. That figure is the honest measure of
its integration cost.

### 2.5 Measured evidence

`NX2_NOTES.md` row 145 records a clean A/B on one machine, three rounds each,
with the profiler enabled in both arms:

| Arm | Draws/second | Frame time |
| --- | --- | --- |
| One thread | 41,100 / 48,600 / 37,000 | 30.5 / 35.0 / 27.1 ms |
| Split (`EDEN_NX2_SPLIT=2`) | 59,200 / 62,400 / 62,700 | 28.6 / 23.3 / 25.1 ms |

The note reports **1.46x throughput and 17% shorter frames**, zero crashes and
zero register differences in 57,089 checks. Row 142 reports ~47,400 draws/s
with the split against ~44,000 without in an earlier three-round comparison.

Limits of that evidence:
- One title, one machine, one driver, a profiled build in both arms.
- Draw throughput is not the same as presented FPS; the frame times are the
  more relevant number.
- The split's win depends on the guest having enough independent work. A
  single-threaded, draw-light workload would gain little and could lose to the
  ring and wake overhead.

## 3. Shader and pipeline jobs (`shader_jobs.h`, `shader_jobs.cpp`)

### 3.1 Scheduler

`ShaderJobScheduler` has two worker pools:

- **CPU workers** run a job's CPU stage, which returns the driver stage.
- **Driver workers** run the returned driver stage (for Vulkan, the
  `vkCreate*Pipelines` call).

Both pools take the highest priority first:

- `NeedNow`: a draw or dispatch is waiting.
- `Soon`: bound or boot build of a used pipeline, not drawn yet.
- `Background`: prebuild, precache, relowering, migration.

Properties:
- **Memory budget.** Each stage declares estimated bytes. In-flight bytes are
  the estimates of running stages. A `Soon`/`Background` stage starts only if
  it stays within the budget, or if nothing is in flight; `NeedNow` always
  starts (`Picker`, `shader_jobs.cpp:380-430`).
- **Aging.** A `Background` job queued for more than `background_aging`
  (default 2 s) takes every other pick ahead of `Soon` jobs submitted after it
  (`Pick`, `shader_jobs.cpp:390-420`).
- **Promotion.** `Promote` moves a queued or running job to a higher priority;
  a lower priority is ignored.
- **Immediate work.** `RunOrWait` promotes to `NeedNow`; if still queued, the
  calling thread runs the CPU stage itself and then the driver stage, rather
  than waiting for a worker. If a worker already runs it, the caller waits.
- **Cancellation.** Queued jobs cancel at once. A running CPU stage sees
  `StopRequested()` and no driver stage follows. A running driver stage
  completes. Batches cancel together.
- **Exceptions.** Each stage's exception is captured in the job; only that job
  fails. The scheduler logs it and notifies waiters; it does not terminate the
  pool.
- **Shutdown.** Stops new work, cancels queued jobs, wakes all workers and
  joins them; a CPU stage that finishes during shutdown gets no driver stage.
- **Lifetime.** The CPU stage's captures are released before the driver stage
  starts, so IR pools/programs are not held through the driver compile.
- **Stack.** Workers run on a large stack (`RunOnLargeStack`) at low/background
  priority.

Source: `src/video_core/shader_jobs.h:129-244`, `src/video_core/shader_jobs.cpp:224-649`.

The scheduler is configured in `src/video_core/renderer_vulkan/vk_pipeline_cache.cpp:867-876`:
CPU workers from `GetTotalPipelineWorkers()`, driver workers
`min(cpu_threads, max(hardware_concurrency()/2, 2))`, forced to 1 on drivers
with broken parallel shader compilation and to 2 on legacy AMD GCN.

Tests: `src/tests/video_core/shader_jobs.cpp` has 28 cases covering priority,
promotion, aging, budget, CPU-byte release, NeedNow bypass, cancellation,
batches, shutdown, exceptions, `RunOrWait`, thread naming, multi-producer
consistency, and `ParallelFor`.

### 3.2 Deferred native pipeline emission

`docs/nx2/async-native-pipelines.md` is unusually candid:

- The deferred **native pipeline** path moves SPIR-V emission, shader-module
  creation and the driver build to a worker. Its IR translation stays on
  `GPUBack` because it reads live GPU state (texture types, constant-buffer
  values).
- There is a second, separate path: `shader_stage_translation.h` translates the
  stages of one pipeline as `NeedNow` jobs, with the calling thread running any
  stage no worker took (`TranslateStages`). It is allowed only when the texture
  descriptor pool can be flushed whole (`CanFlushTexturePool`, ≤ 4 MiB) and the
  stage environments are not used elsewhere; past that size the stages
  translate on the GPU thread. So “translation stays on GPUBack” is true of the
  async-native-pipeline path, not of every Lichii translation.
- `Configure` needs `Shader::Info`, so descriptor layouts, allocators and cache
  configuration stay on `GPUBack` under the rasterizer locks.
- Each deferred job owns its IR pools, programs and bindings. The worker does
  not read engine registers, guest memory, buffer/texture caches, or record
  scheduler commands.
- Deferred jobs are capped at the pipeline worker count; when full, the old
  synchronous path runs. Up to four IR pools are retained.
- Pipelines may be evicted before a queued callback runs; callbacks own shared
  state, and destruction drains workers before resources disappear.
- The document explicitly says the reported **0.2-2.3 s per 10 s** is
  overlapped work, **not an FPS improvement**, and that no Ocarina runtime
  measurements or Windows build are claimed. Real savings require cold-cache
  A/B sessions.

That is the correct standard for this kind of change, and it is the standard
EmblemWeave should hold any adoption to.

## 4. Parallel texture unswizzling (`unswizzle_worker.*`)

`UnswizzleWorker::TryUnswizzle` is a narrow fast path:

- Minimum **2 MiB** of host output (`ParallelUnswizzleMinBytes`,
  `unswizzle_worker.h:14`).
- Only native tiling, `depth == 1`, `block_depth == 0`.
- Only when the row pitch is **not** 4-byte aligned (`(pitch & 3) == 0` returns
  false, `unswizzle_worker.cpp:23-27`). This is a surprisingly restrictive
  condition and means many ordinary aligned textures do not take the path.
- The guest input is copied into an owned scratch buffer first, because the
  source may overlap the destination or be reused by the guest.
- The image is split at a row/band boundary into two halves. One half is queued
  to the shared texture worker pool; the calling thread does the other half.
  A `std::latch` joins them.
- Guards: guest bytes ≤ 32 MiB and ≤ 2× host bytes; input/output spans must be
  large enough. If the pool refuses the job, it returns false and the serial
  path runs.
- `verify` runs the serial implementation into a reference buffer, compares,
  logs a mismatch, and uses the reference.

Usage: `src/video_core/texture_cache/util.cpp:994-1000` calls it per layer when
the image is 2D, single-sample, and otherwise eligible.

Tests: `src/tests/video_core/unswizzle_workers.cpp` has 12 cases, including GOB
layout sweeps, below-threshold and unsplittable inputs, a saturated pool,
overlapping source/destination, compressed mip tails, independent owners
sharing the pool, and two microbenchmarks (tagged so they do not run by
default).

There is **no game-level frame-time evidence** for this change in `NX2_NOTES.md`.
Its case rests on the unit tests and on the general observation that large
uploads are CPU work.

## 5. GPU memory, synchronization and batching

The notes contain several measured, generic ideas:

- **No buffer downloads at fences for Switch 2 titles** (row 131). Eden used to
  copy every GPU-written range back at the next fence (250-330 MB/s,
  13,000-20,000 ranges/s). The note says the native MME DMA reads never touched
  those pages, and with reactive flushing the guest read GPU-written memory at
  most 64 times in 200 s. Turning it off took fences from ~5 µs to ~1.1 µs and
  draw setup down 7%.
- **Read traps for correctness** (row 150). With downloads off, a page written
  by the GPU but not downloaded loses read permission once. The guest's first
  read faults, the handler flushes that range through the buffer cache and
  restores the permission. The range becomes `Preflushable` and is downloaded
  at every later fence. Diagnostic run: ~3,800 pages armed, ~3,500 trapped
  reads.
- **Per-thread device-memory translation cache** (row 125). A cache shared by
  all threads was cleared by a service-thread map between a null check and a
  copy, causing crashes. A per-thread cache with a generation counter removed
  them.
- **Fence batching and fewer cache locks** (rows 120, 123, 124). The fence path
  slept on the texture/buffer cache mutexes ~8% of the GPU thread; one lock per
  cache per fence, a spinning recursive mutex and separate locks took that to
  3.7%. Of 5,500-7,000 fences a second, ~20% had nothing to wait for; skipping
  empty fences took `SignalFence` from 7.6% to 3.9%. Releasing signalled fences
  together reduced cache-lock spinning from 7.0% to 5.5%; the per-fence cost
  itself was unchanged within noise.
- **Buffer collector sized by the buffers, not the whole device** (row 139).
  The collector compared total device use (textures ~6.8 GB) with thresholds
  from an 8 GB card, so it evicted up to 64 buffers every frame and downloaded
  GPU-written ones with a wait (~5% of the GPU thread). Sizing the buffer
  collector by its own 1 GiB/2 GiB thresholds removed that cost and stopped
  presents blocking on those courses.
- **Skip same-value state writes** (row 135). ~6 million 3D register writes a
  second, ~60% writing the value the register already holds, mostly from macro
  replays. Dropping them after shadow-RAM processing took macro calls from
  185 ns to 169 ns and draw setup from 5.95 µs to 5.80 µs.
- **Defer inline-write invalidations** (row 134). Each inline write invalidated
  the buffer, texture, shader and query caches separately. Coalescing guest
  memory writes and merging ranges at the next `FlushCaching` cut inline writes
  from 10.5% to 4.4% of the GPU thread.

These are the most directly reusable results because they are about avoiding
redundant synchronization, not about Switch 2 instruction semantics.

## 6. Dispatch, batching and emulator-specific work

Measured but **not portable** without the same guest architecture:

- Turing MME64 macro memoization (~97% exact repeats) and an x86-64 macro JIT
  (148 ns vs 174 ns per call, rows 117 and 140).
- Native compute QMD handling, release slots 24/28/52, per-channel MME data RAM
  (rows 60, 84, 136).
- Task/mesh shader emulation and DLSS-style tensor-network stand-ins (rows 64,
  89).
- Switch 2 ASTC/GOB tiling and BC1/BC3 recompression choices (rows 99, 103,
  107).

Measured and generic, but already present in EmblemWeave in some form:

- Submission batching and present pacing.
- Same-state and no-op bind suppression.
- Fence batching and selective cache locking.

## 7. Evidence classification

| Claim | Kind | What supports it | What it does not show |
| --- | --- | --- | --- |
| 1.46x draw throughput, 17% shorter frames from the split | Measured, one title | `NX2_NOTES.md` row 145, 3 rounds, profiler on both arms | FPS in other titles, CPU-bound or draw-light scenes; profiler-free absolute numbers |
| ~47,400 vs ~44,000 draws/s (stage 2) | Measured, one title | Row 142 | Same |
| Async pipeline emission 0.2-2.3 s per 10 s | Measured overlap, not FPS | `docs/nx2/async-native-pipelines.md` | Any FPS gain; cold-cache A/B was not run; no Windows build claimed |
| Parallel unswizzle | Unit-tested only | 12 test cases incl. serial comparison | Frame time, upload latency, real texture shapes |
| No downloads at fences + read traps | Measured | Rows 131, 150 | Other titles that genuinely read GPU-written buffers every frame |
| Fence/cache-lock reductions | Measured | Rows 120, 124 | Titles with different fence rates |
| Same-value state skip | Measured | Row 135 | Titles whose state changes are not repetitive |
| Macro memo/JIT | Measured | Rows 117, 140 | Anything outside NVN2 MME64 |

## 8. Findings that matter for EmblemWeave

1. The GPU split is a **method-level** producer/consumer design. EmblemWeave
   already has a **Vulkan-call-level** producer/consumer design (`nvn_cmdq`).
   They are not substitutes; the remaining question is whether EmblemWeave can
   move more draw preparation, not whether it needs “a second GPU thread”.
2. Lichii's shader scheduler is a well-tested **priority/budget/cancellation**
   design, but its CPU stage translates guest shaders at runtime. EmblemWeave's
   shaders are translated offline to SPIR-V, so only module creation and driver
   pipeline builds could be scheduled this way.
3. Lichii's parallel unswizzle is narrow and has no frame-time evidence. It is
   a candidate only for the remaining synchronous large-texture path.
4. Lichii's memory work is mostly **removing redundant downloads and locks**,
   with a correctness mechanism (read traps) rather than a blanket removal.
   EmblemWeave already has a page-guarded lazy compute-result path with CPU-read
   tracking; that area needs verification, not a port.
5. The strongest transferable asset may be the **verification methodology**:
   every optimization has a switch, a serial reference, counters, and an A/B
   result. That is cheaper to adopt than any individual algorithm.

## 9. Licensing and attribution audit

Repository licence: `LICENSE.txt` is **GNU GPL version 3**; the repository also
ships `LICENSES/GPL-3.0-or-later.txt`, `GPL-2.0-or-later.txt`, and the usual
third-party licence texts (Apache-2.0, BSD-2/3-Clause, MIT, MPL-2.0, etc.). The
project describes itself as building on Eglantine, Eden, yuzu and Sudachi.

Per-file audit of the mechanisms examined:

| File | SPDX | Copyright notice | Licence |
| --- | --- | --- | --- |
| `src/video_core/gpu_split.h` | `GPL-3.0-or-later` | Copyright 2026 Eden Emulator Project | GPL-3.0-or-later |
| `src/video_core/gpu_split.cpp` | `GPL-3.0-or-later` | Copyright 2026 Eden Emulator Project | GPL-3.0-or-later |
| `src/video_core/shader_jobs.h` | `GPL-3.0-or-later` | none in the file | GPL-3.0-or-later |
| `src/video_core/shader_jobs.cpp` | `GPL-3.0-or-later` | none in the file | GPL-3.0-or-later |
| `src/video_core/shader_stage_translation.h` | `GPL-3.0-or-later` | none in the file | GPL-3.0-or-later |
| `src/video_core/shader_stage_translation.cpp` | `GPL-3.0-or-later` | none in the file | GPL-3.0-or-later |
| `src/video_core/textures/unswizzle_worker.h` | `GPL-3.0-or-later` | Copyright 2026 Eden Emulator Project | GPL-3.0-or-later |
| `src/video_core/textures/unswizzle_worker.cpp` | `GPL-3.0-or-later` | Copyright 2026 Eden Emulator Project | GPL-3.0-or-later |
| `src/video_core/renderer_vulkan/pipeline_shader_emission.h` | `GPL-3.0-or-later` | none in the file | GPL-3.0-or-later |
| `src/tests/video_core/shader_jobs.cpp` | `GPL-3.0-or-later` | none in the file | GPL-3.0-or-later |
| `src/tests/video_core/unswizzle_workers.cpp` | `GPL-3.0-or-later` | Copyright 2026 Eden Emulator Project | GPL-3.0-or-later |

These files depend only on the C++ standard library, the project's own
types/utilities and Vulkan; no separate third-party licence is introduced by
the specific mechanisms above. The repository as a whole contains
GPL-2.0-or-later and permissive third-party code elsewhere, which matters only
if other files are reused.

Compatibility with EmblemWeave:
- The production engine is GPL-3.0-or-later (`runtime/CMakeLists.txt`), and
  `LICENSE` is GPLv3. GPL-3.0-or-later code can be combined with it; the
  combined work stays GPL-3.0-or-later.
- GPL-2.0-or-later files would also be compatible via the “or later” option,
  but the mechanisms examined are GPL-3.0-or-later, so that question does not
  arise for them.
- Attribution is mandatory: keep the copyright notice, the SPDX identifier and
the licence text; mark modified files and state the changes. The safest course,
and the one recommended here, is an independent implementation of the
*algorithm* with a citation in the commit/design note, not a copy of the code.
- No Lichii source was copied into EmblemWeave during this assignment.
