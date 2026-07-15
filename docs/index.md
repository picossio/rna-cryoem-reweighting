# RNA CryoEM Ensemble Reweighting

RNA molecules don't sit still — they explore many shapes, and a single static
structure often can't explain what a cryoEM experiment actually sees. This site
walks through a pipeline that combines cryoEM particle images with coarse-grained
conformational sampling to recover a **reweighted ensemble of RNA conformations**:
not just one structure, but a probability distribution over the shapes the
molecule actually adopts.

![Pipeline overview](assets/images/Diagram.png)

The pipeline has two inputs — cryoEM particles and an RNA sequence with its
secondary structure — and five stages that turn them into a final conformational
ensemble. Each stage page below links out to the specific tool or in-house script
used, so you can go as deep as you like.

[Start with the Overview →](overview.md){ .md-button .md-button--primary }
[Jump straight to Stage 1 →](stage1-sampling.md){ .md-button }
