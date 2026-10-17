# Adaptive Multimodal Sepsis Early-Warning System

Research prototype for estimating early sepsis risk from ICU time-series data, with a reliability-aware decision layer and a separately evaluated chest X-ray branch. The intended research question is how model uncertainty and evidence availability might inform ALERT, MONITOR, or REVIEW outputs.

**Clinical safety:** this software is for research and education only. It is not a validated medical device, diagnostic system, or treatment recommendation. Do not use it for patient care. The Gradio demo can create a public share link by default; use only non-sensitive, authorized data in it.

## Repository contents

```text
.
├── .gitignore
├── DATA_AND_ARTIFACTS.md
├── PROJECT_AUDIT.md
├── PRE_RELEASE_REPORT.md
├── README.md
├── VALIDATION_REPORT.md
├── requirements.txt
└── notebooks
    ├── sepsis_demo_app.ipynb
    └── sepsis_v4.ipynb
```

No datasets, checkpoints, result files, or license file are currently included.

## Notebook roles and order

1. [`notebooks/sepsis_v4.ipynb`](notebooks/sepsis_v4.ipynb) loads the PhysioNet 2019 sepsis data, preprocesses and splits by patient, trains and evaluates models, and writes results/checkpoints to `results/` under `SEPSIS_WORK_DIR`. Its optional NIH X-ray training branch is off by default (`RUN_IMAGING = False`).
2. [`notebooks/sepsis_demo_app.ipynb`](notebooks/sepsis_demo_app.ipynb) loads the generated `sepsis_monitor_bundle.pt`, LightGBM text models, and optional X-ray checkpoint, then launches a Gradio interface. Run it after training and make the training output available as a Kaggle notebook output/input. It also needs the original sepsis patient files for its held-out-patient browser, and the NIH image data for X-ray examples.

## Environment and dependencies

The only environment inspected for this audit is Python 3.13.7 on Windows. Installed versions observed there were NumPy 2.3.5, pandas 2.3.3, scikit-learn 1.7.2, Matplotlib 3.10.7, joblib 1.5.2, Pillow 12.0.0, and SciPy 1.16.3. PyTorch, torchvision, LightGBM, Gradio, and ipykernel were not installed. This is not a tested notebook runtime, and compatibility of this interpreter with the full stack is **unknown**.

The training notebook has no installation cell. The demo notebook runs `%pip install -q gradio==6.29.0`; that is the only explicit package pin found. Other third-party imports are NumPy, pandas, PyTorch, torchvision, LightGBM, Matplotlib, scikit-learn, joblib, and Pillow. The `requirements.txt` lists the non-PyTorch packages; install PyTorch and torchvision separately using the official selector for the chosen OS and accelerator. No CUDA/PyTorch combination was verified. Kaggle may provide some of these packages already; actual Kaggle versions were not inspected.

```bash
python -m pip install -r requirements.txt
# Install torch and torchvision separately for your chosen CPU/CUDA runtime.
```

Use Kaggle Notebooks as the recommended starting environment because the code defaults to Kaggle input/output locations. Exact Kaggle runtime versions and GPU requirements have not been verified. Training includes multiple model fits and bootstrap/ablation experiments; execution time and memory needs are unknown. The X-ray branch describes a substantial workload in its notebook markdown, but is disabled by default.

## Data requirements and setup

The training notebook searches recursively under `SEPSIS_INPUT_ROOT` (default `/kaggle/input`) for `.psv` files. If none are found, it searches CSV files for a `SepsisLabel` column and expects patient/hour columns used by its loader. Its assertion names the Kaggle input “Prediction of Sepsis.” The notebook identifies the study data as PhysioNet 2019, but contains no exact dataset URL or Kaggle identifier. Dataset version, license, access class, and exact downloadable bundle are therefore **unknown**; obtain data only through an authorized source and confirm it matches the expected fields.

The optional X-ray path searches for `Data_Entry_2017.csv` and `.png` images below the same input root. Notebook text calls this NIH ChestX-ray14 / “NIH Chest X-rays,” but supplies no exact download URL or Kaggle identifier. Access details and dataset version are **unknown**. The image branch is separate from the PhysioNet patient cohort in the notebook.

### Download ZIP and open in Kaggle

