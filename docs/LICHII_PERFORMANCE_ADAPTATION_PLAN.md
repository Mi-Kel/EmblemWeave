# Lichii performance adaptation plan

**Date:** 2026-11-22
**Lichii revision:** `66c555ad9a21b1caaf57a5d9f8f1fea6f90c2508`, branch `nx2`
**Target:** `engine/fefw-recomp/runtime` only (not the launcher, not the Rust runtime)
**Status:** research plan; no production source was changed

## 0. Bottom line

EmblemWeave's render thread is already fast in the measured scene: replay is
~2.1 ms, GPU time ~0.019 ms, and `nvn_cmdq` already moves Vulkan recording to a
worker. The reported 9.18 ms CPU frame time is dominated by a ~7 ms
`Present/Limiter` wait whose attribution is not yet proven
(`docs/CPU_PARALLELISM_RESULTS.md:131-137`). Until that wait is split into
GPU-serial wait, present/acquire wait, limiter sleep and limiter spin, no
renderer-parallelism change can be shown to raise FPS.

Therefore:
- **No Lichii technique is HIGH PRIORITY as a straight port.** The highest-value
  action is instrumentation of the existing wait, which is prerequisite work,
  not a Lichii algorithm.
- The best Lichii-derived candidate is a **unified pipeline job control layer**
  (priority, promotion, cancellation, budget, exception isolation, statistics).
  It is PROMISING because it can remove avoidable pipeline stalls, not because
  Lichii measured FPS with it.
- Parallel texture unswizzling is EXPERIMENTAL: narrow eligibility, no
  game-level evidence, correctness-sensitive.
- The method-level GPU split, runtime shader translation and NVN2 macro/QMD
  work are NOT APPLICABLE.

## 1. Master evidence table

