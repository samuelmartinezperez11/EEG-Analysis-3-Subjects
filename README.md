# EEG Motor State Clustering — 3 Subjects

Unsupervised analysis pipeline for EEG signals recorded with an OpenBCI Cyton board. The notebook takes raw eight-channel recordings from three subjects, cleans them, extracts spectral features from the Alpha and Beta bands, and clusters sliding windows into five motor states using UMAP + K-Means.

This repository accompanies the manuscript *"Unsupervised Outlier Detection Pipeline for Low-Cost EEG Signals Based on Dimensionality Reduction and K-Means Clustering"* (see [Citation](#citation)).

## Overview

The pipeline is fully unsupervised — no movement labels are required. It discovers structure in the data by measuring how band power shifts across channels over time, then groups similar windows together.

```
Raw CSV → KNN imputation → Bandpass 8–30 Hz → Sliding-window PSD
        → Drop faulty subject → Drop top 2 % energy
        → Per-subject robust scaling → UMAP → K-Means (k=5) → Silhouette
```

## Repository contents

| Path | What it is |
|---|---|
| `Datapreproceso_limpio.ipynb` | Main analysis notebook (13 sections, runs top to bottom) |
| `Data` | Text file pointing to the dataset archived on Zenodo |
| `LICENSE` | MIT |
| `README.md` | This file |

> **Note.** `Data` is a pointer, not a folder. The recordings are archived on Zenodo and must be downloaded before running the notebook.

## Data

Three CSV files, one per subject, with `EXG Channel N` columns plus a `Timestamp` column:

```
datos_limpios_Datos_user_1.csv
datos_limpios_Datos_user_2.csv
datos_limpios_Datos_user_3.csv
```

Recorded with an OpenBCI Cyton board, eight channels over the fronto-parietal region. The sampling rate defaults to 249.02 Hz and is recalculated from the `Timestamp` column when available.

The dataset is archived on Zenodo with a permanent DOI:

> **DOI:** [10.5281/zenodo.22059396](https://doi.org/10.5281/zenodo.22059396)
> *(confirm the exact DOI on the Zenodo record page once it is published)*

**Subject 3 carries a hardware fault** — near-zero variance across all channels. It is kept in the repository on purpose: detecting that fault from the data alone is the point of the accompanying paper, and the recording is what makes the outlier-detection result reproducible.

The external validation set used in the paper is the public **BCI Competition IV Dataset 2a** (9 subjects, 22 EEG + 3 EOG channels, 250 Hz), available at <https://www.bbci.de/competition/iv/>. It is not redistributed here.

## Pipeline

**1. Cleaning and imputation.** Missing values in EEG channels are filled with a KNN imputer. Channels containing more than 70 % zeros are treated as disconnected electrodes and dropped. Remaining zeros are reinterpreted as missing data and imputed a second time. Non-EEG columns pass through untouched.

**2. Spectral inspection.** Welch's method (Hamming window, 1024 samples) produces a power spectral density plot per channel, with all three subjects overlaid and the standard EEG bands shaded — a visual sanity check before any modeling.

**3. Bandpass filtering.** A fourth-order Butterworth filter applied with `filtfilt` (zero phase) isolates the 8–30 Hz range, where motor-related Alpha and Beta activity lives.

**4. Feature extraction.** One-second sliding windows with 50 % overlap. For each window and channel, mean power in the Alpha (8–13 Hz) and Beta (13–30 Hz) bands is computed, producing `2 × n_channels` features per window.

**5. Quality control.** Two outlier-removal steps run before modeling. At subject level, recordings flagged as acquisition failures are dropped (`EXCLUIR_SUJETOS`); Subject 3 is excluded by default because of its hardware fault. At window level, the total spectral energy of each window is computed and the top 2 % (98th percentile) is discarded (`PERCENTIL_ENERGIA`), removing broadband transient artifacts that the bandpass filter does not attenuate. Both steps are unsupervised and configurable from Section 1.

**6. Normalization.** `RobustScaler` (median and interquartile range) is fitted per subject rather than globally, so differences in absolute signal amplitude between recordings do not dominate the clustering, and any extreme value surviving the energy filter does not drag the normalization.

**7. Dimensionality reduction.** Four 2D projections — PCA, t-SNE, UMAP and Kernel PCA (RBF) — colored by subject, to verify that the features carry usable structure before clustering. This is also where the Subject 3 fault becomes visible: its windows collapse into a tight, isolated cloud instead of mixing with the rest.

**8. Clustering and evaluation.** An 80/20 train/test split feeds a scikit-learn `Pipeline` combining UMAP reduction with K-Means (k = 5). Fitting the reducer inside the pipeline ensures it learns only from training data. Cluster quality is measured with the Silhouette score on held-out windows and visualized with a silhouette plot.

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

Open the notebook in Jupyter or Google Colab, download the CSV files from Zenodo (see [Data](#data)), and edit the configuration cell in Section 1:

```python
DIR_ENTRADA = '/content/'          # folder containing the three CSV files
DIR_SALIDA  = '/content/cleaned_data/'
```

Every parameter — sampling rate, frequency bands, window length, overlap, cluster count, random seed — is defined in that single cell. Then run the notebook top to bottom.

To apply the trained model to new recordings, pass them through the same cleaning, filtering, extraction and normalization steps, then call:

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
| 9 | Quality control, normalization and final matrix |
| 9.1 | Exclusion of faulty subjects (macro-level outliers) |
| 9.2 | Spectral-energy window filter (micro-level outliers) |
| 9.3 | Robust scaling and final matrix `X` |
| 10 | PCA · t-SNE · UMAP · Kernel PCA |
| 11 | UMAP + K-Means pipeline |
| 12 | Silhouette evaluation and cluster plots |
| 13 | Summary |

## Reproducing the paper

The notebook implements the pipeline exactly as described in the manuscript: KNN imputation, 8–30 Hz Butterworth filtering, Welch PSD features over Alpha and Beta, exclusion of Subject 3, removal of the top 2 % of windows by spectral energy, per-subject robust scaling, and an 80/20 UMAP + K-Means pipeline evaluated with the silhouette coefficient on held-out windows.

Cell outputs are **cleared** in this repository so that no stale numbers are shown alongside the current code. Run the notebook top to bottom on the recordings archived on Zenodo to regenerate every figure and metric.

To reproduce the exploratory projections of Figure 2 in the paper — the ones that reveal the Subject 3 fault — set `EXCLUIR_SUJETOS = []` and run Sections 1 through 10. Subject 3 must be present for that plot to show the fault.

The paper also reports a run of this same pipeline on the public BCI Competition IV Dataset 2a. That run was carried out only to verify that the method does not flag outliers in a dataset acquired under controlled conditions, and it is not part of this repository.

## Notes

Cluster labels are arbitrary identifiers. The pipeline finds five separable states in the signal, but mapping them to specific physical movements requires labeled ground truth. The Silhouette score reports internal cluster cohesion and separation, not classification accuracy against real movements.

Results depend on the random seed. UMAP and K-Means are both stochastic; the seed is fixed in the configuration cell so runs are reproducible.

## Citation

If you use this code or data, please cite:

```bibtex
@article{martinezperez2026eeg,
  title   = {Unsupervised Outlier Detection Pipeline for Low-Cost EEG Signals
             Based on Dimensionality Reduction and K-Means Clustering},
  author  = {Martinez-Perez, Samuel and Jimenez-Turizo, Enso Jovanis and
             De Avila-Pereira, Hernando Jose},
  journal = {Sensors},
  year    = {2026},
  note    = {Manuscript submitted for publication}
}
```

## Authors

- **Samuel Martinez-Perez** — Universidad Simón Bolívar, Barranquilla, Colombia · [ORCID 0009-0004-9630-5235](https://orcid.org/0009-0004-9630-5235) · samuel.martinezp@unisimon.edu.co
- **Enso Jovanis Jimenez-Turizo** — Universidad Simón Bolívar, Barranquilla, Colombia
- **Hernando Jose De Avila-Pereira** — Universidad Simón Bolívar, Barranquilla, Colombia

## License

Released under the MIT License. See [LICENSE](LICENSE).rbitrary identifiers. The pipeline finds five separable states in the signal, but mapping them to specific physical movements requires labeled ground truth. The Silhouette score reports internal cluster cohesion and separation, not classification accuracy against real movements.
