# Security

Security in AscendOS is a property of the whole design, not a subsystem. It shows
up in every band.

| Pillar | Mechanism |
|---|---|
| Authority | Pure capabilities; least privilege by construction ([ADR-0001](../adr/0001-pure-capability-model.md)) |
| Boot integrity | Secure boot + DICE layered attestation ([Boot](boot.md)) |
| Memory safety | Memory-safe kernel language + ARMv9 MTE tagging (see caveats below) |
| Control-flow integrity | PAC (pointer auth) + BTI (branch target) |
| Isolation | Per-process address spaces, SMMU-scoped DMA, user-space drivers |
| Confidential compute | ARMv9 CCA Realms via optional `realmd`/EL2 |
| Policy & audit | `secd` as a central decision point + append-only audit log |
| Randomness | Entropy/RNG subsystem; gates key generation ([ADR-0006](../adr/0006-entropy-rng-subsystem.md)) |
| Safe extensibility | Verified bytecode sandbox; no native kernel modules ([ADR-0004](../adr/0004-no-native-kernel-modules.md)) |

## MTE is a mitigation, not a guarantee

It's tempting to list "MTE" as a memory-safety pillar and move on. That
overstates what the hardware actually buys, and we'd rather be precise:

- **Tags are probabilistic.** ARM MTE uses 4-bit tags (16 values), so a blind
  mismatch is caught only ~15/16 of the time. It raises the cost of an exploit;
  it does not close the class.
- **It has been defeated by speculation.** TIKTAG (Kim et al., IEEE S&P 2025)
  demonstrates speculative-execution side channels that leak MTE tags, which
  undermines the probabilistic protection in practice. MTE is a moving target,
  not a settled defense.
- **The cheap number is the weak mode.** The widely-quoted "~1–2% overhead" is
  *asynchronous* mode, which only tells you a violation happened *eventually* —
  too coarse for precise enforcement. **Synchronous** mode (a fault at the
  offending instruction) is what gives real spatial/temporal guarantees, and it
  costs materially more — single-digit percent on newer cores, worse on small
  (A5x-class) cores.
- **Hardware is scarce.** As of 2026 the only widely shipping consumer MTE
  silicon is Google Tensor G3+ (Pixel 8 and later); server parts like AmpereOne
  are emerging. AscendOS must run usefully on hardware *without* MTE, so MTE can
  only ever be defense-in-depth, never a load-bearing assumption.

Net: MTE stays in the design as one layer, but the real memory-safety story is
the kernel language and the isolation model (capabilities, SMMU, user-space
drivers). MTE is the belt, not the trousers.

## CCA/RME hardware availability

ARM Confidential Compute Architecture (CCA) and the Realm Management Extension
(RME) are first-class in the design, but the hardware reality in 2026 is thin:

- **No widely shipping CCA silicon exists yet.** ARM has announced RME in
  ARMv9.2-A; reference implementations exist in FVP (Fixed Virtual Platform)
  models, but production silicon with RME enabled is limited to early server
  reference platforms. No consumer SoC ships RME today.
- **The optional design is correct.** Marking `realmd` and EL2 usage as optional
  is the right call — AscendOS must boot and be useful on hardware without CCA.
- **Migration path.** When CCA silicon arrives, the capability model already
  provides the isolation primitives (`RealmCap` objects). The gap is in EL2 setup
  and RMI (Realm Management Interface) plumbing, which is entirely in `realmd`
  and doesn't affect the core kernel.

Net: CCA stays as an architectural target, not a boot requirement. The design
should not depend on RME features for any security property that matters on
current hardware.

## Entropy and randomness

Originally absent, now a named subsystem
([ADR-0006](../adr/0006-entropy-rng-subsystem.md)) — because a headless, CLI-only
OS is the documented worst case for early-boot entropy starvation, and weak
randomness silently breaks attestation nonces, key generation, badge
randomization, and TLS.

- **Mixed sources:** ARMv8.5 `FEAT_RNG` (`RNDR`) where present, DICE/boot-
  measurement secrets, a persisted cross-boot seed, runtime jitter.
- **Blocks until seeded:** a `getrandom`-style contract — no best-effort weak
  bytes — with a "RNG ready" signal `svcd` uses to gate key-using services.
- **Seed integrity:** the persisted seed is protected by dm-verity + `secd` and
  regenerated after use.

## Transient execution & timing side-channels

Isolation (capabilities, SMMU, separate address spaces) does **not** by itself
stop Spectre-class transient-execution leaks — Spectre is a class, "there is no
single mitigation" (FAU EuroSec 2021). Two consequences AscendOS must own:

- **Scope:** the threat model (below) must state which transient-execution
  variants are in scope and plan for context-switch flushing and branch-predictor
  controls on ARM (e.g. `CSDB`/barriers, predictor invalidation on domain
  crossing).
- **Real-time tension (important):** those mitigations cost cycles on every
  domain crossing, which directly fights the hard-real-time goal pursued via MCS
  scheduling contexts ("Spectre and Meltdown vs. Real-Time", Linux Foundation).
  This is a genuine conflict between two design goals and is recorded as such, not
  hidden. A microkernel's frequent crossings make both the exposure and the
  mitigation cost larger than on a monolith.

## A note on threat model

We don't yet have a written threat model, and we should. What's an attacker
assumed to control? What's out of scope (physical attacks? side channels like
TIKTAG?)? Until that document exists, treat the security claims here as design
*intent* that hasn't been adversarially reviewed. This is a known gap and a good
place to contribute — see issue #2.

## Sources

- TIKTAG: Breaking ARM's Memory Tagging Extension with Speculative Execution —
  Kim et al., IEEE S&P 2025 (<https://taesoo.kim/pubs/2025/kim:tiktag-sp.pdf>).
- MTE modes and overhead — Arm Community blog, "Delivering enhanced security
  through MTE"; Android Open Source Project, "Arm memory tagging extension".
- MTE hardware availability — Arm Learning Paths, "Enable MTE on Google Pixel 8".
- Early-boot entropy — "Recommendations for Randomness in the OS" (HotOS/CMU);
  Red Hat RNG guidance; ARM `FEAT_RNG`. See
  [ADR-0006](../adr/0006-entropy-rng-subsystem.md).
- Side-channels — meltdownattack.com; "The Price of Meltdown and Spectre" (FAU
  EuroSec 2021); "Spectre and Meltdown vs. Real-Time" (Linux Foundation).

Full detail: blueprint §11.
