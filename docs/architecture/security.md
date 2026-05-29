# Security

Security in AscendOS is a property of the whole design, not a subsystem. It shows
up in every band.

| Pillar | Mechanism |
|---|---|
| Authority | Pure capabilities; least privilege by construction ([ADR-0001](../adr/0001-pure-capability-model.md)) |
| Boot integrity | Secure boot + DICE layered attestation ([Boot](boot.md)) |
| Memory safety | Memory-safe kernel language + ARMv9 MTE tagging |
| Control-flow integrity | PAC (pointer auth) + BTI (branch target) |
| Isolation | Per-process address spaces, SMMU-scoped DMA, user-space drivers |
| Confidential compute | ARMv9 CCA Realms via optional `realmd`/EL2 |
| Policy & audit | `secd` as a central decision point + append-only audit log |
| Safe extensibility | Verified bytecode sandbox; no native kernel modules ([ADR-0004](../adr/0004-no-native-kernel-modules.md)) |

## A note on threat model

We don't yet have a written threat model, and we should. What's an attacker
assumed to control? What's out of scope (physical attacks? side channels?)?
Until that document exists, treat the security claims here as design *intent*
that hasn't been adversarially reviewed. This is a known gap and a good place to
contribute.

Full detail: blueprint §11.
