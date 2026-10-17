# Data and artifact dependency map

All paths and names below come from notebook source, not from files present in this checkout. No dataset was downloaded or inspected.

## Dataset inputs

| Notebook / stage | Dataset named in source | Discovery pattern and expected fields | Source/access status | Present here? |
|---|---|---|---|---|
| `sepsis_v4.ipynb`, training | PhysioNet 2019; Kaggle assertion also calls it “Prediction of Sepsis” | Recursively under `SEPSIS_INPUT_ROOT` (default `/kaggle/input`), first prefer `.psv` files. Each PSV should contain `HR, O2Sat, Temp, SBP, MAP, DBP, Resp, EtCO2, BaseExcess, HCO3, FiO2, pH, PaCO2, SaO2, AST, BUN, Alkalinephos, Calcium, Chloride, Creatinine, Bilirubin_direct, Glucose, Lactate, Magnesium, Phosphate, Potassium, Bilirubin_total, TroponinI, Hct, Hgb, PTT, WBC, Fibrinogen, Platelets, Age, Gender, Unit1, Unit2, HospAdmTime, ICULOS, SepsisLabel`; filename stem is used as patient ID. If no PSV exists, discover CSV whose header contains `SepsisLabel`; code also uses `Patient_ID` and `Hour`, or `ICULOS` as fallback ordering. | URL, Kaggle identifier, exact release and access terms: **UNKNOWN**. Name is mentioned, no source locator is embedded. | No |
| `sepsis_v4.ipynb`, optional X-ray | NIH ChestX-ray14 / NIH Chest X-rays | Find `Data_Entry_2017.csv` and `.png` files under the same input root. CSV fields used include `Image Index`, `Finding Labels`, and `Patient ID`. | URL, Kaggle identifier, release and access terms: **UNKNOWN**. | No |
| `sepsis_demo_app.ipynb`, patient browser | Same sepsis patient data | Requires `split_patient_ids.json` beside the selected bundle and scans `SEPSIS_INPUT_ROOT` recursively for every test-listed `.psv`; missing listed files stop setup. | Inherits unknown source/access above. | No |
| `sepsis_demo_app.ipynb`, X-ray tab | Same NIH image data | Finds `Data_Entry_2017.csv`, all matching PNGs; model checkpoint is a separate generated input. | Inherits unknown source/access above. | No |

The training code defines a hospital split from patient IDs (`p1...` versus other IDs for PSV; numeric `Patient_ID >= 100000` versus lower IDs for CSV). It does not require a separate external-hospital data path. The NIH imaging task is separate from the sepsis task in notebook text and code.

### Obtaining missing inputs

Use only a source and release for which you have authorization. Confirm the actual source, access terms, release and directory layout with the dataset maintainer before running. Preserve the provider's folder structure under Kaggle **Add Input** or set `SEPSIS_INPUT_ROOT` to a directory containing it. The notebook source does not provide a URL or immutable version, so this repository cannot give a verified download command. Do not substitute another dataset based on filename similarity alone.

## Path configuration

| Variable | Used by | Default | Purpose |
|---|---|---|---|
| `SEPSIS_INPUT_ROOT` | both notebooks | `/kaggle/input` | Root to search for datasets and, when no explicit bundle is selected, candidate bundle files |
| `SEPSIS_BUNDLE_PATH` | demo notebook | unset | Exact `sepsis_monitor_bundle.pt` file to use; without it, exactly one discovered candidate is required |
| `SEPSIS_WORK_DIR` | training notebook and its later X-ray cell | `/kaggle/working` | Training cache/output location |
| `SEPSIS_QUICK` | training notebook | `0` | Selects a reduced configuration when set to `1`; changes experiment parameters |
| `SEPSIS_IMG_PRETRAINED` | training notebook | `1` | Controls whether optional torchvision DenseNet pretrained weights are requested |
| `DEMO_SHARE` | demo notebook | `1` | Requests a public Gradio share URL when enabled |
| `DEMO_BLOCK` | demo notebook | `0` | Controls Gradio launch blocking behavior |
| `DEMO_PORT` | demo notebook | `7860` | Local Gradio port |

Training creates `results/` below `SEPSIS_WORK_DIR`; it does not accept an independent results directory. The demo uses the selected bundle directory as the source for companion models, split IDs, summary result CSVs, and X-ray checkpoint/metrics. When no `SEPSIS_BUNDLE_PATH` is set, it scans `SEPSIS_INPUT_ROOT` and `/kaggle/working` and proceeds only if exactly one bundle candidate exists.

## Generated training outputs

`<SEPSIS_WORK_DIR>/sepsis_raw_cache.pkl` is a pickle of the processed patient table. It is an intermediate derived from the clinical dataset and should remain local.

`<SEPSIS_WORK_DIR>/results/` contains the following outputs when the corresponding code path runs:

