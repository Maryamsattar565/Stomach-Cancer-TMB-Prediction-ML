# Processed Data

**`STOMACH_MERGED_DATA_NEW.csv`** — not included in this repo (large,
auto-generated). Regenerate it locally rather than uploading it.

## What it is

Merged table combining `data_clinical_patient.txt`, `data_clinical_sample.txt`,
and `data_mrna_seq_v2_rsem.txt`, joined on `PATIENT_ID`/`SAMPLE_ID`. One row
per sample (~412), with clinical fields plus one column per gene
(~20,500), including the target `TMB_NONSYNONYMOUS`.

## How to regenerate

Run the merge cell in `Stomach_TMB_Prediction_ML.ipynb`, which loads the
three raw files, transposes the expression matrix so genes become columns,
and joins everything into one CSV.

## Note

If this file is too large to upload where you're hosting it, use Git LFS,
a cloud storage link, or gzip it instead of committing it directly.
