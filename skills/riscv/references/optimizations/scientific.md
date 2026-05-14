# K — Scientific / HPC Kernels

40 ideas for sparse LA, stencil/CFD, FEM/multigrid, Krylov solvers, FFT, N-body,
MD, Monte Carlo, graph analytics, high-precision arithmetic, dense BLAS,
tensor contractions, climate, octree.

---

## Sparse linear algebra

**K1. SELL-C-σ SpMV with `vluxei` gather.** Row blocks of width C sorted by
nnz within σ-blocks. `vluxei32/64` gathers x by column index; `vfmacc.vv` inner
product; `vfredusum.vs` across slice. K1: C=8, σ=8. X280: C=16. Variable-length
register absorbs the irregular tail via slice-matching `vsetvli`.

**K2. HYB-format SpMV (ELL + COO tail).** ELL fast path with strided
`vlse32.v` + `vfmacc.vv`; COO residue handled scalar. Split rows by 90th-pctile
nnz; keeps the hot 90 % vectorised, scalar tail is negligible.

**K3. SpGEMM outer-product with `vcompress` merge.** Per non-zero (i,k) of A:
gather B[k,:] via `vluxei32`, MAC into partial C[i,:], `vcompress.vm` to compact
between updates. Hash buffer of size O(n_cols/VLEN) per active row in L1.

**K4. SpTRSV via SIMD-level wavefront.** AMD / MC64 reorder exposes level
sets; within a level, vectorise across rows: `vluxei32` gathers solved
x-values, `vfnmsac.vv` subtracts off-diagonal, scalar divide. ~2.7× over
unordered SpTRSV.

## Stencil

**K5. 3D diamond tiling.** Combine t and x into a skewed diamond tile;
innermost x at full RVV width with `vle32` + `vfmacc`; halo via `vslide1up`
/ `vslide1down` keeps data in registers. One `vsetvli` per diamond row.

**K6. Space-time trapezoidal tiles + Zicbop.** Gauss-Seidel/SOR honours the
dependency cone. Double-buffer: vector unit runs tile T while `prefetch.r` 4
cache lines ahead for tile T+1. Per-core L2 keeps working set bounded.

**K7. Cache-oblivious pencil-sweep on Ara2.** Pencil along fast-varying axis;
at VLEN=1024 the entire pencil fits in `m8`. Recursive two-level (z then x)
split; coefficient broadcast via `vfmv.v.f`. No per-VLEN retuning.

## CFD

**K8. LBM D3Q19 streaming with `vlseg`-like AoS layout.** SoA of 19 velocity
planes; pull-streaming gather as 19 `vle32` loads from contiguous neighbours
(achievable with face-first layout); BGK collision is pure `vfmacc` over the
spatial batch. Near-bandwidth-bound.

**K9. Godunov Riemann solver branch-free.** Vectorise across cell faces;
replace char-decomposition branches with `vmerge.vvm` + `vmslt.vf`. HLLE / Roe
flux entirely in masked vector arithmetic. ~2× vs scalar.

**K10. AMR ghost-cell exchange via `vluxei` + `vsuxei`.** Compact index map;
`vluxei32` gathers parent values, interpolate with vector FMA, `vsuxei32`
scatters into child ghost layer. Pre-sort indices for cache friendliness.

## FEM & multigrid

**K11. Matrix-free FEM sum-factorisation.** Hex-mesh degree-p operator factors
into p+1 1D GEMM sweeps. Vectorise rows with `vle32` + `vfmacc`; broadcast
scalar input via `vfmv.v.f`; fuse 3D directions into one kernel.

**K12. BSR-4 coarse-grid operator.** On the coarse grid, fixed 4-block CSR
loaded via `vlseg4e32.v`; `vfmacc.vv` for the block GEMV. Removes per-element
index computation — ~2× compute-to-memory ratio.

**K13. Chebyshev smoother fused SpMV + AXPY.** Single streaming pass: SpMV +
`vfmacc.vf` + scalar scale. Halves DRAM traffic vs separate calls; pipeline
with Zicbop prefetch for the next A tile.

## Krylov

**K14. s-step CA-CG with polynomial matrix-power kernel.** Compute A¹r,
A²r, …, Aˢr simultaneously via Newton/Chebyshev recurrence; store the s basis
vectors as a block. Each SpMV uses the K1 RVV kernel. TSQR-style block dot
products via `vle32` + `vfmacc.vv`.

