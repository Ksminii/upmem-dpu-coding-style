# DPU Kernel Conventions

Evidence tags name the source repo under github.com/upmem.

## §1 Include order

SDK headers in `<>` first, alphabetized; project headers in `""` after; the
shared common header last:

```c
/* upvc:dpu/src/task.c */
#include <alloc.h>
#include <barrier.h>
#include <defs.h>
#include <mram.h>
#include <mutex.h>
#include <perfcounter.h>

#include "debug.h"
#include "common.h"
```

Common SDK headers: `<defs.h>` (`me()`), `<mram.h>`, `<alloc.h>`, `<barrier.h>`,
`<mutex.h>`, `<perfcounter.h>`, `<attributes.h>` (`__host` etc.),
`<built_ins.h>`, `<seqread.h>`, `<stdio.h>` (DPU printf).

## §2 main() boot pattern

Every tasklet runs `int main()`. Fetch the tasklet id immediately; tasklet 0
does one-time init; a barrier separates init from work, and a second barrier
lets tasklet 0 finalize (e.g. timing) after everyone finishes. `return 0`
(a nonzero return is a status the host can read).

```c
BARRIER_INIT(init_barrier, NR_TASKLETS);

int main(void) {
    uint32_t tasklet_id = me();     /* sysname_t in upvc */
    if (tasklet_id == 0) {
        mem_reset();
        perfcounter_config(COUNT_CYCLES, true);
        request_pool_init();
        result_pool_init();
    }
    barrier_wait(&init_barrier);
    run_work(tasklet_id);
    barrier_wait(&init_barrier);    /* before tasklet 0 reads final time */
    return 0;
}
```

`NR_TASKLETS` is never defined in source — the build injects it (build.md §2)
and the runtime spawns that many tasklets from the same define.

## §3 Memory placement attributes

| Attribute | On | Meaning |
|---|---|---|
| `__host` | WRAM globals | host-visible scalars / param structs / result arrays |
| `__mram_noinit` | MRAM globals | bulk buffers the host fills (skips zero-init) |
| `__dma_aligned` | every WRAM buffer that is an `mram_read`/`mram_write` target — **stack variables included** | `__attribute__((aligned(8)))` |
| `__mram_ptr` | pointers into MRAM | required in DMA casts; direct dereference allowed when convenience beats speed |

```c
__host struct dpu_params tasklet_params[NR_TASKLETS];        /* keccakf */
__mram_noinit uint8_t DPU_BUFFER[BUFFER_SIZE];               /* dpu_demo */
__dma_aligned uint8_t DPU_CACHES[NR_TASKLETS][BLOCK_SIZE];   /* dpu_demo */

/* stack DMA target still needs the attribute (UPIS:dpu/src/main.c) */
__dma_aligned response_t response = { ... };
mram_write(&response, &DPU_RESPONSES_VAR[response_id], sizeof(response));
```

Direct `__mram_ptr` dereference example: the CIGAR reverse in
usecase_dpu_alignment indexes `__mram_ptr uint8_t *mram` element-wise.

## §4 WRAM allocation

- Default: **static file-scope arrays indexed by tasklet id** —
  `__dma_aligned static dout_t global_dout[NR_TASKLETS];` (upvc). For stricter
  alignment, wrap in an aligned struct:
  `struct WramAligned64 { uint8_t buffer[64 * NR_GROUPS]; } __attribute__((aligned(64)));`
  (usecase_dpu_alignment).
- `mem_alloc()` is fine for per-kernel caches — but tasklet 0 must call
  `mem_reset()` first (also required before `seqread_alloc()`):
  `uint32_t *cache = (uint32_t *)mem_alloc(BLOCK_LENGTH * sizeof(uint32_t));` (dpu_olap).
- **Nothing big on the stack.** UPMEM's own comment: "As the structure is quite
  big, let's have it in the heap and not in the stack" (upvc:dpu/src/task.c).
  Per-tasklet stack size is an explicit build knob `-DSTACK_SIZE_DEFAULT=…`
  (build.md §4), not a hope.

## §5 MRAM access

