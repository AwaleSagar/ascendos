# AscendOS — ARM64 Microkernel OS

## Master Architecture Blueprint (Figma-Ready Specification)

> **Scope:** Architecture diagram specification only. No source code.
> **Target:** ARM64 (ARMv8.2-A+ / ARMv9-A), CLI-only, no GUI subsystem.
> **Design era:** 2026+ design conventions.
> **Purpose:** Direct translation into a Figma master diagram and the canonical reference for kernel development.

---

## 0. Research Basis (Design Decisions Grounded in Modern Trends)

| Trend (2024–2026) | Decision in AscendOS |
|---|---|
| Capability-based microkernels (seL4, Redox, Fuchsia/Zircon) | Pure capability model; no ambient authority |
| Formal verification momentum (seL4, Rust kernels) | Kernel written in a memory-safe systems language; verifiable TCB < 15 KLOC |
| Async, completion-based I/O (io_uring) | Shared-memory ring queues for ALL syscalls; syscalls become the slow path |
| EEVDF / capacity-aware scheduling | Latency-virtual-deadline scheduler, DynamIQ big.LITTLE aware |
| ARMv9 confidential compute (CCA / Realms), MTE, PAC, BTI | First-class memory tagging, pointer auth, realm isolation |
| Measured + secure boot (TPM/DICE, fTPM) | DICE-based layered attestation chain |
| Driver isolation in user space (Fuchsia, microkernels) | All drivers are user-space capability-confined processes |
| Declarative, immutable, atomic package systems (Nix, OSTree) | Content-addressed, atomic, rollback-capable packages |
| eBPF-style safe extensibility | Sandboxed bytecode VM for observability + policy hooks |

---

## 1. Top-Level System Layering (The Spine)

This is the primary vertical diagram — the backbone of the Figma canvas. Five horizontal bands, kernel/user boundary drawn as a bold line between Band 2 and Band 3.

```mermaid
flowchart TB
    subgraph B5["BAND 5 — Operator Surface (CLI)"]
        SHELL["ash — Ascend Shell"]
        CLITOOLS["Core CLI Utilities"]
        PKGCLI["pkg — Package CLI"]
        OBSCLI["trace / metrics CLI"]
    end

    subgraph B4["BAND 4 — System Services (User Space, Privileged Capabilities)"]
        INIT["svcd — Service Manager / init"]
        NET["netstackd — Network Stack"]
        FS["vfsd — VFS + Filesystems"]
        STORE["blockd — Storage/Block"]
        SEC["secd — Security & Attestation"]
        PKGD["pkgd — Package Daemon"]
        OBSD["obsd — Observability Hub"]
        DEVMGR["devmgr — Device Manager"]
    end

    subgraph B3["BAND 3 — User-Space Drivers & Runtime (Confined)"]
        DRV["Device Drivers (isolated)"]
        LIBOS["libos / runtime shims"]
        VMM["realmd — Realm/VM Monitor (opt)"]
    end

    KBOUND{{"=== KERNEL / USER BOUNDARY (EL1 | EL0) ==="}}

    subgraph B2["BAND 2 — Microkernel (EL1, Minimal TCB)"]
        SCHED["Scheduler"]
        MEM["Memory / Address Spaces"]
        IPC["IPC / Capability Engine"]
        IRQ["IRQ + Timer Core"]
        SYSCALL["Syscall / Ring Dispatch"]
        CAPTBL["Capability Tables"]
    end

    subgraph B1["BAND 1 — Hardware Abstraction & Boot (EL3/EL2 + Firmware)"]
        BOOT["Boot Chain (DICE)"]
        HAL["ARM64 HAL: GIC, SMMU, Timers, MTE/PAC"]
        FW["Secure Monitor (EL3) / Hypervisor (EL2)"]
    end

    B5 --> B4 --> B3 --> KBOUND --> B2 --> B1
```

