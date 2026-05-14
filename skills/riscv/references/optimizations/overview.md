# RVV 1.0 Optimization Catalog — Overview

275-idea catalog for tuning RVV 1.0 kernels. Sourced from peer-reviewed papers
(2007–2026) and open-source kernels. Split by topic — load only what current task
needs.

## Hardware reference targets

| Board / Core | VLEN | Pipeline | Notes |
|---|---|---|---|
| SpacemiT K1 / X60 | 256 | in-order, dual-issue | BPI-F3, 8 cores, RVA22 + Zfh/Zvfh/Zba/Zbb/Zicbom/Zicboz/Zicbop |
| SiFive X280 | 512 | in-order | HiFive Premier; supports Xsfvqmaccqoq INT8 GEMM ext |
| SiFive P670 | 256 | OoO | desktop-class |
| T-Head C908 / C910 | 128–256 | in-order | LicheePi 4A; C906 on D1 is RVV 0.7.1 (Xtheadvector) |
| Sophon SG2042 | 128 | in-order | 64 cores, 4 NUMA clusters, RVV 0.7.1 |
| Sophon SG2044 | 256 | in-order | 64 cores, RVV 1.0, improved memory subsystem |
| Ara2 (academic) | 1024–2048 | long-vector | high-VLEN reference design |
| Vitruvius+ (EPI EPAC) | 16384 | decoupled scalar/vector | OVI interface |
| Spatz (MemPool) | small | cluster of narrow vectors | energy-optimised |

## Topic files

- `generic.md` — base 56 ideas across vectorization, numerics, memory, ILP,
  parallelism, domain kernels. Always-applicable RVV idioms (LMUL sweep, tail
  policy, segmented loads, prefetch, FMA chains, online softmax, etc.).
- `compiler.md` — G1–G35. LLVM/GCC backends, MLIR/IREE/TVM autotuning, SVE→RVV
  and NEON→RVV port, LLM-driven codegen, PGO, MLIR cost models. Load when
  configuring build flags, writing intrinsic wrappers, or porting from another
  SIMD ISA.
- `microarch.md` — H1–H32. K1 vsetvli stalls, VRF banks, V-LSU prefetch, Ara2
  long-vector tuning, Xtheadvector→RVV 1.0 shim, CARM roofline, PMU profiling,
  DVFS, autotuning, NUMA, hugepages, io_uring, SCHED_FIFO. Load when measuring,
  tuning per-board, or chasing the last 10–30 %.
- `nn-llm.md` — I1–I35. FlashAttention, Flash-Decoding, GQA/MLA, KV quant
  (KIVI, ZipCache, RotateKV, XQuant), AWQ/SmoothQuant/SqueezeLLM, MoE
  dispatch, Mamba/RWKV scans, speculative decoding, sparse attention, DejaVu,
  SwiGLU/GELU, Winograd F(8,3) NOVA, V-Seek SG2042 recipes.
- `crypto-codecs.md` — J1–J35. Zk* AES/SHA/SM3/SM4/GHASH, Zvk* vector crypto,
  Zb* bitmanip, post-quantum NTT (Kyber/Dilithium), bitsliced AES,
  BLAKE3/xxHash3/CRC32C, LZ4/zstd/Huffman/Parquet/simdjson/UTF-8, hash join,
  packet classify, JPEG IDCT.
- `scientific.md` — K1–K40. Sparse LA (SELL-C, HYB, SpGEMM, SpTRSV), stencil
  (diamond, space-time, pencil), CFD (LBM, Godunov, AMR), FEM
  (sum-factorisation, multigrid, Chebyshev), Krylov (CA-CG, pipelined CG,
  BiCGStab), FFT (Stockham, Bluestein), N-body (FMM M2L, Barnes-Hut), MD (LJ,
  PME), Monte Carlo, BFS/triangle, double-double, BLIS GEMM, batched ERI,
  Morton/octree.
- `alignment.md` — L1–L46. Striped Smith-Waterman (Farrar/Zhao), Wozniak
  anti-diagonal, SWIPE inter-sequence, WFA, ksw2, edlib (Myers bit-parallel),
  Suzuki-Kasahara difference recurrence, libgaba adaptive band, HMMER3
  MSV/Viterbi, PSSM, Pair-HMM, block aligner, Hirschberg traceback, GASAL2,
  K1/X280/Ara2 hardware-specific tuning. The most worked-out topic; useful
  template for "how to think about RVV for a domain."

## Top-priority quick wins on SpacemiT K1

Order is approximate — touch every kernel as early as possible.

1. `vsetvli` hoisting + LMUL sweep — `generic.md` §A2, §A3; `microarch.md` H1, H10.
2. Zicbop `prefetch.r` ahead of inner load — `generic.md` §C17; `microarch.md` H3.
3. FMA dual/quad accumulator unroll — `generic.md` §D22, §D23.
4. Segmented loads (`vlseg`) for AoS / `vfrec7` + Newton for transcendentals —
   `generic.md` §A4, §B13.
5. Compiler: prefer Clang ≥ 19 with `-O3 -march=rv64gcv_zba_zbb_zfh -flto` —
   `compiler.md` G21; `nn-llm.md` I33.

## Cross-topic routing

| Task | Files to load |
|---|---|
| Authoring a new kernel | `template.md` + `generic.md` + topic file |
| Squeezing the last % on K1 | `microarch.md` + `generic.md` §D |
| Porting from SSE2 / NEON / SVE | `compiler.md` G7, G8 + `alignment.md` L36 |
| LLM inference (llama.cpp / ggml) | `nn-llm.md` + `microarch.md` H11, H18, H31 |
| Crypto / codec / DB | `crypto-codecs.md` + `compiler.md` G7 |
| HPC / CFD / FEM | `scientific.md` + `microarch.md` H2, H4, H17 |
| Sequence alignment / bio | `alignment.md` + `generic.md` §F53 |
| Build / config / autotune | `compiler.md` G4, G13, G21 + `microarch.md` H10, H22, H24 |
| Profiling, regression gates | `microarch.md` H8, H9, H30 |

