# Skaar et al. (2025): code-to-model mapping

Source: upstream Git commit `68e6dd970cfc6bab26459fcb8c34ee0f16560d9e`,
inspected on 2026-09-15. Line numbers below refer to that immutable commit.
Paper: [PMC full text](https://pmc.ncbi.nlm.nih.gov/articles/PMC12417261/),
especially Methods 2.1, 2.3.3, Results 3.3, Figure 5, and Table 3.

| Item | Restricted/aggregated Brian2 | Unrestricted/explicit Brian2 | NEST |
| --- | --- | --- | --- |
| Script/model | `brian_benchmark.py` | `brian_benchmark_explicit.py` | `nest_benchmark.py` calls `iaf_bw_2001` (approximate) then `iaf_bw_2001_exact` (original exact) |
| Evidence | Lines 122–142 create one diagonal spike path, a one-neuron sum group, and `summed` broadcasts | Lines 104–109 define two nonlinear NMDA ODEs on each E→E/E→I synapse; lines 127–135 connect all pairs | Lines 212–213 select both named models; paper Methods 2.2 identifies their scientific meaning |
| Integrator | Brian2 `method='rk4'` for NeuronGroups/Synapses; fixed `dt=0.1 ms` | Same, including `clock-driven` synaptic ODEs | `iaf_bw_2001` and `_exact` use adaptive RKF45 internally, spikes resolved on 0.1 ms kernel intervals (paper Methods 2.3.3) |
| Network | All-to-all recurrent E→E, E→I, I→E, I→I, with autapses: `connect()` has no exclusion | Same; each E pair has one AMPA and one NMDA synapse | All-to-all, with collocated AMPA/NMDA synapses (lines 172–191) |
| Delays | Recurrent event pathways fixed at 0.5 ms; NMDA aggregate `summed` variables update by schedule and are not delayed pairwise | All recurrent event pathways fixed at 0.5 ms, including per-edge NMDA `x += 1` | Recurrent 0.5 ms; external generator connections 0.1 ms (lines 83, 144–155) |
| Inputs/seeds | One `PoissonInput` per recurrent neuron, `N=1`, 2400 Hz, E weight 2.1 nS, I weight 1.62 nS. No `seed()` call | Same | One 2400 Hz generator connected to all neurons, `rng_seed=runner_id+1`, reset for both models (lines 64–92, 212–213) |
| Monitors | E and I `PopulationRateMonitor` | Same plus `StateMonitor(popE, True, 1)` and I equivalent: all available state variables of neuron index 1 (lines 159–164) | E and I spike recorders |

The restricted Brian2 model is **the original nonlinear NMDA formulation under
the special fully connected/equal-delay assumption**, not the paper's new
approximation. The explicit Brian2 model is the **original nonlinear NMDA
formulation with per-synapse state**, suitable for arbitrary connectivity.
It does not use the NEST approximation. The paper names these the restricted
and unrestricted Brian2 implementations in Methods 2.3.3/Figure 5.

## State and synapse representation

Restricted E neuron declares nine stored fields: `label` (constant population
label), `V` (membrane voltage), `s_AMPA` (recurrent AMPA conductance),
`s_NMDA_tot` (dimensionless NMDA input sum), `s_GABA` (GABA conductance),
`s_AMPA_ext` (background AMPA conductance), `I_input` (extra current, zero by
default), `s_NMDA` (nonlinear presynaptic gate), and `x` (fast NMDA rise state).
It also defines four derived currents `I_AMPA`, `I_NMDA`, `I_GABA`,
`I_AMPA_ext` (lines 66–86). Restricted I neurons store `V`, `s_AMPA`,
`s_NMDA_tot`, `s_GABA`, `s_AMPA_ext`, and the same four derived currents
(lines 88–102). An auxiliary one-neuron group stores `s`. E→E NMDA diagonal
events increment `x_pre`; E→auxiliary `summed` connections sum `s_NMDA_pre`;
auxiliary→E/I `summed` connections broadcast the total. Those auxiliary
connections are real Brian2 Synapses and must be retained in any replay.

Explicit E neurons store seven declared fields: `label`, `V`, `s_AMPA`,
`s_GABA`, `s_AMPA_ext`, `I_input`, and `s_NMDA_tot`; I neurons store five:
`V`, `s_AMPA`, `s_GABA`, `s_AMPA_ext`, and `s_NMDA_tot`. Both define four
derived currents. Each E→E and E→I NMDA synapse stores `x` (rise),
`s_NMDA` (nonlinear gate), and `w_NMDA` (dimensionless weight, set to 1).
The `summed` synaptic equation `s_NMDA_tot_post = w_NMDA*s_NMDA` evaluates
the postsynaptic total each timestep. E→E AMPA stores `w` in siemens.
Other recurrent AMPA/GABA paths use scalar conductances in their event code.
The distinct AMPA and NMDA Synapses are paired, never merged.

The NEST script does not expose its full internal state schema. It configures
AMPA/GABA/NMDA time constants, NMDA rise/decay/alpha, leak, voltage,
refractory parameters and Mg concentration in each neuron; the exact model's
NMDA gates are associated with its receptor-specific connections. A precise
field count requires inspecting the NEST 3.8 model implementation, so it is
**not asserted** from this Python wrapper alone.

## Scale, initial conditions, and timing

Both Brian scripts calculate `N=2560*scale`, `NE=int(0.8*N)`,
`NI=int(0.2*N)`; scripts use `scale=1,2,4,8` for the paper's 2,560 to
20,480 range. Job files also run `scale=0.25,0.5`, which are smaller
pre-paper points, and request eight OpenMP threads. For `scale=1`, explicit
Brian2 creates 2,048 E and 512 I neurons; recurrent synapse counts are
4,194,304 E→E AMPA, 1,048,576 E→I AMPA, 4,194,304 E→E NMDA,
1,048,576 E→I NMDA, 1,048,576 I→E GABA, and 262,144 I→I GABA:
**11,796,480** in total. Both E and I voltages start at −70 mV, and
`s_NMDA_tot` starts at zero; unassigned state arrays use Brian2 defaults.
Recurrent conductances scale as 1600/NE or 400/NI, while external weights
remain fixed. `runtime=1000 ms` gives 10,000 ticks. Brian2's default
floating dtype is float64; the upstream script does not request float32.

Both Brian scripts call `run(runtime)` to construct/queue work, then
`device.build(..., run=False)` to generate and compile standalone C++, and
time **only** `device.run()` around `time.time()` (restricted lines 174–187;
explicit lines 170–183). Their CSV omits model construction, code generation,
compilation, postprocessing and total wall time. NEST times `nest.Simulate`
and then spike-count retrieval within the timed region (lines 195–206),
excluding network construction. These regions are not automatically fair
end-to-end comparisons. The paper Figure 5/Table 3 report simulation times
on JURECA/AMD EPYC 7742; they are reference values, not speedup denominators
for other hardware.

## Source discrepancies that affect validation

The paper's Table 2 lists 250 pF inhibitory capacitance, 2 ms AMPA/GABA
time constants and 2/1 ms refractory periods, whereas all three benchmark
scripts use 200 pF inhibitory capacitance, 5 ms GABA and Brian2
1.9/0.9 ms refractory (NEST 2/1 ms). The benchmark **code** is the execution
source of truth; these differences must remain visible. Brian2's unseeded
PoissonInput and NEST's one shared Poisson generator also preclude identical
stochastic replay without a separately documented input protocol.
