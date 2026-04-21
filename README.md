# riscv-skill

RISC-V Vector (RVV) C intrinsics skill, published via [skills.sh](https://skills.sh).

## Using the skill

Install it into your agent with the [skills CLI](https://github.com/vercel-labs/skills):

```bash
npx skills add alexfdez1010/riscv-skill
```

Works with Claude Code and other supported agents. Nothing else needed — the skill ships as `SKILL.md`.

## Source

Content of `SKILL.md` is based on the RVV mini-course at <https://github.com/nibrunie/rvv-examples>.

---

## Development

Everything below is only for contributors editing this repo. End users do not need it.

Tooling: [Prettier](https://prettier.io) for markdown formatting,
[Bun](https://bun.sh) for dependency management.

```bash
bun install           # install deps
bun run format        # rewrite files
bun run check         # verify formatting
bun run pre-commit    # format + check
```
