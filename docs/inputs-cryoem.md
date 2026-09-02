# CryoEM particles

Experimental particle images from cryo-electron microscopy. Each particle
is a single, noisy 2D projection of one RNA molecule in some (unknown)
conformation, at some (unknown) pose (orientation and location). Across
thousands of particles,
the full set implicitly encodes the ensemble of conformations present in
the sample — but not directly; that's what the rest of the pipeline
recovers.

!!! warning "Requirements"
    Particles should be an unclassified, independent and identically
    distributed (i.i.d.) sample of the underlying distribution. Do not use
    classified or otherwise filtered particle sets.

!!! info "Input"
    - CryoEM particle images
