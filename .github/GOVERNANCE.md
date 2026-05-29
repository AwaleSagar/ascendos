# Governance

This document describes how decisions get made in AscendOS. It is deliberately
modest — the project is small, and pretending otherwise would be dishonest.

## Current model: single maintainer

AscendOS is currently maintained by one person,
[@AwaleSagar](https://github.com/AwaleSagar), who has final say on design and
merge decisions. This is the right model for a project at the blueprint stage,
and it has obvious limits: a single point of failure and a single point of view.
The intent is to grow beyond it.

## How decisions are made

- **Small changes** (docs, wording, fixes): a pull request, reviewed and merged.
  Lazy consensus applies — if no one objects within a reasonable window, it goes
  in.
- **Significant design changes:** an [RFC](../docs/rfcs/README.md). The RFC is
  discussed in the open. The maintainer decides to accept, reject, or ask for
  revision, and records the outcome — and the reasoning — as an
  [ADR](../docs/adr/).

The decision trail (RFC → discussion → ADR) is intentionally part of the public
record. The point is that *why* a decision was made survives, including the
options that were rejected.

## Becoming a maintainer

There is no committee to join yet. The realistic path: contribute substantively
and consistently — good critique, solid RFCs, reliable review — and the
maintainer will invite co-maintainers as the contributor base grows. When that
happens, this document will be updated to describe a shared model (likely lazy
consensus among maintainers with a tie-break).

## Changing this document

Governance changes go through an RFC like any other significant change. Until
there are multiple maintainers, the maintainer has final say — but the
discussion happens in public.
