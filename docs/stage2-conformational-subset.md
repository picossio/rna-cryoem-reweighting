# 1. Conformational subset construction

The [prior ensemble](inputs-cg-sampling.md) typically holds 10⁴–10⁶ frames — far
too many to score against cryoEM images one by one. This stage picks a few tens
of structures that between them cover the conformational space, and records
which frames each one stands for.

!!! info "Inputs"
    - Frames from [Prior ensemble](inputs-cg-sampling.md) — trajectory
      coordinates plus a matching topology PDB
    - Number of clusters *K*
    - Collective variables (CVs) to perform farthest-point sampling (FPS) on

## Running it

<div class="grid cards" markdown>

- **cryoGMM** — FPS selection and cluster assignment
  [:octicons-book-16: Subset construction tutorial](https://github.com/minhuanli/cryoGMM/blob/master/docs/fps_clustering_tutorial.md)

</div>

Installing cryoGMM (see [Overview](overview.md#software)) puts the
`cryogmm-fps` command on your path. One invocation, from any directory, does
the whole stage:

```bash
cryogmm-fps \
    --traj_path            /data/traj/positions_all_traj.pt \
    --traj_top             /data/traj/top.pdb \
    --output_root          /data/clusters/my_system \
    --alignment_selection  "(resi > 108) and name BB2" \
    --cv                   file:/data/traj/rmsd_to_closed.dat \
    --cv                   dist:33,529 \
    --cv_labels            "RMSD to closed state" "CV2, dist(G6-A81)" \
    --n_clusters           40 \
    --seeds                42,12345,162,160,70 \
    --backend              numpy \
    --refine
```

These are the settings used in the paper for the P4-P6 domain: the CV space is
spanned by an RMSD to the closed state and the G6-A81 distance. The command
picks 40 representative frames spread out over that space, assigns every other
frame to its nearest representative, and repeats the whole thing for five
random seeds. It takes a few minutes for ~10⁵ frames and needs no GPU.

### Arguments

| Argument | Expected value |
|----------|----------------|
| `--traj_path` | **Required.** Trajectory coordinates: a `.pt` tensor of shape `(N_frames, N_atoms, 3)` in nm, or any MDTraj-readable trajectory file. |
| `--traj_top` | **Required.** Topology PDB, matching the trajectory's atom order. |
| `--output_root` | **Required.** Where results go — `{output_root}/{n_clusters}_clusters/set_{i}/`. |
| `--alignment_selection` | An MDTraj selection string to superpose on, e.g. `"(resi > 108) and name BB2"`. Choose the rigid part of the molecule. |
| `--cv` | **Required**, unless `--cv_path` is given. One collective variable per flag, repeatable: `dist:i,j`, `angle:i,j,k`, `torsion:i,j,k,l`, `file:path` or `npy:path`. |
| `--cv_path` | A precomputed `(N_frames, D_cv)` `.npy` array, used instead of `--cv`. |
| `--cv_labels` | Axis labels for `clustering.png`, one per CV. Cosmetic. |
| `--n_clusters` | Integer, the size of the subset. Default `40`. |
| `--seeds` | Comma-separated integers, one independent set per seed. Default `42,12345,162,160,70`. |
| `--backend` | `fpsample` (default, fast Rust implementation) or `numpy`. |
| `--refine` | Flag. Runs the max-min refinement pass after FPS — see [Refinement](#refinement). |

Also often useful: `--angstrom_to_nm` for trajectories stored in Å,
`--mode resample` (see [How many sets](#how-many-sets)), and `--no_pdb` to skip
writing the representative structures. The complete list is in the
[subset construction tutorial](https://github.com/minhuanli/cryoGMM/blob/master/docs/fps_clustering_tutorial.md#step-run-the-fps-clustering).

### Output

It writes one directory per seed:

```
/data/clusters/my_system/
  40_clusters/
    set_0/
      center_idx.npy       # (K,)  frame index of each representative
      cluster_labels.npy   # (N,)  cluster label for every frame
      center_0.pdb ...     # the representative conformations
      clustering.png       # CV space coloured by cluster, centers marked
    set_1/ ...
```

- `center_*.pdb` are the structures [Stage 2](stage3-likelihood.md) scores
  against the particle images.
- The two `.npy` files are what [Stage 4](stage5-gmm.md) reads to build the
  mixture model.
- `clustering.png` is a diagnostic — see [Checking the result](#checking-the-result).

The example above is the P4-P6 domain; `--alignment_selection`, `--cv` and
`--n_clusters` are the three that need thought for your own system, and
[Choosing the parameters](#choosing-the-parameters) covers each one.

## How it works

Two things happen. First, **farthest-point sampling** picks *K* frames: start
from one frame, then repeatedly add whichever frame lies furthest from
everything already picked. Second, every remaining frame is assigned to its
**nearest selected frame**, partitioning the trajectory into *K* clusters.

FPS rather than, say, k-means, because the two answer different questions.
K-means places centers where the density is, so a state that the forcefield
visits rarely may get no representative at all. FPS spreads the centers evenly
over the *occupied region*, regardless of how often each part was visited. That
is what you want here: the prior population of a conformation is exactly the
thing the cryoEM data is supposed to correct, so the subset should not bake it
in. This stage decides **coverage**; [Stage 3](stage4-reweighting.md) decides
population.

The clusters matter as much as the centers. The spread of frames within a
cluster is what later becomes the local covariance — the "width" attached to
each representative in the final mixture model — so each cluster needs enough
members to estimate that from.

!!! note "Where the covariance comes from"
    The local covariance around each representative is derived from
    `cluster_labels.npy` — it is the spread of that cluster's frames, estimated
    in a local PCA basis with a Ledoit-Wolf shrinkage estimator. In practice
    that calculation is done by cryoGMM's `gmm_build.py`, which you run in
    [Stage 4](stage5-gmm.md); it is cached on disk and reused across all
    downstream jobs. Conceptually it belongs to this stage, so it is listed
    among the outputs below.

## Choosing the parameters

### Collective variables

FPS is run in a low-dimensional **collective-variable space**, not on Cartesian
coordinates. Two or three CVs that separate the conformational states of
interest work well — typically inter-residue distances, an angle, or an RMSD to
a reference state such as a known closed conformation.

Each `--cv` flag adds one dimension, computed on the fly from the aligned
trajectory:

| Spec | Meaning |
|------|---------|
| `dist:i,j` | Distance between atoms `i` and `j` |
| `angle:i,j,k` | Angle at atom `j` |
| `torsion:i,j,k,l` | Dihedral angle |
| `file:path` | A column of text, e.g. an externally computed RMSD trajectory |
| `npy:path` | A 1-D `.npy` array |

Align the trajectory first, via `--alignment_selection`, on the structurally
rigid part of the molecule — otherwise the CVs pick up overall tumbling rather
than genuine conformational change. In the example above, P4-P6 is superposed
on the backbone beads of residues beyond 108, and the two CVs are an RMSD to
the closed state and the G6-A81 distance.

!!! warning "CVs are compared on their raw scale"
    FPS uses a plain Euclidean metric, so a CV spanning a wider numerical range
    counts for more when the centers are chosen — the example above mixes an
    RMSD in Å with a distance in nm. Rescale the columns and pass them with
    `--cv_path` if you want the CVs weighted equally.

!!! note
    FPS could in principle be run on pairwise RMSD instead of CVs, but that is
    not supported here: the RMSD matrix is quadratic in the number of frames,
    and CV space is what the downstream analysis is expressed in anyway.

### How many clusters

`--n_clusters` trades resolution against statistics in both directions: more
centers describe the conformational density more finely, but each cluster then
holds fewer frames — a noisier local covariance — and [Stage 2](stage3-likelihood.md)
has more structures to score against every particle image. 40 is a reasonable
starting point for a system like P4-P6 with ~10⁵ frames.

### How many sets

Farthest-point sampling is randomised only through its starting frame, so a
fixed seed reproduces a clustering exactly, and different seeds give genuinely
independent subsets. Running five of them (the default `--seeds`) and carrying
each one through the whole pipeline gives five reweighted ensembles; their
spread is the uncertainty on the final result.

To isolate a different source of variation — how much the answer depends on
*which member* of each cluster represents it, with the clustering held fixed —
use `--mode resample` instead. See the
[resampling section](https://github.com/minhuanli/cryoGMM/blob/master/docs/fps_clustering_tutorial.md#resampling-mode)
of the tutorial.

### Refinement

Greedy FPS is order-dependent, and can leave two centers closer together than
necessary. `--refine` sweeps the centers, proposing random candidate swaps and
accepting any that increases the minimum pairwise distance between them; it
stops as soon as a full sweep finds no improvement.

This is not cosmetic — on the P4-P6 run above it moved between 0 and 9 of the
40 centers depending on the seed. It is the slower path, since the candidate
pool defaults to 10% of the trajectory per center per sweep (`--pool_frac`).

!!! note "Backends"
    `--backend fpsample` (the default) and `--backend numpy` implement the same
    greedy algorithm, but `fpsample` computes in single precision, so the two
    can select different frames where distances are nearly tied. Pick one and
    stay with it for reproducibility; the paper's clustering used `numpy`.

## Checking the result

The script reports the cluster occupancy for each set:

```
Cluster occupancy: min 9, median 709, max 12786
  WARNING: 1 cluster(s) hold fewer than 10 frames
```

A cluster with very few frames carries too little data for a meaningful local
covariance. Stage 4 handles the extreme case gracefully — such a cluster falls
back to a fixed narrow width at its center, and is flagged in the output — but
if many clusters are that small, the subset is too fine for the trajectory you
have. Reduce *K*, or choose CVs that spread the density more evenly.

Then open `clustering.png`: the centers should tile the populated region of CV
space, with no large occupied area left without a nearby center.

!!! success "Outputs"
    - **Representative conformations** — the selected conformations, in PDB
      format (all-atom or one bead per residue)
    - **Cluster assignment** — which frames each representative stands for
    - **Covariance** — the local conformational flexibility around each
      representative, equivalent to a 3N×3N covariance matrix in Cartesian
      space (computed in [Stage 4](stage5-gmm.md); see the note above)

Next: [2. Likelihood →](stage3-likelihood.md)
