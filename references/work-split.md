# Tasklet Work Partitioning

Pick by workload shape. Evidence tags name the source repo under github.com/upmem.

## §1 Uniform work → static block-cyclic ("rake")

The canonical pattern, documented by name in dpu_demo's kernel header comment:
each tasklet strides over MRAM in `BLOCK_SIZE` steps interleaved by tasklet id.

```c
/* dpu_demo:checksum/dpu/dpu.c */
for (uint32_t buffer_idx = tasklet_id * BLOCK_SIZE; buffer_idx < BUFFER_SIZE;
     buffer_idx += (NR_TASKLETS * BLOCK_SIZE)) {
    mram_read(&DPU_BUFFER[buffer_idx], cache, BLOCK_SIZE);
    ...
}

/* shift variant with a log2 companion macro (dpu_olap:dpu/take/main.c) */
for (unsigned int off = tasklet_id << BLOCK_LENGTH_LOG2;
     off < len; off += BLOCK_LENGTH * NR_TASKLETS)
```

## §2 Host-side static ranges

The host precomputes one contiguous `[first, last)` slice per (dpu, tasklet)
with ceil-division balancing and pushes one param struct per tasklet slot:

```c
/* keccakf:app_host.c */
params.fkey = fkey + (nkey * idx + nr_of_tasklets - 1) / nr_of_tasklets;
```

UPIS keeps this as a compile-time fallback next to the dynamic path:
`for (seg = me(); seg < NR_SEGMENTS_PER_MRAM; seg += NR_TASKLETS)`.

## §3 Uneven work → dynamic queue under a mutex

Shared cursor; each tasklet pulls the next item until exhausted. The item's
`mram_read` may sit inside the critical section, but result writes go outside
it.

```c
/* UPIS:dpu/src/main.c (default, behind #define TASKLET_LOAD_SHARE) */
mutex_lock(job_mutex);
id = jobid++;
mutex_unlock(job_mutex);
if (id >= nr_jobs) break;
```

upvc structures both ends as pools: a mutex-protected input FIFO
(`request_pool.c: request_pool_next()`) and a mutex-protected output FIFO with a
per-tasklet MRAM swap area (`dout.c`) so only the final write is serialized.
Result-slot allocation = mutexed shared counter, MRAM write outside the lock.

## §4 Tasklet groups (master/slave)

For work items too big for one tasklet, usecase_dpu_alignment organizes 24
tasklets as 6 groups of 4 (`#define group() (me() / 4)`, `NR_GROUPS`): one
master per group runs the outer loop; slaves park on a raw stop/resume
instruction protocol and execute a function selected per wakeup:

```c
/* libnwdpu/dpu/nw_common.h */
__asm__ volatile("stop false, 1f; 1:");          /* slave parks itself */
__asm__ volatile("resume %[id], 0" ::[id] "r"(tid)); /* master wakes slave */
```

with a per-tasklet selector struct (`tasklet_params[NR_TASKLETS]`, enum
`Functions {AFFINE, ..., SHIFT_FV}`) and all per-group state in one
`struct align_t align_data[NR_GROUPS]` indexed by `group()`. Per-group barriers
must be enumerated by hand (kernel.md §6). This is an advanced pattern — reach
for it only when a single work item needs multiple tasklets.

## §5 Host-side pre-balancing for heterogeneous work

When per-item cost varies widely, balance before launch
(usecase_dpu_alignment:libnwdpu/host/AppSet.hpp):

1. Cost model per item (`count_compute_load` ≈ banded cell count).
2. Sort items by cost descending.
3. Carve rank-sized buckets against a load threshold (`take_load`).
4. Within a rank, greedy longest-processing-time: assign each item to the
   currently least-loaded DPU (`bucket_sets`).

Combine with §3 on-device: host balancing gets DPUs roughly even, the dynamic
queue absorbs the remainder within each DPU.
