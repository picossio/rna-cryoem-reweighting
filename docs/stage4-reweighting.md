# 3. Ensemble reweighting

The likelihood matrix from Stage 2 is turned into a set of weights: how much
each representative conformation should count toward the final ensemble.

!!! info "Inputs"
    - Image-to-structure likelihood matrix from [Stage 2](stage3-likelihood.md)

## Method

We use a multiplicative gradient procedure with early stopping that
iteratively updates the per-conformation weights so that the weighted
ensemble best explains the observed particle image set.

!!! warning "🚧 Tool link coming soon"
    The EM reweighting script isn't public yet. This section will
    link directly to it once it's released.

!!! success "Outputs"
    - **Weights** — reweighted probabilities for each conformation

Next: [4. Gaussian mixture model →](stage5-gmm.md)
