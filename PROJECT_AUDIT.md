# Project audit

Audit date: 2026-10-17. Inspection was static; notebook cells were not executed.

## Inventory and repository state

Working directory: project root. Initially present files:

| File | Bytes | Role |
|---|---:|---|
| `README.md` | 3,448 | Existing project description, with placeholder filenames/versions and incomplete setup information |
| `.gitignore` | 324 | Existing cache/data/temp exclusions |
| `notebooks/sepsis_v4.ipynb` | 118,977 | Training/evaluation notebook |
| `notebooks/sepsis_demo_app.ipynb` | 47,692 | Interactive Gradio demo |

At the initial audit there was no `.git` directory. For the current publication preparation, an empty repository has since been initialized on `main`; it has no commits, configured remote, or staged files. No data, model, result, or configuration files were present. There is no `AGENTS.md` in the project. The existing local backup now contains the initial README and `.gitignore` plus pre-change copies of both notebooks; no dataset or patient data was copied.

## Notebook validation and roles

Both files parse as Jupyter Notebook format 4.5 JSON. `sepsis_v4.ipynb` has 47 cells (23 code, 24 Markdown); `sepsis_demo_app.ipynb` has 10 cells (9 code, 1 Markdown). Neither contains saved code-cell outputs, so there are no displayed execution results to preserve.

`sepsis_v4.ipynb` is the training/evaluation notebook. Its sequence is setup; load/cached data; audit and patient split/preprocessing; evaluation helpers; LightGBM and neural model training; hybrid and policy evaluation; stress/external validation and ablations; monitor replay; artifact/report saving; optional separate NIH X-ray training and Grad-CAM. `RUN_IMAGING = False` by default. Several experiment flags, including cross-hospital validation, seed runs, trigger curve, selective diagnostics, conformal analysis, and irrelevant-evidence control, are set true in setup.

`sepsis_demo_app.ipynb` is the downstream demo. It installs `gradio==6.29.0`, locates a training bundle, reconstructs model code and loads the bundle/LightGBM models, scans sepsis patient files for held-out examples, optionally loads the X-ray model/results, then launches a Gradio UI. Its default share setting is true and its launch message expects internet access.

## Inputs and data flow

### Training notebook

- Root: `SEPSIS_INPUT_ROOT`, default `/kaggle/input`; recursive scan skips subdirectories whose names start with `images` in the general file finder.
- Primary accepted layout: one or more `.psv` patient files, one patient per file; basename is treated as patient ID. Hospital is inferred from whether the ID starts with `p1`.
- Fallback layout: the first discovered `.csv` with `SepsisLabel` in its header. The loader then expects `Patient_ID`, `SepsisLabel`, and feature columns used by the code; it uses `Hour`, otherwise `ICULOS`, to order patient rows. The assertion names “Prediction of Sepsis.”
- The notebook describes PhysioNet 2019. No exact URL, Kaggle dataset identifier, release, access level, license, or file manifest is embedded. Source/access are UNKNOWN beyond the in-notebook name.
- Optional X-ray data: recursive search for `Data_Entry_2017.csv` and PNGs, described as NIH ChestX-ray14 / NIH Chest X-rays. Exact URL/identifier/release/access are UNKNOWN.
- External hospital split is derived from inferred patient/hospital IDs; no separate external dataset path is defined.
- Preprocessing caches normalized/merged raw patient table as `<SEPSIS_WORK_DIR>/sepsis_raw_cache.pkl`; split IDs are written to `results/split_patient_ids.json`. Cache validity is not tied to data hashes or source version.

### Demo notebook

- Sepsis inputs: scans `SEPSIS_INPUT_ROOT` for `.psv` files and requires `split_patient_ids.json` beside the selected bundle. It uses only the listed test IDs and stops if any listed patient file is missing; it has no fallback cohort.
- Bundle selection: `SEPSIS_BUNDLE_PATH`, when set, must point to an existing `sepsis_monitor_bundle.pt` and is used exactly. Otherwise the demo requires exactly one discovered candidate under `SEPSIS_INPUT_ROOT` or `/kaggle/working`; zero or multiple candidates stop with instructions. Bundle structure, every listed LightGBM companion, and the colocated split file are checked before inference.
- Optional image inputs: `Data_Entry_2017.csv` and PNG files; optional checkpoint/result files described below.
- The demo uses a user-uploaded patient file for replay and permits chest X-ray uploads. Its default public share link is a privacy consideration; do not pass private patient data.

