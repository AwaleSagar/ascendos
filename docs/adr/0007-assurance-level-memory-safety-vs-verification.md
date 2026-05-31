# ADR-0007: Separate "memory safety" from "formal verification" as assurance goals

- **Status:** Accepted (sets terms; the language choice remains open)
- **Date:** 2026-05-31
- **Deciders:** @AwaleSagar

## Context

The blueprint leans toward a memory-safe systems language (Rust) *and* talks about
a "verifiable" TCB, sometimes as if these were the same property. They are not,
and conflating them sets a false expectation about how much assurance the language
choice buys.

## Evidence

- **seL4's verification is of C**, with a decade-mature Isabelle/HOL proof chain
  (seL4 proofs page). There is no comparably mature, end-to-end *functional-
  correctness* proof stack for a Rust kernel today.
- **Rust delivers memory safety, not functional correctness.** Kernels need
  `unsafe` precisely where assurance matters most (MMIO, page tables, the ring),
  and `unsafe` is outside the safety guarantee.
- **Rust practice (Rust blog, "What we heard about Rust's challenges", 2026;
  Redox/Theseus):** the real low-level cost is designing data structures around
  the borrow checker (intrusive lists, shared kernel state), not the checker
  itself. Rust is viable for OS work — but as a *memory-safety*, not a
  *verification*, tool.

## Decision

Treat two assurance levels as **distinct, explicitly chosen** goals:

- **Memory safety** — bounds/use-after-free/type safety. **Rust delivers this**
  (modulo `unsafe`).
- **Functional verification** — machine-checked proof the code meets a spec.
  **Rust does not yet deliver this at seL4's level.**

The language ADR (still open, Phase 1) must state which level AscendOS targets:

1. If the goal is **seL4-level functional verification**, weigh C-plus-mature-proof
   against Rust-plus-emerging-tooling honestly, and accept that full verification
   may not be achievable in Rust on the project's timeline.
2. If the goal is **memory-safety + strong testing/auditing** (a defensible target
   for a small TCB), choose Rust and **reframe "verifiable" as "auditable"**
   throughout the docs, so we don't overclaim.

This ADR does not pick the language; it forbids conflating the two goals.

## Consequences

**Easier:** honest expectations; the eventual language decision is made against a
clear assurance target rather than a vague "verifiable."

**Harder:** may force an uncomfortable admission that seL4-level assurance and
Rust are not (yet) jointly achievable — better admitted now than discovered late.

## Alternatives considered

- **Keep "memory-safe = verifiable" framing.** Rejected: misleading and would set
  the project up to overclaim.
- **Commit to C + proof now.** Premature — the language ADR is a Phase 1 decision
  with its own trade-offs (ergonomics, contributor pool). This ADR only sets the
  terms that decision must respect.
