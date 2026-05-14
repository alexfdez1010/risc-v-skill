# L — Sequence Alignment / Smith-Waterman

46 ideas for porting and tuning Striped Smith-Waterman (Farrar / Zhao) and
related pairwise sequence alignment kernels to RVV 1.0. Primary target
SpacemiT K1 / X60; secondary X280, C908, SG2042, Ara2.

This is the deepest-worked topic — also useful as a template for thinking
about RVV for a new domain.

---

## Core striped SSW (Farrar / Zhao)

**L1. 8-bit saturating striped SW.** Farrar layout at SEW=8 / LMUL=m4 using
`vsaddu.vv` + `vssubu.vv` + `vmaxu.vv` for H/E/F recurrence. 32 lanes on K1;
~10-instruction inner loop pairs naturally with K1 dual-issue.

**L2. 8 → 16-bit overflow fallback.** `vmsgtu.vx` + `vcpop.m` checks vMaxScore
vs 255−bias each row; on overflow restart at SEW=16/LMUL=m2. Avoids
permanent 16-bit penalty; only 1–5 % of protein alignments overflow.

**L3. Striped query-profile construction.** Precompute `P[residue][segment][lane]
= substitution[residue][query] + bias`. Load via `vle8` / `vluxei8`. Profile
fits in L1; reused across all DB sequences.

**L4. `vrgather` in-register profile lookup.** Hold per-residue profile
vectors in registers; pick scores per DB residue with `vrgather.vv`. 1-cycle
lookup, no L1 traffic. Ideal for 4-residue DNA.

**L5. Bias-encoded unsigned arithmetic.** Bias=128 so values lie in [128, 255];
enables 1-cycle `vmaxu.vv` instead of signed compare in hot loop.

**L6. Prefix-scan NW layout.** Diagonal-wavefront vectorisation; propagate
F-carry via O(log segLen) tree reduction (`vslideup`/`vslidedown`) instead of
Lazy-F. Faster than Farrar for global alignment.

**L7. Wozniak anti-diagonal vectorisation.** Vectorise along i+j=const where
cells are independent; load via `vlse8.v` stride=refLen+1. No intra-vector
stalls, no Lazy-F.

## Inter-sequence parallelism

**L8. SWIPE inter-sequence parallelism.** Pack N = VL/SEW DB sequences (32
lanes on K1) into one register; each lane runs an independent SW. ~6× over
Farrar striped for protein DB.

**L9. SWIPE 4-residue unroll.** Process 4 DB residues per query advance, with
4 sets of H/E/F in LMUL groups. Cuts L1 traffic.

**L10. Lazy-F early-termination.** Per-segment `vmsgtu.vx` + `vcpop.m`
short-circuits when no F-update exceeds H−gapO. Lazy-F dominates on long
queries; early-exit saves 50–90 % iterations.

**L11. `vredmaxu.vs` for per-column max.** Single instruction replaces
libssw's 9-instruction `max16` macro.

**L12. SEW=16 striped SW for long/affine.** 16-bit profile, `vsadd.vv` +
`vmax.vv` at LMUL=m2 (16 lanes on K1). Correct when qLen·maxScore > 255
without libgaba complexity.

## Difference recurrences (Suzuki-Kasahara / libgaba)

**L13. Suzuki-Kasahara difference recurrence.** Encode ΔH, ΔV, ΔE, ΔF (8-bit
fits) instead of absolute scores. Retains SEW=8/LMUL=m4 even for megabase
reads. ~2.1× over 16-bit; 4-op critical path.

**L14. libgaba adaptive banded DP.** Width-32 forefront vector that shifts
up/down based on which edge holds the max. O(n·W) regardless of divergence;
no out-of-band failures on noisy long reads.

## KSW2 variants

**L15. KSW2 banded extension with Z-drop.** Banded diagonal DP (±W) with
Z-drop early termination when score lags global max. >50 % of minimap2
extensions terminate early.

**L16. KSW2 dual-gap-cost affine.** Two affine gap models (E1/F1 short,
E2/F2 long); take min cost. Concave gap function fits SNPs + SVs in one pass.

## Wavefront / WFA

**L17. WFA wavefront alignment.** Extend score-indexed wavefronts diagonally;
match extension is bitwise/scalar; update is SIMD-friendly. O(ns) time;
20–300× over SW for low-divergence reads.

