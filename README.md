# RISC-V Skill

RISC-V Vector (RVV 1.0) C intrinsics skill plus a 275-idea optimization catalog,
published via [skills.sh](https://skills.sh).

## Using the skill

Install it into your agent with the [skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add alexfdez1010/risc-v-skill
```

Works with Claude Code and other supported agents.

## What you get

- RVV 1.0 intrinsics reference: VL/AVL strip-mining, LMUL/SEW, tail/mask
  policies, contiguous / strided / segmented / gather loads, reductions,
  softmax, NTT, CRC, perf counters.
- 275 optimization ideas, split by topic, condensed into one-bullet
  What / Why / Where entries — covering compilation, microarchitecture,
  NN/LLM, crypto/codecs, scientific computing, and sequence alignment.
- Worked kernels (vec add, transpose, softmax, NTT, reduction, CRC,
  microbenches) and an authoring template.

## Layout — progressive disclosure

Only `SKILL.md` is always loaded. References are pulled in on demand based on
the task. Routing rules live in `SKILL.md` §Loading Guidance.

- `skills/riscv/SKILL.md` — entry. Scope, quick template, routing tables.
- `skills/riscv/references/core-concepts.md` — VL/AVL, VLMAX, tail/mask
  policies, naming, memory access, reductions, conversions, widen/narrow,
  masks, bit ops, config macros.
- `skills/riscv/references/patterns.md` — algorithmic templates: vector add,
  transpose, reduction, softmax, NTT, CRC, microbenches, perf counters.
- `skills/riscv/references/intrinsics.md` — cheat sheet by category.
- `skills/riscv/references/examples.md` — 7 worked kernels with host/kernel
  interaction.
- `skills/riscv/references/template.md` — step-by-step new-kernel guide.
- `skills/riscv/references/glossary.md` — AVL, VL, VLMAX, SEW, LMUL,
  tail/mask policies, Zvbc.
- `skills/riscv/references/optimizations/` — 275-idea catalog, split by topic
  with citation-free bodies plus a single bibliography file:
  - `overview.md` — hardware targets, top quick-wins, cross-topic routing.
  - `generic.md` — base 56 ideas (A–F): vectorization, numerics, memory
    hierarchy, ILP, parallelism, domain kernels.
  - `compiler.md` — G1–G35: LLVM/GCC, MLIR/IREE/TVM, SVE/NEON port,
    LLM-driven codegen, PGO.
  - `microarch.md` — H1–H32: K1/X280/SG2042/Ara2 tuning, PMU + roofline,
    DVFS, NUMA, hugepages, io_uring, autotuner persistence.
  - `nn-llm.md` — I1–I35: FlashAttention, GQA/MLA, KV quantization, AWQ /
    SmoothQuant / SqueezeLLM, MoE, Mamba/RWKV, speculative decoding, sparse
    attention, NOVA Winograd, V-Seek recipes.
  - `crypto-codecs.md` — J1–J35: Zk* / Zvk*, Zb*, post-quantum NTT,
    bitsliced AES, BLAKE3 / xxHash3 / CRC32C, LZ4 / zstd / Huffman / Parquet
    / simdjson / UTF-8, hash join, packet classify, JPEG IDCT.
  - `scientific.md` — K1–K40: sparse LA, stencil/CFD, FEM/multigrid,
    Krylov, FFT, N-body, MD, Monte Carlo, BLIS GEMM, batched ERI,
    Morton/octree.
  - `alignment.md` — L1–L46: striped Smith-Waterman, Wozniak anti-diagonal,
    SWIPE, WFA, ksw2, edlib, libgaba, HMMER3, PSSM, Pair-HMM, block aligner,
    Hirschberg.

The bibliography (DOIs, arXiv ids, repo paths) lives in
`REFERENCES.md` at the repository root — deliberately outside the skill so it
is never loaded into an agent's context.

## Source of optimization ideas

Most ideas in `skills/riscv/references/optimizations/` were mined using
[`paperhound`](https://github.com/alexfdez1010/paperhound) across RISC-V,
LLVM/MLIR, NN/LLM, crypto, HPC, and bioinformatics literature (2007–2026).
Six parallel `paperhound` sub-agents covered distinct topic clusters; output
was cross-checked against the kernels in libssw, parasail, ksw2, WFA2-lib,
edlib, llama.cpp, IREE, and OpenSSL Zvkned before each idea was condensed
into a What/Why/Where bullet. Full DOIs and arXiv ids are listed in
`REFERENCES.md`.

The base RVV intrinsics content in `SKILL.md` is built on the RVV mini-course
at <https://github.com/nibrunie/rvv-examples>.

## License

MIT — see `LICENSE`.
