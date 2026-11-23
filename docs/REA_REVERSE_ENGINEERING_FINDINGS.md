# X:\REA reverse-engineering findings

**Date:** 2026-11-22
**Binary analysed:** `nxrt_gfx_tests.exe` + `nxrt_gfx_tests.pdb` from
`EmblemWeave/build/launcher-ci-off`, copied read-only to
`X:\REA\work\ghidra-input`
**Build timestamp (PE header):** 2026-11-20 15:21:02
**Source snapshot at analysis:** 2026-11-22 (concurrently edited by FCC)
**Lichii binary:** none. Lichii was not built (it would contend with FCC's
active builds) and no third-party binary was downloaded.

## 1. X:\REA inventory

| Path | Version / role | Used here |
| --- | --- | --- |
| `X:\REA\ghidra` | Ghidra 12.1 | Yes, headless import attempts (section 3) |
| `X:\REA\jdk21` | Temurin OpenJDK 21.0.13+9 | Yes, Ghidra runtime |
| `X:\REA\rea` + `rea.sh` / `rea.cmd` | REA `rea-agents` 6.7.0, contained under X: | No: its JavaScript/Electron/native-app providers do not apply to a C++ MSVC binary; Ghidra is the relevant provider |
| `X:\REA\USAGE.md` | Containment notes for REA | Yes, to confirm wrappers and caches stay on X: |
| `X:\REA\visualvm` | JVM profiling | No: the target is native |
| `X:\REA\jdk17`, `android-sdk`, `apk`, `morphe`, `Shizuku` | Android/JVM tooling | No |
| Visual Studio 2026 (18) Community | MSVC 14.52.37025, `dumpbin` 14.52.37029 | Yes, `dumpbin` disassembly |
| DIA SDK (VS 18) | `msdia140.dll`, `dia2.h`, `diaguids.lib` | Yes, PDB symbol extraction |
| CMake 4.4.4, Ninja 1.13.2 | Build tools | Not needed: no rebuild |
| LLVM in VS | `clang-format`, `clang-tidy` only | No `llvm-pdbutil` / `llvm-objdump` |

The useful discovery was the **DIA SDK already installed with Visual Studio**.
It reads the exact PDB without registering a COM server, which made
symbol-level analysis possible after Ghidra's analyzers failed to converge.

## 2. Binary provenance and validity

This is the most important limitation of the whole exercise.

| Item | Timestamp |
| --- | --- |
| PE `Time/Date stamp` | 2026-11-20 15:21:02 |
| `nxrt_gfx_tests.pdb` (original) | 2026-11-20 15:21:04 |
| `nvn_cmdq.cpp` | 2026-11-20 04:54:13 (before the build) |
| `nvn_texprep.cpp` | 2026-11-16 15:25:10 (before) |
| `nvn_prewarm.cpp` | 2026-11-18 22:42:04 (before) |
| `nvn_gpl.cpp` | 2026-11-19 01:25:37 (before) |
| `nvn_replay.cpp` | 2026-11-21 23:22:40 (**after**) |
| `nvn_render_prep.cpp` | 2026-11-21 22:50:54 (**after**) |
| `nvn_pipeline.cpp` | 2026-11-21 23:04:37 (**after**) |
| `nvn_gpu_pool.cpp` | 2026-11-22 17:53:45 (**during this analysis**) |

Consequences:
- The binary can validate `nvn_cmdq`, `nvn_texprep`, `nvn_prewarm`, `nvn_gpl`
  and `wait_serial` as they existed on 2026-11-20.
- It **cannot** validate the current `nvn_replay.cpp`, `nvn_render_prep.cpp`,
  `nvn_pipeline.cpp` or `nvn_gpu_pool.cpp`.
- The PDB contains compilands `nvn_replay`, `nvn_cmdq`, `nvn_texprep`,
  `nvn_prewarm`, but **no `nvn_render_prep` compiland**, and the image contains
  no `render_prep` string. The descriptor path was simply not in that build.
