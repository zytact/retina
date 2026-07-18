# AGENTS.md

The goal of this project is to create a model trained on the RASTA dataset which has OCTA images and other sorts of images. It will predict Coronary Artery Disease. The directory contains one jupyter notebook, but do not try to run it. It will be run on a separate Windows machine with GPU. Your job is simply to work on it.

The active dataset expansion adds four cohorts to the original four: `5_GIANTS`, `6_AWARD`, `7_EVIRED`, and `8_ORNET_TEMOINS`. Must now operate on all eight folders (not fully done and will not be done in one pass but rather as the user explicitly asks): `1_RETINORM`, `2_ORNET`, `3_FAMILIPO`, `4_MRCC`, `5_GIANTS`, `6_AWARD`, `7_EVIRED`, and `8_ORNET_TEMOINS`, with stable zero-based labels 0-7 in folder-number order. This expansion is currently implemented only through Phase 3. Keep all previous four-class artifacts intact and write the expanded-run outputs to new output directories rather than deleting or overwriting old outputs. Phase 3 OCTA-SimCLR for the expanded run uses 120 epochs.

This local checkout is a code/documentation review environment, not the training/runtime environment. Do not treat Linux path portability, Windows backslash paths, absent local dataset files, or absent local `.pt` checkpoints as blockers by themselves. Training artifacts such as `.pt` checkpoints may remain only on the Windows GPU machine unless the user explicitly asks to copy them here.

Do not create any other python files or any other notebook files intended to run.

Instead of the [main.ipynb](main.ipynb) file, the main notebook is now [segmentation_comparison_8class.ipynb](segmentation_comparison_8class.ipynb)

The original [PLAN.md](PLAN.md) is no longer to be used, other than as a reference. The new [PLAN_segmentation_comparison.md](PLAN_segmentation_comparison.md) is to be now used and the purpose is to redo the training with 8 classes and also use 5 more segmentation models making a total of 6 separate multimodal fusions.

As usual, [PROGRESS.md](PROGRESS.md) is for you track your changes.

Since the dataset is not on this machine, the directory structure will be in [TREE.md](TREE.md) generated from the unix tree command.

The clinical data is in [Table for publication](table_for_publication.xlsx)

For creating docs/explainers, put them in @docs directory and use Impeccable skill, [PRODUCT.md](PRODUCT.md) and [DESIGN.md](DESIGN.md)

All phase 13 outputs are not available as they are 1.47 GB and it would take a lot of work to move it from training machine to here.