| Artifact(s) | Format / creation | Demo dependency |
|---|---|---|
| `dataset_summary.json` | JSON data audit | None identified |
| `split_patient_ids.json` | JSON patient IDs for train/validation/test | Required by demo; test cohort is not inferred from all discovered files |
| `lightgbm_seed{i}.txt` | LightGBM Booster text model; one per configured seed | Yes; filenames stored in bundle |
| `{key}{tag}.pt`, `history_{key}{tag}.csv` | PyTorch state dict and CSV training history, variable keys/tags by model/branch/seed | Demo core state is carried in bundle; standalone state files not loaded by inspected demo |
| `sepsis_monitor_bundle.pt` | PyTorch serialized mapping containing RG-GRU-D state dict, LightGBM filenames, stack weights, policy, config, normalization and variable names | **Required core model bundle** |
| `main_results_test.csv`, `v4_paired_tests.csv`, `evidence_seeking_experiment.csv`, `v4_conformal_review.csv`, `v4_shortcut_ablation.csv` | CSV evaluation tables | Optional About rows are populated only from valid source rows; missing/invalid values display as unavailable |
| `policy_results_test.csv`, `selective_prediction.json`, `stress_tests_test.csv`, `evidence_seeking_experiment.csv`, `external_validation_A_to_B.csv`, `v4_shortcut_ablation.csv`, `v4_seed_runs.csv`, `v4_paired_tests.csv`, `v4_trigger_efficiency.csv`, `v4_selective_diagnostics.csv`, `v4_conformal_review.csv`, `v4_irrelevant_evidence.csv` | CSV/JSON evaluation outputs; creation depends on branches and successful completion | Some demo summary rows consume selected CSVs; the demo does not rank bundles using `v4_trigger_efficiency.csv` |
| `run_config.json`, `SLIDE_SUMMARY.md` | JSON config and generated Markdown report | Not required to load core model |
| `demo_live_log.csv` | CSV hourly replay log | Written by training monitor cell |
| `lightgbm_feature_importance.csv` | CSV feature importance | No |
| `figures/*.png` | Matplotlib plots; includes `01_missingness.png`, `02_training_curves.png`, and plots named in notebook save calls | Not required by model loader |
| `cxr_densenet121.pt` | PyTorch DenseNet `state_dict`, only with `RUN_IMAGING=True` and data attached | Optional X-ray prediction model |
| `cxr_auroc_test.csv`, `cxr_auroc_ci_test.csv` | CSV metrics; generated in image training/evaluation branches | Optional displayed X-ray metric source |
| `figures/08_gradcam.png` | PNG visualization, generated by the X-ray result cell | No |

The plot names present in source are `01_missingness.png`, `02_training_curves.png`, `03_roc_pr_calibration.png`, `04_selective_prediction.png`, `05_stress_and_evidence_seeking.png`, `06_live_dashboard.png`, `07_shap_explanation.png`, optional `08_gradcam.png`, `09_shortcut_ablation.png`, `10_trigger_efficiency.png`, `11_conformal_review.png`, and `12_irrelevant_evidence_gate.png`.

Additional dynamically named outputs and plots can be seen in the save calls of `sepsis_v4.ipynb`. All files are created at run time; none are present in the repository directory.

## Demo handoff and ordering

Run the training notebook first. Make one output directory visible to the demo that contains `sepsis_monitor_bundle.pt`, all matching `lightgbm_seed{i}.txt` files, and the result CSVs next to them. Attach the sepsis dataset separately because the demo enumerates test `.psv` files. Attach NIH data and `cxr_densenet121.pt` only to enable X-ray predictions and example image selection.

The training notebook defaults `RUN_IMAGING=False`, so the fresh core run does not create `cxr_densenet121.pt`. The X-ray branch is independent and currently does not feed the multimodal sepsis model. To create its checkpoint, a maintainer must deliberately enable the branch and provide the actual NIH dataset. The code may request pretrained torchvision weights depending on `SEPSIS_IMG_PRETRAINED`.

Demo bundle selection uses `SEPSIS_BUNDLE_PATH` exactly when set. Otherwise it proceeds only when exactly one bundle candidate exists under the configured input root and `/kaggle/working`; zero or multiple candidates stop with instructions. It checks that each bundle-listed LightGBM filename exists beside the bundle and reports all missing files together.

## Unknowns and safe next steps

- Dataset source URLs, identifiers, versions, redistribution terms, and complete filenames: **UNKNOWN**; confirm from the authorized dataset provider.
- Whether Kaggle output preserves the expected directory nesting and lets users set `SEPSIS_BUNDLE_PATH`: not tested. Inspect the output tree and ensure same-directory dependencies reach the demo.
- No data or checkpoints are bundled. Run the training notebook with real authorized data to generate missing files; do not use fabricated or synthetic replacements.
- Demo X-ray metrics are CSV-only, restricted to recognized findings with finite AUROCs in [0,1]. A partial mean is the unweighted mean of valid class AUROCs, matching training cell 44's mean over classes with both positive and negative labels; the valid-finding count is displayed. Missing metric rows are marked "AUROC unavailable"; no valid rows produce "Results unavailable — run X-ray evaluation first." `final_review_burden.csv` and `final_xray_control.csv` have no identified producer in the training notebook, so their About rows remain unavailable.
