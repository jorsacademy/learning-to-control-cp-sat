# Learning to Control CP / CP-SAT Search

A small, reproducible research benchmark for **learning search-control decisions in constraint programming**, rather than using CP-SAT only as a black-box optimizer.

The project studies fixed-`k` graph coloring as a finite-domain CSP. It has two deliberately separate layers:

1. a transparent exact depth-first CP search where variable ordering is fully controllable at every search node; and
2. an OR-Tools CP-SAT benchmark restricted to search controls that the public Python API actually exposes: `add_decision_strategy(...)` plus `FIXED_SEARCH`.

The repository is an independent educational/research implementation. It is **not** a reproduction of a published codebase and does not claim paper-level benchmark performance.

## Motivation

Search heuristics matter because propagation and branching order can change the size of a CP search tree by orders of magnitude even though the underlying feasible set is unchanged. Classical CP therefore uses variable-ordering rules such as first-fail / minimum-domain, degree-based rules, weighted-degree variants, impact-based search, activity-based search, restarts, and heuristic portfolios.

Machine learning introduces a different question: can a policy infer, from the current propagated state, which branching variable is likely to make the subsequent exact search cheaper?

This is related to learning-to-branch in mixed-integer programming, but the solver state is different:

- **MIP:** LP relaxation, fractional variables, reduced costs, pseudocosts, strong branching, branch-and-bound.
- **CP:** finite domains, propagation, failures, variable/value ordering, restarts, and backtracking.

The distinction is substantive. This repository does not transplant an LP-branching interface into CP-SAT where that interface does not exist.

## Research Question

The main research question is:

> Can supervised imitation of an expensive propagation-based lookahead heuristic reduce finite-domain CP search cost on held-out graph-coloring CSPs, relative to strong interpretable classical variable-ordering baselines, without changing exactness?

Sub-questions are:

- Does top-1 expert imitation accuracy translate into fewer search nodes or backtracks?
- How does a learned dynamic policy compare with minimum-domain, domain/degree, static degree, fixed order, and random order?
- Does the learned policy generalize to larger/denser out-of-distribution graphs?
- What is the inference overhead of the learned policy relative to the search work it saves or adds?
- When the same learned scores are used only to produce a **root-level static order** for CP-SAT, does that external control help relative to CP-SAT's own search and simpler static orders?

Prediction accuracy and decision quality are reported separately. High imitation accuracy is not treated as evidence of lower search cost.

## Mathematical Problem

For an undirected graph `G=(V,E)` and a fixed number of colors `k`, define one finite-domain variable per vertex:

\[
x_v \in \{0,1,\ldots,k-1\} \qquad \forall v\in V.
\]

For every edge:

\[
x_u \neq x_v \qquad \forall (u,v)\in E.
\]

The decision problem is whether a feasible `k`-coloring exists. There is no optimization objective in this first version; the research metric is **search effort**, not coloring quality.

Synthetic instances are generated in two transparent families:

- **planted feasible:** edges are sampled only across different planted color classes, so a feasible coloring is known by construction;
- **certified infeasible:** a planted feasible graph is augmented with a `(k+1)`-clique, which is a direct certificate that the graph is not `k`-colorable.

These construction labels are useful for data-generation checks. Small instances are also verified independently by exhaustive enumeration.

## Methodology

### 1. Transparent finite-domain CP search

`src/learning_cp_sat/search.py` implements depth-first search with singleton propagation for binary `!=` constraints. Assigning `x_v=c` removes `c` from neighboring domains; newly singleton domains are propagated until a fixpoint or contradiction.

The learned component changes **only the next variable selected for branching**. Value ordering is intentionally held fixed (ascending color) so the experiment isolates variable ordering.

With no node/time limit, every policy traverses the same exact search space in a different order and therefore preserves completeness. A run is called `INFEASIBLE` only after exhaustive search. If a configured limit is reached first, the status is `UNKNOWN`, never `INFEASIBLE`.

### 2. Expensive expert: propagation lookahead

At a search state, the expert tentatively assigns every currently available value of every candidate variable and runs propagation. A candidate score combines:

- the fraction of values causing an immediate contradiction; and
- mean domain reduction on surviving probes.

This is a CP analogue of an expensive strong-lookahead branching rule: informative but too costly to use as the only practical heuristic. It is not claimed to be a globally optimal variable ordering.

Expert trajectories generate candidate-level supervised training data. The expert answer is never included as an input feature.

