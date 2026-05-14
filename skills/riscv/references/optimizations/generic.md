# Generic RVV 1.0 Optimization Ideas (Base 56)

Generic tactics applicable to most RVV kernels. Hardware baseline: RVV 1.0 +
RVA22, optional Zfh/Zvfh/Zba/Zbb/Zicbom/Zicboz/Zicbop. Reference cores: K1/X60
(in-order, dual-issue, VLEN=256), X280, C908, Ara2 (long-vector).

Numbering matches the upstream Tolaria note `risc-v-kernel-optimization-ideas`.

---

## A. Vectorization & RVV ISA exploitation

**A1. LMUL auto-tune per kernel shape.** Sweep m1/m2/m4/m8 vs problem size. Small
reductions favour m1 (less reg pressure, fewer bubbles). Large GEMM tails favour
m4/m8 (amortize per-iter overhead). Decide at codegen time.

**A2. Tail/mask policy specialization.** Default `ta,ma`. Use `tu` only when a
reduction must carry state in tail lanes. Spurious `tu` serialises the X60 back
end.

**A3. Strip-mining via `vsetvli`, hoisted.** One `vsetvli` per outer iteration;
reuse `vl` for the body. Avoids 2–4 cycle per-iter recompute on in-order cores.
Use whole-register moves to swap LMUL groups without re-issuing `vsetvli`.

**A4. `vlseg`/`vsseg` for AoS layouts.** RGB, complex, NHWC: `vlseg2..8e32`
replaces gather/scatter. ~4× over emulated gather on X60.

**A5. `vlse`/`vsse` strided for column stripes.** Skip pre-transpose for
tall-skinny GEMV; LSU pipelines stride cost. Pair with Zicbop prefetch.

**A6. `vluxei` indexed loads.** Embedding lookup, sparse softmax, sparse
attention. Keep index width ≤ data width.

**A7. Mask-driven control flow.** `vmseq`/`vmslt` + `vmerge` or masked
arithmetic instead of if-then-else inside vector loops. Big win on
activation kernels with data-dependent branches.

**A8. Whole-register moves (`vmv*r.v`).** Free register copies; avoid spilling
to L1. Compilers often miss when they could fuse a `vsetvli`-induced reshape
into a single move.

**A9. Custom-extension fast paths.** Detect Xsfvqmaccqoq (SiFive INT8 GEMM),
Xtheadvector, SpacemiT custom ops at build time. Ifunc resolver with a vanilla
RVV fallback for portable binaries.

**A30. `vrgather` in-register LUT.** Sigmoid/GELU coarse tables, byte-permutes,
AES sbox. Index in the vector register file; no LSU. `vrgatherei16` keeps the
index path narrow when data is wider.

**A31. `vcompress.vm` for sparse compaction.** Drop-zero, top-k, NMS, ReLU+pack
in one pass. Eliminates the scalar fallback the autovectorizer uses for
data-dependent output length.

**A32. `vslideup`/`vslidedown` for stencil + shift-register loops.** 1D conv,
prefix-sum, FIR: keep one panel in registers, slide instead of reload.
`vslide1up/down` handle per-iter boundary; pair with `m4` to amortize slide
latency.

**A33. `vcpop.m` / `vfirst.m` for masked counts and early-exit search.**
Non-zero-count, find-first-match, validation kernels. One vector instruction
per chunk instead of serial scalar reduction.

**A34. Long-vector chaining on wide-VLEN cores.** On Ara2 (VLEN≥1024) keep
AVL near architectural cap; stripe dependent ops so successive instructions
chain through the issue queue without stalling.

---

## B. Precision & numerics

**B10. Zfh / Zvfh half-precision lanes.** Activations, attention scores, norm
stats in fp16. Accumulate in fp32 with `vfwmacc`. Doubles SIMD lane utilization
for one extra conversion round.

**B11. BF16 GEMM via Zvfbfmin / Zvfbfwma.** Prefer when present (wider
exponent). Otherwise emulate with `vfwcvt` + integer shifts — still cheaper
than fp32→fp16. Critical for LLM range.

**B12. INT8/INT16 quantized NN.** `vqmacc.vv` for inner MAC, saturating
`vnclip` for re-quant. Per-channel scale hoisted via `vfmv.v.f`. 2–4× over fp32
on quantization-tolerant nets.

**B13. Range-reduced transcendentals.** Polynomial exp/tanh/sigmoid/gelu/mish
via Horner over chained `vfmacc`. Avoid `vfdiv`/`vfsqrt` (multi-cycle,
non-pipelined on X60). `vfrec7` + 1 Newton for `1/x`; `vfrsqrt7` for `1/sqrt`.

