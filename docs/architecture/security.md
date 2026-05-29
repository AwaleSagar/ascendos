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

Full detail: blueprint §11.
