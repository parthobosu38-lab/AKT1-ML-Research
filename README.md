# AKT1 Kinase Inhibitor Discovery Pipeline

An end-to-end computational pipeline for discovering novel AKT1 (protein kinase B alpha)
inhibitors, combining ChEMBL bioactivity curation, a hybrid machine-learning classifier +
regressor for activity prediction, *de novo* molecule generation with REINVENT4, novelty
assessment against public databases, and structure-based validation via pocket detection
and molecular docking.



## Pipeline Overview

| # | Notebook | Purpose |
|---|----------|---------|
| 01 | `Data_Curation.ipynb` | Pulls AKT1 (ChEMBL target `CHEMBL4282`) bioactivity data via the ChEMBL web-resource client, filters to IC50 values with `confidence_score >= 8`, normalizes units to nM, and converts to pIC50. |
| 02 | `Data_Preprocessing.ipynb` | Labels compounds into a three-tier activity scheme, generates ECFP4 + RDKit topological + MACCS fingerprints, performs a Bemis–Murcko scaffold split (train/val/test), and applies leakage-safe feature filtering. |
| 03 | `Classification_Model.ipynb` | Trains and compares Logistic Regression, Random Forest, XGBoost, and LightGBM classifiers (plus stacking/ensemble variants) under 5-fold scaffold-grouped CV with Optuna hyperparameter tuning. |
| 04 | `Regression_Model.ipynb` | Same scaffold-grouped CV / Optuna design as the classifier notebook, predicting continuous pIC50 with Ridge, Random Forest, and XGBoost, ensembled into a weighted blend. |
| 05 | `External_Validations.ipynb` | Freezes the classifier + regressor into a single hybrid predictor bundle (fixed blend weight and decision threshold from development data only) and externally validates it. |
| 06 | `REINVENT4_Molecules_Generations.ipynb` | Fine-tunes a REINVENT4 prior via transfer learning on known AKT1 actives, then runs reinforcement learning to generate novel candidate molecules. |
| 07 | `Post_Generation_Curation_of_REINVENT4.ipynb` | Filters generated molecules for validity, uniqueness, and drug-likeness (QED, synthetic accessibility via `sascorer`, synthetic complexity via `scscore`, PAINS/structural alert filtering). |
| 08 | `Hybrid_Classifier_Regressor_Screening_of_Curated_AKT1_Candidate.ipynb` | Reloads the frozen hybrid bundle and screens the curated, generated candidate library to flag predicted actives. |
| 09 | `Novelty_Assessment.ipynb` | Checks top candidates by InChIKey/SMILES against PubChem, ChEMBL, ZINC, and BindingDB to flag genuinely novel structures. |
| 10 | `Fpocket_Script.ipynb` | Automatically retrieves eligible AKT1 crystal structures from the RCSB PDB (resolution/ligand filters), runs `fpocket` on each, and ranks candidate binding pockets. |
| 11 | `Docking_Script.ipynb` | Batch-docks curated candidates into the validated AKT1 binding site with AutoDock Vina (fixed seed, resumable), producing ranked binding affinities. |

Run the notebooks in numeric order — each one consumes artifacts (feature/split pickles,
the frozen model bundle, curated molecule tables) produced by an earlier stage.

## Repository Structure

```
.
├── 01__Data_Curation.ipynb
├── 02__Data_Preprocessing.ipynb
├── 03__Classification_Model.ipynb
├── 04__Regression_Model.ipynb
├── 05__External_Validations.ipynb
├── 06__REINVENT4_Molecules_Generations.ipynb
├── 07__Post_Generation_Curation_of_REINVENT4.ipynb
├── 08__Hybrid_Classifier_Regressor_Screening_of_Curated_AKT1_Candidate.ipynb
├── 09__Novelty_Assessment.ipynb
├── 10__Fpocket_Script.ipynb
├── 11__Docking_Script.ipynb
├── requirements.txt
├── LICENSE
├── CITATION.cff
└── README.md
```

Data, model, and results directories (e.g. `data/`, `models/`, `results/`) are created by
the notebooks at runtime and are intentionally not included here — add a `.gitignore` for
them if you don't want generated artifacts committed.

## Installation

The core pipeline (notebooks 01–05, 08, 09) needs only Python packages:

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Two stages depend on external tools that are **not** installable via `pip` and must be
set up separately:

- **Notebook 06 (REINVENT4):** clone and install [MolecularAI/REINVENT4](https://github.com/MolecularAI/REINVENT4)
  per its own instructions (it manages its own PyTorch/conda environment). The notebook
  clones the repo and drives it via `subprocess`.
- **Notebook 10 (fpocket):** clone and build [Discngine/fpocket](https://github.com/Discngine/fpocket),
  or install it from `conda-forge` (`conda install -c conda-forge fpocket`).
- **Notebook 11 (docking):** requires an [AutoDock Vina](https://vina.scripps.edu/) executable
  (`vina`) on your `PATH`, plus a `config.txt` with the receptor path and grid box
  (`center_x/y/z`, `size_x/y/z`) validated by redocking — not just the raw fpocket output.

Notebooks 07 and 09 also fetch a couple of small external assets at runtime (RDKit's
`sascorer.py`/SA-score model, and the `connorcoley/scscore` repo) — no action needed
beyond internet access on first run.

> **Colab origin:** these notebooks were developed in Google Colab and include
> Colab-specific cells (`google.colab` Drive mounts, `!apt-get`, GPU checks). Running
> locally in Jupyter is fine — just skip or adapt those cells. Notebook 11 also has a
> hardcoded example path (`os.chdir('G:\AKT1\Docking')`); update it to your own working
> directory before running.

## Methodology Notes

- **Data leakage control:** scaffold-based (Bemis–Murcko) splitting and scaffold-grouped
  cross-validation are used throughout so structurally related compounds never appear in
  both train and evaluation folds.
- **Hybrid model:** the classifier and regressor are combined into a single frozen bundle
  with a blend weight and decision threshold fixed from development data only, then
  reused unchanged for external validation and screening.
- **Reproducibility:** Optuna studies and the Vina docking runs use fixed random seeds
  (see individual notebooks for exact values, e.g. `SEED = 42` in the docking script).
- **Novelty definition:** a generated molecule is only flagged novel if it returns a
  confirmed "not found" from every database source that was successfully queried;
  sources that failed to respond are excluded rather than counted as evidence of novelty.

## Citation

If you use this pipeline, please cite it — see [`CITATION.cff`](./CITATION.cff)
(fill in your author details and, once available, the associated paper).

## License

Released under the [MIT License](./LICENSE) — update the copyright holder name before
publishing. Note that this covers the code in this repository only; ChEMBL, PubChem,
ZINC, BindingDB, and PDB data retain their own respective usage terms, and REINVENT4 /
fpocket / RDKit / AutoDock Vina are separate third-party tools under their own licenses.

## Acknowledgments

This work builds on several open tools and databases: [ChEMBL](https://www.ebi.ac.uk/chembl/),
[RDKit](https://www.rdkit.org/), [REINVENT4](https://github.com/MolecularAI/REINVENT4),
[fpocket](https://github.com/Discngine/fpocket), [AutoDock Vina](https://vina.scripps.edu/),
[PubChem](https://pubchem.ncbi.nlm.nih.gov/), [ZINC](https://zinc.docking.org/), and
[BindingDB](https://www.bindingdb.org/).
