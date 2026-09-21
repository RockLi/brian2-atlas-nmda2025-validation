# Concise validation results

## Scope

The evaluation uses the explicit/general Brian2 benchmark from Skaar, Haug and
Plesser (2025), upstream commit
`68e6dd970cfc6bab26459fcb8c34ee0f16560d9e`. The primary CPU contract is
float64, fixed-step RK4 with `dt = 0.1 ms`, one second of biological time and
the original connectivity, delays, state variables and monitoring scope.

## Scientific validation

Identical-event comparisons passed at 640, 2,560, 5,120 and 10,240 neurons. At
10,240 neurons, the maximum voltage error was `8.33e-17 V` and the maximum NMDA
gate error was `3.72e-15`; discrete outputs satisfied the deterministic gate.
The 20,480-neuron run retained all 754,974,720 synapses and 28 finite public
output fields. The deterministic claim ends at 10,240 neurons.

## Unified execution matrix

All reported execution modes are shown in one table. A dash means that the
mode was unsupported or that no comparable run was recorded for that exact
host, allocation, workload and numerical contract. Times are seconds.

| Host and allocation | Neurons | Contract and timing scope | Brian2 CPU | Atlas CPU | Atlas MPI | Atlas CUDA | Atlas Metal |
| --- | ---: | --- | ---: | ---: | ---: | ---: | ---: |
| Linux EPYC, 8 pinned cores | 2,560 | float64, compiled-region median | 69.942 | 61.998 | — | — | — |
| Linux EPYC, 8 pinned cores | 5,120 | float64, compiled-region median | 289.100 | 194.660 | — | — | — |
| Linux EPYC, 8 pinned cores | 10,240 | float64, compiled-region median | 1,256.384 | 598.964 | — | — | — |
| Linux EPYC, 8 pinned cores | 20,480 | float64, compiled-region median | 4,443.328 | 2,271.710 | — | — | — |
| Apple Silicon Mac Studio, 8 threads | 2,560 | float64, compiled-region median | 72.823 | 42.909 | — | — | — |
| Apple Silicon Mac Studio, 8 threads | 10,240 | float64, compiled-region median | 756.458 | 552.947 | — | — | — |
| Linux EPYC, 40 physical cores | 10,240 | float64, simulation only | — | 257.774 | 153.675 | — | — |
| Linux cluster, 40 ranks on 1 node | 20,480 | float64, simulation only | — | — | 588.812 | — | — |
| Linux cluster, 40 ranks on 4 nodes | 20,480 | float64, simulation only | — | — | 586.537 | — | — |
| NVIDIA L4 | 5,120 | float32, warm median | — | — | — | 158.144 | — |
| NVIDIA A100-SXM4-40GB | 5,120 | float32, warm median | — | — | — | 189.966 | — |
| Apple M3, 8 GPU cores | 640 | float32, optimized warm median | — | — | — | — | 20.321 |
| Apple M3, 8 GPU cores | 2,560 | float32, optimized warm median | — | — | — | — | 110.802 |

The first four rows give matched Linux CPU speedups of 1.13x, 1.49x, 2.10x
and 1.96x, with Atlas peak-RSS changes of +5.5%, +7.3%, +7.9% and +8.3%.
On the same 40 physical Linux cores at 10,240 neurons, MPI is 1.68x faster than
shared-memory workers. Fixed-rank placement across four nodes provides almost
no improvement at 20,480 neurons because communication and synchronization
occur at every timestep.

CUDA and Metal use a separate float32 scientific contract and are not compared
as speedups over the float64 CPU rows. The Metal values include a general
sparse event-delivery improvement and preserve the frozen public outputs byte
for byte. Paper timings from different hardware are not used as denominators.

## Functional network check

The authors' NEST decision-network fixture was reproduced with 400 matched
exact/approximate trials at each of five coherence values: 2,000 pairs and
4,000 simulations. Both models reproduced the rising decision trend and
sustained selective activity. This comparison validates functional behaviour;
the exact/approximate NEST cost ratio is not a Brian2-Atlas engine speedup.

![Decision probability across five coherence levels.](figures/decision_psychometric_400_20260920.png)

## Why this result is useful

The authors' approximation addresses the high scientific cost of explicit NMDA
state. This validation asks a complementary systems question: how much of that
cost can be reduced by a new execution backend while preserving the original
explicit/general Brian2 formulation? The full-scale reproduction, deterministic
numerical gates and matched-resource CPU measurements make this useful as an
independent external validation rather than a replacement for the paper's
scientific contribution.

The strongest engine claims are the matched eight-core CPU results: 2.10x at
10,240 neurons and 1.96x at 20,480 neurons. Dividing the 8-core Brian2 time at
20,480 neurons by the 40-rank Atlas MPI time gives a 7.55x cross-configuration
observation, but it is not reported as a pure MPI speedup because both resources
and timing scope differ. The matched same-host MPI result is 1.68x at 10,240
neurons on the same 40 physical cores. The near-flat one-node to four-node result
at 20,480 neurons is also informative: timestep-level communication, rather
than available compute, limits this fixed-rank placement experiment.

## Interpretation limits

- Deterministic same-event validation ends at 10,240 neurons.
- CUDA and Metal use float32 and remain separate from the CPU comparison.
- Multi-node measurements hold total MPI ranks fixed and test placement rather
  than increasing-resource strong scaling.
- The NEST fixture uses different simulator semantics and timing scope.
- This package is a result summary, not a backend source release or a complete
  independent reproduction environment.
