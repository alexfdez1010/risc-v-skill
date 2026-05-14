# I — Neural Networks & LLM Inference / Training

35 ideas for transformer / CNN / SSM / MoE kernels on RVV 1.0. Primary targets
K1/X60, SG2042/SG2044, X280, Ara2.

---

## Attention — Flash-style & decoding

**I1. Flash-Decoding.** KV-length split across P thread groups; each computes
local (max, partial_sum, partial_out); final merge via log-sum-exp identity.
Converts decode-phase GEMV (Q-length=1, KV up to 128K) into a parallel
reduction on many-core RISC-V.

**I2. Two-level FlashAttention tiling for prefill.** Outer Q-row tile across
cores; inner KV-block per FlashAttention-2. Per-core L2 ≤ 128 KB; no
cross-core sync during accumulation. Reported 2.9–3.0× over baseline llama.cpp
on SG2042.

**I3. Fused RoPE in QKᵀ micro-kernel.** Apply rotary embedding to Q/K
in-register before the dot product. Precompute sin/cos tables; load two
consecutive lanes, `vfmul` by cos entry, `vfnmsub` by rotated partner,
accumulate directly into the QKᵀ tile. Saves one full Q+K pass per layer.

**I4. GQA with broadcast K/V loads.** With G query heads sharing one K/V head,
load K/V once and broadcast via `vmv.v.v` or constant `vrgather`. Interleave
G independent FMA chains, reduce at end. LLaMA-3 uses G=8 → ~8× K/V load
saving.

**I5. MLA low-rank KV reconstruction.** Store compressed latent c_KV of
dim d_c ≪ H·d_head. Reconstruct K/V on decode by multiplying c_KV by learned
W_K/W_V; fuse into the QKᵀ dot product in one vector pass. DeepSeek-V2/V3
reports 93.3 % KV reduction. Short-fat GEMMs favour LMUL=m2–m4.

**I6. Ring attention (pass-Q variant).** Multi-node ring: each node keeps
K/V shard, Q rotates. Smaller Q payload than pass-KV; reduces inter-node BW.
Each node runs standard FlashAttention block + ring-reduce log-sum-exp.

## KV cache management & quantization

**I7. INT2/INT4 asymmetric KV (KIVI).** Per-channel INT4 key, per-token INT2/4
value; scales in a side channel. Dequantize on-the-fly inside attention GEMV
via `vwmul.vx` + `vfcvt`. ~2.6× memory reduction → 4× larger batches.

**I8. Salient-token dense + bulk quantized KV (ZipCache).** Top 5–20 %
salient tokens in fp16; rest in INT4/INT2. Two-path load (fp16 + INT4) merged
via `vrgather`-indexed gather. 4.98× compression at 0.38 % accuracy drop.

**I9. Contiguous-virtual-memory KV (vAttention).** Skip PagedAttention's
non-contiguous physical pages; let the OS demand-page large contiguous virtual
KV regions. RVV uses standard `vle`/strided loads. Pair with hugepages
(`generic.md` C41). Reported up to 1.97× over PagedAttention.

**I10. Cross-layer KV delta coding (XQuant).** Base INT4 + delta INT2 between
adjacent layers; sub-1.4-bit effective. Reconstruct via in-register `vfadd`.
Best on deep nets (32–80 layers).

## Quantization — weights & activations

**I11. AWQ INT4 weight unpack + GEMV.** 8 INT4 weights per 32-bit word.
Unpack in-register via `vsrl` + `vand` masks; widen with `vzext.vf2`; INT8
MAC via `vmacc.vv`; per-group scale via `vmul` of a once-loaded scalar. Halves
DRAM reads for decode GEMV.

**I12. SmoothQuant offline activation smoothing.** Pre-migrate activation
outliers into weights offline so both sides INT8-quantizable. Runtime: INT8
activation via `vmulh` + `vnclip`; GEMM uses `vqmacc.vv`. INT-only path
avoids fp stalls on in-order X60.

