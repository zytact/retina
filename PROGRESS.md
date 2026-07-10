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

### Phase 3 progress
- Added a new **Phase 3 — OCTA-SimCLR Pretraining** section to `main.ipynb`.
- Implemented layer-specific SimCLR pretraining components entirely inside notebook cells:
  - OD/OS bilateral natural-positive pair dataset with unilateral same-image augmented fallback,
  - conservative OCTA augmentations: horizontal flip, mild rotation, random resized crop, Gaussian blur, and mild intensity jitter,
  - 1-channel ConvNeXt-Tiny backbone initialized from ImageNet when available,
  - 2-layer projection head (768 → 256 → 128),
  - NT-Xent loss with temperature 0.07,
  - independent Sup/Deep/CC training loop with SGD, momentum, weight decay, cosine LR, AMP, gradient accumulation, checkpointing, and backbone-only export.
- Added Phase 3 pair-summary/audit outputs and a backbone registry for later Phase 4 loading.
- Set expensive CUDA pretraining gate to `RUN_PHASE3_PRETRAINING = True` for the GPU run.
- Added `PHASE3_REQUIRE_CUDA = True` so Phase 3 fails fast if accidentally launched in a non-CUDA runtime.
- Added Phase 3 bounded print logging in `main.ipynb`:
  - disabled tqdm/widget progress by default to avoid frozen notebook progress bars,
  - set default physical batch size to 128 and Windows-safe `PHASE3_NUM_WORKERS = 0`,
  - prints first 3 epochs, every 5 epochs, final epoch, and each completed layer-run with elapsed/ETA,
  - writes exact per-epoch history CSV after every epoch to avoid spamming notebook output.
- Added Phase 3 flat-loss safety diagnostics after observed loss stayed at `ln(255) ≈ 5.541`:
  - preflight cell checks one throwaway optimizer step, grad flow, parameter delta, embedding/feature variance, positive-vs-negative cosine stats, and saves `outputs/phase3_octa_simclr/phase3_preflight_diagnostics.csv`,
  - per-epoch history now records NT-Xent random-baseline loss and delta,
  - layer-run aborts at epoch 10 if loss remains within `0.02` of random baseline, saving a checkpoint first.
- After preflight showed `inf` gradients and zero parameter update, disabled Phase 3 AMP, lowered LR from `0.1` to `0.03`, added finite-gradient checks, and added gradient clipping at norm `1.0`.

## 2026-06-22

### Phase 3 resumability
- Made Phase 3 OCTA-SimCLR training resumable in `main.ipynb`.
- Changed Phase 3 config cell to enable automatic resume from `*_last.pt` and save the last checkpoint every epoch.
- Changed Phase 3 checkpoint/training-loop cell to:
  - centralize last/best/backbone checkpoint paths,
  - load existing `*_last.pt` automatically,
  - restore model, optimizer, scheduler, history, and RNG state,
  - continue at the next epoch,
  - treat already-complete checkpoints as complete and export the backbone without retraining.
- Changed Phase 3 launch cell to print resume settings before layer runs.

## 2026-06-23

### Phase 3 logging update
- Changed `main.ipynb` Phase 3 console logging to restore live visibility without tqdm.
- Phase 3 now prints:
  - run start/resume summary with remaining epochs and ETA if prior epoch timings exist,
  - `START epoch ...` before every epoch,
  - `END epoch ...` after every epoch with loss, LR, epoch time, average epoch time, elapsed time for the current resumed run, estimated remaining time, and estimated total layer-run time.
- Set `PHASE3_LOG_EVERY_N_EPOCHS = 1` so end-of-epoch logs print every epoch by default.

### Phase 1/2 fast reload
- Added a standalone fast-reload cell to `main.ipynb` before Phase 3.
- The cell restores imports, constants, preprocessing helpers, `manifest_clin`, `phase2_manifest`, `phase2_folds`, `clinical_preprocessors`, and `layer_stats_by_fold` from saved Phase 1/2 outputs.
- It checks required artifact files first and raises a clear message to rerun Phase 1/2 if any are missing.

### Planning update
- Updated `PLAN.md` to replace optional human-provided masks with self-generated OCTA pseudo-masks.
- Added classical vessel/FAZ/CC flow-deficit pseudo-mask generation, optional frozen segmentation-student refinement, mask QC, biomarker extraction, and classifier mask/biomarker fusion.
- Adjusted losses, ablations, contributions, and deliverables to reflect weak segmentation without manual ground-truth masks.
- Note: existing Phase 2 notebook placeholder masks now need replacement by generated masks in a later implementation pass.
- Updated HTML docs under `docs/` to reflect self-generated OCTA pseudo-masks, optional segmentation student, generated biomarkers, tabular fusion, and current notebook status through Phase 3.
- Added `docs/segmentation-pseudo-masks-explainer.html` as the dedicated explainer for classical masks, optional Attention U-Net/U-Net++ student refinement, mask QC, and dataset fields.

## 2026-07-02

### Docs update
- Added `docs/architecture-overview-diagram.svg` as the standalone high-level architecture diagram for guide/advisor use.
- Added `docs/architecture-overview-guide.html` as a higher-level architecture explainer for advisor/supervisor discussions.
- Added `docs/architecture-overview-guide.md` as a pointer note.
- Framed the pipeline at a broader system level: curated inputs → layer-specific encoders → cross-layer + multimodal fusion → prediction/calibration/explainability.
- Kept current-vs-next-stage separation explicit so the explainer stays honest about what is already implemented versus downstream evaluation work.

