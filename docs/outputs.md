# Outputs

The pipeline's end product is not a single structure, but a **probability
distribution over RNA conformations** — and a set of samples drawn from it.

!!! success "Conformational samples from probability distribution"
    A final ensemble of 3D RNA conformations, sampled from the Gaussian
    mixture model built in [Stage 5](stage5-gmm.md). Each sample is a
    plausible structure; together, they represent how the RNA's shape varies
    and how likely each variant is, consistent with both the physics-based
    sampling in [Stage 1](stage1-sampling.md) and the experimental cryoEM
    images used throughout the pipeline.

This ensemble can be used downstream for structural analysis, visualization,
or as a starting point for further modeling — rather than relying on a single
static structure that may not represent the full picture.

See [About](about.md) for the paper and citation.
