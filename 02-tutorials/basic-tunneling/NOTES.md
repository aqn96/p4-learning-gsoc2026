# Basic Tunneling Exercise

## What is Tunneling?
Tunneling is when you wrap an original packet inside a new header that overrides normal routing. The switches ignore the original IP address and route based on the tunnel header instead. This is the same core concept behind VPNs, MPLS, VXLAN, GRE — you slap an extra header on, route on that, and strip it at the end.

## How VPNs Use This
Your device takes the original IP packet, encrypts it, wraps it in a new header addressed to the VPN server. Every router in between only sees the outer header — they have no idea what's inside. The VPN server strips the outer header, decrypts the original, and forwards it from its own IP. The website thinks the request came from the VPN server, not you. Return traffic follows the same path in reverse. You're not invisible though — you're just trusting the VPN provider instead of your ISP.

## The myTunnel Header
Custom header with two 16-bit fields:
- `proto_id` — tells the switch what's encapsulated inside (0x0800 = IPv4). Like etherType but one layer deeper.
- `dst_id` — destination identifier the switch uses to pick the output port. Not an IP address — just a simple ID. 16 bits gives 65,536 possible destinations which is overkill for 3 hosts but way simpler than a full 32-bit IP.

The etherType for myTunnel is 0x1212.

## What I Built

### Parser
Added a new parser state `parse_myTunnel`. The parser checks etherType — if 0x1212 go to tunnel state, if 0x0800 go to IPv4. Inside the tunnel state, check proto_id to see if there's an IPv4 packet inside and parse that too. The packet structure when tunneled: Ethernet | myTunnel | IPv4 | TCP | payload.

### Action: myTunnel_forward
Simpler than ipv4_forward — just sets `standard_metadata.egress_spec` to the port. No MAC updates, no TTL decrement. Tunnel forwarding doesn't care about L2 stuff.

### Table: myTunnel_exact
Exact match on `hdr.myTunnel.dst_id`. Exact match means the field must match perfectly — like a hash map lookup. Different from LPM which matches prefixes. For a simple destination ID, exact match makes sense.

### Apply Block
Tunnel check comes FIRST. If myTunnel header is valid, use tunnel table. Else if IPv4 is valid, use IP table. Order matters because a tunneled packet has BOTH valid headers (parser extracts both). Whichever you check first wins — tunnel should win because that's the whole point.

### Deparser
Emit ethernet, then myTunnel, then ipv4. Deparser only emits valid headers automatically so no if-statements needed.

### Control Plane (JSON files)
Each switch gets its own sX-runtime.json with tunnel entries mapping dst_id to output ports. Same dst_id maps to different ports on different switches because their physical connections are different. But the end result is the same — dst_id 2 always reaches h2 no matter which switch you're on.

## Key Concepts

### Tunnel Persists Across Hops
The tunnel header stays on the packet the entire journey. Every switch sees the same dst_id and uses its own table to decide the output port. The header isn't stripped at each hop — it rides along the whole way.

### Every Switch Runs the Same P4 Code
make run compiles basic_tunnel.p4 and loads it on all switches. They all run the same forwarding logic but get different control plane rules (different JSON files). Same program, different table entries.

### Match Types
- `exact` — field must match perfectly, simple lookup
- `lpm` — longest prefix match, for IP routing with subnets
- `ternary` — bitmask match, most flexible

### P4 is Case Sensitive
`hdr.myTunnel` and `hdr.mytunnel` are different. `standard_metadata` vs `standard_metadata_t`... one is the instance, the other is the type. Got burned by this a few times and made a lot of issues.

## Testing
All three test cases worked:
1. `./send.py 10.0.2.2 "P4 is very cool language"` — normal IP forwarding, h2 receives it
2. `./send.py 10.0.2.2 "P4 is very cool language" --dst_id 2` — tunneled, h2 receives it with tunnel header visible
3. `./send.py 10.0.3.3 "P4 is very cool language" --dst_id 2` — IP says h3 but tunnel says h2. h2 gets it anyway. This proves the tunnel overrides IP routing.

For contribution/regression testing work, I also implemented a PTF suite in
`p4lang/tutorials` PR #733 with seven automated tests:

- `Ipv4DropOnMissTest`
- `Ipv4ForwardTest`
- `TunnelForwardTest`
- `TunnelDropOnMissTest`
- `MixedTrafficTest`
- `TunnelUnknownProtoTest`
- `TtlBoundaryTest`

This complements manual `send.py` verification by pinning expected behavior in
repeatable tests for CI and future compiler/runtime upgrades.

## Learnings

- Tunneling is the same fundamental concept behind VPNs, MPLS, VXLAN — wrap, route on the wrapper, unwrap. Once you get that pattern everything else is just variations on it.
- You can define whatever header fields you want in P4. dst_id doesn't need to be an IP address — 16 bits is plenty when you only have a few destinations. That's the power of P4, you're not stuck with existing protocols.
- Order in the apply block matters a lot. A tunneled packet has both myTunnel and IPv4 headers valid (parser extracts both). If you check IPv4 first the tunnel never gets used. Took me a bit to get why tunnel has to be checked first.
- Case sensitivity in P4 will get you. `hdr.myTunnel` vs `hdr.mytunnel`, `standard_metadata` vs `standard_metadata_t`. The _t is the type, without _t is the instance. Same as class vs object.
- The control plane JSON files are where the actual forwarding rules live. Same P4 program on every switch, different JSON rules per switch because they're in different positions in the topology.
- To make this more realistic, switches should add/remove tunnel headers instead of hosts. Ingress switch maps destination IP to dst_id and adds the header with `setValid()`. Egress switch recognizes it's the last hop and strips the header with `setInvalid()`. Transit switches just forward on dst_id. The original IP packet stays untouched inside the tunnel the whole time. This is basically how MPLS works in production.

