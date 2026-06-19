# PROGRESS.md

## 2026-06-19

### Completed
- Read `AGENTS.md`, `PLAN.md`, `TREE.md`, and inspected the empty `main.ipynb`.
- Converted `main.ipynb` into the runnable project entry point for **Phase 1 — Dataset Analysis** only.
- Added notebook cells for:
  - configuration via `RASTA_DATA_ROOT` and clinical spreadsheet path,
  - clinical table loading,
  - RASTA patient-eye manifest scanning,
  - class distribution, bilateral eye availability, layer availability, and leakage checks,
  - clinical missingness heatmap and demographic summaries,
  - image resolution / lightweight quality audit,
  - biomarker summaries and cohort boxplots,
  - preliminary leakage-safe patient-level fold assignment for review.
- Added lightweight supporting package files under `src/rasta_cad/` to keep notebook code clean and reusable.

### Notes
- The notebook was **not run**, per project instruction and because the dataset is not present on this machine.
- Current implementation scope is Phase 1 only; later phases should be added incrementally after reviewing Phase 1 outputs on the dataset/GPU machine.

### Adjustment
- Made `main.ipynb` fully self-contained and removed all dependency on helper Python files under `src/`.
- Future work will keep implementation inside notebook cells only, phase-by-phase.
- Updated the notebook configuration cell to make Windows dataset/clinical paths easy to edit via `DATA_ROOT_STR` and `CLINICAL_PATH_STR`.
