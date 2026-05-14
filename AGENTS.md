# AGENTS.md

Project guide for agents working on this repository.

## What this repo is

`risc-v-skill` — a Claude Code / [skills.sh](https://skills.sh) skill that
packages RISC-V Vector (RVV 1.0) C intrinsics knowledge plus a 275-idea
optimization catalog. Distributed via `npx skills add alexfdez1010/risc-v-skill`.

## Layout

```text
.
├── AGENTS.md                  # this file
├── CLAUDE.md                  # symlink → AGENTS.md
├── README.md                  # user-facing description
├── REFERENCES.md              # bibliography (outside the skill — never loaded by the agent)
├── LICENSE                    # MIT
└── skills/
    └── riscv/
        ├── SKILL.md           # skill entry point — always loaded by the agent
        └── references/        # on-demand files (progressive disclosure)
            ├── core-concepts.md
            ├── intrinsics.md
            ├── patterns.md
            ├── examples.md
            ├── template.md
            ├── glossary.md
            └── optimizations/
                ├── overview.md
                ├── generic.md
                ├── compiler.md
                ├── microarch.md
                ├── nn-llm.md
                ├── crypto-codecs.md
                ├── scientific.md
                └── alignment.md
```

## Design principles

1. **Progressive disclosure.** Only `SKILL.md` is always loaded. Every
   reference file is opt-in via the loading guidance in `SKILL.md`.
   `SKILL.md` must stay small — a few KB — so the agent can decide what to
   load without burning context.
2. **One topic per file.** `references/optimizations/*.md` is split by domain
   (compiler / microarch / NN-LLM / crypto-codecs / scientific / alignment)
   so the agent fetches only what the current task needs.
3. **Citation-free skill files.** Topic files contain ideas, not bibliography.
   All paper / repo references live in the repo-root `REFERENCES.md`, which is
   deliberately **outside** the skill so the agent never loads it. Skill files
   must not even mention the bibliography — every token counts.
4. **Terse, action-oriented entries.** Each optimization idea is one short
   bullet: What → Why → Where. Code goes in `references/examples.md` /
   `references/patterns.md`, not in optimization files.
5. **VLA-correct examples.** Any code shown must use `vsetvl` strip-mining
   loops and the tail-agnostic / mask-agnostic policies by default. No
   hard-coded VLEN.

## Conventions

- Markdown only. No HTML.
- Code blocks must have a language tag (`bash`, `c`, `asm`).
- Tables use `| --- |` separators with spaces.
- Headings are sentence case. Avoid trailing punctuation in headings.
- Hardware names: `SpacemiT K1 / X60`, `SiFive X280`, `T-Head C908`,
  `Sophon SG2042`, `Ara2`, `Vitruvius+`, `Spatz`. Be explicit about VLEN when
  it changes the advice.
- RVV instruction mentions in backticks: `` `vfmacc.vv` ``, `` `vsetvli` ``.
- Extensions in capital letters: Zfh, Zvfh, Zba, Zbb, Zbc, Zbs, Zicbop,
  Zvkned, Zvkb, etc.

## Editing rules

- Adding an optimization idea: pick the topic file. Use the bold-ID convention
  (`**G36. Short title.**`). Keep it to ~3 short sentences (What / Why /
  Where). Add the citation to the repo-root `REFERENCES.md` if it's
  paper-sourced — never inside the skill.
- Adding a topic file: also list it in `SKILL.md` "Reference files" and in
  `optimizations/overview.md`. Update the routing tables in both.
- Adding an RVV-core reference (under `references/` outside `optimizations/`):
  cross-link from the relevant optimization file when there's a natural
  pointer.
- Don't grow `SKILL.md` past ~150 lines. If you need more, push detail into a
  reference file and link.

## Source of optimization ideas

Most ideas were mined using
[`paperhound`](https://github.com/alexfdez1010/paperhound) across RISC-V,
LLVM/MLIR, NN/LLM, crypto, HPC, and bioinformatics literature (2007–2026).
Six parallel `paperhound` sub-agents covered distinct topic clusters; the
output was cross-checked against existing kernels (libssw, parasail, ksw2,
WFA2-lib, edlib, llama.cpp, IREE, OpenSSL Zvkned) before each idea was
condensed into a What/Why/Where bullet.

## Building / publishing

The skill is consumed by `npx skills add alexfdez1010/risc-v-skill`. There is
no build step — the contents of `skills/riscv/` are served as-is. To preview
locally, copy `skills/riscv/` into a Claude Code session's `.claude/skills/`
directory or run the `skills` CLI against a local clone.

## When in doubt

- Match the existing terse, code-forward style of `SKILL.md`.
- Prefer condensing over expanding. The agent's context is the bottleneck.
- If an idea fits cleanly into the base 56 (`generic.md`), put it there
  instead of creating a new domain file.
