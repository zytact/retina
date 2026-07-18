# Retina RASTA project keypoints

## Framing and dataset

The project implements a multimodal four-cohort classifier for RASTA retinal OCTA scans. The model uses the Sup, Deep, and CC retinal layers together with clinical variables, generated biomarkers, and frozen weak-label masks. The four groups are folder-derived cohorts, not a direct binary CAD endpoint. MRCC is the clearest CAD-related cohort in this dataset, but the label is not equivalent to a clinically validated CAD diagnosis.

| Cohort | Working interpretation | Patients | Eyes |
| --- | --- | ---: | ---: |
| RETINORM | reference / normal retinal cohort | 75 | 136 |
| ORNET | other retinal / vascular cohort | 77 | 150 |
| FAMILIPO | familial / related cohort | 81 | 156 |
| MRCC | most CAD-related cohort in this framing | 32 | 34 |

The dataset contains 265 patients and 476 eyes: 211 bilateral, 35 OD-only, and 19 OS-only. No patient appears in multiple cohorts, and Sup, Deep, and CC are present for all eyes. BMI is missing in 44 rows and dyslipidemia in 2 rows. Missing values are handled with learned tokens rather than full-dataset imputation.

## Leakage-safe preprocessing

Patient identity is the split unit, while an eye is the prediction unit. OD and OS are grouped before assignment so both eyes from one person remain in the same outer fold. The five outer folds contain 94, 93, 98, 94, and 97 eyes with zero patient overlap.

- Images use percentile normalization, CLAHE, resize to 224 x 224, and fold-fitted statistics.
- Clinical variables and generated biomarkers use training-fold-only normalization and learned missing tokens.
- Synchronized train-only geometric changes are shared across Sup, Deep, and CC. Mild image perturbations and clinical masking are excluded from validation and test.
- Inner validation, calibration, and held-out scoring remain separate from the outer test eyes.

This structure supports an evaluation claim tied to patient separation, fold-local statistics, and out-of-fold predictions.

## Architecture

Sup, Deep, and CC pass through independent ConvNeXt-Tiny encoders. Cross-layer attention combines their visual tokens into F_octa. Clinical variables and generated biomarkers pass through a missing-token-aware tabular encoder into F_tabular. Frozen U-Net++ generated masks pass through a mask encoder into F_mask.

The visual and tabular streams meet through compatibility fusion:

`F_cond = LayerNorm(F_octa + Proj_tab(F_tabular))`

The model concatenates F_cond with F_mask and projects the result to H in R^512. The classifier maps H through the standard head into a 128-dimensional penultimate representation and standard class logits. Prototype logits are computed from distances to learned class prototypes and combined with the standard logits using the learned non-negative prototype weight. Temperature scaling converts the final logits into calibrated four-class probabilities.

## Representation and mask evidence

The repository records 15 SimCLR runs at 100 epochs for each retinal layer. Best losses are 1.54-1.80 for Sup, 1.81-2.07 for Deep, and 2.66-2.93 for CC, below the random baseline of about 5.541.

The mask stage begins with deterministic classical pseudo-masks and trains a compact U-Net++ student against QC-passed pseudo-labels. The QC pass rate is 0.9937. Across outer folds, representative student scores are approximately 0.890-0.898 Dice and 0.802-0.815 IoU for Sup / Deep vessel masks, plus 0.757-0.788 Dice and 0.609-0.651 IoU for CC flow-deficit masks. These are weak-label refinements, not human ground truth.

The representative montage for 3CHCA66_OD shows the Sup, Deep, and CC OCTA slabs alongside the classical_improved pseudo-mask outputs for Sup vessel, Deep vessel, Sup FAZ, Deep FAZ, and CC flow. These visualizations illustrate the morphology signal used to build mask-branch inputs; the downstream refinement path trains a U-Net++ student against QC-passed pseudo-labels.

## Classification and calibration

The primary evaluation uses calibrated outer-fold out-of-fold predictions. When both eyes are available, patient probabilities are formed by averaging the calibrated OD and OS probabilities. Confidence intervals use a 1,000-sample stratified patient-cluster bootstrap.

