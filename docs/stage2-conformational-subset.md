# 1. Conformational subset construction

The pool of conformations (or trajectory frames) from
[prior ensemble](inputs-cg-sampling.md) is far too large to score
against cryoEM images one-by-one. This stage cuts it down to a compact,
representative subset.

!!! info "Inputs"
    - Frames from [Prior ensemble](inputs-cg-sampling.md)
    - Number of clusters
    - (optional) Collective variables (CVs) to perform farthest-point sampling (FPS) on

## Method

Farthest-point sampling (FPS) is used to select a diverse, representative set
of conformations from the full pool, and a Gaussian mixture model (GMM) is
built over them.

<div class="grid cards" markdown>

- **cryoGMM** — script for FPS selection and covariance estimate
  [:material-github: minhuanli/cryoGMM](https://github.com/minhuanli/cryoGMM)

</div>

!!! note
    cryoGMM's FPS-clustering support is still growing — today it covers GMM
    construction (see [Stage 4](stage5-gmm.md)); the FPS selection step itself
    is expected to land in this same repo soon. FPS is performed in CV space;
    it could also be done using RMSD, but that option is not supported here.

!!! success "Outputs"
    - **Representative conformations** — selected conformations representing
      the ensemble, in PDB format (all-atom or one bead per residue)
    - **Covariance** — covariance matrix (3N×3N) representing the
      conformational flexibility around each subset

Next: [2. Likelihood →](stage3-likelihood.md)