**I13. SqueezeLLM dense+sparse decomposition.** INT3/4 uniform dense + fp16
sparse outlier (~0.45 % nnz, CSR). Dense via I11; sparse via `vluxei` over
sorted column indices.

**I14. QMoE sub-1-bit MoE codebook.** 8-bit codebook indices over a 256-entry
fp16 codebook (shared across experts). On decode, `vrgather.vv` against the
codebook vector reconstructs full-precision weights in-register before expert
GEMV.

## Mixture-of-Experts

**I15. Top-k gating via `vfredmax` + `vcompress`.** `vfredmax.vs` for max;
`vmslt` threshold; `vcompress.vm` packs surviving token-indices per expert.
O(T/vl) passes vs O(T·G) scalar.

**I16. Expert GEMM with fused token unpack + activation.** Pre-pack selected
token embeddings densely (from I15 output); run standard GEMM with fused
SwiGLU (I27); `vsuxei` scatters results back to original positions.

## State-space and linear-recurrence models

**I17. Mamba selective SSM scan.** Blelloch parallel scan inside a vl-wide
chunk: `vslidedown` at doubling strides + `vmul` + `vadd` for the associative
combine; log₂(vl) passes; up-sweep for finalization. Tile across sequence with
scalar inter-chunk carry. RVV analog of GPU warp scan.

**I18. RWKV wkv scan.** Channel-parallel `vfmul` + `vfmacc` + `vfredusum` per
time step; time-decay broadcast via `vfmv.v.f`; fuse division via `vfrec7` +
Newton. State C floats (≈ 8 KB for fp32, 2048-ch) stays in L1 across tokens.

## Speculative & parallel decoding

**I19. Medusa/Hydra fused draft heads.** Stack k draft-head weights
vertically; one batched `vfmacc` pass over hidden x produces k logit vectors.
For Hydra's sequential dependency add an autoregressive pass at the same
fused weight footprint.

**I20. Tree-attention verification mask.** Precomputed bitmask per spec step
stored in a vector register; `vmand` against the standard causal mask before
QKᵀ. Cheap and branch-free; EAGLE-2 reports 3.05–4.26× decode speedup.

## Prefill/decode disaggregation & scheduling

**I21. Sarathi chunked prefill + stall-free decode.** Split long prompts into
fixed chunks (e.g., 512 tokens); interleave one prefill chunk with one decode
batch per iter. Naturally pipelines compute-bound prefill with
bandwidth-bound decode on many cores. Assign prefill chunks to 32 cores,
decode to the other 32 on SG2042.

**I22. Ragged-batch attention with per-sequence `vsetvli`.** Iterate
sequences; per-seq `vsetvli vl=seqlen[i]`. Avoids padding to max_seqlen
(typical 30–50 % savings in mixed-length batches). `vsetvli` is 1–2 cycles on
X60.

## Sparse attention patterns

**I23. Sliding-window attention with `vslidedown` reuse.** Tile KV into
W-token panels; load first panel; slide registers by W per outer iter
(`vslidedown`); load only the new W elements. Zero-cost overlap shift for the
window.

**I24. Block-sparse attention with `vcpop`-guided loop.** Precompute block
adjacency bitmask; `vcpop.m` counts nnz blocks per row; loop only over those;
`vrgather` permutes selected K/V blocks into contiguous registers.

## MLP sparsity

**I25. DejaVu contextual sparsity.** Lightweight predictor MLP outputs bitmask
of near-zero FFN outputs. Run masked FFN GEMV: `vmseq` + skip all-zero rows
(or `vmerge` zero on inactive neurons + `vcompress` to dense output). Up to 2×
on OPT-175B; ~5 % predictor overhead.

**I26. PowerInfer hot-neuron preloading.** Profile activation frequency;
classify ~5 % "hot" neurons, pack contiguously into L2/L3; cold neurons stay
DRAM-resident with Zicbop-prefetched conditional load. Power-law activation
in OPT/LLaMA fits this well.