### Phase 3 completion audit
- Confirmed Phase 3 output histories exist for all 15 SimCLR runs: 5 folds × Sup/Deep/CC.
- Each history has 100 epochs, finite losses, and loss decreased well below the NT-Xent random baseline (~5.541).
- Best losses: Sup ~1.54–1.80, Deep ~1.81–2.07, CC ~2.66–2.93.
- Confirmed positive-pair summaries exist for all folds/layers, with bilateral OD/OS positives plus unilateral self-augmented fallback.
- Confirmed `.pt` checkpoint/backbone files are intentionally absent from this copy because they are too large to move.
- Noted current local `outputs/phase2_preprocessing/` copy is incomplete for fold 3/4 preprocessor/stat artifacts; rerunning fast reload here would fail unless those artifacts are restored or regenerated.

### Phase 4 progress
- Reimplemented **Phase 4 — Image Encoders with Projection Heads** in `main.ipynb` after intentional removal of old Phase 4/5 cells.
- Added independent Sup/Deep/CC ConvNeXt-Tiny encoders with no shared backbone parameters.
- Added Phase 4 projection head: `Linear(768→512) → GELU → LayerNorm → Linear(512→256) → LayerNorm`.
- Forward output now includes ordered `layer_tokens [B,3,256]`, per-layer tokens, pooled 768-d features, and retained final spatial maps for Grad-CAM.
- Added fold/layer-specific Phase 3 backbone loading helpers with metadata validation and graceful fallback when large `.pt` files are absent locally.
- Added backbone availability audit at `outputs/phase4_image_encoders/phase4_backbone_availability.csv`.
- Added a clearly marked **SUMMARIZATION CELL — Status through Phase 3 only** before Phase 4, so Phase 1–3 artifact health can be checked without rerunning analysis/preprocessing/pretraining.

### Phase 5 progress
- Added **Phase 5 — Tabular Metadata Encoder** cells to `main.ipynb`.
- Implemented fold-aware tabular encoder support:
  - clinical learned missing-token imputation with one trainable scalar token per feature,
  - learned scalar embeddings for ordinal clinical fields (e.g. smoking),
  - optional generated OCTA biomarker stream with fold-fitted z-score stats,
  - learned biomarker missing tokens for missing/QC-failed biomarker values,
  - training-time random masking of 10% observed clinical features.
- Implemented Phase 5 architecture exactly as planned:
  - `LayerNorm(D_tab)` → `Linear(D_tab→128)` → `GELU` → `Dropout(0.3)` → `Linear(128→256)` → `GELU` → `LayerNorm` → `Dropout(0.2)`.
- Added utilities to infer Phase 8 biomarker columns when available, save fold-wise biomarker stats, save tabular schema JSON files, and write `outputs/phase5_tabular_encoder/phase5_tabular_encoder_summary.csv`.
- Added a no-image-read smoke test for output shape `[B, 256]` when Phase 2 artifacts are loaded.
- Revised Phase 5 after Phase 4 output review: biomarker detection now includes current `FAZ/Faz/DENS_/PERF_` table columns, and eye-specific biomarker columns ending `_OD`/`_OS` are aligned to each sample's `eye` instead of feeding both eyes as separate tabular features.

### Phase 6 progress
- Added **Phase 6 — Cross-Layer Attention Module** cells to `main.ipynb`.
- Implemented layer-identity embeddings for ordered Sup/Deep/CC tokens before attention.
- Implemented pre-norm transformer encoder block with 8 heads, 32-d head size, 256→1024→256 GELU feed-forward path, dropout 0.1, residuals, and mean pooling to `F_octa ∈ R^256`.
- Added attention extraction and layer-dominance utility for later explainability: per-sample Sup/Deep/CC dominance from attention received across heads/query tokens.
- Added `Phase6LayerAwareOCTAEncoder` wrapper to compose Phase 4 image encoders with Phase 6 attention.
- Added smoke test and saved Phase 6 summary/audit artifacts under `outputs/phase6_cross_layer_attention/` when run.

### Phase 7 progress
- Added **Phase 7 — Multimodal Cross-Attention Fusion** cell to `main.ipynb`.
- Implemented `Phase7CrossAttentionFusion`: OCTA query attends to tabular/biomarker context with 4-head cross-attention, residual add, LayerNorm, mask concat, and 512-d unified projection.
- Implemented `Phase7MaskEncoder`: 5 generated pseudo-mask channels (`sup_vessel`, `deep_vessel`, `sup_faz`, `deep_faz`, `cc_flow`) through a lightweight CNN to `F_mask ∈ R^128`.
- Added `phase7_stack_generated_masks` to stack Phase 2/8 mask tensors in canonical order.
- Added `Phase7MultimodalFusionModel` wrapper that can compose Phase 6 OCTA encoder, Phase 5 tabular encoder, and Phase 7 mask/fusion modules.
- Added no-image-read smoke test with placeholder generated masks and saved Phase 7 summary artifacts under `outputs/phase7_multimodal_fusion/` when run.
- PLAN clarity note only: Phase 7 notebook implementation was not changed during Phase 8D. Because completed cells through Phase 8D should not be rerun/redone, the current `PLAN.md` now specifies that a future agent should add forward-only Phase 9+ compatibility code/wrapper to produce `F_cond[256] + F_mask[128] -> H[512]`, with no second tabular concatenation, without requiring rerun of Phase 7/8 cells.

### Phase 8 progress
- Added **Phase 8 — Self-Generated Segmentation Masks and Biomarkers** cell to `main.ipynb`.
- Implemented deterministic classical pseudo-mask generation:
  - Sup/Deep vessel masks via percentile normalization, CLAHE-preprocessed input, Frangi/Sato vesselness, adaptive thresholding, morphology cleanup, and skeletonization.
  - Sup/Deep FAZ masks via central low-flow/avascular connected-component selection with area and centroid plausibility checks.
  - CC flow-deficit masks via local background correction, Sauvola/adaptive dark-region detection, and morphology cleanup.