**L18. WFA adaptive heuristic + X/Z-drop.** Periodically prune lagging
diagonals, abort on X-drop/Z-drop. Cuts O(s²) memory; 2–5× on noisy long
reads.

## Block / global aligners

**L19. Block aligner (adaptive block shifts).** R×C SIMD block (e.g., 32×32)
greedily shifts/grows to track the score max. 5–10× over Farrar for protein
global; handles big gaps without banded restriction.

**L20. SWIPE temp-profile via `vrgather`.** Replace SSSE3 `pshufb` temp-profile
with a `vrgather.vv` over a vector-register substitution table. 1-instruction
32-lane lookup; no L1 pressure.

## VLA + microarch tuning

**L21. VLA-agnostic dynamic segLen.** `segLen = ceil(qLen / vlenb)` at
runtime; single binary across K1/X280/Ara2. Wider VLEN auto-shrinks segLen
and Lazy-F cost.

**L22. LMUL sweep on K1.** m1 (VL=32 at SEW=8) is optimal given 12-register
budget; m4 wastes scratch, m8 starves H/E/F/profile regs.

**L23. Zicbop prefetch of DB sequence.** `prefetch.r` 8–16 outer iterations
ahead of the reference loop. Hides L2 latency on in-order K1.

## Bit-parallel edit distance

**L24. Myers bit-parallel edit distance (edlib).** Each DP row as 64-bit
P/M bitvectors; update via AND/OR/XOR/add. 64× per reference position for
pure edit distance.

**L25. Hyyrö bit-parallel LCS / bounded edit.** Carry-word extension of Myers
for bounded affine. Hyyrö's `Vj[i]` == Suzuki-Kasahara ΔV — informs unified
8-bit recurrence design.

## Profile-HMM (HMMER3)

**L26. HMMER3 MSV filter (striped).** Farrar striped at SEW=8, only `vadd` +
`vmax`, no F / Lazy-F. Score capped at 8-bit; pure throughput.

**L27. HMMER3 Viterbi striped (SEW=16).** Full HMM Viterbi with M/I/D at
SEW=16; inter-state transitions via slides. RVV doubles SSE2 width.

**L28. PSSM lookup via `vrgather`.** Treat PSSM rows as striped profile;
`vrgather.vv` fetches 32 PSSM scores per DB column. Works as long as
20·qLen fits L1.

## Pair-HMM

**L31. Pair-HMM forward (GATK).** Striped Pair-HMM with M/I/D at SEW=32
float (or bf16 SEW=16); inner loop `vfmacc.vv` + `vfmax.vv`. Pair-HMM is
30–60 % of HaplotypeCaller; bf16 doubles throughput.

## Traceback / long-range

**L29. 2-bit traceback matrix.** Pack direction (none/diag/up/left) into 2-bit
fields, 32 cells/word. 4× smaller; keeps long alignments L1-resident; CIGAR
run-length via `vctz`.

**L30. Hirschberg linear-space alignment.** Find midpoint via forward + backward
half-DP (O(n) space), recurse on halves. Makes chromosome-scale alignment
feasible.

## Data packing & batching

**L32. 2-bit DNA packing + decode.** 128 bases per VLEN=256 reg via `vsrl.vi`
+ `vand.vi` decode. 4× DRAM bw reduction; 100bp read fits one register.

**L33. Query-profile reuse across DB hits.** Build once, pin to L1, iterate
over all DB seqs. Underpins VSEARCH/SWIPE GCUPS.

**L34. Length-bucketed DB batching (SWIPE).** Sort DB seqs by length, batch
within ±10 % length, pad with zero-score nulls. Without bucketing, lanes go
idle when shorter seqs finish.

**L35. Mask-driven sequence-end handling.** On per-lane end, record score,
reset H/E/S, load next DB seq via `vmseq` + `vmerge`. Branch-free; ~1 %
overhead.

## Compiler / ISA bridge

**L36. SSE2 → RVV intrinsic header (sse2rvv).** Maps `_mm_adds_epu8` /
`_mm_max_epu8` / `_mm_movemask` / `_mm_slli_si128` to RVV. Mirrors libssw's
sse2neon path; near-zero-source-change port.