**Figma layout note:** Render Bands 3–5 in one color family (user space, cool blue), Band 2 in a single high-contrast color (kernel, warm red), Band 1 in neutral graphite (hardware/firmware). The kernel/user boundary line is the single most important visual element — make it a 4px bold divider with the `EL1 | EL0` label centered.

---

## 2. ARM64 Privilege & Exception-Level Map

The substrate the whole design hangs on. One small reference block in a corner of the canvas.

```mermaid
flowchart LR
    EL0["EL0 — User Space\n(apps, drivers, services)"]
    EL1["EL1 — Microkernel\n(scheduler, mem, IPC)"]
    EL2["EL2 — Hypervisor / Realm Mgmt\n(optional: realmd)"]
    EL3["EL3 — Secure Monitor\n(boot root, PSCI, attestation root)"]

    EL0 -->|SVC trap / ring doorbell| EL1
    EL1 -->|HVC| EL2
    EL2 -->|SMC| EL3
    EL3 -->|RMM / CCA| EL2
```

- **EL3 Secure Monitor:** PSCI power management, SMC dispatch, attestation root-of-trust, world switch.
- **EL2:** reserved for optional `realmd`; normally idle to preserve minimal footprint.
- **EL1:** the entire AscendOS microkernel. Target TCB ≤ 15 KLOC.
- **EL0:** everything else, including drivers.

---

## 3. Boot Sequence (Layered Boot Path Diagram)

A dedicated horizontal swimlane diagram. Each stage measures the next (DICE layering) — draw the measurement arrows as dashed upward lines into the TPM/attestation column.

```mermaid
flowchart LR
    S0["S0: BootROM\n(immutable, on-die)\nRoot of Trust"]
    S1["S1: BL1 / SoC FW\nverify BL2"]
    S2["S2: BL2 Platform Init\nDRAM, clocks, SMMU"]
    S3["S3: BL31 EL3 Monitor\nPSCI, secure world"]
    S4["S4: Boot Loader (EL2/EL1)\nload kernel + DTB"]
    S5["S5: Microkernel Init (EL1)\ncaps, mem, sched bring-up"]
    S6["S6: Root Task\nspawn svcd"]
    S7["S7: svcd\nlaunch core services"]
    S8["S8: ash shell\noperator ready"]

    S0 --> S1 --> S2 --> S3 --> S4 --> S5 --> S6 --> S7 --> S8

    DICE[("DICE / TPM\nLayered Measurements")]
    S0 -.measure.-> DICE
    S1 -.measure.-> DICE
    S2 -.measure.-> DICE
    S3 -.measure.-> DICE
    S4 -.measure.-> DICE
    S5 -.measure.-> DICE
```

**Boot path responsibilities table:**

| Stage | EL | Responsibility | Footprint Target |
|---|---|---|---|
| S0 BootROM | EL3 | Verify first mutable stage, derive CDI | n/a (ROM) |
| S1 BL1 | EL3 | Minimal init, verify BL2 signature | < 64 KB |
| S2 BL2 | EL3 | DRAM/clock/SMMU init, load BL31 | < 256 KB |
| S3 BL31 | EL3 | Install secure monitor, PSCI | < 128 KB |
| S4 Loader | EL2→EL1 | Parse DTB, relocate kernel, pass boot caps | < 128 KB |
| S5 Kernel init | EL1 | Page tables, cap space, per-CPU sched, IRQ | kernel image < 256 KB |
| S6 Root task | EL0 | Holds all bootstrap capabilities, hands to svcd | minimal |
| S7 svcd | EL0 | Dependency-ordered parallel service launch | — |
| S8 ash | EL0 | Operator login / CLI | — |

**Cold-boot target:** power-on → shell prompt in < 250 ms on reference SoC.

---

## 4. Microkernel Internals (Band 2 Detail)

The single most detailed block in the diagram. Expand into a sub-canvas / Figma component with internal sub-blocks.