**B14. Fast softmax two-pass in fp16.** Fuse max-reduce / sub / exp / sum / div
into one streaming pass when the row fits in the register group. Eliminates a
tensor traversal vs naive three-pass.

**B35. Mixed-precision iterative refinement.** Heavy GEMM body in fp16/bf16; fp32
residual correction step. Near fp16 throughput with near fp32 accuracy.

**B36. Kahan / Neumaier compensated summation.** Per-lane error compensator
(`vfadd` + `vfsub` recovery) alongside accumulator. Two extra FMAs per step;
keeps O(ε) error on long dot products where `vfredosum` is still lossy.

**B37. Saturating arithmetic for fused activation clamping.** `vssadd`/`vssub`/
`vssra`/`vnclip` fold ReLU6 / hard-tanh / quant clamp into the MAC path. Removes
post-MAC compare-and-select.

**B38. Stochastic rounding for BF16 training.** Per-lane PRNG dither (xorshift
via `vxor`) before truncation. Avoids systematic round-to-nearest bias; lets
BF16 stand in for fp32 on more layers.

---

## C. Memory hierarchy

**C15. Goto-style GEMM tiling.** Five nested loops; mc×kc in L2, kc×nc in L3.
K1 starter: mc=64, kc=256, nc=1024 for fp32. Halve mc, double nc for fp16.

**C16. Pack A/B panels with `vsseg`.** Match panel LMUL to consuming
micro-kernel's LMUL so register groups line up without reshuffles.

**C17. Software prefetch via Zicbop (`prefetch.r/w/i`).** 4–8 cache lines ahead
of the inner loop. Critical on in-order cores — sweep prefetch distance per
microarch.

**C18. Cache-block flush via Zicbom (`cbo.clean`/`cbo.flush`).** After a producer
writes a tile the consumer will not re-read, flush to free L1 ways. Big win on
fused kernel chains.

**C19. Non-temporal stores (`cbo.zero` + bypass).** Initialize large output
buffers with `cbo.zero`; saves write-allocate traffic.

**C20. NHWC vs NCHW per layer.** Depthwise prefers NHWC (channel-vec).
Standard conv prefers `im2col`→GEMM in NCHW. Auto-select via cost model; fuse
layout transpose at chain boundaries.

**C21. `im2col`-free direct convolution.** 3×3 stride-1: vectorize across W
with `vlseg` over channel. Saves staging buffer; register-tile weights across
spatial positions.

**C39. Cache-oblivious recursive tiling.** Z-order / Hilbert traversal on the
output index space; recurse down to register-tile leaf. Robust when cache
hierarchy is unknown or asymmetric.

**C40. Compressed weight formats with on-the-fly decode.** INT4/INT8 packed
weights stream from DRAM; decode in-register via `vsra`+`vand`+`vfwcvt` (uniform)
or `vrgather` LUT (non-uniform codebook). Halves DRAM bw on weight-bound LLM
inference.

**C41. Hugepages for multi-MB GEMM panels.** Cuts dTLB misses on long
dot-products and packed-panel scans. K1 dTLB is small; a typical packed B
spans 100s of 4 KB pages. See `microarch.md` H26 for concrete recipe.

**C42. Block-cyclic distribution for irregular workloads.** Round-robin tile
groups across cores for SpMV, variable-seq-len attention, mixed-resolution
batches. Smooths imbalance without dynamic-scheduling tax.

---

## D. Instruction-level parallelism & scheduling

**D22. Manual unroll 4×–8×.** X60 is dual-issue in-order — unroll independent
FMA chains so the scheduler can overlap latency. Compiler chronically
under-unrolls at high LMUL.

**D23. FMA chain interleaving (multi-accumulator).** Two or four independent
accumulators; reduce at end with `vfredusum`. Halves dependency-chain latency.

**D24. Source-level instruction fusion.** Hand-fuse `bias_add → activation`,
`conv → bn → relu`, `mul → add`. Keep tensors in registers. Constant-fold BN
into preceding conv at load time.

**D25. `vfredusum` vs `vfredosum`.** Unordered where allowed; reserve ordered
for numerically sensitive layers (softmax denominator, large dot products).
Bit-exact reproducibility requires ordered.

**D26. Hoist `vsetvli` and broadcasts.** `vfmv.v.f` once per panel. Hoist
`vsetvli` to the outer loop where AVL is invariant. Both transforms are missed
by LICM when loops are complex.

