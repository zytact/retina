# SEGMENTATION ARCHITECTURE COMPARISON PLAN

## Purpose

Compare six segmentation architectures as the segmentation component of the same complete multimodal RASTA pipeline. This is not a segmentation-model ensemble experiment. Each architecture defines one independent end-to-end multimodal variant:

```text
Sup/Deep/CC OCTA
  -> three fold-specific OCTA-SimCLR ConvNeXt-Tiny encoders
  -> per-layer projection heads
  -> cross-layer self-attention
  -> F_octa

Clinical features
  + segmentation-derived biomarkers
  -> tabular MLP with learned missing-value handling
  -> F_tabular

Architecture-specific segmentation masks
  -> five-channel mask encoder
  -> F_mask

F_octa + F_tabular + F_mask
  -> compatibility fusion
  -> prototype-augmented eight-class classifier
  -> temperature calibration and uncertainty estimation
```

The scientific question is:

> When every other component and training decision is controlled, which segmentation architecture produces the most useful masks and biomarkers for the complete multimodal cohort-classification system?

The comparison must distinguish two outcomes:

1. How well each segmentation student agrees with the classical improved pseudo-labels.
2. How well the complete multimodal classifier performs when driven by that student's masks and biomarkers.

Pseudo-label agreement is not equivalent to anatomical accuracy. The dataset is assumed to have no human segmentation ground truth, so segmentation metrics must be described as agreement with weak pseudo-labels.

## Scope and boundaries

- Use all eight cohorts with stable zero-based labels in folder-number order:
  - `1_RETINORM` -> 0
  - `2_ORNET` -> 1
  - `3_FAMILIPO` -> 2
  - `4_MRCC` -> 3
  - `5_GIANTS` -> 4
  - `6_AWARD` -> 5
  - `7_EVIRED` -> 6
  - `8_ORNET_TEMOINS` -> 7
- Use 2D Sup, Deep, and CC OCTA images only.
- Preserve the fixed patient-grouped Phase 2 folds.
- Use only matching expanded-run Phase 3 OCTA-SimCLR artifacts.
- Keep all historical four-class and existing U-Net++ artifacts intact.
- Write every comparison artifact to new architecture-specific output directories.
- Implement the comparison in `segmentation_comparison_8class.ipynb` and leave `main.ipynb` unchanged.
- Copy the Phase 1-3 source cells exactly from `main.ipynb`, including their existing expanded eight-class output paths and Phase 3 reuse behavior.
- Adapt only Phase 4 onward for the segmentation comparison and its isolated output tree.
- Do not create a separate Python module.
- This plan ends at Phase 14.

## Compared multimodal variants

| ID | Segmentation architecture | Comparison role |
|---|---|---|
| `unetpp` | U-Net++ | Existing nested-skip reference architecture |
| `unet` | U-Net | Classical biomedical segmentation baseline |
| `attention_unet` | Attention U-Net | Attention-gated U-Net family model |
| `deeplabv3plus` | DeepLabV3+ | Atrous multiscale-context model |
| `segformer_b0` | SegFormer-B0 | Compact transformer segmentation model |
| `imn` | Image Magnification Network | OCTA-specific thin-vessel model |

The six complete variants are:

```text
OCTA pipeline + U-Net++          masks/biomarkers + tabular pipeline + classifier
OCTA pipeline + U-Net            masks/biomarkers + tabular pipeline + classifier
OCTA pipeline + Attention U-Net  masks/biomarkers + tabular pipeline + classifier
OCTA pipeline + DeepLabV3+       masks/biomarkers + tabular pipeline + classifier
OCTA pipeline + SegFormer-B0     masks/biomarkers + tabular pipeline + classifier
OCTA pipeline + IMN              masks/biomarkers + tabular pipeline + classifier
```

Do not average, vote, cascade, or otherwise combine predictions from different segmentation architectures in this comparison.

## Experimental invariants

The segmentation architecture is the only intended experimental variable. Hold the following fixed:

