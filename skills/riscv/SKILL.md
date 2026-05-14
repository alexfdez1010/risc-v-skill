---
name: riscv-rvv
description:
  RISC-V Vector (RVV) C intrinsics + optimization catalog. Use when writing,
  reading, optimizing, or porting RVV kernels (vsetvl/AVL loops, LMUL/SEW,
  loads/stores, reductions, softmax, NTT, CRC, perf counters), or when picking
  among 275 paper-sourced RVV 1.0 optimization ideas across compilation,
  microarchitecture, NN/LLM, crypto/codecs, scientific/HPC, and sequence
  alignment.
license: MIT
---

## Scope

- RVV 1.0 C intrinsics and the vector-length-agnostic execution model.
- Instruction forms, policies, data types.
- Memory access patterns (contiguous / strided / segmented / gather / slide).
- Reduction, softmax-style normalization, NTT-style modular arithmetic, CRC
  folding.
- Perf counter usage with `rdinstret` / `rdcycle`.
- 275-idea optimization catalog split by topic, derived from peer-reviewed
  papers and open-source kernels.

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

## Reference files (load on demand)

Load only what the current task needs. Progressive disclosure — keep the
context small.

### Core RVV reference

- `references/core-concepts.md` — VL/AVL, VLMAX, tail/mask policies, naming,
  memory access, reductions, conversions, widen/narrow, masks, bit ops, config
  macros.
- `references/patterns.md` — algorithmic templates: vector add, transpose,
  reduction, softmax, NTT/polynomial mult, CRC, microbenches, perf counters.
- `references/intrinsics.md` — cheat sheet by category (loads/stores,
  arithmetic, reductions, conversions, permute, bit ops).
- `references/examples.md` — 7 worked kernels (vec add, transpose, softmax,
  NTT, reduction, CRC, microbenches) with host/kernel interaction.
- `references/template.md` — step-by-step template for authoring a new RVV
  kernel.
- `references/glossary.md` — AVL, VL, VLMAX, SEW, LMUL, tail/mask policies,
  Zvbc.

### Optimization catalog (275 ideas)

Start with the overview, then load only the topic file that matches the task.

- `references/optimizations/overview.md` — entry index. Hardware targets, top
  quick-wins on K1, cross-topic routing table. Read this first when picking
  among ideas.
- `references/optimizations/generic.md` — base 56 generic tactics (A–F across
  vectorization, numerics, memory hierarchy, ILP, parallelism, domain
  kernels). Always-applicable RVV idioms; start here on any new kernel.
- `references/optimizations/compiler.md` — G1–G35. LLVM/GCC backends,
  MLIR/IREE/TVM autotuning, SVE→RVV / NEON→RVV / x86→RVV port, build flags,
  PGO, LLM-driven codegen. Load when setting up the build or porting from
  another SIMD ISA.
- `references/optimizations/microarch.md` — H1–H32. Per-board tuning
  (K1/X280/SG2042/Ara2), PMU/roofline profiling, energy + DVFS, NUMA,
  hugepages, io_uring, RT scheduling, autotuner persistence. Load when
  measuring, chasing the last 10–30 %, or working multi-core/NUMA.
- `references/optimizations/nn-llm.md` — I1–I35. FlashAttention,
  Flash-Decoding, GQA/MLA, KV quantization (KIVI/ZipCache/RotateKV/XQuant),
  AWQ/SmoothQuant/SqueezeLLM, MoE dispatch, Mamba/RWKV scans, speculative
  decoding, sparse attention, DejaVu, SwiGLU/GELU, NOVA Winograd, V-Seek
  recipes.
- `references/optimizations/crypto-codecs.md` — J1–J35. Zk* (AES/SHA/SM*/
  GHASH), Zvk* vector crypto, Zb* bitmanip, post-quantum NTT, bitsliced AES,
  BLAKE3/xxHash3/CRC32C, LZ4/zstd/Huffman/Parquet/simdjson/UTF-8, hash join,
  packet classify, JPEG IDCT.
- `references/optimizations/scientific.md` — K1–K40. Sparse LA (SELL-C, HYB,
  SpGEMM, SpTRSV), stencil/CFD, FEM/multigrid, Krylov, FFT (Stockham,
  Bluestein), N-body (FMM, Barnes-Hut), MD (LJ, PME), Monte Carlo, BFS /
  triangle, double-double, BLIS GEMM, batched ERI, Morton/octree.
- `references/optimizations/alignment.md` — L1–L46. Striped Smith-Waterman
  (Farrar/Zhao), Wozniak anti-diagonal, SWIPE inter-sequence, WFA, ksw2,
  edlib (Myers), Suzuki-Kasahara difference recurrence, libgaba adaptive
  band, HMMER3 MSV/Viterbi, PSSM, Pair-HMM, block aligner, Hirschberg,
  GASAL2, K1/X280/Ara2 hardware-specific tuning.

## Loading Guidance

### Routing for RVV kernel work

- Starting a new kernel → `template.md` + `patterns.md` for the target
  algorithm.
- Reading or modifying existing RVV code → `core-concepts.md` + `intrinsics.md`.
- Picking between strided / gather / segmented → `core-concepts.md` §5 and
  `patterns.md` §2.
- NTT / Barrett / modular → `patterns.md` §5.
- CRC / carry-less multiply → `patterns.md` §6 + `intrinsics.md` §Bit Ops.
- Unfamiliar term → `glossary.md`.

### Routing for optimization questions

Always begin with `optimizations/overview.md`, then load the topic file (or
files) that match the task. The overview's "cross-topic routing" table is the
single source of truth.

| Task signal | Files to load |
| --- | --- |
| Authoring a new kernel | `optimizations/generic.md` + topic file |
| Squeezing the last % on K1 | `optimizations/microarch.md` + `generic.md` §D |
| Porting from SSE2 / NEON / SVE | `optimizations/compiler.md` (G7, G8) + `alignment.md` L36 |
| LLM inference (llama.cpp / ggml) | `optimizations/nn-llm.md` + `microarch.md` H11, H18, H31 |
| Crypto / codec / DB | `optimizations/crypto-codecs.md` |
| HPC / CFD / FEM | `optimizations/scientific.md` + `microarch.md` H2, H4, H17 |
| Sequence alignment / bio | `optimizations/alignment.md` |
| Build / config / autotune | `optimizations/compiler.md` + `microarch.md` H10, H22, H24 |
| Profiling, regression gates | `optimizations/microarch.md` H8, H9, H30 |