```mermaid
flowchart TB
    subgraph KERNEL["Microkernel (EL1) — Minimal TCB"]
        direction TB
        ENTRY["Trap & Ring Dispatch\n(SVC + shared-memory doorbells)"]
        CAP["Capability Engine\n(derive, revoke, badge, mint)"]
        SCHED["Scheduler\n(EEVDF-style, capacity-aware)"]
        MM["Memory Manager\n(untyped → typed retype)"]
        ASID["Address-Space / ASID Mgr"]
        IPC["Synchronous Endpoints +\nAsync Notification Words"]
        IRQ["IRQ Objects + Timer"]
        FAULT["Fault Handler\n(routes to user pager)"]
    end

    ENTRY --> CAP
    CAP --> SCHED & MM & IPC & IRQ
    MM --> ASID
    IPC --> SCHED
    IRQ --> SCHED
    FAULT --> IPC
```

### 4.1 Component Responsibilities

- **Trap & Ring Dispatch:** Two entry paths. (a) Classic `SVC` trap for control ops. (b) **io_uring-style shared-memory submission/completion rings** per thread for high-frequency calls — the kernel polls/wakes on doorbell, eliminating most trap overhead. This is the defining performance feature.
- **Capability Engine:** All authority is an unforgeable capability stored in kernel-managed CSpace. Operations: `mint`, `derive`, `badge` (identity tagging), `revoke` (recursive). No ambient authority anywhere — mirrors seL4/Zircon.
- **Scheduler:** Per-CPU runqueues, EEVDF-style virtual-deadline fairness, **capacity-aware placement** for DynamIQ big.LITTLE (E-cores vs P-cores), scheduling contexts (CPU budget is itself a capability → enables hard real-time + safe over-commit). Work-stealing across same-cluster cores.
- **Memory Manager:** Single physical-memory model via **untyped → retype** (seL4 pattern): all kernel objects are user-allocated from untyped capabilities, so the kernel never has an internal heap → no in-kernel OOM, fully accountable memory. ARMv9 **MTE** tags enforced; **PAC/BTI** for control-flow integrity.
- **Address-Space / ASID Manager:** Multi-level translation tables, ASID recycling, optional 52-bit VA, contiguous-hint (block) mappings for TLB efficiency.
- **IPC:** Synchronous fastpath endpoints (register-passed short messages) + asynchronous **notification words** (bitfield signals). Zero-copy bulk transfer via shared-frame capability grants.
- **IRQ + Timer:** IRQs are capability objects delivered as notifications to user-space drivers; per-CPU generic timer; tickless idle.
- **Fault Handler:** Page/permission faults are converted to IPC messages routed to the responsible **user-space pager** — kernel itself never does demand paging policy.

### 4.2 What is explicitly NOT in the kernel
Filesystems, network stack, drivers, paging policy, package logic, naming/discovery, scheduling *policy* beyond the mechanism. All in user space.

---

## 5. Kernel Objects & Capability Model

Reference table block — pair with a small entity diagram.

| Object | Represents | Key Operations |
|---|---|---|
| `Untyped` | Raw physical memory region | retype → any object |
| `CNode` | Capability storage slot array | copy, mint, delete |
| `TCB` | Thread control block | configure, resume, bind sched-context |
| `Endpoint` | Synchronous IPC port | send, recv, call, reply |
| `Notification` | Async signal word | signal, wait, poll |
| `Frame` | Mappable page | map, unmap, grant |
| `PageTable` | Translation level | map into ASID |
| `IRQHandler` | A hardware interrupt line | ack, set-notification |
| `SchedContext` | CPU time budget | bind, refill params |
| `RealmCap` | Confidential VM context (opt) | create, attest, enter |

