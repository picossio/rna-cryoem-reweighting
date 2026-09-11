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
conformations and whose proportions are unknown. Starting from uniform
weights, each iteration reweights every conformation upward or downward based
on how much it helps explain the images, given the current mixture — the
standard expectation-maximization update for mixture proportions. Weights stay
non-negative and normalized throughout, with no separate projection step, and
every iteration is guaranteed not to decrease the likelihood.

Iteration stops on a user-set tolerance rather than a fixed budget of
iterations. The quantity `max(grad) - 1` upper-bounds the gap between the
current log-likelihood and that of the optimal weights, so once it falls
below `--tol` the weights are known to be within `tol` of the best
achievable. That is what "early-stopped" means here: it is a bound as
opposed to a heuristic cutoff — see details in
[Mordant et al.](https://arxiv.org/html/2609.01688v1).

See the Supplementary Information section 3.5 of
[Evans et al.](https://www.nature.com/articles/s42003-026-09859-6) for how
this is adapted to compositional species and nested likelihood.

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
[sets](stage2-conformational-subset.md#how-many-sets) and different `tol`
thresholds. Run with `--verbose` to
watch the loss; it cannot increase, so an increase indicates a numerical
problem, and `--double` is the first thing to try.

!!! success "Outputs"
    - **Weights** — reweighted probabilities for each representative
      conformation, one per column of the likelihood matrix, summing to 1

Next: [4. Gaussian mixture model →](stage5-gmm.md)
