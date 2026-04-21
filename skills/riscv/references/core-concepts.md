# Core RVV Concepts

## 1. Vector Length (VL) and Application Vector Length (AVL)

RVV is vector-length agnostic. You do not assume a fixed VL. The common pattern
is:

```c
size_t avl = n;
while (avl > 0) {
    size_t vl = __riscv_vsetvl_e32m1(avl);
    vfloat32m1_t v = __riscv_vle32_v_f32m1(ptr, vl);
    // ... compute ...
    __riscv_vse32_v_f32m1(ptr, v, vl);
    avl -= vl;
    ptr += vl;
}
```

Key points:

- `avl` is the number of elements left to process.
- `__riscv_vsetvl_*` returns the VL actually used (<= avl).
- The loop updates pointers by `vl` elements.

## 2. VLMAX and Full-Width Accumulators

Some reductions use a full-length accumulator initialized with `vsetvlmax`:

```c
size_t vlmax = __riscv_vsetvlmax_e32m1();
vfloat32m1_t acc = __riscv_vfmv_v_f_f32m1(0.f, vlmax);
```

This is critical when you need to keep partial sums across loop iterations and
must preserve tail lanes.

## 3. Tail and Mask Policies

When operating on partial vectors (vl < VLMAX), the code sometimes uses
tail-undisturbed variants (suffix `_tu`, `_tumu`, `_mu`) so that inactive lanes
are preserved (important for accumulators):

- Tail agnostic: `ta`
- Tail undisturbed: `tu`
- Mask agnostic: `ma`
- Mask undisturbed: `mu`

You will see intrinsics such as `__riscv_vfadd_vv_f32m1_tu(...)` to preserve
tail elements.

## 4. Intrinsics Naming and Types

- Types: `vfloat32m1_t`, `vint32m1_t`, `vuint32mf2_t`, `vfloat32m4_t`, etc.
  - `m1`, `m2`, `m4`, `m8` represent LMUL.
  - `mf2`, `mf4` are fractional LMUL.
- Intrinsics encode operation, data type, and LMUL, for example:
  - `__riscv_vle32_v_f32m1` (load 32-bit floats, LMUL=1)
  - `__riscv_vfadd_vv_f32m1` (vector-vector add)
  - `__riscv_vfadd_vf_f32m1` (vector-scalar add)

General format: `__riscv_<op>_<suffix>` where suffix encodes element type and
LMUL.

## 5. Memory Access Patterns

- **Contiguous**: `vle32`, `vse32`.
- **Strided**: `vlse32`, `vsse32` (stride in bytes).
- **Segmented**: `vlseg4e32`, `vsseg4e32` for interleaved data.
- **Gather/Permute**: `vrgather`, `vrgatherei16` for reordering.
- **Slide**: `vslideup`, `vslidedown` for shifting lanes.

## 6. Reductions

Reductions appear in two styles:

- **Vector reduction instructions** (e.g., `vredmin.vs`, `vredsum.vs`,
  `vfredusum.vs`):
  - Works on a vector input and reduces to a scalar element in a vector
    register.
  - Often paired with `vfmv.f.s` or `vmv.x.s` to extract scalar.

- **Two-step reduction** for LMUL>1:
  - First reduce within each vector register.
  - Then reduce the final vector to a scalar (common for min/sum).

## 7. Conversions and Rounding

- Float to int conversion: `__riscv_vfcvt_x_f_v_i32m1`.
- For deterministic float-to-int conversions, set rounding explicitly:
  - `fesetround(FE_TONEAREST);`

## 8. Widening/Narrowing and Reinterpretation

- Widening: `vwmul`, `vwsll`, `vzext`.
- Narrowing: `vnsra`, `vnsrl`.
- Reinterpretation: `__riscv_vreinterpret_v_i32m1_f32m1` (bitwise reinterpret
  without conversion).

## 9. Masks

- Mask types are `vboolX_t` and depend on SEW/LMUL.
- Masked intrinsics are used for conditional updates (example: Barrett reduction
  in polynomial mult).

## 10. Bit Manipulation and Carry-less Multiply

CRC-style workloads often use:

- `vclmul` / `vclmulh` (Zvbc extension)
- `vbrev`, `vbrev8`, `vrev8` (Zvbb/Zvkb style operations)
- Endianness is handled through byte-reversal steps.

## Common Configuration Macros

These macros affect how the code is specialized:

- `LMUL`, `WLMUL`, `NLMUL`: vector grouping for polynomial mult.
- `E32_MASK`: mask type width derived from LMUL.
- `USE_PRECOMPUTED_ROOT_POWERS`: select precomputed tables in NTT.
- `USE_VREM_MODULO`: choose between remainder instruction vs Barrett reduction.
- `COUNT_INSTRET`, `COUNT_CYCLE`: select perf counter.
- `HAS_ZVBB_SUPPORT`: toggle for bit-manip instructions.
- `LINUX_PERF_COUNT`: uses perf_event_open for host perf counters.