- Eight-cohort label schema
- Patient-grouped outer folds and inner validation splits
- Phase 1 manifest rules
- Phase 2 image preprocessing, clinical preprocessing, and fold statistics
- Phase 3 fold-specific OCTA-SimCLR backbone checkpoints
- Phase 4 image encoders and projection heads
- Phase 5 tabular MLP architecture and learned missing-value handling
- Phase 6 cross-layer attention
- Phase 7/9 compatibility fusion contract
- Five-mask channel order
- Mask encoder architecture
- Classical improved pseudo-label targets and QC rules
- Segmentation training loss, optimizer, augmentation, epoch budget, checkpoint selection, and threshold-selection policy
- Classifier head, prototype mechanism, losses, augmentation, and staged training schedule
- Calibration and MC-dropout procedure
- Evaluation metrics, confidence intervals, seeds, and reporting format

Model-specific changes are allowed only where required by the architecture, such as a transformer patch embedding or IMN magnification path. Record parameter count, FLOPs at 224 x 224, peak GPU memory, training time, and inference time so differences in capacity and efficiency remain visible.

## Output isolation

Use a new root:

```text
outputs/segmentation_comparison_8class/
```

Recommended layout:

```text
outputs/segmentation_comparison_8class/
  shared/
    configuration/
    fold_audits/
    pseudo_label_audits/
  unetpp/
    phase8_segmentation/
    phase9_masks_biomarkers/
    phase12_training/
    phase13_explainability/
    phase14_evaluation/
  unet/
  attention_unet/
  deeplabv3plus/
  segformer_b0/
  imn/
  comparison/
```

Each model directory must contain a resolved configuration file with the model ID, architecture settings, seed, fold, input/output contract, pseudo-label source, checkpoint paths, and dependency versions. Never infer the active model from a directory name alone.

# PHASE 1 - EXPANDED DATASET AUDIT

Reuse the expanded eight-cohort Phase 1 manifest when it is complete and valid. Do not rerun analysis merely to start this comparison.

Before proceeding, audit:

- All eight cohort folders are represented.
- Labels are exactly 0-7 in folder-number order.
- Patient and eye identifiers are stable.
- Sup, Deep, and CC paths resolve on the Windows training machine.
- Duplicate and bilateral-eye handling remains unchanged.
- The comparison sample count and class distribution are saved.

Write comparison-specific audit summaries under `shared/fold_audits/` without modifying historical Phase 1 outputs.

# PHASE 2 - PREPROCESSING AND FIXED SPLITS

Reuse the expanded-run preprocessing artifacts and fixed patient-grouped five-fold assignments.

Required rules:

- Every eye from one patient remains in the same outer fold.
- The same outer folds are used by all six segmentation and classifier variants.
- Create one deterministic patient-grouped inner validation split from the four outer-training folds for each outer fold.
- Persist the inner split once and reuse it for every architecture.
- Fit image, clinical, and biomarker normalization statistics on outer-training data only, with inner-validation handling kept leakage-safe.
- Validation and outer-test images receive no random augmentation.

The expanded run currently stops at Phase 3. Complete the Phase 4-14 eight-class integration as part of this comparison rather than reusing incompatible four-class downstream artifacts.

# PHASE 3 - OCTA-SIMCLR BACKBONE REUSE

Reuse the completed expanded eight-class, 120-epoch, fold-training-only OCTA-SimCLR artifacts for Sup, Deep, and CC.

For every outer fold, require three matching backbones:

```text
fold k Sup ConvNeXt-Tiny
fold k Deep ConvNeXt-Tiny
fold k CC ConvNeXt-Tiny
```

Do not retrain SimCLR separately for each segmentation architecture. All six variants for a fold must load the same three Phase 3 backbone states. Reject historical four-class checkpoints and mismatched fold/layer metadata.

# PHASE 4 - IMAGE ENCODERS AND PROJECTION HEADS

Use the same three independent ConvNeXt-Tiny image encoders for all six variants. Initialize them from the matching fold-specific Phase 3 weights.

For each layer:

```text
ConvNeXt-Tiny final feature map
  -> global average pooling: 768
  -> FC 768 -> 512 -> GELU -> LayerNorm
  -> FC 512 -> 256 -> LayerNorm
```

