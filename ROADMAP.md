# Roadmap

This roadmap is honest about uncertainty. Dates are intentions, not promises,
and later phases are genuinely *unknown* in scope until the earlier ones teach
us more. If a phase slips, that is expected, not a failure.

Confidence legend: **(firm)** we know how to do this · **(likely)** plausible,
some unknowns · **(aspirational)** depends on what earlier phases reveal.

---

## Phase 0 — Blueprint **(firm)** — *you are here*

The design exists on paper and is open for critique.

- [x] Architecture blueprint (single-file master spec)
- [x] Documentation scaffold and decision records
- [x] Community process (contributing, governance, RFCs)
- [ ] First external review pass — *help wanted*

**Exit criteria:** at least one round of substantive outside critique on the
capability model and the ring-based syscall interface.

**Biggest unknown:** whether the ring-based interface (ADR-0003) holds up under
scrutiny, or whether we fall back to a more conventional trap-based design.

---

## Phase 1 — Validation **(likely)**

Turn the blueprint into something defensible before committing to bring-up.

- [ ] Choose implementation language and toolchain (leaning Rust; not decided)
- [ ] Paper-prototype the boot path (S0–S8) against a concrete reference SoC
- [ ] Pin down the capability object set and IPC ABI in detail
- [ ] Resolve open questions logged across the architecture pages

**Exit criteria:** a frozen capability ABI draft and a chosen toolchain.

**Biggest unknown:** how much the ARMv9 confidential-compute (CCA) ambitions
constrain the EL2/EL3 design if we want them from the start vs. later.

---

## Phase 2 — Bring-up **(aspirational)**

First time anything runs.

- [ ] Boot to EL1 under QEMU `virt`
- [ ] Minimal kernel init: page tables, capability space, one scheduler
- [ ] First IPC primitive working end-to-end

**Exit criteria:** a root task prints to a serial console.

---

## Phase 3 — Minimal services **(aspirational)**

- [ ] Root task hands off to `svcd`
- [ ] A real UART driver in user space
- [ ] An interactive serial console / shell stub

---

## Phase 4 — Subsystems **(aspirational)**

Storage, networking, and packaging — one at a time, each with its own RFC and
exit criteria. Not planned in detail until Phase 3 teaches us what the service
model really costs.

---

*Changes to this roadmap go through a normal pull request. Significant scope
changes should reference an RFC.*
