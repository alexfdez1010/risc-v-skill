# G — Compiler & Code-Generation for RVV 1.0

35 implementable techniques across LLVM, GCC, MLIR/IREE, TVM autotuning,
SVE/NEON→RVV port, LLM-driven codegen. Load when configuring builds, writing
intrinsic wrappers, porting from another SIMD ISA, or chasing autovectorizer
misses.

---

**G1. `RISCVInsertVSETVLI` demand-merge.** LLVM's pass inserts a `vsetvli` per
def-use boundary; tune it to merge consecutive sets with identical AVL/SEW/LMUL
via a lattice merge. Eliminates redundant `vsetvli` in loops with mid-body LMUL
changes. Each `vsetvli` is a 2–4 cycle bubble on X60.

**G2. Register-pressure-aware LMUL predictor.** Compile-time spill dry-run
estimates spill cost per LMUL; pick the highest that stays under the 32-reg
ceiling. 73/76 correct on TSVC vs fixed `m1` default.

**G3. Linear-recurrence vectorization.** Extend `LoopVectorizer` with a
`RecurrenceDescriptor` for prefix-sum scans: intra-chunk Hillis–Steele via
`vslideup` doubling + scalar inter-chunk carry. 17.64× over scalar baseline on
plus-scan.

**G4. TVM MetaSchedule autotune.** Expose LMUL, tile sizes, unroll factor,
loop-order as tunables; sample via probabilistic grammar; train cost model on
real hardware. Beats GCC by 46 % and muRISCV-NN by 29 % on CNN/NN ops.

**G5. Fixed-point auto-tuning with P-extension.** Replace fp32 tiles with INT16
on `SMBB16`/`KMABB`; auto-tune binary-point per operator. 2.54–6.15× inst-count
reduction at <1 % accuracy loss on MobileNet/ResNet.

**G6. IREE `linalg.mmt4d` lowering.** Lower `linalg.generic` contractions to
`mmt4d` before vector lowering; hand-tuned RVV micro-kernels for the leaf tile
plug in via `codegen.llvm_func`. Beats upstream IREE + llama.cpp on Llama-3.2-1B
tokens/s.

**G7. SVE→RVV mask translation (SIMDe).** SVE predicates are element-indexed;
RVV masks are bit-packed (1 bit per VLEN/8). Implement `sveze_to_rvv_mask()`
with `vsetvli` + `vmsbf` + bitops; data-flow pass eliminates redundant
`vmset.m`/`vmclr.m`. 1.39–28.9× over scalar fallback.

**G8. NEON→RVV fixed-to-VLA strategy (SIMDe).** Map 128-bit NEON either to
`vsetvli vlmax=1` (correct but slow) or strip-mine the call site to a VL chunk
loop. SSA analysis detects same-width NEON→NEON chains and rewrites the entire
chain as one VLA loop.

**G9. muRISCV-NN: hand-written intrinsics beat autovec.** CMSIS-NN port with
manual LMUL selection per operator, `vmacc.vx`/`vwmacc.vx` widening for INT8,
`vnclip.wi` for re-quant. Up to 60 % lower runtime than LLVM autovectorizer with
smaller ROM.

**G10. Zoozve: arbitrary LMUL via ISA ext.** Research extension allowing any
integer register grouping (not just power-of-two); data-adaptive register
allocator in LLVM. ≥10.1× reduction in FFT dyn-inst count.

**G11. VLEN-adaptive layout via MLIR `transform.tile`.** Detect VLEN at
compile-time-specialization (`csrr t0, vlenb`) and rearrange perf-critical
arrays so one full `vl` maps to one cache line. Eliminates cross-cache-line
penalties on VLA kernels deployed across heterogeneous VLEN.

**G12. Scalable-vector vectorizer hints.** `#pragma clang loop
vectorize_width(scalable)` + `-mrvv-vector-bits=zvl256b|zvl512b` forces scalable
mode and `vsetvli`-based tail handling, eliminating the scalar remainder loop.

**G13. GCC `rvv-vector-bits` static vs dynamic VL.** Static-VL folds tail
checks but assumes a fixed VF; dynamic-VL emits `vsetvli` per iter. Pick per
loop via pragma or PGO trip-count histograms.

**G14. OpenCV Universal Intrinsics RVV HAL.** Replace scalar tails with masked
RVV (`vlseg`/`vle` + `vl=remainder`); use `vcvt` widening for resize/warp
bilinear. Tens-of-percent speedup on available RISC-V boards.

**G15. LOOPer learned polyhedral autoscheduler.** Transformer cost model over
loop-nest features drives Pluto/Tiramisu transformations; covers
non-rectangular domains. 1.84× over Tiramisu / 1.42× over Pluto on PolyBench.

**G16. CRT: LLM x86→RISC-V assembly transpiler.** Fine-tuned model achieves
88.68 % accuracy + 1.73× over Rosetta 2. Trained on (x86 bblock, RISC-V
bblock) pairs validated by unit tests. Can emit RVV when source used SSE/AVX.

**G17. LLM for LLVM pass selection.** 7B model on (IR, optimal `-O` sequence)
pairs; auxiliary heads predict pre/post inst-counts. 3 % better size reduction
than `-Oz` over a large suite.

**G18. LLM compiler feedback loop.** Re-prompt with measured inst-counts +
compilability after first pass. +0.53 % over single-shot. Spike/QEMU as cheap
oracle for `vsetvli` reorder legality.

**G19. RVV double-precision math library.** Cody-Waite range reduction via
`vfmsub`+`vfmul`, degree-12–15 Horner via `vfmacc`, 1–4 ULP at fp64. Competitive
with Sleef/libmvec on latency.

