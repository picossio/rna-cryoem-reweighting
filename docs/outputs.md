# Outputs

The pipeline's end product is not a single structure, but **the ensemble
distribution of an RNA consistent with the cryo-EM images** — and a set of
samples drawn from it.

!!! success "Outputs"
    - **Weights for each representative subset** — useful when the system
      includes both compositional and conformational species, as for P4-P6.
    - **Conformational samples from the probability distribution** — a final
      ensemble of 3D RNA conformations, sampled from the Gaussian mixture
      model built in [Stage 4](stage5-gmm.md). Each sample is a plausible
      structure consistent with both the physics-based sampling in
      [Prior ensemble](inputs-cg-sampling.md) and the experimental cryoEM
      images used throughout the pipeline.

This ensemble can be used downstream for structural analysis, visualization,
quantitative comparison to other distributions, cross-validation, and
estimation of average observables.

See [About](about.md) for the paper and citation.
