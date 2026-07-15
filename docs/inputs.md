# Inputs

The pipeline starts from two independent sources of information: experimental
images, and prior knowledge about the RNA itself.

!!! info "CryoEM particles"
    Experimental particle images from cryo-electron microscopy. Each particle
    is a single, noisy 2D projection of one RNA molecule in some (unknown)
    conformation, at some (unknown) orientation. Across thousands of particles,
    the full set implicitly encodes the ensemble of conformations present in
    the sample — but not directly; that's what the rest of the pipeline
    recovers.

!!! info "RNA sequence + secondary structure"
    The RNA sequence, together with a predicted or experimentally known
    secondary structure (which bases pair with which). Fixing the secondary
    structure defines the space of plausible 3D conformations that the
    coarse-grained sampling in [Stage 1](stage1-sampling.md) will explore.

These two inputs meet again in [Stage 3 (Likelihood)](stage3-likelihood.md),
where representative conformations generated from the sequence/structure are
scored against the cryoEM particle images.
