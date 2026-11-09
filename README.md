# AEF Forest Type Mapping

Supplementary data and code for **AlphaEarth Foundation Embeddings for Fine-Scale Forest Type Classification and Regional Mapping**.

This repository provides the original AEF training samples and classification scripts, together with the common samples and code used to compare AEF embeddings with conventional remote-sensing features.

## Data files

| Files | Description |
|---|---|
| `training_data_part1.csv`–`training_data_part4.csv` | Original full AEF sample set: 123,626 records and 70 fields, split by rows with a header in each part. |
| `common_samples_part1.csv.gz`–`common_samples_part4.csv.gz` | Common sample table for the four-group comparison: 119,736 records and 148 fields, split by rows into four gzip-compressed CSV files. |
| `shared_sample_split.csv` | Sample identifiers, reference information, and the fixed training/test assignments for the common-sample comparison. |

The original AEF sample set contains `id`, `point_id`, `lon`, `lat`, `lc_code`, `type`, and the 64 embedding dimensions `A00`–`A63`.

The common table contains the AEF embeddings, conventional features, sample metadata, and matching diagnostics. It has 41,922 records with 2015 reference labels paired with 2017 features, and 77,814 records with 2020 labels paired with 2020 features. Metadata and matching diagnostics are not predictors.

## Classification code

| File | Purpose |
|---|---|
| `01_model_training_accuracy_comparison.py` | Compares classification strategies and downstream classifiers using the original full AEF sample set. |
| `02_feature_importance_and_subset_sensitivity_ET_KNN.py` | Ranks AEF features and evaluates feature subset sensitivity with ET for forest/non-forest discrimination and KNN for forest-type classification. |
| `03_regional_prediction_ET_KNN_top28.py` | Applies the selected stage-specific 28-dimensional AEF subsets for regional forest-type mapping. |
| `three_groups_common_training.ipynb` | Trains and evaluates AEF ET–KNN, AEF RF–RF, and the conventional optical–topographic RF–RF baseline on common samples. |
| `add_raw_bands_RF_comparison.ipynb` | Adds the spectral-band RF–RF baseline, reusing the preceding experiment's saved partition and RF settings. |

The original supplementary description is also available as `Supplementary Material 1. Training data and model code.docx`.

## Common-sample comparison

| Configuration | Input features | Dimensions | Stage 1 / Stage 2 |
|---|---|---:|---|
| AEF main | Annual AEF embeddings, `A00`–`A63` | 64 | ET / KNN |
| AEF control | Annual AEF embeddings, `A00`–`A63` | 64 | RF / RF |
| Conventional optical–topographic | 40 quarterly spectral bands, 20 quarterly spectral indices, and 4 topographic variables | 64 | RF / RF |
| Conventional spectral-band | 40 quarterly spectral bands | 40 | RF / RF |

For each quarter (`Q1`–`Q4`), the spectral bands are `B2`, `B3`, `B4`, `B5`, `B6`, `B7`, `B8`, `B8A`, `B11`, and `B12`; the indices are `NDVI`, `EVI`, `NDMI`, `NBR`, and `NDRE`. Fields use names such as `Q1_B2` and `Q1_NDVI`. Topographic fields are `elevation`, `slope_deg`, `aspect_sin`, and `aspect_cos`.

All four configurations use the same 89,781 training records and 29,955 test records, including 28,778 forest training records and 9,547 reference-labelled forest test records. Join the table and split file using `source_id`.

The original split used the first fold of a four-fold stratified group partition, with shuffling and seed 42. Strata combine reference-label year and target class; groups are locations rounded to six decimal places across both years. This is a single holdout experiment, not four-fold cross-validation or spatial-block validation. The released `shared_sample_split.csv` is the authoritative partition; any regenerated split must match its sample-level assignments.

Forest-type evaluation uses reference-labelled forest test samples. Final six-class predictions route samples using the first-stage predictions and include error propagation. Confusion matrices have reference classes in rows and predicted classes in columns.

## Running the code

For the original AEF workflow, concatenate the four `training_data_part*.csv` files by rows and run scripts 01, 02, and 03 in order, adjusting their input and output paths.

For the common-sample comparison, first reconstruct `common_samples.csv` from the four compressed parts. Run this Python code in the repository folder; it preserves the original CSV bytes and numeric precision:

```python
from pathlib import Path
import gzip
import hashlib
import shutil

output = Path("common_samples.csv")
if output.exists():
    raise FileExistsError("common_samples.csv already exists; keep or move it first.")

with output.open("wb") as destination:
    for number in range(1, 5):
        with gzip.open(f"common_samples_part{number}.csv.gz", "rb") as source:
            header = source.readline()
            if number == 1:
                destination.write(header)
            shutil.copyfileobj(source, destination)

digest = hashlib.sha256()
with output.open("rb") as source:
    for block in iter(lambda: source.read(1024 * 1024), b""):
        digest.update(block)
assert digest.hexdigest() == "6281defeb4925bf4bfe2a6dd9b6a336198adfeaa4178c015fbbc19be3a3c6659"
print("Common sample table reconstructed and verified.")
```

Run `three_groups_common_training.ipynb` first, then `add_raw_bands_RF_comparison.ipynb`. Set `COMMON_PATH` to the reconstructed table in both notebooks. Set `OUTPUT_PARENT` to the desired output folder. For the second notebook, set `PREVIOUS_RESULTS` to the timestamped result directory generated by the first notebook.

The recorded common-sample experiment used Python 3.12.3, NumPy 1.26.4, pandas 2.2.3, and scikit-learn 1.5.2. Other dependencies include matplotlib, joblib, and threadpoolctl; tqdm is optional. RF and ET use 300 trees. KNN uses seven neighbors with distance weighting and a scaler fitted only on forest training samples. Consult the scripts for the dependencies of the original multi-classifier experiments.

The full-sample AEF experiments and common-sample comparisons have separate sample configurations and partitions. Efficiency measurements exclude feature extraction, downloads, and AEF pretraining. The spectral-band group was added in a later session, so original cross-group timings are supplementary observations rather than a single-session controlled speed benchmark. Saved model size is not peak memory consumption.

## License

This repository is licensed under the Creative Commons Attribution 4.0 International License (CC BY 4.0). See `LICENSE`.
