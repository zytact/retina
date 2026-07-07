# ROLE

Act as a team of experts consisting of:

1. Retinal Ophthalmologist
2. OCTA Imaging Scientist
3. Computer Vision Researcher
4. Deep Learning Architect
5. Medical AI Research Scientist
6. Biostatistician
7. IEEE Journal Reviewer

Your task is to design, critique, and implement a publication-quality multimodal OCTA classification system using the RASTA dataset.

---

# PROJECT GOAL

Build a robust, reproducible, and publication-quality multimodal classification system that classifies retinal OCTA scans into disease cohorts.

The system must:

- Leverage all three OCTA retinal layers independently before fusion
- Integrate clinical metadata with proper handling of missing values
- Produce uncertainty-quantified predictions suitable for clinical screening
- Be explainable at both the spatial (per-layer heatmap) and feature (clinical attribution) level
- Outperform single-layer and single-modality baselines

**Scope:** Version 1 — 2D OCTA images only. Do NOT use angiocubes. Angiocube integration is planned for Version 2 (roadmap in Phase 20).

---

# RESEARCH HYPOTHESIS

Different disease cohorts produce distinct retinal microvascular changes.

These changes appear differently across:

- Superficial Vascular Complex (SVC) — arteriolar and capillary density changes
- Deep Vascular Complex (DVC) — deeper capillary rarefaction, sensitive to diabetic microangiopathy
- Choriocapillaris (CC) — choroidal perfusion and endothelial function

A model that encodes each layer independently, then learns cross-layer interactions, then fuses with clinical context should outperform any single-layer or single-modality classifier.

The biological rationale is that each layer reflects a different physiological compartment. Cross-layer correlation patterns — not individual layer signals — contain additional discriminative information about disease type.

---

# DATASET DESCRIPTION

Assume access to the RASTA dataset.

Per patient (per eye):

```
Sup_OD.bmp    Deep_OD.bmp    CC_OD.bmp
Sup_OS.bmp    Deep_OS.bmp    CC_OS.bmp   (if available)
```

Clinical metadata fields:

- Age (continuous)
- Sex (binary)
- BMI (continuous)
- Smoking (ordinal: never / former / current)
- Hypertension (binary)
- Diabetes (binary)
- Dyslipidemia (binary)
- Additional available fields

Current Phase 1/2 notebook artifacts use the available clinical spreadsheet columns and eye-aligned publication-table biomarkers where present. The current target remains the disease cohort derived from folders, not a newly derived binary CAD endpoint.

Label:

Disease cohort folder (multi-class). Version 1 remains disease-cohort classification; do not pivot this implementation to binary CAD prediction unless labels, metrics, heads, baselines, and manuscript framing are intentionally redesigned later.

## Version 1 label semantics

The primary supervised target is the RASTA folder-derived cohort label, not a direct binary CAD endpoint:

- `RETINORM` — control/normal cohort from the AwARD study
- `ORNET` — obstructive sleep apnea retinal vascular network cohort
- `FAMILIPO` — familial hypercholesterolemia cohort, cardiovascular-risk/atherosclerosis related
- `MRCC` — coronary revascularization cardiac surgery cohort, the clearest CAD-related group

The clinical spreadsheet contains cardiovascular covariates and risk/comorbidity fields such as hypertension, diabetes mellitus, stroke, vascular disease, dyslipidemia, smoking, and CHA2DS2-VASc, plus OCTA biomarkers. These are model inputs or analysis covariates in Version 1, not the folder-derived class labels. CAD is represented indirectly, especially through MRCC and the broader `Vascular disease` covariate. Any future binary CAD or cardiovascular-risk model must define a separate clinically validated endpoint and should not be treated as equivalent to the current four-class cohort classifier.

Generated internally; the RASTA dataset is assumed to provide **no manual segmentation masks**:

- Vessel pseudo-masks for SVC and DVC
- FAZ pseudo-masks for SVC and DVC
- Choriocapillaris flow-deficit pseudo-masks
- Quantitative OCTA biomarkers extracted from generated masks

These masks are automated weak labels, not human-annotated ground truth. The pipeline must clearly distinguish classical pseudo-mask generation, optional segmentation-student refinement, and downstream disease classification.

---

# PHASE 1 — DATASET ANALYSIS

Perform comprehensive dataset analysis before any modeling.

Generate:

1. Total sample count (patients and eyes separately)
2. Class distribution per cohort — compute class imbalance ratio
3. Bilateral eye availability (OD only vs OD+OS)
4. Missing values per clinical feature — per cohort breakdown
5. Age, sex, BMI distributions per cohort
6. Image resolution and quality distribution
7. Layer availability audit (are all three layers present for all patients?)
8. Potential confounders (age-disease correlation, sex imbalance per cohort)
9. Data leakage risk assessment — verify no patient appears in multiple cohorts

Also generate:

- Class-wise biomarker box plots from available publication-table biomarkers and image-derived metrics. Phase 8 later adds generated-mask biomarkers; do not imply Phase 1 already has Phase 8 masks.
- Inter-cohort demographic comparison table
- Missing data heatmap (patients × clinical features)

Use these findings to inform class-weighting, stratification strategy, and imputation design.

---

# PHASE 2 — PREPROCESSING PIPELINE

## Bilateral Eye Handling

Treat OD and OS as independent samples during training and evaluation.

**Critical rule:** During patient-level splitting, both OD and OS of the same patient MUST be assigned to the same fold. Never allow the same patient's two eyes to appear in different folds. This prevents label leakage via shared patient-level disease state.

Implement a patient-stratified fold assignment that:

- Groups all eyes by patient ID before splitting
- Stratifies folds by cohort label AND patient age decade
- Verifies zero patient overlap across folds after splitting

Current completed Phase 2 artifacts were generated with patient grouping and a stratification key that includes cohort label, age decade, and sex when sex is available. Local copied artifact audit shows 476 eye samples, fold counts 94/93/98/94/97, and zero patient leakage across folds. Do not regenerate Phase 2/3 artifacts merely to remove sex from the split key; report the actual completed split strategy.

## OCTA Image Preprocessing

For each image (Sup, Deep, CC):

1. **Quality control:** SNR thresholding + lightweight CNN-based quality classifier. Flag and exclude low-quality scans. Log all exclusions.
2. **Grayscale normalization:** Percentile-based (2nd–98th percentile) per image. Avoids sensitivity to extreme pixel values.
3. **CLAHE:** Apply contrast-limited adaptive histogram equalization. Clip limit=2.0, tile grid=8×8. Apply per-layer, not globally.
4. **Resize:** Bicubic interpolation to 224×224.
5. **Intensity standardization:** Per-channel z-score normalization using training set statistics only (compute on train fold, apply to val/test fold).
6. **Corruption check:** Remove zero-variance, all-black, or unreadable images. Log counts.