- Added generated-mask QC for vessel density, FAZ area/centering, and CC flow-deficit fraction.
- Added Phase 8 biomarker extraction: vessel density, skeleton density, FAZ area/perimeter/circularity/centroid distance, CC flow-deficit fraction/count/mean area/median area.
- Added artifact generation utilities writing masks, `phase8_generated_masks_biomarkers.csv`, `phase8_mask_qc.csv`, summary JSON, and visual montage sheets.
- Added `phase8_merge_artifacts`, `RastaPhase8Dataset`, and `make_phase8_datasets_for_fold` so Phase 8 masks/biomarkers can feed Phase 5/7 without changing classifier code.
- Left `PHASE8_RUN_GENERATION = False` by default because local machine has no dataset; set it to `True` on the data machine after Phase 2/fast reload.
- Patched Phase 8 morphology cleanup for scikit-image 0.26+ deprecation warnings: replaced deprecated binary opening/closing calls and added compatibility wrappers for `remove_small_objects`/`remove_small_holes` threshold parameters.
- Reviewed newly added Phase 8 outputs: 476 samples, 321/476 QC pass (67.4%). Found montage generation bug (`plt` missing) causing 155 `phase8_error` entries and zero montage files; patched `main.ipynb` to import `matplotlib.pyplot as plt` and prevent montage failures from overriding mask QC.
- Reviewed rerun Phase 8 outputs: 476 samples, 355/476 QC pass (74.6%), 155 montage PNGs generated, no `phase8_error` column. Remaining failures are mostly vessel-density QC (Sup 49, Deep 86); FAZ failures rare (1 Sup, 3 Deep), CC flow passes all. Visual montage review suggests vessel masks are very sparse and mainly capture large/edge vessels, so classical vessel thresholding needs tuning before using student refinement.
- Updated `PLAN.md`: added explicit classical-mask improvement gate before U-Net/Attention U-Net/U-Net++; student refinement now allowed only after improved classical pseudo-labels pass QC/visual review. Added U-Net++ as optional stronger student, not current default.
- Rewrote Phase 8 vessel-mask extraction in `main.ipynb` for an improvement pass: multiscale vesselness + local contrast + top-hat scoring, adaptive/percentile candidate thresholds, lighter morphology, bottom-right watermark suppression, tighter vessel QC density range, and `mask_source='classical_improved'`. Added montage definition to `PLAN.md`.
- Made `PLAN.md` explicit that U-Net++ must not be silently skipped: after improved classical masks, agent must document Decision A/B/C. Default expectation is to evaluate U-Net++ unless classical-only is explicitly justified by QC/montages.
- Reviewed `classical_improved` Phase 8 outputs: 476 samples, 473/476 QC pass (99.4%). Vessel densities improved substantially (Sup mean 0.127, Deep mean 0.096). Remaining failures are 3 FAZ failures only. Found montage directory contains stale PNGs from prior run because old montages were not cleared; patched `main.ipynb` to delete old montage PNGs before generation. Current mask files themselves look improved via a fresh contact sheet; rerun Phase 8 once more to refresh montage sheets.
- Checked PLAN Step C and added explicit U-Net++ decision gate to `main.ipynb` as **Phase 8C/D** instead of moving to Phase 9.
- Current documented Phase 8 gate decision is **pending**. The config default is `B_train_unetpp_student`, but this is only a default option, not a completed gate decision.
- Set default Step C option to `B_train_unetpp_student` with training disabled by default (`PHASE8_RUN_STUDENT_TRAINING = False`) until refreshed Phase 8 artifacts are available on data/GPU machine.
- Implemented compact U-Net++ student code in-notebook: QC-passed pseudo-mask dataset, Sup/Deep 2-head vessel+FAZ task, CC flow-deficit task, Tversky+Focal+Dice loss, Dice/IoU metrics, fold training loop, checkpoint/history artifact outputs, and decision summary CSV/JSON.
- Patched Phase 8C/D U-Net++ fold launch bug: `task` now remains `supdeep`/`cc`; fold-specific names are passed separately via `run_name` (e.g. `fold0_supdeep`).
- Patched Phase 8C/D missing import: added `import torch.nn.functional as F` for U-Net++ upsampling/loss functions.
- Added Phase 8C/D U-Net++ Phase-3-style training visibility and resumability: `PHASE8_STUDENT_AUTO_RESUME=True`, per-epoch START/END logs with loss/Dice/IoU/time/ETA/best Dice, `last.pt` every epoch with optimizer/history, `best.pt`, and history CSV resume support.

### PLAN clarity audit after Phase 8D
- Reviewed `PLAN.md` for implicit downstream assumptions after Phase 8D.
- Tightened Phase 9+ wording so the tabular encoder, mask encoder, fusion modules, prototype parameters, selected `mask_source`, and generated-biomarker handling are explicitly included in training, loss, ablation, baseline, explainability, statistical-validation, and deliverable sections.
- Clarified current Phase 8 gate status after review: Phase 8 is **not complete** for downstream Phase 9 training yet. Current local outputs include refreshed `classical_improved` masks for 476 samples, automated QC pass rate ~0.994, and 51 non-empty montage PNGs. Phase 8 U-Net++ student training has started on the separate training/GPU machine; it is only disabled in this local copy. Required next steps are:
  1. bring back/inspect training-machine U-Net++ outputs,
  2. visually review refreshed montages and QC failures,
  3. keep documented Decision B only if U-Net++ student artifacts are actually produced: checkpoints, history, Dice/IoU, student-vs-classical montage comparison, and biomarker stability comparison,
  4. otherwise revise the decision to A (`classical_improved` only, with QC/montage rationale) or C (tune classical masks again) before Phase 9.
