# Outputs

The pipeline's end product is not a single structure, but **the ensemble
distribution of an RNA consistent with the cryo-EM images** — and a set of
samples drawn from it.

!!! success "Outputs"
    - **Weights for each representative subset** — useful when the system
      includes both compositional and conformational species, as for P4-P6.
    - **Conformational samples from the probability distribution** — a final
      ensemble of 3D RNA conformations, sampled from the Gaussian mixture
      model built in [Stage 4](stage5-gmm.md). Each sample is a plausible
      structure consistent with both the physics-based sampling in
      [Prior ensemble](inputs-cg-sampling.md) and the experimental cryoEM
      images used throughout the pipeline.

## Where to find it

Everything lives under the `--output_root` given to
[Stage 4](stage5-gmm.md), keyed by **job** (one experimental condition) and
**set** (one clustering from [Stage 1](stage2-conformational-subset.md)):

```
/data/gmm_results/my_system/
  40_centers/
    Job_1001/                       # one experimental condition
      Set_0/                        # one clustering from Stage 1
        gmm_pi.pt                   # weights over the representative subset
        gmm_samples/
          gmm_sample_BB2_000.pdb    # ← the reweighted ensemble
          gmm_sample_BB2_001.pdb
          ...
          gmm_centers.txt           # which conformation each sample came from
      Set_1/ ...
    Job_1002/ ...
```

So you get one ensemble of `--n_samples` structures **per condition, per set**.
The conditions are what you compare; the spread across sets is the uncertainty
on that comparison.

The `Set_*` directories sitting alongside `Job_*`, and the `bond_dist_gaussian/`
folder at the root, hold the fitted mixture itself — component means,
covariances and the per-cluster bases. They are intermediates, cached so that
adding a new condition does not refit anything, and are not needed for
downstream analysis.

This ensemble can be used downstream for structural analysis, visualization,
quantitative comparison to other distributions, cross-validation, and
estimation of average observables.

## Analysing the ensemble

cryoGMM carries the analysis used in the paper as two further steps of its
[pipeline tutorial](https://github.com/minhuanli/cryoGMM/blob/master/docs/tutorial.md):

- **[Collective variables](https://github.com/minhuanli/cryoGMM/blob/master/docs/tutorial.md#step-1-compute-cvs)**
  — draw from the fitted mixture for each condition and reduce every sampled
  structure to a few interpretable numbers, alongside the same quantities
  measured on the prior ensemble for reference. Weight statistics are averaged
  across your [sets](stage2-conformational-subset.md#how-many-sets), which is
  where the error bars come from.
- **[Figures](https://github.com/minhuanli/cryoGMM/blob/master/docs/tutorial.md#step-2-generate-figures)**
  — three views of the result: the CV distribution per condition, stacked for
  comparison; a heatmap of the total variation distance between every pair of
  conditions; and a trend of the mean CV against a quantitative variable such
  as Mg²⁺ concentration.

!!! note "Worked examples, not general tools"
    Both are written around the P4-P6 study — the collective variables are
    fixed residue indices in the script, and the figure scripts default to that
    experiment's condition IDs and labels. Treat them as a template to adapt,
    rather than something to point at a different system unchanged. The
    ensemble itself is plain PDB files, so any analysis you already have will
    read it.

See [About](about.md) for the paper and citation.
