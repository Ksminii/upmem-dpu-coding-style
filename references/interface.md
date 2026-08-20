# Host↔DPU Interface

Evidence tags name the source repo under github.com/upmem.

## §1 Shared header contract

Exactly one plain-C header (or one directory of them) is included by BOTH the
host and the kernel. It may contain only: `#include <stdint.h>`, fixed-width
structs, enums, and capacity `#define`s. No host headers, no SDK headers.

Layouts seen: `common/common.h` (dpu_demo), `common/inc/*.h` (upvc, UPIS),
repo-root `cdefs.h` (usecase_dpu_alignment), `shared/umq/*.h` (dpu_olap).

```c
/* usecase_keccakf:keccakf_dpu_params.h — a complete shared header */
struct dpu_params {
    uint32_t fkey;  /* first key */
    uint32_t lkey;  /* last key */
    uint32_t loops; /* #loops */
};
struct dpu_result {
    uint64_t sum;
    uint64_t cycles;
};
```

Per-tasklet result arrays are dimensioned by `NR_TASKLETS` in this header —
`dpu_result_t tasklet_result[NR_TASKLETS];` (dpu_demo) — which is why the host
compiler must receive the same `-DNR_TASKLETS` as the kernel (build.md §2).

Keep the param struct small and ship bulk data as its own transfer. UPMEM's own
comment: "Sequence buffer is send separatly to keep the structure small enough
otherwise stack errors occurs" (usecase_dpu_alignment:cdefs.h).

## §2 8-byte compliance

Every transferred struct must be 8-byte compliant. Two accepted mechanisms:

```c
/* dpu_olap:shared/umq/kernels.h — tag the whole struct */
typedef struct __attribute__((aligned(8))) { ... } kernel_partition_inputs_t;

/* usecase_dpu_alignment:cdefs.h — size fields to a multiple of 8 */
// int8_t pad[4]; /// padding for mram compliance   <- kept as a comment scar

/* upvc:common/inc/common.h — round member sizes with a macro */
#define ALIGN_DPU(val) (((val) + 7) & ~7)
uint8_t nbr[ALIGN_DPU(SIZE_NEIGHBOUR_IN_BYTES)];
```

## §3 Symbol-name macro contract

For every transferred object the shared header defines the **type** and a
**macro naming the DPU symbol**, so host and kernel cannot drift. The kernel
*defines* the symbol with the macro; the host *addresses* it stringified:

```c
/* shared header */
#define XSTR(x) STR(x)
#define STR(x) #x
#define DPU_REQUEST_VAR m_dpu_request
typedef struct { uint32_t offset; uint32_t count; ... } dpu_request_t;

/* kernel (upvc:dpu/src/request_pool.c) */
__host nb_request_t DPU_NB_REQUEST_VAR;
__mram_noinit dpu_request_t DPU_REQUEST_VAR[MAX_DPU_REQUEST];

/* host (upvc:host/src/dpu_backend.c) */
DPU_ASSERT(dpu_push_xfer(set, DPU_XFER_TO_DPU, XSTR(DPU_REQUEST_VAR),
                         0, max_dispatch_size, DPU_XFER_ASYNC));
```

Symbol names themselves are short lowercase nouns: `"sequences"`, `"metadata"`,
`"output"`, `"buffer"`, `"kernel"` (naming.md §3).

To multiplex several kernels in one binary, add a `__host enum Kernel kernel;`
selector switched in `main()`; the host sets it per launch (dpu_olap).

## §4 MRAM layout strategies

Three strategies, in order of prevalence:

1. **Linker-declared worst-case arrays** (default everywhere): named
   `__mram_noinit` globals sized by compile-time capacity macros, doubling as
   transfer symbols. No runtime offset computation.

   ```c
   __mram_noinit uint8_t sequences[SCORE_MAX_SEQUENCES_TOTAL_SIZE];
   __mram_noinit uint8_t cigars[MAX_CIGAR_SIZE];      /* 32MB of MRAM */
   __host uint32_t buffer_length = 0;
   ```

2. **MRAM heap** for data too large or too variable to declare: kernel
   addresses relative to `DPU_MRAM_HEAP_POINTER`, host writes by name:

   ```c
   /* kernel (upvc:dpu/src/task.c) */
   uintptr_t addr = (uintptr_t)DPU_MRAM_HEAP_POINTER + (base + idx) * sizeof(coords_and_nbr_t);
   mram_read((__mram_ptr void *)addr, cache, len);
   /* host */
   DPU_ASSERT(dpu_push_xfer(rank, DPU_XFER_TO_DPU, DPU_MRAM_HEAP_POINTER_NAME,
                            0, max_mram_size, DPU_XFER_DEFAULT));
   ```

3. **Prebuilt MRAM images**: build the whole MRAM offline into `mram_%04u.bin`
   files, stream one per DPU per pass; the image starts with an address table
   the kernel indirects through (upvc host/src/mram_dpu.c, UPIS decoder.c —
   which documents the MRAM map in a long comment atop the file).

dpu_olap additionally packages reusable MRAM layers worth copying:
`mram_alloc.h` — a bump allocator mirroring WRAM's `mem_alloc`
(`__mram_ptr void *mram_alloc(size_t)`, 64-bit aligned, not thread safe) —
and `mram_ra.h` — 4B/8B random access built on 8B-aligned `mram_read`/
`mram_write` with a `_Generic` dispatch macro and read-modify-write for
sub-8B stores.

## §5 Variable-length items

Bump layout + offset/length table inside the fixed metadata struct
(usecase_dpu_alignment, sequence alignment with per-pair lengths):

```c
typedef struct NwMetadataDPU {
    uint32_t indexes[DPU_MAX_NUMBER_OF_SEQUENCES]; /// start offset of Nth item
    uint16_t lengths[DPU_MAX_NUMBER_OF_SEQUENCES]; /// length of Nth item
    ...scoring params...
} NwMetadataDPU;
```

- Each item is individually padded to 8 B before concatenation
  (`round_up8((len + 3) / 4)` for 2-bit-packed DNA).
- Outputs use the same shape: one flat MRAM blob + host-precomputed start
  offsets (`cigar_indexes[]`, per-pair capacity `len1+len2` rounded to 8),
  actual lengths returned in the result struct.
- Kernel-side lookup is direct: `init_dna1(&sequences[metadata.indexes[s1]])`.
- The host asserts capacity while packing:
  `assert(cigar_index < MAX_CIGAR_SIZE && "not enough space for cigar on dpu!");`

## §6 Capacity guards

Static-assert on the host side that the fixed regions fit:

```c
/* upvc:host/src/mram_dpu.c */
#define MRAM_SIZE_AVAILABLE (MRAM_SIZE \
    - MAX_DPU_REQUEST * sizeof(dpu_request_t) \
    - MAX_DPU_RESULTS * sizeof(dpu_result_out_t))
_Static_assert(MRAM_SIZE_AVAILABLE > 0,
    "Too many request and/or result compare to MRAM_SIZE");
```

Capacity macros carry unit comments: `#define MAX_CIGAR_SIZE 32000000LU // 32MB of MRAM for cigars`.
