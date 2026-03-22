# Exercise 2 Contribution Workflow (PR #733)

## Context

This note captures what I learned while submitting follow-up test coverage for the `basic_tunnel` exercise in `p4lang/tutorials`.

- Issue: [p4lang/tutorials#103](https://github.com/p4lang/tutorials/issues/103)
- PR: [p4lang/tutorials#733](https://github.com/p4lang/tutorials/pull/733)
- Scope: add PTF tests for exercise 2 and refine CI workflow structure

## What was added in PR #733

- `exercises/basic_tunnel/ptf/basic_tunnel.py` with seven tests:
  - `Ipv4DropOnMissTest`
  - `Ipv4ForwardTest`
  - `TunnelForwardTest`
  - `TunnelDropOnMissTest`
  - `MixedTrafficTest`
  - `TunnelUnknownProtoTest`
  - `TtlBoundaryTest`
- `exercises/basic_tunnel/runptf.sh` for compile/run/cleanup lifecycle
- `exercises/basic_tunnel/Makefile` `test` target
- `exercises/basic_tunnel/.gitignore` to keep generated artifacts out of commits
- `.github/workflows/test-exercises.yml` refactor:
  - matrix strategy to reduce duplication
  - `concurrency` group with cancel-in-progress
  - per-job timeout
- Cleanup requested by maintainer:
  - removed `make test` mention from `exercises/basic/README.md`

## Process lessons from this cycle

### 1) Keep open-source communication visible

Before pushing a follow-up PR, post an update in the related issue thread with current status and next step. This helps avoid duplicated effort and gives maintainers a clear signal on what is in progress.

### 2) Validate where confidence really comes from

`act` is helpful for quick feedback, but for this type of networking-heavy test flow the most reliable signal is still GitHub Actions on hosted runners.

### 3) Treat DCO as part of the normal workflow

PR #733 required DCO remediation during iteration. The fix path was straightforward:

```bash
git rebase HEAD~4 --signoff
git push --force-with-lease origin feat/ptf-tests-basic-tunnel
```

Going forward, use sign-off when creating commits on repos that enforce DCO:

```bash
git commit -s -m "test: add PTF tests for basic_tunnel exercise"
```

### 4) Distinguish student docs from maintainer docs

A key maintainer suggestion was to avoid overemphasizing contributor-only test commands in student-facing tutorial READMEs. This was a good reminder to keep contributor workflows documented, but in the right place.

## Why this matters for future contributions

The value is not only adding tests, but making the contribution easy to review and maintain:

- tests are explicit and behavior-focused
- CI is less duplicated and more bounded
- repo-facing docs stay aligned with audience expectations
- contribution hygiene (DCO, issue updates) is handled early
