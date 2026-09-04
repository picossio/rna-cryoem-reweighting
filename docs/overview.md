# Overview

CryoEM gives you thousands of noisy 2D images of individual RNA particles, each
a snapshot of the molecule in some conformation. On its own, that data doesn't
tell you which conformations are present or how common each one is. Separately,
you can use physics-based simulations to generate a large pool of *plausible*
RNA shapes, consistent with a known sequence and secondary structure. This
method's job is to connect the two: figure out which of those simulated shapes
best explain the experimental images, and in what proportions.

That's done in several stages, starting from the inputs:

**Inputs**

- **CryoEM images** — thousands of noisy 2D snapshots of individual RNA
  particles.
- **Prior ensemble** — a large pool of candidate 3D conformations
  generated from coarse-grained (CG) sampling with fixed secondary structure.

**Pipeline stages**

1. **Conformational subset construction** — reduce the large pool of CG
   conformations down to a manageable, representative subset of structures,
   and estimate the local covariance for each one.
2. **Likelihood** — score how well each representative conformation explains
   the experimental cryoEM particle images.
3. **Ensemble reweighting** — turn those scores into a set of weights: how much
   each conformation is populated.
4. **Gaussian mixture model** — use the weighted, representative conformations
   (plus their covariance) to build a smooth probability distribution over
   RNA shape space, and draw a full conformational ensemble from it.

![Pipeline overview](assets/images/Diagram.png){ .centered-figure width="320" }

## Software

Stages 1, 3 and 4 all run out of a single package, **cryoGMM**. Install it once
before you start:

```bash
git clone https://github.com/minhuanli/cryoGMM.git
cd cryoGMM
pip install -e .
```

That pulls in `numpy`, `torch`, `mdtraj`, `scikit-learn`, `fpsample`, `tqdm`
and `matplotlib`. Stages 1 and 3 run comfortably on a laptop; Stage 4 expects a
GPU by default (`--device cuda:0`).

Stage 2 uses a separate likelihood package — either
[CryoLike](https://github.com/flatironinstitute/CryoLike) or
[cryoSBI](https://github.com/flatironinstitute/cryoSBI) — each with its own
installation instructions; see that stage's page.

Generating the prior ensemble in the first place needs a coarse-grained
sampling package, which is outside this pipeline; see
[Prior ensemble](inputs-cg-sampling.md).

## Where to go next

Each stage has its own page — see [CryoEM particles](inputs-cryoem.md) and
[Prior ensemble](inputs-cg-sampling.md) for what feeds the pipeline,
then walk through [Stage 1](stage2-conformational-subset.md) onward, and
[Outputs](outputs.md) for what you get out the other end.