**G20. GEMV + dynamic quantization for LLM decode.** Quantize fp32 input to
INT8 in-register via `vfncvt.x.f.w` + `vssra`; INT8×INT8 GEMV with `vwmacc.vx`
widening to INT32; dequant per-block scale from prefetched fp32 vector. 38.3 %
avg GOPS improvement, peaking 56.3 % at 1024.

**G21. Clang vs GCC on RVV.** On SG2042 (C910, RVV 1.0), Clang 19 beats GCC
13.2 by 34 % token-gen / 25 % prefill on the same llama.cpp source. Cause: more
aggressive inlining + loop unrolling, exposing longer vector chains to
`vsetvli`. Flags: `-O3 -mcpu=thead-c910 -flto -fvectorize`.

**G22. NUMA `--interleave=all` on SG2042.** Default NUMA balancing causes page
migrations during LLM inference; interleaving cuts peak bandwidth pressure
30–40 %. Combine with per-thread-group `--membind`.

**G23. Vectorized scan auto-vectorization.** Two-phase Hillis-Steele
(intra-chunk vslideup doubling; inter-chunk scalar carry). Wired as new
`RecurrenceVectorizer` pass. 17× on prefix-sum.

**G24. MLIR FFT via linalg dialect.** Butterfly as `linalg.generic` over
complex; `transform.tile` exposes register tile; lower through `vector` to RVV.
`vector.shuffle`→`vrgather` for bit-reversal; `vector.fma`→`vfmacc`.
Graph-coloring keeps accumulator regs in distinct LMUL groups.

**G25. TinyIREE heterogeneous HAL.** Unified `func.func` + `memref` dialects
target RVV/Zve32x/Zbb; per-LMUL specializations compiled in; runtime VL query
selects variant. Single binary across 256/512/1024-bit VLEN.

**G26. Register dispersion in LLVM RegAlloc.** Modified allocator tags preferred
physical reg slots per LMUL group so logically consecutive registers map
non-contiguously across VRF banks.

**G27. Backporting RVV 0.7.1 ↔ 1.0 assembly.** LLVM-MC-based translator handles
renamed instructions, `vsetvl` encoding, segment-load operand order,
draft-only `vamoswap`/mask-pos changes.

**G28. Template-based GEMM micro-kernel generator.** Parameterize (MR, NR, LMUL,
SEW) where MR×NR is the register-tile. Single template covers SVE + RVV;
matches hand-tuned BLIS on ARM Carmel.

**G29. Per-layer GEMM parameter tuning for BERT.** 3-parameter (MC, KC, NC)
lookup-table trained offline per board. 2.5× over OpenBLAS on RISC-V.

**G30. AutoFDO / PGO for vectorization hints.** `perf record -b` (works on
P670, C908). `-fprofile-use` lets the vectorizer see actual trip-count
distributions; mark cold tails `__builtin_unpredictable(0)`; promote hot scalar
loops with `__builtin_expect`.

**G31. Inastemp-style C++ template wrapper.** Architecture-neutral types
(`rvv_float<N>`, `rvv_int<N>`) with operators that emit `vmerge.vvm` or masked
arithmetic for `if_else`. Lets domain scientists write numerics in portable
form.

**G32. LLVM IR as pivot for cross-arch translation.** Compile to IR with `-O2`,
train seq2seq on (IR, RVV-annotated asm). IR removes syntactic noise and
standardizes control flow; +11 % over source-level NMT.

**G33. ML cost model for MLIR fusion/tile.** Train on (`linalg` op seq, measured
latency); predict CPU util, inst count, reg pressure at the dialect level before
LLVM lowering. Drives fuse-or-not and tile-size decisions.

**G34. 5G/LTE baseband with RVV intrinsics.** `vint16m4` accumulators for FIR;
`vrgather.vv` for QAM-64/256 constellation demap; `vmslt.vx` + `vcompress.vm`
for zero-trellis pruning in LDPC. Matches DSP-ISA performance.

**G35. Probabilistic loop-bound propagation for AVL specialization.** Abstract
interp or profiling determines a loop's trip count is always a multiple of
`vlmax`; insert `__builtin_assume(n % vlmax == 0)` to skip the remainder loop.
On in-order cores a 1–3 element residue can cost as much as the vectorized
body.

---

## Build-flag cheat sheet

Clang ≥ 19 for RVV 1.0 hot code:

```bash
-O3 -march=rv64gcv_zba_zbb_zfh \
-mcpu=spacemit-x60   # or thead-c910 / sifive-x280 / generic-rv64
-flto -fvectorize -funroll-loops=4 \
-ffast-math          # if numerics tolerate
```

For per-microarch hints:

```bash
-mrvv-vector-bits=zvl256b   # K1
-mrvv-vector-bits=zvl512b   # X280
```

Mixed compiler trick (see `nn-llm.md` I33): Clang 19 for most TU, Xuantie GCC
10.4 for kernels using vendor-specific extensions.

## Routing inside this file

| Task | Ideas |
| --- | --- |
| Reduce `vsetvli` count | G1, G2, G21 |
| Auto-vectorize a scan / recurrence | G3, G23 |
| Autotune kernels | G4, G5, G29 |
| Port from SVE / NEON / SSE2 | G7, G8, G16, G32 |
| Single binary across VLENs | G11, G12, G25 |
| MLIR / IREE / TVM pipeline | G4, G6, G24, G25, G33 |
| PGO / autoFDO | G30 |
| RISC-V 0.7.1 ↔ 1.0 migration | G27, see also `microarch.md` H7 |