| Level | N | Accuracy | 95% CI | Macro ROC-AUC | Macro F1 | Balanced accuracy | Top-2 |
| --- | ---: | ---: | --- | ---: | ---: | ---: | ---: |
| Patient | 265 | 0.800 | 0.755-0.842 | 0.898 | 0.765 | 0.761 | 0.943 |
| Eye | 476 | 0.748 | 0.705-0.791 | 0.877 | 0.699 | 0.700 | 0.908 |

Patient-level recall is 0.840 for RETINORM, 0.857 for ORNET, 0.815 for FAMILIPO, and 0.531 for MRCC. Eye-level expected calibration error is 0.0346 and patient-level ECE is 0.0682. The small MRCC cohort remains the limiting class for interpretation.

## Correctly predicted MRCC example

Patient/eye: **4FORO45_OD**  \
True label: **MRCC**  \
Predicted label: **MRCC**  \
Calibrated MRCC probability: **0.9990770816802979**, displayed as **99.9%**

The Grad-CAM overlay shows `true=MRCC, predicted=MRCC` and localizes image regions supporting the MRCC score in Sup, Deep, and CC. The matching cross-layer attention row is:

| Sup | Deep | CC |
| ---: | ---: | ---: |
| 32.4% | 26.3% | 41.3% |

CC has the largest cross-layer attention share for this eye. These percentages describe retinal-layer attention within F_octa; they are not total fused-modality contributions.

The repository currently has no patient-specific tabular attribution or mask-branch attribution artifact for 4FORO45_OD. This example therefore explains image evidence, retinal-layer weighting, and the mathematical fusion path, but cannot quantify which clinical variable, generated biomarker, or mask feature drove this individual prediction. Grad-CAM and attention are associative explanations, not causal proof.

## Scientific strengths and limits

Strengths include patient-grouped outer folds with inner validation, fold-specific self-supervision and normalization, a multimodal design with a frozen mask branch, calibrated out-of-fold evaluation, clustered bootstrap intervals, and explicit checkpoint and resume controls.

Limits include the small and imbalanced MRCC cohort, folder-derived labels that are not direct CAD labels, one dataset without external validation, weakly supervised pseudo-masks rather than human ground truth, and incomplete local explainability beyond the available fold 0 artifacts.

## Evidence boundary

The evidence reviewed here includes the dataset audit, SimCLR histories, mask and student metrics, cross-validation summaries, local Phase 13 fold 0 explainability outputs, and the Phase 14 evaluation bundle. Phase 14 evaluation is complete. Local Phase 13 outputs include Grad-CAM and attention artifacts for fold 0, while patient-specific tabular and mask attribution artifacts are not present for 4FORO45_OD.

## Compact glossary

- **OCTA:** optical coherence tomography angiography, used here as retinal vascular imagery.
- **Sup / Deep / CC:** superficial, deep, and choriocapillaris retinal layers.
- **OD / OS:** right eye / left eye.
- **OOF:** out-of-fold prediction, where every sample is scored by a model that did not train on its patient.
- **ECE:** expected calibration error, a summary of confidence versus observed correctness.
- **MRCC:** the smallest RASTA cohort and the clearest CAD-related group in this framing.
- **F_octa / F_tabular / F_mask:** representations produced by the visual, tabular, and mask branches.
- **F_cond:** the LayerNorm compatibility-fused visual and tabular representation.

## How the fused model reaches a class conclusion

The Sup, Deep, and CC images pass through independent ConvNeXt-Tiny encoders. Cross-layer attention combines their visual tokens into F_octa and gives the reported retinal-layer dominance for 4FORO45_OD. Clinical variables and generated biomarkers pass through the missing-token-aware tabular encoder into F_tabular. U-Net++ generated masks pass through the mask encoder into F_mask.

Compatibility fusion forms `F_cond = LayerNorm(F_octa + Proj_tab(F_tabular))`. The model concatenates F_cond with F_mask and projects to H in R^512. The standard classifier head maps H to a 128-dimensional penultimate representation and standard class logits. A prototype branch computes prototype logits from distances to learned class prototypes. The standard logits and prototype logits are combined using the learned non-negative prototype weight. Temperature scaling converts the final logits into calibrated four-class probabilities. The largest calibrated probability determines the predicted class, MRCC for 4FORO45_OD.
