# RISC-V Skill

RISC-V Vector (RVV) C intrinsics skill, published via [skills.sh](https://skills.sh).

## Using the skill

Install it into your agent with the [skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add alexfdez1010/risc-v-skill
```

Works with Claude Code and other supported agents.

## Layout — progressive disclosure

Skill split into entry point + on-demand references. Agent loads only `SKILL.md` upfront; refs pulled in as task requires. Keeps context small.

- `skills/riscv/SKILL.md` — entry. Scope, quick template, loading guide. Always loaded.
- `skills/riscv/references/core-concepts.md` — VL/AVL, VLMAX, tail/mask policies, naming, memory access, reductions, conversions, widen/narrow, masks, bit ops, config macros.
- `skills/riscv/references/patterns.md` — algorithmic templates: vector add, transpose, reduction, softmax, NTT, CRC, microbenchmarks, perf counters.
- `skills/riscv/references/intrinsics.md` — cheat sheet grouped by category (loads/stores, arithmetic, reductions, conversions, permute, bit ops).
- `skills/riscv/references/examples.md` — 7 worked kernels with host/kernel interaction.
- `skills/riscv/references/template.md` — step-by-step guide for new RVV kernel.
- `skills/riscv/references/glossary.md` — AVL, VL, VLMAX, SEW, LMUL, tail/mask policies, Zvbc.

Routing rules live in `SKILL.md` §Loading Guidance.

## Source

Content of `SKILL.md` is based on the RVV mini-course at <https://github.com/nibrunie/rvv-examples>.
