# 2. Conformational subset construction

The pool of conformations (or trajectory frames) from Stage 1 is far too large
to score against cryoEM images one-by-one. This stage cuts it down to a
compact, representative subset.

!!! info "Inputs"
    - Frames from [Stage 1](stage1-sampling.md)
    - Collective variables (CVs) to perform farthest-point sampling (FPS) on
    - Number of clusters

## Method

Description of the conformational subset construction method — farthest-point
sampling (FPS) is used to select a diverse, representative set of conformations
from the full pool, and a Gaussian mixture model (GMM) is built over them.

<div class="grid cards" markdown>

- **cryoGMM** — in-house script for FPS selection and building the GMM
  [:material-github: minhuanli/cryoGMM](https://github.com/minhuanli/cryoGMM)

</div>

!!! note
    cryoGMM's FPS-clustering support is still growing — today it covers GMM
    construction (see [Stage 5](stage5-gmm.md)); the FPS selection step itself
    is expected to land in this same repo soon.

!!! success "Outputs"
    - **Representative conformations** — selected conformations representing
      the ensemble, in PDB format (all-atom or one bead per residue)
    - **Covariance** — covariance matrix (3N×3N) from the conformational analysis

Next: [3. Likelihood →](stage3-likelihood.md)
