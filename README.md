# NISQ-Aware Quantum Kernel SVM for Network Intrusion Detection

Code, frozen protocol files, per-run results and audit scripts for the manuscript
*NISQ-Aware Quantum Kernel SVM for Network Intrusion Detection: A Regime-Specific Benchmark on
NSL-KDD* (IEEE Transactions on Emerging Topics in Computing, manuscript TETC-2026-05-0252,
revision 1). Release tag `tetc-r1` marks the exact state the manuscript reports.

The question the study asks is not *whether* a quantum kernel beats classical learners on
intrusion detection, but *where* it does, and whether that region is worth the circuit cost.

```
NSL-KDD (41 features) -> one-hot (122 columns) -> SelectKBest (K=20) -> PCA (n*=4)
   -> min-max to [0, pi] -> ZZFeatureMap (4 qubits, r=2, full entanglement) -> SVC on the precomputed Gram matrix
```

Every classical baseline (linear, poly-2 and RBF SVMs, random forest, XGBoost) consumes the same
four-component representation as the circuit; the paired protocol isolates the kernel, not the
pipeline. A separate reference arm (Table IV of the manuscript) trains XGBoost and the random
forest on all 122 one-hot features under the same subsets, seeds and test set.

---

## Reproduce the manuscript's numbers without re-running the experiments

The per-run results of every experiment are committed under `results/`. Four audit scripts
recompute every published statistic, every plotted number and every number written in the prose
from those raw records. `audit_c4.py` does **not** call the pipeline's own statistics functions:
it re-implements the paired tests with SciPy and compares, because using the code that produced a
number to verify that number lets shared errors through. These audits caught four real errors in
the revision code before release, one of which had changed the selected `n*` from 4 to 5.

```bash
uv sync                           # dependencies pinned in pyproject.toml / uv.lock (or: pip install -e .)
python runners/audit_c4.py        # 100/100  paired statistics and the 110-cell regime map
python runners/audit_figures.py   #  36 items  every plotted number
python runners/audit_prose.py     # 123/123  every number written in the text, incl. Table IV and the release tag
python runners/verify_lemma1.py   #  15/15   second-order expansion of the ZZ kernel (Lemma 1)
python runners/check_latex.py     #          structure of the .tex sources
```

Notes on what the audits print:

- `audit_c4.py` reports 94/100 with SciPy 1.13: three Wilcoxon p-values with ties differ from
  the SciPy 1.17 values by up to 0.10, and no verdict changes. With SciPy 1.17 it reports 100/100.
- `audit_figures.py` reports 9 SKIP items right after a clone. Git does not store file
  modification times, so the "figure is newer than its inputs" check has no basis; the 27 checks
  that compare plotted numbers with the artifacts still run. Run
  `python runners/make_paper1_figures.py` first to check provenance as well.

---

## What the revision changed

Re-running everything under the corrected protocol (10 runs instead of 5, symmetric tuning of
quantum and classical models, random forest and XGBoost added, scores on the full KDDTest+
partition) showed that five claims of the submitted version do not hold:

| Submitted version | Measured |
|---|---|
| Advantage in the low-data regime | Reversed: the classical models win at small N |
| "+6.7 points on rare attacks, d = +0.68" | Not reproducible; the released code computed no rare-class metric |
| Theorem 1: n* = 4 maximises J | False; J is maximised at n = 2 |
| QSVM 0.854 > SVM-RBF 0.838 | XGBoost 0.8503 > QSVM-ZZ 0.8469 (same embedding, same protocol) |
| "Quantum advantage is real" | 21 favourable / 21 unfavourable / 68 inconclusive over 110 comparisons |

Three results take their place:

1. **An ordering that reverses with training-set size.** On the shared four-dimensional
   embedding, QSVM-ZZ is next to last at N = 100 and first at N = 10^4, changing sign between
   N = 2000 and 5000 in all six baseline x tuning-arm combinations. Against full-feature XGBoost
   (reference arm) the kernel reaches parity at N >= 5000, not a lead; full-feature XGBoost on the
   entire training partition scores 0.804, above every four-qubit model.
2. **A dimension-selection rule that transfers.** The same lexicographic rule, with no threshold
   changed, returns n* = 4 on NSL-KDD and n* = 6 on UNSW-NB15, recovered on 10/10 subsets.