1. On the actual GitHub repository page, use **Code → Download ZIP**, then extract the archive locally. A GitHub URL is not configured or known in this project.
2. In Kaggle, create a notebook and upload/import `notebooks/sepsis_v4.ipynb` from the extracted folder.
3. Use **Add Input** to attach an authorized copy of the PhysioNet 2019 sepsis dataset. Keep its supplied folder/file structure; the code recursively searches it. Confirm `.psv` patient files are present, or that a CSV with `SepsisLabel` and required columns is present.
4. Set Kaggle accelerator/runtime options based on resources available. No GPU is required by an explicit assertion; CPU fallback exists, but speed and memory feasibility have not been tested.
5. Run the training notebook top to bottom. The default is the full configuration (`SEPSIS_QUICK` defaults to `0`). Do not interpret QUICK mode as equivalent to the default experiment. It changes epochs, seeds, rounds, bootstrap counts, and other sampling counts.
6. Use Kaggle's save-version/output workflow to preserve the training run. The code writes `results/` under `/kaggle/working` by default. Inspect the saved output and confirm it contains `results/sepsis_monitor_bundle.pt` and every LightGBM file named in the bundle before proceeding. The output nesting/attachment behavior has not been verified here; if needed, download the generated model/result files and attach them to the demo notebook in a directory the demo can scan. The training notebook also writes `sepsis_raw_cache.pkl` directly under `SEPSIS_WORK_DIR`; this is derived patient data and must remain local.
7. Upload/import `notebooks/sepsis_demo_app.ipynb`. Attach the same artifact directory that contains the selected bundle, every listed LightGBM companion, `split_patient_ids.json`, and evaluation CSVs. Set `SEPSIS_BUNDLE_PATH` to the exact `sepsis_monitor_bundle.pt` path before running if multiple bundles are visible; without it, the demo proceeds only when exactly one candidate exists. Missing held-out patient files stop cohort setup. In Kaggle, if more than one bundle is attached, set `SEPSIS_BUNDLE_PATH` in a separate code cell before the demo selection cell; copy the exact bundle path shown in Kaggle's file browser, for example `os.environ["SEPSIS_BUNDLE_PATH"] = "<exact path to sepsis_monitor_bundle.pt>"`.
8. Run the demo top to bottom. The X-ray checkpoint and AUROC CSVs must be from the same selected artifact directory. X-ray metrics are read only from valid rows in `cxr_auroc_test.csv` (checked first) or, if it has no valid rows, `cxr_auroc_ci_test.csv`; the demo does not combine files. Missing findings display as unavailable, and a partial mean reports its valid finding count. If there are no valid metric rows, the demo reports that X-ray results are unavailable. The training notebook does not produce `final_review_burden.csv` or `final_xray_control.csv`, so those About rows remain unavailable.

## Paths, artifacts, and evaluation

Training paths are configurable before kernel startup with `SEPSIS_INPUT_ROOT` and `SEPSIS_WORK_DIR`; defaults are `/kaggle/input` and `/kaggle/working`. Outputs go to `<SEPSIS_WORK_DIR>/results` (plus the raw cache at `<SEPSIS_WORK_DIR>/sepsis_raw_cache.pkl`).
The demo supports `SEPSIS_INPUT_ROOT` and `SEPSIS_BUNDLE_PATH`. If `SEPSIS_BUNDLE_PATH` is set, it must identify the exact `sepsis_monitor_bundle.pt` file. If unset, discovery scans `SEPSIS_INPUT_ROOT` and `/kaggle/working` and continues only with exactly one candidate. Model and result files are read only from that bundle directory. See [`DATA_AND_ARTIFACTS.md`](DATA_AND_ARTIFACTS.md) for the exact handoff.

The training notebook records `SEED = 42` and sets Python, NumPy, and PyTorch seeds. Its source defaults include 30 max epochs, patience 6, batch size 64, learning rate `5e-4`, weight decay `1e-4`, hidden size 128, dropout 0.3, 30 MC passes, 100 bootstrap samples, 5 LightGBM seeds and 3,000 LightGBM rounds. The default also enables the listed v4 analyses. These are code defaults, not a record of a completed run; `SEPSIS_QUICK=1` changes multiple settings. Other fixed seeds occur in individual analyses; they are listed in the audit. Deterministic algorithms are not enabled, and hardware/library differences may change results. The test outputs are generated at run time; no verified metrics or trained artifacts are distributed here.

## Known limitations and information to confirm

### Implemented, proposed, and unverified

- **Implemented in notebook source:** time-series feature processing, LightGBM and GRU-family training/evaluation, a hybrid risk/policy path, simulated hourly replay, a Gradio interface, and an optional stand-alone NIH X-ray classifier/evaluation branch.
- **Proposed or demonstrated only:** the hybrid is labelled “proposed” in the notebook; review-triggered evidence requests and live monitor behavior are simulated on held-out data. No hospital-system integration or prospective workflow is shown.
- **Not implemented as joint fusion:** the notebook says NIH X-rays are from different patients and are not fused into the sepsis time-series model. The image branch is a separate task.
- **Not verified:** clinical effectiveness/safety, fresh-ZIP end-to-end execution, environment compatibility, data/artifact handoff on Kaggle, and metric values from this checkout.

- Neither notebook has been run end to end in this environment. Reproduction is not verified.
- Exact source URLs/identifiers, dataset release/version/access terms, package compatibility, Kaggle runtime image, and hardware are unknown.
- The demo no longer presents embedded fallback metrics as results. Summary rows show an explicit unavailable message when their source CSV/row is missing or invalid. X-ray AUROCs are CSV-only; the mean is computed over valid rows using the training notebook definition (mean of evaluable per-finding AUROCs), with the valid finding count shown.
- The training notebook's optional X-ray model uses `torchvision` pretrained DenseNet weights by default if image training is enabled; obtaining pretrained weights may require internet access. No data/model download was performed during this audit.
- A public Gradio share link exposes the demo to its link audience. Do not upload private patient data or credentials.
- The demo loads a PyTorch bundle with `weights_only=False`; only attach/load a bundle produced by a trusted training run. Do not load an artifact from an unknown source.
- Code is a research prototype, not a clinically validated system. Demo displays are not independent validation.
- Author/team, institution, publication citation, and license are not present. License and citation are **pending confirmation**; do not infer reuse rights.

Actual performance claims must come from real, documented executions with the corresponding data and configuration. See [`VALIDATION_REPORT.md`](VALIDATION_REPORT.md) for static checks and the final end-to-end procedure.