## Clinical Metadata Preprocessing

1. **Continuous features** (age, BMI): z-score normalization using training fold mean and std.
2. **Binary features** (sex, hypertension, diabetes, dyslipidemia): encode as integers, preserve as-is.
3. **Ordinal features** (smoking: 0/1/2): embed as learned ordinal token, not raw integer.
4. **Missing value handling:** Do NOT use mean/median imputation. Instead, use a **learnable missing-value mask token** per feature. Current completed notebook implementation uses a learned scalar replacement per flat tabular feature, plus learned scalar embeddings for ordinal values, because the Phase 5 encoder consumes a flat tabular vector rather than per-feature token sequences. This allows the model to learn optimal behavior for absent data rather than assuming a statistical proxy.
5. **Feature selection:** Compute mutual information between each clinical feature and the disease label on the training fold. Document features with MI < 0.01 — consider dropping or flagging as low-information.

## Dataset Object

Current implemented Phase 2 returns OCTA images, clinical tensors, labels, patient/eye/fold metadata, and zero/optional placeholder masks. It does not yet return Phase 8 generated masks or generated-mask biomarkers.

Phase 2 base sample:

```python
{
  "sup_image": Tensor[1, 224, 224],
  "deep_image": Tensor[1, 224, 224],
  "cc_image": Tensor[1, 224, 224],
  "clinical_features": Tensor[D_clin],
  "clinical_mask": Tensor[D_clin],
  "vessel_mask": Tensor[1, 224, 224],  # optional/zero placeholder before Phase 8
  "faz_mask": Tensor[1, 224, 224],     # optional/zero placeholder before Phase 8
  "flow_mask": Tensor[1, 224, 224],    # optional/zero placeholder before Phase 8
  "label": int,
  "patient_id": str,
  "eye": str,
  "fold": int,
}
```

After Phase 8 enrichment, each sample additionally returns:

```python
{
  "sup_vessel_mask":  Tensor[1, 224, 224], # generated pseudo-mask
  "deep_vessel_mask": Tensor[1, 224, 224], # generated pseudo-mask
  "sup_faz_mask":     Tensor[1, 224, 224], # generated pseudo-mask
  "deep_faz_mask":    Tensor[1, 224, 224], # generated pseudo-mask
  "cc_flow_mask":     Tensor[1, 224, 224], # generated pseudo-mask
  "biomarkers":       Tensor[D_bio],       # generated mask/image biomarkers
  "mask_source":      str,                 # one of: "classical_improved", "unetpp_student", "attention_unet_student"
  "biomarker_mask":   Tensor[D_bio],       # 1=observed/QC-pass, 0=missing/QC-failed
}
```

---

# PHASE 3 — OCTA-SIMCLR PRETRAINING

**Why pretraining is essential:**

The RASTA dataset is medium-sized. ConvNeXt-Tiny initialized from ImageNet has been trained on natural RGB images — a large domain gap from grayscale vessel-contrast OCTA. Self-supervised pretraining on unlabeled OCTA (or all available RASTA images before labels are used) adapts the visual representations to the retinal vascular domain before fine-tuning.

## Positive Pair Strategy: Bilateral Natural Positives

For patients with both OD and OS:

**Positive pair:** (Sup_OD, Sup_OS) from the same patient.

The two eyes of the same patient share the same systemic disease state. Their OCTA images are the same modality and disease context, just mirrored. This is a strong, medically meaningful positive pair signal without any manual annotation.

For patients with only one eye:

**Positive pair:** Two augmented views of the same image (standard SimCLR).

Current completed Phase 3 implementation uses strict fold-training-only self-supervised pretraining (`PHASE3_PRETRAIN_SCOPE = 'fold_train'`), not transductive all-data pretraining. Each downstream fold has separate Sup/Deep/CC SimCLR histories/backbones trained without that fold's held-out patients.

## Architecture

- Backbone: ConvNeXt-Tiny (initialized from ImageNet)
- Projection head: 2-layer MLP (768 → 256 → 128), appended during pretraining only, discarded afterward
- Loss: NT-Xent (normalized temperature-scaled cross-entropy) with temperature τ=0.07

Three independent contrastive tasks: one per OCTA layer (Sup, Deep, CC).

The encoders do NOT share weights — separate SimCLR pretraining per layer.

## Augmentation during pretraining

Use conservative augmentations that preserve vascular structure:

- Random horizontal flip (p=0.5)
- Random rotation ±15° (p=0.4)
- Random crop (scale=0.7–1.0) + resize to 224×224
- Gaussian blur (σ=0.1–2.0, p=0.3)
- Intensity jitter (brightness ±0.1, contrast ±0.1) — applied mildly

Do NOT use: color inversion, aggressive distortion, cutout, or random erasing. These destroy the vessel density signal.

## Pretraining schedule

- Epochs: 100
- Optimizer: SGD with momentum=0.9, weight_decay=1e-4
- Historical/current successful run setting: LR=0.03 with cosine decay. This superseded the original LR=0.1 attempt because the first GPU pass showed unstable/`inf` gradients and no parameter update at LR=0.1.
- Historical/current successful run setting: batch size 256 effective, using physical batch size 128 plus gradient accumulation if memory limited
- Historical/current successful run setting: AMP disabled for SimCLR after failed gradient preflight
- Historical/current successful run setting: gradient clipping max norm 1.0
- Abort/safety diagnostics: compare loss against NT-Xent random baseline and save preflight diagnostics if loss remains flat

These Phase 3 notes document the already-completed successful pretraining setup. They do **not** require rerunning Phase 3 if valid Phase 3 histories/backbones already exist.

After pretraining, retain only backbone weights. Discard projection heads.

---

# PHASE 4 — IMAGE ENCODERS WITH PROJECTION HEADS

## Backbone

Architecture: **ConvNeXt-Tiny**

- Initialized from ImageNet pretrained weights
- Further adapted via OCTA-SimCLR (Phase 3)
- Output dimension from final stage: 768

Three independent encoders: one each for Sup, Deep, CC.

Do NOT share backbone weights between layers.

**Biological justification:** Each retinal layer has distinct vascular architecture. SVC shows arterioles and capillaries in a 2D mesh. DVC shows sparser, finer capillaries in a different depth plane. CC shows a mosaic perfusion pattern. These textures are fundamentally different — a shared backbone would impose the same filter bank on all three, limiting layer-specific feature learning.