3. **A measured boundary with a mechanism.** At 8 qubits all 48 comparisons favour the classical
   side; the off-diagonal dispersion of the Gram matrix decays about twice as fast for the ZZ map
   as for its entanglement-free control (the ratio the circuit's phase-term count predicts) and
   tracks macro-F1 (r = +0.77).

---

## Two sampling regimes: do not mix them

| Contribution | Training pool | N | Test set | Runs |
|---|---|---|---|---|
| C1 width and n*, C2 entanglement, C3 shift, width sweep | rare-enriched (rare classes raised to 10%, the pool of the submitted version) | 1000 | fixed 300-record split (C3: re-mixed test of 1000) | 10 |
| C2 noise check (FakeManilaV2) | rare-enriched | 1000 | fixed 300 | 1 |
| C4 sample-complexity sweep, rare attacks, reference arm | natural class prior (0.83% rare) | 10^2 to 10^4 | full KDDTest+ (22,544 records) | 10 |

The same model at the same N scores differently under the two regimes (QSVM-ZZ at N = 1000:
0.8469 on the enriched/300 protocol, 0.7717 on the natural/full protocol). Table II of the
manuscript states which figure uses which.

Absolute scores on the official NSL-KDD split are far below the 0.99 often reported in the
literature; `results/nslkdd/c4_revision/c4_protocol_vs_literature.csv` shows why (XGBoost with
all features: 0.804 on the official split, 0.999 on a random split within the training partition).

---

## Layout

```
src/c4_pipeline.py        Core: data, nested subsets, representation, kernels, tuning, statistics
src/reliability.py        Core of the companion paper (probability calibration)

runners/                  One script per task
  audit_c4.py, audit_figures.py, audit_prose.py, verify_lemma1.py, check_latex.py   audits (see above)
  run_c4.py                 sample-complexity sweep (main result; several hours)
  run_ref_arm_fullfeat.py   reference arm: XGBoost / random forest on all 122 features (~1 h)
  make_ref_arm_table.py     pairs the reference arm with QSVM-ZZ -> Table IV and its macros
  run_c1_ksens.py           dimension-selection rule as a function of K
  run_ksweep.py             feature-budget sweep
  run_width_sweep.py        circuit-width sweep (4 to 10 qubits)
  run_gram_concentration.py Gram-matrix concentration
  run_hardware_kernel.py    execution path for a real backend (dry-run only; no hardware numbers are reported)
  make_paper1_figures.py    regenerates the nine figures
  make_overleaf_zip.py      packages the manuscript sources

configs/c4_protocol.json  Frozen protocol: seeds 100-109, N grid, nesting rules, grids, both sampling regimes
config.py                 Central paths

notebooks/nslkdd/         C1..C4_revision.ipynb (current); notebooks/unsw/ transfer to UNSW-NB15
data/     { nslkdd/, unsw/ }   raw and preprocessed data
models/   { nslkdd/, unsw/ }   fitted transformers (joblib) and Gram matrices (npy)
results/  { nslkdd/, unsw/ }   JSON/CSV artifacts: the source of every number in the manuscript

paper/paper1/             LaTeX sources of the revision
  main_revision.tex         clean copy (pdflatex)
  main_annotated.tex        annotated copy, changed text highlighted in yellow (lualatex; same sources, one flag)
  response_letter.tex       point-by-point response
  cover_letter.tex          cover letter
  sections/ tables/ figs_revision/ authors/
  v1_submitted/             the version submitted in May 2026 (PDF, for comparison)
paper/paper2/             Companion paper on calibration (submitted to IJNM)
docs/                     Working notes of the authors, in Vietnamese
```

Where each headline number lives:

| Result | File |
|---|---|
| 110-cell regime map | `results/nslkdd/regime_map_rows.csv` |
| Sample-complexity sweep, paired statistics | `results/nslkdd/c4_revision/c4_pairwise_statistics_natural.csv` |
| Sample-complexity sweep, per run | `results/nslkdd/c4_revision/c4_per_run_natural_refit_per_N.csv` |
| Reference arm (Table IV) | `results/nslkdd/c4_revision/ref_arm_fullfeat.csv`, `ref_arm_paired.csv` |
| Official split vs random split | `results/nslkdd/c4_revision/c4_protocol_vs_literature.csv` |
| K = 80 / n = 8 variant | `results/nslkdd/c4_revision/variant_K80n8/` |
| Dimension-selection rule | `results/nslkdd/c1_revision/c1_ksensitivity.json` |
| Gram concentration | `results/nslkdd/c1_revision/c1_gram_concentration.json` |
| Noise check (single run) | `results/nslkdd/c2_revision/c2_noise_validation.csv` |
| UNSW-NB15 transfer | `results/unsw/c4_revision/` |

Only the figures in `paper/paper1/figs_revision/` belong to the revision (provenance in
`figs_revision/MANIFEST.md`). Figures under `results/*/c3_multirun/`, `results/*/c4_multirun/` and
`data/*/processed_data/` come from the submitted protocol (5 seeds, asymmetric tuning, no tree
baselines) and contradict the manuscript; they are kept only for the record.

Kernel caches (`results/*/c3_revision/cache/`, `results/*/c4_revision/cache/`, 1.7 GB) are not in
the repository; the notebooks recompute them when missing, and the audits do not need them.

---

## Re-running

```bash
uv sync
python runners/run_c4.py                 # main result, several hours
python runners/run_ref_arm_fullfeat.py   # reference arm, about one hour on four cores
python runners/make_ref_arm_table.py
python runners/make_paper1_figures.py
python runners/audit_c4.py               # confirm the numbers match
```

Quantum kernels are evaluated as **exact statevector fidelities**, not sampled. Because the
ZZFeatureMap is diagonal after each Hadamard layer, the state has a closed form and need not be
simulated gate by gate; the shortcut is several hundred times faster than Qiskit and agrees with
the Qiskit reference to within 4e-15 (`audit_c4.py`, part C). Finite-shot sampling and the
backend-derived noise model are applied separately, as study conditions, and never mixed into the
main results.

Environment used for the reported numbers: NumPy 2.4, SciPy 1.17, scikit-learn 1.8, XGBoost 3.3,
Qiskit 2.3 with qiskit-machine-learning 0.9, Qiskit Aer 0.17. XGBoost with the histogram tree
method varies by about +-0.001 macro-F1 between machines even single-threaded; no claim in the
manuscript rests on a smaller difference.

---

## Manuscripts

| | Question | Status |
|---|---|---|
| Paper 1 (this repository) | In which regime is a NISQ-feasible quantum kernel worth its cost? | Revision 1 under review at IEEE TETC |
| Paper 2 (`paper/paper2/`) | Are the alarm probabilities of a QSVM calibrated? | Submitted to IJNM, August 2026 |