| Technique | Source | Original bottleneck | Evidence | EmblemWeave equivalent | Expected benefit | Risks | Adaptation cost | Recommendation |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Method-level GPU front/back split | `gpu_split.h/.cpp`; `NX2_GPU_THREAD_SPLIT.md`; `NX2_NOTES.md` row 145 | One host GPU thread at ~100%, host GPU 20-40% | Measured 1.46x draws/s and 17% shorter frames, one title, 3 rounds, profiler on | `nvn_cmdq` splits at Vulkan-call level (measured 17.8→13.0 ms p50, 55→77 FPS in issue #29) | Unknown; could move NVN decode/draw setup off the render thread | Guest memory reuse, register mirroring, ordering, 82 call sites, CPU contention | High | NOT APPLICABLE (as a whole); revisit only if profiling shows >2 ms of decode/draw setup |
| SPSC ring with batched publish, idle early publish, spin-then-sleep | `gpu_split.h:1147-1257`, `gpu_split.cpp:501-550` | Producer/consumer latency and cache-line ping-pong | Structural; split A/B above | `nvn_cmdq.cpp:46,113-165` already does this (32 MiB, batch 16, 10-100 µs spin, `WaitOnAddress`) | Microseconds; only if `full_waits`/publish latency are nonzero | Ring bugs are catastrophic; low headroom | Low | EXPERIMENTAL (only if statistics justify) |
| Write-granule tracking for selective waits | `gpu_split.h:1174-1195` | Waiting for all prior GPU work when one range is read | Structural; verification mode | `nvn_gpu_pool.cpp:785,971-1008,1577-1582` page-level `gpu_new`/`cpu_read` guards | None expected; EmblemWeave is finer-grained | Page-guard faults, stale data | High | NOT APPLICABLE |
| Priority shader/pipeline scheduler (NeedNow/Soon/Background) | `shader_jobs.h/.cpp`; 28 tests | Background shader work delays a draw's pipeline | Unit tests; no FPS claim in Lichii docs | Separate prewarm/GPL/texprep pools; no shared priority or promotion | Fewer long-tail pipeline stalls; p99 frame time, not average FPS | Priority inversion, starvation, shutdown, driver thread safety | Medium | PROMISING |
| Byte-budgeted admission + 2 s aging | `shader_jobs.cpp:430-470` | Unbounded in-flight compile memory; background backlog starves Soon work | Unit tests (`shader job budget…`, `…age Background…`) | No byte budget across pools; per-pool limits only | Lower memory spikes; more predictable queueing | Budget mis-set can serialize builds; hard to tune | Medium | PROMISING (fold into the scheduler) |
| Cancellation, exception isolation, `RunOrWait` | `shader_jobs.h:162-177`; tests | Stale jobs hold workers; one failure kills a pool; caller waits idle | 28 tests incl. cancellation/shutdown/exceptions | Pool-specific; no uniform cancellation token | Stability and latency under cache eviction/level change | Use-after-free if a running driver stage is cancelled; must let running driver complete | Medium | PROMISING |
| Deferred native pipeline emission | `docs/nx2/async-native-pipelines.md`; `shader_stage_translation.h` | Runtime IR translation + module creation on the GPU thread (translation itself is parallel only for flushable small texture pools) | Overlap only: 0.2-2.3 s per 10 s; explicitly not FPS; no Windows build | Shaders are offline SPIR-V (`nvn_shaders.cpp:4`); pipeline build already on workers | None | None (no runtime translation stage to move) | n/a | NOT APPLICABLE |
| Parallel unswizzle (2 MiB threshold, native tiling) | `unswizzle_worker.*`; `texture_cache/util.cpp:994` | Large CPU untiling on the upload path | 12 unit tests incl. serial comparison; no frame-time evidence | `tex_prep` prepares whole textures; the untile itself is serial | Lower load/upload spikes for eligible textures; no average-FPS claim | Overlap, guest reuse, scratch memory, pool saturation | Medium | EXPERIMENTAL |
| No fence downloads + read traps | `NX2_NOTES.md` rows 131, 150 | Every fence copied GPU-written ranges (250-330 MB/s) | Measured fences 5→1.1 µs, draw setup −7% | `compute_out_lazy` guarded pages + CPU first-touch copy-back | None expected; already implemented for compute-out | Stale reads if guards miss a writer | High (to widen) | NOT APPLICABLE (verify existing coverage instead) |
| Same-value state suppression | `NX2_NOTES.md` row 135 | ~60% of 6M register writes repeat the current value | Measured macro 185→169 ns, draw setup 5.95→5.80 µs | `bind_is_same` (`nvn_replay.cpp:5089`), `out_masks` (1939) | Already captured; no new gain | Raw-state compare cost | n/a | NOT APPLICABLE |
| UBO update batching | `NX2_NOTES.md` rows 95/107 | Per-update barriers and copies | Measured in Lichii | `batch_ubo_updates` (`nvn_replay.cpp:4957-5012`) | Already captured | Barrier correctness | n/a | NOT APPLICABLE |
| Macro memoization / x86-64 JIT | `NX2_NOTES.md` rows 117, 140 | NVN2 macro replay | Measured (97% repeats; 148 vs 174 ns) | No NVN2 macro engine | None | n/a | n/a | NOT APPLICABLE |

## 2. Prerequisite (FCC-owned, do not duplicate)

### 2.1 Split the `Present/Limiter` wait

Required before any performance claim:
- Timestamp at end of replay, after submit, at present/acquire, and after the
  limiter returns.
- Publish as `wait_gpu`, `wait_present`, `limiter_sleep`, `limiter_spin`.
- Record P50/P95/P99 and a histogram per scene.
- Reconcile the 9.18 ms total, the 5.55 ms 180 FPS budget and the 108.9 FPS
  result; the current report cannot all be true at once.

Acceptance: the four components sum to the frame interval within 0.2 ms on
P50 and P99.

### 2.2 Finish or remove `render_prep`

`render_prep` is default off, and `build_set_async` waits before returning, so
the caller gains no overlap (`nvn_replay.cpp:2900-2942`; `CONFIG.md:69`). FCC is
already working in this area. Do not add a Lichii-style descriptor scheduler on
top. The decision is binary: make descriptor-pool/allocator use thread-safe and
remove the internal wait, or delete the path.

## 3. Recommended Lichii-derived work

### 3.1 P1: Pipeline job control layer (PROMISING)

**What to build (independent implementation, no copied source):**

A small scheduler in `runtime/src/gfx/nvn` used by prewarm, GPL and boot
precache:

```text
enum class JobPriority { NeedNow, Soon, Background };
struct Job { priority; submitted_at; cpu_bytes; driver_bytes; cancel_flag; state; };
```

- Two stages per job (CPU preparation, driver build), two bounded worker pools.
- `NeedNow` bypasses the byte budget; `Soon`/`Background` admit only within
  `pipeline_job_budget_mb` (default measured, not guessed).
- Background older than `pipeline_job_aging_ms` (default 2000) alternates with
  Soon work, so a backlog cannot starve newer builds.
- `Promote(handle, NeedNow)` when a draw/dispatch misses and would build
  synchronously.
- Queued jobs cancel immediately; a running driver build completes, then its
  result is discarded if cancelled. This is Lichii's rule and it is the only
  safe one.
- Capture per-stage exceptions; fail only that job; log with the pipeline key.
- Statistics: queued/running per priority, queue wait P50/P95/P99, budget
  stalls, promotions, cancellations, failed jobs.

**Where it plugs in:** `nvn_prewarm.cpp` (background builds), `nvn_gpl.cpp`
(part builds/upgrades), and the synchronous miss path in `nvn_pipeline.cpp`.
Keep `prewarm_threads` and `gpl_threads` as pool sizes; do not add new knobs
until the layer is proven.

**Expected benefit (label: theoretical):** lower P99 pipeline stalls and fewer
worker-seconds spent on stale builds. It will not raise average FPS in a scene
that never misses a pipeline. Measure `NeedNow` waits before/after; if the
baseline has none, stop.

**Risks:** priority inversion; holding pipeline keys while a driver build
runs; destruction ordering (drain before cache/pipeline destruction);
driver-level thread safety (keep a single driver thread on known-broken
drivers, as Lichii does); cancellation of a running Vulkan build is not
possible, only its result can be dropped.

**Cost:** medium (2-3 files new, integration in 3 existing, tests).

### 3.2 P2: Parallel unswizzle for eligible large uploads (EXPERIMENTAL)

**What to build:**

- A helper `TryParallelUntile(dst, src, layout, pitch, ...)` that is called
  only when: output ≥ 2 MiB, native tiling, depth 1, block_depth 0, pitch not
  4-byte aligned, source/destination spans valid, and the texture worker pool
  has a free slot.
- Copy guest input into owned scratch before dispatch (the guest may reuse or
  overlap it).
- Split at a row band; caller does one half, one pool worker the other; join
  with a latch/future.
- `verify` mode: run serial into a reference, compare, log and use the
  reference on mismatch. Tests must cover GOB layouts, compressed edges,
  overlapping ranges, saturated pool, and cancellation.
- Fall back to serial on any refusal; never block the render thread waiting
  for a slot.

**Where:** the synchronous untiling site used by `nvn_texdata.cpp` /
`nvn_gpu_texture.cpp`, not the `tex_prep` background path (which is already
off the critical thread).

**Expected benefit (label: theoretical):** lower upload/load spikes for large
textures that miss `tex_prep`; no claim on average FPS. Lichii has no
game-level measurement, so the prototype must produce its own.

**Risks:** double memory for scratch, pool contention with `tex_prep`,
incorrect band boundaries for compressed formats, lifetime if the source is
freed early.

**Cost:** medium.

### 3.3 P3: Ring micro-tuning (EXPERIMENTAL, only with evidence)

Only if `nvn_cmdq` statistics show `full_waits > 0` or `sync_ns` is a
meaningful share:
- publish after 8 records when the worker is idle (Lichii's
  `PublishAfterWait`), instead of always 16;
- separate the consumer position and producer position onto different cache
  lines (verify current layout first);
- keep the 32 MiB ring; do not shrink it without measuring full waits.

Do **not** raise `replay_threads` above 2; the current wrappers are built for
one worker, and a second would need per-command-buffer ownership.

## 4. What not to build

| Item | Reason |
| --- | --- |
| Method-level GPU split | 3,554 lines + 82 call sites in Lichii; EmblemWeave already has a Vulkan-level split with measured wins; no evidence decode/draw setup is the bottleneck |
| Runtime shader translation / deferred IR | EmblemWeave uses offline SPIR-V; nothing to translate |
| NVN2 macro JIT, QMD 3.0, MME64, tensor stand-ins | Depend on Switch 2 guest ISA/hardware |
| Second descriptor path | `render_prep` already exists and is unfinished (FCC-owned) |
| Blanket download removal | Guarded lazy compute-out already exists; widening needs a read-trap design and proof |
| Copying Lichii source | GPL-3.0+ compatible, but unnecessary; reimplement the algorithm and preserve attribution if any code is ever taken |

## 5. Benchmark methodology

Use the existing instrumentation (`nvn.rstats`, profiler labels, `nvn_cmdq`
counters) and add only what is missing.

**Scenes:** the densest town/street route, a battle scene, a menu/loading
transition, and a cold-start shader-cache run. Fixed 120 s captures, same save
and camera path.

**Metrics:**
- average FPS; P50/P95/P99 frame time; frames > 20 ms and > 33 ms;
- render-thread busy split: replay, descriptor build, pipeline lookup/build,
  submit, present wait, limiter sleep/spin;
- main-thread execution time (game logic thread, separate from the render
  thread) and time blocked in NVN calls;
- synchronization overhead: `nvn_cmdq` waits and ms/s waited, ring-full waits,
  fence/cache-lock time, timeline-semaphore wait time;
- `nvn_cmdq`: records/s, MB/s, worker busy ms/s, waits and ms/s waited,
  ring-full waits, direct fallbacks;
- GPU submit count and time; queue wait;
- worker utilization and queue-wait percentiles;
- RSS and peak scratch memory; load time; cold pipeline-build time.

**Protocol:**
- 3 alternating A/B runs per scene, median and spread reported.
- Profiler off for FPS; separate profiled runs only for attribution.
- Pin the power plan and driver; record CPU/GPU model, driver version, build
  hash, config, and whether the shader cache was warm.
- Correctness gate: frame-image hash or reference comparison for unswizzle;
  no visual diff, no new validation errors, no crash/hang in 30 minutes.
- Acceptance: P99 frame time improves by ≥5% with non-overlapping run spreads,
  or the change is rejected; average FPS is reported separately and must not
  regress.

## 6. Phases and coordination

| Phase | Work | Owner | Exit criterion |
| --- | --- | --- | --- |
| 0 | Wait-split instrumentation and baseline | FCC + this work, coordinated | Four wait components reconcile with frame time |
| 1 | Pipeline job control layer prototype in an isolated worktree | New work, after FCC agreement | Unit tests + no P99 regression in A/B |
| 2 | `NeedNow` promotion on a real miss; cancellation/exception tests | New work | Measured reduction in synchronous pipeline builds |
| 3 | Parallel unswizzle prototype behind a default-off switch | New work | Serial-reference tests pass; load-spike improvement measured |
| 4 | Ring micro-tuning only if counters justify | New work | Ring-full waits or sync time measurably reduced |

Do not touch FCC's build directories, `E:\EmblemWeave`, saves, caches or
configuration. Use a separate worktree/copy and separate shader cache for any
prototype. No commits, pushes, PRs or issue comments as part of this research.

## 7. Licensing and attribution

- EmblemWeave engine: GPL-3.0-or-later (`runtime/CMakeLists.txt`).
- Lichii: GPL-3.0 repository; the examined mechanisms are
  `GPL-3.0-or-later`. `gpu_split.*` and `unswizzle_worker.*` carry “Copyright
  2026 Eden Emulator Project”; `shader_jobs.*` carries the SPDX line without a
  separate copyright notice.
- GPL-3.0-or-later is compatible with EmblemWeave's engine license, but any
  copied code must retain copyright, SPDX and notices, and the combined work
  must remain GPL-3.0-or-later.
- This plan recommends **independent reimplementation of the algorithms**. If
  code is ever copied, record the exact file, revision and copyright in the
  source header and `NOTICE`-style documentation.
- No Lichii source was copied into EmblemWeave during this assignment.

## 8. Answers to the final-assessment questions

1. **Genuinely applicable:** pipeline job control (priority/promotion/
   cancellation/budget/exception isolation), parallel unswizzle for a narrow
   eligible shape, and the verification methodology. Ring tuning is applicable
   only with counter evidence.
2. **Highest potential benefit:** unknown for FPS, because the measured render
   work is ~2 ms and the dominant wait is unattributed. The largest identified
   *render-side* opportunity is finishing descriptor preparation, which is
   FCC's work, not a Lichii port.
3. **Overlaps with FCC:** `nvn_cmdq` (second thread), `render_prep` (descriptor
   sets), prewarm/GPL, `tex_prep`, and lazy compute-out. Do not duplicate.
4. **Prototype first:** the job control layer, because it is testable in
   isolation and has the clearest architectural fit.
5. **Requires architectural change:** the method-level GPU split, runtime
   shader translation, and any widening of readback traps. None is justified
   now.
6. **Do not port:** GPU split as a whole, NVN2 macro/QMD/MME work, runtime
   shader translation, a second descriptor path, blanket download removal.
7. **What X:\REA and Ghidra added:** see
   `REA_REVERSE_ENGINEERING_FINDINGS.md`. In short, the binary confirms which
   mechanisms are actually compiled into the production runtime (the cmdq
   worker, prewarm/GPL, texprep and guarded-pool paths) and which are absent
   (no priority shader-job scheduler, no parallel unswizzle). Source inspection
   alone cannot distinguish shipped code from dormant code.
8. **Licensing:** GPL-3.0-or-later is compatible; preserve notices; prefer
   independent implementation; no source was copied.
9. **Controlled tests:** wait-split instrumentation, alternating A/B with
   P50/P95/P99, queue-wait and worker-utilization counters, cold/warm shader
   cache, serial-reference image comparison for unswizzle, and a correctness
   gate. See section 5.

## 9. Classification summary

| Class | Items |
| --- | --- |
| **HIGH PRIORITY** | Wait-split instrumentation (prerequisite, not a Lichii port); finish or remove `render_prep` (FCC-owned, not a Lichii port) |
| **PROMISING** | Unified pipeline job control with NeedNow promotion, budget, aging, cancellation and exception isolation |
| **EXPERIMENTAL** | Parallel unswizzle for eligible large uploads; `nvn_cmdq` idle-publish/cache-line tuning |
| **NOT APPLICABLE** | Method-level GPU split, runtime shader translation/deferred IR, NVN2 macro JIT/memoization, QMD 3.0/MME64, tensor stand-ins, GOB-specific tiling, blanket download removal, second descriptor path |