## Activation functions

**I27. SwiGLU fused gate-multiply.** Two GEMVs (W_up, W_gate) share input x;
load rows of W_up + W_gate alternately; two independent `vfmacc` chains
(acc_up, acc_gate); apply `vfmul` + sigmoid in-register. Sigmoid via `vfrec7`
of (1 + Horner-exp(-x)).

**I28. GELU with `vfsgnjx` + Horner + `vmerge` fallback.** For |x|>4 fall to
ReLU (`vfmax 0`); for |x|≤4 degree-5 minimax via chained `vfmacc`. `vfsgnjx`
extracts the sign bit for the half-plane dispatch. <0.5 ULP at 5 `vfmacc` +
2 `vmerge`.

## Winograd convolution extension

**I29. NOVA F(8,3) with rational interpolation points.** Standard F(8,3)
condition number κ ≈ 2×10⁵, unusable in fp16. NOVA's fractional points (±5/6,
±7/6, ±3/5) reduce κ by ~172 484×. Transforms map to 16–24 `vfmacc` per tile;
Hadamard product is one `vfmul` chain. 6.25× arithmetic reduction over direct
3×3 in fp16. Requires Zvfh/Zvfbfwma.

## Compilation, codegen, runtime

**I30. ifunc dispatch for Q4_0 / Q8_0 / fp16 weight formats.** Single binary;
resolver checks `HWCAP_RVV`, `HWCAP_ZVL256B`, and the gguf format flag at load
time. No runtime branching in the hot loop. Deployment-layer extension of
`generic.md` A9.

**I31. TVM Relax symbolic-shape RVV schedule.** Symbolic shape annotations let
the schedule parameterise loops on runtime `vl`; one kernel binary covers all
VLENs. Autotune LMUL, unroll, prefetch as scalar schedule params.

**I32. SG2042 NUMA policy + 32-thread cap.** Disable NUMA auto-balancing;
`numactl --interleave=all`; cap threads at 32–48 even on 64 cores (BW saturation
gives diminishing returns past that). See also `microarch.md` H18.

**I33. Mixed-compiler ggml build (Clang 19 + Xuantie GCC).** Most TUs through
Clang 19 for better inlining + RVV vectorization; vendor-extension kernels via
Xuantie GCC 10.4; link both. Flags:
`-march=rv64gcv_zba_zbb_zfh -O3 -ffast-math -funroll-loops=4`.

## Advanced attention architectures

**I34. MLKV cross-layer KV sharing.** Keep K_L/V_L in L2 (`cbo.clean` not
`cbo.flush`) and reuse for layer L+1; turns a bandwidth-bound layer into a
compute-bound one. Requires model trained or fine-tuned for MLKV.

**I35. RotateKV: Hadamard rotation + INT2 KV.** Walsh-Hadamard transform on
K's channel dim spreads outliers uniformly so uniform INT2 quant is safe.
FWHT decomposes to log₂(d_head) butterfly stages (`vslideup` + `vsub` +
`vadd`). Apply FWHT to Q on decode so rotation cancels; INT2 GEMV via
`vwmul`. <0.3 perplexity at 3.97× peak memory reduction.

---

## Routing inside this file

| Phase / structure | Ideas |
| --- | --- |
| Prefill (compute-bound) | I2, I3, I21, I22, I27 |
| Decode (bandwidth-bound) | I1, I4, I5, I20, I22 |
| KV quantization | I7, I8, I10, I35 |
| Weight quantization | I11, I12, I13, I14 |
| MoE | I15, I16 (and I14) |
| SSM / RNN | I17 (Mamba), I18 (RWKV) |
| Speculative decoding | I19, I20 |
| Sparse / windowed | I23, I24, I25, I26 |
| Compilation / runtime | I30, I31, I32, I33 |
| Pair with `microarch.md` | H11, H18, H20, H31 |
