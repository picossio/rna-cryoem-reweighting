# 1. Coarse-grained sampling

We fix the RNA secondary structure and sample conformational space using a
coarse-grained forcefield, generating a large pool of candidate 3D structures
(or long trajectories) consistent with that secondary structure.

!!! info "Inputs"
    - RNA sequence + secondary structure (see [Inputs](inputs.md))

## Method

Description of the coarse-grained sampling method — the secondary structure is
held fixed while the tertiary fold is allowed to explore conformational space
under a chosen forcefield.

Supported forcefields / tools:

<div class="grid cards" markdown>

- **Martini**
  [:octicons-link-external-16: cgmartini.nl](https://cgmartini.nl/)
- **SPQR**
  [:octicons-mark-github-16: srnas/spqr](https://github.com/srnas/spqr)
- **FARFAR2**
  [:octicons-link-external-16: Rosetta docs](https://docs.rosettacommons.org/docs/latest/FARFAR2)

</div>

!!! success "Outputs"
    - Conformational samples or long trajectories, in PDB format.

Next: [2. Conformational subset construction →](stage2-conformational-subset.md)
