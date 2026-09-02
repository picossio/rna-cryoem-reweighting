# 3. Ensemble reweighting

The likelihood matrix from Stage 2 is turned into a set of weights: how much
each representative conformation should count toward the final ensemble.

!!! info "Inputs"
    - Image-to-structure likelihood matrix from [Stage 2](stage3-likelihood.md)

## Method

We use a multiplicative gradient procedure with early stopping that
iteratively updates the per-conformation weights so that the weighted
ensemble best explains the observed particle image set. See the Supplementary Information section 3.5 for details on how this method is adapted to the compositional species and nested likelihood, and the [corresponding manuscript](https://www.nature.com/articles/s42003-026-09859-6) introducing this method for cryo-EM.

<div class="grid cards" markdown>

- **cryoGMM** — script for multiplicative gradient procedure
  [:material-github: minhuanli/cryoGMM](https://github.com/minhuanli/cryoGMM/blob/master/cryogmm/utils/reweighting.py)

</div>

Given a log-likelihood matrix, particularly the **Image-to-Structure Likelihood-matrix** from [stage 2](stage3-likelihood.md), this code can be run as:
```
weights = multiplicative_gradient(log_likelihood)
```
which gets the weights from early-stopping at a default tolerance of `tol=10**-3`.

!!! success "Outputs"
    - **Weights** — reweighted probabilities for each conformation

Next: [4. Gaussian mixture model →](stage5-gmm.md)
