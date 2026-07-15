# 3. Likelihood

Each representative conformation from Stage 2 is scored against the
experimental cryoEM particle images: how well does this 3D shape explain what
the microscope actually saw?

!!! info "Inputs"
    - CryoEM particle images (see [Inputs](inputs.md))
    - Representative conformations from [Stage 2](stage2-conformational-subset.md)

## Method

Description of the likelihood calculation method — each conformation is
forward-projected and compared against the particle images to compute an
image-to-conformation likelihood.

<div class="grid cards" markdown>

- **cryoLike**
  [:material-github: flatironinstitute/CryoLike](https://github.com/flatironinstitute/CryoLike)
- **cryoSBI**
  [:material-github: flatironinstitute/cryoSBI](https://github.com/flatironinstitute/cryoSBI/tree/classifier_multi_particle)

</div>

!!! success "Outputs"
    - Image-to-conformation likelihood matrix

Next: [4. Ensemble reweighting →](stage4-reweighting.md)
