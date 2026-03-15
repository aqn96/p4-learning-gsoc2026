# Planter Framework — Learning Notes

Notes from running the Planter qualification task and reading the paper. For general P4 concepts, ASICs, and the software switch insight, see `p4-fundamentals.md`.

## Running the Qualification Task

I ran a full end-to-end Planter workflow: Decision Tree (Type 4, legacy/stable) on the Iris dataset, targeting BMv2 with v1model architecture. The interactive setup asks for model type, variation, dataset, feature count, tree depth, and leaf node limit. Each choice matters — numbered type variations go with `standard_classification`, and the `performance` use case is only for EB/LB/DM variations.

What surprised me was how hands-off the P4 side is. I never touched the generated P4 code. Planter handled everything from training to table entry generation to P4 code output. The generated program compiled and ran on BMv2 without modification. That's the whole point of the framework — the user stays in Python-land.

## Three-Matrix Validation

Planter verifies correctness at each stage of the pipeline:

1. **Matrix 1 — Python baseline:** Train the model in scikit-learn, record accuracy metrics on the test set. This is your ground truth.
2. **Matrix 2 — Table logic verification:** Convert the model to match-action entries, run the same test data through the table logic in Python. Results should match Matrix 1 exactly.
3. **Matrix 3 — P4 deployment verification:** Compile the generated P4 code, deploy on BMv2, send test packets through Mininet, compare output against Matrix 1 and 2.

All three matched at 95.56% accuracy in my run. The fact that they match across all three stages confirms nothing is lost in conversion — the table entries faithfully represent the trained model, and the P4 program faithfully executes the table lookups.

This methodology is how I'd validate the new algorithms I'm proposing (AdaBoost and Linear Regression). Same three matrices, adapted for the model type — regression metrics like MSE and R² instead of classification metrics for Linear Regression.

## Mapping Methodologies — What I Learned Hands-On

`p4-fundamentals.md` covers what DM, EB, and LB are at a high level. Here's what I picked up from actually seeing DM in action and studying the paper more closely:

**Direct-Mapping (DM)** — In my qualification task, each tree split became LPM table entries. Planter took the 4 Iris features and created a separate feature table for each one. The number of table entries was small because LPM encoding compresses ranges efficiently:
- Feature 0: 80 input values → 1 LPM entry
- Feature 1: 45 input values → 2 LPM entries
- Feature 2: 70 input values → 5 LPM entries
- Feature 3: 26 input values → 4 LPM entries

That compression is significant. Without LPM, you'd need one exact-match entry per unique input value.

**Encode-Based (EB)** — Each feature is encoded independently into a code, then a decision table combines the codes. This is how ensemble methods work in Planter — each weak learner produces a code, and the final table aggregates them. AdaBoost fits here because each stump (single-split tree) produces a code, and the decision table applies the pre-computed stump weights to make the final classification.

**Lookup-Based (LB)** — Pre-computes mathematical operations and stores results in tables. This exists because ASICs can't multiply natively. For Linear Regression (`y = w₁x₁ + w₂x₂ + ... + b`), on hardware you'd need to pre-compute `wᵢ × xᵢ` for every possible value of `xᵢ` and store the results. On a software switch like P4Pi, the CPU can just multiply directly — which is an opportunity for higher precision with fewer table entries.

## The Paper's Suggested Extensions

Section 3 of the paper lists algorithms the authors themselves identified as candidates:

| Methodology | Current Count | Suggested Additions |
|---|---|---|
| DM | 3 models | DNN, CNN, LSTM |
| EB | 6 models | CatBoost, LightGBM, AdaBoost |
| LB | 5 models | Lasso, Linear Regression, Polynomial Regression |

I ruled out the DM candidates (DNN, CNN, LSTM) — too complex for a Pi and too hard to fit into a GSoC timeline.

**Why AdaBoost (EB):** Planter already has Random Forest (bagging — independent trees, equal voting) and XGBoost (gradient boosting — sequential trees, gradient-based error correction). AdaBoost is the third major ensemble paradigm — adaptive boosting, where each stump focuses on examples the previous stumps got wrong. It fills a real gap. Also, stumps are depth-1 trees, so stage consumption is minimal on the Pi.

**Why Linear Regression (LB):** It would be Planter's first general-purpose regression model. Everything currently in Planter does classification. Adding regression expands what researchers can do — latency prediction, throughput estimation, QoE scoring. The implementation is also relatively straightforward: a weighted sum of features plus a bias term.

## Issues I Hit During Setup

These are mostly version compatibility issues, not bugs in Planter's logic. But they're real friction for anyone trying to get started today:

- **Outdated `packages.txt`:** Pinned versions like `scipy==1.5.2` and `numpy==1.17.3` don't work with Python 3.12. Installed latest versions manually.
- **matplotlib style rename:** `plt.style.use('seaborn')` renamed to `'seaborn-v0_8'` in newer matplotlib. Fixed in `table_generator.py`.
- **pandas API change:** `.max()[0]` no longer works with string-indexed Series. Replaced with `.max().iloc[0]`.
- **System Python vs venv:** Planter runs the BMv2 test with `sudo python3`, which skips the virtualenv. Had to install scapy, numpy, and scikit-learn for system Python separately.
- **Missing protobuf module:** The `p4runtime` pip package doesn't include `p4.tmp.p4config_pb2`. Copied it manually from the venv to system Python.

These would be worth documenting or patching upstream at some point. A new user on Python 3.12 would hit every one of these.

## Notes on p4c-dpdk

For the GSoC project, Planter needs a new target adapter for the p4c-dpdk backend. Some things I've gathered from studying the docs and source:

- DPDK uses the PSA (Portable Switch Architecture) model, not v1model. The generated P4 code would need different architecture declarations.
- The p4c-dpdk compiler lives in `p4lang/p4c/backends/dpdk`.
- P4Runtime API support on DPDK may not be complete. One of the maintainers mentioned in a GitHub issue that he wasn't sure about the state of it. This is a potential blocker I flagged in my proposal's risk section.
- My plan is to develop and validate everything on BMv2 first (where I already have the full pipeline working), then compile through p4c-dpdk early and often to catch issues before they accumulate.
