# Open-Source Contributions

Tracking my contributions to p4lang and related repositories as part 
of my GSoC 2026 application.

---

## PR #730 — PTF Tests for Basic Forwarding Exercise

**Repo:** [p4lang/tutorials](https://github.com/p4lang/tutorials)
**PR:** [#730](https://github.com/p4lang/tutorials/pull/730)
**Issue:** [#103](https://github.com/p4lang/tutorials/issues/103) — 
Add unit tests with top-level Makefile (open since 2017)
**Status:** Merged

### What it does
Adds automated PTF-based regression tests for the basic forwarding 
exercise (`exercises/basic`). The tests run against the solution P4 
program, so they catch if a compiler update or repo change breaks 
expected behavior.

### What's included
- `ptf/basic_fwd.py` — Three tests:
  - **DropTest** — verifies packets are dropped with no table entries
  - **FwdTest** — verifies single-entry IPv4 forwarding with correct 
    MAC rewrite and TTL decrement
  - **MultiEntryTest** — verifies multiple LPM entries route to 
    correct ports
- `runptf.sh` — compiles the solution, creates/cleans veth pairs, 
  starts `simple_switch_grpc`, runs PTF tests, then tears down
- `Makefile` `test` target — one command (`make test`) for local 
  and CI execution
- `.github/workflows/test-exercises.yml` — automated CI job for 
  exercise 1 tests on PR/push
- CI artifact upload for `exercises/basic/logs/` to simplify 
  post-failure debugging
- `.gitignore` — excludes build artifacts and log files
- README update with instructions for running the tests

### Iteration & approach
I reached out to maintainers before finalizing the structure. The 
first draft worked but depended on external pieces (`~/p4-guide/testlib` 
and external veth setup). After feedback from a maintainer, I refactored 
to tutorials-native dependencies (`utils/p4runtime_lib`) and made 
`runptf.sh` self-contained for interface lifecycle.

The CI setup then followed naturally: if local test execution is one 
command, CI should call the same command. That kept behavior consistent 
between local validation and automated checks.

### What I learned from this PR cycle
1. **"Can this run with zero local assumptions?"**
   If the answer is no, CI will expose it quickly. Making test setup 
   self-contained was the key unlock.
2. **"Does this dependency actually exist?"**
   I hit a real-world CI pitfall by using a non-existent image name 
   first. Verifying with `docker pull <image>` before wiring YAML 
   would have saved a cycle.
3. **"Is local CI emulation useful for networking-heavy tests?"**
   Yes, but with caveats. `act` was helpful for faster feedback, but 
   Apple Silicon architecture and container permissions can still 
   differ from GitHub-hosted Linux runners.
4. **"Will maintainers be able to debug failures quickly?"**
   Uploading BMv2/P4Runtime logs as artifacts makes failed checks 
   much more actionable.

### Next steps
- Address any remaining review feedback on test scope and 
  licensing headers
- Add edge-case tests (LPM tie-breaker, TTL boundary, 
  non-IPv4 behavior)
- Extend this pattern to `basic_tunnel` in a follow-up PR

---

## PR #733 — PTF Tests for Basic Tunneling Exercise

**Repo:** [p4lang/tutorials](https://github.com/p4lang/tutorials)
**PR:** [#733](https://github.com/p4lang/tutorials/pull/733)
**Issue:** [#103](https://github.com/p4lang/tutorials/issues/103)
**Status:** Open (ready for review, checks passing)

### What it does
Adds automated PTF-based regression tests for the `basic_tunnel`
exercise (`exercises/basic_tunnel`) against `solution/basic_tunnel.p4`.
This extends the test pattern from PR #730 to tunnel-specific behavior.

### What's included
- `ptf/basic_tunnel.py` — Seven tests:
  - **Ipv4DropOnMissTest** — verifies plain IPv4 packets drop with
    no IPv4 table entries
  - **Ipv4ForwardTest** — verifies IPv4 forwarding, MAC rewrite, and
    TTL decrement
  - **TunnelForwardTest** — verifies tunneled forwarding by `dst_id`
  - **TunnelDropOnMissTest** — verifies tunneled packets drop when
    no tunnel entry exists
  - **MixedTrafficTest** — verifies IPv4 and tunnel traffic paths stay
    independent
  - **TunnelUnknownProtoTest** — verifies forwarding depends on
    `dst_id` even with unexpected `proto_id`
  - **TtlBoundaryTest** — verifies TTL boundary decrement behavior
- `runptf.sh` — compiles solution, starts `simple_switch_grpc`,
  runs tests, and cleans up interfaces/processes
- `Makefile` — adds `make test` target
- `.gitignore` — excludes generated artifacts (`build/`, logs, pcap)
- `.github/workflows/test-exercises.yml` — refactored to matrix
  strategy, concurrency, and timeout controls
- Cleanup update: removed `make test` mention from `exercises/basic/README.md`
  per maintainer guidance (contributor test flow vs student exercise docs)

### Workflow lessons from this PR cycle
1. Keep issue threads updated when moving from exercise 1 to exercise 2,
   so maintainers can track progress and avoid duplicate work.
2. Use `act` for quick feedback, but treat GitHub Actions as the final
   source of truth for networking-heavy jobs.
3. Handle DCO/sign-off expectations early (`git commit -s`), especially
   when rebasing iterative test commits.
4. Prefer tutorial-native dependencies and self-contained scripts to reduce
   setup friction for reviewers and future contributors.

---

## PR #9 — Fix matplotlib deprecation in Planter

**Repo:** 
[In-Network-Machine-Learning/Planter](https://github.com/In-Network-Machine-Learning/Planter)
**PR:** [#9](https://github.com/In-Network-Machine-Learning/Planter/pull/9)
**Issue:** [#8](https://github.com/In-Network-Machine-Learning/Planter/issues/8)
**Status:** Open

### What it does
Fixes a breaking deprecation across 15 `table_generator.py` files 
where `plt.style.use('seaborn')` was removed in matplotlib 3.6+. 
Updated to `plt.style.use('seaborn-v0_8')` across all affected files. 
Discovered this while running the qualification task end-to-end on 
BMv2 — the error breaks visualization output for every model that 
generates plots.

---

## PR #7 Review — Fix pandas deprecation in Planter

**Repo:** 
[In-Network-Machine-Learning/Planter](https://github.com/In-Network-Machine-Learning/Planter)
**PR:** [#7](https://github.com/In-Network-Machine-Learning/Planter/pull/7)
**Status:** Contributed review

### What I did
Independently verified the proposed `.max()[0]` → `.max().iloc[0]` 
fix and identified 6 additional `.min()[0]` instances with the same 
breakage that the original PR missed. Left detailed review comments 
noting the additional locations so the fix would be complete.

---

## Community Engagement

- Joined [P4 Zulip #gsoc channel](https://p4lang.zulipchat.com/) 
  and introduced myself
- Reached out to Davide Scano for guidance on contribution direction 
  — he confirmed Issue #103 was a good fit
- In direct contact with mentor Peng Qian throughout the 
  qualification task and proposal process
