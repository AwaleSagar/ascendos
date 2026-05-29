# RFC process

ADRs record decisions *after* they're made. RFCs are how a significant decision
gets *proposed and discussed* in the first place. If you want to change the
architecture, you write an RFC.

## When you need an RFC

- **You don't** for typos, wording, link fixes, or clarifications — just open a
  pull request.
- **You do** for anything that changes the architecture, adds or removes a
  subsystem, or revisits an existing ADR.

Not sure? Open an *RFC proposal (intent)* issue first and ask.

## The flow

1. **Float the intent.** Open an *RFC proposal* issue with a summary,
   motivation, and the alternatives you've considered.
2. **Write the RFC.** Copy [`0000-template.md`](0000-template.md) to
   `docs/rfcs/NNNN-short-title.md` and open a pull request. Use the next free
   number.
3. **Discuss in the open.** The PR is where the conversation happens. Expect
   revisions. The strongest RFCs engage honestly with their own weaknesses.
4. **Decision.** The maintainer accepts, rejects, or asks for changes (see
   [GOVERNANCE](https://github.com/AwaleSagar/ascendos/blob/main/.github/GOVERNANCE.md)).
   An accepted RFC is merged, and its outcome is recorded as a new
   [ADR](../adr/README.md) so the *decision* lives alongside the others.

## What makes a good RFC

The same thing that makes a good ADR: a clear problem, an honest treatment of
alternatives, and a frank account of the costs of your proposal — not just its
benefits. An RFC that pretends it has no downsides is an RFC that hasn't been
thought through.
