# H — Microarchitecture, Energy, Autotuning, Profiling

32 ideas for tuning to specific RVV cores, measuring with PMU/roofline, managing
DVFS/thermal, and persisting autotuner state. Load when measuring K1/X280/SG2042
or chasing the last 10–30 %.

---

**H1. `vsetvli` stall budget.** PMU ratio `vsetvli / vector_insts` should be ≤1
per LMUL-group load. Each `vtype`-changing `vsetvli` flushes the X60 vector pipe
(2–4 cycles). Rewrite loops > 2 by hoisting or fusing nests so AVL is invariant.

**H2. VLEN=256 VRF bank-conflict avoidance.** X60 VRF is banked; same-cycle
reads to the same bank stall 1 cycle. For 4-way accumulator unroll, place
`acc0/acc2` on bank A, `acc1/acc3` on bank B; alternate. Counter (if exposed):
`vec_rf_bank_conflict`.

**H3. V-LSU latency hiding via Zicbop chain.** Issue `prefetch.r` N cache lines
ahead; sweep N ∈ {8…128} and fit bandwidth-vs-distance curve. Persist winning N
per kernel. In-order pipe cannot reorder loads around misses. Reported 25–35 %
gain on C908/C910.

**H4. Winograd AVL tuning for VLEN≥1024.** Ara2/Vitruvius+: set AVL to a
compile-time constant `= VLEN/SEW` so each transform stage fills entire
registers. Lets compiler chain 16 transform FMAs without pipe drain. Cache-size
crossover near 64 MB.

**H5. Multi-core narrow-vector vs single wide-core.** Short vectors are
scalar-issue-bound on wide cores. Split work across 8× 2-lane Ara2 instances →
3× throughput + 1.5× energy efficiency vs one 16-lane instance at short
problem sizes. Runtime dispatch routes by problem shape.

**H6. Xsfvqmaccqoq INT8 GEMM fast-path.** SiFive X280/P670/X100 expose a 4×
INT8 MAC opcode. Build-time detect via `misa` + Linux `HWCAP_ISA_*`; dispatch
via GNU ifunc with vanilla `vqmacc.vv` fallback. Resolves once at load time.
Extends `generic.md` A9.

**H7. Xtheadvector 0.7.1 → RVV 1.0 shim.** Compile-time header maps T-Head
intrinsics to RVV 1.0 where semantically equivalent; inserts explicit `vsetvli`
+ encoding fixups for diffs (mask policy bits). Activates when
`__riscv_xtheadvector` set and intrinsics version < 1000000. Required to run
RVV 1.0 code on Allwinner D1 / TH1520 / pre-1.0 firmware on C908. See also
`compiler.md` G27.

**H8. CARM cache-aware roofline.** Build per-level rooflines (L1/L2/L3/DRAM)
at scalar / `m1` / `m4` / `m8`; overlay each kernel's OI on the multi-ridge
roofline. Decides bandwidth-bound (→ layout work) vs compute-bound (→
LMUL/unroll work).

**H9. PMU vector-utilization profiling.** `perf stat -e r<hex>` on vendor
vector-active and vector-stall counters. Derive `vec_util = vec_active /
total_cycles`. Below 85 %, use `perf record --call-graph dwarf` to find
`vsetvli` clusters and bank conflicts. CI gate fails if util regresses.
Empirical event-code mapping needed on K1/SG2042 (undocumented as of 2025).

**H10. Empirical LMUL/SEW/AVL cost model.** Per-board sweep of 21 legal (LMUL,
SEW) pairs × 8 AVL values; measure latency + throughput for `vfmacc` / `vlse`
/ `vsse` / `vfredosum` / `vrgather`. Fit
`latency = base + ceil(AVL/(VLEN/SEW)) × uop_lat`. Publish JSON; wire into
scheduler. Standard LLVM cost model is microarch-blind.

**H11. LP-GEMM layout propagation across sequential GEMMs.** Decompose into
`pack_A` / `pack_B` / `kernel` / `unpack_C`; propagate C's packed layout
directly as next GEMM's A input; `vsseg` packs in micro-kernel's register
order. 2.25× over OpenBLAS on AVX-512; gain ≥ similar on RVV (no vendor BLAS).

