# RVV Examples — Host/Kernel Interaction

Self-contained summaries of main kernels and how a typical host harness drives
them. The host is the scalar driver that allocates buffers, initializes inputs,
calls the RVV kernel, and reads perf counters. The kernel is the RVV intrinsics
code that runs on the same core but uses vector instructions (device-style
computation inside a host-driven loop).

## Common Host Pattern

```c
// allocate + initialize
float *src = ...; float *dst = ...; size_t n = ...;
// perf counter
unsigned long start = rdinstret();
kernel(dst, src, n);
unsigned long stop = rdinstret();
printf("%lu instructions\n", stop - start);
```

## Common Kernel Pattern

```c
size_t avl = n;
while (avl > 0) {
  size_t vl = __riscv_vsetvl_e32m1(avl);
  vfloat32m1_t v = __riscv_vle32_v_f32m1(src, vl);
  // ... compute ...
  __riscv_vse32_v_f32m1(dst, v, vl);
  src += vl; dst += vl; avl -= vl;
}
```

## 1. Vector Add (baseline kernel + bench harness)

**Kernel behavior:** load two vectors, add, store.

```c
vfloat32m1_t a = __riscv_vle32_v_f32m1(lhs, vl);
vfloat32m1_t b = __riscv_vle32_v_f32m1(rhs, vl);
vfloat32m1_t c = __riscv_vfadd_vv_f32m1(a, b, vl);
__riscv_vse32_v_f32m1(dst, c, vl);
```

**Host interaction:** allocate `lhs/rhs/dst`, run once, print instruction count.
This is the simplest "host drives kernel" example.

## 2. Matrix Transpose (multiple kernels + benchmark menu)

**Kernel behaviors:**

- Strided-store transpose (write columns using `vsse32`).
- Strided-load transpose (read columns using `vlse32`).
- Segmented load/store for 4x4.
- In-register permutation with `vrgather` or `vslide`.

```c
vfloat32m1_t row = __riscv_vle32_v_f32m1(row_src, vl);
__riscv_vsse32(row_dst, sizeof(float) * n, row, vl);
```

**Host interaction:** build a list of implementations, run each one on the same
input, verify output and compare perf counters.

## 3. Softmax (algorithmic variants + accuracy/perf bench)

**Kernel behaviors:**

- Vectorized exponent approximation (polynomial + exponent reconstruction).
- Accumulate sum with tail-undisturbed vector accumulator.
- Normalize with a vector multiply by reciprocal.

```c
vsum = __riscv_vfadd_vv_f32m1_tu(vsum, vsum, vexp, vl);
vfloat32m1_t row = __riscv_vle32_v_f32m1(dst, vl);
row = __riscv_vfmul_vf_f32m1(row, inv_sum, vl);
__riscv_vse32(dst, row, vl);
```

**Host interaction:** generate random inputs, compute a golden reference, run
multiple kernels, and print perf + error metrics.

## 4. Polynomial Multiplication / NTT (vectorized butterfly stages)

**Kernel behaviors:**

- Split even/odd coefficients with strided loads or compressed loads.
- Vectorized butterfly: multiply by twiddle factors, add/sub, then reduce
  modulo.
- Barrett reduction and masked corrections to keep coefficients in range.

```c
vec_odd = __riscv_vmul_vv_i32m8(vec_odd, vec_twiddle, vl);
vec_even = __riscv_vadd_vv_i32m8(vec_even, vec_odd, vl);
vec_even = __riscv_vrem_vx_i32m8(vec_even, modulo, vl);
```

**Host interaction:** allocate polynomial/ring buffers, call forward NTT /
multiply / inverse NTT routines, then display results or perf counters.

## 5. Reduction (vector reductions to scalar)

**Kernel behaviors:**

- Use `vred*`/`vfred*` to reduce a vector into a scalar lane.
- Extract the scalar with `vfmv.f.s` or `vmv.x.s`.

```c
vfloat32m1_t acc = __riscv_vfmv_v_f_f32m1(0.f, vlmax);
vfloat32m1_t sum = __riscv_vfredusum_vs_f32m1_f32m1(v, acc, vl);
float s = __riscv_vfmv_f_s_f32m1_f32(sum);
```

**Host interaction:** set up inputs, run the reduction kernel, and print perf
counters or the reduced scalar.

## 6. CRC (vector folding using carry-less multiply)

**Kernel behaviors:**

- Load expanded data, use carry-less multiply (`vclmul`/`vclmulh`).
- Byte-reversal and endianness handling via `vrev8`/`vbrev`.
- Reduce folded vectors with `vredxor`.

```c
vuint64m1_t lo = __riscv_vclmul_vv_u64m1(a, b, vl);
vuint64m1_t hi = __riscv_vclmulh_vv_u64m1(a, b, vl);
```

**Host interaction:** prepare buffers for CRC input, call the vector CRC
kernel(s), and compare against a scalar CRC reference.

## 7. Microbenchmarks (instruction-level kernels)

**Kernel behaviors:**

- Short inline loops to measure latency/throughput for a specific instruction
  pattern.
- Variations by SEW/LMUL to isolate vector configuration costs.

**Host interaction:** run each microbenchmark in a loop and print perf counters
to interpret instruction costs.
