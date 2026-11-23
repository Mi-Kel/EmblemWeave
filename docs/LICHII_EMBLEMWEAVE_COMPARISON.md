# Lichii and EmblemWeave: architecture comparison

**Date:** 2026-11-22
**Lichii revision:** `66c555ad9a21b1caaf57a5d9f8f1fea6f90c2508`, branch `nx2`
**EmblemWeave target:** `engine/fefw-recomp/runtime` (production C++ engine, GPL-3.0-or-later)
**Companion documents:** `LICHII_ARCHITECTURE_RESEARCH.md`, `REA_REVERSE_ENGINEERING_FINDINGS.md`, `LICHII_PERFORMANCE_ADAPTATION_PLAN.md`

## 1. Method and limits

This comparison is source-to-source. No Lichii binary was built or run, no
EmblemWeave gameplay benchmark was run for this assignment, and no game data,
saves, shader caches or build directories were touched. EmblemWeave numbers
quoted here come from its existing `docs/CPU_PARALLELISM_RESULTS.md` and
`docs/CPU_PARALLELISM_AUDIT.md`, with the caveat in section 4.1.

EmblemWeave is being edited concurrently by FCC. `nvn_gpu_pool.cpp` changed at
17:53 during this reading, and `nvn_replay.cpp` / `nvn_render_prep.cpp` are
newer than the only local test binary. Line numbers are the ones observed at
read time and must be re-checked before any edit.

## 2. EmblemWeave architecture map

| # | Area | Where it lives | Threading |
| --- | --- | --- | --- |
| 1 | Recompiled game execution | `src/kernel/sched.cpp`, `src/kernel/thread.cpp`; six guest cores | Guest threads on host threads; Core 5 is the render thread |
| 2 | NVN replay and command processing | `src/gfx/nvn/nvn_replay.cpp` (`replay_begin_submit` 5732, `replay_lists` 5834, `replay_end_submit` 5897) | Render thread; `nvn_cmdq` worker for Vulkan entry points |
| 3 | Vulkan resource creation | `nvn_gpu.cpp`, `nvn_gpu_texture.cpp`, `nvn_pipeline.cpp` | Render thread, plus pool/prep workers |
| 4 | Descriptor set construction | `nvn_replay.cpp:2900-2945`, `nvn_render_prep.cpp` | Render thread by default; `render_prep` workers exist but each call waits |
| 5 | Pipeline creation and caching | `nvn_pipeline.cpp`, `nvn_shaders.cpp`, `nvn_prewarm.cpp`, `nvn_gpl.cpp` | Prewarm/GPL worker pools; background cache save |
| 6 | Texture processing | `nvn_texdata.cpp`, `nvn_texprep.cpp`, `nvn_gpu_texture.cpp` | `tex_prep` pool for whole-texture preparation; upload/untile serial |
| 7 | Buffer management | `nvn_gpu_pool.cpp`, `nvn_gpu.cpp` staging ring | Pool mirror with guarded pages; copy-back on CPU first touch |
| 8 | GPU submission | `nvn_gpu.cpp:120-180` `submit_batch`/`flush_batch`, `nvn_replay.cpp` `split_submit_ms` | Batched `vkQueueSubmit2`, optional mid-frame splits |
| 9 | CPU/GPU synchronization | `nvn_gpu.cpp:1156` `wait_serial`, `:454` `wait_serial_blocking`, timeline semaphores | Game threads wait; render thread may flush |
| 10 | Threading and task scheduling | `nvn_cmdq.cpp`, `nvn_texprep.cpp`, `nvn_prewarm.cpp`, `nvn_gpl.cpp`, `nvn_render_prep.cpp`, ASTC/pool helpers | Ad-hoc pools, no unified priority scheduler |
| 11 | Frame presentation | `nvn_present.cpp`, `nvn_gpu.cpp:1020` `take_batch_for_present` | Present thread / main thread |

