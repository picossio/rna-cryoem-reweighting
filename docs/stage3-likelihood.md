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
  [:material-github: flatironinstitute/cryoSBI_classifier](https://github.com/flatironinstitute/cryoSBI_classifier)

</div>

## cryoSBI

cryoSBI amortized the likelihood calculation by learning a surrogate model of the likelihood function. A
neural network is trained on simulated particle images of the representative
conformations. The imaging parameters like the pose, defocus, noise and neighbouring particles all drawn at
random. After training the model on syntehtic data, it is then run once over the experimental particles. Its output for
each particle is one row of the likelihood matrix. The cost is paid up front
(hours of training per set). On the other hand the likelihood evaluation takes then seconds per dataset, which is what
makes it practical to analyze many datasets.

### Running it

<div class="grid cards" markdown>

- **cryoSBI_classifier** — simulation, training and inference
  [:material-github: flatironinstitute/cryoSBI_classifier](https://github.com/flatironinstitute/cryoSBI_classifier)

</div>

`pip install .` in a clone of the repository (plus `pip install MDAnalysis`)
puts three commands on your path: `make_torch_models`, `train_classifier` and
`classifier_inference`. Configuration is a `conf/` directory of Hydra YAML
files — copy `examples/conf/` from the repository into your project and edit
it. Then, for every set from Stage 1:

**1. Build the model file.** One `.pt` per set, from a comma-separated list of
PDBs. The order of the list is the column order of the likelihood matrix, so
list the representatives in cluster order and put compositional species last:

```bash
PDBS=""
for i in $(seq 0 39); do PDBS+="/data/clusters/my_system/40_clusters/set_0/center_${i}.pdb,"; done
PDBS+="/data/dimers/dimer_conf1/cv-0.pdb,/data/dimers/dimer_conf2/cv-0.pdb"

make_torch_models \
    --pdb_files "${PDBS}" \
    --output_file models/set_0.pt \
    --atom_selection all
```

`--atom_selection all` keeps every bead or atom (any MDAnalysis selection is accepted).

**2. Describe the images.** `conf/simulation.yaml` sets what the simulator
renders. These are the P4-P6 settings:

```yaml
simulation:
  model_file: XXX            # set per run on the command line
  n_pixels: 128              # box size of the particle stack
  pixel_size: 2.007          # Å/px of the particle stack
  sigma: [0.5, 3.0]          # Gaussian width per atom, Å
  shift: 15.0                # max in-plane shift, Å
  defocus: [0.5, 3.5]        # µm
  snr: [0.0005, 0.5]         # sampled log-uniformly
  amp: 0.1                   # amplitude contrast
  b_factor: [1.0, 100.0]
  n_bg_min: 0                # neighbouring particles in the box
  n_bg_max: 3
  padding_factor: 1.25       # canvas for placing neighbours
  exclusion_radius: 0.0
  max_placement_attempts: 500
  garbage_class: true        # adds the junk column
  min_garbage: 5             # a junk image is a pile-up of 5–10 structures
  max_garbage: 10
```

`conf/train.yaml` sets the network and the training budget. We used the
`REGNETY` embedding (128 dimensions) with the `PROTOTYPE` head, and
`epochs: 300`, `batches_per_epoch: 200`, `simulation_batch_size: 512`. That means we simulated about
3×10⁷ images per trained classifier.

**3. Train one classifier per set.**

```bash
train_classifier \
    --config-path "$PWD/conf" --config-name config.yaml \
    simulation.model_file="$PWD/models/set_0.pt" \
    hydra.run.dir=outputs/set_0
```

Images are simulated on the fly on the GPU and never stored. For P4-P6 (43
classes, the budget above) a run took 6.5 h on one A100 with 16 CPU workers.
The five sets are independent jobs. The result is `outputs/set_0/estimator.pt`,
with periodic checkpoints and TensorBoard logs beside it.

**4. Evaluate the particles.**

```bash
classifier_inference \
    --config-path "$PWD/conf" --config-name inference.yaml \
    inference.folder_with_mrcs=/data/particles/J1651/ \
    inference.estimator_weights=outputs/set_0/estimator.pt \
    inference.output_dir=inference/set_0/J1651
```

To evaluate the likelihood per particle we only need a single forward pass through the classifier: 54,000 particles took 37 s on an A100. Repeat
for every dataset and every set of models.

### Output

```
inference/set_0/J1651/
  likelihoods.pt   # (N_particles, N_columns) — the likelihood matrix
  embeddings.pt    # (N_particles, 128) — per-particle embeddings, diagnostics only
```

- **Rows** are particles, in the order of the `.mrc` files (sorted by name)
  and of the particles within each file.
- **Columns** follow `--pdb_files`, then the junk column last. For P4-P6:
  columns 0–39 are the monomer clusters (`center_0` … `center_39`, matching
  `center_idx.npy`), 40 and 41 the two dimer conformations, 42 junk.
- **Values** are the classifier's logits. Softmaxed along a row they are the
  posterior over the columns under the uniform prior used in training, so each
  row is log *p*(image | structure) up to an additive constant that is the same
  across the row. [Stage 3](stage4-reweighting.md) is invariant to that
  constant, so the file is used as it is. Compare values within a row, never
  between rows.

### How it works

Every training batch is drawn from the prior: a model index (uniform over the
columns), a random orientation, in-plane shift, defocus, B-factor, Gaussian
width, SNR and a number of neighbouring particles. To simulate the particle, the atoms are projected,
then the CTF is applied and noise is added. With the model index uniform, a
classifier trained with cross-entropy loss which converges to the likelihood *p*(image | structure)
marginalised over everything else. That marginalisation is what cryoLike computes explicitly for every image. Here it
is *amortised*: learned once, then one forward pass per particle.

### Checking the result

- **Before training**, render a few images and put them next to experimental
  particles at the same box and pixel size:

    ```python
    from cryo_sbi import CryoEmSimulator
    sim = CryoEmSimulator("conf/simulation.yaml", device="cpu")
    images, params = sim.sample_and_simulate(num_sim=8, return_parameters=True)
    ```

    The simulated images should look similar to the experimental particles. The simulator renders
    positive density. `inference.invert_contrast: true` (the default) flips
    stacks stored with negative density, and `inference.whitening: true`
    flattens the noise spectrum before classification.

- **During training**, watch `Accuracy/epoch` in TensorBoard
  (`tensorboard --logdir outputs`). It should plateau, not reach one: on
  P4-P6 it saturated at 0.55 for 43 classes (chance level is 0.02).

!!! note "Compositional species"
    The likelihood matrix can be built with both conformational and
    compositional species. When the sample contains several species — for
    P4-P6, monomer conformations alongside dimers and a junk/noise class with
    cryoSBI — the extra columns are additional mixture components, that will
    be reweighted in [Stage 3](stage4-reweighting.md).

!!! success "Outputs"
    - Image-to-structure likelihood matrix

Next: [3. Ensemble reweighting →](stage4-reweighting.md)