```mermaid
flowchart LR
    UT["Untyped"] -->|retype| TCB & EP["Endpoint"] & FR["Frame"] & CN["CNode"] & SC["SchedContext"]
    CN -->|holds| CAPS["Capabilities (badged)"]
    TCB -->|binds| SC
    TCB -->|cspace| CN
    FR -->|map| AS["Address Space"]
```

---

## 6. IPC & Data Flow (The Nervous System)

Show this as a request/response sequence — it explains how a syscall-free, ring-based, capability-confined system actually moves data.

```mermaid
sequenceDiagram
    participant App as App (EL0)
    participant Ring as SQ/CQ Ring (shared mem)
    participant K as Microkernel (EL1)
    participant Svc as Service (EL0, e.g. vfsd)

    App->>Ring: enqueue request (cap badge + op)
    App->>K: doorbell (if service idle)
    K->>K: validate capability + badge
    K->>Svc: deliver via Endpoint / notification
    Svc->>Svc: process (zero-copy shared frame)
    Svc->>Ring: enqueue completion
    Svc->>K: signal notification
    K->>App: wake on completion word
    App->>Ring: dequeue result
```

**Data-flow principles for the diagram:**
1. **Control plane** (capability checks, wakeups) = solid red arrows through kernel.
2. **Data plane** (bulk bytes) = green arrows that *bypass* the kernel via shared frames — draw them going around Band 2, never through it. This visual asymmetry is the whole point.
3. Every cross-process arrow must originate from a held capability badge.

---

## 7. System Services Mesh (Band 4 Detail)

Each service is an isolated EL0 process holding a precise capability set. Draw as a hub diagram with `svcd` at center and dependency arrows.

```mermaid
flowchart TB
    SVCD["svcd (init / supervisor)\nlifecycle, deps, restart, health"]

    SVCD --> DEVMGR & SECD & VFSD & NETD & PKGD & OBSD

    DEVMGR["devmgr\nenumerate, bind drivers,\nresource caps"]
    SECD["secd\nattestation, policy,\nkey custody, audit"]
    VFSD["vfsd\nVFS mux, namespaces,\nmount table"]
    NETD["netstackd\nTCP/IP, sockets"]
    PKGD["pkgd\natomic pkg ops, rollback"]
    OBSD["obsd\nmetrics, traces, logs"]

    DEVMGR --> DRIVERS["User-space Drivers\n(NIC, NVMe, UART, RTC...)"]
    VFSD --> BLOCKD["blockd\nblock cache, scheduler"]
    BLOCKD --> DRIVERS
    NETD --> DRIVERS
    PKGD --> VFSD
    OBSD -. taps .-> NETD & VFSD & DEVMGR & SVCD
    SECD -. policy gate .-> SVCD & DEVMGR & NETD
```

### Service responsibilities

- **svcd:** PID 1 equivalent. Declarative dependency graph, parallel start, capability handout, supervision/restart with backoff, health probes, socket/endpoint activation (lazy start).
- **devmgr:** Walks the device tree (DTB) / ACPI, allocates MMIO + IRQ + DMA(SMMU) capabilities, matches and spawns the correct driver process, hot-plug events.
- **secd:** Holds attestation evidence, evaluates policy (capability-grant decisions), key custody, immutable audit log, secure-boot/runtime measurement verification.
- **vfsd:** Per-process composable namespaces (Plan 9-influenced), mount table, path resolution, mux to filesystem implementations.
- **netstackd:** User-space TCP/IP, sockets-as-capabilities, zero-copy via shared frames with NIC driver.
- **pkgd:** Content-addressed store, atomic transactional install/upgrade/rollback, generation management.
- **obsd:** Aggregates metrics/traces/logs, hosts the safe-bytecode probe VM, exposes the CLI query surface.

---

## 8. Storage Subsystem