**D43. Inline asm for schedule-critical inner loops.** Compilers mis-schedule
RVV around `vsetvli` boundaries and chain breaks. Hand-write the 20–40
instruction hot window with explicit reg allocation; wrap behind a clean
intrinsic-style API.

**D44. Alias/alignment hints (`restrict`, `__builtin_assume_aligned`).**
Unblocks overlap and alignment checks; removes versioned-loop guards.

---

## E. Parallelism & system

**E27. Static OpenMP/pthread tiling on M.** K1: 8 cores → split outer M, keep
K and N intra-core. Pin with `sched_setaffinity`; avoid dynamic schedules at
small core counts.

**E28. NUMA-aware allocation.** First-touch panels on consumer thread. SG2042
multi-cluster: remote-socket access ~2× local. See `microarch.md` H18, H32 for
concrete bindings.

**E29. Concurrent kernel pipelining.** Core *i* runs micro-kernel on tile T
while core *i+1* prefetches and packs tile T+1. Lock-free ring buffer between
producer/consumer — see `microarch.md` H25.

**E45. Work-stealing scheduler.** Sparse SpMV, variable-seq-len attention,
ragged-batch inference: per-core deques with stealing. Cheap when steals are
rare (good static partition + steal-on-empty).

**E46. Heterogeneous-cluster aware partitioning.** Mixed perf/eff clusters:
compute-bound micro-kernels on perf cluster, packing/IO/decompression on eff
cluster. Match work-units to per-cluster vector throughput.

---

## F. Domain kernels & algorithmic restructuring

**F47. Online (one-pass) softmax.** Milakov–Gimelshein running-max + running-sum
recurrence updated lane-wise. Halves DRAM traffic vs two-pass; building block
for FlashAttention. Detail in `nn-llm.md`.

**F48. Tiled fused attention (FlashAttention).** Block QKᵀ → online softmax →
PV in vector registers per tile; never materialize the full attention matrix.
See `nn-llm.md` I1–I3.

**F49. Winograd F(2,3) for 3×3 convolutions.** 2.25× fewer multiplies for more
adds and small constant transforms. Pre-transform weights at load. For F(8,3)
with FP16 see `nn-llm.md` I29 (NOVA rational interpolation points).

**F50. Strassen-style recursion for very large GEMM.** 7-multiply 2×2 split
recursed to register-tile leaf. K1 crossover ~1024×1024; pair with fp64
accumulation if stability matters.

**F51. Radix-2/4 FFT with `vrgather` butterflies.** Bit-reversal in one
`vrgather`; butterfly via `vfmacc` + `vfnmsac`. See `scientific.md` K17–K19 for
Stockham, Bluestein, real-to-complex variants.

**F52. Vectorized prefix-sum (scan).** Hillis–Steele within a register via
`vslideup` doubling stride; cross-register carry. Generalizes to any
associative reduction.

**F53. Striped DP layout (Farrar pattern).** Smith–Waterman, Needleman–Wunsch,
Levenshtein striped across vector lanes. Removes per-cell anti-diagonal
dependency. Deep treatment in `alignment.md`.

**F54. Sparse CSR SpMV via `vluxei` + segmented reduction.** Indexed gather of
x by column index, MAC against values, segmented reduction over row pointers.
See `scientific.md` K1–K4 for SELL-C, HYB, SpGEMM, SpTRSV.

**F55. Fused RMS-norm / LayerNorm with `vfrsqrt7` + Newton.** Streaming pass:
square-reduce → rsqrt seed → Newton refine → scale-shift. Replaces three
separate LLM kernels.

**F56. Top-k via masked partial sort + `vcompress`.** Running threshold; mask
candidates above it; `vcompress` survivors; repeat. Avoids full sort when
k ≪ n (sampling, beam search, retrieval).

---

## Quick-wins ranking (SpacemiT K1)

(blank line below preserved for lint cleanliness)

| Rank | Idea | Why |
| --- | --- | --- |
| 1 | A2/A3 vsetvli hoist + LMUL sweep | Touches every kernel; trivial |
| 2 | C17 Zicbop prefetch | In-order; biggest memory-stall payoff |
| 3 | D22/D23 dual-accumulator unroll | Typically doubles GEMM throughput |
| 4 | A4/C21 segmented load for NHWC | Removes im2col staging |
| 5 | B13 `vfrec7` + Newton transcendentals | Activation-heavy NN layers |
