# ADR-0006: A first-class entropy / RNG subsystem

- **Status:** Accepted
- **Date:** 2026-05-31
- **Deciders:** @AwaleSagar

## Context

The security framework assumes unpredictability — attestation nonces, key
generation, capability-badge randomization, TLS in `netstackd`, any future
ASLR — but the blueprint provided **no entropy source**. For a headless,
CLI-only, embedded-leaning OS this is the worst case: no GUI input, sparse
interrupt timing, often no disk seek noise. Entropy starvation at first boot is a
documented, real-world failure class.

## Evidence

- **"Recommendations for Randomness in the Operating System" (HotOS, CMU):** the
  user-space pool seeding from the OS pool "soon after boot" can occur before the
  pool has enough entropy to provide a strong seed.
- **Red Hat RNG guidance:** systems that generate keys very early after boot, when
  little entropy is collected, produce predictable keys — the root cause of the
  large-scale weak-SSH/TLS-key findings on embedded/headless devices.
- **ARMv8.5 `FEAT_RNG`:** provides `RNDR`/`RNDRRS` instructions — an architected
  hardware entropy source where present.
- **Linux `getrandom(2)`:** the proven API contract — block until the CSPRNG is
  initialized rather than return weak bytes.

## Decision

Add an **entropy/RNG subsystem** as a named component:

- **Sources (mixed):** hardware RNG via `FEAT_RNG` `RNDR` where present;
  DICE/boot-measurement secrets contributed at boot; a **persisted seed file**
  carried across boots; plus runtime jitter/interrupt timing.
- **Interface:** a `getrandom`-style call that **blocks until first good seed**;
  a non-blocking variant only for callers that explicitly tolerate it.
- **Gating:** security-critical key generation must wait on a "RNG ready" signal,
  exposed to `svcd` so dependent services don't start key operations too early.
- **Placement:** a small CSPRNG in the kernel (seeded as above) or a privileged
  `entropyd` user service; the kernel exposes only the minimal primitive needed
  to seed and gate. (Exact split decided at implementation; either keeps the
  policy in user space per the project's mechanism/policy rule.)

## Consequences

**Easier:** closes a silent compromise of *every* security property that depends
on unpredictability; matches a proven API contract.

**Harder:** the boot path must reach "RNG ready" before first key use — an
ordering constraint on early services; the persisted seed file needs integrity
and confidentiality (covered by dm-verity + `secd`), and must be re-generated
after first use to avoid replay.

## Alternatives considered

- **No RNG subsystem / ad-hoc per-service randomness.** Rejected: guarantees the
  early-boot weak-key failure mode.
- **Software-only entropy (no hardware RNG).** Acceptable as a fallback on parts
  without `FEAT_RNG`, but must combine with persisted seed + boot secrets and
  block until seeded; hardware RNG is preferred when available.
- **Never block (always return best-effort bytes).** Rejected: this is exactly
  what produced predictable keys in the field.
