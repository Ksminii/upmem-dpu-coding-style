---
name: upmem-dpu-coding-style
description: Use when writing, reviewing, optimizing, or building UPMEM PIM code — DPU kernels (dpu-upmem-dpurte-clang, tasklets, WRAM/MRAM, mram_read/mram_write, barriers, mem_alloc), host code on the dpu.h API (dpu_alloc, DPU_ASSERT, dpu_push_xfer, DPU_FOREACH, async callbacks), Makefiles/CMake that produce DPU binaries, or when a DPU kernel is slow and needs performance work (tasklet count, DMA sizing, multiply/float avoidance).
---

# UPMEM DPU Coding Style

Conventions distilled from the official UPMEM org repos (github.com/upmem, read 2026-08):
`dpu_demo`, `usecase_upvc`, `usecase_UPIS`, `usecase_dpu_alignment`, `dpu_olap`,
`usecase_keccakf`, `tandem_demo`. The checklist below is the working layer; each
section links to a reference file with code snippets, the reasoning, and source
evidence. Load a reference file only when applying a rule you don't already know
by shape.

## Checklist

### Interface (host↔DPU) — [references/interface.md](references/interface.md)
- One plain-C shared header: `<stdint.h>`, structs, capacity `#define`s — nothing else.
- Every transferred object: typedef + symbol-name macro; host addresses it via `XSTR()`.
- Every transferred struct 8-byte compliant (`aligned(8)` or explicit padding).
- Per-tasklet result arrays sized `[NR_TASKLETS]` in the shared header.
- Param struct small; bulk data as a separate transfer.
- `_Static_assert` that fixed MRAM regions fit.
- Variable-length items: bump layout + offset/length table in the metadata struct.

### Kernel — [references/kernel.md](references/kernel.md)
- Include order: SDK `<>` alphabetized, then project `""`, shared header last.
- `main()`: `me()` first; tasklet 0 does `mem_reset()` + init; `barrier_wait` before and after work.
- `__host` on host-visible WRAM, `__mram_noinit` on bulk MRAM, `__mram_ptr` on MRAM pointers.
- `__dma_aligned` on EVERY WRAM DMA target — stack variables included.
- WRAM: static per-tasklet arrays by default; `mem_alloc` only after tasklet-0 `mem_reset()`.
- No big stack objects; stack size set explicitly via `-DSTACK_SIZE_DEFAULT`.
- MRAM access chunked through a fixed WRAM cache; single DMA ≤ 2048 B; handle the tail.
- Sync objects via `BARRIER_INIT`/`MUTEX_INIT` at file scope; minimal critical sections.
- Fatal error = `printf` + `halt()`; heavy DMA checks in compiled-out macros left at call sites.
- perfcounter: config once by tasklet 0; handle 36-bit wrap; host reports max.

### Work split — [references/work-split.md](references/work-split.md)
- Uniform work → static block-cyclic ("rake") by tasklet id.
- Uneven work → mutex-guarded shared cursor (dynamic queue).
- Very uneven work (per-item cost spread ≳10×, dynamic queue alone leaves DPUs idle) → host-side cost model + least-loaded assignment before launch, dynamic queue on top.

### PIM-friendly implementation — [references/pim-friendly.md](references/pim-friendly.md)
- No multiply/divide in hot loops: power-of-two sizes + shifts, `*_LOG2` companion macros.
- Stay 32-bit: split 64-bit ops (e.g. two 32-bit popcounts); 32-bit loop indices.
- No floating point on the DPU — integer/fixed-point, or move that part to the host.
- Tasklets accumulate private partials; host (or tasklet 0) reduces. No shared accumulator under a mutex.
- MRAM persists across launches: load once, relaunch; batch + async transfers overlapped with execution.
- Size DMA to amortize fixed cost: batch small records, powers of two up to 2048 B.
- ≥11 tasklets saturate the pipeline (PrIM); trade deliberately against per-tasklet WRAM.
- Assume compute-bound; cut instruction count before restructuring DMA — confirm with perfcounter.
- Shard with zero inter-DPU dependencies; broadcast small shared state; merge on the host.
- Respect the 24 KB IRAM budget: no aggressive inlining/unrolling without a measured win.
- Balance across DPUs first (slowest DPU gates the rank), tasklets second.

### Host — [references/host.md](references/host.md)
- Every `dpu_*` call wrapped in `DPU_ASSERT`.
- Same data → `dpu_broadcast_to`; per-DPU → `dpu_prepare_xfer` loop + one `dpu_push_xfer` (max size, dummy buffers for idle DPUs).
- Variable-size readback two-phase: count symbol first, then max-sized gather.
- Async chain: xfers `DPU_XFER_ASYNC` → `dpu_launch(DPU_ASYNCHRONOUS)` → `dpu_callback(ASYNC|NONBLOCKING|SINGLE_CALL)` → one `dpu_sync`.
- Rank = refill/scheduling unit; callbacks index per-rank slots (no shared state).
- Loop vars `each_dpu`/`each_rank`; counts `nr_*`; dump DPU logs on fault.

### Build — [references/build.md](references/build.md)
- Kernel: `dpu-upmem-dpurte-clang`, one-shot compile+link. Host: `dpu-pkg-config --cflags --libs dpu`.
- `-DNR_TASKLETS` fed to BOTH compilers from ONE build variable.
- Config stamp file so `-D` changes force rebuild of both sides.
- DPU `-O2`, host `-O3`, `-Wall -Wextra -Werror -g`.

### Naming / format — [references/naming.md](references/naming.md)
- `snake_case`; `SCREAMING_SNAKE` macros with unit comments; types `_t`; DPU symbols short lowercase nouns.
- clang-format enforced (UPMEM: Webkit base, 130 cols, `ForEachMacros: DPU_FOREACH`).
- Debug/stats = compiled-out macro pairs, call sites unconditional.
- Kernel file tops with a prose block: parallelization strategy + MRAM map.

## Common mistakes

| Mistake | Consequence | Rule |
|---|---|---|
| WRAM DMA target not `__dma_aligned` | corrupt transfers / faults | kernel.md §3 |
| Transferred struct not 8B multiple | xfer error or silent misalignment | interface.md §2 |
| `NR_TASKLETS` differs between host and kernel builds | fault at launch, only at some T | build.md §2–§3 |
| Single DMA > 2048 B | fault | kernel.md §5 |
| `mem_alloc` without tasklet-0 `mem_reset()` | heap corruption across launches | kernel.md §4 |
| perfcounter read without wrap handling | negative/absurd timings on long kernels | kernel.md §8 |
| Per-DPU `dpu_push_xfer` sizes differ | must push max size for all; short buffers overrun | host.md §3 |
| float/double or `/`,`%` in a kernel hot loop | software emulation, order-of-magnitude slowdown | pim-friendly.md §1, §3 |
| Few tasklets with no WRAM reason | idle pipeline cycles (needs ~11 to saturate) | pim-friendly.md §7 |
| Re-transferring resident data every launch | wasted host↔DPU bandwidth; MRAM persists | pim-friendly.md §5 |
