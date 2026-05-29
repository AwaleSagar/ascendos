# Observability

> Status: outline. Phase 4 concern.

A single hub (`obsd`) over three sources: static kernel tracepoints (near-zero
cost when disabled), service metrics, and dynamic probes attached at runtime via
a safe bytecode VM (eBPF-class — no native modules, preserving the TCB). Metrics,
traces, and logs flow through one pipeline with OpenTelemetry-compatible export
and a CLI query surface (`trace`, `metrics`, `log`).

Full detail: blueprint §13.
