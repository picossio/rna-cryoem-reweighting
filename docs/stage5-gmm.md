# 4. Gaussian mixture model

The weights, representative conformations, and covariance information are
combined into a Gaussian mixture model: a smooth probability distribution over
RNA conformational space that can be sampled from directly.

!!! info "Inputs"
    - Weights from [Stage 3](stage4-reweighting.md)
    - Representative conformations from [Stage 1](stage2-conformational-subset.md)
    - Covariance from [Stage 1](stage2-conformational-subset.md)

## Method

This is the inference step that fits the mixture model and draws samples
from it.

<div class="grid cards" markdown>

- **cryoGMM** — script for the GMM inference
  [:material-github: minhuanli/cryoGMM](https://github.com/minhuanli/cryoGMM)

</div>

!!! success "Final output"
    - **Conformational samples from reweighted ensemble distribution** —
      conformations, sampled from the inferred GMM. See [Outputs](outputs.md).