```mermaid
flowchart TB
    APP2["App / CLI"] --> VFSD2["vfsd (VFS + namespaces)"]
    VFSD2 --> FSIMPL["FS Implementations\n(log-structured / CoW)"]
    FSIMPL --> BLOCKD2["blockd\nbuffer cache, I/O scheduler,\ncheckpoint, TRIM"]
    BLOCKD2 --> NVME["NVMe / eMMC / virtio-blk\nDriver (EL0)"]
    NVME -->|SMMU-guarded DMA| HW["Storage HW"]
    SECD2["secd"] -. encryption keys / integrity .-> FSIMPL
```

- **Filesystem:** Copy-on-write + log-structured hybrid; per-block checksums; atomic snapshots feeding `pkgd` rollback.
- **blockd:** Unified buffer cache, deadline/budget-aware I/O scheduler, write barriers, async via completion rings.
- **Integrity:** Optional per-file authenticated encryption with keys custodied by `secd`; dm-verity-style integrity trees for the immutable base image.
- **DMA safety:** All device DMA passes through **SMMU** translation scoped by capability — a driver can only touch frames it was granted.

---

## 9. Networking Stack

```mermaid
flowchart TB
    APP3["App (socket cap)"] --> NETD3["netstackd"]
    subgraph NETD3["netstackd (EL0)"]
        SOCK["Socket Layer\n(caps, namespaces)"]
        L4["L4: TCP / UDP / QUIC-ready"]
        L3["L3: IPv6-first, IPv4"]
        L2["L2: framing, neighbor"]
    end
    SOCK --> L4 --> L3 --> L2
    L2 --> NICDRV["NIC Driver (EL0)\nring buffers"]
    NICDRV -->|SMMU DMA| NIC["NIC HW"]
    OBSD3["obsd"] -. flow taps .-> NETD3
    SECD3["secd"] -. firewall policy .-> NETD3
```

- **IPv6-first**, dual-stack; QUIC-ready L4; pluggable congestion control.
- **Zero-copy** RX/TX via shared frames between `netstackd` and NIC driver.
- **Sockets are capabilities** — no global port namespace ambient authority; firewall = capability policy enforced via `secd`.
- **Offload-aware:** checksum/segmentation offload negotiated through driver caps.

---

## 10. Device Driver Model

The "drivers in user space" story — a key differentiator. Draw the isolation boundary boldly.

```mermaid
flowchart TB
    DEVMGR4["devmgr"] -->|grants MMIO+IRQ+DMA caps| DRV4
    subgraph DRV4["Driver Process (EL0, confined)"]
        PROBE["probe / bind"]
        MMIO["MMIO access (mapped frames)"]
        IRQH["IRQ handler (notification)"]
        DMA["DMA buffers (SMMU-scoped)"]
        IFACE["Service Interface (endpoint)"]
    end
    DRV4 -->|publishes| BUS["Service Bus (endpoints)"]
    CRASH["Driver crash"] -. supervised restart .-> SVCD5["svcd"]
    SVCD5 -. respawn .-> DRV4
```

- **Isolation:** A faulty driver cannot corrupt the kernel or other drivers — it only holds its own MMIO/IRQ/DMA capabilities. Crash → `svcd` restarts it; state recovered from checkpoints where applicable.
- **Uniform contract:** every driver exposes a typed endpoint interface (block, net, char, bus).
- **Bus drivers:** PCIe, USB, I2C/SPI as their own confined processes that hand child capabilities to leaf drivers.
- **Hot-plug:** `devmgr` reacts to bus events, spawns/reaps driver processes dynamically.

---

## 11. Security Framework (Cross-Cutting Overlay)

Render as a translucent overlay panel touching every band — security is not a layer, it's a property.

```mermaid
flowchart TB
    subgraph SEC["Security Framework"]
        ROT["Root of Trust\n(BootROM + DICE CDI)"]
        ATT["Attestation\n(layered measurements, remote quote)"]
        CAPM["Capability Model\n(no ambient authority)"]
        MTEPAC["HW Mitigations\nMTE + PAC + BTI"]
        REALM["Confidential Compute\nARM CCA Realms (opt)"]
        POL["Policy Engine (secd)\ngrant decisions + audit"]
        SBX["Sandbox VM\nsafe bytecode for probes/policy"]
    end
    ROT --> ATT --> POL
    CAPM --> POL
    MTEPAC -.enforced by.-> CAPM
    REALM --> ATT
    SBX --> POL
```

