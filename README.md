# Brian2-Atlas NMDA 2025 validation

This repository contains a concise, shareable result summary for an external
validation of Brian2-Atlas on the published explicit/general NMDA workload from
Skaar, Haug and Plesser (2025).

The scientific model was kept recognisable as the authors' Brian2 model. The
benchmark does not replace the explicit formulation with the paper's NMDA
approximation.

## Highlights

- Deterministic float64 validation passed through 10,240 neurons.
- The complete 20,480-neuron configuration retained 754,974,720 synapses.
- Matched eight-core Linux runs measured 1.13x to 2.10x CPU speedups.
- A unified matrix aligns CPU, MPI, CUDA and Metal results by tested setup.
- Same-host MPI outperformed shared-memory workers at 10,240 neurons, while
  fixed-rank multi-node scaling saturated.
- CUDA and Metal were evaluated under a separate float32 contract.
- 2,000 matched NEST decision-network pairs reproduced the expected functional
  trend and sustained selective activity.

See [RESULTS.md](RESULTS.md) for the short result note. Source identity and
scientific mapping are recorded in [SOURCE.md](SOURCE.md) and
[model_mapping.md](model_mapping.md). Machine-readable summaries are under
[`results/`](results/).

This is a preliminary exchange package. It excludes the manuscript, backend
source, complete raw archive and operational tooling. The upstream authors'
code is not vendored because the recorded commit has no license file; use
`scripts/fetch_upstream.sh` to obtain it directly.

Verify the included files with `shasum -a 256 -c MANIFEST.sha256`.

No license is granted for Brian2-Atlas implementation material by this package.
The cited paper is available under CC BY 4.0 from its publisher/PMC; the
upstream code repository must be treated according to its own license status.