## Projection Heads (post-encoder, pre-attention)

After global average pooling (768-d), apply a per-encoder projection head:

```
FC(768 → 512) → GELU → LayerNorm
FC(512 → 256) → LayerNorm
```

Output: 256-d token per layer.

**Why project before attention?**

768-d tokens in multi-head attention are computationally expensive and prone to overfitting with small N. Projecting to 256-d before the attention module reduces parameters, improves training stability, and is standard practice in vision-language transformer fusion.

## Spatial Feature Map Retention

Before global average pooling, extract the final spatial feature map (7×7×768 after ConvNeXt-Tiny's last stage) and retain a reference for Grad-CAM computation. Do not pool prematurely.

## Output per encoder

- Fs: Sup features ∈ ℝ^256
- Fd: Deep features ∈ ℝ^256
- Fc: CC features ∈ ℝ^256

---

# PHASE 5 — TABULAR METADATA ENCODER

## Input handling

Concatenate all processed clinical features into a single vector of dimension D_clin. After Phase 8, append generated OCTA biomarkers to form D_tab = D_clin + D_bio.

Missing clinical features are replaced by their learned mask token (trainable 1-d embedding per feature, not a fixed zero or mean). Biomarker missingness can occur if mask QC fails; represent failed biomarkers with a biomarker-missing mask token and explicit QC flag.

## Architecture

```
Input: [D_tab]
LayerNorm(input)
FC(D_tab → 128) → GELU → Dropout(0.3)
FC(128 → 256) → GELU → LayerNorm → Dropout(0.2)
```

Output: F_tabular ∈ ℝ^256

## Why GELU over ReLU for clinical data

Clinical feature distributions are often non-Gaussian and contain negative z-scored values. GELU's smooth, non-zero gradient near zero allows better gradient flow for near-zero features (e.g., slightly below-average BMI). ReLU hard-zeros these, losing gradient information.

## Missing value training strategy

During training, randomly mask 10% of observed clinical features as if missing (set to mask token). This simulates real-world missingness and makes the encoder robust to absent clinical data at inference time.

---

# PHASE 6 — CROSS-LAYER ATTENTION MODULE

**Goal:** Model interactions between SVC, DVC, and CC feature representations before fusing with clinical data.

## Why cross-layer interactions matter

SVC pathology (e.g., arteriolar narrowing) predicts DVC rarefaction in hypertension. CC flow deficits correlate with SVC perfusion changes in diabetic retinopathy. These cross-layer correlation patterns differ across disease cohorts and constitute discriminative information not captured by any single encoder.

## Architecture

**Step 1: Layer-identity embedding**

Before attention, add a learnable layer-identity positional embedding to each token:

```
T_sup  = Fs + e_sup
T_deep = Fd + e_deep
T_cc   = Fc + e_cc
```

Where e_sup, e_deep, e_cc ∈ ℝ^256 are learnable layer-identity vectors, initialized independently.

**Why this matters:** Without layer-identity embeddings, multi-head self-attention treats the three tokens as an unordered set. The model cannot learn that "token 2 is always the DVC" because the tokens have no identity marker — any permutation produces identical attention scores. Layer-identity embeddings fix this.

**Step 2: Multi-head self-attention**

Stack the three tokens as a sequence: S = [T_sup ; T_deep ; T_cc] ∈ ℝ^(3×256)

Apply transformer encoder block:

- 8 attention heads, head dimension = 32
- Pre-norm (LayerNorm before attention)
- Feed-forward: 2-layer MLP (256 → 1024 → 256) with GELU
- Dropout = 0.1

**Step 3: Aggregation**

Apply mean pooling over the three attended tokens:

F_octa = MeanPool([H_sup; H_deep; H_cc]) ∈ ℝ^256

**Output:** F_octa — Layer-Aware OCTA Representation

---

# PHASE 7 — MULTIMODAL CROSS-ATTENTION FUSION

**Goal:** Fuse imaging representation (F_octa) with tabular context (F_tabular: clinical features + generated biomarkers) and generated segmentation-mask features (F_mask).

**Why the current fusion block is still useful:**

Version 1 currently uses a single pooled tabular token, so the attention block is mathematically equivalent to a learned tabular projection/residual conditioning step rather than selective attention over individual clinical fields. This is acceptable as a compact first fusion layer, but it must not be described as feature-level clinical attention. If feature-level selectivity is needed later, tokenize clinical/biomarker features into multiple K/V tokens before cross-attention.

## Architecture

**Tabular + biomarker encoding:**

Generated OCTA biomarkers are appended to the tabular stream after fold-fitted normalization:

```
F_tabular = TabularEncoder(Concat([clinical_features, biomarkers])) ∈ ℝ^256
```

Current completed Phase 5 code can infer publication-table/image-derived biomarker columns and eye-align `_OD`/`_OS` columns. Before Phase 9, rebuild or verify Phase 5 biomarker specs/statistics against the Phase 8-enriched manifest so generated Phase 8 biomarkers are included with training-fold-only normalization.

**Residual tabular-conditioning block:**

```
Q = Linear(F_octa)          ∈ ℝ^(1 × 256)
K = Linear(F_tabular)       ∈ ℝ^(1 × 256)
V = Linear(F_tabular)       ∈ ℝ^(1 × 256)

# With one K/V token, softmax(QK^T / √256) is 1.0.
# Therefore this is residual tabular conditioning, not feature-level attention.
Attended = V
F_fused = F_octa + Attended             (residual connection)
F_fused = LayerNorm(F_fused)
```

Future upgrade path: replace `F_tabular` with per-feature/per-biomarker tokens and use true 4-head cross-attention across those tokens.

**Implemented generated mask branch fusion and projection to shared space:**

Generated masks are produced upstream by Phase 8. The classifier never assumes human-provided masks.

Current completed Phase 7 notebook code keeps the first-pass contract. It attends from one OCTA token to one tabular token, then concatenates attended OCTA/tabular features, mask features, and the tabular token again:

```
M = Stack([sup_vessel, deep_vessel, sup_faz, deep_faz, cc_flow])
F_mask = MaskEncoder(M)                                  ∈ ℝ^128

F_cross = LayerNorm(F_octa + Attended)                   ∈ ℝ^256
H = Linear(Concat([F_cross ; F_mask ; F_tabular]) 640 → 512)
H = GELU(H)
H = LayerNorm(H)
```

This is not the preferred final contract, but it is completed code through Phase 8D and should not be rerun just to change architecture.

**Future implementation note:** before Phase 9 classifier training, add forward-only compatibility code in Phase 9 or later that consumes existing Phase 4–8 artifacts and uses each modality once:

```
F_cond = LayerNorm(F_octa + Proj_tab(F_tabular))          ∈ ℝ^256
F_mask = MaskEncoder(M)                                  ∈ ℝ^128
H = Linear(Concat([F_cond ; F_mask]) 384 → 512)           ∈ ℝ^512
```

Do **not** patch, rerun, or redo completed Phase 7/8 cells solely to fix this contract mismatch. Do **not** concatenate `F_tabular` again after `F_cond`; tabular information has already entered through `Proj_tab(F_tabular)`.

Output: H ∈ ℝ^512 — Unified multimodal representation.

---

# PHASE 8 — SELF-GENERATED SEGMENTATION MASKS AND MASK ENCODER

The dataset is assumed to contain **no manual vessel, FAZ, or flow-deficit masks**. Therefore, segmentation in Version 1 is an automated weak-segmentation pipeline:

```text
Preprocessed OCTA image
  → classical pseudo-mask generation
  → optional segmentation-student refinement
  → mask QC + biomarker extraction
  → downstream classifier mask branch
```

Do not describe these as human ground-truth segmentations unless a manually annotated validation subset is later created.

## Step A — Classical pseudo-mask generation

Generate deterministic pseudo-labels for every fold using training-independent image-processing rules. These masks are reproducible weak labels, not learned from disease labels.

### SVC and DVC vessel masks

For Sup and Deep layers:

```text
percentile normalization
→ CLAHE
→ denoising / mild Gaussian smoothing
→ Frangi or Sato vesselness filtering
→ adaptive/local thresholding
→ morphological opening/closing
→ small-component removal
→ optional skeletonization for biomarkers
```

Outputs:

```text
sup_vessel_mask, deep_vessel_mask
```

### SVC and DVC FAZ masks

FAZ is detected as the central avascular low-flow region.

```text
vessel mask / low-flow map
→ central search window around image center
→ connected-component selection nearest center
→ contour smoothing / hole filling
→ plausibility QC by area, eccentricity, and centroid distance
```

Outputs:

```text
sup_faz_mask, deep_faz_mask
```

Skip FAZ generation for CC unless dataset-specific review shows CC FAZ-like annotations are meaningful.

### CC flow-deficit masks

For choriocapillaris:

```text
percentile normalization
→ CLAHE or local background correction
→ local thresholding (Phansalkar/Niblack/Sauvola-style)
→ dark-region detection
→ morphology cleanup
→ area filtering
```

Output:

```text
cc_flow_mask
```

## Step B — Classical-mask improvement gate before any student model

Do **not** train U-Net, Attention U-Net, or U-Net++ directly on poor classical pseudo-labels. A segmentation student can only imitate and regularize its training targets; it cannot recover missing vessels without human ground truth.

Before student refinement is allowed, classical masks must pass a documented review gate:

1. Generate Phase 8 masks and montage sheets.
2. Visually inspect random samples and all QC failures.
3. Confirm Sup/Deep vessel masks capture major vessels **and** capillary texture, not only sparse edge/large-vessel fragments.
4. Confirm FAZ masks are centered and plausible.
5. Confirm CC flow-deficit masks are not dominated by watermark/edge artifacts.
6. Tune classical vessel extraction if masks are too sparse:
    - lower vesselness or adaptive-threshold cutoff,
    - reduce small-object removal aggressiveness,
    - combine Frangi/Sato vesselness with adaptive intensity thresholding,
    - remove watermark/edge artifacts before QC/biomarkers.

Only after this gate is passed may a student be trained.

## Step C — Explicit student-refinement decision gate

A future agent must **not silently skip** student refinement just because it is described as optional. After improved classical masks are generated, make and document one of these explicit decisions in `PROGRESS.md`:

```text
Decision A — Use improved classical masks only
Allowed only if QC + montage review show masks are anatomically acceptable and stable enough.
Required note: why student refinement is unnecessary.

Decision B — Train U-Net++ student
Required if improved classical masks are anatomically plausible but still noisy, fragmented, threshold-unstable, or biomarker-unstable.
Required outputs: student checkpoints, student-vs-classical Dice/IoU, montage comparison, biomarker stability comparison.

Decision C — Do not train student yet; tune classical masks again
Required if improved classical masks are still anatomically poor/sparse/wrong. U-Net++ must not be trained on bad pseudo-labels.
```

Default expectation after the improvement pass: **evaluate U-Net++ student refinement unless Decision A is explicitly justified**. In other words, U-Net++ is optional only after a documented QC-based decision, not something to ignore.

## Step D — Segmentation student model

If Decision B is selected, train a segmentation student to imitate pseudo-labels and produce smoother standardized masks. This model fits **upstream of the disease classifier**, between preprocessing and classification.

Student architecture:

```text
Primary student to evaluate: U-Net++
Fallback/lighter variant if compute-limited: Attention U-Net
Encoder: EfficientNet-B0 or ConvNeXt-Nano, 1-channel OCTA input
Decoder: U-Net++ decoder, or Attention U-Net decoder for fallback
Output heads:
  Sup/Deep model: 2 channels = vessel + FAZ
  CC model: 1 channel = flow-deficit
Activation: sigmoid per channel
```

Training target: QC-passed/improved classical pseudo-masks from Step A/B, not raw failed masks.

Student loss:

```
L_student = α · L_Tversky(β=0.7) + (1-α) · L_Focal(γ=2) + λ_dice · L_Dice
```

Use confidence weighting where possible: downweight pseudo-label regions that fail QC or lie near unstable boundaries.

During downstream disease-classifier training, freeze the segmentation student by default:

```text
OCTA image → frozen student segmenter → generated masks → mask encoder + biomarker extractor
```

End-to-end fine-tuning of the student is not allowed in Version 1 unless a stability loss preserves mask quality; otherwise classifier gradients may distort segmentation.

## Mask QC

Reject or flag generated masks with implausible properties:

- Vessel density outside plausible range
- FAZ centroid too far from image center
- FAZ area too small/large
- CC flow-deficit mask nearly all black or all white
- Excessive tiny connected components

Generate visual montage sheets for random samples and all QC failures. A montage is a review contact sheet showing the original Sup/Deep/CC OCTA images beside the generated Sup vessel, Deep vessel, Sup FAZ, Deep FAZ, and CC flow masks for the same eye. Manually inspect a subset of generated masks and report accept/reject rate if used in a paper.

## Biomarker extraction

Extract quantitative features from generated masks:

- Vessel density
- Vessel skeleton density / vessel length density
- FAZ area
- FAZ perimeter
- FAZ circularity
- FAZ centroid distance from image center
- CC flow-deficit percentage
- CC flow-deficit count
- Mean / median flow-deficit area

These biomarkers are normalized using training-fold statistics and appended to the clinical/tabular encoder.

## Mask encoder for classifier

Input:

```text
[sup_vessel_mask, deep_vessel_mask, sup_faz_mask, deep_faz_mask, cc_flow_mask] ∈ ℝ^(5×224×224)
```

Architecture:

```
ConvNeXt-Nano (or 3-layer lightweight CNN)
GlobalAvgPool → 128-d
Projection: FC(128 → 128) → GELU → LayerNorm
```

Output: F_mask ∈ ℝ^128

Current local outputs include refreshed `classical_improved` masks for 476 samples with high automated QC pass rate (local copied summary: QC pass rate ≈ 0.9937). This satisfies the pseudo-label quality precondition for Decision B. The project decision is to proceed with **Decision B — train U-Net++ student**, because improved classical masks are QC-acceptable and U-Net++ is the selected refinement path. Phase 8 U-Net++ student training was run on the separate Windows GPU/training machine; this local checkout is not the runtime environment and does not need local `.pt` checkpoints unless explicitly requested.

Operational rule: the downstream classifier must use `mask_source = "unetpp_student"` for this project. The `classical_improved` masks are pseudo-label targets for training/evaluating the U-Net++ student, not the operational downstream mask source. This checkout is not the runtime environment, so local absence of `.pt` files is not a reason to fall back to classical masks. On the Windows GPU/training machine, keep the U-Net++ checkpoints in place and use the frozen student for downstream masks. If a future rerun shows anatomical degradation or unacceptable metrics, stop and explicitly revise the Phase 8 decision before Phase 9 rather than silently switching to `classical_improved`. Compare performance with and without generated masks, generated biomarkers, and student refinement in ablation (Phase 15).

---

# PHASE 9 — CLASSIFICATION HEAD WITH PROTOTYPE AUGMENTATION

## Phase 9 implementation preflight — do this first

Before implementing Phase 9, **do not rerun or redo completed Phase 1–8D cells**. Treat existing Phase 1–8D notebook outputs/artifacts as fixed inputs.

Check the existing Phase 7 output contract. If the notebook still uses the earlier one-token cross-attention/final-concat implementation, do **not** edit/rerun Phase 7. Instead, add Phase 9+ forward-only compatibility code/wrapper that produces the required unified representation:

```
F_cond = LayerNorm(F_octa + Proj_tab(F_tabular))   ∈ ℝ^256
F_mask = MaskEncoder(masks)                        ∈ ℝ^128
H = Project(Concat([F_cond, F_mask]) 384 → 512)    ∈ ℝ^512
```

Also verify Phase 8 gate status before training: Phase 9 training must use the documented Decision B state and the available Phase 8D history/metric artifacts. The local non-GPU checkout does not need `.pt` checkpoint files, because they remain on the Windows GPU/training machine. For this project, Phase 9+ must use `mask_source = "unetpp_student"`; do not implement a local fallback to `classical_improved` just because this review machine lacks checkpoints.

Phase 9 consumes the unified multimodal representation `H ∈ R^512`. In the full model, `H` must include OCTA features, the trained tabular encoder output `F_tabular`, and the mask encoder output `F_mask` from the selected Phase 8 `mask_source`. The Phase 9 implementation may materialize fold-specific `unetpp_student` mask and biomarker manifests by running frozen Phase 8D checkpoints in inference mode. This is not redoing Phase 8 training; it is the file-based bridge that lets Phase 9+ consume U-Net++ masks and generated biomarkers consistently.

## Standard head

```
FC(512 → 256) → GELU → Dropout(0.3)
FC(256 → 128) → GELU → Dropout(0.3)
FC(128 → N_classes)
```

## Prototype-augmented logits

For imbalanced disease cohorts, augment the standard linear classifier with prototype-based logits:

During training, maintain a running class prototype P_k ∈ ℝ^128 (exponential moving average of class-conditional mean embeddings in the penultimate layer).

Prototype state and scalar weights are part of the disease-classifier head and must be trained/updated in Phase 12 Stage 2 and Stage 3. Phase 9 implementation should initialize prototypes from class-conditional penultimate embeddings after the first classifier epoch, update prototypes by EMA only for classes present in the current batch, save/load prototype buffers in checkpoints, and include a no-prototype ablation.

At inference, compute prototype similarity:

```
proto_logit_k = -‖z - P_k‖² / τ_proto
```

Final logit = standard_logit + α · proto_logit

where α is a learned scalar weight, initialized conservatively, and τ_proto = 0.1. Calibrate final combined logits using the same validation/calibration path as the standard classifier.

This provides a non-parametric classification signal that generalises better for rare cohorts with few samples, complementing the standard linear head.

## Uncertainty quantification

Apply Monte Carlo Dropout at inference (30 forward passes with Dropout active).

Report per-sample:

- Mean prediction: P(class | x) = (1/30) Σ softmax(f_t(x))
- Epistemic uncertainty: σ²_k = Var_t[softmax(f_t(x))_k] per class
- Entropy: H = -Σ P_k log P_k (overall prediction uncertainty)

Flag predictions with entropy > threshold as "uncertain — recommend manual review."

## Post-training calibration

Apply temperature scaling on the validation fold:

Learn scalar T by minimising validation negative log-likelihood / cross-entropy of `softmax(logits / T)`.

After fitting T, report ECE and Brier score on validation/test predictions. Do not optimise ECE directly in the default post-hoc calibration path.

---

# PHASE 10 — LOSS FUNCTIONS

## Primary classification loss

**Default:** Weighted Cross-Entropy with label smoothing ε=0.1

Class weights: inverse class frequency (computed per training fold).

```
L_cls = -Σ_k w_k · y_k · log(p_k)   (weighted CE with smoothed targets)
```

**If severe imbalance (max_class_freq / min_class_freq > 10):** Switch to Focal Loss:

```
L_focal = -Σ_k (1 - p_k)^γ · log(p_k)    γ=2
```

Compare both in experiments. Report which performs better per dataset split.

**Class-balanced sampling:** Do not default to simultaneous inverse-frequency loss weighting and `WeightedRandomSampler`, because using both can overcorrect minority classes. Recommended default for Phase 9 is weighted CE/focal loss with ordinary shuffling. Add `WeightedRandomSampler` only as an ablation or if minority-class recall remains poor.

## Segmentation student loss

Segmentation is trained as a separate upstream stage against generated pseudo-labels, not against human masks.

```
L_student = α · L_Tversky(β=0.7) + (1-α) · L_Focal(γ=2) + λ_dice · L_Dice
```

Tversky with β=0.7 penalises false negatives more heavily — critical for sparse capillary, FAZ, and flow-deficit masks.

Do not include `L_student` in the main classifier objective by default. The segmentation student is frozen during disease-classifier training.

## Calibration objective

Default calibration is **post-hoc temperature scaling** on the validation fold after classifier training. Do not backpropagate ECE through the classifier in the default Version 1 run.

Optional experiment only:

```
L_cal = ECE(p, y)    # differentiable mini-batch estimate
L_total_with_cal = L_cls + λ₂ · L_cal
```

Use this train-time calibration variant only as an ablation/experiment and report it separately from the default model.

## Total disease-classifier loss

Default Version 1 classifier training:

```
L_total = L_cls
```

Backpropagate `L_total` through the complete disease classifier: OCTA encoders when unfrozen, projection heads, cross-layer attention, tabular encoder, mask encoder, multimodal fusion, prototype/logit parameters, and classification head. Exclude the frozen segmentation student unless the optional joint variant is explicitly enabled.

Optional joint segmentation experiment only:

```
L_total_joint = L_cls + λ₁ · L_student
```

Use joint training only if mask-quality monitoring confirms that classifier gradients do not degrade segmentation masks. For the optional joint variant only, start with λ₁=0.1–0.5 and tune via validation while monitoring mask QC.

Label smoothing: applies to L_cls only.

---

# PHASE 11 — DATA AUGMENTATION

Apply augmentations to OCTA images during training only. Never augment validation or test images.

## OCTA-specific augmentation pipeline

```python
transforms.Compose([
    transforms.RandomHorizontalFlip(p=0.5),
    transforms.RandomVerticalFlip(p=0.3),
    transforms.RandomRotation(degrees=15),
    transforms.RandomAffine(
        degrees=0,
        translate=(0.05, 0.05),
        scale=(0.95, 1.05),
        shear=5
    ),
    transforms.ColorJitter(brightness=0.1, contrast=0.1),
    AddGaussianNoise(sigma=0.01, p=0.2),
    transforms.ToTensor(),
    transforms.Normalize(mean=fold_mean, std=fold_std)
])
```

Do NOT apply:

- Aggressive color jitter (destroys vessel contrast)
- Random erasing or cutout (removes vascular structure)
- Large-scale elastic deformation (distorts FAZ morphology)
- Random grayscale (images are already grayscale)

**For clinical features:** Add Gaussian noise (σ=0.05) to continuous features during training with probability 0.3. Randomly zero out one observed clinical feature per sample with probability 0.1 (simulates missing data at training time).

Do not apply clinical-feature augmentation to validation/test folds. Do not corrupt generated biomarkers by default; only normalize them with training-fold statistics and use biomarker missing tokens/QC flags for missing or failed values.

**Apply same augmentation to all three OCTA layers simultaneously** (same random seed per sample) to preserve cross-layer spatial correspondence.

---

# PHASE 12 — TRAINING STRATEGY

## Downstream disease-classifier scope

The downstream disease classifier includes:

- Sup/Deep/CC OCTA backbones
- Phase 4 projection heads
- Phase 5 tabular encoder for clinical features plus Phase 8 generated biomarkers
- Phase 6 cross-layer attention
- Phase 7 multimodal fusion
- Phase 7 mask encoder using masks from the selected Phase 8 `mask_source`
- Phase 9 prototype/logit parameters and classification head

The segmentation student is upstream of the classifier and remains frozen during disease-classifier training unless the optional joint variant is explicitly enabled.

## 3-Stage Training Schedule

**Stage 1: OCTA-SimCLR pretraining** (unlabeled)

- Epochs: 100
- Use the already-completed Phase 3 artifacts when available; do not rerun Phase 3 solely because this plan was clarified later.
- Historical/current successful run settings: LR=0.03 cosine decay, AMP disabled after failed preflight, gradient clipping max norm 1.0
- All three encoders independently

**Stage 2: Frozen OCTA backbones, train downstream classifier modules**

- Epochs: 15
- LR: 3e-4
- Train projection heads, tabular encoder, mask encoder, cross-layer attention, multimodal fusion, prototype/logit parameters, and classification head
- Freeze Sup/Deep/CC OCTA backbone weights loaded from Stage 1
- Freeze any segmentation student selected in Phase 8D
- Purpose: initialise downstream heads before catastrophic forgetting risk

**Stage 3: End-to-end disease-classifier fine-tuning**

- Epochs: 100
- Backbone LR: 3e-5 (10× smaller than head LR)
- Head LR: 3e-4
- Train Sup/Deep/CC OCTA backbones, projection heads, tabular encoder, mask encoder, cross-layer attention, multimodal fusion, prototype/logit parameters, and classification head
- Keep any Phase 8D segmentation student frozen unless the optional joint variant is explicitly enabled and mask-quality monitoring is active
- Linear warmup first 5 epochs
- Cosine annealing (T_0=20, T_mult=2)
- Early stopping: patience=15 on validation macro-F1

## Optimizer

AdamW

- β₁=0.9, β₂=0.999
- Weight decay: 1e-2

## Batch size

32 (or 16 with gradient accumulation ×2 = effective 32 if GPU limited)

Mixed precision: enabled (bfloat16 or float16 with GradScaler)

## Cross-validation

5-fold stratified cross-validation

- Preserve patient grouping: all eyes from one patient always in same fold
- Current completed artifacts stratify by disease cohort label, patient age decade, and sex when available; local copied audit shows zero patient leakage. Keep these fixed folds for all Phase 9+ work to avoid invalidating Phase 2/3/8 artifacts.
- Report: mean ± std across 5 folds for all metrics
- Final model: ensemble of 5 fold models (average softmax outputs)

For publication-strength Phase 9+ training/evaluation, treat each fixed fold as the outer held-out evaluation fold. From the remaining four folds, create a patient-grouped stratified inner validation/calibration split, recommended 80/20 by patient within the outer-training patients, for early stopping and temperature scaling. Do not use the outer held-out fold both for model selection/calibration and final reported metrics.

## Reproducibility

Set all random seeds (Python, NumPy, PyTorch, CUDA).
Log all hyperparameters, dataset splits, and random states.
Use deterministic PyTorch operations where available.

---

# PHASE 13 — EXPLAINABILITY

## Grad-CAM per layer

For each encoder (Sup, Deep, CC), compute Grad-CAM with respect to the final convolutional feature map before global average pooling. Grad-CAM and attention analyses are explanatory aids and should be reported alongside ablation evidence, not as definitive causal proof.

Generate per-patient, per-class heatmaps:

- Sup heatmap: which SVC regions drove the classification decision
- Deep heatmap: which DVC regions drove the classification decision
- CC heatmap: which CC regions drove the classification decision

Overlay on original OCTA en-face images. Use clinically appropriate colormap (viridis or hot).

Present side-by-side per-layer heatmaps for example patients in each cohort.

## Attention rollout — layer dominance analysis

Treat attention-derived layer scores as a layer-dominance proxy, not as causal proof of model reasoning. Support any interpretation with layer ablations and consistency with Grad-CAM findings.

From the cross-layer attention module (Phase 6), extract the attention weight matrix A ∈ ℝ^(3×3) per sample.

Compute per-sample layer-dominance score:

```
dominance_k = (1/N) Σ_i A_{ik}    for k ∈ {Sup, Deep, CC}
```

Aggregate per cohort: which retinal layer dominates attention for each disease class?

This can produce a clinically interpretable hypothesis such as: "Disease X appears to rely more on CC-associated signals; Disease Y appears to rely more on SVC-associated signals." Avoid claiming attention weights alone prove causal anatomical drivers.

## SHAP values for clinical features

Apply SHAP to the trained tabular encoder inputs. Report clinical-feature attributions separately from generated-biomarker attributions.

Generate per-patient:

- Waterfall plots showing clinical feature contribution to prediction
- Force plots per patient
- Population beeswarm plots per cohort

Identify which clinical features are most discriminative per disease class.

## Integrated analysis

Correlate Grad-CAM hotspot regions with layer-dominance scores — do the spatial findings agree with the attention-level finding about which layer dominates?

---

# PHASE 14 — EVALUATION METRICS

All metrics reported with 95% bootstrap confidence intervals (n=1000 bootstrap samples, stratified). For publication-facing inference, bootstrap at the patient level or use patient-cluster bootstrap rather than treating bilateral eyes from the same patient as independent.

For cross-validation, compute metrics from out-of-fold held-out predictions only. Do not report training-fold metrics as final performance. Because OD/OS eyes can both come from one patient, report eye-level metrics for sample-level performance and patient-level or patient-clustered metrics for publication/statistical inference. Recommended patient-level prediction: average OD/OS softmax probabilities per patient before metric computation.

## Primary metrics (classification)

- Macro-averaged ROC-AUC (one-vs-rest)
- Macro-F1
- Balanced accuracy (accounts for class imbalance)
- Per-class sensitivity and specificity

## Secondary metrics

- Weighted F1
- PR-AUC (Macro) — particularly important when classes are imbalanced
- Per-class precision, recall, F1
- Confusion matrix (normalised by true class count)
- Top-2 accuracy (whether correct class is in top-2 predictions)

## Calibration

- Expected Calibration Error (ECE, 10 bins) — primary calibration metric
- Brier score
- Reliability diagram per class

## Uncertainty evaluation

- Mean prediction entropy vs. classification accuracy (are uncertain predictions also less accurate?)
- Coverage analysis: percentage of correct predictions in the top-30% confidence bin

## Statistical comparison

For each baseline model vs. proposed model:

- McNemar's test on binary correct/incorrect vectors across all test samples
- DeLong test for pairwise AUC comparison
- Bonferroni correction for multiple comparisons
- Report p-values and effect sizes (Cohen's h for AUC comparisons)

---

# PHASE 15 — ABLATION STUDIES

Retrain from scratch with identical seeds for each ablation. Report mean ± std over 5 folds.

Unless a row explicitly removes a component, train all remaining modules with the same Phase 12 schedule as the full model. Every ablation must record the selected Phase 8 `mask_source` and whether generated biomarkers are included.

| Configuration              | Modification                                                                                                                                                                                                                                  |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Full model                 | Train OCTA encoders, projection heads, tabular encoder with clinical + generated biomarkers, mask encoder, cross-layer attention, multimodal fusion, prototype head, MC-Dropout, and temperature calibration using the selected `mask_source` |
| −SimCLR pretraining        | ImageNet init only, no OCTA pretraining                                                                                                                                                                                                       |
| −Cross-layer attention     | Concatenate Fs, Fd, Fc directly → project to 256-d                                                                                                                                                                                            |
| −Layer-identity embeddings | Remove e_sup, e_deep, e_cc from cross-layer attention                                                                                                                                                                                         |
| −Tabular branch            | Remove `F_tabular` entirely: no clinical features and no generated biomarker vector; classify on `F_octa + F_mask` only                                                                                                                       |
| −Mask branch               | Remove `F_mask`; keep tabular encoder with clinical features and generated biomarkers                                                                                                                                                         |
| −Mask biomarkers           | Keep clinical tabular branch, but remove generated OCTA biomarker vector from tabular encoder                                                                                                                                                 |
| Classical masks only       | Use improved classical pseudo-masks for `F_mask` and biomarkers; bypass segmentation student                                                                                                                                                  |
| Student masks only         | Use frozen U-Net++/Attention U-Net student outputs for `F_mask` and biomarkers, allowed only after improved classical pseudo-labels pass QC                                                                                                   |
| −Prototype head            | Standard linear head only                                                                                                                                                                                                                     |
| −Uncertainty (MC-Dropout)  | Single deterministic forward pass                                                                                                                                                                                                             |
| −Temperature calibration   | Uncalibrated softmax probabilities                                                                                                                                                                                                            |
| −Label smoothing           | Hard one-hot targets                                                                                                                                                                                                                          |
| −Class-balanced sampling   | Uniform sampling only                                                                                                                                                                                                                         |
| Sup only                   | Single encoder; remove Deep and CC                                                                                                                                                                                                            |
| Deep only                  | Single encoder; remove Sup and CC                                                                                                                                                                                                             |
| CC only                    | Single encoder; remove Sup and Deep                                                                                                                                                                                                           |
| Sup + Deep                 | Two encoders; remove CC                                                                                                                                                                                                                       |
| Sup + Deep + CC            | Three encoders; no tabular or mask branch                                                                                                                                                                                                     |
| Images + Tabular           | Sup + Deep + CC + trained tabular encoder with clinical + generated biomarkers; no mask encoder                                                                                                                                               |
| Images + Tabular + Masks   | Full model confirmation: images + trained tabular encoder + trained mask encoder using selected `mask_source`                                                                                                                                 |

Analyze and discuss the marginal contribution of each component.

---

# PHASE 16 — BASELINE COMPARISON

Implement and compare against the following baselines. Use identical preprocessing, augmentation, and cross-validation splits.

Clinical-only baselines use Phase 2 clinical features only; they do not receive generated OCTA biomarkers unless explicitly named as a biomarker baseline. CNN baselines use OCTA images only and do not receive clinical features, masks, or generated biomarkers.

| Baseline                      | Type              | Input                                                                                            |
| ----------------------------- | ----------------- | ------------------------------------------------------------------------------------------------ |
| Logistic Regression           | Classical         | Clinical features only                                                                           |
| Random Forest                 | Ensemble          | Clinical features only                                                                           |
| XGBoost                       | Gradient boosting | Clinical features only                                                                           |
| ResNet50                      | CNN               | Sup image only                                                                                   |
| ResNet50 (3-channel)          | CNN               | Sup+Deep+CC stacked as RGB                                                                       |
| EfficientNet-B0               | CNN               | Sup image only                                                                                   |
| Vision Transformer (ViT-S/16) | Transformer       | Sup image only                                                                                   |
| ConvNeXt-Tiny (single)        | CNN               | Sup image only                                                                                   |
| Proposed — full               | Multimodal        | Sup + Deep + CC + clinical features + generated masks and biomarkers from selected `mask_source` |

For CNN baselines: use identical splits, preprocessing, augmentations, optimizer family, epoch budget, and early stopping. Train only the baseline backbone and classifier head end-to-end from ImageNet weights, with no SimCLR or multimodal modules.

---

# PHASE 17 — STATISTICAL VALIDATION

## Group-level analysis

For each biomarker or clinical feature:

- Mann-Whitney U test: cohort A vs. cohort B (pairwise, all pairs)
- One-way ANOVA: across all cohorts simultaneously
- Kruskal-Wallis test: non-parametric alternative when normality is violated

For generated biomarkers, state the Phase 8 `mask_source` used to extract them. Keep clinical-feature statistics separate from generated-biomarker statistics.

## Fairness analysis

Evaluate model performance stratified by:

- Sex (male vs. female)
- Age quartile (<50, 50–60, 60–70, >70)
- Presence of diabetes
- Bilateral vs. unilateral eye availability

Report whether model performance degrades for underrepresented subgroups.

## Multivariate analysis

Logistic regression (one-vs-rest) using clinical features only: use as a clinical baseline and to quantify the added value of imaging features.

Partial correlation: predicted disease probability vs. each clinical feature, controlling for age and sex.

---

# PHASE 18 — NOVEL RESEARCH CONTRIBUTIONS

Identify and articulate the following contributions for publication:

1. **Bilateral natural positives for OCTA-SimCLR:** First contrastive pretraining strategy that exploits OD/OS pairs as positive samples, leveraging the unique bilateral structure of ophthalmic datasets to avoid label requirements.

2. **Layer-identity tokens in cross-layer attention:** Explicit layer-identity embeddings ensure the transformer-based cross-layer attention module can distinguish which token corresponds to which retinal depth — a critical but commonly omitted detail.

3. **Learnable mask-token imputation for clinical data:** Replaces statistical imputation (mean/mode) with a learned representation of "this feature was not measured," preserving the model's ability to reason about data absence as a distinct clinical state.

4. **Prototype-augmented classification for rare cohorts:** Hybrid parametric + non-parametric classification head that improves generalisation for minority disease classes without requiring oversampling or additional annotations.

5. **Layer-dominance analysis as a clinical explainability tool:** Systematic attribution of predictions to specific retinal layers via attention weight analysis, producing cohort-level findings interpretable by ophthalmologists and cardiologists.

6. **Uncertainty-aware disease classification:** Integration of MC-Dropout uncertainty quantification and ECE-calibrated probabilities, enabling clinical triage — uncertain predictions can be flagged for expert review rather than acted upon directly.

7. **Automated weak OCTA segmentation for mask-guided classification:** Generates vessel, FAZ, and CC flow-deficit pseudo-masks without manual annotations, optionally refines them with a frozen segmentation student, and converts them into interpretable biomarkers for disease classification.

---

# PHASE 19 — DELIVERABLES

Provide all of the following:

1. **Dataset loader** — complete PyTorch Dataset class with patient-level stratified splitting, bilateral eye handling, mask-token imputation, and data integrity checks
2. **Training pipeline** — staged schedule (SimCLR → pseudo-mask generation/student decision → train classifier heads including tabular/mask/fusion modules with OCTA backbones frozen → fine-tune full disease classifier with segmentation student frozen), logging, checkpointing
3. **Model implementation** — all modules: encoders, projection heads, cross-layer attention, fusion, classification head, segmentation student, mask encoder
4. **Validation pipeline** — 5-fold CV loop with metric accumulation, bootstrap CI computation, calibration step
5. **Explainability module** — Grad-CAM (per layer), attention rollout (layer dominance), SHAP (clinical features), generated-mask visual overlays
6. **Experiment manager** — notebook-contained configuration dictionaries/cells, all hyperparameters logged via CSV/JSON plus optional wandb or MLflow. Do not create external YAML/config files unless the project instruction is changed.
7. **Reproducibility config** — notebook-contained fixed seeds, deterministic ops, dataset split serialisation
8. **Full PyTorch implementation inside `main.ipynb`** — modular notebook classes/functions with notebook-local validation/smoke tests for critical components. Do not create additional Python files unless the project instruction is changed.
9. **Results tables** — all metrics, all ablations, all baselines, with bootstrap CIs
10. **Publication-ready methodology section** — IEEE TPAMI / Nature Biomedical Engineering style
11. **Segmentation artifacts** — pseudo-mask generator, optional segmentation-student checkpoints, mask QC reports, visual montage sheets, and biomarker CSVs

---

# PHASE 20 — V2 ROADMAP (ANGIOCUBE INTEGRATION)

Version 2 will extend this system with 3D angiocube features.

Planned additions:

- Replace 2D ConvNeXt-Tiny with shared 3D Swin-T backbone + layer-specific adapter tokens (as designed in the CVD architecture)
- Add Retinal-MAE pretraining with layer-masked reconstruction
- Integrate volumetric biomarker extraction (VAD, VLD, VFD per ETDRS zone)
- Full multi-task learning: classification + biomarker regression + segmentation supervision
- PCGrad dynamic loss weighting

Version 2 target: replace the current cross-layer attention on 3 pooled vectors with full cross-layer attention on 3D spatial feature sequences.

This Version 1 system is explicitly designed to be compatible with V2 extension — the modular architecture allows drop-in replacement of 2D encoders with 3D equivalents.
