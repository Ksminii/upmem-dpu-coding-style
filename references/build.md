# Build Conventions

Evidence tags name the source repo under github.com/upmem.

## §1 Toolchain

- DPU compiler is always **`dpu-upmem-dpurte-clang`**, invoked as a one-shot
  compile+link of ALL kernel sources into one binary:
  `dpu-upmem-dpurte-clang ${DPU_FLAGS} ${DPU_SOURCES} -o $@`
- Host uses `cc`/`g++` with `` `dpu-pkg-config --cflags --libs dpu` `` on the
  compile/link line (or split `--cflags` / `--libs` into CFLAGS/LDFLAGS).

## §2 The `-D` contract (the rule that prevents host/kernel drift)

`-DNR_TASKLETS=${NR_TASKLETS}` goes to **BOTH** compilers from **ONE** build
variable. The DPU runtime spawns that many tasklets from the define; the host
indexes `[NR_TASKLETS]`-sized shared structs with it. A mismatch surfaces only
as a fault at launch.

```make
# UPIS:Makefile — minimal canonical form
NR_TASKLETS ?= 16
CFLAGS = ... `dpu-pkg-config --cflags dpu` -DNR_TASKLETS=${NR_TASKLETS}
LDFLAGS = `dpu-pkg-config --libs dpu`
DPU_FLAGS = -g -O2 -Wall -Werror -Wextra -flto=thin -Idpu/inc -Icommon/inc \
            -DNR_TASKLETS=${NR_TASKLETS} -DSTACK_SIZE_DEFAULT=256

${HOST_BINARY}: ${HOST_SOURCES}
	$(CC) -o $@ ${HOST_SOURCES} $(LDFLAGS) $(CFLAGS) \
	      -DDPU_BINARY=\"$(realpath ${DPU_BINARY})\"
${DPU_BINARY}: ${DPU_SOURCES}
	dpu-upmem-dpurte-clang ${DPU_FLAGS} ${DPU_SOURCES} -o $@
```

Other values passed the same way: `NR_DPUS` (host), `DPU_BINARY` (host,
absolute via `$(realpath ...)` — or avoided entirely with `DPU_INCBIN`),
`STACK_SIZE_DEFAULT` (kernel).

## §3 Config stamp file

Encode the `-D` values in a per-config stamp file both binaries depend on, so
changing `NR_TASKLETS` (or `NR_DPUS`) forces a rebuild of BOTH sides — the
official fix for "stale kernel binary after a -D change":

```make
# dpu_demo:checksum/Makefile
define conf_filename
	${BUILDDIR}/.NR_DPUS_$(1)_NR_TASKLETS_$(2).conf
endef
CONF := $(call conf_filename,${NR_DPUS},${NR_TASKLETS})

${CONF}:
	$(RM) $(call conf_filename,*,*)
	touch ${CONF}

${DPU_TARGET}: ${DPU_SOURCES} ${COMMON_INCLUDES} ${CONF}
${HOST_TARGET}: ${HOST_SOURCES} ${COMMON_INCLUDES} ${CONF}
```

## §4 Flags

- Optimization split: DPU `-O2` (occasionally `-O3 -fno-builtin`), host `-O3`.
- Warnings: `-Wall -Wextra -Werror -g` — ideally one `COMMON_FLAGS` shared by
  host and DPU rules (dpu_demo).
- **Per-tasklet stack is an explicit knob**: `-DSTACK_SIZE_DEFAULT=…` — values
  in the wild: 192 (upvc), 256 (UPIS), 384 (usecase_dpu_alignment), 768
  (tandem_demo). Shrink deliberately when `NR_TASKLETS × stack` approaches the
  WRAM budget; add `-fstack-size-section` to audit (upvc).
- Extras seen: `-flto=thin` (UPIS), `-fshort-enums` (usecase_dpu_alignment —
  keep enum sizes matching across host/kernel if you do this on one side only:
  don't), custom linker script `-Wl,-T.../dpu.lds` (tandem_demo).
- Build dir convention: `BUILDDIR ?= build`, created by
  `__dirs := $(shell mkdir -p ${BUILDDIR})`; binaries named `<app>_dpu` /
  `<app>_host`.

## §5 CMake variant

Kernel and host are separate projects; the kernel builds as an ExternalProject
cross-compiled with the SDK toolchain file, with `NR_TASKLETS` forwarded so
both sides agree by construction:

```cmake
# upvc:CMakeLists.txt (host side)
set(NR_TASKLETS 16)
set(CMAKE_C_FLAGS "... -DNR_TASKLETS=${NR_TASKLETS} -DDPU_BINARY=\\\"...\\\"")
ExternalProject_Add(dpu_task ...
    CMAKE_ARGS -DCMAKE_TOOLCHAIN_FILE=${UPMEM_HOME}/share/upmem/cmake/dpu.cmake
               -DNR_TASKLETS=${NR_TASKLETS})
```

- Kernel side: `include("${UPMEM_HOME}/share/upmem/cmake/dpu.cmake")`,
  `add_executable(kernel-x ...)`,
  `target_compile_definitions(... PUBLIC NR_TASKLETS=${NR_TASKLETS})` **and**
  the same define on `target_link_options` — the linker needs it too (dpu_olap).
- Host side: `include(.../DpuHost.cmake)` →
  `${DPU_HOST_INCLUDE_DIRECTORIES}` / `${DPU_HOST_LIBRARIES}`.
- `project(dpu C ASM)` when the kernel has `.S` files (upvc).
- Debug configs carry `-fsanitize=address` on the host only (dpu_olap).

## §6 Format gate

Ship a `check-format` target diffing `clang-format` output over the tree
(keccakf, UPIS) — see naming.md §1 for the format configs.
