# Core Patterns and Algorithmic Templates

## 1. Vector Add Pattern

Minimal vector loop template:

```c
void vec_add(float *dst, const float *lhs, const float *rhs, size_t n) {
    size_t avl = n;
    while (avl > 0) {
        size_t vl = __riscv_vsetvl_e32m1(avl);
        vfloat32m1_t a = __riscv_vle32_v_f32m1(lhs, vl);
        vfloat32m1_t b = __riscv_vle32_v_f32m1(rhs, vl);
        vfloat32m1_t c = __riscv_vfadd_vv_f32m1(a, b, vl);
        __riscv_vse32_v_f32m1(dst, c, vl);
        lhs += vl;
        rhs += vl;
        dst += vl;
        avl -= vl;
    }
}
```

## 2. Matrix Transpose Patterns

Common strategies:

- Strided stores: `__riscv_vsse32` (transpose by storing into columns).
- Strided loads: `__riscv_vlse32`.
- Segmented load/store: `vlseg4e32`, `vsseg4e32`.
- Permutation-based transpose: `vrgatherei16`, `vslideup`, `vslidedown`.

Important patterns:

- For nxn transpose, a nested loop iterates over rows, and each row uses a
  vector loop with strided stores/loads.
- For 4x4 transpose, in-register permutations reduce memory traffic.

## 3. Reduction

Two main approaches appear:

1. Intrinsic-based micro-benchmarks of reduction instructions:
   - `vredmin.vs`, `vredsum.vs`, `vfredosum.vs`, `vfredusum.vs`.
2. Inline-assembly loop for `rvv_min` and dot-product:
   - Explicit `vsetvli` / `vle32.v` / `vredmin.vs` and `vfredosum.vs` sequences.

Concepts to carry forward:

- Pre-initialize accumulator vector registers.
- When LMUL>1, it is common to run a second reduction pass to combine partial
  accumulators.

## 4. Softmax-Style Normalization

Pattern summary:

- Compute maximum using `vfredmax` with a `-INFINITY` accumulator.
- Compute element-wise exponentials (often using a polynomial approximation).
- Accumulate a sum with `vfredusum` or vector accumulators.
- Normalize each element by multiplying by the reciprocal of the sum.

Notable RVV usage:

- `vfredmax` for max reduction.
- `vfmadd` / `vfnmsac` for polynomial evaluation.
- `vfredusum` for sum reduction.
- Tail-undisturbed updates for accumulators.

## 5. Polynomial Multiplication / NTT

Core data types:

```c
typedef struct {
  int n;
  int modulo;
  int rootOfUnity;
  int invRootOfUnity;
  int invDegree;
} ring_t;

typedef struct {
  int degree;
  int modulo;
  int* coeffs;
  size_t coeffSize;
} polynomial_t;
```

Key facts:

- The primary ring uses `modulo = 3329` and degree 127 (Kyber-style).
- Root parameters: `rootOfUnity = 33`, `invRootOfUnity = 2522`,
  `invDegree = 3303`.
- Modulo polynomial: `x^128 - 1` (represented with coefficients at 0 and 128).

RVV implementation details:

- Use precomputed coefficient index arrays for NTT butterfly schedules.
- Use macro metaprogramming to generate intrinsic names for configurable LMUL.
- Use Barrett reduction to avoid expensive modulo operations.
- Use `vmsge` + masked add/sub for conditional modular corrections.
- Variants often include strided load, indexed load, compressed, Barrett
  reduction, and assembly implementations.

If you author new RVV polynomial code, follow the same patterns:

- Use LMUL-parameterized macros.
- Ensure modulo reductions are consistent (especially for negative results).
- Keep NTT data layout compatible with precomputed index arrays.

## 6. CRC

CRC workloads often use RVV with carry-less multiply (Zvbc):

- Vector folding uses `vclmul` / `vclmulh` on expanded data.
- Endianness handling uses `vrev8`, `vbrev`, or `vbrev8`.
- Reduction is done via `vredxor` into accumulator vectors.

CRC code commonly distinguishes BE vs LE handling, and may use Zvbc32e if
available.

## 7. Microbenchmarks and Measurement Loops

Microbenchmarks use inline assembly loops to measure latency/throughput. Common
goals:

- How to construct dependency chains for latency measurement.
- How to create unrolled, parallel instruction streams for throughput
  measurement.
- How to vary LMUL and SEW for vector instruction benchmarks.

## 8. Perf Counters

Perf counters are read using:

- Inline asm `rdinstret` (retired instructions) or `rdcycle` (cycles).
- Optional Linux perf_event support when running on Linux hosts.
