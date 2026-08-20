# upmem-dpu-coding-style

A [Claude Code skill](https://code.claude.com/docs/en/skills) for writing,
reviewing, and optimizing [UPMEM](https://www.upmem.com/) PIM code — DPU
kernels and `dpu.h` host code. Conventions were extracted by reading the
official UPMEM org repositories (2026-08), plus performance rules from the
PrIM benchmarking paper.

## Structure

- **[SKILL.md](SKILL.md)** — working layer: trigger conditions, one-line rule
  checklist, common-mistakes table. This is all an agent loads by default.
- **references/** — detail layer, one file per topic, loaded on demand. Each
  rule carries code excerpts and an evidence tag naming its source repo/paper.
  - [interface.md](references/interface.md) — shared header contract, symbol-name macros, 8-byte alignment, MRAM layout strategies, variable-length data
  - [kernel.md](references/kernel.md) — includes, boot pattern, placement attributes, WRAM/DMA, synchronization, errors, perfcounter
  - [work-split.md](references/work-split.md) — tasklet partitioning patterns, host-side load balancing
  - [host.md](references/host.md) — lifecycle, transfer tiers, two-phase readback, async pipelines, callbacks, C++ wrappers
  - [build.md](references/build.md) — toolchain, the `-D` contract, config stamp files, flags, CMake
  - [naming.md](references/naming.md) — clang-format, identifiers, comments, compiled-out instrumentation
  - [pim-friendly.md](references/pim-friendly.md) — performance rules shaped by the hardware (no FPU/multiplier, pipeline saturation, DMA sizing, sharding)

## Install

Copy the folder into your skills directory:

```
~/.claude/skills/upmem-dpu-coding-style/
```

Claude Code triggers it automatically on UPMEM/DPU work; or invoke it
explicitly with `/upmem-dpu-coding-style`.

## Sources

Official UPMEM repositories read for the conventions:

- [upmem/dpu_demo](https://github.com/upmem/dpu_demo) — canonical SDK example
- [upmem/usecase_upvc](https://github.com/upmem/usecase_upvc) — genomics variant caller (large multi-module C app)
- [upmem/usecase_UPIS](https://github.com/upmem/usecase_UPIS) — index search
- [upmem/usecase_dpu_alignment](https://github.com/upmem/usecase_dpu_alignment) — sequence alignment (C++ host)
- [upmem/dpu_olap](https://github.com/upmem/dpu_olap) — SQL engine (C++ host wrapper)
- [upmem/usecase_keccakf](https://github.com/upmem/usecase_keccakf) — Keccak-f benchmark
- [upmem/tandem_demo](https://github.com/upmem/tandem_demo) — secure-boot demo (DPU-side conventions only)

Performance rules tagged **(PrIM)** come from:

- Juan Gómez-Luna et al., *Benchmarking a New Paradigm: Experimental Analysis
  of the UPMEM Processing-in-Memory Architecture*, IEEE Access 2022.
  [arXiv:2105.03814](https://arxiv.org/abs/2105.03814) ·
  [SAFARI/prim-benchmarks](https://github.com/CMU-SAFARI/prim-benchmarks)

Rules tagged **(SDK)** come from the [UPMEM SDK documentation](https://sdk.upmem.com/).

## Disclaimer

Not affiliated with or endorsed by UPMEM. Conventions are descriptive
(what the official code does), not an official style guide.