Retain the final spatial feature map for Phase 13 Grad-CAM. Projection-head initialization must be seed-controlled and matched across variants wherever tensor shapes are identical.

# PHASE 5 - CLINICAL AND BIOMARKER TABULAR ENCODER

The tabular branch is not clinical data alone. Its input is:

```text
D_tab = clinical features + architecture-specific generated biomarkers + QC indicators
```

For each segmentation architecture and outer fold:

1. Materialize that architecture's masks without using outer-test labels.
2. Extract biomarkers from those masks.
3. Fit biomarker normalization on outer-training patients only.
4. Build an architecture-specific tabular schema with identical biomarker names and ordering.
5. Use learned missing-value tokens for absent clinical values and invalid biomarkers.

Keep the tabular MLP fixed:

```text
LayerNorm(D_tab)
FC(D_tab -> 128) -> GELU -> Dropout(0.3)
FC(128 -> 256) -> GELU -> LayerNorm -> Dropout(0.2)
```

Output:

```text
F_tabular in R^256
```

The numerical biomarker values may differ by segmentation architecture. The clinical columns, biomarker definitions, column ordering, missingness rules, and encoder architecture must not differ.

# PHASE 6 - CROSS-LAYER OCTA ATTENTION

Keep the existing layer-aware attention fixed for every variant:

```text
Sup token + Sup identity embedding
Deep token + Deep identity embedding
CC token + CC identity embedding
  -> 8-head self-attention
  -> 256 -> 1024 -> 256 feed-forward block
  -> mean pooling
  -> F_octa in R^256
```

Do not alter attention heads, identity embeddings, pooling, dropout, or dimensionality based on segmentation architecture.

# PHASE 7 - CONTROLLED MULTIMODAL FUSION

Use the forward-only compatibility contract that consumes each modality once:

```text
F_cond = LayerNorm(F_octa + Proj_tab(F_tabular))       in R^256

M = Stack([
  sup_vessel,
  deep_vessel,
  sup_faz,
  deep_faz,
  cc_flow,
])                                                     in R^(5 x 224 x 224)

F_mask = SharedMaskEncoder(M)                          in R^128
H = Project(Concat([F_cond, F_mask]) 384 -> 512)       in R^512
```

Do not use the older 640-dimensional fusion path that concatenates `F_tabular` a second time. Use one shared mask-encoder architecture across all six variants. The mask encoder is trained as part of each independent classifier variant, but its architecture and initialization seed policy remain fixed.

# PHASE 8 - SEGMENTATION ARCHITECTURE COMPARISON

## 8.1 Weak-label source

Use improved classical masks as training targets:

- Sup vessel
- Deep vessel
- Sup FAZ
- Deep FAZ
- CC flow deficit

Generate or extend these pseudo-labels for all eight cohorts before training comparison students. Apply the same QC gate to every architecture. Failed pseudo-label samples may be excluded from segmentation loss, but the exclusion list must be identical across architectures and reported by cohort, layer, fold, and target.

These masks remain weak pseudo-labels. Do not call them manual annotations or ground truth.

## 8.2 Common student task contract

Train two student networks per architecture and outer fold:

```text
Sup/Deep student:
  input: one 1-channel Sup or Deep OCTA image
  output channel 0: vessel logits
  output channel 1: FAZ logits

CC student:
  input: one 1-channel CC OCTA image
  output channel 0: flow-deficit logits
```

All outputs must be full-resolution 224 x 224 logits. Apply sigmoid only for probability generation or metric calculation, not inside the training loss implementation.

The Sup/Deep dataset must be deterministic during validation and testing. Expand every eligible eye into one explicit Sup sample and one explicit Deep sample. Do not randomly choose Sup or Deep in `__getitem__` for validation or outer testing. Training may shuffle the explicit samples, but it must not randomly omit one layer per eye per epoch.

## 8.3 Architecture definitions

### U-Net++

- Preserve nested dense skip pathways.
- Use the same compact depth and base channel width as the existing implementation unless a comparison-wide memory constraint requires a documented adjustment.
- Disable deep supervision by default unless it is enabled for all architectures through an equivalent auxiliary-loss policy.

### U-Net

