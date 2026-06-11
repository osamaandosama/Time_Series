# Time-Series Anomaly Detection in Network Traffic


## Overview

This project utilizes the **CIC-IDS2017** dataset from the Canadian Institute for Cybersecurity, which includes real-world network traffic captures of both benign activity and various attacks like DDoS and Brute Force. To enable deep learning-based detection, we preprocess the raw flow data into time-series sequences using a **sliding window** technique. By focusing on temporal features such as flow duration and packet inter-arrival times, we train **LSTM** models to recognize "normal" network rhythms and identify anomalous spikes or patterns that signify potential intrusions.

The work is organized into two iterations:

1. **Baseline Approach** — first exploration: timestamp reconstruction, full EDA, a structured processing pipeline, and an initial univariate LSTM forecaster.
2. **Refined Approach** — a cleaner, reproducible pipeline producing a single engineered dataset, followed by classical unsupervised baselines (Isolation Forest, Local Outlier Factor), an unsupervised **LSTM Autoencoder**, and a supervised **Random Forest** for comparison.

- **Dataset Access:** https://www.unb.ca/cic/datasets/ids-2017.html
- *Dataset:* https://drive.google.com/file/d/1woPLSY8BISofk1aH_LiulXF07jMbiYZZ/view?usp=sharing
- *Dataset After TimeStamp:* https://drive.google.com/file/d/1P4mUR4r_ZPSoBgE1OLSRzogA3LleQGpR/view?usp=sharing
- *Dataset After KNN:* https://drive.google.com/file/d/1XMhi16vGgIYBS6P8snhbR-C9b47WBLSs/view?usp=sharing

---

## Dataset

The dataset spans five days of captured traffic (Monday, July 3 – Friday, July 7, 2017) as documented for CIC-IDS2017. After merging and cleaning, it contains roughly **2.83 million flows** with an attack rate of about **19.7%**.