| Pillar | Mechanism |
|---|---|
| Authority | Pure capabilities; least privilege by construction |
| Boot integrity | Secure boot + DICE layered attestation |
| Memory safety | Safe systems language kernel + MTE tagging |
| Control-flow integrity | PAC (pointer auth) + BTI (branch target) |
| Isolation | Per-process address spaces, SMMU for DMA, user-space drivers |
| Confidential compute | ARMv9 CCA Realms via optional `realmd`/EL2 |
| Policy & audit | `secd` central decision point, append-only audit log |
| Safe extensibility | Verified bytecode sandbox (eBPF-class), no native kernel modules |

---

## 12. Package Management

```mermaid
flowchart LR
    PKGCLI5["pkg CLI"] --> PKGD5["pkgd"]
    PKGD5 --> STORE5["Content-Addressed Store\n(hash-named, immutable)"]
    PKGD5 --> GEN["Generations / Profiles\n(symlink roots)"]
    PKGD5 --> TXN["Atomic Transaction\nstage → verify → swap"]
    SECD6["secd"] -. signature verify .-> TXN
    TXN --> ROLLBACK["Rollback Point\n(snapshot)"]
    PKGD5 --> VFSD6["vfsd mount"]
```

- **Immutable, content-addressed** store (Nix/OSTree lineage): no in-place mutation, perfect reproducibility.
- **Atomic activation:** stage new generation → verify signatures via `secd` → atomic root swap; failure rolls back instantly.
- **Declarative:** desired system state expressed as a manifest; `pkgd` reconciles.
- **A/B base image** support for safe OS upgrades.

---

## 13. Observability Layer

```mermaid
flowchart TB
    subgraph SOURCES["Instrumentation Points"]
        KTRACE["Kernel tracepoints\n(low-overhead, static)"]
        SVCM["Service metrics"]
        PROBES["Dynamic probes\n(safe bytecode VM)"]
    end
    SOURCES --> OBSD7["obsd Hub"]
    OBSD7 --> RING7["Lock-free trace rings"]
    OBSD7 --> METRICS7["Metrics store"]
    OBSD7 --> LOGS7["Structured logs"]
    OBSD7 --> CLI7["trace / metrics / log CLI"]
    OBSD7 --> EXPORT["Export endpoint\n(OpenTelemetry-compatible)"]
```

- **Always-on, low overhead:** static tracepoints compile out to near-zero cost when disabled.
- **Dynamic probes:** attach safe bytecode at runtime (no recompile, no native modules) — eBPF-class capability.
- **Unified pipeline:** metrics + traces + logs through one hub; OTel-compatible export.
- **CLI-native:** `trace`, `metrics`, `log` tools query `obsd` directly over endpoints.

---

## 14. CLI / Operator Surface (Band 5)

- **ash (Ascend Shell):** capability-aware shell — a process inherits only the capabilities the shell explicitly grants; job control, pipelines, namespaces.
- **Core utilities:** coreutils-equivalent, statically composed against `libos`.
- **Admin tools:** `svc` (service control), `pkg`, `dev` (device inspector), `cap` (capability inspector), `trace`/`metrics`/`log`.
- **No GUI subsystem whatsoever** — no compositor, no framebuffer stack, no font/render path. Serial + virtual console only.

---

## 15. Memory Footprint Budget (Minimalism Targets)

| Component | Target Resident |
|---|---|
| Microkernel (EL1) | < 256 KB image, < 1 MB runtime |
| svcd | < 512 KB |
| Each driver | 256 KB – 1 MB |
| netstackd | < 4 MB |
| vfsd + blockd | < 6 MB |
| Idle full system (no apps) | **< 32 MB RAM** |
| Minimum boot config | **< 16 MB RAM** |