- Use a symmetric five-level encoder-decoder.
- Use the same base width and convolution block style as U-Net++ where practical.
- Use concatenated encoder skip connections.

### Attention U-Net

- Start from the controlled U-Net definition.
- Add attention gates to encoder-decoder skip connections.
- Keep depth and base width aligned with U-Net.

### DeepLabV3+

- Use a compact documented encoder and atrous spatial pyramid pooling.
- Use the decoder path to restore 224 x 224 output and refine boundaries.
- Record output stride and atrous rates in the resolved configuration.

### SegFormer-B0

- Use the MiT-B0 hierarchical transformer encoder and lightweight MLP decoder.
- Adapt the input stem explicitly for one-channel OCTA.
- Return full-resolution logits by deterministic interpolation.

### Image Magnification Network

- Follow the OCTA-specific magnification-encoding and reduction-decoding design.
- Preserve the full-resolution thin-vessel output contract.
- Adapt its final head to two channels for Sup/Deep and one channel for CC.

## 8.4 Initialization policy

Choose and lock one initialization policy before launching any comparison run.

Primary policy:

- Use the canonical compact architecture.
- Use publicly documented pretrained encoder weights when the architecture has a standard pretrained encoder.
- Adapt RGB input stems to one channel by averaging pretrained RGB weights.
- Record whether each component is pretrained.
- Do not give one model access to OCTA images or labels unavailable to another.

Because canonical pretraining availability differs across architectures, this comparison estimates the performance of each practical architecture configuration, not a parameter-matched architecture-only causal effect. Report parameter count and initialization source prominently.

## 8.5 Shared segmentation training protocol

Use the same protocol for all architectures:

```text
loss = 0.5 * Tversky(beta=0.7)
     + 0.5 * focal BCE(gamma=2)
     + 0.25 * soft Dice

optimizer: AdamW
initial LR: 3e-4
weight decay: 1e-4
maximum epochs: 60
batch size: 16, or one fixed effective batch size using accumulation
checkpoint selection: highest inner-validation macro channel Dice
```

Use synchronized, conservative geometric and intensity augmentation. Masks receive only the matching geometric transform with nearest-neighbor interpolation.

Select probability thresholds per output channel using inner-validation predictions only. Lock those thresholds before outer-test inference. Store both threshold-free probability maps and thresholded masks. A common fixed threshold of 0.5 may be reported as a secondary standardized result, but must not replace the locked validation-selected operational threshold.

Use identical seed lists. The minimum primary run is one locked seed per fold for all six architectures. If multiple seeds are used, use the same seeds for every model and treat seed as a paired repeated factor.

## 8.6 Segmentation artifacts

For each architecture, fold, and task, save:

- `last.pt` and `best.pt`
- Resumable optimizer/scaler state
- Epoch history CSV
- Resolved configuration JSON
- Inner-validation threshold JSON
- Outer-test probability maps and thresholded masks
- Per-channel metrics and confusion counts
- QC report
- Training and inference timing
- Parameter/FLOP/memory summary
- Comparison montages using the same patient/eye sample list across architectures

Report vessel, FAZ, and CC results separately. Do not hide target-specific failures inside one macro average.

# PHASE 9 - ARCHITECTURE-SPECIFIC MASK MATERIALIZATION AND CLASSIFIER

For every architecture and outer fold:

1. Load that fold's selected Sup/Deep and CC student checkpoints.
2. Run the frozen students in evaluation mode.
3. Materialize five masks per eye using the locked validation-selected thresholds.
4. Run identical mask QC.
5. Extract the common biomarker set.
6. Write an architecture-specific manifest.
7. Refuse to continue if the manifest's `segmentation_model_id`, fold, checkpoint hashes, or mask paths do not match the requested variant.

The operational `mask_source` must name the architecture explicitly, for example:

```text
unetpp_student
unet_student
attention_unet_student
deeplabv3plus_student
segformer_b0_student
imn_student
```

Use the same Phase 9 classifier for all variants:

```text
H: 512
FC 512 -> 256 -> GELU -> Dropout(0.3)
FC 256 -> 128 -> GELU -> Dropout(0.3)
FC 128 -> 8
```

