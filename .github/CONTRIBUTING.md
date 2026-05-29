# Contributing to AscendOS

Thanks for considering it. AscendOS is at the design stage, so contributions
look a little different from a typical code project — the most valuable thing
you can do right now is **argue with the design**.

## What's most useful right now

1. **Critique the decisions.** Read the [ADRs](../docs/adr/) and open an issue
   when you think one is wrong. Disagreement with reasoning attached is worth
   more than agreement.
2. **Find gaps and contradictions** in the [architecture pages](../docs/architecture/).
3. **Propose changes** through the [RFC process](../docs/rfcs/README.md) for
   anything that touches the architecture.
4. **Fix docs** — typos, broken links, unclear passages. Small PRs welcome.

There is no kernel code to contribute to yet. When there is, this document will
grow a build/test section.

## How to propose a change

- **Small** (typo, link, wording, a clarifying sentence): open a pull request
  directly.
- **A design question or doubt:** open a *Design question* issue.
- **A real change to the architecture:** write an RFC. See
  [`docs/rfcs/README.md`](../docs/rfcs/README.md). Accepted RFCs get recorded as
  an ADR.

## Pull request expectations

- Keep PRs focused — one idea per PR.
- If your change affects a decision, say which ADR/RFC it relates to.
- If you change a diagram, update both the `.mmd` source and any prose that
  describes it.
- Run a spell/link check on docs you touch if you can.

## Style

- Plain, direct prose. We actively avoid marketing language ("blazing-fast",
  "revolutionary"). If a claim can't be backed up, it doesn't go in.
- Performance figures are *targets* until something runs — label them as such.
- It's fine to leave a section as a stub marked `> Status: stub`. Honest
  incompleteness beats fake completeness.

## Conduct

By participating you agree to the [Code of Conduct](CODE_OF_CONDUCT.md).

## Questions

Open a [Discussion](https://github.com/AwaleSagar/ascendos/discussions) or see
[SUPPORT](SUPPORT.md).
