# Networking

> Status: outline. A Phase 4 concern; documented here for completeness, not yet
> designed in depth.

A user-space stack (`netstackd`): sockets-as-capabilities, IPv6-first dual stack,
QUIC-ready L4, zero-copy RX/TX via shared frames with the NIC driver. Firewall
policy is expressed as capability policy enforced via `secd`; there is no global
ambient port namespace.

Open question: congestion-control pluggability and offload negotiation across the
driver capability boundary need real design.

Full detail: blueprint §9.
