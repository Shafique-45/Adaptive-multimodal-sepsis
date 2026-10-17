# Pre-publication review

Review date: 2026-10-17. This is a final documentation and static review after the approved demo scientific-integrity changes. Notebook cells were not executed, artifacts were not loaded by this review, and end-to-end success is not claimed.

## Intended release files

- `.gitignore`
- `README.md`
- `requirements.txt`
- `PROJECT_AUDIT.md`
- `DATA_AND_ARTIFACTS.md`
- `VALIDATION_REPORT.md`
- `PRE_RELEASE_REPORT.md`
- `notebooks/sepsis_v4.ipynb` - training/evaluation
- `notebooks/sepsis_demo_app.ipynb` - Gradio demo

Both notebooks remain under their original filenames. The training notebook was not modified. The demo changed only artifact selection and validation, metric availability/display, and explanatory text.

## Files that must stay local

- `adaptive-multimodal-sepsis-backup-20261017/` - preserve this existing backup. It contains the pre-change demo and training notebooks plus the prior README and `.gitignore`; do not delete or include it in a release ZIP.
- Authorized datasets, patient files, derived caches, credentials, private environment files, generated checkpoints, model outputs, logs, and temporary files.
- Current backup files are ignored by the project ignore rules. The explicit release list above excludes the backup. Recheck the final file list before any future staging.

No datasets, model checkpoints, generated training outputs, or credential-like files were found in the reviewed project tree. The initialized `main` branch has no commits, so no Git history scan was possible.

## Artifact dependency map

Training output root is `<SEPSIS_WORK_DIR>/results/` (default `/kaggle/working/results`). The demo uses one selected bundle directory for model and result companions.

| Required or optional artifact | Actual producer and format | Demo behavior / condition |
|---|---|---|
| `sepsis_monitor_bundle.pt` | Training cell 42; `torch.save` mapping with RG-GRU-D state, `lightgbm_files`, stack weights, policy, config, normalization and variable names. Loaded with `torch.load(..., weights_only=False)`. | Required. `SEPSIS_BUNDLE_PATH` selects this exact file when set. Otherwise exactly one discovered candidate is required. The selected bundle must have the expected keys. Load only a trusted artifact. |
| `lightgbm_seed{i}.txt` | Training cell 12; `Booster.save_model`, filenames listed in bundle key `lightgbm_files`. | Every listed file must exist beside the selected bundle. The demo reports all missing companions together before inference. |
| `split_patient_ids.json` | Training cell 8; JSON containing train/val/test ID lists. | Required beside selected bundle. Demo uses test IDs only and stops if the split file is missing/invalid or any listed patient PSV is absent. There is no all-files fallback cohort. |
| Original sepsis `.psv` files | Authorized input dataset; not generated model artifacts. | Required for held-out patient browser/replay. All IDs listed in the split must be available under `SEPSIS_INPUT_ROOT`. |
| `main_results_test.csv` | Training cell 20; CSV. | Source for hybrid/SIRS/qSOFA AUROC summary. Missing/invalid file, row, or numeric value leaves the result unavailable. |
| `v4_paired_tests.csv` | Training experiment cells; CSV. | Source for missing-lab and external-hospital comparison rows. Missing/invalid rows remain unavailable. |
| `evidence_seeking_experiment.csv` | Training cell 24; CSV. | Source for lab-outage utility summary. Missing/invalid row remains unavailable. |
| `v4_conformal_review.csv` | Training conformal evaluation cell; CSV. | Source for conformal review summary. Missing/invalid row remains unavailable. |
| `v4_shortcut_ablation.csv` | Training cell 28; CSV. | Source for shortcut-ablation AUROC. Missing/invalid row remains unavailable. |
| `final_review_burden.csv`, `final_xray_control.csv` | No producer identified in `sepsis_v4.ipynb`. | Not read by the demo after this correction; these About rows remain unavailable. Do not treat stray files with these names as verified project results. |
| `cxr_densenet121.pt` | Training cell 44; PyTorch DenseNet state dict, only if `RUN_IMAGING=True` and NIH metadata/data are available. Default `RUN_IMAGING=False`. | Optional inference checkpoint; only loaded beside the selected bundle. |
| `cxr_auroc_ci_test.csv`, `cxr_auroc_test.csv` | Training cells 46 and 44 respectively; CSVs from optional X-ray evaluation. | Metrics are read only beside the selected bundle. The standard CSV is checked first to preserve cell 44 AUROC precision; the CI CSV is checked only if the standard file has no valid recognized rows. The demo does not combine files. Only finite AUROCs in [0,1] for recognized findings are used. |
| `Data_Entry_2017.csv` and PNG images | Authorized NIH ChestX-ray14 dataset. | Searched under `SEPSIS_INPUT_ROOT` for random examples. This dataset is separate from model/result artifacts. |

Training cell 44 includes a finding AUROC only when the test labels include both positive and negative examples, then prints the unweighted mean over those evaluable findings. The demo computes its partial mean from valid CSV findings using the same unweighted definition and displays the count. Missing findings display `AUROC unavailable`; if no valid metrics exist it displays `Results unavailable — run X-ray evaluation first.` No fallback values are used.

## Security and privacy warnings

- Gradio sharing defaults to enabled (`DEMO_SHARE=1`). Do not use a public share with private/restricted patient data. A local launch can set `DEMO_SHARE=0` before launch.
- The selected PyTorch bundle is deserialized with `weights_only=False`; only attach a bundle from a trusted training run.
- The demo still scans configured Kaggle input and `/kaggle/working` when no explicit bundle path is set. Set `SEPSIS_BUNDLE_PATH` to avoid ambiguity. Automatic discovery stops unless exactly one candidate is found.
- No secret value is included in this report. The prior pattern-based scan is not comprehensive and no full history scan was performed.
- No clinical dataset or private patient information should be placed in the release ZIP.

