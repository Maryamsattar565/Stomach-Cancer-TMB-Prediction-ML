# Data

This folder is where the raw input files go. **They are not included in this
repository** (file size and TCGA data-use terms), so you'll need to download
them yourself before running the pipeline.

## Source

[Stomach Adenocarcinoma (TCGA, PanCancer Atlas)](https://www.cbioportal.org/study?id=stad_tcga_pan_can_atlas_2018),
TCGA, *Cell* 2018 — accessed via [cBioPortal](https://www.cbioportal.org).

## How to get the files

1. Go to the study page linked above.
2. Click **Download** (top right) to get the full study data bundle.
3. From the downloaded folder, copy the three files listed below into this
   `data/` directory.

Alternatively, download via the cBioPortal API / `cbioportalR` / `cgdsr` if
you prefer a scripted approach.

## Required files

| File | What it contains |
|---|---|
| `data_clinical_patient.txt` | One row per patient — age, sex, stage, survival status/time, and other clinical fields. |
| `data_clinical_sample.txt` | One row per sample — sample type, tissue site, MSI scores, molecular subtype, and `TMB_NONSYNONYMOUS` (the prediction target). |
| `data_mrna_seq_v2_rsem.txt` | Gene expression matrix — genes as rows, samples as columns, RSEM-normalized values. |

All three use tab-separated values (`.txt`, tab-delimited). The clinical
files include a few comment/header lines starting with `#` at the top —
these should be skipped when loading (e.g. `pd.read_csv(..., sep="\t",
comment="#")`).

## Joining key

Samples are matched across all three files using the sample identifier
(`SAMPLE_ID` in the clinical sample file, and the column headers in the
expression file). Patient-level fields are joined in via `PATIENT_ID`.

## Expected size

~412 samples after merging and dropping rows with a missing TMB value;
~20,500 gene expression columns before any feature selection.

## Notes

- Do not commit these files to git — they're large (the expression matrix
  alone is well over 100 MB) and redistribution may be restricted by TCGA's
  data-use terms. Add `data/*.txt` to `.gitignore`.
- If a column mentioned in the main README (e.g. `MSI_SCORE_MANTIS`,
  `SUBTYPE`) doesn't appear in your download, check you have the
  PanCancer Atlas version of the study, not an older TCGA Provisional version
  — column names differ slightly between versions.
