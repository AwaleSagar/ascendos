# ADR-0001: Pure capability model over ACLs / ambient authority

- **Status:** Accepted
- **Date:** 2026-05-29
- **Deciders:** @AwaleSagar

## Context

The kernel needs an authorization model. The dominant model in mainstream
systems is identity-plus-ACLs with ambient authority: a process acts as a user,
and the kernel checks that user's permissions against an object's access list at
the point of use. This is familiar but has a well-known failure mode — the
confused deputy — and it makes "what can this process actually do?" effectively
unanswerable without auditing the entire system's permission state.

We want the security model to be *inspectable*: a finite, local answer to what a
process can do. We're also targeting a small, potentially verifiable kernel,
which rewards a uniform model over a pile of special-case permission checks.

## Decision

All authority is held as **capabilities**: unforgeable, kernel-managed
references that name an object together with the rights over it. There is **no
ambient authority** — no implicit power derived from identity, no global
namespace a process can reach into. A process can perform an operation if and
only if it holds a capability for it. This follows the seL4 model: capabilities
live in kernel-managed CSpaces and support mint, derive, badge, and recursive
revoke.

## Consequences

**Easier:**

- "What can this process do?" is answered by enumerating its CSpace — finite and
  local.
- Least privilege is the default, not an opt-in.
- The confused-deputy problem largely disappears: you can only act with
  authority you were explicitly handed.
- Revocation is precise and recursive.

**Harder:**

- Everything must be bootstrapped from capabilities — there's an up-front cost in
  how the root task and `svcd` distribute initial authority.
- Patterns that assume ambient authority (open any file by path as root) don't
  exist; user-space services must mediate naming. More design work in
  `vfsd`/`svcd`.
- Developers coming from POSIX have a learning curve.

## Alternatives considered

- **Identity + ACLs (POSIX-like).** Rejected: ambient authority makes the
  security model non-local and audit-heavy, and it pulls policy into the kernel.
  It conflicts directly with the small-TCB and inspectability goals.
- **Capabilities as a library over an ambient-authority kernel** (e.g.
  Capsicum-style, retrofitted). Rejected: a hybrid keeps the ambient substrate
  and its failure modes underneath. If we're designing from scratch, the clean
  model is cheaper than the hybrid.
- **MAC/label-based (SELinux-style) on top of DAC.** Rejected: powerful but
  large and policy-heavy; the opposite of a small, uniform kernel mechanism.
