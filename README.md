# collide2v-dataloader

A configurable, streaming dataloader for the **COLLIDE-2V** particle-physics dataset.
It is meant to be distributed with the dataset so users can start ML work right away.

> Status: **planning**. No code yet. See [docs/PLAN.md](docs/PLAN.md).

## Goals

- **Process selection**: pick physics processes (one folder per process in the dataset).
- **Feature selection**: pick individual columns. These are jagged/nested awkward
  arrays (per-event lists, and per-jet constituent lists inside them).
- **Streaming** from EOS (local/FUSE or XRootD) and from the Hugging Face copy of the dataset.
- **Sampling modes**:
  1. *Balanced*: equal number of events per class. Binned samples (e.g. DY, QCD) are combined
     within a class according to each bin's cross section.
  2. *Cross-section weighted*: the user gives a total number of events and processes are sampled
     in proportion to σ. This gives a minimally biased "SM cocktail" (needs consistent gen cuts).
  3. *Flat in pT*: a requested number of events per process, reweighted or resampled so the
     pT distribution is flat (mainly for QCD and DY).
- **Simple configs** (YAML) that describe a loading setup in a reproducible way.

## Data locations

- EOS: `/eos/project/f/foundational-model-dataset/samples/production_final/<process>/*.parquet`
- Hugging Face: TBD
- Production code: https://github.com/fermilab-hep-ai/simulation
