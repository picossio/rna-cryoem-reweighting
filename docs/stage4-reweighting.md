# 3. Ensemble reweighting

The likelihood matrix from [Stage 2](stage3-likelihood.md) says how well each
representative conformation explains each particle image. What it does not say
is how *populated* each conformation is. This stage recovers that: a set of
weights over the subset, fitted so the weighted ensemble best explains the
observed images.

!!! info "Inputs"
    - Image-to-structure log-likelihood matrix from
      [Stage 2](stage3-likelihood.md) — shape `(n_images, n_structures)`

## Running it

<div class="grid cards" markdown>

- **cryoGMM** — multiplicative gradient reweighting
  [:octicons-book-16: Ensemble reweighting tutorial](https://github.com/minhuanli/cryoGMM/blob/master/docs/reweighting_tutorial.md)

</div>

Installing cryoGMM (see [Overview](overview.md#software)) puts the
`cryogmm-reweight` command on your path:

```bash
cryogmm-reweight \
    --log_likelihood /data/likelihood/set_0/J1001/log_likelihood.npy \
    --output         /data/sbi/set_0/J1001/weights.pt
```

Run it once per job and per set. It is cheap — one pass over the likelihood
matrix per iteration, typically converging in tens to hundreds of iterations —
and a large matrix can be pushed onto a GPU with `--device cuda:0`.

### Arguments

| Argument | Expected value |
|----------|----------------|
| `--log_likelihood` | **Required.** The matrix from Stage 2, `.npy` or `.pt`, shape `(n_images, n_structures)`. |
| `--output` | **Required.** Where the weights go. `.pt` is what [Stage 4](stage5-gmm.md) reads; `.npy` also works. |
| `--tol` | Early-stopping tolerance. Default `1e-3`. |
| `--device` | `cpu` (default) or e.g. `cuda:0`. |
| `--double` | Flag. Run in float64; helpful for small `--tol` or when many weights are near 0. |

`--max_iterations`, `--stats_frequency` and `--verbose` are also available; the
complete list is in the
[reweighting tutorial](https://github.com/minhuanli/cryoGMM/blob/master/docs/reweighting_tutorial.md#key-arguments).

### Output

A 1-D tensor, one weight per column of the input matrix, summing to 1:

```
Loaded log-likelihood: 5000 images x 40 structures
exiting!
#iterations at exit: 3
Weights written to /data/sbi/set_0/J1001/weights.pt
```

## How it works

Treat the ensemble as a mixture whose components are the fixed representative
conformations and whose proportions are unknown. Each iteration scales every
weight by the average, over images, of how much that conformation contributes
to explaining each image relative to the current mixture:

```
w_j  ←  w_j · (1/n) Σ_i  L_ij / ( Σ_k w_k L_ik )
```

where `L_ij` is the likelihood of image `i` under conformation `j`, and the
denominator is the current mixture's density for that image. In the code this
runs entirely in log space, via `logsumexp`, so it stays numerically stable for
the very small likelihoods typical of noisy particle images.

Written this way the weights start uniform, stay
non-negative, and stay normalized, with no projection step. It is the
expectation-maximization update for mixture proportions, and every step is
guaranteed not to decrease the likelihood.

Iteration stops on a user-set tolerance rather than a fixed budget of iterations. The quantity
`max(grad) - 1` upper-bounds the gap between the current log-likelihood and
that of the optimal weights, so once it falls below `--tol` the weights are
known to be within `tol` of the best achievable. That is what "early-stopped"
means here: it is a bound as opposed to a heuristic cutoff.

See the Supplementary Information section 3.5 for how this is adapted to
compositional species and nested likelihoods, and the
[corresponding manuscript](https://www.nature.com/articles/s42003-026-09859-6)
introducing the method for cryo-EM.

!!! note "Compositional species"
    Columns of the likelihood matrix need not all be conformations. When the
    sample contains several species — for P4-P6, monomer conformations
    alongside dimers and a junk/noise class — the extra columns are additional
    mixture components, reweighted alongside the conformations in the same
    pass. The resulting vector is what Stage 4 consumes.

## Checking the result

How concentrated the weights came out is the quickest diagnostic:

```python
import torch
w = torch.load("weights.pt", weights_only=True).double()
print("effective structures:", float(torch.exp(-(w[w > 0] * w[w > 0].log()).sum())))
```

This is the perplexity of the weight distribution — how many conformations the
ensemble effectively rests on. Collapsing towards 1 means a single structure is
explaining every image, which usually points to a subset too coarse to describe
the data, or an overconfident likelihood. Close to the full subset size means
the images are barely discriminating between conformations, and the reweighted
ensemble will look much like the prior.

Neither extreme is a failure on its own — a genuinely narrow ensemble *should*
concentrate — but the value is worth comparing across your
[sets](stage2-conformational-subset.md#how-many-sets). Run with `--verbose` to
watch the loss; it cannot increase, so an increase indicates a numerical
problem, and `--double` is the first thing to try.

!!! success "Outputs"
    - **Weights** — reweighted probabilities for each representative
      conformation, one per column of the likelihood matrix, summing to 1

Next: [4. Gaussian mixture model →](stage5-gmm.md)
