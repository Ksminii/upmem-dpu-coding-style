# Host-Side Conventions

Evidence tags name the source repo under github.com/upmem.

## §1 Lifecycle and error handling

Canonical sequence, every SDK call wrapped in `DPU_ASSERT` (abort-on-error, no
recovery path — this is the official stance, not laziness):

```c
/* dpu_demo:checksum/host/host.c */
struct dpu_set_t dpu_set, dpu;
uint32_t each_dpu;
DPU_ASSERT(dpu_alloc(NR_DPUS, NULL, &dpu_set));   /* or DPU_ALLOCATE_ALL */
DPU_ASSERT(dpu_load(dpu_set, DPU_BINARY, NULL));
DPU_ASSERT(dpu_get_nr_dpus(dpu_set, &nr_of_dpus));
/* copy in → launch → copy out */
DPU_ASSERT(dpu_launch(dpu_set, DPU_SYNCHRONOUS));
DPU_FOREACH (dpu_set, dpu) { DPU_ASSERT(dpu_log_read(dpu, stdout)); }
DPU_ASSERT(dpu_free(dpu_set));
```

- Application-level errors get their own macros
  (`ERROR_EXIT(err_code, fmt, ...)` with an `enum error_code` — upvc:host/inc/upvc.h);
  cleanup paths use `goto err;` + `DPU_ASSERT(dpu_free(...))` at the label (keccakf).
- Rank callbacks return `dpu_error_t` (`return DPU_OK;`).
- On kernel fault, dump the DPU logs before dying (`dpu_log_read_for_dpu`;
  dpu_olap wraps launch as `exec_with_log`).
- The allocation **profile string** is the tuning surface:
  `dpu_alloc(n, "nrJobsPerRank=64", &set)`; seen values:
  `"cycleAccurate=true,nrJobsPerRank=64"` (upvc),
  `"nrJobPerRank=64,dispatchOnAllRanks=true,cycleAccurate=true"` (UPIS),
  `"backend=hw"` with graceful CPU fallback if `dpu_alloc` fails (upvc index build).

## §2 Loading binaries

- Preferred: embed the kernel in the host binary —
  `DPU_INCBIN(prog, DPU_BINARY); dpu_load_from_incbin(set, &prog, NULL);`
  (upvc, UPIS). `DPU_BINARY` is a build-time `-D` (build.md §4).
- Alternative: `#define DPU_BINARY "build/checksum_dpu"` guarded by `#ifndef`
  (dpu_demo), or a runtime path argument with an existence check
  (usecase_dpu_alignment).

## §3 Transfers

Three tiers, in order of preference:

1. **Broadcast** (same data to all):
   `DPU_ASSERT(dpu_broadcast_to(set, XSTR(SYM), 0, buf, size, DPU_XFER_ASYNC));`
2. **Scatter/gather** (per-DPU data): prepare-then-push with the 3-arg
   `DPU_FOREACH` index variable. **The push size is the max across DPUs**; give
   idle DPUs dummy buffers so every DPU has a valid pointer:

   ```c
   dpu_results_t results[nr_of_dpus];
   DPU_FOREACH (dpu_set, dpu, each_dpu) {
       DPU_ASSERT(dpu_prepare_xfer(dpu, &results[each_dpu]));
   }
   DPU_ASSERT(dpu_push_xfer(dpu_set, DPU_XFER_FROM_DPU, XSTR(DPU_RESULTS),
                            0, sizeof(dpu_results_t), DPU_XFER_DEFAULT));
   ```
3. **Odd offsets** (e.g. per-tasklet slots inside a `__host` array): resolve the
   symbol once, then byte-offset copies —
   `dpu_get_symbol(program, "tasklet_params", &sym);`
   `dpu_copy_to_symbol(dpu, sym, t * sizeof(elem), &elem, sizeof(elem));` (keccakf).

## §4 Two-phase variable-size readback

