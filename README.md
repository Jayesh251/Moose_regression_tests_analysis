## 1. Totals

Of 260 tests (247 example scripts and 13 jardesigner tutorials), 155 work
and 105 fail. This document covers the **74 failures caused by bugs in
MOOSE, the examples or jardesigner**. The remaining 31 are not bugs (missing
optional packages, examples that wait for keyboard input, test time or memory
limits, and 2 not yet diagnosed) and are not listed here.

| Group | Tests | Fix in |
|---|---|---|
| A. Bugs in MOOSE itself | 11 | moose-core |
| B. Examples out of date with the current MOOSE API | 59 | moose-examples, or a compatibility alias in MOOSE |
| C. Bug in jardesigner | 4 | jardesigner |
| **Total** | **74** | |

---

## A. Bugs in MOOSE itself (11 tests)

### A1. Segmentation fault when a Function controls a reaction rate (3 tests, root cause found)

| Test | Notes |
|---|---|
| `snippets/rxdReacDiffusion.py` | uses `setNumKf` |
| `snippets/funcReacLotkaVolterra.py` | uses `setNumKf` |
| `snippets/stochasticLotkaVolterra.py` | uses `setNumKf` (Gsolve) |

Also affected: moose-core `tests/core/test_ksolve_parallel.py`. Also hidden:
`tutorials/Rdesigneur/ex7.6_func_controls_reac_rate.py` uses `setNumKf`. It
fails earlier on `'install'` (B2) today, and will probably reach this crash
once that is fixed.

**What happens.** MOOSE crashes (exit code 139) when the script sets
`stoich.reacSystemPath`. It crashes in the PyPI 5.0.0 release too.

**Why.** In `ksolve/Stoich.cpp`, function `installAndUnschedFuncReac`, the
line that collects the Function's input pools sits inside `#ifndef NDEBUG`:

```cpp
vector<Id> srcPools;
#ifndef NDEBUG
    unsigned int n =
        ei.element()->getNeighbors(srcPools, funcSrcFinfo);   // fills srcPools
    assert(numSrc == n);
#endif
...
poolIndex[i] = convertIdToPoolIndex(srcPools[i]);              // srcPools is empty
```

- Release builds define `NDEBUG`, so `getNeighbors` never runs.
- `srcPools` stays empty, and the loop reads from an empty vector, which
  crashes.
- The nanobind migration commit (`31069917b`, 2025-12-30) moved the
  `#endif`. Before that, only `unsigned int n =` was debug-only, and the call
  always ran.

**Fix.** Call `getNeighbors` outside the `#ifndef`, and keep only the
`assert` debug-only. The fix has not been built and tested yet.

### A2. Segmentation fault in `unsorted/test_gsolve.py` (1 test, cause unknown)

Crashes with exit code 139 in the dev build and in the 5.0.0 release. It does
**not** use `setNumKf`, so the cause differs from A1.
**Fix:** a debug build plus gdb, to find where it crashes.

### A3. MOOSE's own NeuroML loader uses a field that no longer exists (5 tests)

| Tests |
|---|
| `neuroml/GranuleCell/Granule98.py` |
| `neuroml/GranuleCell/Granule98_hsolve.py` |
| `neuroml/GranuleCell/FvsI_Granule98.py` |
| `neuroml/allChannelsCell/allChannelsCell.py` |
| `neuroml/lobster_pyloric/STG_net.py` |

**Error:** `AttributeError: setFieldGeneric::xminA is not found`

**Why.** The examples don't use `xminA` themselves. MOOSE's own loader
`python/moose/neuroml/ChannelML.py` (line 334) does
`moosegate.xminA = VMIN_here*Vfactor`. The `HHGate` class no longer has
`xminA`; its range field is now `min` (`biophysics/HHGate.cpp`).

**Fix.** Update `ChannelML.py` to the current `HHGate` field names. One fix
in MOOSE repairs all 5.

### A4. Python binding missing argument types (2 tests)

| Test | Error |
|---|---|
| `snippets/synapse.py` | `TypeError: Unsupported lookup type: ObjId,ObjId` |
| `tutorials/Rdesigneur/ex11.1_presyn_dend.py` | `TypeError: Unsupported DestFinfo type: vector<ObjId>,double` |

**Why.** These MOOSE fields and functions exist in C++, but the new nanobind
Python binding doesn't support those argument types yet, so Python can't
call them.

**Fix.** Add these type combinations to the binding in moose-core's
`pymoose` code.

---

## B. Examples out of date with the current MOOSE API (59 tests)