**K15. Pipelined CG, fused dot + SpMV.** Dot product of iter k overlaps SpMV
of iter k. Fuse `vfmacc.vv` + `vfredusum.vs` into the SpMV gather body so both
share the x-vector load; one scalar all-reduce.

**K16. BiCGStab two-vector batched dot product.** Compute both dot products
in one pass: load chunk of x via `vle32`, two `vfmacc.vv` with different RHS,
accumulate to two reduction regs. Halves x-read bandwidth.

## FFT

**K17. Stockham FFT with `vrgather` butterflies.** Out-of-place auto-sort;
twiddle factors at each stage form a periodic pattern → encode one period as
a vector register; `vrgatherei16.vv` permutes into butterfly positions.
Butterfly = two `vfmacc.vv` + two `vfnmsac.vv` for the complex FMA.

**K18. Real-to-complex FFT with `vslidedown` folding.** After N/2 complex
FFT, fold remaining N/2 values via conjugate-symmetry: `vslidedown.vx`
reverses, `vfneg` for imag, `vmerge` combines. No full complex array
materialised.

**K19. Bluestein chirp-Z FFT for prime length.** Maps arbitrary-N DFT to three
length-M ≥ 2N−1 FFTs. Chirp multiply vectorises as two `vfmacc.vv` /
`vfnmsac.vv` pairs over a precomputed chirp vector. Reuses Stockham (K17).

## N-body / FMM

**K20. FMM M2L kernel.** Dense (p+1)²-sized complex GEMV per box pair; one
`vfmacc.vv` per row with `vle32` operator load; unroll 4 rows for X60
dual-issue. Pre-translate M2L operators into contiguous arrays at level build.

**K21. Barnes-Hut MAC test vectorised.** Load 8 candidate cells' (centre,
half-extent) via `vlseg4e32.v`; r² via vector FMA; compare vs (d/θ)² with
`vmslt.vf`; `vcpop.m` counts accepted cells; branch on scalar count to split
into near/far lists.

## Molecular dynamics

**K22. LJ force kernel with verlet-list gather.** `vluxei32.v` gathers
neighbour {x,y,z}; three `vfmacc` produce r²; `vfrec7` + two Newton iters for
r⁻⁶; `vfmacc.vv` for the 12-6 force. Newton-3 via `vsuxei32` scatter to
neighbour force array.

**K23. PME direct-space Coulomb.** Degree-5 erfc polynomial via Horner/Estrin
`vfmacc` chain; switch-multiply by smooth cutoff; broadcast β via `vfmv.v.f`
once per tile. Avoids non-pipelined `vfdiv` / `vfsqrt`. Same gather pattern as
K22.

## Monte Carlo

**K24. Per-lane xorshift PRNG.** 8 (m1, K1) or 16 (m2, X280) PRNG state lanes
in vector regs. `vxor.vv` / `vsrl.vx` / `vsll.vx` / `vxor.vv` advances all
lanes per step. fp output via `vfmul.vf` against 2⁻³².

**K25. Metropolis accept/reject via `vmslt` + `vcompress`.** Per batch:
ΔE via force-field eval; exp(−ΔE/kT) via Estrin (K30); compare vs uniform
random (K24) with `vmslt.vf`; `vcompress.vm` packs accepted positions;
`vcpop.m` returns accept count.

## Graph analytics

**K26. Direction-optimising BFS with SELL-C bitset.** Bottom-up phase:
bitset frontier; `vluxei8.v` gathers adjacency bitset; `vand.vv` with
frontier; `vcpop.m` detects active neighbours in one pass.

**K27. Triangle counting via vector set-intersection.** For edge (u,v),
intersect sorted N(u) ∩ N(v): `vle32.v` of next N(u) chunk; broadcast current
N(v) elem via `vfmv.v.f`; `vmseq.vx`; `vcpop.m` matches. Advance the smaller-max
pointer.

## High precision

**K28. Double-double via dual-accumulator FMA.** Bailey DD pair (hi, lo) with
`vfadd.vv` for hi + Knuth two-sum (`vfsub.vv`, two `vfsub.vv` Dekker
splitting); DD multiply via `vfmul.vv` + `vfmsac.vv` residual. Hi and lo in
separate registers (no interleave).

