# 2. Likelihood

Each representative conformation from Stage 1 is scored against the
experimental cryoEM particle images: how well does this 3D shape explain each
image?

!!! info "Inputs"
    - CryoEM particle images (see [CryoEM particles](inputs-cryoem.md))
    - Representative conformations from [Stage 1](stage2-conformational-subset.md)

## Method

Each conformation is compared against each particle image by generating 2D
templates at different poses. We use two likelihood estimators: explicit
integration over poses (cryoLike) and amortized integration over poses
(cryoSBI). See the Supplementary Information for details on when each is
best suited.

<div class="grid cards" markdown>

- **cryoLike**
  [:material-github: flatironinstitute/CryoLike](https://github.com/flatironinstitute/CryoLike)
- **cryoSBI**
  [:material-github: flatironinstitute/cryoSBI](https://github.com/flatironinstitute/cryoSBI/tree/classifier_multi_particle)

</div>

!!! success "Outputs"
    - Image-to-structure likelihood matrix

Next: [3. Ensemble reweighting →](stage4-reweighting.md)
