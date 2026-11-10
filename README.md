
# SALF Training Project — User Guide

This project provides dataset-specific preparation for OBA, HBA, and VitalDB; EEG sensitivity random forest (RF) training; subject grouping into S/NS folders; and Teacher, Baseline Student, and KD Student training with Transformer, LSTM, Graph Encoder, and VAAL backbones.

## 1. Installation and startup

On the Ubuntu server:

```bash
cd /path/to/project
conda activate kd
unzip SALF_minimal_clean.zip
cd SALF

python -m SALF --help
```

To install the package and its declared dependencies, if necessary:

```bash
python -m pip install -r requirements.txt
python -m pip install -e .
```

Display available arguments:

```bash
python -m SALF prepare --help
python -m SALF rf-features --help
python -m SALF rf --help
python -m SALF group --help
python -m SALF scan --help
python -m SALF train --help
python -m SALF select --help
```

## 2. Project layout

```text
SALF/
├── SALF/
│   ├── __main__.py                 # Command-line entry point
│   ├── datasets/
│   │   ├── oba/                    # OBA preprocessing, labels, early features
│   │   ├── hba/                    # HBA preprocessing, labels, early features
│   │   └── vitaldb/                # VitalDB preprocessing, labels, early features
│   ├── data/                       # Subject loading, grouping, label metadata
│   ├── sensitivity/                # Random forest training
│   ├── models/                     # Transformer, LSTM, Graph, VAAL
│   └── training/                   # Teacher, Baseline, KD, model selection
├── README_中文.md
├── README_EN.md
├── requirements.txt
├── pyproject.toml
└── SHA256SUMS
```

## 3. Input data directory

All three datasets use the same subject-folder layout, with one NPZ per subject:

```text
EEG_DATA/
├── OBA/
│   ├── incoming/          # Subjects awaiting grouping
│   ├── S/                 # Sensitive subjects
│   ├── NS/                # Non-sensitive subjects
│   └── subjects.csv       # Subject-to-file index
├── HBA/
│   ├── incoming/
│   ├── S/
│   └── NS/
└── VitalDB/
    ├── incoming/
    ├── S/
    └── NS/
```

Each `ID.npz` must contain at least:

| Field | Required contents |
|---|---|
| `x_psd` | `[N, 20, 40]`, where N is the number of samples |
| `y` | `[N]`, integer DoA class labels |
| `case_ids` | `[N]`, each beginning with `DATASET:ID:` |

Class IDs are:

- `0 = Awake`
- `1 = Optimal anesthesia`
- `2 = Excessive anesthesia`

Prepared files also carry `label_schema=AWAKE_OPTIMAL_EXCESSIVE_V1`. For manually supplied NPZ files, confirm the class meanings and required metadata.

Set the data and output roots:

```bash
DATA=${HOME}/SALF_DATA
OUT=${HOME}/SALF_RUN
```

## 4. Prepare DoA input for each dataset

### 4.1 OBA

The source ZIP should contain paired `psdata_<ID>.txt` and `label_<ID>.txt` files, with the class IDs defined above.

```bash
python -m SALF prepare \
  --dataset OBA \
  --source /path/to/OBA_PSD_LABEL.zip \
  --output "$DATA" \
  --source-label-order canonical
```

### 4.2 HBA

The source directory should contain paired `<ID>_Sdb.csv` and `<ID>_l.csv` files.

```bash
python -m SALF prepare \
  --dataset HBA \
  --source /path/to/HBA_source_csv \
  --output "$DATA"
```

**HBA label calculation:** The CSV converter fits `StandardScaler` on all spectral rows per subject, averages standardized columns 16–23, and splits unconscious windows at their median score. This converter does not apply an `EEGquality` filter.

### 4.3 VitalDB

Provide a case selection table and precomputed feature caches.

The `cases.csv` file must contain at least `caseid`. Each cache must contain the expected `features`, `labels`, and `window_idx` fields.

For cached labels following `0=Awake, 1=Optimal, 2=Excessive`:

