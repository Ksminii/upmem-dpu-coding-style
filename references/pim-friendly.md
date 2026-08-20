# PIM-Friendly Implementation

Rules about the *shape* of the computation, not code style. Two evidence bases,
tagged per rule: repo tags name official github.com/upmem code; **(PrIM)** =
Gómez-Luna et al., "Benchmarking a New Paradigm: Experimental Analysis of the
UPMEM PIM Architecture" (IEEE Access 2022, arXiv:2105.03814); **(SDK)** =
UPMEM SDK documentation.

Hardware model in one line: each DPU is an in-order 32-bit RISC core with a
14-stage pipeline and round-robin ("revolver") tasklet scheduling, no FPU,
only an 8×8-bit hardware multiply step, 64 KB WRAM, 24 KB IRAM, 64 MB MRAM,
and no path to other DPUs except through the host.

## §1 Avoid multiply/divide in hot loops

There is no 32-bit multiplier: `*` lowers to a `mul_step` sequence or runtime
call, `/` and `%` to library calls **(PrIM, SDK)**. Size things as powers of
two and shift; keep a log2 companion for every block-size constant — this is
why the official repos define `BLOCK_LENGTH` + `BLOCK_LENGTH_LOG2` pairs
(dpu_olap):

```c
for (unsigned int off = tasklet_id << BLOCK_LENGTH_LOG2;
     off < len; off += BLOCK_LENGTH * NR_TASKLETS)   /* compiler folds the constant */
```

Non-power-of-two constant divisors deserve suspicion too: the compiler's
reciprocal-multiply trick still needs the slow multiplier. Prefer fixed-point
with power-of-two denominators (scale by 128, not 100), hoist the division out
of the loop, or precompute the scaled values on the host.

## §2 Stay 32-bit

The core is 32-bit: 64-bit arithmetic costs ≥2 instructions per op, and some
64-bit builtins lower to library calls. Split them explicitly:

```c
/* __builtin_popcountll becomes a call on the DPU — split it */
static inline int pop64(uint64_t x) {
    return __builtin_popcount((uint32_t)x)
         + __builtin_popcount((uint32_t)(x >> 32));
}
```

Prefer 32-bit loop indices and offsets; keep 64-bit only where the algorithm's
data demands it (e.g. bit-parallel words).

## §3 No floating point

There is no FPU; float/double emulate in software at order-of-magnitude cost
**(PrIM)**. Use integers or fixed-point on the DPU; if the algorithm needs
real-valued math, do that part on the host.

## §4 Partial results per tasklet, reduction on the host

Tasklets accumulate into private slots; the host (or tasklet 0 after a
barrier) combines. This is the official demo's documented structure: "The host
is in charge of computing the final checksum by adding all the individual
results" (dpu_demo:checksum/dpu/dpu.c). A shared accumulator behind a mutex
serializes the pipeline — reserve mutexes for slot *allocation*, not for
arithmetic (kernel.md §6).

## §5 MRAM is persistent across launches — exploit it

MRAM contents survive between `dpu_launch` calls. Load reference/index data
once and relaunch against it instead of re-transferring per batch — upvc
streams a new MRAM image only per *pass*, not per launch. Host↔DPU transfer
is the scarce resource: batch transfers rank-wide, make them `DPU_XFER_ASYNC`,
and overlap them with DPU execution and CPU work (host.md §5). Never transfer
to a running rank — the call blocks until the rank finishes.

## §6 Size DMA transfers to amortize fixed cost

Each `mram_read`/`mram_write` pays a fixed setup cost plus a per-byte cost;
tiny transfers are latency-bound and large ones bandwidth-bound **(PrIM)**.
Batch small records per DMA (upvc fetches 4 records per read), use chunk sizes
that are small powers of two up to the 2048 B cap, and prefer the largest
WRAM cache the budget allows for streaming loops (kernel.md §5). Sequential
MRAM access through `seqread` beats element-wise random access.

## §7 Tasklet count: ≥11 saturates the pipeline — but trade against WRAM

The 14-stage pipeline with round-robin scheduling needs ~11 concurrent
tasklets before the core issues an instruction every cycle; below that,
cycles go idle **(PrIM)**. Official repos default `NR_TASKLETS = 16`.
The real constraint is WRAM: per-tasklet buffers × T must fit (kernel.md §4),
so memory-heavy kernels may deliberately run fewer, longer tasklets. Make the
choice explicit and per-workload (`NR_TASKLETS ?=` build knob), and know which
side of the trade you're on before tuning anything else.

## §8 Assume compute-bound until measured otherwise

For most workloads the DPU pipeline, not MRAM bandwidth, is the bottleneck —
even streaming kernels saturate compute first **(PrIM)**. Optimizing
instruction count (fewer ops per element, §1–§3) usually pays more than
optimizing memory access patterns. To check which regime you're in
(kernel.md §8 for mechanics): `perfcounter_config` counts either cycles or
instructions, one per run — run once with `COUNT_CYCLES` and once with
`COUNT_INSTRUCTIONS` and compare. Per-tasklet IPC near the issue ceiling
(≈1/max(11, T) of core cycles at low T) means compute-bound — cut
instructions; a cycles count far above what the instruction count explains
means DMA stalls — restructure MRAM access (§6).

## §9 Shard with zero inter-DPU dependencies

DPUs cannot communicate with each other; any cross-DPU dataflow becomes a
host round-trip (gather → CPU merge → scatter) **(SDK, PrIM)**. Partition the
problem so each DPU's work is closed over its own MRAM, duplicate small shared
state into every DPU (broadcast), and push merge steps to the host (§4).
If the algorithm fundamentally needs neighbor exchange per iteration, expect
the host loop to dominate — restructure or keep that part on the CPU.

## §10 Respect the 24 KB IRAM budget

Kernel code shares 24 KB of instruction memory. Aggressive inlining, deep
unrolling, large switch tables, or linking library math (see §1–§3) can
overflow it — surfacing as link errors or evicted hot loops. Keep kernels
small and single-purpose (one binary per kernel — interface.md §3's
`enum Kernel` selector is for *small* variant sets), and prefer `-O2` where
`-O3`'s unrolling bloats code for no measured gain (build.md §4).

## §11 Balance across DPUs first, tasklets second

A launch completes when the slowest DPU finishes: skew across DPUs stalls the
whole rank at the sync point, while skew across tasklets wastes only one DPU's
pipeline slots. Spend effort in this order: (1) even the per-DPU load
host-side (work-split.md §5), (2) let a dynamic queue absorb residue within
each DPU (work-split.md §3). The official alignment app does exactly this
two-level split (usecase_dpu_alignment).
