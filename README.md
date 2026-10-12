# strain-design-preflight

**Model-quality test suites check your model. Nothing checks your scoring
setup.** This package does, and it refuses to hand a wet lab a ranked list of
designs when its own scoring setup cannot support one.

Built by the iGEM IITB 2026 dry lab while engineering *S. cerevisiae* toward a
C14:0-rich fatty-acid profile for a methyl myristate coolant, and generalised
so any team with a COBRApy-compatible genome-scale model can run it by writing
a config file.

---

## Why this exists

[memote](https://doi.org/10.1038/s41587-020-0446-y) and
[FBCModelTests.jl](https://github.com/LCSB-BioCore/FBCModelTests.jl/) check
whether your *model* is well-formed — annotations, mass balance,
stoichiometric consistency, energy-generating cycles. They are excellent and
you should run them.

Neither checks the **objective and scoring function you wrote on top of the
model**. That is where our own pipeline silently failed. Our scoring function
ran without error, produced confident-looking rankings, and was measuring
nothing: the unedited strain scored as highly as the best knockout we could
find. We nearly sent that to the bench.

So: a pre-flight suite for the scoring setup, which gates its own output.

## What is actually new here, stated honestly

We surveyed memote, FBCModelTests, FROG, COBRApy's own validation helpers,
FVA, flux sampling, loopless FBA, pFBA, production envelopes, and the
strain-design tools OptKnock, OptGene, OptForce, RobustKnock, cameo, MEWpy,
OptFlux and StrainDesign. Of six checks:

| | Check | Status |
| --- | --- | --- |
| **D1** | null-intervention parity | **not automated anywhere we found.** StrainDesign's docs warn in prose that the wild type can come back as a solution; no tool tests for it |
| **D2** | landscape discrimination | **not automated anywhere we found.** No tool reports whether its own score can separate candidates |
| D3 | scalarization degeneracy | known. This is FVA / alternate-optima analysis plus [Das & Dennis 1997](https://doi.org/10.1007/BF01197559). We automate and explain it; we did not discover it |
| D4 | free production | known, and **memote already ships it**. Ours is a convenience check on your specific target reactions. Run memote too |
| D5 | growth confound | known ([Schuetz 2007](https://doi.org/10.1038/msb4100162)). Growth coupling is usually *intended* ([Klamt & Mahadevan 2015](https://doi.org/10.1016/j.ymben.2015.05.006)), so this check can never FAIL — only ask whether you meant it |
| D6 | parameter sensitivity | known. Ordinary sensitivity analysis |

**Output gating** — refusing to emit designs when the scoring landscape is
uninformative — is not a behaviour we found in any published strain-design
tool.

Please do not describe this suite as a whole as novel. The defensible claim is:
*a pre-flight suite for the scoring setup rather than the model, which gates
its own output; two of its checks and the gating behaviour appear to be
unautomated elsewhere.*

## The threshold that matters is the assay's, not the solver's

Our first version of the gate compared score differences against solver
tolerance (1e-6) — and it let our own broken pipeline through, because the
best knockout beat doing nothing by 2e-5, which is comfortably above 1e-6.

Solver tolerance is the wrong standard. The question is not "is this
difference real arithmetic?" but **"could the wet lab ever tell these two
strains apart?"** GC-MS fatty-acid composition carries roughly 1–5% relative
error between biological replicates, so a model ranking two designs 0.01%
apart is making a distinction no experiment can test.

The thresholds are therefore relative and tied to measurement resolution
(`DETECTABLE_RELATIVE_MARGIN = 1%`, `MEASURABLE_RELATIVE_MARGIN = 5%`).
Override them for your own assay — a tighter method earns a tighter threshold.

## Install

Full prerequisites, troubleshooting and a per-script guide are in
**[INSTALL.md](INSTALL.md)**. The short version:

```bash
pip install -r requirements.txt     # cobra, numpy, PyYAML
wget https://github.com/SysBioChalmers/yeast-GEM/raw/main/model/yeast-GEM.xml
python3 tests/test_diagnostics.py   # expect: 10 passed, 0 failed
```

Python 3.10+. A solver (GLPK) ships with cobra, so there is nothing else to
install. The model file is third-party and not included — download it as
above. Verified on a clean virtual environment.

## Use it

```python
from rl_strain_design import (
    load_config, CobraEvaluator, run_diagnostics, build_proposal_sheet,
)

config = load_config("configs/yeast9_methyl_myristate.yaml")
evaluator = CobraEvaluator(config)
target = {"C14:0": 0.6, "C16:0": 0.2, "C18:0": 0.1, "C18:1": 0.1}

report = run_diagnostics(evaluator, target)
print(report.to_text())        # or report.to_markdown()
print(report.usable)           # False -> any ranking is an artefact

sheet = build_proposal_sheet(evaluator, target, my_candidates)
print(sheet.gated)             # True -> it refused, and says why
sheet.to_csv("proposals.csv")
```

Worked examples: `python examples/run_diagnostics.py` (needs the model file),
`python examples/run_benchmark.py` (no model needed).

Your evaluator only has to provide `evaluate(combo, target) -> float`. The
suite uses `.config`, `.is_valid()`, `.diagnose()` and `._model` when present
and SKIPs the relevant check when absent, so it works on any scoring function,
not just ours.

## Result on our own project

The suite fails our own shipped config, which is the point of including it:

```
D1 null-intervention parity   FAIL   best intervention beats doing nothing by 0.010%
D2 landscape discrimination   FAIL   2 solver-visible levels, all inside one
                                     measurement-resolution band
D3 scalarization degeneracy   FAIL   weighted sum produced one product at every weight
D4 free production            WARN   all 4 target exchanges bounded at 1000
D5 growth confound            WARN   growth identical across candidates
D6 parameter sensitivity      WARN   ratio_match_alpha moves no score

VERDICT: NOT USABLE
```

Cause: Yeast9's fatty-acid exchanges are unbounded, so when growth is allowed
to fall the model already produces the target ratio with no edits at all.
Raising the growth floor to 20% and 40% of wild type did not change it. The
fix is a redefinition of the objective, which is a modelling decision the
team owns, so the tool reports the problem rather than papering over it.

## Does a learned search pay off? No, on our evidence

The package contains a linear Q-learning agent (`agent.py`) because that is
what we originally built. **We do not recommend using it.** `benchmark.py`
compares search strategies on a synthetic landscape with main effects, sparse
pairwise epistasis and a growth penalty, over 5 seeds:

| epistasis strength | greedy | RL | random (budget-matched) |
| --- | --- | --- | --- |
| 0.0 | 99.9 ± 0% | 82.2 ± 10% | 99.9 ± 0% |
| 0.5 | 99.8 ± 0% | 79.9 ± 12% | 99.8 ± 0% |
| 1.0 | 99.4 ± 1% | 79.7 ± 12% | 99.6 ± 1% |
| 2.0 | 99.5 ± 1% | 81.4 ± 10% | 99.6 ± 1% |
| 4.0 | 99.7 ± 0% | 84.8 ± 9% | 99.8 ± 0% |

(% of the exhaustive optimum, n=30 genes, order ≤3.)

Greedy hill-climbing reaches 99%+ using about `2 × n_genes` evaluations.
The RL agent reaches 80–85% with high variance and far more evaluations —
and random search on the same budget matches greedy. Even in RL's best case
(train once, answer 25 new targets) it used 1,279 evaluations for 83–88%
against greedy's ~1,850 for 98.5%: a 30% saving in compute for an 11-point
loss in quality.

### But greedy is not the right default at iGEM scale either

Over 20 seeds, counting how often each strategy finds the *exact* optimum:

| genes | strategy | finds exact optimum | mean shortfall | evaluations |
| --- | --- | --- | --- | --- |
| 10 | brute force | 100% (by definition) | 0% | 175 |
| 10 | greedy | 65% | 1.14% | 23 |
| 10 | RL | 10% | 10.13% | 1,204 |
| 30 | brute force | 100% | 0% | 4,525 |
| 30 | greedy | 50% | 0.63% | 71 |

At 10 genes on a real GEM (0.30 s per solve) brute force costs **52 seconds**
and is exact. Greedy costs 7 seconds and misses the best answer about a third
of the time. **Use brute force.** Greedy earns its place only when
enumeration stops being affordable — tens of genes at order 4+ — which is not
where most iGEM projects are.

So the recommendation is: brute force at this scale, `greedy_search()` when
enumeration gets too big, and the RL agent not at all. Two honest caveats: the landscape is synthetic,
because our real one is flat and a search comparison on a flat landscape
measures nothing; and a better-tuned or nonlinear agent might do better than
ours. The burden of proof sits with anyone claiming RL helps here.

That random search ties greedy is itself informative — this landscape has many
near-optimal solutions, and where that holds, search sophistication buys very
little. Teams reaching for ML on a combinatorial design problem should check
for that first.

## Adapting it to your project

1. Copy `configs/yeast9_methyl_myristate.yaml`.
2. Point `model_path` at your SBML model; set `biomass_reaction_id`.
3. List `candidate_genes` and `gene_to_model_id` (common name → your model's
   gene IDs).
4. List `exchange_reactions` for your target products.
5. Express known-invalid combinations as `pre_fba_filters` rules
   (three rule types, documented in `filters.py`).
6. Set the measurement thresholds in `diagnostics.py` to your assay's
   replicate error.

Nothing in `rl_strain_design/` needs editing.

## Limitations

- **Nothing here is validated against wet-lab data.** Every number is a model
  prediction.
- The scaling and epistasis benchmarks are **synthetic**. Transfer to a real
  GEM depends on the real landscape having comparable structure, which for our
  project is not established.
- Point mutations in the config's mutation table are **not applied** by the
  evaluator; predictions describe knockout effects only.
- D5 can flag growth-coupled designs, which are often exactly what you want.
  It is a question, not a verdict.
- Our thresholds assume GC-MS composition measurement. Different assay,
  different thresholds.
- The wet-lab effort and testing columns in a proposal sheet are the dry lab's
  estimates and were not confirmed with a wet lab.

## Tests

```bash
python -m pytest tests/        # or run each file directly
```

26 tests. `test_diagnostics.py` includes the case that defeated our first
gate: an evaluator whose scores differ by 1e-5, which is real arithmetic and
useless biology.

## License

MIT — see `LICENSE`.