**K29. Interval arithmetic with `vfmin`/`vfmax`.** [a,b] pair as two
fp32/fp64 vectors. `vfadd.vv` per bound with directed-rounding `frm` swap
between calls (exact) or `vfsub.vf(ulp)` (approx, faster). Half lane
utilisation.

## Mixed precision & numerical robustness

**K30. Estrin polynomial evaluation.** Evaluates degree-D polynomial in
log₂(D) FMA depth by pairing terms; broadcast coeffs via `vfmv.v.f`. On X60's
4-cycle FMA, D=7 drops critical path from 28 to 12 cycles → 2.3× over Horner.

**K31. GMRES-IR3 mixed precision.** Factor in fp16/bf16; iterate fp32 solve
with fp64 residual computation (`vfmacc` widening). Residual SpMV uses K1 at
SEW=64; correction SpMV uses Zvfh SEW=16. Near-fp16 throughput, fp64 accuracy.

**K32. Kahan-compensated FEM assembly.** Per-lane error compensator
alongside accumulator: `vfadd.vv` + `vfsub.vv` recovery. 2 extra FP ops per
step; bounds error at O(eps) regardless of N. Cheaper than fp64 accumulation
at 2× memory cost.

## Dense BLAS-3

**K33. BLIS-style GEMM micro-kernel.** X280 (VLEN=512, 16 fp32 lanes): 6×8
register tile — 6 row regs for C rows; 2 for packed A column (broadcast via
`vfmv.v.f`); 1 for B row; inner loop is 6 `vfmacc.vf`. Outer loops pack into
mc×kc (L2) and kc×nc (L3) via `vsseg` stores. K1 (VLEN=256): 4×4 variant.

**K34. Tasked Cholesky (PLASMA-style).** Diagonal tile via scalar/small BLAS;
off-diagonal updates via RVV DSYRK (DGEMM with symmetric mask via `vmerge`)
and DTRSM (dense wavefront like K4). DAG dispatch via OpenMP `depend`.

## Tensor contractions

**K35. In-register `vrgather` permutation + GEMM.** For small index
permutations (d≤VLEN), implement as `vrgather.vv` instead of out-of-place
transpose. Fuse with BLIS micro-kernel (K33).

**K36. Batched small-GEMM (ERI).** B copies of A and B packed into one wide
vector; single 16-iter `vfmacc.vv` runs B = VLEN/16 simultaneous 4×4 GEMMs.
Amortises `vsetvli` + packing over B GEMMs.

## Climate / atmospheric

**K37. ICON horizontal stencil via `vlseg3`.** (u,v,w) triplets via
`vlseg3e32.v`; stencil weights broadcast with `vfmv.v.f`; scatter tendencies
via `vsseg3e32.v`. Matches BW limit.

**K38. Spectral-element Legendre transform.** (p+1)×(p+1) Vandermonde as
GEMV: `vle32` for matrix row, `vfmacc.vv` for dot, modal coeffs broadcast.
Forward + inverse fused (sum-factorisation).

## Octree / Morton

**K39. Vectorised Morton-code generation.** 3D positions → Z-order codes:
`vsll.vx` (1/2/3-bit offsets) + `vor.vv` chains compute all three
bit-interleave expansions in parallel. 12 vector int ops produce VLEN codes
per iter. Precedes radix sort.

**K40. Octree-leaf force accumulation with `vfirst.m`.** 8 candidate child
cells per batch; `vfirst.m` detects first non-null cell; `vluxei32.v` gathers
its multipole; vector FMA accumulates; `vmslt.vf` masks beyond-cutoff cells.

---

## Routing inside this file

| Pattern | Ideas |
| --- | --- |
| Sparse LA | K1, K2, K3, K4 |
| Stencil / structured CFD | K5, K6, K7, K8 |
| Unstructured CFD / AMR | K9, K10 |
| Matrix-free FEM | K11, K12, K13 |
| Krylov solvers | K14, K15, K16 |
| FFT | K17, K18, K19 |
| N-body / FMM | K20, K21, K40 |
| MD | K22, K23 |
| Monte Carlo | K24, K25 |
| Graph | K26, K27 |
| Precision tooling | K28, K29, K30, K31, K32 |
| Dense BLAS | K33, K34 |
| Tensor contractions | K35, K36 |
| Climate | K37, K38 |
| Spatial data structures | K39, K40 |
