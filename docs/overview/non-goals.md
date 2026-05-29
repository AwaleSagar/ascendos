# Non-goals

Being explicit about what we will *not* do is as important as the goals. These
are deliberate boundaries, not gaps waiting to be filled.

- **No GUI subsystem.** No compositor, no framebuffer stack, no font rendering.
  Not now, not later. This is a CLI-only system.
- **No multi-architecture support (for now).** ARM64 only. Portability
  abstraction has a real design cost and we're not paying it until the ARM64
  design is proven.
- **No POSIX compatibility goal.** We may end up offering a compatibility shim in
  user space eventually, but the kernel ABI is not designed to be POSIX, and
  source compatibility with Linux is explicitly not a target.
- **No binary compatibility with any existing OS.**
- **No in-kernel drivers.** Ever. See
  [ADR-0002](../adr/0002-userspace-drivers.md).
- **No native loadable kernel modules.** Extension happens in user space or
  through a sandboxed bytecode VM. See
  [ADR-0004](../adr/0004-no-native-kernel-modules.md).
- **Not chasing benchmarks before correctness.** Performance targets exist in the
  docs, but the first goal is a design that's right, not one that's fast on a
  microbenchmark.

If you want one of these, AscendOS is probably not the project for you — and
that's fine. Saying no to these is what keeps the rest tractable.
