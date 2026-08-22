# EEG Motor State Clustering — 3 Subjects

Unsupervised analysis pipeline for EEG signals recorded with an OpenBCI-style device. The notebook takes raw multi-channel recordings from three subjects, cleans them, extracts spectral features from the Alpha and Beta bands, and clusters sliding windows into five distinct motor states using UMAP + K-Means.

## Overview

The pipeline is fully unsupervised — no movement labels are required. It discovers structure in the data by measuring how band power shifts across channels over time, then groups similar windows together.

```
Raw CSV → KNN imputation → Bandpass 8–30 Hz → Sliding-window PSD
        → Per-subject standardization → UMAP → K-Means (k=5) → Silhouette
```

## Data

Three CSV files, one per subject, containing `EXG Channel N` columns plus a `Timestamp` column:

```
datos_limpios_Datos_user_1.csv
datos_limpios_Datos_user_2.csv
datos_limpios_Datos_user_3.csv
```

Sampling rate defaults to 249.02 Hz and is recalculated from the `Timestamp` column when available.

## Pipeline

**1. Cleaning and imputation.** Missing values in EEG channels are filled with a KNN imputer. Channels containing more than 70% zeros are treated as disconnected electrodes and dropped. Remaining zeros are reinterpreted as missing data and imputed a second time. Non-EEG columns pass through untouched.

**2. Spectral inspection.** Welch's method (Hamming window, 1024 samples) produces a power spectral density plot per channel, with all three subjects overlaid and the standard EEG bands shaded — a visual sanity check before any modeling.

**3. Bandpass filtering.** A fourth-order Butterworth filter applied with `filtfilt` (zero phase) isolates the 8–30 Hz range, where motor-related Alpha and Beta activity lives.

**4. Feature extraction.** One-second sliding windows with 50% overlap. For each window and channel, mean power in the Alpha (8–13 Hz) and Beta (13–30 Hz) bands is computed, producing `2 × n_channels` features per window.

**5. Normalization.** `StandardScaler` is fitted per subject rather than globally, so differences in absolute signal amplitude between recordings do not dominate the clustering.

**6. Dimensionality reduction.** Four 2D projections — PCA, t-SNE, UMAP, and Kernel PCA (RBF) — colored by subject, to verify that the features carry usable structure before clustering.

**7. Clustering and evaluation.** An 80/20 train/test split feeds a scikit-learn `Pipeline` combining UMAP reduction with K-Means (k=5). Fitting the reducer inside the pipeline ensures it learns only from training data. Cluster quality is measured with the Silhouette score on held-out windows and visualized with a silhouette plot.

## Requirements

```
numpy
pandas
matplotlib
scipy
scikit-learn
umap-learn
```

Install with:

```bash
pip install numpy pandas matplotlib scipy scikit-learn umap-learn
```

## Usage

Open `Datapreproceso_limpio.ipynb` in Jupyter or Google Colab and edit the configuration cell in Section 1:

```python
DIR_ENTRADA = '/content/'          # folder containing the three CSV files
DIR_SALIDA  = '/content/cleaned_data/'
```

All parameters — sampling rate, frequency bands, window length, overlap, cluster count, random seed — are defined in that single cell. Then run the notebook top to bottom.

To apply the trained model to new recordings, pass them through the same cleaning, filtering, extraction, and normalization steps, then call:

```python
labels = pipeline_eeg.predict(new_features)
```

## Notebook structure

| Section | Contents |
|---|---|
| 1 | Global configuration |
| 2 | Dependency installation |
| 3 | File discovery |
| 4 | Cleaning and KNN imputation |
| 5 | Loading preprocessed data |
| 6 | Frequency analysis (Welch PSD) |
| 7 | Bandpass filtering (8–30 Hz) |
| 8 | Feature extraction (Alpha / Beta) |
| 9 | Normalization and final matrix |
| 10 | PCA · t-SNE · UMAP · Kernel PCA |
| 11 | UMAP + K-Means pipeline |
| 12 | Silhouette evaluation and cluster plots |
| 13 | Summary |

## Notes

Cluster labels are arbitrary identifiers. The pipeline finds five separable states in the signal, but mapping them to specific physical movements requires labeled ground truth. The Silhouette score reports internal cluster cohesion and separation, not classification accuracy against real movements.
