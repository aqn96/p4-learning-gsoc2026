# P4 Fundamentals — Learning Notes

## What is P4?

P4 (Programming Protocol-independent Packet Processors) is a domain-specific language for programming the data plane of network devices. The key word is "protocol-independent" — P4 has zero built-in knowledge of any protocol. You define what headers look like, how to parse them, and what to do with them. IPv4, Ethernet, or a completely custom protocol you invented — P4 treats them all the same.

Before P4, the behavior of networking ASICs was hardcoded by the manufacturer. If you wanted new packet processing behavior, you waited years for the next generation of hardware. P4 changed this by making the data plane programmable.

## The P4 Pipeline

Every P4 program follows three stages:

1. **Parse** — the packet arrives as raw bytes. Your P4 parser pulls apart the headers into structured fields. You're telling the switch "here's how to read this packet."

2. **Match-Action** — the core logic. You define tables that match on header fields (like destination IP) and execute actions (like "set output port to 3, rewrite the MAC address"). The control plane populates these tables with rules.

3. **Deparse** — reassemble the packet with any modifications and send it out. This could mean the packet leaves unchanged, with modified fields, or with entirely new headers added.

## P4 Compilation Targets

The same P4 source code can be compiled for different targets:

### BMv2 (Behavioral Model v2)
- Software switch for **development and testing**
- p4c compiles P4 → JSON, BMv2 interprets the JSON
- Runs on any machine, paired with Mininet for virtual network topologies
- Slow but feature-complete, supports the full P4 spec
- This is what the tutorials use

### eBPF (extended Berkeley Packet Filter)
- Runs P4 logic inside the **Linux kernel**
- p4c compiles P4 → eBPF bytecode → loaded into kernel at runtime
- Processes real packets on real network interfaces
- Faster than BMv2 but has constraints (program size limits, restricted operations)
- Good middle ground between simulation and hardware

### DPDK (Data Plane Development Kit)
- P4 logic runs in **user space**, bypassing the kernel entirely
- DPDK takes over the NIC and polls for packets directly
- Much faster than kernel-based processing — eliminates interrupt and context switch overhead
- This is what P4Pi uses on the Raspberry Pi

### Hardware ASICs (e.g., Intel Tofino)
- P4 logic is loaded onto **dedicated networking chips**
- Processing happens in parallel circuitry at wire speed (terabits/second)
- The ultimate deployment target for production networks
- Expensive hardware, but unmatched performance

### Typical development workflow
BMv2 + Mininet (develop/test) → eBPF/DPDK (test with real traffic) → ASIC (production deployment)

## Networking Fundamentals

### MAC Addresses vs IP Addresses
- **MAC address** — Layer 2, 48-bit hardware identifier burned into the NIC by the manufacturer. Used for communication on the local network segment. Does not change. Protocol-independent (works with both IPv4 and IPv6).
- **IP address** — Layer 3, assigned by the network (not the manufacturer). Used for routing across networks. Can change. Either IPv4 (32-bit) or IPv6 (128-bit).

### Why MAC Addresses Get Rewritten at Each Hop
When a packet moves between Layer 3 hops, the IP addresses (source and destination) stay the same end-to-end, but the MAC addresses change at every hop to reflect the current link segment. The source MAC becomes the current router's MAC, and the destination MAC becomes the next-hop router's MAC.

### IPv4 Exhaustion and NAT
There are only ~4.3 billion IPv4 addresses. IANA exhausted its free pool in February 2011. The internet still works on IPv4 because of:

- **NAT (Network Address Translation)** — many devices share one public IP. Your router rewrites the source address from your private IP (192.168.x.x) to its one public IP before packets hit the internet. Return traffic gets translated back.
- **CGNAT (Carrier-Grade NAT)** — ISPs add another layer of NAT, putting thousands of customers behind a small pool of public IPs. This creates three layers: device private IP → ISP internal IP → actual public IP.
- **IPv4 transfer market** — organizations buy/sell address blocks on a secondary market.

### IPv6 Adoption
IPv6 (128-bit addresses) was designed to solve exhaustion, but global adoption is still only ~45-49% as of 2025. Most networks run dual-stack (both IPv4 and IPv6 simultaneously). Modern devices support both.

### TTL (Time to Live)
A counter in the IP header that decrements by 1 at each hop. When it reaches 0, the packet is dropped. This prevents packets from looping forever in the network. IPv6 calls it "Hop Limit" but it works the same way.

## ASICs in Networking

ASIC = Application-Specific Integrated Circuit. A chip designed for one specific job.

- **Traditional ASICs** — packet processing behavior is hardcoded by the manufacturer. Fast but inflexible.
- **Programmable ASICs (e.g., Tofino)** — have reconfigurable pipeline stages that P4 programs define. Your P4 code configures how the physical circuits behave. Fast AND flexible.

Networking ASICs live in **switches and routers**, not in NICs. A NIC is the network adapter in your computer. The ASIC is the forwarding chip inside the switch that connects many devices together.

**SmartNICs** blur this line — they have their own programmable processors and can run P4, doing packet processing at the network interface before the host CPU gets involved.

## In-Network Machine Learning (Planter)

### The Concept
Instead of running ML inference on a separate server, embed the inference logic directly into the packet processing pipeline. Every packet gets classified as it flows through the switch with zero additional latency.

### How Planter Works
1. **Train** a model (decision tree, random forest, SVM, etc.) using normal ML tools like scikit-learn
2. **Convert** — Planter walks through the trained model's structure and translates each decision path into a P4 match-action table entry
3. **Deploy** — the generated P4 code is compiled and loaded onto the target device. No ML framework runs on the device — it's just table lookups.

### Why It Works for Certain Models
A decision tree is a series of if/else branches. Each branch maps directly to a table entry:
- Match: dst_port < 1024, pkt_size > 500 → Action: classify as normal
- Match: dst_port >= 1024, src_ip = 10.0.0.0/8 → Action: classify as suspicious

The model's learned knowledge becomes table rules. The accuracy is identical to the original trained model — nothing is lost in conversion.

### Why Not Large Neural Networks
Neural networks use matrix multiplications, activation functions, and many layers of computation. These don't map naturally to match-action table lookups. Planter supports some small neural networks, but deep learning models with millions of parameters won't fit.

### Supported Models
Decision Trees, Random Forests, XGBoost, KNN, Naive Bayes, SVM, and some lightweight neural networks. Each model type uses a different mapping strategy (Encoding-Based, Lookup-Based, or Direct-Mapping).

### Applications
- Network intrusion detection / anomaly detection
- Traffic classification (video, web, IoT, etc.)
- QoS enforcement based on learned traffic patterns
- Financial transaction classification

## The GSoC Project: P4Pi + Planter + p4c-dpdk

### P4Pi
A Raspberry Pi-based P4 platform. It turns the Pi into a real programmable network device with P4 tools pre-installed.

### DPDK on the Pi
DPDK takes over the Pi's NIC and processes packets in user space, bypassing the kernel. The CPU polls the NIC directly for packets — no interrupts, no context switches, no kernel overhead.

### The Integration
The project integrates Planter into P4Pi so that:
- ML models can be trained on a development machine
- Planter converts them to P4 table logic
- p4c-dpdk compiles the P4 code for the Pi
- The Pi runs real-time ML-powered traffic classification on live packets
- All of this is packaged as a ready-to-use P4Pi system image

### Why It Matters
A $50 Raspberry Pi can do ML-powered packet classification at data plane speed. Students and researchers get hands-on experience with in-network ML without expensive hardware.
