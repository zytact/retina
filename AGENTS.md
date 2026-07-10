# AGENTS.md

The goal of this project is to create a model trained on the RASTA dataset which has OCTA images and other sorts of images. It will predict Coronary Artery Disease. The directory contains one jupyter notebook, but do not try to run it. It will be run on a separate Windows machine with GPU. Your job is simply to work on it.

This local checkout is a code/documentation review environment, not the training/runtime environment. Do not treat Linux path portability, Windows backslash paths, absent local dataset files, or absent local `.pt` checkpoints as blockers by themselves. Training artifacts such as `.pt` checkpoints may remain only on the Windows GPU machine unless the user explicitly asks to copy them here.

Do not create any other python files or any other notebook files intended to run.

The plan will be in [PLAN.md](PLAN.md). Track the progress in a [PROGRESS.md](PROGRESS.md). No need to implement everything all at once. That is why the PROGRESS.md exists.

Since the dataset is not on this machine, the directory structure will be in [TREE.md](TREE.md) generated from the unix tree command.

The clinical data is in [Table for publication](table_for_publication.xlsx)

For creating docs/explainers, put them in @docs directory and use Impeccable skill, [PRODUCT.md](PRODUCT.md) and [DESIGN.md](DESIGN.md)

All phase 13 outputs are not available as they are 1.47 GB and it would take a lot of work to move it from training machine to here.