```bash
python -m SALF prepare \
  --dataset VitalDB \
  --source /path/to/cases.csv \
  --cache-dir /path/to/feature_cache_2s_40d \
  --output "$DATA" \
  --cache-label-order canonical
```

The `prepare` command writes subject NPZ files into each dataset's `incoming/` directory.

If valid `S/` and `NS/` directories are already available, proceed directly to Section 6.

## 5. Early sensitivity features and RF training

The RF takes five EEG features per subject from the first 10 minutes and predicts `S` or `NS`.

These labels are separate from the three-class DoA target `y`.

### 5.1 Prepare the five-feature table

OBA and HBA input tables require the following columns:

```text
subject_id
t_IES_1
t_IES_2
t_alpha_1
t_alpha_2
entropy
```

Validate and prepare OBA features:

```bash
python -m SALF rf-features \
  --dataset OBA \
  --feature-table /path/to/OBA_5features.csv \
  --output features/OBA_checked.csv
```

Validate and prepare HBA features:

```bash
python -m SALF rf-features \
  --dataset HBA \
  --feature-table /path/to/HBA_5features.csv \
  --output features/HBA_checked.csv
```

These commands validate supplied feature tables. The physiological onset and entropy features must be prepared from EEG beforehand.

For VitalDB, the entropy feature can be calculated using its available EEG data. The case table must supply the other four onset timing features:

```bash
python -m SALF rf-features \
  --dataset VitalDB \
  --cohort-root /path/to/VitalDB_ROOT \
  --cases /path/to/vitaldb_cases_with_four_onsets.csv \
  --output features/VitalDB_5features.csv
```

VitalDB's default feature names are:

```text
first_ies_sec_ch1
first_ies_sec_ch2
first_alphaS_sec_ch1
first_alphaS_sec_ch2
permutation_entropy
```

### 5.2 Train RF models

Train the OBA sensitivity classifier:

```bash
python -m SALF rf \
  --dataset OBA \
  --feature-table features/OBA_checked.csv \
  --output runs/RF_OBA
```

Train the HBA sensitivity classifier:

```bash
python -m SALF rf \
  --dataset HBA \
  --feature-table features/HBA_checked.csv \
  --output runs/RF_HBA
```

Train the VitalDB sensitivity classifier:

```bash
python -m SALF rf \
  --dataset VitalDB \
  --feature-table features/VitalDB_5features.csv \
  --output runs/RF_VitalDB
```

The RF models and feature metadata are saved under their respective `runs/RF_<dataset>/` directories.

Run the following command for additional RF options:

```bash
python -m SALF rf --help
```

## 6. Assign subjects to S/NS folders

### 6.1 Group using an S/NS assignment table

The assignment CSV must contain at least:

```csv
subject_id,reference_phenotype
2,S
3,NS
```

Copy cases from `incoming/` into their assigned folders.

For OBA:

```bash
python -m SALF group \
  --dataset OBA \
  --data-root "$DATA" \
  --feature-table features/OBA_checked.csv \
  --reference-labels /path/to/OBA_reference.csv \
  --assignment reference
```

For HBA:

```bash
python -m SALF group \
  --dataset HBA \
  --data-root "$DATA" \
  --feature-table features/HBA_checked.csv \
  --reference-labels /path/to/HBA_reference.csv \
  --assignment reference
```

For VitalDB:

```bash
python -m SALF group \
  --dataset VitalDB \
  --data-root "$DATA" \
  --feature-table features/VitalDB_5features.csv \
  --reference-labels /path/to/VitalDB_reference.csv \
  --assignment reference
```

The command writes:

- `subjects.csv`
- `group_assignment_manifest.csv`
- `group_assignment.json`

These files record subject IDs, assigned groups, and source/destination file details.

RF-based predicted grouping is also supported. Use the following command to inspect its parameters:

```bash
python -m SALF group --help
```

### 6.2 If S/NS folders already exist

Scan the folders without copying or regrouping:

```bash
python -m SALF scan \
  --dataset OBA \
  --data-root "$DATA"

python -m SALF scan \
  --dataset HBA \
  --data-root "$DATA"

python -m SALF scan \
  --dataset VitalDB \
  --data-root "$DATA"
```

