# Template: Authoring New RVV C Code

1. Include headers:

```c
#include <riscv_vector.h>
#include <stddef.h>
```

2. Use AVL/VL loop structure:

```c
void my_kernel(float *dst, const float *src, size_t n) {
    size_t avl = n;
    while (avl > 0) {
        size_t vl = __riscv_vsetvl_e32m1(avl);
        vfloat32m1_t v = __riscv_vle32_v_f32m1(src, vl);
        // ... compute ...
        __riscv_vse32_v_f32m1(dst, v, vl);
        src += vl;
        dst += vl;
        avl -= vl;
    }
}
```

3. For reductions, initialize a full-length accumulator and use tail-undisturbed
   ops.

4. If using float-to-int conversion, explicitly set the rounding mode (as in
   softmax).

5. If your algorithm needs permutation, evaluate whether strided loads/stores,
   slides, or gather are most efficient.
