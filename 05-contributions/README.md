# Open-Source Contributions

Tracking my contributions to p4lang repositories as part of my GSoC 2026 application.

## PR #730 — PTF Tests for Basic Forwarding Exercise

**Repo:** [p4lang/tutorials](https://github.com/p4lang/tutorials)  
**PR:** [#730](https://github.com/p4lang/tutorials/pull/730)  
**Issue:** [#103](https://github.com/p4lang/tutorials/issues/103) — Add unit tests with top-level Makefile (open since 2017)  
**Status:** Draft, CI checks passing, review requested from maintainers

### What it does

Adds automated PTF-based regression tests for the basic forwarding exercise (`exercises/basic`). The tests run against the solution P4 program, so they catch if a compiler update or repo change breaks expected behavior.

### What's included

- `ptf/basic_fwd.py` — Three tests:
  - **DropTest** — verifies packets are dropped with no table entries (default action)
  - **FwdTest** — verifies single-entry IPv4 forwarding with correct MAC rewrite and TTL decrement
  - **MultiEntryTest** — verifies multiple LPM entries route to correct ports
- `runptf.sh` — compiles the solution, creates/cleans veth pairs, starts `simple_switch_grpc`, runs PTF tests, then tears down
- `Makefile` `test` target — one command (`make test`) for local and CI execution
- `.github/workflows/test-exercises.yml` — automated CI job for exercise 1 tests on PR/push
- CI artifact upload for `exercises/basic/logs/` to simplify post-failure debugging
- `.gitignore` — excludes build artifacts and log files
- README update with instructions for running the tests

### Iteration & approach

I reached out to maintainers before finalizing the structure. The first draft worked but depended on external pieces (`~/p4-guide/testlib` and external veth setup). After feedback from maintainer, I refactored to tutorials-native dependencies (`utils/p4runtime_lib`) and made `runptf.sh` self-contained for interface lifecycle.

The CI setup then followed naturally: if local test execution is one command, CI should call the same command. That kept behavior consistent between local validation and automated checks.

### What I learned from this PR cycle

1. **"Can this run with zero local assumptions?"**  
   If the answer is no, CI will expose it quickly. Making test setup self-contained was the key unlock.

2. **"Does this dependency actually exist?"**  
   I hit a real-world CI pitfall by using a non-existent image name first. Verifying with `docker pull <image>` before wiring YAML would have saved a cycle.

3. **"Is local CI emulation useful for networking-heavy tests?"**  
   Yes, but with caveats. `act` was helpful for faster feedback, but Apple Silicon architecture and container permissions can still differ from GitHub-hosted Linux runners.

4. **"Will maintainers be able to debug failures quickly?"**  
   Uploading BMv2/P4Runtime logs as artifacts makes failed checks much more actionable.

### Next steps

- Address any remaining review feedback on test scope and licensing headers
- Add edge-case tests (LPM tie-breaker, TTL boundary, non-IPv4 behavior)
- Extend this pattern to `basic_tunnel` in a follow-up PR

## Community Engagement

- Joined [P4 Zulip #gsoc channel](https://p4lang.zulipchat.com/) and introduced myself
- Reached out to Davide Scano for guidance on contribution direction — he confirmed Issue #103 was a good fit
- In direct contact with mentor Peng Qian regarding the qualification task and proposal

## Extra Contributions

- Add on a fix with dependencies of packages in the Planter community after doing the alternative qualification task [#9](https://github.com/In-Network-Machine-Learning/Planter/pull/9)