For manually prepared files whose class meanings are already verified but lack label metadata, inspect the following command:

```bash
python -m SALF label-stamp --help
```

## 7. Train the four backbones

Supported backbones:

- `transformer`
- `lstm`
- `graph`
- `vaal`

Supported training stages:

- `teacher`
- `base`
- `kd`
- `all`

The Teacher uses sensitive (`S`) subjects.

The Baseline Student uses non-sensitive (`NS`) subjects.

The KD Student also uses non-sensitive subjects and is initialized from the corresponding Baseline checkpoint.

During knowledge distillation, the Teacher is frozen.

To execute KD separately, first complete Teacher and Baseline training for the same split.

### 7.1 Run one stage at a time

Train the Teacher:

```bash
python -m SALF train \
  --dataset OBA \
  --backbone transformer \
  --stage teacher \
  --outer-fold 1 \
  --inner-fold 1 \
  --data-root "$DATA" \
  --output "$OUT" \
  --device cuda:0
```

Train the Baseline Student:

```bash
python -m SALF train \
  --dataset OBA \
  --backbone transformer \
  --stage base \
  --outer-fold 1 \
  --inner-fold 1 \
  --data-root "$DATA" \
  --output "$OUT" \
  --device cuda:0
```

Train the KD Student:

```bash
python -m SALF train \
  --dataset OBA \
  --backbone transformer \
  --stage kd \
  --outer-fold 1 \
  --inner-fold 1 \
  --data-root "$DATA" \
  --output "$OUT" \
  --device cuda:0
```

### 7.2 Run all stages in sequence

```bash
python -m SALF train \
  --dataset HBA \
  --backbone lstm \
  --stage all \
  --outer-fold 1 \
  --inner-fold 1 \
  --data-root "$DATA" \
  --output "$OUT" \
  --device cuda:0
```

Set `--dataset` to `OBA`, `HBA`, or `VitalDB`, and choose any of the four backbones.

### 7.3 Train all inner folds, select a backbone, and run final training

For a subject-wise ten-outer-fold setup, each S/NS group needs at least ten distinct subjects.

The following example runs outer fold 1 for OBA:

```bash
for b in transformer lstm graph vaal; do
  python -m SALF train \
    --dataset OBA \
    --backbone "$b" \
    --stage all \
    --outer-fold 1 \
    --inner-fold all \
    --data-root "$DATA" \
    --output "$OUT" \
    --device cuda:0
done
```

Select the backbone:

```bash
python -m SALF select \
  --dataset OBA \
  --outer-fold 1 \
  --output "$OUT"
```

Run final training:

```bash
python -m SALF train \
  --dataset OBA \
  --backbone auto \
  --stage all \
  --split final \
  --outer-fold 1 \
  --data-root "$DATA" \
  --output "$OUT" \
  --device cuda:0
```

Change `--outer-fold` for other folds.

Consult the training help menu for learning rate, batch size, epochs, and additional arguments:

```bash
python -m SALF train --help
```

## 8. Model output locations

Inner-fold checkpoints are stored under:

```text
$OUT/OBA/outer_01/inner/inner_01/SALF_transformer/
├── teacher/best.pt
├── base/best.pt
└── kd/best.pt
```

Final checkpoints are stored under:

```text
$OUT/OBA/outer_01/final/SALF_transformer/
├── teacher/model.pt
├── base/model.pt
└── kd/model.pt
```

Dataset names, backbones, and fold numbers vary with the command.

Additional metadata files include:

```text
cohort_signature.json
cohort_sources.csv
split_assignments.csv
training_protocol.json
```

These associate the saved weights with subject files, cross-validation splits, and training settings.

## 9. Basic environment and file checks

From the project root:

```bash
# Check dependencies
python -m pip check

# Check the command-line interface
python -m SALF --help

# Check the input dataset
python -m SALF scan \
  --dataset OBA \
  --data-root "$DATA"

# Check packaged file integrity
sha256sum -c SHA256SUMS
```

`SHA256SUMS` verifies that project files match their packaged contents. If a source file is edited, its checksum may no longer match.