## Documentation corrections

- The pre-change documentation described `CXR_MEAN = 0.828` as if it unconditionally overwrote the computed mean. Source inspection showed that assignment existed only in the cell 9 fallback branch when cell 8 had not run. It is now `None`, and the fallback message explicitly reports that X-ray results are unavailable.
- The default per-finding X-ray AUROCs, hard-coded sepsis result constants, and unsupported `1.8% of hours` claim were removed from demo display paths. The About values now require valid CSV-derived rows.
- `final_review_burden.csv` and `final_xray_control.csv` have no identified producer in the training notebook; their About rows remain unavailable.
- Bundle choice is now deterministic: use the exact validated `SEPSIS_BUNDLE_PATH` if supplied; otherwise stop unless exactly one candidate exists. There is no marker-file ranking.
- README and the audit/data map now describe the selected-directory contract, companion checks, held-out split requirement, and optional X-ray generation.
- Dataset locators, versions, terms, and access class remain unknown where absent from source. No identifiers or metric values were invented.

## Dependency and environment status

`requirements.txt` retains verified external imports and the explicit Gradio source pin. Other package versions remain unpinned. PyTorch and torchvision need platform-appropriate installation; no CUDA/PyTorch combination or Kaggle compatibility was tested. No package was installed. The available Windows environment was previously observed to lack PyTorch, torchvision, LightGBM, Gradio, and ipykernel; this does not establish Kaggle package state or compatibility.

## Static validation performed for this change

- Parsed the edited demo notebook as JSON and checked notebook metadata/cell structure.
- Compiled every code cell in both notebooks without executing them; no training, inference, artifact load, download, installation, or dashboard launch occurred.
- Confirmed source-level artifact producers and demo consumers for bundle, LightGBM files, split IDs, summary CSVs, and optional X-ray checkpoint/metric CSVs.
- Checked no fixed X-ray AUROC table, fixed `CXR_MEAN` value, hard-coded summary metric defaults, or `1.8% of hours` statement remains in the edited demo cells.
- Checked all local Markdown links, requirements line syntax, and key artifact references against source.
- Confirmed original notebooks remain in the pre-existing backup directory. No cleanup or deletion was performed.
- Git is initialized on `main` with no commits. Staging was attempted with explicit paths but failed because the sandbox denied creation of `.git/index.lock`; no files were staged. GitHub remote inspection via `git ls-remote` failed because the environment could not connect to `github.com` on port 443. No remote was added, commit created, or push attempted.

Static checks cannot verify runtime behavior, artifact compatibility, metric provenance outside the source notebook, or the correctness of a real training run.

## Remaining blockers and manual end-to-end steps

1. Confirm the authorized sepsis and NIH dataset source, version, access terms, and expected file layout.
2. From a fresh downloaded ZIP, import `notebooks/sepsis_v4.ipynb` into Kaggle, attach the authorized sepsis dataset, and record the Kaggle image, Python/packages, accelerator, data release, notebook revision, and environment settings.
3. Run training/evaluation only in the authorized environment. Confirm the output directory includes `sepsis_monitor_bundle.pt`, every name listed in `lightgbm_files`, `split_patient_ids.json`, and generated result CSVs. Record metrics only from these actual outputs.
4. Attach the output directory and original sepsis PSV files to the demo. If multiple bundles are visible, set `SEPSIS_BUNDLE_PATH` to the exact bundle path. Confirm the companion and split checks pass and all split-listed PSV files are present.
5. Test the demo with non-sensitive authorized data and decide whether Gradio public sharing is appropriate. Exercise missing/invalid result CSV cases to confirm unavailable states.
6. For X-ray testing, deliberately run the optional training/evaluation path with authorized NIH data, then place the generated checkpoint and metric CSVs beside the selected bundle. Verify finding values, partial-count text, and missing-metric behavior against the CSVs.
7. Only after those checks can end-to-end reproducibility be assessed. No end-to-end success is claimed here.

## Release checklist

- [x] Exact notebook names and roles verified.
- [x] Original notebooks preserved in the existing local backup.
- [x] Demo fallback result constants removed and unavailable states added.
- [x] Bundle and companion file checks added; held-out split required.
- [x] X-ray checkpoint and metric source constrained to selected bundle directory.
- [x] Documentation corrected for conditional cell 9 `CXR_MEAN`, and no-producer CSVs recorded.
- [x] Static checks run only.
- [ ] Manual fresh-ZIP Kaggle training and demo run.
- [ ] Confirm data sources, permissions, license/citation, and actual runtime compatibility.
- [ ] Review explicit release file list before any Git staging/publication.

This project has not passed an end-to-end test and is not claimed to be fully reproducible or a validated medical diagnostic system.


## 2026-10-17 publication attempt status

- Current Git branch: `main`; repository has no commits and no configured remote.
- Exact release files were verified on disk. `git status --short` reports the nine intended files as untracked; `git diff --cached --name-only` and staged diff are empty.
- `git init --initial-branch=main` succeeded. `git add --` with the nine explicit release paths failed: `fatal: Unable to create '.git/index.lock': Permission denied`. The backup ignore rule was verified with `git check-ignore`; the backup is not in the release file list.
- Read-only remote inspection of `https://github.com/Shafique-45/Adaptive-multimodal-sepsis.git` via `git ls-remote --heads` failed because this environment could not connect to `github.com:443`. The repository's remote history could not be inspected; no `origin` was configured.
- No commit, remote configuration, or push was performed. Publication remains blocked until `.git` is writable and the GitHub remote can be inspected. Do not force-push or overwrite any remote history.
