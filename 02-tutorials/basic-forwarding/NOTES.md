# Basic Forwarding - Exercise 1

## What I learned

### P4 Language Basics
- P4 programs target a specific architecture (v1model for BMv2) which defines the pipeline: Parser → VerifyChecksum → Ingress → Egress → ComputeChecksum → Deparser
- `bit<N>` is the core type — everything is defined by bit width (e.g. bit<48> for MAC, bit<32> for IPv4)
- `header` is a special type with a built-in isValid() method — becomes true only after extract()
- `typedef` creates aliases for readability (macAddr_t, ip4Addr_t, egressSpec_t)
- Parsers use state machines with `transition select()` for branching — not if/else
- Control blocks use normal `if` statements and contain actions, tables, and an apply block

### Networking Concepts
- Ethernet header is always first — etherType field tells you what protocol follows (0x800 = IPv4)
- IP addresses stay the same end-to-end, MAC addresses get rewritten at every hop
- TTL decrements at each hop to prevent packets from looping forever
- Checksum must be recalculated after modifying any header field (like TTL)
- Egress port = which physical port the packet leaves the switch from

### P4 Architecture (Data Plane vs Control Plane)
- The P4 program defines table structure, match types, and actions (data plane)
- The control plane populates tables with actual rules (via sX-runtime.json or P4Runtime)
- Match types include: exact, lpm (longest prefix match), ternary
- Action parameters come from the control plane, not from the P4 code
- P4 is protocol-independent — you define every header and behavior from scratch

### Build System
- `make run` compiles P4, starts BMv2 switches, loads table entries, launches Mininet
- `simple_switch_grpc` is the BMv2 executable with gRPC interface for control plane communication
- Topology defined in topology.json, table rules in sX-runtime.json

## TODOs I completed
1. **Parser**: extract Ethernet, transition select on etherType, extract IPv4 in separate state
2. **ipv4_forward**: set egress port, rewrite src/dst MAC, decrement TTL
3. **Apply block**: wrapped ipv4_lpm.apply() in hdr.ipv4.isValid() check
4. **Deparser**: emit Ethernet then IPv4 (emit auto-skips invalid headers)

## Result
- h1 ping h2: success
- pingall: all hosts can reach each other across the full pod-topo topology
