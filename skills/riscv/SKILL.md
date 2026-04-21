---
name: riscv-rvv
description:
  RISC-V Vector (RVV) C intrinsics reference. Use when writing, reading, or
  optimizing RVV kernels (vsetvl/AVL loops, LMUL/SEW, loads/stores, reductions,
  softmax, NTT, CRC, perf counters).
license: MIT
---

## Scope

- RVV C intrinsics and vector-length-agnostic execution model
- Instruction forms, policies, and data types
- Memory access patterns (contiguous/strided/segmented/gather/slide)
- Reduction, softmax-style normalization, NTT-style modular arithmetic, and CRC
  folding patterns
- Perf counter usage with `rdinstret` / `rdcycle`
- Build/toolchain details intentionally omitted

## Quick Template

```c
#include <riscv_vector.h>
#include <stddef.h>

void my_kernel(float *dst, const float *src, size_t n) {
    size_t avl = n;
    while (avl > 0) {
        size_t vl = __riscv_vsetvl_e32m1(avl);
        vfloat32m1_t v = __riscv_vle32_v_f32m1(src, vl);
        // ... compute ...
        __riscv_vse32_v_f32m1(dst, v, vl);
        src += vl; dst += vl; avl -= vl;
    }
}
```

## Reference Files (load on demand)

Load only what the current task needs:

- `references/core-concepts.md` — VL/AVL, VLMAX, tail/mask policies, naming,
  memory access, reductions, conversions, widen/narrow, masks, bit ops, config
  macros.
- `references/patterns.md` — algorithmic templates: vector add, transpose,
  reduction, softmax, NTT/polynomial mult, CRC, microbenchmarks, perf counters.
- `references/intrinsics.md` — cheat sheet of common intrinsics grouped by
  category (loads/stores, arithmetic, reductions, conversions, permute, bit
  ops).
- `references/examples.md` — 7 worked kernels (vec add, transpose, softmax, NTT,
  reduction, CRC, microbenches) with host/kernel interaction.
- `references/template.md` — step-by-step template for authoring a new RVV
  kernel.
- `references/glossary.md` — AVL, VL, VLMAX, SEW, LMUL, tail/mask policies,
  Zvbc.

## Loading Guidance

- Starting a new kernel → `template.md` + `patterns.md` for the target
  algorithm.
- Reading/modifying existing RVV code → `core-concepts.md` + `intrinsics.md`.
- Picking between strided/gather/segmented → `core-concepts.md` §5 and
  `patterns.md` §2.
- NTT / Barrett / modular → `patterns.md` §5.
- CRC / carry-less multiply → `patterns.md` §6 + `intrinsics.md` §Bit Ops.
- Unfamiliar term → `glossary.md`.
