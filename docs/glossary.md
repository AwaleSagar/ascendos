# Glossary

Terms used throughout the AscendOS docs.

- **ASID** — Address Space Identifier. Tags TLB entries so address spaces don't
  require a full TLB flush on context switch.
- **Ambient authority** — authority a process has implicitly from its identity
  (e.g. "root can do anything"), rather than from holding an explicit token.
  AscendOS deliberately has none.
- **Capability** — an unforgeable, kernel-managed reference naming an object and
  the rights over it. The only source of authority in AscendOS.
- **CCA** — ARM Confidential Compute Architecture; the ARMv9 feature set for
  Realms (isolated confidential VMs).
- **CDI** — Compound Device Identifier; a secret derived during DICE boot
  measurement, the basis of the attestation chain.
- **CSpace** — the capability address space of a thread; the set of capabilities
  it holds.
- **DICE** — Device Identifier Composition Engine; a layered boot-measurement
  scheme where each stage measures the next.
- **EEVDF** — Earliest Eligible Virtual Deadline First; a scheduling algorithm
  offering fairness with explicit latency targets.
- **EL0–EL3** — ARM64 Exception Levels (privilege levels). EL0 user, EL1 kernel,
  EL2 hypervisor, EL3 secure monitor.
- **MTE** — Memory Tagging Extension (ARMv9); tags memory and pointers to catch
  spatial/temporal safety violations.
- **PAC / BTI** — Pointer Authentication / Branch Target Identification; ARM
  control-flow-integrity features.
- **Realm** — an isolated confidential execution environment under ARM CCA.
- **SchedContext** — a scheduling-context capability representing a CPU time
  budget a thread runs against.
- **SMMU** — System Memory Management Unit; the IOMMU on ARM, used to bound
  device DMA to granted memory.
- **TCB** — Trusted Computing Base; the code that must be correct for the
  security of the system to hold. AscendOS aims to keep it ≤ 15 KLOC.
- **Untyped** — a capability to raw physical memory, from which typed kernel
  objects are created by *retyping*.
