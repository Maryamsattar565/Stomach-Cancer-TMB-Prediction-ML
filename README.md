# Predicting Tumor Mutational Burden from Gene Expression in Stomach Adenocarcinoma (TCGA-STAD)

Machine learning pipeline that predicts tumor mutational burden (TMB,
nonsynonymous mutations/Mb) from genomic and transcriptomic profiling data
(RSEM-normalized gene expression) plus clinical covariates, using data from
the [Stomach Adenocarcinoma (TCGA, PanCancer Atlas)](https://www.cbioportal.org/study?id=stad_tcga_pan_can_atlas_2018)
study (TCGA, *Cell* 2018).

## Why this matters

TMB is a clinically used biomarker for immune checkpoint inhibitor response,
but measuring it directly requires whole-exome or large-panel sequencing. A
transcriptomic signature that approximates TMB would let it be estimated
from genomic profiling data generated for other purposes, without
additional sequencing cost.

## Data

Source: [Stomach Adenocarcinoma (TCGA, PanCancer Atlas)](https://www.cbioportal.org/study?id=stad_tcga_pan_can_atlas_2018),
TCGA, *Cell* 2018, accessed via cBioPortal.

**Raw data files are not included in this repository** (size and data-use
terms). To reproduce, download the study from cBioPortal and place these
files in `data/`:

| File | Description |
|---|---|
| `data_clinical_patient.txt` | Patient-level clinical annotations |
| `data_clinical_sample.txt` | Sample-level clinical/molecular annotations, incl. `TMB_NONSYNONYMOUS` (target) |
| `data_mrna_seq_v2_rsem.txt` | RSEM-normalized gene expression, genes × samples |

These are merged into one wide table (412 samples × ~20,500 raw features)
before modeling.

## Methods

1. **Merge** clinical patient + clinical sample + genomic expression data on sample ID.
2. **Target**: `TMB_NONSYNONYMOUS`, log1p-transformed for training; predictions
   are back-transformed (`expm1`) to report in original mut/Mb units.
3. **Leakage control**: dropped identifier columns and any feature that
   directly encodes or is derived from the mutation-calling / MSI pipeline
   used to compute TMB itself — `MSI_SCORE_MANTIS`, `MSI_SENSOR_SCORE`, and
   `SUBTYPE` (TCGA molecular subtype, which is assigned partly from mutation
   burden). This ensures the model learns from expression, not from a
   restatement of the target.
4. **Preprocessing** (inside a scikit-learn `ColumnTransformer`, fit only on
   training folds): median imputation + log1p for numeric features;
   most-frequent imputation + one-hot encoding for categorical features.
5. **Feature selection**: `VarianceThreshold` → `SelectKBest(f_regression)`,
   tuned as part of the hyperparameter search.
6. **Models**: Elastic Net, Random Forest, and XGBoost, each tuned with
   `RandomizedSearchCV` (10 iterations, 3-fold CV, scored on R²).
7. **Evaluation**: 3-fold CV on the training set, plus a held-out 20% test
   set, reported on both log and original TMB scale.
8. **Interpretation**: permutation importance computed end-to-end on the raw
   test set, through the full pipeline (preprocessing + feature selection +
   model), not just the final estimator.

## Results

3-fold cross-validation (training set), log-scale TMB:

| Model | CV R² (mean of 3 folds) |
|---|---|
| Elastic Net | 0.627 |
| XGBoost | 0.632 |
| Random Forest | 0.596 |

Held-out test set (20%, n≈83):

| Model | Test R² (log) | Test RMSE (log) | Test R² (mut/Mb) | Test RMSE (mut/Mb) | Test MAE (mut/Mb) |
|---|---|---|---|---|---|
| **Elastic Net** | **0.764** | **0.525** | **0.675** | **16.11** | **6.33** |
| XGBoost | 0.751 | 0.539 | 0.460 | 20.76 | 7.03 |
| Random Forest | 0.722 | 0.570 | 0.435 | 21.24 | 8.03 |

Elastic Net is the best model on every metric, on both scales.

Best Elastic Net hyperparameters: `alpha ≈ 0.464`, `l1_ratio = 0.05` (mostly
ridge-like), `select_kbest__k = 500`, `variance__threshold = 1e-5`.

### Top permutation-importance features (Elastic Net, test set)

| Rank | Gene | Mean decrease in test R² |
|---|---|---|
| 1 | MLH1 | 0.0355 |
| 2 | EFNA3 | 0.0256 |
| 3 | HOXA9 | 0.0204 |
| 4 | MSH4 | 0.0190 |
| 5 | RNF160 | 0.0142 |
| 6 | EPM2AIP1 | 0.0142 |
| 7 | GTF2A2 | 0.0138 |
| 8 | SMAP1 | 0.0127 |
| 9 | FUT10 | 0.0124 |
| 10 | HPSE | 0.0103 |

No single gene dominates — importance decays smoothly across dozens of
features, consistent with a polygenic expression signature. Notably, two of
the top four genes (`MLH1`, `MSH4`) are DNA mismatch-repair pathway members;
loss of mismatch repair is a canonical driver of hypermutation, which lends
biological plausibility to the signal.

### Figures

| File | Description |
|---|---|
| `Model_comparison_1.png` | Bar chart of test R² / RMSE by model |
| `observed_VS_predicted_2.png` | Observed vs. predicted log1p(TMB), Elastic Net |
| `Permutation_Importance_3.png` | Top 20 permutation-importance features |

## Limitations

- **Small test set (n ≈ 83)**: the ~4-point R² gaps between models are
  suggestive rather than statistically conclusive, and no independent
  external cohort was used for validation.
- **Real-scale predictions are less reliable than log-scale ones**: real-scale
  R² (0.44–0.68) is more variable across models than log-scale R²
  (0.72–0.76), because the log transform compresses the influence of a small
  number of hypermutated outliers.
- **Not precise enough for exact mutation counts**: Elastic Net's real-scale
  RMSE (16.1 mut/Mb) is large relative to typical non-hypermutated TMB
  values, so the model is better suited to ranking/stratifying samples
  (e.g., low vs. high TMB) than predicting an exact TMB value.

## Workflow

```



  ┌─────────────────────────────┐   ┌─────────────────────────────┐   ┌─────────────────────────────┐
  │ data_clinical_patient.txt   │   │  data_clinical_sample.txt   │   │ data_mrna_seq_v2_rsem.txt   │
  │   (Patient Demographics)    │   │      (Sample Metadata)      │   │   (RNA-Seq Expression)      │
  └──────────────┬──────────────┘   └──────────────┬──────────────┘   └──────────────┬──────────────┘
                 │                                 │                                 │
                 └────────────────┬────────────────┘                                 │
                                  ▼                                                  │
                       Clinical Data Merge                                           │
                     (Primary Key: PATIENT_ID)                                       │
                                  │                                                  │
                                  │                                                  ▼
                                  │                                      Clean mRNA Matrix
                                  │                                  (Drop Duplicate Hugo_Symbols
                                  │                                   & Transpose Genes to Cols)
                                  │                                                  │
                                  └────────────────────────┬─────────────────────────┘
                                                           │
                                                           ▼
                                                [ INNER JOIN MERGE ]
                                           Match Patient IDs Across Datasets
                                                           │
                                                           ▼
                                            STOMACH_MERGED_DATA_NEW.csv
                                            (412 Samples x 20,567 Features)
                                                           │
                                                           ▼
                                             Target Variable Definition
                                     y = log1p(TMB_NONSYNONYMOUS) [np.log1p]
                                                           │
                                                           ▼
                                              Target Leakage Prevention
                                       (Drop MSI_SCORE_MANTIS, MSI_SENSOR_SCORE,
                                           SUBTYPE, IDs, and Raw Targets)
                                                           │
                                                           ▼
                                               Leak-Free Data Split
                                        (80% Train: N=329 | 20% Test: N=83)
                                                           │
                                                           ▼
                                             ML Preprocessing Pipeline
                                    ┌──────────────────────────────────────┐
                                    │ • Median/Mode Imputation            │
                                    │ • One-Hot Encoding (Categoricals)    │
                                    │ • VarianceThreshold Filtering        │
                                    │ • SelectKBest (f_regression)         │
                                    └──────────────────┬───────────────────┘
                                                       │
                                                       ▼
                                      Hyperparameter Tuning & 3-Fold CV
                                           (RandomizedSearchCV)
                                  ┌────────────────────┼────────────────────┐
                                  ▼                    ▼                    ▼
                             Elastic Net          XGBoost            Random Forest
                           (CV R² = 0.627)      (CV R² = 0.632)     (CV R² = 0.596)
                                  │                    │                    │
                                  └────────────────────┼────────────────────┘
                                                       │
                                                       ▼
                                     Final Evaluation on Hold-out Test Set
                                        (Winner: Elastic Net | Test R² = 0.764)
                                                       │
                            ┌──────────────────────────┼──────────────────────────┐
                            ▼                          ▼                          ▼
                   Permutation Importance      Visualizations Generated     Exported Artifacts
                  (Top Predictive Genes)      (Scatter Plots & Bar Charts) (Results & Parameter CSVs)

====================================================================================================
Raw TCGA files (clinical patient, clinical sample, mRNA-seq RSEM)
        │
        ▼
Merge on sample ID → wide table (412 samples × ~20,500 features)
        │
        ▼
Drop identifiers + leakage columns (MSI scores, SUBTYPE)
        │
        ▼
Train/test split (80/20) → log1p(TMB) target
        │
        ▼
ColumnTransformer: impute + log1p (numeric) / impute + one-hot (categorical)
        │
        ▼
VarianceThreshold → SelectKBest(f_regression)
        │
        ▼
RandomizedSearchCV over Elastic Net / Random Forest / XGBoost (3-fold CV)
        │
        ▼
Evaluate on held-out test set (log scale + back-transformed mut/Mb scale)
        │
        ▼
Permutation importance (end-to-end, on raw test data)
        │
        ▼
Export results, plots, and best hyperparameters
```

## Reproducing

This pipeline was developed and run in Google Colab. To reproduce:

1. Open `Stomach_TMB_Prediction_ML.ipynb` in Google Colab.
2. Upload `data_clinical_patient.txt`, `data_clinical_sample.txt`, and
   `data_mrna_seq_v2_rsem.txt` to the Colab session (or mount Google Drive
   and place them there), matching the paths referenced in the notebook
   (e.g. `/content/...`).
3. Install dependencies not preinstalled in Colab:
   ```bash
   !pip install -r requirements.txt
   ```
4. Run all cells to merge the data, train the models, and regenerate the
   results and figures.

To run locally instead, install `requirements.txt` in a virtual environment,
adjust the hardcoded `/content/` paths to local paths, and run the notebook
with Jupyter.

## Output files

| File | Contents |
|---|---|
| `stomachTMB_Test_Results.csv` | Hold-out test metrics per model |
| `stomachTMB_CV_Results.csv` | Per-fold cross-validation metrics |
| `stomachTMB_Permutation_Importance.csv` | Permutation importance, all tested features |
| `stomachTMB_best_Parameters.csv` | Best hyperparameters per model |

## Planned next steps

- [ ] Add dimensionality reduction (e.g. PCA) and a feature-union step
      combining PCA components with the current `SelectKBest` features, to
      test whether a lower-dimensional representation improves
      generalization.
- [ ] Add a trivial baseline (predict the training median) and a
      clinical-only model, to quantify the specific contribution of
      expression data.
- [ ] Repeat the train/test split across multiple random seeds to report a
      confidence interval on R², given the small test set.
- [ ] Try an external gastric cancer cohort for validation, if available.

## Data source / citation

[Stomach Adenocarcinoma (TCGA, PanCancer Atlas)](https://www.cbioportal.org/study?id=stad_tcga_pan_can_atlas_2018),
TCGA, *Cell* 2018, accessed via [cBioPortal](https://www.cbioportal.org).
Please cite the TCGA Research Network and cBioPortal (Cerami et al. 2012;
Gao et al. 2013) when using this analysis.

## License

Add a license (e.g., MIT) before making this repository public.
