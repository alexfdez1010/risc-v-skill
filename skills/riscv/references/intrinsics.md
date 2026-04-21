# Intrinsic Patterns

Common intrinsics used in RVV C code.

## Vector Length

- `__riscv_vsetvl_e32m1(avl)`
- `__riscv_vsetvlmax_e32m1()`

## Loads / Stores

- `__riscv_vle32_v_f32m1(ptr, vl)`
- `__riscv_vse32_v_f32m1(ptr, v, vl)`
- `__riscv_vlse32_v_f32m1(ptr, stride_bytes, vl)`
- `__riscv_vsse32(ptr, stride_bytes, v, vl)`
- `__riscv_vlseg4e32_v_f32m1x4(ptr, vl)`
- `__riscv_vsseg4e32_v_f32m1x4(ptr, v, vl)`

## Arithmetic

- `__riscv_vfadd_vv_f32m1(a, b, vl)`
- `__riscv_vfadd_vf_f32m1(a, scalar, vl)`
- `__riscv_vfmul_vv_f32m1(a, b, vl)`
- `__riscv_vfmul_vf_f32m1(a, scalar, vl)`
- `__riscv_vfmadd(...)`, `__riscv_vfnmsac(...)` (FMA and fused neg multiply-add)

## Reductions

- `__riscv_vfredusum_vs_f32m1_f32m1(vec, acc, vl)`
- `__riscv_vfredosum_vs_f32m1_f32m1(vec, acc, vl)`
- `__riscv_vredmin_vs_i32m1_i32m1(vec, acc, vl)`
- `__riscv_vredxor_vs_u64m1_u64m1(vec, acc, vl)`

## Conversions / Reinterpret

- `__riscv_vfcvt_x_f_v_i32m1(vec, vl)`
- `__riscv_vfcvt_f_x_v_f32m1(vec, vl)`
- `__riscv_vreinterpret_v_i32m1_f32m1(vec)`

## Widen/Narrow

- `__riscv_vwmul_vx_i64(...)`
- `__riscv_vnsra_wx_i32(...)`
- `__riscv_vzext_vf2_u64m1(...)`

## Permute / Slide / Gather

- `__riscv_vslideup_vx_*` / `__riscv_vslidedown_vx_*`
- `__riscv_vrgatherei16_vv_*`
- `__riscv_vget_v_f32m4_f32m1` / `__riscv_vcreate_v_f32m1_f32m4`

## Bit Ops and Carry-less Multiply

- `__riscv_vclmul_vv_u64m1(...)`
- `__riscv_vclmulh_vv_u64m1(...)`
- `__riscv_vrev8_v_u32m1(...)`
- `__riscv_vbrev_v_u64m1(...)`