- Always staged through a fixed-size `__dma_aligned` WRAM cache in a chunked
  loop. **Single DMA ≤ 2048 B.** Chunk sizes are small powers of two
  (`#define BLOCK_SIZE (256)` dpu_demo; tandem_demo uses exactly 2048).
- Handle the tail (`size % CHUNK`) as a separate transfer with the leftover
  byte count (tandem_demo).
- Batch several records per DMA to amortize fixed cost — upvc fetches
  `NB_REF_PER_READ (=4)` records per `mram_read`.
- For sequential byte-stream parsing use the SDK `seqread` reader instead of a
  hand-rolled cache (UPIS:dpu/src/decoder.c):
  `seqread_init(seqread_alloc(), mram_addr, &reader)`,
  `seqread_get(ptr, sizeof(uint8_t), &reader)`, `seqread_seek`, `seqread_tell`.
- For random access / patterned access, wrap raw DMA in a small POD accessor
  struct + `create_*`/`*_next`/`*_get`/`*_flush` functions, one per pattern —
  usecase_dpu_alignment ships a family (`mram_sequential_reader_64.h`,
  `mram_reverse_sequential_reader_64.h`, `mram_bit_array_32.h`,
  `mram_2bits_array_64.h`, `dna_reader.h`), each taking a group id so N
  readers share one aligned WRAM buffer (`buffer + 64 * id`).

## §6 Synchronization

- Static init macros at file scope only: `BARRIER_INIT(name, NR_TASKLETS);`,
  `MUTEX_INIT(name);`. No dynamic barrier/mutex creation.
- Keep critical sections minimal — grab an index under the lock, do the MRAM
  write outside it:

  ```c
  mutex_lock(job_mutex);
  id = jobid++;
  mutex_unlock(job_mutex);
  ...
  mutex_lock(scores_mutex);
  output.scores[slot] = tmp_s;      /* WRAM write only */
  mutex_unlock(scores_mutex);
  ```

- Barriers cannot be indexed; if you need per-group barriers, enumerate them by
  hand (`BARRIER_INIT(barrier1, 4); … barrier6` selected by an if-chain —
  usecase_dpu_alignment:libnwdpu/dpu/nw_common.h).
- No `<sem.h>` on the DPU in any official repo — semaphores appear host-side
  only (POSIX, in pipeline code).

## §7 Errors and printf

- Fatal on the DPU = warn then halt, so the host observes a fault:
  `printf("WARNING! ...\n"); halt();`
- `assert()` and `_Static_assert` freely:
  `_Static_assert(__builtin_popcount(NR_SEGMENTS_PER_MRAM) == 1, "...");` (UPIS).
- Heavy DMA validation lives in compiled-out macros kept at call sites
  unconditionally — `ASSERT_DMA_ADDR(addr)` / `ASSERT_DMA_LEN(len)` check 8-byte
  alignment and WRAM/MRAM bounds only when a debug define is set
  (upvc:dpu/inc/debug.h).
- printf needs a buffer declared at file scope: `STDOUT_BUFFER_INIT(128 * 1024);`
  (upvc, unconditional) or guarded `#ifdef TRACE STDOUT_BUFFER_INIT(256); #endif`
  (UPIS). Reserve printf for warnings-before-halt and compiled-out trace macros;
  per-tasklet debug prints are prefixed with the tasklet id:
  `printf("[%02d] Checksum = 0x%08x\n", tasklet_id, ...);` (dpu_demo).

## §8 perfcounter

- Configured exactly once, by tasklet 0: `perfcounter_config(COUNT_CYCLES, true);`
  read by anyone with `perfcounter_get()`.
- The counter is 36-bit — long kernels must detect wrap and accumulate:

  ```c
  /* keccakf:bench_keccakf_dpu.c */
  if (current < *last)
      accumulate += 1ULL << 36;
  ```

  (dpu_demo truncates deliberately instead: `(uint32_t)perfcounter_get()` with a
  "keep the 32-bit LSB" comment — fine only for short kernels.)
- The host reports the **max** cycles across tasklets/DPUs as the kernel time.
- Guard perf plumbing behind a compile flag (`#if ENABLE_PERF … output.perf_counter
  = perfcounter_get();` — dpu_olap).
