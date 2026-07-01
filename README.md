# OpenADMET PXR Challenge: Activity Prediction

This repository contains my workflow for the **OpenADMET PXR Induction Blind Challenge** activity prediction track. This was my first complete machine learning project in drug discovery, and I used it as a way to learn how molecular representations, classical ML models, and calibration come together in an applied ADMET prediction task.

## Background

Pregnane X Receptor (PXR) is a nuclear receptor involved in regulating drug-metabolizing enzymes and transporters. Predicting whether a compound induces PXR is important in drug discovery because PXR activation can contribute to drug-drug interactions, altered metabolism, toxicity liabilities, and late-stage development risk.

The OpenADMET PXR challenge was to predict **pEC50 values** for a blinded analog set of compounds. 

The official challenge page and data are available through OpenADMET and Hugging Face:

- OpenADMET PXR Challenge Space: https://huggingface.co/spaces/openadmet/pxr-challenge?ref=openadmet.ghost.io
- OpenADMET PXR train/test dataset: https://huggingface.co/datasets/openadmet/pxr-challenge-train-test
- OpenADMET challenge announcement: https://openadmet.ghost.io/announcing-the-next-openadmet-blind-challenge-predicting-pxr-induction/

## Project Goal

The goal of this project was to build a reproducible model pipeline for predicting PXR activity as pEC50 values from molecular structure.

The final workflow combines:

- SMILES-based molecular graph learning using Chemprop
- classical machine learning on molecular fingerprints and descriptors
- greedy ensemble selection using the revealed Phase 1 labels
- Huber calibration
- conservative similarity-based residual correction
- final all-513 compound submission generation

## Repository Structure

```text
pxr-challenge/
│
├── README.md
├── notebooks/
│   └── pxr_phase2_ensemble.ipynb
|
├── outputs/
│   └── selected_ensemble_weights.csv

```

The notebook is the main workflow.

## Data

The notebook loads the OpenADMET PXR challenge data from Hugging Face.

The workflow uses:

1. the original training set,
2. the blinded 513-compound test set,
3. the Phase 1 unblinded labels released during Phase 2.

The final submission file contains predictions for all 513 compounds. For compounds whose Phase 1 labels were officially revealed, the true revealed pEC50 values are inserted into the final file, as required by the final submission instructions.


## Modeling Workflow

### 1. Data preparation

The notebook first loads the training, test, and Phase 1 unblinded data. SMILES strings are standardized using RDKit, and canonical SMILES are generated for matching compounds across files.

For Chemprop training, non-numeric target values such as `"no data"` are converted to missing values so that Chemprop can safely read multitask regression targets.

### 2. Molecular features

For classical machine learning models, the workflow generates molecular features using RDKit, including:

- Morgan fingerprints
- MACCS keys
- RDKit physicochemical descriptors

These features are used by linear and tree-based regression models.

### 3. Candidate models

The workflow trains multiple candidate models, including:

- Chemprop message-passing neural networks with different depth, dropout, hidden-size, and seed settings
- Ridge regression on molecular fingerprints
- ElasticNet regression
- Random Forest
- ExtraTrees
- HistGradientBoosting
- Gradient boosting models including XGBoost, LightGBM, and CatBoost

Chemprop was the strongest individual model family in my runs, but the final model still used an ensemble because some classical models contributed complementary errors.

### 4. Phase 1 diagnostic validation

To mimic the Phase 2 setting, candidate models were first evaluated on the revealed Analog Set 1 labels. This helped compare models using the challenge-style metrics before making predictions for the still-blinded compounds.

The diagnostic metrics used were:

- Mean Absolute Error (MAE)
- Relative Absolute Error (RAE)
- coefficient of determination (R²)

### 5. Greedy ensemble selection

Instead of averaging all models, I used greedy ensemble selection on the Phase 1 diagnostic set. The procedure repeatedly added the model that gave the largest improvement in Phase 1 RAE.

The selected ensemble from my final run was:

| Model | Weight |
|---|---:|
| Chemprop d4_h300_do05_s2 | 0.5714 |
| Ridge fingerprint model | 0.1429 |
| Chemprop d5_h300_do10_s4 | 0.1429 |
| HistGradientBoosting regressor | 0.1429 |

This means the final model was dominated by Chemprop message-passing neural networks, with smaller contributions from a fingerprint-based Ridge model and a histogram gradient boosting model.

### 6. Retraining with Phase 1 labels

After model selection, the selected models were retrained using:

```text
original training data + Phase 1 revealed labels
```

This allowed the final Phase 2 model to learn from compounds in the same analog-expansion region as the final blinded test compounds.

### 7. Calibration

Raw ensemble predictions were calibrated using the Phase 1 unblinded labels. I compared raw, linear, Huber, and isotonic calibration.

Although isotonic calibration gave the best apparent score when fit and evaluated on the same Phase 1 data, cross-validation showed that it was more likely to overfit. Therefore, the final workflow used **Huber calibration**, which was more robust in cross-validation.

### 8. Similarity-based residual correction

After calibration, I applied a conservative residual correction for test compounds that were highly similar to Phase 1 compounds.

The final residual correction settings were:

```text
minimum similarity = 0.70
maximum correction = 0.35 pEC50 units
correction strength = 0.50
```

This correction uses the residuals of nearby revealed Phase 1 analogs to slightly adjust predictions for similar compounds, while limiting the maximum allowed correction to reduce overfitting.

## Final Diagnostic Performance

On the unblinded Phase 1 diagnostic set, the final selected workflow achieved:

| Metric | Value |
|---|---:|
| MAE | 0.4341 |
| RAE | 0.5435 |
| R² | 0.6264 |

These values are diagnostic scores on the revealed Phase 1 set, not final blinded Phase 2 leaderboard scores.

## Final Prediction Method

The final prediction method can be summarized as:

> A greedy-selected ensemble dominated by Chemprop message-passing neural networks, with additional contribution from a Ridge fingerprint model and a HistGradientBoosting regressor. The ensemble predictions were calibrated using Huber regression and adjusted using conservative similarity-based residual correction based on revealed Phase 1 analogs.

## How to Run

### 1. Clone the repository

```bash
git clone https://github.com/MaryumIrshad/pxr-challenge.git
cd pxr-challenge
```

### 2. Create or activate an environment

The exact environment may depend on your machine. The main dependencies are:

```text
python
pandas
numpy
scikit-learn
rdkit
chemprop
xgboost
matplotlib
```

Optional dependencies:

```text
lightgbm
catboost
```

### 3. Run the notebook

Open the notebook in Jupyter or VS Code:

```text
notebooks/pxr_phase2_ensemble_github_ready.ipynb
```

Run the notebook from top to bottom. The final official-style submission file is written to:

```text
pxr_phase2_consolidated_workdir/outputs/pxr_final_submission_all_513_consolidated.csv
```

## Notes

- This repository is intended to document my modeling workflow and learning process.
- Challenge data and final submission files are not included.
- The notebook was cleaned for readability and reproducibility, but users may need to adjust paths or package versions depending on their environment.