- Updated `PLAN.md` after single-review findings: smoking is ordinal only; `mask_source` enum is explicit (`classical_improved`, `unetpp_student`, `attention_unet_student`); SimCLR LR/AMP/grad-clipping are documented as the already-completed stable run settings and do not require rerun; Phase 7 fusion is described as residual tabular conditioning because one-token cross-attention degenerates; Phase 7 final fusion dimensions are explicit (`F_cond[256] + F_mask[128] -> H[512]`) but should be handled by forward-only Phase 9+ compatibility code, not by rerunning completed Phase 7/8 cells; canonical fold stratification remains cohort + age decade to preserve current Phase 2/3 artifacts; default calibration is post-hoc temperature scaling rather than train-time ECE; Phase 19 deliverables now respect the notebook-only implementation constraint.
- Follow-up audit found two remaining handoff traps and patched only docs, not `main.ipynb`: Phase 7 notebook still uses the completed first-pass `Concat([f_cross, f_mask, f_tabular]) 640→512` contract, so Phase 9 must add a forward-only compatibility wrapper if using the cleaner `F_cond[256] + F_mask[128] -> H[512]` contract; Phase 8 notebook default `B_train_unetpp_student` is configured for the training-machine run, but local disabled training/no local student artifacts is not enough for Phase 9 to use `unetpp_student`.
- Adjusted `PLAN.md` so Phase 1 biomarkers refer to available publication-table/image-derived metrics, Phase 2 sample contract states masks are placeholders until Phase 8 enrichment, Phase 7 documents the implemented notebook contract, and Phase 9 explicitly verifies the Phase 8 Decision A/B/C artifact state before training.

## 2026-07-06

### Phase 8D artifact-policy clarification
- Clarified in `AGENTS.md` that the project runtime is the separate Windows GPU machine; this local checkout is for code/docs review, so Linux path portability and missing local `.pt` checkpoints are not blockers by themselves.
- Clarified in `PLAN.md` that Phase 8D `.pt` checkpoints may remain on the training machine. The copied `outputs/phase8_unetpp_student/*/history.csv` and aggregate history provide the available Dice/IoU metrics locally.
- Kept student-vs-classical montage comparison and biomarker stability comparison as preferred future validation artifacts, but documented that they were not produced for this completed Phase 8D run because adding them now would require additional student inference/rerun work. Do not rerun completed Phase 8D solely to create those retrospective comparisons; generate them only if Phase 8D is rerun or extended later.
- Audited Phase 8D segmentation metrics in `outputs/phase8_unetpp_student/phase8_unetpp_student_metric_audit.csv` and `.json`: existing histories support validation loss, Dice, IoU, and derived foreground F1 (same formula as Dice for binary masks). Pixel accuracy/balanced accuracy/precision/recall/specificity and separate test metrics were not logged and cannot be reconstructed from aggregate Dice/IoU alone without stored predictions/confusion counts or rerun/extra inference.
- Patched Phase 8C/D U-Net++ rerun code in `main.ipynb` to log expanded segmentation metrics: validation Dice/F1, IoU, pixel accuracy, balanced accuracy, precision, recall/sensitivity, specificity, FPR/FNR, predicted/target positive rates, and optional outer-fold test metrics. Added `PHASE8_STUDENT_USE_INNER_VAL_FOR_TEST=True`, which keeps each existing Phase 2 fold as an outer test split and uses a patient-grouped inner validation split for best-epoch selection before writing per-run `test_metrics.csv/json` and aggregate `phase8_unetpp_student_all_test_metrics.csv`.
- Removed the Phase 9, Phase 10, Phase 11, Phase 12, and Phase-12 fast-reload cells from `main.ipynb` at user request, so the notebook now stops after the Phase 8C/D U-Net++ segmentation-student cell.

### Phase 9 progress
- Added **Phase 9 — Classification Head with Prototype Augmentation** to `main.ipynb`.
- Added Phase 9 preflight checks that:
  - audit the completed Phase 7 fusion contract and explicitly require the forward-only compatibility wrapper instead of rerunning Phase 7/8,
  - audit Phase 8D U-Net++ history CSVs and selected `mask_source`,
  - allow local `.pt` checkpoint absence because checkpoints may remain on the Windows GPU machine.
- Implemented `Phase9CompatibilityFusion` with the intended forward contract:
  - `F_cond = LayerNorm(F_octa + Proj_tab(F_tabular))`,
  - `F_mask = MaskEncoder(masks)`,
  - `H = Project(Concat([F_cond, F_mask]) 384 → 512)`.
- Implemented `Phase9PrototypeClassifier`:
  - standard `512 → 256 → 128 → N_classes` head,
  - EMA class prototypes in the 128-d penultimate space,
  - prototype logits `-||z - P_k||² / τ_proto`,
  - learned non-negative prototype weight via `softplus(alpha)`.
- Added helpers for:
  - post-hoc temperature scaling on validation logits,
  - MC-dropout inference with mean probabilities, epistemic variance, and entropy,
  - optional frozen Phase 8D U-Net++ online mask generation from training-machine checkpoints,
  - rebuilding Phase 5 biomarker specs/statistics against the Phase 8-enriched manifest before Phase 9 datasets/models.
- Added `Phase9MultimodalClassifier`, `make_phase9_datasets_for_fold`, and `build_phase9_model_for_fold` scaffolding for later Phase 12 training.
- Added a no-image-read Phase 9 smoke test for fusion, prototype head, and temperature-scaling helper.
- Patched Phase 9 after review findings:
  - `Phase9FrozenUNetPPMaskGenerator.forward()` now refuses to run unless both Phase 8D checkpoints are actually loaded, preventing random initialized student masks.
  - `Phase9MultimodalClassifier` now refuses to report `unetpp_student` unless a loaded generator exists or batch masks are explicitly marked `mask_source='unetpp_student'`.
  - `build_phase9_model_for_fold()` now records requested vs effective mask source and falls back to `classical_improved` when local/GPU-runtime U-Net++ checkpoints are unavailable and strict checkpoint loading is disabled.