Relevant switches (from `runtime/docs/CONFIG.md`):
- `[nvn] replay_threads` default auto (2 with ≥8 logical CPUs, else 1).
- `[nvn] replay_batch` default 16.
- `[nvn] render_prep` default **false**; `render_prep_threads` default hw/4, 1..4.
- `[nvn] prewarm_threads` default `max(1, min(6, cpus/4))`.
- `[nvn] tex_prep_threads` default hw/3, clamped 2..8.
- `[nvn] compute_out_lazy` default true.
- `[nvn] split_submit_ms`, `submit_batch`, `batch_ubo_updates`, `lean_barriers`.

## 3. Direct comparison

| Lichii mechanism | EmblemWeave equivalent | Relationship | Assessment |
| --- | --- | --- | --- |
| GPU front/back split at engine-method boundary (`gpu_split`) | `nvn_cmdq` splits at the **Vulkan-call** boundary (`vkCmd*`, `vkUpdateDescriptorSets`) | Partial overlap; different layer | Full method split is a large architectural change; the measured win in EmblemWeave's existing split is already substantial (section 4.1) |
| 8 MiB SPSC ring, batched publish, idle early publish, spin-then-sleep | 32 MiB SPSC ring, publish every `replay_batch` (16), spin 10-100 µs then `WaitOnAddress`, tail read only when full | Same design, different tuning | Only micro-tuning is novel; measure ring-full waits and publish latency before changing |
| Method-level register mirroring, copied multi-word payloads, drain sites | NVN replay owns decoded state; `nvn_cmdq` copies arguments into the ring; `build_set` caches per-set state | Different problem | Lichii solves guest-memory reuse across a thread boundary; EmblemWeave decodes on the render thread, so it is not exposed the same way |
| Write tracking by 256 B / 64 KiB granule and selective waits | `nvn_gpu_pool.cpp:785,971-1008` page-level `gpu_new`/`cpu_read`; `mem::ww` guards | Same goal, different granularity | EmblemWeave is finer-grained for compute-out; no need for Lichii's sets |
| `ShaderJobScheduler`: NeedNow/Soon/Background, CPU+driver pools, byte budget, 2 s aging, promotion, cancellation, exception isolation, `RunOrWait` | Prewarm/GPL/texprep each have their own pool and ordering; no shared priority, no promotion, no byte budget, no per-job cancellation | **Gap** | Most transferable scheduling design |
| Deferred native pipeline emission (SPIR-V + modules on worker) | Shaders are offline-translated SPIR-V (`nvn_shaders.cpp:4`); pipeline build is already on prewarm/GPL workers | Mostly **not applicable** | No runtime IR translation to move |
| Parallel texture unswizzle (2 MiB, native tiling, depth 1, block_depth 0, pitch not 4-byte aligned) | `tex_prep` prepares whole textures on workers; `tegra::untile` itself is serial on upload | **Gap, narrow** | Only large synchronous uploads in the eligible shapes |
| No buffer downloads at fences + read traps | `compute_out_lazy` guarded pages; copy-back on CPU read (`nvn_gpu_pool.cpp:1577-1582, 1758`) | **Already present** | Verify coverage and counters, do not re-port |
| Per-thread translation cache with generation | EmblemWeave has its own caches and locks; not the same device-memory map | Not directly comparable | Only relevant if a shared cache is found to be contended |
| Fence batching, one lock per cache, skip empty fences (rows 120, 123, 124) | Timeline semaphores, `submit_batch`, `wait_serial`; cache locks exist but were not audited here | Possible small win | Profile lock/fence cost first |
| Same-value state write suppression | `bind_is_same` (`nvn_replay.cpp:5089`) and `out_masks` (1939); raw state compare | **Already present** | Lichii's 60% duplicate rate is NVN2-macro-specific |
| UBO update batching | `batch_ubo_updates` (`nvn_replay.cpp:4957-5012`) | **Already present** | No action |
| Submission batching / present pacing | `submit_batch`, `split_submit_ms`, `take_batch_for_present` | **Already present** | No action |
| Macro memoization/JIT, QMD 3.0, MME64, GOB tiling | No NVN2 macro engine | **Not applicable** | Depends on Switch 2 guest ISA/hardware |