**H12. Transprecision voltage scaling.** At workload boundaries (fp32 decode →
INT8 GEMM), request lower `cpufreq` voltage-frequency point. CV²f says 20 % V
drop + ½ freq ≈ 50 % energy save vs fp64-rate point on INT8 work. Sysfs knob
exposes per-kernel declared precision. fp64 → fp8 alone can be up to 16.6×
efficiency.

**H13. Vector-inst energy via PMU + board power.** `vec_inst_energy =
total_J / vector_FLOPs` via `perf stat` + RAPL or INA219/PMIC. Cross
precisions + LMUL. If `m8/fp32` >10 % more J/op than `m4/fp32`, prefer `m4`
on battery. ~55–61 % of typical RVV power is in FPUs.

**H14. RL-based DVFS governor.** Linux CPUfreq module observes (IPC,
LLC-miss, vec-util) at 1 ms; Q-learning policy trained offline on
GEMM/SpMV/FFT/attention; reward = throughput/power. Beats `ondemand` on
bursty vector workloads.

**H15. Thermal-throttle detect + burst schedule.** Monitor
`cpufreq_stats/time_in_state`; correlate with `vec_active_cycles` per
wall-second. If sustained `m8 vfmacc` triggers clock-down, alternate 10 ms
vector-heavy epochs with 2 ms scalar/sleep. K1 5–20 W TDP; SG2042 ~75 W on
all-core vector.

**H16. Winograd co-design: VLEN/L2 sweet spot.** For each (VLEN, L2) compute
analytically optimal Winograd tile maximising compute-to-memory × cache-fit
probability. VLEN=1024 + 64 KB L1 → ~32×32 fp32 tile. Parameterise generator
on `(F, r)`; bake values at compile time.

**H17. OVI memory-decoupling pattern.** On Vitruvius+ (OVI), issue a batch of
`vlse` / `vluxei` ahead of arithmetic so V-LSU ROB fills while compute units
drain. Software mirror of hardware arith/memory decoupling. SiFive X280 has
a 16-entry V-LSU buffer.

**H18. SG2042 64-core NUMA dispatch.** 4× 16-core clusters with local L3
slice. Bind GEMM teams to one cluster via `sched_setaffinity`. Large
matrices: block-cyclic row-band of A per cluster with B stripe broadcast;
pre-touch with `madvise(MADV_WILLNEED)`. See also `nn-llm.md` I32.

**H19. VRF register dispersion (software).** Manual register allocation
(inline asm) scatters accumulators across the 32 arch regs to reduce spatial
hotspot. Complements hardware RegDisp.

**H20. Vectorized FlashAttention with low-cost exp.** `exp(x) ≈ pow2(x/ln2)`
via `vfmacc` scale + `vfcvt.f.x` + bit-shift. ~0.5 % rel error, sufficient
for attention. Combine with tiling matched to V-LSU burst. First vectorized
FlashAttention on RVV. Full treatment in `nn-llm.md`.

**H21. Analytical LMUL-vs-throughput model.** `T(LMUL, SEW, N) = N /
(ceil(N/(VLEN/SEW)) × uop_lat(LMUL))`. Wire into MLIR lowering. Re-parameterised
in ~30 min per board (H10). Always accurate; no training data.

**H22. ML-driven tile-size autotuner (AutoTVM-style).** Search space `(mc, kc,
nc, LMUL, SEW, unroll)`; 200–500 trials measured via cycle counter; GBM cost
model + Bayesian optimisation. Persist per (kernel, shape, board) in SQLite.
cuDNN/MIOpen pattern, missing from RVV BLAS.

**H23. Energy-aware multi-objective autotuner.** Dual objective throughput +
J/op via INA219 sampled at 10 ms. NSGA-II / scalar weighted-sum on the Pareto
frontier; pick op-point by deployment context.

**H24. Persistent autotuning cache.** First run → 50 trials → write winning
params to `~/.cache/rvv_kernels/<board_uuid>/<kernel_sig>.bin`. Subsequent runs
mmap; zero overhead. `--retune` flag after firmware update. Standard
cuDNN pattern.

