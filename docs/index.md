# AscendOS

*A capability-based ARM64 microkernel — design blueprint, pre-implementation.*

!!! warning "This is a design document, not working software"
    There is no code here yet. Nothing boots. Performance figures are targets,
    not measurements. This site documents a design and the reasoning behind it,
    and exists so people can poke holes in it before a kernel is written.

## Where to start

- **Why this exists:** [Motivation](overview/motivation.md) and
  [Non-goals](overview/non-goals.md).
- **The design:** [Architecture](architecture/README.md) — read **Boot** first.
- **The contested decisions:** [ADRs](adr/README.md). Start with
  [ADR-0003](adr/0003-ring-based-syscall-interface.md), the design's biggest bet.
- **Where it's going:** the [roadmap](https://github.com/AwaleSagar/ascendos/blob/main/ROADMAP.md).

The single-file master spec is in the repository under
[`blueprint/`](https://github.com/AwaleSagar/ascendos/blob/main/blueprint/AscendOS-Architecture-Blueprint.md).
These pages are a split, navigable view of it.
