# 1. Conformational subset construction

The pool of conformations (or trajectory frames) from the
[prior ensemble](inputs-cg-sampling.md) is far too large to score
against cryoEM images one-by-one. This stage cuts it down to a compact,
representative subset — a few tens of structures that between them cover the
conformational space the simulation explored — and records which frames belong
to each one, so the local flexibility around each structure can be estimated
later.

!!! info "Inputs"
    - Frames from [Prior ensemble](inputs-cg-sampling.md) — trajectory
      coordinates plus a matching topology PDB
    - Number of clusters *K*
    - Collective variables (CVs) to perform farthest-point sampling (FPS) on

## Method

Two things happen here. First, **farthest-point sampling** picks *K* frames:
start from one frame, then repeatedly add whichever frame lies furthest from
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

### Choosing collective variables

FPS is run in a low-dimensional **collective-variable space**, not on Cartesian
coordinates. Two or three CVs that separate the conformational states of
interest work well — typically inter-residue distances, an angle, or an RMSD to
a reference state such as a known closed conformation.

Align the trajectory first, on the structurally rigid part of the molecule, so
that the CVs measure genuine conformational change rather than overall
tumbling. For the P4-P6 domain, for example, the trajectory is superposed on
the backbone beads of residues beyond 108, and the two CVs are the A50-C120 and
G6-A81 distances.

!!! note
    FPS could in principle be run on pairwise RMSD instead of CVs, but that is
    not supported here: the RMSD matrix is quadratic in the number of frames,
    and CV space is what the downstream analysis is expressed in anyway.

## Running it

<div class="grid cards" markdown>

- **cryoGMM** — FPS selection and cluster assignment
  [:material-github: minhuanli/cryoGMM](https://github.com/minhuanli/cryoGMM)
  [:octicons-book-16: Subset construction tutorial](https://github.com/minhuanli/cryoGMM/blob/master/docs/fps_clustering_tutorial.md)

</div>

Assuming cryoGMM is installed (see [Overview](overview.md#software)):

```bash
python scripts/fps_clustering/fps_clustering.py \
    --traj_path            /data/traj/positions_all_traj.pt \
    --traj_top             /data/traj/top.pdb \
    --output_root          /data/clusters/my_system \
    --alignment_selection  "(resi > 108) and name BB2" \
    --cv                   dist:325,785 \
    --cv                   dist:33,529 \
    --cv_labels            "CV1, dist(A50-C120)" "CV2, dist(G6-A81)" \
    --n_clusters           40 \
    --seeds                42,12345,162,160,70
```

Each `--cv` flag adds one dimension to the CV space, computed on the fly from
the aligned trajectory: `dist:i,j`, `angle:i,j,k`, `torsion:i,j,k,l`, or
`file:path` / `npy:path` to read a precomputed quantity such as an RMSD
trajectory. The full argument list is in the
[subset construction tutorial](https://github.com/minhuanli/cryoGMM/blob/master/docs/fps_clustering_tutorial.md#step-run-the-fps-clustering).

This step is cheap — a few minutes for ~10⁵ frames, mostly spent loading the
trajectory — and needs no GPU.

It writes one directory per set:

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

The `center_*.pdb` files are the structures that [Stage 2](stage3-likelihood.md)
scores against the particle images. The two `.npy` files are what
[Stage 4](stage5-gmm.md) reads to build the mixture model.

!!! note "Where the covariance comes from"
    The local covariance around each representative is derived from
    `cluster_labels.npy` — it is the spread of that cluster's frames, estimated
    in a local PCA basis with a Ledoit-Wolf shrinkage estimator. In practice
    that calculation is done by cryoGMM's `gmm_build.py`, which you run in
    [Stage 4](stage5-gmm.md); it is cached on disk and reused across all
    downstream jobs. Conceptually it belongs to this stage, so it is listed
    among the outputs below.

### Several sets give the error bar

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

### Checking the result

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
