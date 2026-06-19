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

### Phase 2 progress
- Added a new **Phase 2 — Preprocessing Pipeline and Dataset** section to `main.ipynb`.
- Implemented notebook-local OCTA preprocessing:
  - image corruption / zero-variance / low dynamic-range / SNR QC with exclusion logging,
  - optional CNN-quality hook for a future lightweight scan-quality classifier,
  - percentile 2nd–98th normalization,
  - CLAHE with OpenCV fallback to scikit-image/no-op,
  - bicubic resize to 224×224,
  - training-fold-only OCTA layer mean/std computation.
- Implemented clinical preprocessing:
  - fold-fitted continuous z-scoring,
  - binary and ordinal feature handling,
  - missingness masks instead of mean/median imputation,
  - trainable `ClinicalMaskTokenImputer` for learned per-feature missing tokens and ordinal embeddings.
- Implemented Phase 2 artifacts:
  - QC-passed manifest and full QC exclusion logs,
  - patient-level stratified folds with zero patient-overlap validation,
  - fold-wise clinical preprocessor JSON files,
  - fold-wise clinical mutual-information reports,
  - fold-wise OCTA layer normalization statistics.
- Implemented `RastaPhase2Dataset` returning the planned sample dictionary with synchronized Sup/Deep/CC augmentation and zero placeholder masks when segmentation masks are unavailable.