MOOSE changed or removed something, and these examples weren't updated. Each
row is one fix.

| # | Cause (error) | Tests | What changed | Fix |
|---|---|---|---|---|
| B1 | `TypeError: type 'nanobind.nb_type_0' is not an acceptable base type` | 17 | `traub_2005/py/channelbase.py` defines `class ChannelMeta(moose.melement.__class__)`, a metaclass derived from MOOSE's class type. nanobind doesn't allow this (nanobind switch, 2025-12-30). Plain subclassing still works. | Rewrite `channelbase.py` without the custom metaclass. One file fixes all 17. |
| B2 | `ValueError: 'install' is not in list` | 16 | rdesigneur removed the `'install'` option for chemical distributions (Upi, 291e22af0, 2024-12-12) | Update `chemDistrib` in each example to the current syntax |
| B3 | `moose.utils` has no attribute `connect_CaConc` (6), `setDefaultDt` (2), `resetSim` (1) | 9 | Helper functions removed from `moose.utils` (`connect_CaConc` in 73e72ec26, 2026-03-22) | Update the examples, or restore the helpers in `moose.utils` |
| B4 | `moose` has no attribute `mooseReadNML2` | 2 | NeuroML2 loading function renamed or removed | Use the current NeuroML2 loader |
| B5 | `setFieldGeneric::tau is not found` (2), `tabFillExpr is not found` (1) | 3 | Old `HHGate` field and function names in the ephys examples | Update to the current `HHGate` interface |
| B6 | `setFieldGeneric::ndata is not found` | 1 | Field removed from the synapse class | Update the example |
| B7 | `getDataIndex is not found` (did you mean `dataIndex`) | 1 | Old getter name | Use `dataIndex` |
| B8 | `getField is not found` | 1 | Old method on the info object | Update to current field access |
| B9 | `No field ... 'setNumVars'` | 1 | Function now works out the number of variables from the expression (`setNumVar` is deprecated) | Remove the `numVars` call |
| B10 | `start(): incompatible function arguments` (called with two numbers) | 1 | `moose.start(runtime, notify=False)`: the second argument is now a flag | Call `moose.start(runtime)` |
| B11 | `moose.vec.__init__(): incompatible arguments` | 1 | New signature `vec(path, n=1, dtype='')` | Update the call |
| B12 | `search() got an unexpected keyword argument 'model_id'` | 1 | ICG channel database search changed (recent commit 52f76f72f) | Update the call |
| B13 | Morphology name not in the registry (`morph.get('traub91_CA1')`) | 1 | Registry names changed | Use a current name from the list |
| B14 | `ValueError: object does not exist: /library/osc/kinetics/a` | 2 | The example expects the oscillator model at `/library/osc/kinetics/...`. Most likely rdesigneur now builds the prototype at a different path. | Find the new path and update |
| B15 | `SquidSetup.schedule() takes 1 positional argument but 2 were given` | 1 | The example's own helper changed but the caller didn't | Fix the call inside the example |
| B16 | `'SquidAxon' object has no attribute 'C'` | 1 | `traub_2005/py/vclamptest.py` uses an old version of the squid example's class | Update the example |

Tests in each row:

- **B1 (17):** `traub_2005/py/` `archan`, `cachans`, `cell_test_util`, `cells`,
  `channel_test_util`, `channelbase`, `channelinit`, `display_morphology`,
  `dump_f_i_curves`, `fig_a2_fs`, `fig_a3`, `kchans`, `nachans`,
  `test_hsolve_tcr`, `test_singlecomp`, `test_tcr`, `testutils`.
- **B2 (16):**
  - `tutorials/Rdesigneur/`: `ex6.0_chem_osc`, `ex7.0_spatial_chem_osc`,
    `ex7.1_diffusive_gradient`, `ex7.2_CICR`, `ex7.3_simple_transport`,
    `ex7.6_func_controls_reac_rate`, `ex8.0_multiscale_KA_phosph`,
    `ex8.1_synTrigCICR`, `ex8.2_multiscale_gluR_phosph_3compt`,
    `ex8.3_spine_vol_change`, `ex8.4_3d_synTrigCICR`,
    `ex10.1_spine_Ca_influx_to_dend`,
    `ex10.2_spine_Ca_influx_to_branched_neuron`, `ex13.0_BCM_chem`;
  - `paper-2015/Fig6_NetMultiscale/ReducedModel.py`;
  - `parallelSolver/Fig2_v4.py`.