**H25. Lock-free SPSC ring for inter-core streaming.** Producer (packing) →
consumer (GEMM) via shared-memory ring (size = 2 packed panels). `vsse`
write, `vlse` read. Fence with `fence rw,rw` (intra-cluster cache-coherent).
~20 ns hand-off vs 50–200 ns mutex. Concrete impl of `generic.md` E29.

**H26. Hugepage recipe for GEMM panels.** `posix_memalign(buf, 2097152, size)`
+ `madvise(buf, size, MADV_HUGEPAGE)`. Verify via `/proc/self/smaps` →
`AnonHugePages`. Tune `vm.nr_hugepages` so ≥4× (nc·kc·4) / 2 MB are
pre-allocated. K1 dTLB ~32–64 entries; one 2 MB page saves 31× 6-cycle refills
per panel iter.

**H27. io_uring streaming pre-fetch.** `IORING_OP_READ_FIXED` + registered
hugepage-backed buffers. Userspace CQ poll removes per-op syscall. Overlap I/O
with vector pipe: core A `vlse` slot N while io_uring completes slot N+1.
Saves 2–5 µs per 64 KB on `read()`. Linux ≥ 5.10.

**H28. SCHED_FIFO + cpuset isolation.** `cset shield` an isolated cpuset;
threads at `SCHED_FIFO` 50 via `pthread_setschedparam`; disable IRQ affinity
via `/proc/irq/*/smp_affinity_mask`. `cyclictest --mlockall -t1 -p99` to
validate. In-order cores hit by 10 µs IRQ take ~100 cycle pipeline refill —
no OoO buffer.

**H29. Vector-unit warm-up at library init.** Some implementations gate vector
clock when idle. Issue 16 `vmv.v.v v0, v0` LMUL=m1 instructions at init;
bracket with `rdcycle` to measure cold-start penalty. Observed on C906
(~30–80 cycles). Hides startup latency from app code.

**H30. CI vectorization-quality gate (RAJA Performance Suite).** `rajaperf
--size LARGE --reps 5 --kernels POLYBENCH_GEMM,POLYBENCH_2MM,LCALS_HYDRO` per
commit on K1 / D1 / JH7110; fail build if any kernel's IPC regresses > 5 %
from stored baseline.

**H31. SG2042/SG2044 LLM token-rate tuning.** Identify per-phase bottleneck
with PMU: GEMV-bound decode → tune LMUL + prefetch for (1×d) shape;
GEMM-bound prefill → apply LP-GEMM (H11). Reported 2.9–3.0× over baseline on
DeepSeek R1 Distill (Llama 8B / Qwen 14B). Concrete illustration of H10
cost-model-driven LMUL selection.

**H32. Posit / quire accumulation as `vfredosum` alternative.** Software quire
(128-bit) via pair of fp64 regs with `vfmacc` widening; defer rounding to final
reduction. Cuts dot-product error by up to 4 orders vs IEEE 754. Lower overhead
than Kahan (`generic.md` B36) on workloads where neither hardware support nor
full Kahan are warranted.

---

## Routing inside this file

| Goal | Ideas |
| --- | --- |
| Measure first | H8 (roofline), H9 (PMU), H10 (cost model fit) |
| Squeeze K1 inner loop | H1 (vsetvli), H2 (VRF banks), H3 (prefetch) |
| Multi-core / NUMA | H18 (SG2042 dispatch), H25 (lock-free ring), H28 (isolation) |
| Energy budget | H12 (DVFS), H13 (J/op), H14 (RL gov), H15 (thermal) |
| Autotune | H10/H21 (analytical), H22 (ML), H23 (energy-aware), H24 (persistent) |
| Cross-VLEN binary | H4 (long-vec AVL), H6 (Xsfvqmaccqoq ifunc), H7 (0.7.1 shim) |
| TLB / IO / RT | H26 (hugepages), H27 (io_uring), H28 (SCHED_FIFO) |
| Long-vector hardware | H4, H5 (cluster trade-off), H16 (Winograd) |
| LLM-specific | H11, H20, H31 (also see `nn-llm.md`) |