## Paths and environment assumptions

Training reads `SEPSIS_INPUT_ROOT` / `/kaggle/input`, writes to `SEPSIS_WORK_DIR` / `/kaggle/working`, and writes result plots under `results/figures`. `SEPSIS_QUICK` defaults to `0`; it changes training/experiment settings when set to `1`. `SEPSIS_IMG_PRETRAINED` defaults to `1`. Demo recognizes `SEPSIS_INPUT_ROOT`, `SEPSIS_BUNDLE_PATH`, `DEMO_SHARE` (default `1`), `DEMO_BLOCK` (default `0`), and `DEMO_PORT` (default `7860`). Automatic bundle discovery still scans `/kaggle/working`; explicit path selection is available for other layouts.

Paths found include `/kaggle/input`, `/kaggle/working`; no `/kaggle/temp`, `/content`, hard-coded Windows path, or other absolute data path was found in notebook source. The Kaggle paths are environment-specific defaults. A local run would need to set `SEPSIS_INPUT_ROOT` and `SEPSIS_WORK_DIR` before kernel startup; demo expects the training artifacts to be copied/attached where its search can find them.

## Artifacts written by training

All `OUT_DIR` results are under `<SEPSIS_WORK_DIR>/results`; figures go to its `figures/` subdirectory. The actual code writes:

- Root cache: `sepsis_raw_cache.pkl` (derived patient-level table; sensitive and should remain local).
- Dataset/splits: `results/dataset_summary.json`, `results/split_patient_ids.json`.
- LightGBM: `results/lightgbm_seed{i}.txt` (`i` ranges over configured ensemble seeds), `results/lightgbm_feature_importance.csv`.
- Neural state/history: `results/{key}{tag}.pt` and `results/history_{key}{tag}.csv`, where key/tag depend on enabled training branches and seed/hospital runs.
- Evaluation tables/config: `main_results_test.csv`, `operating_points_test.csv`, `policy_results_test.csv`, `selective_prediction.json`, `stress_tests_test.csv`, `evidence_seeking_experiment.csv`, `external_validation_A_to_B.csv`, `v4_shortcut_ablation.csv`, `v4_seed_runs.csv`, `v4_paired_tests.csv`, `v4_trigger_efficiency.csv`, `v4_selective_diagnostics.csv`, `v4_conformal_review.csv`, `v4_irrelevant_evidence.csv`, `run_config.json`, `SLIDE_SUMMARY.md`.
- Demo replay: `demo_live_log.csv`.
- Optional X-ray: `cxr_auroc_test.csv`, `cxr_densenet121.pt`, `cxr_auroc_ci_test.csv`; plot `figures/08_gradcam.png`.
- Figures: `01_missingness.png`, `02_training_curves.png`, `03_roc_pr_calibration.png`, `04_selective_prediction.png`, `05_stress_and_evidence_seeking.png`, `06_live_dashboard.png`, `07_shap_explanation.png`, `08_gradcam.png` (optional X-ray evaluation), `09_shortcut_ablation.png`, `10_trigger_efficiency.png`, `11_conformal_review.png`, and `12_irrelevant_evidence_gate.png`. All are generated outputs, not tracked source.

No outputs are present in this checkout.

## Demo artifact handoff

Required core handoff from training:

- `results/sepsis_monitor_bundle.pt`: PyTorch serialization dictionary with model state, selected config/policy/normalization/variable data and expected LightGBM filenames.
- `results/lightgbm_seed{i}.txt` for every name in bundle key `lightgbm_files` (saved LightGBM Booster text format).
- `results/split_patient_ids.json` for the intended held-out test IDs, plus source patient `.psv` files for the demo’s patient selector.
- `results/main_results_test.csv`, `v4_paired_tests.csv`, `evidence_seeking_experiment.csv`, `v4_conformal_review.csv`, and `v4_shortcut_ablation.csv` feed selected About rows. Missing or invalid rows are shown as unavailable. The demo does not use `operating_points_test.csv` for About metrics.

