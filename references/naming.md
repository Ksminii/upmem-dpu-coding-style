# Naming, Formatting, Documentation

Evidence tags name the source repo under github.com/upmem.

## §1 clang-format

Formatting is tool-enforced, not reviewed by eye. UPMEM's own config
(identical in dpu_demo and keccakf):

```yaml
BasedOnStyle: Webkit
ColumnLimit: 130
PointerAlignment: Right
ForEachMacros: ['DPU_FOREACH', 'DPU_RANK_FOREACH']
```

- `ForEachMacros` makes `DPU_FOREACH (set, dpu) {` format like a for-loop
  (note the space before `(`).
- Net effect: 4-space indent, K&R braces (function brace on its own line),
  long lines allowed, `type *ptr` right-aligned pointers.
- dpu_olap (Arrow-adjacent) instead uses `BasedOnStyle: Google, ColumnLimit: 90`
  — pick one config per repo and add a `check-format` make target (build.md §6).

## §2 Identifiers

- `snake_case` for functions and variables; `SCREAMING_SNAKE` for macros and
  capacity constants — always with unit comments:
  `#define MAX_CIGAR_SIZE 32000000LU // 32MB of MRAM for cigars`.
- Types end `_t` (`dpu_request_t`, `dout_t`, `parser_t`) or stay bare
  `struct dpu_params` (keccakf). Shared host↔DPU structs may be PascalCase in
  C++-host repos (`NwMetadataDPU`).
- Counters: `nr_*` (SDK style) or `nb_*` (upvc) — consistent within a repo.
- Loop variables follow the SDK foreach idiom: `each_dpu`, `each_rank`,
  `each_tasklet`, `each_request`.
- Power-of-two sizes get log2 companions for shift indexing:
  `BLOCK_LENGTH` + `BLOCK_LENGTH_LOG2`.
- Multi-module kernels prefix each module's API with the module name:
  `request_pool_init`, `request_pool_next`, `result_pool_write`, `dout_add`
  (upvc — one `.c` + one `.h` per module).

## §3 DPU symbol names

Short lowercase nouns: `"sequences"`, `"metadata"`, `"output"`, `"cigars"`,
`"buffer"`, `"buffer_length"`, `"kernel"`. Variants seen: an `m_` prefix
marking DPU/MRAM residency (`m_dpu_request`, upvc); uppercase for the two main
param blobs (`__host ... INPUT; __host ... OUTPUT;`, dpu_olap). Always defined
through the symbol-name macro contract (interface.md §3), never as bare string
literals scattered through host code.

## §4 Comments and documentation

- Doxygen on public functions and shared structs: `/** @brief ... @param ...
  @return */`, `///` trailing comments on struct fields.
- **Every kernel file opens with a prose block** describing the
  parallelization strategy (dpu_demo names its strategy — "rake") and, where
  relevant, the MRAM map (UPIS:decoder.c documents the full address-table
  layout in a comment atop the file).
- Cite papers by URL in a comment where an algorithm comes from.
- License header on every file (Apache-2.0 / MIT / BSD depending on repo).

## §5 Compiled-out instrumentation

Every debug/stats/trace feature is a macro pair — expands to code when its
flag is set, to nothing otherwise — and **call sites are kept unconditional**
so measured and debug binaries share all code paths:

```c
/* upvc:common/inc/stats.h pattern */
#ifdef STATS_ON
#define STATS_INCR_NB_REQS(stats) do { (stats)->nb_reqs++; } while (0)
#define STATS_ATTRIBUTE
#else
#define STATS_INCR_NB_REQS(stats)
#define STATS_ATTRIBUTE __attribute__((unused))
#endif

void process(STATS_ATTRIBUTE stats_t *stats, ...);  /* param survives both configs */
```

Trace printfs follow the same shape (`#define PRINT printf` / `#define PRINT(...)`
— UPIS trace.h), gated by `TRACE`/`ENABLE_LOG`/`ENABLE_TRACE`/`PERF_VERSION`
flags centralized in one shared header (dpu_olap:shared/umq/cflags.h). Wrap
statement macros in `do { ... } while (0)`.
