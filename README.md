# P4 GSoC 2026 — Onramp & Qualification

Learning journal, tutorial exercises, and qualification task for my Google Summer of Code 2026 application with the [P4 Language Consortium](https://p4.org/).

**Target Project:** [Project 3.3 — Integrating P4-based In-Network Machine Learning framework into P4Pi](https://github.com/p4lang/gsoc/blob/main/2026/ideas_list.md)

## What This Repo Is

This is my onramp — everything I'm doing to build the knowledge and skills needed for the GSoC project. It includes:

- Notes and code from working through the P4 tutorials
- Environment setup documentation
- Exploration of Planter, P4Pi, and p4c-dpdk
- My qualification task work
- Research notes and reading

This is **not** the GSoC project itself. The actual project work would happen in the relevant p4lang repositories if accepted.

## Repository Structure

```
.
├── README.md
├── 01-environment-setup/           # How I set up my dev environment
│   └── vm-setup.md                 # VirtualBox + Ubuntu ARM64 on Apple Silicon
├── 02-tutorials/                   # P4 tutorial exercises
│   ├── basic-forwarding/           # Exercise 1: Basic IPv4 forwarding
│   ├── basic-tunneling/            # Exercise 2: Basic tunneling
│   ├── ecn/                        # Explicit Congestion Notification
│   ├── firewall/                   # Stateful firewall
│   └── ...
├── 03-exploration/                 # Deep dives into project-related tools
│   ├── planter/                    # Planter framework exploration
│   ├── p4pi/                       # P4Pi platform notes
│   └── p4c-dpdk/                   # DPDK backend exploration
├── 04-qualification-task/          # GSoC qualification task
│   └── planter-bmv2-workflow/      # End-to-end Planter workflow on BMv2
├── 05-notes/                       # Learning notes and concept maps
│   ├── p4-fundamentals.md          # Core P4 concepts
│   ├── networking-fundamentals.md  # Networking concepts (NAT, IPv4/6, etc.)
│   └── in-network-ml.md            # In-network ML concepts
└── resources.md                    # Papers, docs, and useful links
```

## Progress

### Phase 1: Environment & Foundations
- [x] Set up VirtualBox VM (Ubuntu 24.04 ARM64 on Apple Silicon M3 Pro)
- [x] Install P4 development tools (p4c 1.2.5.10, BMv2 1.15.0, Mininet)
- [x] Complete Basic Forwarding tutorial
- [x] Complete Basic Tunneling tutorial
- [ ] Complete additional tutorials (ECN, Firewall, Source Routing)

### Phase 2: Project Exploration
- [ ] Read Planter paper (ACM SIGCOMM CCR 2024)
- [ ] Study Planter user manual and codebase
- [ ] Explore P4Pi platform and documentation
- [ ] Understand p4c-dpdk backend

### Phase 3: Qualification Task
- [ ] Complete end-to-end Planter workflow on BMv2
- [ ] Submit PR to a p4lang repository
- [ ] Document findings and results

### Phase 4: Application
- [x] Draft GSoC proposal
- [x] Email mentor (Peng Qian) with draft attached
- [x] Join P4 Zulip #gsoc channel
- [ ] Submit final application through GSoC website

## Dev Environment

| Component | Details |
|---|---|
| Host | macOS, Apple Silicon M3 Pro, 18GB RAM |
| VM | VirtualBox 7.1+, Ubuntu Server 24.04 LTS ARM64 |
| VM Specs | 6GB RAM, 4 CPUs, 60GB disk |
| p4c | v1.2.5.10 |
| simple_switch (BMv2) | v1.15.0 |

## Key Resources

- [P4 Tutorials](https://github.com/p4lang/tutorials)
- [Planter](https://github.com/In-Network-Machine-Learning/Planter)
- [P4Pi](https://github.com/p4lang/p4pi)
- [p4c-dpdk Backend](https://github.com/p4lang/p4c/tree/main/backends/dpdk)
- [Planter Paper (SIGCOMM CCR 2024)](https://dl.acm.org/doi/10.1145/3687230.3687232)
- [P4 Zulip](https://p4lang.zulipchat.com/)
- [GSoC Application Instructions](https://github.com/p4lang/gsoc/blob/main/2026/application_instructions.md)