- `nvn_gpu_pool.cpp` changed at 17:53 while this work was in progress. FCC is
  actively editing, so every line number in the companion documents is a
  snapshot and must be re-checked before any edit.

## 3. Ghidra attempts and the targeted fallback

Ghidra 12.1 headless was run four times against the copied EXE + PDB:

1. Full analysis with a 1,800 s per-file timeout. The PDB main-symbol pass
   succeeded, then the PDB function-internals pass entered a decompiler loop on
   a large data blob (`140e83d50`) and made no completion progress for >20 min
   at ~2 GB heap. Killed.
2. Retry with a pre-script disabling `Decompiler Parameter ID`, `Decompiler
   Switch Analysis`, `Stack` and a guessed `Constant Propagation` option.
   `jstack` showed the `" Constant Reference Analyzer"` (leading space; the
   class is `ConstantPropagationAnalyzer`) still running whole-program
   constant propagation. Killed.
3. Retry with the exact analyzer name disabled. `jstack` then showed
   `FindNoReturnFunctionsAnalyzer` doing a whole-program reference scan. Killed
   after it exceeded the useful time budget.
4. Final attempt with `-noanalysis`: import completed and the project was
   saved, but without analysis the PDB function symbols are not applied, so the
   export script wrote 0 functions (`nxrt5`, `X:\REA\work\ghidra-project`).
   Attempts 1-3 saved no usable project because they were killed mid-analysis.

This is a tooling limitation, not a claim about the binary. Ghidra was not
allowed to run for hours on a 25 MB MSVC test executable while FCC was using
the machine.

A **targeted** decompile did succeed afterwards: the saved `nxrt5` project was
processed with `-noanalysis` and `DecompileRange.java`, which disassembles only
`0x14095D7A0-0x14095D9E0`, creates one function and decompiles it. Ghidra
returned `FUN_14095d7a0`; its output confirms the publish, wake, spin and
`SwitchToThread` loop described below. This is a range decompile with no
symbols or types, not a full-program analysis.

## 4. DIA + dumpbin method

Two small tools were built under `X:\REA\work` (not in EmblemWeave):

- `X:\REA\work\dia\pdb_symbols.cpp`: loads `msdia140.dll` with
  `NoRegCoCreate`, opens the PDB, enumerates public symbols, writes
  `RVA`, `size`, `undecorated name`.
- `X:\REA\work\dia\extract.py`: parses `dumpbin -disasm:nobytes` output,
  slices exact `[RVA, RVA+size)` ranges, and annotates direct `call` targets
  with PDB names.

Commands:
```text
dumpbin -disasm:nobytes nxrt_gfx_tests.exe > disasm.txt   (4,526,859 lines)
pdb_symbols.exe nxrt_gfx_tests.pdb symbols.txt             (86 matching symbols)
python extract.py
ghidra analyzeHeadless ... nxrt5 -process ... -noanalysis -postScript DecompileRange.java
```

Artifacts: `X:\REA\work\dia\symbols.txt`, `disasm.txt`, `func_*.asm`;
`X:\REA\work\ghidra-decompile.log`.

Only public symbols are available this way. Internal/static functions such as
the cmdq worker are not individually named; their code is covered by the
`worker entry region` range derived from the `_beginthreadex` argument in
`cmdq::install`.

## 5. Symbol and instruction findings

All addresses are VAs (image base `0x140000000`). Instruction counts are from
`dumpbin` within the symbol's exact size.

