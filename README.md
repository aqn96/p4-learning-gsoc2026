# P4 GSoC 2026 — Onramp & Qualification

Learning journal, tutorial exercises, contributions, and qualification task for my Google Summer of Code 2026 application with the P4 Language Consortium.

**Target Project:** [Project 3.3 — Integrating P4-based In-Network Machine Learning framework into P4Pi](https://github.com/p4lang/gsoc/blob/main/2026/ideas_list.md)

## What This Repo Is

This is my onramp — everything I'm doing to build the knowledge and skills needed for the GSoC project. It includes:

- Notes and code from working through the P4 tutorials
- Environment setup documentation
- My qualification task work
- Research notes and reading
- A log of my open-source contributions to p4lang repositories

This is not the GSoC project itself. The actual project work would happen in the relevant p4lang repositories if accepted.

## Repository Structure

```
.
├── .github/workflows/                   # GitHub Actions CI workflows
│   └── (in tutorials PR) test-exercises.yml  # PTF CI for exercise 1
├── README.md
├── 02-tutorials/                       # P4 tutorial exercises
│   ├── basic-forwarding/               # Exercise 1: Basic IPv4 forwarding
│   ├── basic-tunneling/                # Exercise 2: Basic tunneling
│   └── NOTES.md                        # Tutorial notes and food-for-thought answers
├── 04-qualification-task/              # GSoC qualification task
│   ├── qualification-task.md           # Full writeup of end-to-end Planter workflow
│   ├── DT_standard_classification_Iris.p4  # Generated P4 program
│   ├── Planter_config.json             # Run configuration
│   ├── log.json                        # Classification results log
│   └── screenshots/                    # Evidence screenshots
├── 05-contributions/                   # Open-source contributions to p4lang
│   └── README.md                       # Summary of PRs and community engagement
├── docs/                               # Environment setup docs
│   └── vm-setup.md                     # VirtualBox + Ubuntu ARM64 on Apple Silicon
└── learning/                           # Research notes and papers
    ├── planter_paper.pdf               # Planter SIGCOMM CCR 2024 paper
    ├── p4-fundamentals.md              # P4 concepts, pipeline, targets, networking fundamentals
    ├── planter-notes.md                # Planter framework, qualification task, mapping methodologies
    ├── ptf-testing-notes.md            # Notes on PTF testing patterns and PR workflow
    └── testing-cicd-notes.md           # CI/CD, Docker, and dependency management lessons
```

## Progress

### Phase 1: Environment & Foundations

- [x] Set up VirtualBox VM (Ubuntu 24.04 ARM64 on Apple Silicon M3 Pro)
- [x] Install P4 development tools (p4c 1.2.5.10, BMv2 1.15.0, Mininet)
- [x] Complete Basic Forwarding tutorial
- [x] Complete Basic Tunneling tutorial
- [ ] Complete additional tutorials (ECN, Firewall, Source Routing)

### Phase 2: Project Exploration

- [x] Read Planter paper (ACM SIGCOMM CCR 2024)
- [x] Study Planter user manual and codebase (Section 8.3 — target adapter interface)
- [x] Read BMv2 and T4P4S adapter source code (run_model.py, test_model.py)
- [x] Study p4c-dpdk backend source (backend.cpp, midend.cpp, dpdkArch.cpp)
- [x] Explore P4Pi platform and documentation
- [x] Confirmed PSA architecture module compatibility with p4c-dpdk target

### Phase 3: Qualification Task

- [x] Complete end-to-end Planter workflow on BMv2 (Decision Tree Type 4, Iris dataset, 95.56% accuracy)
- [x] Document findings and results
- [x] Submit PR to p4lang repository — [PR #730](https://github.com/p4lang/tutorials/pull/730)
- [x] Add automated CI workflow in PR #730 to run exercise 1 PTF tests on PR/push

### Phase 4: Application

- [x] Draft GSoC proposal
- [x] Email mentor (Peng Qian) with draft
- [x] Join P4 Zulip #gsoc channel
- [x] Iterated proposal with mentor feedback — added DPDK adapter design,
  finish_runtime_dpdk() translation layer, and pipeline diagram
- [x] Submit final application through GSoC website

## Open-Source Contributions

| Contribution | Repo | Status | Description |
|---|---|---|---|
| PR #730 | p4lang/tutorials | Merged | PTF-based regression tests for basic forwarding (DropTest, FwdTest, MultiEntryTest) plus GitHub Actions CI workflow |
| PR #9 | In-Network-Machine-Learning/Planter | Open | Fix plt.style.use('seaborn') → 'seaborn-v0_8' for matplotlib 3.6+ compatibility across 15 table_generator.py files |
| PR #7 review | In-Network-Machine-Learning/Planter | Contributed | Independently verified pandas .max()[0] → .max().iloc[0] fix and identified 6 additional .min()[0] instances |

See [05-contributions/README.md](05-contributions/README.md) for details.

## Dev Environment

| Component | Details |
|---|---|
| Host | macOS, Apple Silicon M3 Pro, 18GB RAM |
| VM | VirtualBox 7.1+, Ubuntu Server 24.04 LTS ARM64 |
| VM Specs | 6GB RAM, 4 CPUs, 60GB disk |
| p4c | v1.2.5.10 |
| simple_switch (BMv2) | v1.15.0 |
| Mininet | 2.3.1b4 |

## Key Resources

- [P4 Tutorials](https://github.com/p4lang/tutorials)
- [Planter](https://github.com/In-Network-Machine-Learning/Planter)
- [P4Pi](https://github.com/p4lang/p4pi)
- [p4c-dpdk Backend](https://github.com/p4lang/p4c/tree/main/backends/dpdk)
- [Planter Paper (SIGCOMM CCR 2024)](https://dl.acm.org/doi/10.1145/3687230.3687232)
- [P4 Zulip](https://p4lang.zulipchat.com/)
- [GSoC Application Instructions](https://github.com/p4lang/gsoc/blob/main/2026/application_instructions.md)