### Phase 10 progress
- Added **Phase 10 — Loss Functions** to `main.ipynb`.
- Implemented training-fold class-count utilities, inverse-frequency class weights, imbalance-ratio audit, and automatic selection between weighted cross-entropy and focal loss.
- Implemented `Phase10WeightedLabelSmoothingCE` for the default weighted CE + label smoothing objective.
- Implemented `Phase10FocalLoss` for severe imbalance fallback when max/min class frequency exceeds the configured threshold.
- Added `phase10_build_loss_bundle()` for Phase 12 training integration, with `WeightedRandomSampler` disabled by default and class-weight disabling when sampler ablation is enabled to avoid double-correcting minority classes.
- Added `Phase10DiseaseClassifierLoss` so the default disease-classifier objective is explicitly `L_total = L_cls`, excluding segmentation-student loss and train-time calibration loss.
- Added optional outer-fold loss-plan CSV generation when `phase2_folds` is loaded, plus a smoke-test cell path and `outputs/phase10_losses/phase10_summary.json`.

### Phase 11 progress
- Added **Phase 11 — Data Augmentation** to `main.ipynb`.
- Implemented synchronized training-only OCTA augmentation for Sup/Deep/CC: horizontal/vertical flips, ±15° rotation, small translation/scale/shear, mild brightness/contrast jitter, and mild Gaussian noise.
- Implemented mask-aware augmentation so Phase 8 generated masks receive the same geometric transform as the images using nearest-neighbour interpolation, with no intensity or noise corruption.
- Added clinical-feature augmentation: small Gaussian noise on observed continuous features and random one-feature observed masking during training only; generated biomarkers are left uncorrupted.
- Added `RastaPhase11Dataset`, `make_phase11_datasets_for_fold()`, and a Phase 11-aware override of `make_phase9_datasets_for_fold(..., use_phase11_augmentation=True)` for Phase 12 integration.
- Added a no-image-read smoke test and summary artifacts under `outputs/phase11_data_augmentation/` when the cell is run.

### Phase 12 progress
- Added **Phase 12 — Training Strategy** to `main.ipynb`.
- Implemented the 3-stage downstream schedule scaffold:
  - Stage 1 explicitly reuses completed Phase 3 SimCLR backbones and does not rerun pretraining.
  - Stage 2 trains downstream classifier modules with Sup/Deep/CC OCTA backbones frozen for 15 epochs at LR `3e-4`.
  - Stage 3 unfreezes OCTA backbones and fine-tunes end-to-end for up to 100 epochs with backbone LR `3e-5`, head LR `3e-4`, 5-epoch warmup, cosine warm restarts, gradient clipping, AMP, and early stopping on inner-validation macro-F1.