Maintain EMA class prototypes in the 128-dimensional penultimate space. Combine standard logits with distance-based prototype logits using the same learned non-negative weighting and temperature for every variant.

No classifier variant may silently fall back to classical masks or U-Net++ masks when its requested architecture-specific masks are missing.

# PHASE 10 - CONTROLLED LOSSES

## Segmentation loss

Use the shared Phase 8 student loss for all six architectures. Keep the segmentation student frozen during classifier training.

## Classification loss

Use the same fold-derived classification loss for every multimodal variant:

- Weighted cross-entropy with label smoothing 0.1 by default.
- Focal loss only when the same predeclared class-imbalance rule selects it.
- Do not combine inverse-frequency loss weights with a weighted sampler by default.

The default classifier objective is:

```text
L_total = L_classification
```

Do not jointly fine-tune any segmentation architecture with disease-label gradients in the primary comparison. Otherwise, the comparison would mix segmentation architecture with different degrees of disease-label adaptation.

# PHASE 11 - SYNCHRONIZED AUGMENTATION

Use one shared augmentation implementation and configuration across all variants.

During segmentation training:

- Apply identical geometric transforms to the OCTA image and pseudo-mask.
- Use nearest-neighbor interpolation for masks.
- Apply mild intensity transforms to images only.

During multimodal classifier training:

- Apply the same random geometric transform to Sup, Deep, CC, and all five materialized masks.
- Apply brightness, contrast, and noise only to classifier OCTA inputs.
- Do not corrupt masks or generated biomarkers with intensity augmentation.
- Apply the existing training-only clinical noise and observed-feature masking.
- Do not augment inner-validation or outer-test samples.

Use an architecture-independent augmentation seed derived from patient/eye, fold, epoch, and global seed so paired variants see equivalent transformations.

# PHASE 12 - MULTIMODAL TRAINING FOR EACH SEGMENTATION VARIANT

Train six independent five-fold multimodal systems. U-Net++ must be run through the same expanded eight-class comparison protocol. Do not treat an older four-class U-Net++ classifier as a directly comparable result.

Per architecture and fold:

## Stage 1 - Reuse OCTA-SimCLR

- Load the same completed expanded Phase 3 Sup, Deep, and CC backbones.
- Do not run new SimCLR training.
- Save a reuse audit with checkpoint hashes.

## Stage 2 - Train downstream modules with OCTA backbones frozen

- Epochs: 15
- LR: 3e-4
- Train projection heads, tabular encoder, mask encoder, cross-layer attention, compatibility fusion, prototype state/logit parameters, and classifier head.
- Freeze the three OCTA backbones.
- Freeze the selected segmentation students.

## Stage 3 - Fine-tune the disease classifier

- Maximum epochs: 100
- OCTA backbone LR: 3e-5
- Downstream-module LR: 3e-4
- Warmup: 5 epochs
- Cosine schedule
- Early stopping: patience 15 on inner-validation macro-F1
- Keep segmentation students frozen.

Use the same optimizer, effective batch size, gradient clipping, AMP policy, seeds, prototype initialization/update rules, and checkpoint-resume behavior for every architecture.

The outer test fold must not control model selection, early stopping, segmentation threshold selection, classifier temperature scaling, or hyperparameter changes. Outer-test metrics may be computed after the best checkpoint is locked, not used as per-epoch feedback.

For each fold, save calibrated outer-fold predictions with:

- Patient ID and eye
- True cohort label
- Eight raw logits
- Eight calibrated probabilities
- Predicted class
- Segmentation model ID
- Segmentation checkpoint hashes
- SimCLR checkpoint hashes
- Classifier checkpoint hash
- Fold and seed

# PHASE 13 - MATCHED EXPLAINABILITY

Run the same explainability procedures for all six trained multimodal variants using the same deterministic, class-balanced outer-test sample list per fold.

Required outputs:

- Sup, Deep, and CC Grad-CAM from the OCTA encoders
- Cross-layer attention-dominance summaries
- SHAP attribution for clinical features
- SHAP attribution for architecture-specific generated biomarkers
- Side-by-side segmentation masks for the same eyes
- Integrated summaries connecting OCTA saliency, layer dominance, tabular attribution, and mask QC

