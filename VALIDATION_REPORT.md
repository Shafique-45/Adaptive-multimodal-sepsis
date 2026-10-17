# Validation report

Date: 2026-10-17. Validation is static and non-expensive. No notebook cell, training job, dataset download, model load, dashboard, package installation, or GPU workload was run.

## Checks and current status

| Check | Status | Evidence / limitation |
|---|---|---|
| Repository inventory and file sizes | PASS | Current source tree contains two notebooks, README, `.gitignore`, `requirements.txt`, three audit/review reports, and the pre-existing backup directory. No model/data artifact was present. |
| Git state/remotes | BLOCKED | Empty repository initialized on `main`; no commits or remote. `git ls-remote` could not connect to GitHub. Staging failed because creating `.git/index.lock` was denied by filesystem permissions; adding the approved `origin` URL was also denied when Git tried to lock `.git/config`. |
| Notebook JSON parse, format and cell counts | PASS | Both parse as nbformat 4.5; training 47 cells (23 code, 24 Markdown), demo 10 cells (9 code, 1 Markdown). |
| Saved notebook output review | PASS | Neither notebook has saved code-cell outputs. Output bodies were not displayed or executed. |
| Credential pattern scan | PASS WITH LIMIT | Four initial project files scanned for private-key blocks, common credential assignments, AWS key IDs, and bearer strings. No matches. This is not a comprehensive secret scanner or Git-history scan. |
| Model artifact loading review | WARNING | Demo continues to use `torch.load(..., weights_only=False)` for trusted PyTorch bundles. Bundle structure and companion names are checked in source; no artifact was loaded. |
| Package availability | PASS WITH LIMIT | Environment metadata inspected without installing: Python 3.13.7; missing PyTorch, torchvision, LightGBM, Gradio, ipykernel. Other observed versions are in README. Does not establish compatibility. |
| Static Python syntax | PASS | Both notebooks were compiled cell by cell after stripping IPython `%`/`!` lines; no cell was executed. This does not establish runtime behavior. |
| Markdown/local references and requirements syntax | PASS WITH LIMIT | Local Markdown links resolve; simple requirement-line syntax parses. Markdown rendering and package resolution were not tested. |
| Artifact output/input name agreement | PASS WITH LIMIT | Static source confirms the training bundle lists LightGBM filenames; demo checks every listed companion beside the selected bundle and requires the split file. No artifact was loaded. |
| End-to-end training and evaluation | NOT RUN | Requires authorized data and ML dependencies; prohibited for this preparation without explicit approval. |
| Demo launch and Kaggle workflow | NOT RUN | No notebook cell or dashboard was run. Kaggle output attachment and environment-variable setup remain unverified. |
| Full reproducibility | NOT VERIFIED | No exact data release, complete dependency lock, compatible runtime, run outputs, or hardware record is available. |

## Static validation commands

The following command can be rerun from the repository root to parse notebook JSON and compile each code cell without executing it. The `%pip` IPython magic is removed solely for compilation:

```powershell
@'
import json, pathlib
for p in pathlib.Path("notebooks").glob("*.ipynb"):
    nb=json.loads(p.read_text(encoding="utf-8"))
    for i,c in enumerate(nb["cells"]):
        if c["cell_type"] != "code": continue
        src="".join(c.get("source", []))
        src="\n".join(line for line in src.splitlines() if not line.lstrip().startswith(("%", "!")))
        compile(src, f"{p}:cell-{i}", "exec")
    print(f"PASS {p}")
'@ | python -
```

Verify dependency-file syntax as line-oriented pip requirements (this does not install packages):

```powershell
Get-Content requirements.txt
```

Inspect README references and actual tree manually before staging. Confirm that every Markdown local link points to an existing file. No tests or model execution are prescribed by this static check.

## Fresh ZIP end-to-end test still required

1. Download the final GitHub ZIP after publication approval and extract it into a clean directory.
2. In Kaggle, import `notebooks/sepsis_v4.ipynb` and attach the authorized dataset release after confirming its identity and required structure.
3. Record Kaggle runtime image, Python and installed package versions, accelerator, dataset release/identifier, notebook revision, and environment variables. Start with the full default settings only when compute is approved; `SEPSIS_QUICK=1` changes the experiment and is not equivalent.
4. Run all training cells. Confirm outputs and artifact names listed in `DATA_AND_ARTIFACTS.md`, especially the bundle and every LightGBM text file. Record actual metrics only from these generated outputs.
5. Attach the training output and original patient data to `sepsis_demo_app.ipynb`. If multiple candidates are visible, set `SEPSIS_BUNDLE_PATH` in a code cell before bundle-selection cell 2, using the exact path shown by Kaggle. Confirm all split-listed PSV files and every listed LightGBM file are present.
6. For privacy, use only data authorized for the demo. Confirm whether Gradio sharing is enabled; its current default requests a public share URL. Test the demo locally with `DEMO_SHARE=0` if a public link is not appropriate.
7. Compare displayed summaries against the source CSVs. Missing or invalid rows should display unavailable. For X-ray results, confirm the checkpoint and metric CSVs are in the same explicitly selected artifact directory and verify the displayed valid-finding count.


## Final publication-preparation checks

- Parsed both notebook JSON files, confirmed nbformat 4, original cell counts, and no saved cell outputs.
- Compiled every code cell independently after stripping IPython `%` and `!` lines; no cell was executed.
- Checked source-level bundle selection, bundle keys, LightGBM companion preflight, required split file and cohort behavior, CSV-only X-ray metrics, unavailable states, and removal of fixed fallback metrics/unsupported prevalence claim.
- Cross-checked X-ray AUROC producers and averaging definition against training cell 44; confirmed no training producer for `final_review_burden.csv` or `final_xray_control.csv`.
- Checked local Markdown links and line-oriented requirements syntax. No package installation or dependency resolution was performed.
- Compared notebooks against the existing backup: training remains byte-identical; original pre-change demo backup remains intact.

These static checks do not establish compatibility, load or validate model artifacts, verify metric provenance beyond notebook source, or demonstrate an end-to-end run.