- Added patient-grouped outer-fold training support with an inner patient-grouped validation/calibration split from the four non-held-out folds, preserving OD/OS grouping and fixed Phase 2 fold assignments.
- Integrated Phase 9/10/11 components into the Phase 12 training path: Phase 9 multimodal model, Phase 10 weighted CE/focal-loss bundle, Phase 11 synchronized image/mask/clinical augmentation, prototype initialization after the first classifier epoch, EMA prototype updates, and post-hoc temperature scaling on the inner validation split.
- Added checkpoint/history outputs under `outputs/phase12_training/fold_{k}/`, including `stage2_*`, `stage3_*`, per-stage history CSVs, fold summaries, calibrated outer-test prediction CSVs, and cross-validation summary writers.
- Kept heavy training disabled by default (`PHASE12_RUN_TRAINING = False`) for this local review checkout; enable it on the Windows GPU/runtime machine after Phase 1–11 cells/artifacts are available.
- Patched Phase 12 after first GPU run feedback: added automatic resume from `stage*_last.pt`, restored optimizer/scaler state, reused existing history CSVs, and added Phase-3-style START/END epoch logging with LR, losses, macro-F1, balanced accuracy, epoch time, rolling average, ETA, prototype state, and checkpoint names.
- Patched Phase 12 AMP scaler construction to use the newer `torch.amp.GradScaler('cuda', ...)` API when available.
- Patched Phase 12 checkpoint loading to pass `weights_only=False` intentionally for trusted project checkpoints that contain optimizer/scaler metadata, avoiding PyTorch's future-warning ambiguity.
- Patched Phase 9 U-Net++ checkpoint loading similarly for trusted local/training-machine checkpoints.
- Changed Phase 12 `PHASE12_PIN_MEMORY` default to `False` because the Windows/Jupyter GPU run hit CUDA OOM inside DataLoader pin-memory before the model forward pass; pin-memory remains opt-in.
- Added a **FAST RELOAD / SUMMARY — Everything Phase 12 needs after kernel restart** notebook cell immediately before Phase 12. It reads `main.ipynb`, executes the Phase 1/2 disk fast-reload cell plus Phase 3 definition cells and Phase 4–12 definition cells, forces Phase 3/12 heavy run flags off during reload, writes `outputs/phase12_training/phase12_fast_reload_summary.json`, and leaves the runtime ready to call `phase12_train_cross_validation(PHASE12_FOLDS)` manually.
- Fixed Phase 12 fast-reload behavior after GPU-runtime feedback: the Phase 8 cell still had `PHASE8_RUN_GENERATION=True`, so reload could accidentally regenerate classical masks. Changed Phase 8 default to `False` and made the Phase 12 fast-reload cell forcibly patch `PHASE8_RUN_GENERATION=False` and `PHASE8_RUN_STUDENT_TRAINING=False` while loading definitions.
- Corrected the Phase 12 fast-reload responsibility boundary: restored Phase 8's own intentional-run flag behavior, and hardened the fast-reload cell so it unconditionally rewrites any `PHASE8_RUN_GENERATION = ...` and `PHASE8_RUN_STUDENT_TRAINING = ...` assignments to `False` only within the executed reload copy. This keeps previous cells unchanged while making the summary/reload cell safe after kernel restart.
- Hardened the Phase 12 fast-reload cell further so it also forces all smoke/preflight allocation flags off while loading definitions: Phase 4/6/7/9/10/11/12 smoke tests and Phase 9 preflight are disabled in the executed reload copies. Reload now should not train, regenerate masks, run Phase 3 preflight/pretraining, or allocate GPU memory for smoke tests; it only restores definitions/artifacts and writes a lightweight reload summary.
- Corrected Phase 12 fast-reload ordering after user feedback: because the reload cell sits immediately before Phase 12, it no longer executes the Phase 12 cell from notebook source. It now restores only prerequisites through Phase 11, then instructs the user to run the visible Phase 12 cell below in normal notebook order before launching/resuming training.
- Patched the Phase 9 frozen U-Net++ checkpoint loader to try `torch.load(..., weights_only=True)` first and only fall back to a trusted full checkpoint load with the PyTorch FutureWarning suppressed. Also wrapped Phase 12 trusted checkpoint resume loads to suppress the same warning intentionally for self-produced optimizer/scaler checkpoints.
- Fixed the Phase 12 training-strategy cell so classifier setup no longer loads Phase 8D U-Net++ checkpoints by default. Added `PHASE12_USE_ONLINE_STUDENT_MASKS=False` and mask-source resolution: Phase 12 now consumes masks already present in the Phase 8 manifest; if `unetpp_student` is requested but only `classical_improved` masks are present and online loading is disabled, it explicitly falls back to `classical_improved` and records this in the fold summary. Added a Stage 1 reuse report so Phase 12 prints/audits that Stage 1 is reuse of completed Phase 3 SimCLR backbones, not a skipped training stage.
- Hardened the Phase 9 frozen U-Net++ checkpoint loader after the GPU warning at `cell_33`: the `TypeError` fallback now suppresses PyTorch's trusted `weights_only=False` FutureWarning too. Also clarified the Phase 12 Stage 1 report text (`completed_by_reusing_existing_phase3_backbones_no_training_in_phase12`) so warning noise is not mistaken for Stage 1 being skipped.
- Verified the Phase 12 cell against `PLAN.md`: Stage 1 reuses Phase 3 SimCLR backbones, Stage 2 freezes only OCTA backbones while training downstream modules, Stage 3 unfreezes OCTA backbones with 10x lower backbone LR, patient-grouped outer/inner splits are used, and any segmentation student remains frozen/offline by default. Hardened resumability by trimming history to the last checkpoint epoch, saving `best` before `last`, persisting `best_macro_f1`/`stale_epochs`/`early_stopped` in `stage*_last.pt`, and skipping extra epochs when resuming a previously early-stopped stage.

### Phase 8C/D reload and training visibility
- Split the previous combined Phase 8C/D notebook cell into Phase 8C definitions/decision recording and Phase 8D training launch, so Phase 8D can be rerun independently after prerequisites are loaded.
- Added a **FAST RELOAD / SUMMARY — Everything Phase 8D needs up to and INCLUDING Phase 8C** cell immediately before Phase 8D. It executes prerequisite definition/artifact-loading cells through Phase 8C with heavy work disabled: no Phase 3 training/preflight, no Phase 4/6/7 smoke allocation, no Phase 8 classical regeneration, and no Phase 8D student training. It merges existing Phase 8 artifacts into `phase2_folds_with_phase8`, prints readiness/QC/fold/mask-source summary, and writes `outputs/phase8_unetpp_student/phase8d_fast_reload_summary.json/.csv`.
- Clarified the Phase 8D fast reload output so it explicitly lists executed phase labels, states `Phase 8C included: True`, and hard-asserts required Phase 8C objects such as `Phase8UNetPlusPlusStudent`, `phase8_evaluate_student_model`, and `train_phase8_unetpp_student` before allowing Phase 8D to run.
- Verified and hardened Phase 8D resumability: `PHASE8_STUDENT_AUTO_RESUME=True`, each run loads `last.pt`, restores model/optimizer/history, resumes from the next epoch, writes `history.csv` and `last.pt` every epoch, keeps `best.pt`, and reuses existing `test_metrics.json` when a completed checkpoint is rerun.
- Expanded Phase 8D console visibility so every epoch prints `START epoch ...` and `END epoch ...` with train/validation loss plus Dice/F1, IoU, pixel accuracy, balanced accuracy, precision, recall/sensitivity, specificity, FPR, FNR, predicted-positive rate, target-positive rate, timing, ETA, and best Dice. Test-metric prints now show the same expanded metric set.

## 2026-07-07

### Phase 8D mask-source policy correction
- Updated `PLAN.md` so downstream Phase 9+ work must use `mask_source = "unetpp_student"` for this project.
- Clarified that `classical_improved` masks are pseudo-label targets for U-Net++ training and QC, not the operational downstream mask source.
- Updated `main.ipynb` Phase 8 artifact merge/reload behavior so old Phase 8 CSVs with `mask_source = classical_improved` are treated as `pseudo_label_source`, while the effective downstream `mask_source` is set to `unetpp_student`.
- Added Phase 8D output scaffolding to write `phase8_selected_mask_source_summary.*` after GPU training, making the selected downstream source explicit without pretending classical mask files are U-Net++ outputs.
- Cleared stale outputs from the edited Phase 8 cells so old local review text does not imply classical masks are the downstream source.

