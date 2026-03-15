# Open-Source Contributions

Tracking my contributions to p4lang repositories as part of my GSoC 2026 application.

## PR #730 — PTF Tests for Basic Forwarding Exercise

**Repo:** [p4lang/tutorials](https://github.com/p4lang/tutorials)
**PR:** [#730](https://github.com/p4lang/tutorials/pull/730)
**Issue:** [#103](https://github.com/p4lang/tutorials/issues/103) — Add unit tests with top-level Makefile (open since 2017)
**Status:** Draft, review requested from @jafingerhut and @Dscano

### What it does

Adds automated PTF-based regression tests for the basic forwarding exercise (`exercises/basic`). The tests run against the solution P4 program, so they catch if a compiler update or repo change breaks expected behavior.

### What's included

- `ptf/basic_fwd.py` — Three tests:
  - **DropTest** — verifies packets are dropped with no table entries (default action)
  - **FwdTest** — verifies single-entry IPv4 forwarding with correct MAC rewrite and TTL decrement
  - **MultiEntryTest** — verifies multiple LPM entries route to correct ports
- `runptf.sh` — compiles the solution, starts simple_switch_grpc, runs PTF tests, cleans up
- `.gitignore` — excludes build artifacts and log files
- README update with instructions for running the tests

### Approach

Reached out to the maintainers to confirm this was a good issue to tackle. Read through the existing PTF examples in [p4-guide](https://github.com/jafingerhut/p4-guide/tree/master/ptf-tests) to understand the testing patterns and boilerplate, then adapted that structure for the basic forwarding exercise. Tests run against the solution P4 program (not the starter code) so they serve as regression tests.

Two open dependency questions flagged for maintainer input:

1. The test script currently references `~/p4-guide/testlib` for `p4runtime_shell_utils.py`, which lives outside the tutorials repo.
2. The `veth` interface setup depends on an external script.

### Next steps

- Address review feedback once Andy or Davide respond
- Move out of draft status
- Add PTF tests for `basic_tunnel` exercise as a follow-up PR

## Community Engagement

- Joined [P4 Zulip #gsoc channel](https://p4lang.zulipchat.com/) and introduced myself
- Reached out to Davide Scano for guidance on contribution direction — he confirmed Issue #103 was a good fit
- In direct contact with mentor Peng Qian regarding the qualification task and proposal