| Symbol | VA | Size (B) | Insns | Calls | Notable |
| --- | --- | ---: | ---: | ---: | --- |
| `cmdq::install` | `0x140956960` | 3,577 | 684 | 25 | `VirtualAlloc` at `0x140956AE6`, `_beginthreadex` at `0x14095761D`, `config_get_int` ×2 |
| `cmdq::Scope::~Scope` | `0x14095D4B0` | 737 | 164 | 7 | calls `do_sync` at `0x14095D7A0`, then clears TLS flag, stores `active=0`, timestamps |
| `cmdq::sync` | `0x14095DA50` | 43 | 8 | 0 | tail-jumps to `do_sync` when the queue and TLS flag are set |
| `cmdq` producer wait (`do_sync`) | `0x14095D7A0` | 576* | 159 | 13 | PAUSE at `0x14095D8E0`, 0xF9F iterations, `WakeByAddressSingle` ×2, `SwitchToThread` |
| `cmdq` worker region | `0x14095CE70` | 1,248* | 312 | 25 | `rdtsc` spin + PAUSE, `WaitOnAddress` at `0x14095D16A`, `SetThreadPriority` at exit |
| `cmdq::shutdown` | `0x14095D350` | 204 | 50 | 7 | `WakeByAddressSingle` |
| `gpu::replay_begin_submit` | `0x140960230` | 11,833 | 2,162 | 111 | largest replay function |
| `gpu::replay_end_submit` | `0x14096CC70` | 1,238 | 284 | 20 | |
| `gpu::wait_serial` | `0x14069BAF0` | 3,202 | 767 | 71 | one fence instruction |
| `texprep::take` | `0x1407A93A0` | 4,243 | 1,053 | 91 | |
| `prewarm_init` | `0x140754D20` | 6,045 | 1,410 | 87 | 14 `lock`-prefixed instructions |
| `gpl_prepare` | `0x14070D470` | 8,592 | 2,046 | 150 | |

\* Range, not a symbol size: `do_sync` is the gap before `Direct::Direct`;
the worker region is `_beginthreadex`'s argument through `shutdown`.

### 5.1 `cmdq::install`: the existing second thread is real

The disassembly shows, in order:
- `config_get_int` for `replay_threads` and `replay_batch`;
- `_Thrd_hardware_concurrency`;
- a 32 MiB `VirtualAlloc` (`kRingBytes = 32 << 20`);
- construction of the atomics and the queue;
- `_beginthreadex` with entry `0x14095CE70`;
- the log line naming `nvn-rec`.

There is no `render_prep` call, no `gl`/`vk` wrapper for descriptor sets, and
no evidence of a second worker. This matches `nvn_cmdq.cpp` as it stood before
the build.

### 5.2 The producer never sleeps; the worker does

`do_sync` (`0x14095D7A0`) is the wait at `Scope::~Scope` and at explicit `sync`:
- publishes the remaining records;
- if the worker is marked sleeping, `WakeByAddressSingle`;
- checks the consumed position;
- spins up to **0xF9F = 3,999** iterations, each a `PAUSE` plus an acquire load;
- after the spin, wakes the worker again and calls `SwitchToThread`, then
  repeats.

It contains no `WaitOnAddress`. The worker region does:
- an `rdtsc`-based short spin with `PAUSE`;
- sets the sleeping flag;
- rechecks the published position;
- calls `WaitOnAddress` (5 ms timeout), then clears the flag and processes.

Ghidra's decompile of the producer wait shows the loop directly:

```c
if (tail < published) {
  iVar3 = 0;
  do {
    if (3999 < iVar3) {
      if (active) WakeByAddressSingle(&pub);
      SwitchToThread();
    }
    iVar3 = iVar3 + 1;
    tail = load_acquire(&consumed);
  } while (tail < published);
}
```

That is a spin of 4,000 iterations, then `WakeByAddressSingle` plus
`SwitchToThread` on every subsequent iteration until the worker catches up.

So the source comment's “10-100 µs spin then `WaitOnAddress`” is accurate for
the **worker**, not for the producer. The producer yields with
`SwitchToThread`; it never blocks on the address. This is a useful distinction
that source reading alone blurred.

### 5.3 `Scope` is cheap; the wait is the cost

`Scope::Scope` is 144 B and only sets TLS/active and wakes a sleeping worker.
`Scope::~Scope` is 737 B and calls `do_sync`, then does the stats timestamp.
`sync` is 43 B and tail-jumps to `do_sync`. The overhead is therefore not in
scope entry/exit bookkeeping; it is in the wait path when the worker has not
caught up.

### 5.4 Pipeline and texture functions are compiled in