Never gather the worst-case result array. Pull a per-DPU count symbol first,
then gather `max_count * sizeof(result)` — sized per rank, typically inside a
rank callback (upvc). UPIS mirrors this with `nr_total_responses`.

## §5 Async pipeline and callbacks

Everything is enqueued in order on the set's async queue and closed by one
`dpu_sync`:

```c
dpu_push_xfer(..., DPU_XFER_ASYNC);                    /* inputs */
DPU_ASSERT(dpu_launch(set, DPU_ASYNCHRONOUS));
/* async FROM_DPU xfers for outputs */
DPU_ASSERT(dpu_callback(set, postprocess_fn, ctx,
    DPU_CALLBACK_ASYNC | DPU_CALLBACK_NONBLOCKING | DPU_CALLBACK_SINGLE_CALL));
...
DPU_ASSERT(dpu_sync(set));
```

- **Rank = the scheduling/refill unit.** Callbacks flip a per-rank "free" flag
  (usecase_dpu_alignment: `get_free_rank()` spin-waits with `sleep_for(1ms)`)
  or post a POSIX semaphore to interlock CPU pipeline stages with rank
  completion (upvc: `dpu_callback(set, sem_post_fn, sem, ASYNC|NONBLOCKING|SINGLE_CALL)`).
- `dpu_callback` passes one `void *`: pack two u32s in a punned union and
  static-assert its size, or heap-allocate a named context struct per batch:

  ```c
  /* upvc:host/src/dpu_backend.c */
  typedef union {
      struct { uint32_t dpu_offset; uint32_t pass_id; };
      uint64_t info;
  } pass_info_t;
  _Static_assert(sizeof(pass_info_t) == sizeof(uint64_t),
      "dpu_callback using this type will not be functional");
  ```
- Callback rank-safety: index result vectors by rank id so callbacks never
  share slots ("should be thread safe because different ranks access different
  indexes" — dpu_olap:host/join/join_dpu.cc).
- Timing inside async chains: inject timers as non-blocking async callbacks
  bracketing phases (dpu_olap `ASYNC_TIMER`).

## §6 Rank bookkeeping and multi-buffering

- Enumerate ranks once at init (`DPU_RANK_FOREACH`), store per-rank DPU counts
  and a running offset (`rank_mram_offset[rank]` / `dpu_offset[]` prefix sums)
  so callbacks can map `(rank_id, each_dpu)` → global DPU id.
- Multi-buffering: rotate N host buffers keyed by pass id so CPU stages
  (dispatch/accumulate) overlap DPU execution; semaphores gate reuse:
  `#define REQUESTS_BUFFERS(pass_id) requests_buffers[(pass_id) % NB_DISPATCH_AND_ACC_BUFFER]`
  (upvc, N=4).
- Loop variables: `each_dpu`, `each_rank`, `each_tasklet`; counts `nr_of_dpus`,
  `nr_*` (SDK style) or `nb_*` (upvc).

## §7 C++ hosts

Wrap `extern "C" { #include <dpu.h> }` behind RAII:

- Full wrapper (dpu_olap:host/dpuext/dpuext.hpp, modeled on UPMEM's official
  C++ API): `DpuError : std::exception` + `throwOnErr(dpu_error_t)` replacing
  `DPU_ASSERT`; `DpuSet` frees in the destructor; `copy()` overloads keyed by
  argument shape (vector → broadcast, vector<vector> → scatter, dst-first →
  gather); a `bool async` field unifying sync/async so every op picks
  `DPU_XFER_ASYNC : DPU_XFER_DEFAULT`; `std::function` callbacks; and a
  `dpu_set_t &unsafe()` escape hatch.
- Thin wrapper (usecase_dpu_alignment:libnwdpu/host/Rank.hpp): keep `DPU_ASSERT`,
  add a non-copyable `Rank` owning one `dpu_alloc_ranks(1, ...)` set (rank = the
  scheduling unit), an `algo` policy member, and an `m_available` flag flipped
  by an async callback.
