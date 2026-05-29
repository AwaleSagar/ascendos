# Networking

> Status: outline. A Phase 4 concern; documented here for completeness, not yet
> designed in depth.

A user-space stack (`netstackd`): sockets-as-capabilities, IPv6-first dual stack,
zero-copy RX/TX via shared frames with the NIC driver. Firewall policy is
expressed as capability policy enforced via `secd`; there is no global ambient
port namespace.

## This approach has precedent (and we should reuse, not reinvent)

A user-space TCP/IP stack is well-trodden ground, which de-risks it:

- **Redox OS** runs its network stack in user space on **smoltcp**, a
  standalone, `no_std`, heapless Rust TCP/IP stack — a strong candidate to port
  rather than write from scratch.
- **ixy.rs** demonstrated a memory-safe user-space NIC driver ported to Redox —
  direct evidence the confined-driver model works for networking.
- High-performance precedents (DPDK-based Z-stack, Snabb, F-Stack) confirm that
  user-space stacks with zero-copy can match or beat in-kernel throughput; they
  also show the cost is real engineering, not a free lunch.

Decision leaning (to be confirmed by an ADR): **port smoltcp** for the initial
stack rather than writing one, given it already targets exactly our constraints.

## A correction on QUIC

Earlier wording called the design "QUIC-ready L4". That's imprecise: **QUIC runs
over UDP** (RFC 9000) — it is an application/transport protocol layered on a UDP
socket, not a peer of TCP/UDP at L4. The accurate statement is: the socket layer
must expose UDP well enough that a user-space QUIC library can sit on top. No
special L4 slot is needed.

## Open question

- Congestion-control pluggability and offload negotiation (checksum/segmentation)
  across the driver capability boundary need real design.

## Sources

- smoltcp / Redox user-space stack — Redox 0.3.5 release notes; "Porting ixy.rs
  to Redox" (TUM).
- QUIC over UDP — RFC 9000 (IETF).
- User-space stack performance — "A High-performance DPDK-based Zero-copy TCP/IP
  Protocol Stack" (Z-stack, NSF/PAR).

Full detail: blueprint §9.