### Phase 9 and 10 implementation
- Added **Phase 9 - Classification Head with Prototype Augmentation** to `main.ipynb`.
- Implemented frozen fold-specific Phase 8D U-Net++ checkpoint loading through `Phase9FrozenUNetPPMaskGenerator`; missing checkpoints now raise an error instead of falling back to classical masks.
- Added `RastaPhase9Dataset` with unstandardized `[0, 1]` OCTA tensors for the frozen student while preserving standardized OCTA tensors for the image encoder.
- Added `phase9_materialize_unetpp_masks_for_fold(fold)` to run completed U-Net++ checkpoints in inference mode and write fold-specific `unetpp_student` mask PNGs, biomarkers, QC, and manifests under `outputs/phase9_classifier/unetpp_student_masks/fold_{k}/`.
- Added Phase 9 compatibility fusion using `F_cond = LayerNorm(F_octa + Proj_tab(F_tabular))`, U-Net++ masks through the mask encoder, and `[F_cond, F_mask] -> H[512]` with no second tabular concatenation.
- Added `Phase9PrototypeClassifier`, temperature-scaling helper, MC-dropout prediction helper, dataset builders, and `build_phase9_model_for_fold()` scaffolding.
- Added **Phase 10 - Loss Functions** to `main.ipynb` with weighted label-smoothed CE, focal loss for severe imbalance, inverse-frequency class weights, optional weighted sampler helper, and `Phase10DiseaseClassifierLoss` where `L_total = L_cls` only.

### Phase 11 implementation
- Added **Phase 11 - Data Augmentation** to `main.ipynb`.
- Implemented synchronized train-only geometric augmentation shared across Sup/Deep/CC classifier images, U-Net++ student input images, and any materialized `unetpp_student` mask tensors.
- Kept validation/test samples unaugmented.
- Applied mild brightness/contrast/noise only to classifier OCTA encoder inputs, not to masks and not to U-Net++ student inputs, so frozen student inference stays stable while masks remain spatially aligned.
- Added clinical augmentation: Gaussian noise for observed continuous features and random observed-feature masking during training only.
- Added `RastaPhase11Dataset`, `phase11_make_datasets_for_fold()`, and optional `phase11_override_phase9_dataset_builder()` for Phase 12 integration.
- Phase 11 requires `mask_source = "unetpp_student"` and raises if the Phase 9 materialized U-Net++ manifest is missing or invalid.

### Phase 12 implementation
- Added **Phase 12 - Training Strategy** to `main.ipynb`.
- Implemented Stage 1 as an explicit reuse report for completed Phase 3 SimCLR backbones, with no Phase 3 rerun.
- Implemented leakage-safe outer-fold training with a patient-grouped inner validation split from the outer-training patients for early stopping and temperature scaling.
- Integrated Phase 9, 10, and 11 components: U-Net++ `mask_source`, Phase 11 train-only augmentation, Phase 10 classifier loss bundle, prototype updates, post-hoc temperature scaling, and calibrated outer-test prediction export.
- Added Stage 2 training with OCTA backbones frozen and Stage 3 fine-tuning with lower backbone LR, warmup/cosine LR scaling, AMP, gradient clipping, checkpoint resume, and early stopping on inner-validation macro-F1.
- Ensured the frozen Phase 8D U-Net++ mask generator stays in eval mode even when the parent classifier is set to train mode.
- Kept heavy training disabled by default with `PHASE12_RUN_TRAINING = False`; call `phase12_train_cross_validation(PHASE12_FOLDS)` on the training machine. If fold-specific Phase 9 U-Net++ manifests are missing, Phase 12 can auto-materialize them from the existing Phase 8D checkpoints before training.

### Phase 12 resumability/logging and split-metric update
- Updated Phase 12 stage training to print explicit `START` and `END` lines for each stage and epoch, matching the earlier long-running training cells.
- Stage 2 and Stage 3 histories now log train, inner-validation, and outer-test loss, macro-F1, balanced accuracy, and per-class recall every epoch. Outer-test metrics are logged for visibility only; best checkpoints and early stopping still use inner-validation macro-F1.
- Added epoch timing, elapsed time, ETA, LR list, gradient norm mean/max, prototype alpha, prototype-initialized class count, best validation macro-F1, and stale-epoch count to the logs/history.
- Hardened resume behavior: `last.pt` stores `early_stopped` and `completed` flags, and reruns skip already completed or early-stopped stages instead of adding extra epochs.
- Expanded Stage 1 into a reuse audit with Sup/Deep/CC backbone existence, history existence, loss summary where available, split sizes, and `stage1_reuse_report.csv/json`.

## 2026-07-09

### Phase 13 progress
- Added **Phase 13 - Explainability** to `main.ipynb` after completed Phase 12 training.
- Implemented trained fold-model loading from Phase 12 checkpoints with fold temperature reuse.
- Added Grad-CAM generation for Sup, Deep, and CC final spatial maps, including predicted-class, true-label, or explicit per-class targets, per-sample overlay PNGs, per-layer `.npy` heatmaps, and summary CSVs under `outputs/phase13_explainability/gradcam/`.
- Added cross-layer attention dominance extraction from Phase 6 attention weights, with per-sample and cohort-level summaries under `outputs/phase13_explainability/attention/`.
- Added tabular attribution support for clinical and generated-biomarker features: KernelSHAP when `shap` is installed, with an explicitly labeled occlusion fallback when it is not.
- Added integrated Grad-CAM plus attention consistency summaries, including per-layer correlation artifacts under `outputs/phase13_explainability/integrated/`.
- Kept heavy explainability disabled by default in the local review copy; run `phase13_run_cross_validation(PHASE13_FOLDS)` on the training/runtime machine after Phase 12 checkpoints are available.