| Stage | Description | Link |
|-------|-------------|------|
| **Official source** | Canadian Institute for Cybersecurity dataset page | https://www.unb.ca/cic/datasets/ids-2017.html |
| **Raw merged dataset** | All daily CSVs merged into `merged.csv` | [Download](https://drive.google.com/file/d/1woPLSY8BISofk1aH_LiulXF07jMbiYZZ/view?usp=sharing) |
| **After Timestamp** | Synthetic timestamp column added for temporal analysis | [Download](https://drive.google.com/file/d/1P4mUR4r_ZPSoBgE1OLSRzogA3LleQGpR/view?usp=sharing) |
| **After KNN** | Missing / infinite values imputed via KNN, column-reduced | [Download](https://drive.google.com/file/d/1XMhi16vGgIYBS6P8snhbR-C9b47WBLSs/view?usp=sharing) |

> **Note:** The `Data/` folder is git-ignored. Download the CSV(s) you need and place them where each notebook expects them (paths are defined at the top of every notebook).

---

## Repository Structure

```
Time_Series/
├── README.md
├── log.md                       # Commit / contribution history
├── .gitignore
└── notebooks/
    ├── Baseline_Approach/
    │   ├── DataTimeStamp.ipynb            # Reconstruct a temporal axis from flow durations
    │   ├── Data_Processing_Pipeline.ipynb # End-to-end pipeline as reusable functions
    │   ├── DatasetAfterKNN.ipynb          # KNN imputation + feature reduction
    │   ├── EDA.ipynb                      # Exploratory analysis, feature importance, correlations
    │   └── Time_col_on_LSTM.ipynb         # First (univariate) LSTM forecasting experiment
    │
    └── Refined_Approach/
        ├── Second_Approach_Data_Process.ipynb  # Produces engineered_network_traffic.csv
        ├── Baselines.ipynb                      # Isolation Forest & Local Outlier Factor
        ├── LSTM_Autoencoder.ipynb               # Unsupervised reconstruction-based detector
        └── RF.ipynb                             # Supervised Random Forest comparison
```

---

## Methodology

### 1. Data Processing & Feature Engineering
- **Cleaning:** strip and standardize column names, drop irrelevant columns, remove duplicates, and replace `inf` / `-inf` with `NaN`.
- **Imputation:** missing values filled with a **KNN imputer** (`n_neighbors = 5`).
- **Temporal axis:** because the raw flows lack a usable timestamp, a synthetic one is generated, spread uniformly across the documented 5-day window (`2017-07-03 08:00` → `2017-07-07 17:00`), so attacks can be visualized over time.
- **Feature selection:** a focused subset covering flow, byte, packet, and port metrics.
- **Scaling:** Min-Max scaling to `[0, 1]`.
- **Rolling statistics:** rolling mean and standard deviation (window of 5–10 rows) to capture short-term bursts.
- **Sliding-window segmentation:** sequences are built into shape `(N, seq_len, n_features)` to feed the temporal models.

### 2. Exploratory Data Analysis
The EDA notebook covers flow-duration behavior over time (log-scale, multi-order-of-magnitude variance), attack distribution over the timeline (DDoS and PortScan dominate, DoS Hulk is continuous while GoldenEye / FTP-Patator come in bursts), feature distributions, a Random-Forest feature-importance ranking, and correlation heatmaps for the top features.

### 3. Models

| Model | Type | Idea |
|-------|------|------|
| **Isolation Forest** | Unsupervised baseline | Randomly partitions the feature space; anomalies are isolated in fewer splits. |
| **Local Outlier Factor** | Unsupervised baseline | Flags points whose local density is much lower than their neighbors'. |
| **LSTM Autoencoder** | Unsupervised, deep | Trained on normal traffic only; high reconstruction error → anomaly. Encoder → latent bottleneck → decoder. |
| **Random Forest** | Supervised | Trained with SMOTE on the labeled, time-engineered features as a strong supervised reference point. |

**Shared configuration (Refined Approach):** `seq_len = 30` (15 for RF), `train_frac = 0.60` (time-ordered, no shuffle), `hidden_dim = 64`, `latent_dim = 16`, `num_layers = 2`, `dropout = 0.2`.

---

## Results

Metrics are reported on the held-out test split. **AUPRC** (area under the Precision-Recall curve) is emphasized because the data is imbalanced.

| Detector | ROC-AUC | AUPRC | Attack Recall | Attack Precision |
|----------|:-------:|:-----:|:-------------:|:----------------:|
| Isolation Forest | 0.807 | 0.447 | — | — |
| Local Outlier Factor | 0.781 | 0.695 | 0.03 | 0.72 |
| LSTM Autoencoder | 0.596 | 0.503 | 0.19 | 0.87 |
| Random Forest *(supervised)* | — | — | 0.70 | 0.63 |

> Supervised Random Forest on the test set: **Accuracy 0.760**, **Precision 0.635**, **Recall 0.698**, **F1 0.665** (at the tuned threshold of `0.0146`).

**Takeaways:**
- The unsupervised detectors trade recall for precision: the LSTM Autoencoder is highly precise on attacks (0.87) but conservative in recall, while LOF achieves the strongest AUPRC among the unsupervised methods.
- The supervised Random Forest offers the most balanced precision/recall once a threshold is tuned, serving as an upper reference for what the unsupervised models approach without labels.

---

## Getting Started

### Requirements

```bash
pip install torch scikit-learn pandas numpy matplotlib seaborn imbalanced-learn gdown
```

(The `LSTM_Autoencoder.ipynb` uses **PyTorch**; the early `Time_col_on_LSTM.ipynb` experiment uses **TensorFlow/Keras**.)

### Running

1. Clone the repository:
   ```bash
   git clone https://github.com/osamaandosama/Time_Series.git
   cd Time_Series
   ```
2. Download the dataset stage you need from the table above and place it where the notebook expects it (e.g. `engineered_network_traffic.csv` for the Refined Approach). Several notebooks can auto-download via `gdown`.
3. Open the notebooks in Jupyter or Google Colab and run top to bottom:
   - Start with `Refined_Approach/Second_Approach_Data_Process.ipynb` to regenerate the engineered dataset, **or** download it directly.
   - Then run `Baselines.ipynb`, `LSTM_Autoencoder.ipynb`, and `RF.ipynb`.

---

## Contributors

- **Osama** ([@osamaandosama](https://github.com/osamaandosama)) — LSTM modeling, timestamp/KNN dataset stages
- **Fatma Ahmed** — data processing pipeline, EDA, refined data processing
- **Manar Fathy Elshenawy** — classical baselines, project structure, documentation

---

## Acknowledgments

Dataset provided by the **Canadian Institute for Cybersecurity (CIC)**, University of New Brunswick — CIC-IDS2017.