---

## 16. Future Extensibility Hooks

```mermaid
flowchart LR
    CORE["Stable Capability ABI"] --> EXT1["New drivers\n(just new EL0 processes)"]
    CORE --> EXT2["New filesystems\n(vfsd plugins)"]
    CORE --> EXT3["Realms / VMs\n(realmd, EL2)"]
    CORE --> EXT4["Safe bytecode\nmodules (probes/policy)"]
    CORE --> EXT5["SMP scale-out\n(per-CPU runqueues)"]
    CORE --> EXT6["Multikernel option\n(per-cluster kernels)"]
```

- **Stable capability ABI** is the only frozen contract; everything above evolves freely.
- **No native kernel modules ever** — extension happens in user space or via the sandbox VM, preserving the verifiable TCB.
- **Scale path:** single core → SMP → NUMA-aware → optional multikernel per cluster.

---

## 17. Figma Construction Guide (How to Lay This Out)

**Canvas:** one large frame, portrait, ~1920 × 4000.

**Color system:**
- Hardware/firmware (Band 1): graphite `#2B2D31`
- Kernel (Band 2): signal red `#E5484D`
- User space (Bands 3–5): blues `#3E63DD` → `#8DA4EF` gradient by altitude
- Security overlay: translucent amber `#FFB224` @ 15% opacity spanning all bands
- Data plane arrows: green `#30A46C`; control plane arrows: red `#E5484D`

**Typography:** one mono family (e.g. Berkeley Mono / JetBrains Mono) for all component labels — reinforces the systems-level identity. 8px grid. 4px corner radius on every block.

**Components to make reusable (Figma components/variants):**
1. *Service block* (title + responsibility list + capability badge slot)
2. *Kernel object pill*
3. *Capability arrow* (with badge tag)
4. *Boot stage card* (stage / EL / footprint)
5. *EL band header*

**Layering order (z-index):**
1. Band backgrounds (bottom)
2. Component blocks
3. Dependency arrows
4. Security overlay (translucent, top)
5. The bold kernel/user boundary divider (always visible on top)

**Diagram set to produce on the canvas:**
1. §1 Spine (hero, top)
2. §2 EL map (top-right reference)
3. §3 Boot swimlane
4. §4 Kernel internals (expanded detail block)
5. §6 IPC sequence
6. §7 Service mesh
7. §8–§10 Subsystem trios (storage / net / drivers) side-by-side
8. §11 Security overlay legend
9. §12–§13 Package + observability
10. §16 Extensibility fan-out (bottom)

---

## 18. One-Page Summary Map (Everything At Once)

```mermaid
flowchart TB
    HW["HARDWARE: ARM64 SoC — cores, GIC, SMMU, MTE/PAC, timers, NVMe, NIC"]
    FW["FIRMWARE: BootROM → BL1/2 → BL31 (EL3) — DICE root of trust"]
    K["MICROKERNEL (EL1): caps · sched(EEVDF) · mem(untyped) · IPC(sync+async rings) · IRQ"]
    DRV["DRIVERS (EL0, confined): NIC · NVMe · UART · bus"]
    SYS["SERVICES (EL0): svcd · devmgr · secd · vfsd · netstackd · blockd · pkgd · obsd"]
    CLI["OPERATOR (EL0): ash shell · pkg · svc · trace · cap"]
    SECO["SECURITY OVERLAY: capabilities · DICE attestation · MTE/PAC/BTI · SMMU · CCA realms"]

    HW --> FW --> K --> DRV --> SYS --> CLI
    SECO -.spans all.-> HW
    SECO -.spans all.-> CLI
```

---

*End of blueprint. This document is the master reference; reproduce each numbered Mermaid block as a Figma diagram following §17.*