Label every artifact with the segmentation model ID. Do not average explanations across architectures before preserving model-specific results.

Explainability is descriptive. Do not claim that Grad-CAM, attention, or SHAP proves causal anatomical mechanisms.

# PHASE 14 - SEGMENTATION AND MULTIMODAL EVALUATION

## 14.1 Authoritative prediction set

Use only held-out outer-fold out-of-fold predictions. Combine the five folds only after validating that every expected patient/eye appears exactly once per architecture and that no outer-training prediction is present.

Report both eye-level performance and patient-level performance. Form patient-level class probabilities by averaging available OD/OS probabilities before computing patient-level metrics. Use patient-cluster resampling for confidence intervals so bilateral eyes are not treated as independent.

## 14.2 Segmentation agreement metrics

Against held-out classical improved pseudo-labels, report per task and per output channel:

- Dice / foreground F1
- IoU
- Precision
- Recall / sensitivity
- Specificity
- Balanced accuracy
- Pixel accuracy
- False-positive and false-negative rates
- Predicted and target positive rates
- QC pass rate
- Vessel skeleton agreement and connectivity measure for Sup/Deep vessel masks
- FAZ centroid, area, and circularity error for Sup/Deep FAZ masks
- CC flow-deficit fraction and component-count error

Include 95% patient-cluster bootstrap confidence intervals. Clearly title these results as pseudo-label agreement.

## 14.3 Multimodal classification metrics

Primary:

- Macro one-vs-rest ROC-AUC
- Macro-F1
- Balanced accuracy
- Per-class sensitivity and specificity

Secondary:

- Macro PR-AUC
- Weighted F1
- Per-class precision, recall, F1, ROC-AUC, and PR-AUC
- True-class-normalized confusion matrix
- Top-2 accuracy

Calibration:

- ECE with 10 bins
- Multiclass Brier score
- Per-class reliability diagrams

Uncertainty:

- Thirty-pass MC-dropout mean probabilities
- Predictive entropy
- Classwise epistemic variance
- Accuracy by entropy quartile
- Coverage and accuracy for the top 30 percent most-confident predictions

## 14.4 Pairwise architecture comparison

Use U-Net++ as the prespecified reference and compare each of the five alternatives against it. Also produce a complete descriptive six-by-six difference matrix, but do not select the best model using outer-test results and then retroactively redefine the reference.

For primary patient-level metrics:

- Compute paired differences on the same held-out patients.
- Use patient-level paired bootstrap confidence intervals for macro-F1, balanced accuracy, macro ROC-AUC, and macro PR-AUC differences.
- Use an exact paired correctness test for patient-level predictions.
- Correct the five primary reference comparisons for multiple testing.
- Report adjusted p-values, absolute differences, confidence intervals, and practical effect sizes.

Do not rank models from point estimates alone. A model is favored only when its multimodal performance, uncertainty/calibration behavior, segmentation QC, and efficiency are considered together.

## 14.5 Final comparison tables

Produce:

1. Segmentation pseudo-label agreement by architecture, task, channel, and fold.
2. Mask QC and biomarker stability by architecture.
3. Eye-level multimodal classification results.
4. Patient-level multimodal classification results.
5. Calibration and uncertainty results.
6. Parameters, FLOPs, memory, training time, and inference time.
7. Paired differences from U-Net++ with corrected inference.
8. Per-cohort classification results for all eight labels.
9. A final selection record stating the chosen segmentation architecture and the prespecified decision rule used.

## Completion criteria

This comparison is complete only when:

- All six architectures use the same expanded eight-cohort samples and fixed folds.
- Every architecture has fold-specific Sup/Deep and CC segmentation checkpoints.
- Deterministic outer-test masks and biomarkers exist for every architecture.
- Every full multimodal variant has five held-out classifier prediction files.
- There is no silent mask-source fallback.
- Segmentation results are labeled as pseudo-label agreement.
- Patient-level and eye-level Phase 14 reports are complete.
- All comparison artifacts are isolated from historical outputs.
- The selected model is supported by the locked Phase 14 decision rule rather than a single favorable metric.