**L37. Inline asm for the Lazy-F hot loop.** 10-instruction Rognes pattern in
inline asm with fixed reg allocation. Prevents compiler-inserted `vsetvli` /
spills that wreck in-order K1 schedule.

**L38. Tail handling for irregular last segment.** `vsetvli` with actual
remaining length + `ta` policy; zero-pad profile tail lanes. Bias-padded tails
would produce spurious scores.

## System integration

**L39. GASAL2-style async CPU-GPU overlap.** GPU runs alignment + packing;
CPU (RVV) prepares next batch + post-processes. Pattern applies to future
RISC-V SoCs with iGPU. 21× over Parasail CPU-only on the original GASAL2.

## Anti-diagonal variants

**L40. Anti-diagonal via `vlse8` strided.** Load anti-diagonals from
row-major storage with `vlse8.v` stride=refLen. Independent cells kill
Lazy-F entirely. Only when refLen ≤ VLEN_bytes.

**L41. KSW2 anti-diagonal difference DP.** `ksw_extz2_sse` stores ΔH 8-bit
signed along anti-diagonals with banded sweep + Z-drop. Independent
rediscovery of Suzuki-Kasahara.

## Long-vector

**L42. Long-vector (Ara2 VLEN ≥ 1024) layout.** SEW=8 with VLEN=1024 often
collapses segLen to 1; Lazy-F runs ≤1 pass. Striped layout degenerates to
anti-diagonal-like.

## Scheduling

**L43. K1 dual-issue scheduling.** Pair vector arithmetic
(`vsaddu`/`vmaxu`) with vector load/store (`vle8`/`vse8`) per cycle. K1
issues 2/cycle only across different FUs; doubles inner-loop throughput.

**L44. Hoist gap/bias broadcasts (`vmv.v.x`).** `vmv.v.x` for `vGapO`,
`vGapE`, `vBias` before the outer loop, not inside. Saves refLen·3 vector
instructions.

## Seed extension

**L45. Minimap2 forward-transform batching.** Vectorise across pending seed
extensions, one extension per LMUL group. 2–4× over serial on K1.

**L46. `vfirst.m` for ending-position recovery.** After scoring,
`vmseq.vx` against scalar max then `vfirst.m` gives the read position in O(1).
Replaces libssw's segLen·VL scalar scan.

---

## K1 top priorities (from upstream survey)

| # | Idea | Why first |
| --- | --- | --- |
| 1 | L1 + L5 + L22 + L43 (8-bit striped, bias, m1, dual-issue) | Maximum lane utilisation + ~10-inst inner loop |
| 2 | L2 + L12 (8→16 fallback) | Keep fast path hot >95 % of hits |
| 3 | L3 + L4 (query profile in-register via vrgather) | Eliminates per-residue L1 traffic |
| 4 | L10 + L37 (Lazy-F early exit + inline asm) | K1 cannot speculate around Lazy-F branch |
| 5 | L8 + L9 + L20 + L34 + L35 (SWIPE inter-sequence) | Removes Lazy-F entirely |
| 6 | L13 + L41 (Suzuki-Kasahara difference) | Keeps SEW=8 valid at multi-Mbp lengths |
| 7 | L23 + microarch H3 (Zicbop prefetch) | In-order K1 cannot hide L2 misses |
| 8 | L44 + microarch H1 (hoist broadcasts/vsetvli) | Compiler often misses |
| 9 | L21 + L38 (VLA layout, ta tails) | Single binary, correct short queries |
| 10 | L37 + compiler G2 (inline asm hot loop) | Compilers misschedule across vsetvli |

## Routing inside this file

| Variant | Ideas |
| --- | --- |
| Intra-sequence striped (Farrar / Zhao) | L1–L5, L10–L12, L21–L23, L43, L44 |
| Inter-sequence SWIPE | L8, L9, L20, L34, L35 |
| Anti-diagonal | L7, L40, L41 |
| Wavefront / WFA | L17, L18 |
| Bit-parallel edit | L24, L25 |
| Profile HMM / PSSM | L26, L27, L28, L31 |
| Banded / long-read (ONT/PacBio) | L13, L14, L15, L16 |
| Block aligner | L19 |
| Traceback / long-range | L29, L30 |
| Compiler / port | L36, L37 |
| Hardware-specific | L22, L23, L42, L43, L44 |