### 3. Learned policy

Each candidate variable is represented by eight state features:

- normalized current domain size;
- normalized static degree;
- remaining dynamic degree;
- fraction of singleton neighbors;
- saturation (distinct singleton colors in the neighborhood);
- mean neighboring domain size;
- minimum neighboring domain size;
- normalized search depth.

Two supervised models are trained from the same training split:

- standardized ridge regression as a simple conventional predictor;
- random-forest regression as a nonlinear predictor.

Validation top-1 imitation accuracy selects which fitted model is inserted into the final learned policy. The test and OOD sets are not used for this selection.

### 4. CP-SAT external-control layer

OR-Tools CP-SAT's Python API exposes declarative decision strategies such as `CHOOSE_FIRST`, `CHOOSE_MIN_DOMAIN_SIZE`, and domain reduction rules, and `FIXED_SEARCH` can force a supplied strategy. The public Python surface does **not** expose an arbitrary user callback that is invoked at each internal CP-SAT branching node to choose the next variable.

Accordingly this repository does not fake such a hook. It compares:

- CP-SAT `default` search under a controlled single-worker configuration;
- fixed vertex order;
- static degree order;
- `learned_root`, where the learned model scores the root state and produces a static variable permutation supplied through `add_decision_strategy` + `FIXED_SEARCH`.

The CP-SAT layer therefore measures **external/static strategy control**, not dynamic learned branching inside CP-SAT internals.

## Baselines

The transparent search benchmark includes:

- `fixed`: vertex index order;
- `degree`: largest static graph degree first;
- `mrv`: minimum remaining values / first-fail, with degree tie-breaking;
- `dom_degree`: domain size divided by remaining dynamic degree;
- `random`: seeded random variable order;
- `expert`: expensive propagation lookahead;
- `learned`: fitted supervised imitation policy.

The strongest classical baseline is not assumed in advance. Results are reported for all policies, including negative learned-policy results.

The CP-SAT benchmark includes `default`, `fixed`, `degree`, and `learned_root`.

## Exactness / Verification

Exactness is separated from search efficiency.

For the custom CP layer:

- no-limit DFS is complete;
- learned scores only reorder branching variables;
- `FEASIBLE` assignments are independently audited;
- `INFEASIBLE` means the search tree was exhausted;
- configured budget exhaustion is `UNKNOWN`.

For small instances, `src/learning_cp_sat/oracle.py` uses independent exhaustive Cartesian enumeration. It intentionally does not reuse the propagation or branching implementation. Tests require the custom solver to agree with this oracle.

For CP-SAT, solver status is reported exactly as returned. `OPTIMAL` is not used as a generic synonym for “good”; in this feasibility CSP, CP-SAT may return `OPTIMAL` after establishing feasibility because there is no objective.

## Feasibility Audit

Every returned assignment is checked outside the search routine:

- all vertices are present;
- each color is inside `0..k-1`;
- every edge has different endpoint colors.

A failed audit raises rather than being counted as a successful solve. The vector representation has one slot per vertex, so duplicate vertex identifiers are structurally excluded; missing decisions appear as missing/`None` entries and are rejected.

## Evaluation Protocol

`configs/benchmark.json` defines disjoint train, validation, test, and OOD seeds.

- **Train:** expert-state collection and model fitting only.
- **Validation:** imitation accuracy and model choice only.
- **Test:** final in-distribution search evaluation.
- **OOD:** larger and denser graphs; no retraining.

For search, the main metrics are:

- solved rate under configured limits;
- search nodes;
- backtracks;
- propagation events;
- wall-clock time;
- learned-policy candidate evaluations;
- expert lookahead probes.

For CP-SAT:

- branches;
- conflicts;
- wall-clock time;
- solved rate.

For each policy, the experiment reports mean, sample standard deviation, median, and p90. It also reports paired learned-minus-MRV node differences with a bootstrap 95% confidence interval. Pairing is by the same held-out instance.

The CI smoke configuration is intentionally tiny and is **not a scientific result**.

## Computational Cost

Solver control is not free. The benchmark separately records:

- candidate evaluations made by the learned model;
- expensive expert lookahead probes;
- propagation events;
- search nodes/backtracks;
- wall-clock time.

A learned policy that reduces nodes while greatly increasing total time is not described as an unconditional improvement.

## Reproducibility

All dataset and policy randomness uses explicit seeds from JSON configuration files. Experimental constants are not scattered through the implementation.

Install:

```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\\Scripts\\activate
python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
```

Run tests:

```bash
pytest -q
```

Run the CI-scale smoke experiment:

```bash
python scripts/run_benchmark.py \
  --config configs/smoke.json \
  --output artifacts/smoke.json
```

Run the larger research configuration:

```bash
python scripts/run_benchmark.py \
  --config configs/benchmark.json \
  --output artifacts/benchmark.json
```

The JSON output contains raw per-instance records plus aggregate statistics; this keeps downstream analysis auditable.

## Repository Structure

```text
.
├── .github/workflows/ci.yml
├── configs/
│   ├── benchmark.json
│   └── smoke.json
├── docs/
│   └── literature-and-protocol.md
├── scripts/
│   └── run_benchmark.py
├── src/learning_cp_sat/
│   ├── audit.py
│   ├── cpsat_control.py
│   ├── experiment.py
│   ├── expert.py
│   ├── features.py
│   ├── instances.py
│   ├── learning.py
│   ├── oracle.py
│   ├── propagation.py
│   ├── search.py
│   └── statistics.py
├── tests/
├── LICENSE
├── pyproject.toml
└── README.md
```

## CI

GitHub Actions tests Python 3.11 and 3.12 and performs:

- editable installation;
- `pip check` dependency validation;
- Ruff linting;
- Ruff formatting check;
- mathematical/methodological pytest suite;
- a separate end-to-end smoke experiment including OR-Tools CP-SAT;
- smoke-result schema validation.

The unit tests cover deterministic generation, infeasibility certificates, independent exhaustive oracle agreement, feasibility auditing, exact solver status semantics, seed determinism, learned-policy exactness, imitation metric bounds, CP-SAT fixed-search integration, statistics, and a tiny end-to-end experiment.

## Experimental Interpretation

A useful outcome can be positive or negative.

- Higher validation imitation accuracy may fail to reduce test search nodes.
- A learned policy may reduce nodes but lose on wall time because model inference is expensive.
- A policy can help the transparent custom solver but fail to help CP-SAT because the latter exposes only a static decision strategy here and retains its own SAT/CP machinery.
- OOD performance can degrade even when in-distribution performance improves.

These are research findings, not implementation failures.

## Limitations

- The benchmark is synthetic graph coloring, not an industrial scheduling dataset.
- The propagation engine is intentionally small: binary inequality constraints with singleton propagation, not a production CP kernel with global constraints, explanation-based propagation, nogoods, or sophisticated restarts.
- The expert is one-step lookahead, not an oracle for globally minimal search trees.
- The learned models use compact hand-designed features rather than a GNN over a generic variable-value-constraint graph.
- Only variable ordering is learned; value ordering and restart control are held outside the first version's scope.
- CP-SAT integration is external/static because the public Python API does not expose the dynamic callback needed to reproduce the transparent custom-solver experiment internally.
- Wall-clock values depend on hardware, Python/scikit-learn/OR-Tools versions, and runner load.

## Claims Boundary

This repository demonstrates a controlled methodology for studying learned CP search control. It does **not** claim:

- state-of-the-art performance;
- that the learned policy is superior to classical heuristics;
- that expert imitation accuracy guarantees lower search cost;
- that the lookahead expert is globally optimal;
- that CP-SAT exposes a custom per-node Python branching callback;
- that CI smoke results constitute a scientific benchmark;
- that synthetic graph-coloring behavior transfers directly to industrial scheduling or planning;
- production readiness or industrial savings.

“Exact” means only that, without configured limits, the transparent search is complete and the learned policy changes ordering rather than pruning admissible values. “Optimal” is not used for a heuristic ordering.

## Research Context / Related Repositories

This repository is designed to stand alone, but it sits next to several related portfolio threads:

- [`learning-to-branch-mip-gnn-scip-pytorch`](https://github.com/jorsacademy/learning-to-branch-mip-gnn-scip-pytorch): learned branching in MIP using SCIP's real branch-rule callback and LP-derived state. The present repo is the CP counterpart and emphasizes domains + propagation rather than LP relaxation + branching.
- [`optimal-conference-meeting-scheduling-cp-sat`](https://github.com/jorsacademy/optimal-conference-meeting-scheduling-cp-sat): CP-SAT used as a problem solver for scheduling. The present repo instead studies search control itself.
- [`constraint-learning-for-industrial-engineering`](https://github.com/jorsacademy/constraint-learning-for-industrial-engineering): broader learning/constraint research context.
- [`sequential-decision-analytics`](https://github.com/jorsacademy/sequential-decision-analytics): sequential decision methodology; relevant conceptually to learned search policies.
- [`neural-combinatorial-optimization-tsp-attention-model-pytorch`](https://github.com/jorsacademy/neural-combinatorial-optimization-tsp-attention-model-pytorch): direct neural construction for combinatorial optimization, contrasted here with learning inside an exact search procedure.

No code dependency is introduced between these repositories.

## Literature Map

### Classical CP search foundations

- Haralick, R. M., Elliott, G. L. (1980). *Increasing Tree Search Efficiency for Constraint Satisfaction Problems*. Artificial Intelligence 14(3), 263–313. https://doi.org/10.1016/0004-3702(80)90051-X
- Boussemart, F., Hemery, F., Lecoutre, C., Sais, L. (2004). *Boosting Systematic Search by Weighting Constraints*. ECAI 2004, 146–150.
- Refalo, P. (2004). *Impact-Based Search Strategies for Constraint Programming*. CP 2004, LNCS 3258, 557–571. https://doi.org/10.1007/978-3-540-30201-8_41
- van Beek, P. (2006). *Backtracking Search Algorithms*. In *Handbook of Constraint Programming*.

### Learning search heuristics

- Song, W., Cao, Z., Zhang, J., Xu, C., Lim, A. (2022). *Learning Variable Ordering Heuristics for Solving Constraint Satisfaction Problems*. Engineering Applications of Artificial Intelligence 109, 104603. https://doi.org/10.1016/j.engappai.2021.104603
- Doolaard, F., Yorke-Smith, N. (2022). *Online Learning of Variable Ordering Heuristics for Constraint Optimisation Problems*. Annals of Mathematics and Artificial Intelligence. https://doi.org/10.1007/s10472-022-09816-z

### Recent 2024–2026 direction

- Marty, T., Boisvert, L., François, T., Tessier, P., Gautier, L., Rousseau, L.-M., Cappart, Q. (2024). *Learning and Fine-Tuning a Generic Value-Selection Heuristic inside a Constraint Programming Solver*. Constraints 29, 234–260. https://doi.org/10.1007/s10601-024-09377-4
- Xu, J., Wu, Y., Li, H., Yin, M. (2025). *Prediction-Based Adaptive Variable Ordering Heuristics for Constraint Satisfaction Problems*. AAAI 2025, 11390–11398. https://doi.org/10.1609/aaai.v39i11.33239
- Cappart, Q., Guns, T., Lombardi, M., Pesant, G., Tsouros, D. (2025). *Combining Constraint Programming and Machine Learning: From Current Progress to Future Opportunities*. Journal of Artificial Intelligence Research 84. https://doi.org/10.1613/jair.1.19533

The 2024–2025 work reinforces two themes used here: learning branching heuristics from solver state and evaluating them by downstream search behavior rather than prediction metrics alone. The 2025 survey also emphasizes that CP+ML includes both learned solver components and learned modeling/selection components; this repository deliberately narrows scope to search control.

### Relation to learning-to-branch in MIP

- Gasse, M., Chételat, D., Ferroni, N., Charlin, L., Lodi, A. (2019). *Exact Combinatorial Optimization with Graph Convolutional Neural Networks*. NeurIPS 2019. https://proceedings.neurips.cc/paper/2019/hash/d14c2267d848abeb81fd590f371d39bd-Abstract.html

Gasse et al. imitate strong branching using MIP's variable-constraint bipartite state. The conceptual analogy in this repository is expensive lookahead imitation, but the mechanics are CP-specific: finite domains and propagation replace LP relaxations and fractional branching candidates.

### OR-Tools CP-SAT API / solver sources

- CP-SAT Python model API: https://github.com/google/or-tools/blob/stable/ortools/sat/python/cp_model.py
- CP model decision-strategy proto: https://github.com/google/or-tools/blob/stable/ortools/sat/cp_model.proto
- CP-SAT parameters (`FIXED_SEARCH`, branching/search parameters): https://github.com/google/or-tools/blob/stable/ortools/sat/sat_parameters.proto
- Fixed-search examples and solver documentation: https://github.com/google/or-tools/tree/stable/ortools/sat/docs

As of the 2026 OR-Tools 9.15 line, these public interfaces support declarative strategy/configuration control; this project stays within those interfaces.

## License

MIT. See `LICENSE`.
