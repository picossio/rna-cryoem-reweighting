# Prior ensemble

We fix the RNA secondary structure and sample conformational space using a
coarse-grained forcefield, generating a large pool of candidate 3D structures
(or long trajectories) consistent with that secondary structure.

!!! warning "Requirements"
    The prior ensemble should sample the full conformational space, capturing
    all conformational changes relevant to the data.

## Method

Several CG sampling methods can be used to generate the prior ensemble,
typically starting from a PDB structure (for example, a cryo-EM
reconstruction), fixing the secondary structure, and allowing the tertiary
fold to explore conformational space under a chosen forcefield. We reference
several CG methods below, but do not provide implementation details here.

Supported forcefields / tools:

<div class="grid cards" markdown>

- **Martini**
  [:octicons-link-external-16: cgmartini.nl](https://cgmartini.nl/)
- **SPQR**
  [:octicons-mark-github-16: srnas/spqr](https://github.com/srnas/spqr)
- **FARFAR2**
  [:octicons-link-external-16: Rosetta docs](https://docs.rosettacommons.org/docs/latest/FARFAR2)

</div>

!!! info "Inputs"
    - Conformational samples or long trajectories, in PDB format.

Next: [1. Conformational subset construction →](stage2-conformational-subset.md)