- **B3 (9):**
  - `neuroml/CA1PyramidalCell/` `CA1`, `CA1_hsolve`, `FvsI_CA1`;
  - `neuroml/LIF/` `FvsI_LIF`, `LIFxml_firing`, `twoLIFxml_firing`;
  - `snippets/Izhikevich_with_synapse`, `snippets/gapjunction`,
    `snippets/traub_naf`.
- **B4:** `neuroml2/run_cell`, `neuroml2/run_hhcell`.
- **B5:** `ephys/granule_cell`, `ephys/granule_cell_hhchannelf`,
  `ephys/granule_cell_hhchannel2`.
- **B6:** `snippets/compartment_net`. **B7:** `snippets/nsdf`.
  **B8:** `snippets/savemodel`. **B9:** `snippets/changeFuncExpression`.
- **B10:** `snippets/diffusion_using_Gillespie`.
  **B11:** `paper-2015/Fig2_elecModels/Fig2A`.
  **B12:** `snippets/icg_channel_registry`.
  **B13:** `snippets/morphology_registry`.
- **B14:** `tutorials/Rdesigneur/ex7.4_travelling_osc`, `ex7.5_bidirectional_transport`.
- **B15:** `squid/squid_setup`. **B16:** `traub_2005/py/vclamptest`.

**Biggest wins:** fixing B1 (one file, 17 tests) and B2 (16 tests) repairs 33
of the 59.

---

## C. Bug in jardesigner (4 tests)

| Tests |
|---|
| `Tut4_synapse_sum` |
| `Tut5_summation_thresh` |
| `Tut7_Ca_influx` |
| `cTut6_CICR` |

**Error:** `NameError: name 'absoluteTemplatePath' is not defined`

**Why.** These tutorials have a 3D view. jardesigner's standalone-output code
looks for `jardes3Dtemplate.html`, which is not in the repository. The error
handler then prints a variable that doesn't exist, so a second error hides
the first. The simulation and plots finish before the error.

**Fix.** Add the template file to the repository (and the package), and
correct the variable name in the error handler.

---

## 2. Bugs in tests that run but give unreliable numbers

These examples run without error, so they are not counted among the
failures above. But their numbers change between runs, so they can only be
smoke tests. The causes are bugs in MOOSE or in the examples.

### MOOSE bugs

| Test | Problem |
|---|---|
| `tutorials/ChemicalBistables/mapkFB.py` | Sometimes garbage values (up to 1e+158) when the GSL solver fails. In `VoxelPools::advance` the error is printed and followed by `assert(0)`, which release builds remove, so the simulation continues with bad numbers. 13 of 80 runs on the dev build, 1 of 80 on the PyPI wheel. Quarantined. |
| `snippets/loadKineticModel.py` | Same GSL failure, seen once. Quarantined. |
| `snippets/helloMoose.py` | Values that should be 0 differ every run (1e-322 to 1e+226). Suspected uninitialised memory. Quarantined. |
| `snippets/interpol.py` | Same pattern, at 1e-311. Quarantined. |
| `snippets/switchKineticSolvers.py` | Differs between runs (1e-316 to 1e-9). Quarantined. |
| `snippets/HigginsGraupnerBrunel2014_LifetimeCaPlasticity.py` | Differs between runs even though it calls `moose.seed(100)`. Not explained yet. |

### Example bugs: random numbers without a seed

| Test | Problem |
|---|---|
| `snippets/RandSpikeStats.py` | MOOSE random numbers without `moose.seed()` |
| `tutorials/ChemicalBistables/doseResponse.py` | `randomInit()` without `moose.seed()` |
| `snippets/randomspike.py` | numpy random without a seed |
| `snippets/compartment_net_no_array.py` | numpy random without a seed |
| `tutorials/Rdesigneur/ex5.0_random_syn_input.py` | random synaptic input without a seed |
| `tutorials/Rdesigneur/ex9.2_spines_in_neuronal_morpho.py` | different number of spines each run (1220 vs 1325) |

**Fix.** Add seeds, so these become numeric tests.

---

## 3. Suggested order of fixing

1. **A1 segfault:** a small, clear fix in `Stoich.cpp` that unblocks at least
   3 tests and moose-core's own test. Build, test and send a PR.
2. **B1 and B2:** 33 tests fixed with few changes (one file for B1).
3. **A3 NeuroML loader:** one file in MOOSE fixes 5 tests.
4. **The GSL error handling in `VoxelPools::advance`:** stop or raise an
   error instead of continuing with garbage. Then investigate the
   intermittent failure with valgrind.
5. **The rest of group B, then C, A4 and A2.**
6. **Rebuild the suite after each batch.** Every fixed example becomes a
   real test; `compare.py` reports it as FIXED.
