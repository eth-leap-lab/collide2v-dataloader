# Plan & design notes

_Started 2026-11-18. This is a living document. Decisions get recorded here as we make them._

## 1. Requirements

From the project description:

- Select **processes** (dataset folders) and **features** (parquet columns).
- Handle jagged/nested awkward arrays.
- **Streaming** loader, usable in a variety of ML projects. Configs should be easy to write.
- Ship together with the dataset locations. Support EOS now and Hugging Face (HF) later.

From the FM-dataset meeting:

- **Balanced loader**: equal events per class. For DY and QCD, combine the sub-sample
  (histogram/bin) folders according to each folder's cross section.
- **Cross-section loader**: the user gives only the total number of events, and each
  event is sampled according to σ. The aim is an SM sample that is as unbiased as possible,
  so gen-level cuts have to match across processes.
- **Flat-pT loader**: the requested number per process, flat in pT (QCD, DY). Histogram the
  pT, normalise by bin population, and make every pT range equally represented.
- Streaming from HF or wherever the data is stored.
- Look at the sampling in the **ParT training backend** (`weaver-core`, which the notes call
  "vivarcore"). It has a reweighting/sampling scheme driven by `weights:` in its YAML
  data config, with flat reweighting in 2D histogram bins. To check.

## 2. What the data looks like

Findings from a read-only inspection on 2026-11-18.

- 49 process folders under `production_final/`. Some are still being produced (e.g. VBFH*, HH_*).
- Files: `<process>-NEVENT10000-RS<seed>.parquet`, ~10k events, **~2.1 GB each**
  (~200 kB/event). There are ~100–3000 files per process, so the full dataset is O(100 TB).
- **One row group per file** (~3.1 GB uncompressed). Column chunks can be read on their own,
  but a file can't be split into smaller row ranges. This affects streaming granularity,
  shuffling, and HF streaming. Should we re-chunk for the HF upload?
- 271 top-level columns, **all `large_list`**. Per-event scalars (MET, HT, Event_*)
  are length-1 lists. Doubly nested columns: `*_Constituents`, `*_ConstituentsIdx`,
  `Vertex_Constituents`.
- Column prefixes: `FullReco_*`, `L1T_*` (same object set), `Gen_*`, `Vertex_*`, `Event_*`.
  Kinematics are stored as `halffloat`.
- Parquet key-value metadata: `dataset_version=collide2v_v1.0`, `nEvents`, `source_root`.

Odd things that need checking:

- `Event_CrossSection` is **not** a constant per-process σ. It looks like a running/cumulative
  value, it isn't monotonic, and its values are multiples of the per-event weight.
- `Event_Weight` is 0 for ~2/3 of events in `ggHbb/...RS13000001` (MLM matching?
  merging?).
- `Event_Number` goes up in steps of 20 and resets inside a file. Were several chunks merged?
- `ggHbb/...RS13000001` has `source_root = .../WJets_delphes_events.root`. Is that just a
  naming leftover in the pipeline, or a mislabelled file?
- There is no DY folder and no HT/pT-binned QCD folder yet (only `QCD_HT50tobb`).

## 3. Open questions

### Answered / partly answered (2026-11-20)

1. **Cross sections**: they are probably *not* in the dataset itself. They are likely in the
   MadGraph process cards in the simulation repo, on whichever branch matches the current
   production. Candidate branches: `MG5_cards`, `process-card-fixes`, `update-patch-madgraph`
   (26 branches in total). **Still to find.** Until then, cross sections go in a hand-maintained
   `xsec.yaml` that ships with the package, so the sampling code doesn't depend on where they
   come from.
3. **DY / QCD bins**: some may already exist and more are coming "in the coming days". The
   bin set is not fixed yet, so the catalog has to find bins automatically instead of
   hard-coding them.
6. **Flat-pT**: *pT of what* is still unclear. Philip will ask. It might just mean reweighting
   across the binned files. Keep it behind a pluggable "sampling variable" until then.
7. **Output format**: **padded tensors + masks** for ML/preprocessing workflows. (Details in §4.)
8. **HF**: layout TBD, not uploaded yet. **Focus on EOS now**, behind a storage abstraction
   so HF can be added as another backend later.

### Still open

2. What do `Event_Weight` and `Event_CrossSection` mean exactly (see §2)? Needs someone from
   the production side.
4. **Gen cuts**: are they consistent across processes (needed for the SM cocktail)? Probably
   answered by the same cards as Q1.
5. **Classes vs processes**: who defines the class mapping?
9. **Train/val/test splits**.
10. **EOS access**: FUSE vs XRootD.
11. Framework targets, Python version, packaging, license.

## 4. Rough architecture (draft, to discuss)

- `catalog`: file index per process (path, nEvents, σ, bin info). Built once and cached.
  Source can be EOS or HF.
- `config`: YAML → dataclasses. Fields: processes/classes, features, sampling mode, N, seed, split.
- `sampler`: turns a config + catalog into a list of (file, event-index) draws per epoch.
  There is one sampler per mode: balanced / xsec / flat-pT.
- `reader`: column-projected parquet reads (pyarrow + fsspec), with per-file event selection
  and prefetching.
- `formatter`: awkward → padded numpy/torch + masks, or stays awkward.
- `torch` adapter: `IterableDataset` with worker/rank sharding.

## 5. Decision log

| Date | Decision | Why |
|------|----------|-----|
| 2026-11-18 | Repo initialised. No package skeleton yet | The layout depends on the open questions above |
| 2026-11-20 | Remote: github.com/eth-leap-lab/collide2v-dataloader | — |
| 2026-11-20 | Primary output is padded tensors + masks | Ready to use for ML/preprocessing |
| 2026-11-20 | EOS first, behind a storage-backend interface. HF is added later | HF upload not done yet |
| 2026-11-20 | Cross sections come from a shipped `xsec.yaml`, not from parquet columns | The per-event σ columns aren't usable (§2) |
