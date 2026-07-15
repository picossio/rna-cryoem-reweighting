# 5. Gaussian mixture model

The weights, representative conformations, and covariance information are
combined into a Gaussian mixture model: a smooth probability distribution over
RNA conformational space that can be sampled from directly.

!!! info "Inputs"
    - Weights from [Stage 4](stage4-reweighting.md)
    - Representative conformations from [Stage 2](stage2-conformational-subset.md)
    - Covariance from [Stage 2](stage2-conformational-subset.md)

## Method

Description of the Gaussian mixture model method — related to
[Stage 2](stage2-conformational-subset.md); this is the in-house inference
step that fits the mixture model and draws samples from it.

<div class="grid cards" markdown>

- **cryoGMM** — in-house script for the GMM inference
  [:material-github: minhuanli/cryoGMM](https://github.com/minhuanli/cryoGMM)

</div>

!!! success "Final output"
    - **Conformational samples from probability distribution** — the final
      ensemble of conformations, sampled from the resulting probability
      distribution. See [Outputs](outputs.md).
