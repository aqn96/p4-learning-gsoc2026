# Testing (CI/CD), Docker, and Dependency Management — Learning Notes

These are my notes from turning local PTF tests into a repeatable CI workflow during PR #730 and PR #733 in `p4lang/tutorials`.

## GitHub Actions (CI) — What clicked for me

At first, CI felt like "just write YAML." In practice, it's about turning assumptions into explicit steps.

### Mental model

- `on:` defines when tests run (`pull_request`, `push`)
- `jobs` defines where they run (clean Linux runner each time)
- `steps` define how to recreate the environment and run tests

The key insight: a CI pass means "this can be reproduced from scratch," not just "it worked once on my VM."

### CI vs CI/CD wording for this workflow

For tutorials PRs, the strict term is **CI**:

- compile P4
- deploy to BMv2 software switch inside test environment
- run automated PTF regression tests

There is deployment happening, but it is deployment to a test target (BMv2),
not production hardware. In conversation and resumes, "CI/CD" is commonly used
as shorthand, but if I need precise language:

- **CI (precise):** automatic build + deploy-to-test-target + verify on PR
- **CD (production sense):** automatic promotion to real hardware environments
  (e.g., ASIC targets), which this tutorial pipeline does not do

### What I now check before opening a PR

1. Is there a single command (`make test`) that represents truth for both local and CI?
2. Does CI call that exact command instead of a parallel custom path?
3. If CI fails, will logs/artifacts make root cause obvious?

If I cannot answer yes to all three, the setup is not mature yet.

## Docker in CI — What matters in networking tests

For P4 test flows (veth, BMv2, PTF), Docker is not just packaging; it's execution context.

### Lessons learned

- Image names must be verified, not guessed.  
  I now validate with `docker pull <image>` before committing workflow changes.
- Some networking tests need elevated container permissions.  
  If interfaces are created in test scripts, container privilege settings must be explicit.
- "Works with `act` locally" is useful but not final.  
  On Apple Silicon, architecture and emulation can differ from GitHub-hosted Linux behavior.

### Local feedback loop I like

1. Run `make test` in the same environment used for development.
2. Run `act pull_request` for fast CI syntax/environment sanity.
3. Treat GitHub Actions runner results as the final gate.

## CI structure refinements from exercise 2

In PR #733, I refactored workflow structure to improve maintainability:

- matrix strategy instead of duplicating similar jobs
- `concurrency` with `cancel-in-progress` to avoid stale CI runs
- explicit job `timeout-minutes` to prevent hung networking jobs

These were small YAML changes, but they reduced repetition and made CI
behavior more predictable during iterative pushes.

## Compliance lesson: DCO should be first-pass clean

Even when tests pass, contributor workflows can still block merge.
I hit this with DCO/sign-off requirements while iterating on PR #733.

My new default on repos that enforce DCO:

- always commit with `git commit -s`
- if needed, remediate quickly with `git rebase --signoff`

## Package Dependencies — How I think about them now

Dependency issues caused more friction than test logic itself.

### Dependency principles I want to keep

- Prefer repository-native libraries over external ad-hoc dependencies.
- Remove hidden local assumptions (hardcoded paths like `~/...`).
- Make setup self-contained in scripts so contributors don't need tribal knowledge.
- Keep dependency installation or environment selection explicit in CI.

### Practical checks

- "Could a new contributor run this with only repo instructions?"
- "Could CI run this on a clean machine with no manual intervention?"
- "Did I document why each dependency is needed?"

If one answer is no, I should expect brittle tests.

## What changed in my engineering mindset

I used to think of testing as "validate behavior."  
Now I think of CI as "validate behavior + validate reproducibility + validate maintainability."

For open source, that distinction matters. Passing tests are good; passing tests that others can trust and debug are better.
