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

## Deep Dive: DPDK Adapter Design (from source code reading)

After reading through the BMv2, T4P4S, and bmv2_DPU adapter source 
code alongside Section 8.3 of the Planter User Manual, I now have a 
much clearer picture of what the DPDK adapter actually needs to 
implement. These notes replace the speculation in the section above.

### How Planter target adapters actually work

Every target adapter is a code generator, not a runner. When Planter 
executes, `run_model.py` and `test_model.py` don't run the switch 
directly — they write bash scripts and Python test files that do the 
actual work. This is why the interface is consistent across targets 
even though the underlying systems are completely different.

The Planter User Manual (Section 8.3) specifies exactly six functions 
every adapter must implement:

**In `run_model.py`:**
- `file_names(Planter_config)` — returns work_root, model_test_root, 
  file_name, test_file_name
- `add_make_run_model(fname, config)` — generates the bash script that 
  copies the .p4 file and runs compilation
- `main(if_using_subprocess)` — entry point called by Planter.py

**In `test_model.py`:**
- `write_common_test_classification(fname, Planter_config)` — generates 
  Scapy packet test for classification models
- `write_common_test_dimension_reduction(fname, Planter_config)` — for 
  regression/dimension reduction models (Linear Regression hooks here)
- `finish_runtime(Planter_config)` — loads table entries to the target
- `generate_test_file(config_file)` — calls the appropriate test writer
- `main(...)` — entry point

### Folder structure decision

Looking at all targets:
- BMv2: `compile/` + `software/` (needs Mininet for network emulation)
- bmv2_DPU: `compile/` + `software/` (same as BMv2, just on DPU hardware)
- T4P4S: `compile/` only (IS the software environment)
- Tofino: `compile_auto/` only (hardware ASIC)
- alveo_u280: `behavioral/` + `compile/` + `hardware/`

DPDK SWX should follow T4P4S — `compile/` only. The DPDK pipeline 
runs directly on the host Linux system so there is no need for a 
separate Mininet emulation environment.

### What changes from BMv2 in run_model.py

BMv2 `add_make_run_model()` generates:
```bash
p4c-bm2-ss --p4v 16 --p4runtime-files build/<model>.p4info.txt \
    -o build/<model>.json <model>.p4
sudo simple_switch_grpc -i 0@veth0 -i 1@veth1 <model>.json
```

DPDK `add_make_run_model()` will generate:
```bash
p4c-dpdk --arch psa <model>.p4 -o <model>.spec
sudo dpdk-pipeline -l 1 -- -s <model>.spec
```

Key difference: BMv2 produces `.json`, DPDK produces `.spec`. These 
are completely different artifact formats consumed by different runtimes.

### PSA architecture already exists — no rewrite needed

Planter already has `src/architectures/psa/p4_generator.py`. Reading 
the source confirms it generates a complete PSA-compliant P4 file 
with ingress parser, ingress control, egress parser (stub), egress 
control (stub), and `PSA_Switch()` main instantiation.

The egress blocks are stubs — they just `transition accept` immediately. 
This is fine because Planter's ML inference runs entirely in ingress. 
And p4c-dpdk is ingress-only by default anyway. The two align perfectly.

This means the DPDK adapter does NOT need a new architecture module. 
It reuses PSA as-is.

### finish_runtime_dpdk() — the novel piece

Every existing Planter target uses BMv2's p4runtime gRPC for table 
loading. Looking at `finish_runtime()` in BMv2 test_model.py:
```python
Runtime["target"] = "bmv2"
Runtime["p4info"] = "build/<model>.p4.p4info.txt"
Runtime["bmv2_json"] = "build/<model>.json"
Runtime["table_entries"] = Table_entries["table_entries"]
json.dump(Runtime, open(model_test_root+'/s1-runtime.json', 'w'))
```

DPDK SWX has no p4runtime server. Table entries go through the 
DPDK pipeline CLI using `table add` commands. `finish_runtime_dpdk()` 
needs to read `Tables/Runtime.json` and translate each entry into 
the equivalent DPDK SWX CLI format. This translation layer has no 
precedent in any existing Planter target — it is the core novel 
engineering contribution of the adapter.

### test_model.py — confirmed nearly identical to BMv2

Compared test_model.py across all software switch targets:

| Target | Scapy srp1() | Table loading | Interface |
|---|---|---|---|
| BMv2 software | yes | simple_switch_CLI | veth |
| bmv2_DPU software | yes | simple_switch_CLI | standard Linux |
| T4P4S | yes | p4runtime via t4p4s.sh | eth0 |
| Tofino | NO — SDE stub | SDE | hardware |
| DPDK (new) | yes | finish_runtime_dpdk() | tap PMD |

The Planter custom header (ethertype 0x1234, IntField features, 
IntField result) is identical across all Scapy-based targets. 
DPDK's tap PMD exposes a Linux interface so `srp1()` works unchanged.

Linear Regression specifically hooks into 
`write_common_test_dimension_reduction()` which already exists and 
uses Pearson's correlation instead of classification accuracy. No 
new testing interface needed for regression.

### Known p4c-dpdk constraints (verified from README)

- Egress pipeline: ingress-only by default, `--enableEgress` flag 
  needed for non-empty egress (not needed for Planter models)
- Field sizes: must be multiples of 8 bits — Planter header uses 
  IntField (32-bit) and XByteField (8-bit), all aligned
- Counters/meters: non-standard API, but Planter classification 
  models don't use these externs
- No p4runtime server: table loading must go through DPDK SWX CLI

None of these block the core deliverables.
