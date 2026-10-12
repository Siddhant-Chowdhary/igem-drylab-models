# Installation and prerequisites

Verified on a clean Python 3.11 virtual environment: `pip install -r
requirements.txt` followed by all 26 tests passing, with no other setup.

---

## 1. Prerequisites

### Python

**Python 3.10 or newer.** Check with:

```bash
python3 --version
```

If you do not have it: [python.org/downloads](https://www.python.org/downloads/).
On Windows, tick **"Add Python to PATH"** during installation or the commands
below will not be found.

### Python packages

Three, all installed by the one command in section 2:

| Package | Version | What it does here |
| --- | --- | --- |
| `cobra` | >= 0.29 | COBRApy. Loads the SBML model and runs every flux balance analysis |
| `numpy` | >= 1.24 | All the numerical work: scoring, statistics, the synthetic landscape |
| `PyYAML` | >= 6.0 | Reads the project config files in `configs/` |

### A linear-programming solver — nothing to do

COBRApy bundles **GLPK** through `optlang`, and it installs automatically with
`cobra`. You do not need Gurobi, CPLEX or any licence. Confirm after
installing:

```bash
python3 -c "import cobra; print(cobra.Configuration().solver.__name__)"
# expected: optlang.glpk_interface
```

Gurobi is faster if your institution has a licence, but nothing here requires
it and every published result was produced on GLPK.

### Optional extras

| Package | Needed for | Install |
| --- | --- | --- |
| `pytest` | Nicer test output. The tests also run without it | `pip install pytest` |
| `pandas` | Only the Jupyter notebook, for display tables | `pip install pandas` |
| `jupyter` | Only if running the notebook locally rather than in Colab | `pip install jupyter` |

### The metabolic model file — **not included, you must download it**

`yeast-GEM.xml` is about 11 MB and belongs to the
[SysBioChalmers/yeast-GEM](https://github.com/SysBioChalmers/yeast-GEM)
project, so it is deliberately excluded from this repository (see
`.gitignore`). Download it into the repository root:

```bash
wget https://github.com/SysBioChalmers/yeast-GEM/raw/main/model/yeast-GEM.xml
```

No `wget` (common on Windows and macOS)? Either:

```bash
curl -L -o yeast-GEM.xml https://github.com/SysBioChalmers/yeast-GEM/raw/main/model/yeast-GEM.xml
```

or open that URL in a browser, save the file, and move it into the repository
root next to `README.md`.

**Only `run_feasibility.py`, `run_diagnostics.py` and
`run_search_comparison.py` need the model.** `run_benchmark.py` and the whole
test suite run without it.

### Disk and time

About 400 MB once the Python packages are installed, plus 11 MB for the model.
The longest script (`run_diagnostics.py`) takes roughly 15 minutes; the rest
finish in under 10.

---

## 2. Install

```bash
git clone <your repository URL>
cd strain-design-preflight

python3 -m venv venv                 # optional but recommended
source venv/bin/activate             # Windows: venv\Scripts\activate

pip install -r requirements.txt
wget https://github.com/SysBioChalmers/yeast-GEM/raw/main/model/yeast-GEM.xml
```

Check it worked:

```bash
python3 tests/test_diagnostics.py    # expect: 10 passed, 0 failed
```

That test needs no model file, so if it passes your Python setup is correct
even if the download failed.

---

## 3. Run it

In the order that makes sense to read:

```bash
python3 examples/run_feasibility.py        # ~5 min, needs the model
python3 examples/run_diagnostics.py        # ~15 min, needs the model
python3 examples/run_benchmark.py          # ~5 min, no model needed
python3 examples/run_search_comparison.py  # ~10 min, needs the model
```

| Script | What it answers |
| --- | --- |
| `run_feasibility.py` | What C14:0 yield and purity the model can reach at a viable growth rate, which route it uses, and whether that route is robust |
| `run_diagnostics.py` | Whether the profile-matching score can rank anything, then tries to build a wet-lab proposal sheet (it refuses, by design) |
| `run_benchmark.py` | Whether a learned search beats greedy or brute force |
| `run_search_comparison.py` | Brute force vs greedy on the real model, with the diagnostics run first |

### Run the tests

```bash
python3 -m pytest tests/      # if pytest installed
# or, with no pytest:
python3 tests/test_filters.py
python3 tests/test_diagnostics.py
python3 tests/test_benchmark.py
```

26 tests total. None needs the model file.

### Or skip all of this

`igem_drylab_current_work.ipynb` reproduces every analysis in one notebook.
Upload it to [Google Colab](https://colab.research.google.com) and choose
**Runtime > Run all**. It installs the packages, downloads the model and
writes the code itself — nothing to set up, about 20 minutes. The saved
outputs are already in the file, so it can also simply be read.

---

## 4. Use it on your own project

Nothing in `rl_strain_design/` needs editing.

1. Copy `configs/yeast9_methyl_myristate.yaml` to a new file.
2. Set `model_path` to your own SBML model and `biomass_reaction_id` to its
   growth reaction.
3. List your `candidate_genes` and the `gene_to_model_id` mapping from common
   names to the gene IDs your model uses.
4. List `exchange_reactions` for the products in your target profile.
5. Express known-invalid gene combinations as `pre_fba_filters` rules — three
   rule types, documented in `rl_strain_design/filters.py`.
6. Set the measurement thresholds in `rl_strain_design/diagnostics.py`
   (`DETECTABLE_RELATIVE_MARGIN`, `MEASURABLE_RELATIVE_MARGIN`) to your own
   assay's replicate error. The defaults assume GC-MS composition at 1–5%.

The diagnostic suite needs only an object with
`evaluate(combo, target) -> float`. It does not require COBRApy, our config
system, or our evaluator — checks that need more capability skip rather than
crash, so it works on any scoring function.

```python
from rl_strain_design import run_diagnostics

class MyEvaluator:
    def evaluate(self, combo, target):
        return my_existing_scoring_function(combo, target)

report = run_diagnostics(MyEvaluator(), {"product_A": 0.7, "product_B": 0.3},
                          candidate_combos=[("geneX",), ("geneY",)])
print(report.to_text())
print(report.usable)      # False means any ranking is an artefact
```

---

## 5. Troubleshooting

| Problem | Cause and fix |
| --- | --- |
| `OSError: The file with 'yeast-GEM.xml' does not exist` | The model is missing or you are not in the repository root. Download it (section 1) and run scripts from the root, not from inside `examples/` |
| `ModuleNotFoundError: No module named 'rl_strain_design'` | Run from the repository root. The example scripts add the root to the path themselves; a shell started elsewhere will not find it |
| `ModuleNotFoundError: No module named 'cobra'` | Dependencies not installed, or your virtual environment is not active. Re-run `pip install -r requirements.txt` |
| A script seems to hang with no output | Python buffers output when piped. Use `python3 -u examples/...` |
| Script is very slow | Each FBA solve takes about 0.3 s and `run_diagnostics.py` does several hundred. 15 minutes is normal |
| `pip` not found on Windows | Python was installed without "Add to PATH". Reinstall and tick that box, or use `py -m pip` |

---

## 6. What is in the repository

```
strain-design-preflight/
├── rl_strain_design/              the package
│   ├── diagnostics.py             the six pre-flight checks  ← the main contribution
│   ├── proposals.py               wet-lab proposal sheets, gated on the checks
│   ├── benchmark.py               synthetic landscape + search-strategy comparison
│   ├── cobra_evaluator.py         COBRApy evaluator (3-stage lexicographic objective)
│   ├── search.py                  brute force, greedy, RL training loop
│   ├── agent.py                   the RL agent — retained, NOT recommended
│   ├── filters.py                 declarative rules for invalid gene combinations
│   ├── config.py                  YAML config loader
│   └── __init__.py
├── configs/
│   └── yeast9_methyl_myristate.yaml    our project, as a worked example
├── examples/
│   ├── run_feasibility.py
│   ├── run_diagnostics.py
│   ├── run_benchmark.py
│   ├── run_search_comparison.py
│   ├── feasibility_frontier.json       raw numbers behind the frontier tables
│   ├── benchmark_results.json          raw numbers behind the scaling table
│   ├── benchmark_experiments.json      raw numbers behind the epistasis sweep
│   ├── benchmark_table.txt
│   ├── diagnostic_report_yeast9.md     the suite's own output on our config
│   └── proposals_yeast9.csv            the refusal, as a CSV
├── tests/                         26 tests, none needing the model
├── README.md                      what it is, what is new, the results
├── INSTALL.md                     this file
├── requirements.txt
├── pyproject.toml
└── LICENSE                        MIT
```

Not in the repository, by design: `yeast-GEM.xml` (third-party, 11 MB,
download it) and `venv/` or `__pycache__/` (see `.gitignore`).
