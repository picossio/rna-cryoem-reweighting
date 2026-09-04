# 4. Gaussian mixture model

Stages 1 to 3 leave you with a finite subset of conformations and a weight for
each. This stage turns that into a **continuous** distribution over RNA shape
space — a Gaussian mixture, one component per representative conformation,
weighted by the reweighted populations and widened by how much the prior
ensemble moved around within each cluster — and draws a full conformational
ensemble from it.

!!! info "Inputs"
    - **Weights** from [Stage 3](stage4-reweighting.md)
    - **Cluster assignment** from [Stage 1](stage2-conformational-subset.md) —
      `center_idx.npy` and `cluster_labels.npy`
    - **Prior ensemble** trajectory and topology, the same ones clustered in
      [Stage 1](stage2-conformational-subset.md)

## Running it

<div class="grid cards" markdown>

- **cryoGMM** — GMM construction and sampling
  [:octicons-book-16: Pipeline tutorial](https://github.com/minhuanli/cryoGMM/blob/master/docs/tutorial.md#step-0-build-the-gmm)

</div>

```bash
cryogmm-build-gmm \
    --traj_path             /data/traj/positions_all_traj.pt \
    --traj_top              /data/traj/top.pdb \
    --cluster_root          /data/clusters/my_system/{n_clusters}_clusters \
    --weights_path_template /data/sbi/set_{set_id}/J{job_id}/weights.pt \
    --output_root           /data/gmm_results/my_system \
    --bb_selection          "name BB2" \
    --alignment_selection   "(resi > 108) and name BB2" \
    --job_ids               1001,1002,1003,1004 \
    --set_ids               0,1,2,3,4 \
    --n_clusters            40 \
    --device                cuda:0
```

`--cluster_root` and `--weights_path_template` are templates: the `{set_id}`,
`{job_id}` and `{n_clusters}` placeholders are filled in as the script loops,
so one command covers every set and every experimental condition.

### Arguments

| Argument | Expected value |
|----------|----------------|
| `--traj_path` | **Required.** The prior ensemble trajectory — the same `.pt` or MDTraj-readable file clustered in Stage 1. |
| `--traj_top` | **Required.** Topology PDB matching the trajectory. |
| `--cluster_root` | **Required.** Stage 1's output directory. May contain `{n_clusters}`; `set_{set_id}` is appended automatically. |
| `--weights_path_template` | **Required.** Path to Stage 3's weights, containing `{set_id}` and `{job_id}`. |
| `--output_root` | **Required.** Where the mixture and its samples are written. |
| `--bb_selection` | **Required.** MDTraj selection for the atoms the mixture is built over, e.g. `"name BB2"` for one bead per residue. |
| `--alignment_selection` | **Required.** MDTraj selection for superposition; use the same one as Stage 1. |
| `--job_ids` | Comma-separated experimental conditions, one weight file each. |
| `--set_ids` | Comma-separated sets from Stage 1. Default `0,1,2,3,4`. |
| `--n_clusters` | Size of the subset. Default `40`. |
| `--n_samples` | Conformations to draw per job and set. Default `200`. |
| `--device` | Torch device. Default `cuda:0`. |
| `--cov_reg_min` | Covariance eigenvalue floor. Default `1e-4`; use `1e-2` for trajectories in Å. |
| `--force` | Recompute cached whiteners rather than reusing them. |

### Output

```
/data/gmm_results/my_system/
  bond_dist_gaussian/
    mu_bond.npy, sigma_bond.npy      # backbone bond-length statistics
  40_centers/
    Set_0/
      whiteners/whitener_cluster_0.pkl ...
      gmm_cluster_0_Mu_normed.pt     # component mean
      gmm_cluster_0_Sigma.pt         # component covariance
      gmm_cluster_0_is_degenerate.npy
    Job_1001/
      Set_0/
        gmm_pi.pt                    # normalised mixture weights
        gmm_samples/
          gmm_sample_BB2_000.pdb ... # the reweighted ensemble
          gmm_centers.txt            # which component each sample came from
```

The PDBs under `gmm_samples/` are the pipeline's end product — see
[Outputs](outputs.md).

## How it works

**A width for every representative.** Each cluster's frames are collected and a
local principal-component basis is fitted to them, keeping however many
components are needed to explain 95% of that cluster's variance — a floppy
region of the molecule earns more dimensions than a rigid one. The covariance
in that reduced basis is estimated with a Ledoit-Wolf shrinkage estimator,
which behaves far better than a plain sample covariance when a cluster holds
only a few hundred frames. This is the covariance that
[Stage 1](stage2-conformational-subset.md) lists among its outputs; it is
computed here because this is where it is used.

**Clusters too small to fit.** A cluster with fewer than about twice its
retained dimensions gets an identity basis pinned to its center frame and a
narrow fixed width instead, and is flagged in `gmm_cluster_k_is_degenerate.npy`.
That is a deliberate fallback rather than a failure — the center is a real
frame from the prior ensemble, with valid geometry.

**Sampling with a geometry filter.** A component is drawn according to the
weights, a point is drawn from its Gaussian, and it is mapped back to Cartesian
coordinates. A Gaussian in shape space has no notion of chemistry, so some
draws come back with stretched or compressed backbone bonds. Each proposal is
therefore checked against the distribution of consecutive bond lengths measured
over the whole prior ensemble, and rejected if it is implausible — which is why
the sampler over-proposes by `--bond_oversample_factor` and then filters down
to `--n_samples`.

!!! note "Caching"
    The per-cluster bases and covariances depend only on the clustering, not on
    the weights, so they are computed once per set and reused across every job.
    On a cluster, the natural parallel axis is therefore the set — see the
    Slurm example in the
    [tutorial](https://github.com/minhuanli/cryoGMM/blob/master/docs/tutorial.md#running-on-a-cluster-slurm).
    Add `--force` to recompute rather than reuse.

## Checking the result

Watch the per-cluster line as the whiteners are fitted:

```
Cluster 37: N=4991, keepdims=11
Cluster 39: N=79, keepdims=5
Cluster 12: N=6, keepdims=3, need>=6 — identity whitener at cluster center (bond filter skipped)
```

A handful of degenerate clusters is normal; many of them means the subset is
too fine for the trajectory, and *K* should come down in
[Stage 1](stage2-conformational-subset.md#how-many-clusters).

You always get `--n_samples` structures back, so the thing to watch for is a
warning saying otherwise:

```
Warning: cluster 12 produced only 3/17 samples after bond filter. Sampling with replacement.
```

That means the component is proposing mostly implausible geometry and the
shortfall was made up by duplicating what survived, so those samples are not
independent draws. Raising `--bond_oversample_factor` gives the filter more to
work with. A harsher variant — `0/... passed bond filter ... falling back to
top-N by bond log-prob` — means nothing at all cleared the cutoff for that
component, and its samples should be treated with suspicion.

It is also worth opening a few of the sampled PDBs and confirming they look
like the molecule rather than a tangle: the filter checks consecutive bond
lengths, not the whole fold.

For a trajectory stored in Ångström, note that this step has no unit
conversion — it works in whatever units the trajectory uses, and the bond
statistics are measured from that same trajectory, so it stays self-consistent.
What does need changing is `--cov_reg_min`, which should go to `1e-2`, since
eigenvalue thresholds scale with the square of the coordinate unit.

!!! success "Final output"
    - **Conformational samples from the reweighted ensemble distribution** —
      conformations drawn from the inferred GMM, in PDB format. See
      [Outputs](outputs.md).

Next: [Outputs →](outputs.md)