## 4. Overlap with FCC's current work

### 4.1 `nvn_cmdq` already is the “second GPU thread”

`nvn_cmdq.cpp` intercepts the Vulkan entry points the replay makes and performs
them on a worker (`worker_main`, line 123). The render thread waits only at the
end of the recording scope (`Scope::~Scope`, line 737; `do_sync`, 165), before
command-buffer end/submit/reset/allocation, and for SDKs that record through
their own entry points (`cmdq::Direct`, used at `nvn_replay.cpp:3220, 3268,
3551, 5619`).

The project's own configuration documentation records:
- issue #29, 16-thread AMD, Dagsion street: render work p50 **17.8 → 13.0 ms**,
  **55 → 77** presented frames/s;
- RTX 3080, 4K: p50 **16.8 → 13.6-14.2 ms**, frames over 20 ms **−25%**.

This is measured, shipped, and directly comparable in kind to Lichii's split.
Adding another GPU thread above it without evidence would risk exactly the
“second split” complexity Lichii warns about (back still 92% busy, 53% draw
setup).

The honest open question is different: `nvn_cmdq` defers **Vulkan calls**, not
NVN decode, descriptor construction, pipeline lookup or draw setup. Those stay
on the render thread. Lichii's split moves a larger share. Whether that share
is worth moving in EmblemWeave must be measured, not assumed.

### 4.2 `render_prep` is not currently a working parallel path

`bind_sets` (`nvn_replay.cpp:2900`) submits sets 1-3 to
`renderprep::build_set_async`, then immediately spins:

```cpp
if (renderprep::build_set_async(work_items[work_count])) {
    // Submitted successfully, will wait later
}
...
while (work_items[i].result == VK_NULL_HANDLE) {
    std::this_thread::yield();
}
```

`build_set_async` itself waits internally (per the comment at line 2938), so the
caller gains no overlap. The switch is default **off** because of Vulkan driver
thread-safety issues (AMD) per `CONFIG.md:69`. This is FCC's area. Any proposal
to add a Lichii-style descriptor scheduler here would duplicate unfinished work
and would not fix the actual blocker (descriptor-pool/allocator thread safety
and driver behaviour). **Coordinate; do not build a second mechanism.**

Provenance caveat: the only local `nxrt_gfx_tests.exe` was built on
2026-11-20 15:21, before `nvn_render_prep.cpp` (2026-11-21 22:50) and the current
`nvn_replay.cpp` (2026-11-21 23:22). Its PDB contains no `nvn_render_prep`
object and its image contains no `render_prep` string. That build cannot
confirm or refute the current descriptor code; it only shows the path was not
in that binary. The source was read directly for this section.

### 4.3 Pipeline preparation already has workers

`prewarm` builds recorded pipelines on `prewarm_threads`; GPL has
`gpl_threads` and one upgrade thread; both use a clamp of 1..2 for part workers
plus a burst. What is missing relative to Lichii is not “a worker pool” but:
- one ordering across prewarm, GPL, texture prep and boot precache;
- byte-budget admission;
- `NeedNow` promotion when a draw waits;
- cancellation of stale work;
- per-job exception isolation and observable queue-wait statistics.

That is a scheduling layer, not a new compiler.

### 4.4 Texture preparation

`tex_prep` already prepares whole textures on background workers with a 2 ms
wait before stand-ins (`nvn_texprep.cpp:115`), a `tex_prep_mb` budget (461) and
a `tex_prep_min_kb` floor (462). The remaining serial cost is the actual
untile/upload for textures that were not prepared or arrive late. Lichii's
parallel unswizzle only helps a narrow shape and has no frame-time evidence,
so it is a last-mile optimization, not a replacement for `tex_prep`.

### 4.5 Memory and readback