`prewarm_init` (6,045 B, 14 `lock` prefixes), `prewarm_take` (1,970 B),
`gpl_prepare` (8,592 B), `gpl_link` (3,876 B) and `texprep::take` (4,243 B) are
present with substantial bodies. `prewarm_init`'s lock prefixes are consistent
with mutex/once initialization, not a hot per-draw path. There is no symbol for
a Lichii-style `ShaderJobScheduler`, `UnswizzleWorker` or priority queue; those
concepts are not implemented in this build.

### 5.5 `wait_serial` is not a spin

`wait_serial` is 3,202 B with 71 calls and one fence. It is a blocking
timeline-semaphore wait (`vkWaitSemaphores`), not a busy loop. The fence is
consistent with a one-time atomic/order sequence inside the function; the
function's cost is the wait itself, not instruction count.

## 6. Call graph (direct calls only)

```text
cmdq::install
  ├─ config_get_int("replay_threads"), config_get_int("replay_batch")
  ├─ _Thrd_hardware_concurrency
  ├─ VirtualAlloc(32 MiB)
  ├─ _beginthreadex(worker entry 0x14095CE70)
  └─ logf

cmdq::Scope::~Scope
  ├─ do_sync (0x14095D7A0)
  │    ├─ atomic publish/load
  │    ├─ WakeByAddressSingle
  │    ├─ PAUSE × up to 3,999
  │    └─ SwitchToThread
  ├─ atomic active=0
  └─ QueryPerformanceFrequency/Counter

cmdq worker region
  ├─ log_set_thread_name, diag_register_thread
  ├─ config_get_str
  ├─ PAUSE / rdtsc spin
  ├─ WaitOnAddress(5 ms)
  ├─ command dispatch through a function pointer
  └─ diag_unregister_thread, SetThreadPriority
```

The call graph confirms the architectural claim in
`LICHII_EMBLEMWEAVE_COMPARISON.md`: EmblemWeave already ships a
Vulkan-call-level producer/consumer thread with a 32 MiB ring and an adaptive
producer wait. It does not have a method-level split.

## 7. What source inspection alone did not establish

1. **The current descriptor code is not in the only local binary.** Reading
   `nvn_replay.cpp` suggests `renderprep::build_set_async` is on the draw path.
   The PDB shows no `nvn_render_prep` compiland and the image has no
   `render_prep` string. The binary is a day older than that source.
2. **The producer's wait is `SwitchToThread`, not `WaitOnAddress`.** The source
   comment reads as if both sides sleep. The worker sleeps; the producer spins
   4,000 times and then yields on every iteration. Ghidra's decompile of
   `0x14095D7A0` confirms this shape directly.
3. **The installed build uses `_beginthreadex`**, i.e. the MSVC `std::thread`
   implementation, and allocates the ring with one 32 MiB `VirtualAlloc`, not a
   pool or a growable buffer.
4. **`prewarm_init` is lock-heavy at initialization, not at steady state.**
   Its 14 lock prefixes belong to once/mutex setup; the per-draw path is
   elsewhere.
5. **`wait_serial` is a blocking wait**, so a “CPU spin” hypothesis for the
   7 ms present/limiter figure is not supported by its code shape.
6. **No Lichii mechanism is present.** No `ShaderJobScheduler`,
   `UnswizzleWorker`, `GpuSplit`, `build_set_async` or `render_prep` symbols
   exist in the PDB. This is expected, but it rules out “it is already there”
   claims for those names.

## 8. Limits

- Public symbols only: static/local functions and inlined code are not named.
- `dumpbin` gives assembly, not decompiled C++; types and inlining come from
  the source, not the binary.
- The EXE is a **test harness** (`CMakeFiles/nxrt_gfx_tests.dir/tests/...`),
  not the launcher or game runtime; its code layout is representative but its
  execution is not the game.
- No Lichii binary was built or inspected. Lichii findings are source- and
  document-based (see `LICHII_ARCHITECTURE_RESEARCH.md`).
- No dynamic profiling, ETW, GPU capture or gameplay run was performed.
- The binary predates several current source files; see section 2 before using
  any of these findings as evidence about today's code.