Optional X-ray handoff: `cxr_densenet121.pt` is a PyTorch `state_dict` required for X-ray predictions. The demo reads metrics from `cxr_auroc_test.csv` beside the selected bundle; only if that file has no valid recognized rows does it check `cxr_auroc_ci_test.csv`. It never combines files and only loads an X-ray checkpoint beside the same selected bundle. It displays finite AUROCs in [0,1] for recognized findings; missing findings are marked "AUROC unavailable". A partial mean averages valid findings, matching training cell 44's mean across classes with both positive and negative test labels, and displays the count. Without valid rows it shows "Results unavailable — run X-ray evaluation first." Default training skips image training. `final_review_burden.csv` and `final_xray_control.csv` have no identified producer in the training notebook and their display rows remain unavailable.

The selected bundle directory is the single source for LightGBM companions, split IDs, summary result CSVs, optional X-ray checkpoint, and optional X-ray metric CSVs. X-ray metadata/images are found under the configured input root. No marker file ranks bundles.

## Dependencies and reproducibility controls

Third-party imports across both notebooks: NumPy, pandas, PyTorch, torchvision, LightGBM, Matplotlib, scikit-learn, joblib (conditionally imported), Pillow (image paths), and Gradio. The demo explicitly installs Gradio 6.29.0. Standard-library imports include `os`, `sys`, `json`, `time`, `math`, `glob`, `random`, and `warnings`. No package versions beyond the Gradio install command are pinned in source. Environment versions inspected are recorded in README; PyTorch, torchvision, LightGBM, Gradio, and ipykernel were absent.

Verified primary seed is 42. Explicit additional seeds include NumPy bootstrap RNG 0 in evaluation helpers, `123` default stress mask, trigger RNG 7, selective diagnostic RNG 3, irrelevant evidence RNG 11, NIH group split seed 42 and bootstrap RNG 0, and demo sample RNG 42 / sample selection 0. The demo also hardcodes `MC=30` from bundle config fallback; the actual training bundle’s config is used when present. Deterministic execution is not enforced.

No Python version pin, GPU requirement, runtime image, dataset version, or verified environment is given in the notebooks. PyTorch device selection uses CUDA if available and otherwise CPU. Optional pretrained DenseNet weights may require a download when imaging is enabled; no download was performed.

## Security review

- A static scan of the four initial project files found no matches for private-key blocks, common credential assignments, AWS key IDs, or bearer-token strings. Values were not printed. This is a pattern scan, not a full secret scanner.
- No `.env`, dataset, or generated artifact was found. The initialized `main` branch has no commits or Git history, so there was no history to scan. The current release candidates are untracked; the attempted stage was blocked because Git could not create `.git/index.lock`.
- Both notebooks had zero saved code outputs. Source contains no discovered credential pattern. No patient data was present.
- `.gitignore` is extended to exclude common local secrets, caches, raw/private data, generated outputs, and the recoverable backup folder. Generated models/results are not blindly ignored by individual extension; the `results/` directory is ignored for the generated output tree.
- Demo uses a public Gradio share link by default. Uploaded data must not include private or restricted patient information.
- Demo loads `sepsis_monitor_bundle.pt` via `torch.load(..., weights_only=False)`. This uses Python pickle deserialization; only load a bundle generated by a trusted run/source. The loading format remains unchanged.
- No files are staged or committed, and nothing has been pushed or uploaded. GitHub connectivity was unavailable during remote inspection.

## Unresolved issues

1. Exact dataset downloads/identifiers, release, access terms, and expected complete directory structures are not stated in the code.
2. The demo previously contained fallback metric constants. The current source initializes summary rows as unavailable and replaces them only from valid source rows; it contains no fixed X-ray AUROC table or `CXR_MEAN = 0.828` fallback.
3. Kaggle output/attachment nesting remains unverified. The demo now stops on multiple candidates unless `SEPSIS_BUNDLE_PATH` selects one, and checks listed LightGBM companions.
4. Demo notebook’s title/markdown references “FINAL” notebook/output, while the actual training notebook here is `sepsis_v4.ipynb`.
5. Full dependency compatibility and exact reproducibility are unverified.
6. Author, institution, citation and license are unknown.

The demo notebook changes only artifact selection/validation, result availability/display, and explanatory text. The training notebook and its methodology, labels, splits, model logic, thresholds, and training parameters were not changed.