### Phase 13 logging update
- Added live Phase 13 progress logging to `main.ipynb`.
- Phase 13 now prints fold start/end messages, dataset sizes, checkpoint loading, task start/end messages, batch counters, sample counters, elapsed time, ETA, and artifact paths for attention dominance, Grad-CAM, tabular attribution, integrated analysis, and cross-validation combination outputs.
- Added configurable logging controls: `PHASE13_LOG_EVERY_N_BATCHES`, `PHASE13_LOG_EVERY_N_SAMPLES`.

### Phase 13 class-balanced explainability correction
- Fixed the explainability sample-selection bias that made capped Grad-CAM and tabular attribution examples come mostly/all from the first sorted cohort (`RETINORM`).
- Added class-balanced deterministic example selection for Grad-CAM and tabular attribution using `PHASE13_CLASS_BALANCED_EXAMPLES=True`.
- Changed default Grad-CAM targets to `PHASE13_GRADCAM_TARGETS = ['pred', 'label', 'all_classes']`, so reruns generate predicted-class, true-label, and explicit per-class heatmaps.
- Added `label_name`, `pred_name`, `target_class`, `target_name`, and separate predicted-vs-target probabilities to Grad-CAM summaries and overlay titles.
- Updated integrated Grad-CAM/attention correlation artifacts to preserve target-mode/class context instead of averaging all targets together silently.
- Required real KernelSHAP by default for tabular attribution with `PHASE13_REQUIRE_REAL_SHAP=True`; added `shap` to project dependencies and refreshed `uv.lock`.
- Note: existing Phase 13 output CSVs/images were produced before this correction and remain stale until Phase 13 is rerun on the training/runtime machine.

### Phase 13 OOM hardening and output review
- Reviewed newly copied partial Phase 13 outputs. The rerun completed fold 0 attention and Grad-CAM, but stopped before combined cross-validation summaries/tabular outputs; only fold 0 artifacts are present locally.
- Confirmed fold 0 Grad-CAM selection is now class-balanced: 24 samples total, 6 each from RETINORM, ORNET, FAMILIPO, and MRCC. Fold 0 selected-sample predictions are not all RETINORM: FAMILIPO 8, RETINORM 7, ORNET 7, MRCC 2.
- Created a local contact sheet for visual review at `outputs/phase13_explainability/gradcam/fold_0/phase13_gradcam_contact_sheet_true_class_examples.png`.
- Hardened Phase 13 against 8 GB GPU OOM: Grad-CAM now defaults to batch size 1, tabular attribution iteration to batch size 1, KernelSHAP prediction calls are chunked with `PHASE13_SHAP_PREDICT_BATCH_SIZE=4`, and CUDA cache cleanup is called after Grad-CAM batches/targets, tabular samples, and folds.
- The previous CUDA OOM is most likely from KernelSHAP or repeated image batches during tabular explanation, not from class-balanced Grad-CAM itself. Recommended runtime setting for 8 GB GPUs remains `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` before launching Jupyter/Python.
- Added a dedicated **FAST RELOAD / SUMMARY - Everything Phase 13 needs through Phase 12** cell immediately before Phase 13. It sets `PYTORCH_CUDA_ALLOC_CONF` for the restarted kernel, executes prerequisite definition/artifact-loading cells through Phase 12 with all heavy work disabled, suppresses incidental summary-file writes during reload, disables Phase 12 auto-materialization of U-Net++ masks, checks existing fold checkpoints/manifests, and leaves the runtime ready to run the Phase 13 cell without rerunning earlier notebook cells or regenerating outputs.

## 2026-07-10

### Phase 14 progress
- Added **Phase 14 - Evaluation Metrics** to `main.ipynb`. It scores only Phase 12 outer-fold OOF predictions and writes a dedicated evaluation bundle under `outputs/phase14_evaluation/`.
- Implemented deterministic calibrated eye-level metrics and patient-level metrics, with patient predictions formed by averaging OD/OS probabilities. Both are accompanied by 95% stratified patient-cluster bootstrap confidence intervals (1,000 samples by default), so bilateral eyes are not treated as independent inference units.
- Implemented macro one-vs-rest ROC-AUC, macro/weighted F1, balanced accuracy, macro PR-AUC, per-class precision/recall/sensitivity/specificity/F1/AUC, true-class-normalized confusion matrices, top-2 accuracy, ECE-10, multiclass Brier score, reliability-bin CSVs, and reliability diagrams.
- Added checkpoint-based 30-pass MC-dropout artifact generation that enables only dropout modules while keeping normalisation and the frozen U-Net++ mask generator in evaluation mode. This writes separate held-out predictions, entropy, and classwise epistemic variance, then merges them only after strict OOF identity validation. It supports entropy-versus-accuracy quartiles and top-30%-confidence coverage analysis without replacing the authoritative deterministic OOF metrics.
- Added reusable paired model-comparison utilities for Phase 16 baselines: exact McNemar test, per-class one-vs-rest DeLong AUC test, Bonferroni correction, and Cohen's h. Added a disabled-by-default, no-file smoke test.
- Kept evaluation and MC-dropout disabled by default in the local review checkout. On the Windows training machine, run the existing Phase 13 fast-reload cell, then `phase14_generate_mc_dropout_predictions()` and `phase14_evaluate_predictions()` after all Phase 12 fold checkpoints are available.
- Added detailed live progress logging: MC-dropout logs each fold and batch with completed samples, elapsed time, batch duration, and ETA; optional five-pass milestones can be enabled. Evaluation logs each validation/output step, eye/patient primary metrics, every 10 bootstrap replicates by default with ETA, per-fold summaries, and final artifact locations.