`compute_out_lazy` (`nvn_gpu_pool.cpp:1577-1582`) already keeps compute results
in the copy, marks pages newer-in-copy (`gpu_new`), guards them with
`PAGE_NOACCESS`, and copies back when the CPU reads them (`cpu_read`, 2018).
This is the same idea as Lichii's read traps, at page granularity. The
difference is scope: Lichii traps any GPU-written guest page after disabling
downloads; EmblemWeave traps compute-out copies. Before adding anything, audit
whether non-compute GPU writes still trigger eager downloads.

### 4.6 The 9.18 ms baseline must not be read as render work

`CPU_PARALLELISM_RESULTS.md` reports 9.18 ms CPU frame time, of which
`Present/Limiter` is ~7.0 ms and “mostly limiter spin” (lines 131-137). Two
problems:
- 9.18 ms at 108.9 FPS is the real frame interval, but the stated 180 FPS
  target is a 5.55 ms budget; 2 ms work + 7 ms spin does not fit that budget.
- The breakdown is self-reported by the engine, not independently traced.

So the correct conclusion is **not** “there are 7 ms of free render time”.
It is “the dominant cost is a wait whose attribution is unverified”. Split it
into: wait for GPU serial, wait for present/acquire, limiter sleep, and limiter
spin, before using this baseline to justify any architectural change.

## 5. Architectural differences that block a direct port

1. **Guest API level.** Lichii splits NVN2 method execution; EmblemWeave splits
   Vulkan recording. Moving EmblemWeave up a level means decoding NVN commands,
   owning all register state and copied payloads on a second thread, and
   reproducing every wait/drain site. Lichii needed 3,554 lines and 82 call
   sites for that.
2. **Shader model.** Lichii translates guest shaders at runtime; EmblemWeave
   consumes offline SPIR-V. There is no IR-translation stage to defer.
3. **Guest core count and workload.** EmblemWeave runs six guest cores with one
   render core; adding another host render thread competes with guest cores on
   the same CPU. Lichii's own note says a 4-thread CPU has no spare core.
4. **One title vs general emulator.** Lichii's measurements are one game on one
   machine. EmblemWeave targets one game, which makes measurement easier but
   also makes “it helped Mario Kart World” weak evidence.
5. **Driver thread safety.** EmblemWeave has already found descriptor building
   unsafe on some drivers; Lichii handles this by forcing one driver shader
   thread on broken drivers. Any worker-based design must have the same
   escape hatch.

## 6. Missing capabilities worth considering

Ranked by evidence and fit:

1. **A single priority/budget-aware pipeline job scheduler** with promotion,
   cancellation, exception isolation and queue-wait statistics. Lichii's is
   tested in 28 cases. EmblemWeave has pools but no shared policy.
2. **Explicit `NeedNow` promotion for a draw that blocks on a pipeline**, so
   background prewarm cannot delay a visible draw. This is small and directly
   measurable.
3. **Cancellation and exception isolation** for prewarm/GPL jobs. Today a stale
   prewarm can occupy a worker and a failed job's handling is pool-specific.
4. **Parallel unswizzle for eligible large uploads**, behind a serial-reference
   verification switch. Narrow benefit; prototype only after measuring the
   synchronous upload share.
5. **Ring micro-tuning** (idle early publish, cache-line separation of consumer
   position) only if `nvn_cmdq` statistics show ring-full waits or publish
   latency. Currently these are speculative.

## 7. What not to port

- The method-level GPU split as a whole (architectural cost, existing
  Vulkan-level split, no measured EmblemWeave evidence).
- NVN2 macro memoization/JIT, QMD 3.0, MME64, tensor stand-ins, GOB-specific
  tiling: guest-ISA/hardware-specific.
- Lichii's runtime shader translation and deferred IR emission: EmblemWeave has
  no runtime IR translation.
- A second descriptor-building mechanism: `render_prep` already exists and is
  unfinished; fix or remove it, do not duplicate it.
- Blanket removal of GPU→CPU downloads: EmblemWeave already has guarded lazy
  compute-out; any widening needs a read-trap proof, not a blind switch.
