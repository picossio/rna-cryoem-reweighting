# 4. Ensemble reweighting

The likelihood matrix from Stage 3 is turned into a set of weights: how much
each representative conformation should count toward the final ensemble.

!!! info "Inputs"
    - Image-to-conformation likelihood matrix from [Stage 3](stage3-likelihood.md)
    - Gradient descent hyperparameters

## Method

Description of the ensemble reweighting method — an expectation-maximization
(EM) procedure iteratively updates the per-conformation weights so that the
weighted ensemble best explains the observed particle images.

!!! warning "🚧 Tool link coming soon"
    The in-house EM reweighting script isn't public yet. This section will
    link directly to it once it's released.

!!! success "Outputs"
    - **Weights** — reweighted probabilities for each conformation

Next: [5. Gaussian mixture model →](stage5-gmm.md)
