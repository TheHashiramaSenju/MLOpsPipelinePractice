<div align="center">

# 🧪 MLOps Pipeline Practice

**A hands-on journey through the full MLOps lifecycle — DVC pipelines, experiment tracking, DVCLive logging, and GCS remote storage.**

</div>

***

> 🏫 **Credits + Ownership**: Core structure inspired by Vikash Das's MLOps classes — but built out, broken, debugged, and extended with my own experiments, mistakes, and learning moments. Every bug I hit is documented here as a lesson.

***

## 📋 Table of Contents

- [What This Project Is](#-what-this-project-is)
- [Pipeline Architecture](#-pipeline-architecture)
- [Tech Stack](#-tech-stack)
- [Project Structure](#-project-structure)
- [Getting Started](#-getting-started)
- [DVC Pipeline Deep Dive](#-dvc-pipeline-deep-dive)
- [Experiment Tracking with DVCLive](#-experiment-tracking-with-dvclive)
- [Remote Storage — GCS Bucket](#-remote-storage--gcs-bucket)
- [Key Learnings & Mistakes Made](#-key-learnings--mistakes-made)
- [YAML Config Notes](#-yaml-config-notes)

***

## 🔍 What This Project Is

This is a **full ML pipeline practice project** built to get hands-on experience with the real-world MLOps toolchain. It goes far beyond "just training a model" — it covers:

- Building a **reproducible, stage-based DVC pipeline**
- **Versioning models and data** with DVC so every experiment is trackable
- Using **DVCLive** for real-time metrics and experiment logging
- **Running, managing, and deleting experiments** from the CLI
- Pushing versioned artifacts to **Google Cloud Storage (GCS)** using a custom IAM service account
- Writing proper **`dvc.yaml` and `params.yaml`** configs — and learning what breaks when you get paths wrong

The model itself is a **Random Forest classifier** trained on a standard dataset, but the *pipeline orchestration* around it is the real learning objective.

***

## 🏗️ Pipeline Architecture

The pipeline has **5 sequential stages**, each defined as a DVC stage in `dvc.yaml`:

```
Raw Data
   │
   ▼
┌─────────────────────┐
│  1. data_ingestion  │  ← Fetches & splits raw data (test_size configurable)
└─────────────────────┘
   │
   ▼
┌──────────────────────────┐
│  2. data_preprocessing   │  ← Cleans, handles nulls, encodes categoricals
└──────────────────────────┘
   │
   ▼
┌───────────────────────────┐
│  3. feature_engineering   │  ← TF-IDF / feature selection (max_features param)
└───────────────────────────┘
   │
   ▼
┌──────────────────────┐
│  4. model_building   │  ← Trains RandomForest (n_estimators, random_state)
└──────────────────────┘
   │
   ▼
┌───────────────────────┐
│  5. model_evaluation  │  ← Logs metrics via DVCLive → reports/metrics.json
└───────────────────────┘
```

DVC tracks **dependencies, parameters, outputs, and metrics** across all stages. Re-running `dvc repro` only re-executes stages whose inputs have changed.

***

## 🛠️ Tech Stack

| Tool | Purpose |
|------|---------|
| **DVC** | Pipeline orchestration, data & model versioning |
| **DVCLive** | Experiment metrics logging (accuracy, loss, etc.) |
| **GCS (Google Cloud Storage)** | Remote artifact storage (data, models) |
| **Google Cloud IAM** | Custom service account with least-privilege access to GCS |
| **params.yaml** | Central config for all hyperparameters |
| **dvc.yaml** | Pipeline stage definitions |
| **Python + scikit-learn** | ML model (RandomForestClassifier) |
| **Git** | Code versioning (DVC sits on top of this) |

***

## 📁 Project Structure

```
MLOpsPipelinePractice/
│
├── src/                        # All pipeline stage scripts
│   ├── data_ingestion.py
│   ├── data_preprocessing.py
│   ├── feature_engineering.py
│   ├── model_building.py
│   └── model_evaluation.py
│
├── data/                       # DVC-tracked data (not in Git)
│   ├── raw/                    # Output of data_ingestion stage
│   ├── interim/                # Output of data_preprocessing stage
│   └── processed/              # Output of feature_engineering stage
│
├── models/
│   └── model.pkl               # DVC-tracked trained model artifact
│
├── reports/
│   └── metrics.json            # Evaluation metrics output
│
├── dvclive/                    # DVCLive experiment logs
│   ├── metrics.json
│   ├── params.yaml
│   └── plots/metrics/
│
├── experiments/                # Saved experiment snapshots
├── dvc.yaml                    # Pipeline stage definitions
├── dvc.lock                    # Locked stage hashes (auto-generated)
├── params.yaml                 # Hyperparameter config
├── .dvc/                       # DVC internals (remote config, cache)
├── .dvcignore
└── .gitignore
```

***

## 🚀 Getting Started

### Prerequisites

- Python 3.10+
- Git
- A Google Cloud project with a GCS bucket
- A GCP service account JSON key with `Storage Object Admin` role

### Installation

```bash
# Clone the repo
git clone https://github.com/TheHashiramaSenju/MLOpsPipelinePractice.git
cd MLOpsPipelinePractice

# Create and activate a virtual environment
python -m venv venv
source venv/bin/activate        # Linux/macOS
# venv\Scripts\activate         # Windows

# Install dependencies
pip install dvc dvclive dvc-gs scikit-learn pandas
```

### Configure GCS Remote

```bash
# Set your remote (replace with your actual bucket name)
dvc remote add -d myremote gs://your-bucket-name/path

# Authenticate using your service account key
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/your-service-account.json
```

### Run the Full Pipeline

```bash
dvc repro
```

This runs all 5 stages in order, caching outputs at each step.

***

## 🔄 DVC Pipeline Deep Dive

### Checking Pipeline Status

```bash
dvc status           # Shows which stages are outdated
dvc dag              # Visualizes the pipeline DAG
```

### Running Experiments

```bash
# Run with default params
dvc exp run

# Run with overridden param
dvc exp run --set-param model_building.n_estimators=200

# List all experiments
dvc exp show

# Delete a specific experiment
dvc exp remove <exp-name>

# Clean up all stale/temp experiments
dvc exp gc
```

### Comparing Experiments

```bash
dvc exp show --sort-by metrics.json:accuracy
```

### Pushing & Pulling Artifacts

```bash
dvc push        # Upload data/models to GCS remote
dvc pull        # Download cached artifacts from GCS remote
```

***

## 📊 Experiment Tracking with DVCLive

DVCLive is integrated inside `model_evaluation.py` to log metrics at each evaluation step. It auto-generates plots, JSON metric files, and integrates directly with `dvc exp show`.

### What gets logged

- `accuracy`, `precision`, `recall`, `f1_score`
- All params from `params.yaml`
- Step-wise plots under `dvclive/plots/metrics/`

### Viewing Live Metrics

```bash
# After running an experiment
dvc exp show

# Open the DVCLive report (if using VS Code extension or browser)
# dvclive/report.html
```

### The `params.yaml` file controls everything

```yaml
data_ingestion:
  test_size: 0.2

feature_engineering:
  max_features: 1000

model_building:
  n_estimators: 100
  random_state: 42
```

Changing any value here and re-running `dvc repro` will only re-execute the affected downstream stages.

***

## ☁️ Remote Storage — GCS Bucket

Artifacts (raw data, processed data, and `model.pkl`) are stored in a **Google Cloud Storage bucket**. The remote is configured using a **custom IAM service account** — not the default application credentials — following the principle of least privilege.

### IAM Setup Done

- Created a dedicated GCP **service account** for this project
- Granted only `Storage Object Admin` on the specific bucket (not project-wide)
- Downloaded the JSON key and set it as `GOOGLE_APPLICATION_CREDENTIALS`
- Configured DVC remote with `dvc remote modify`

### DVC Remote Config (`.dvc/config`)

```ini
[core]
    remote = myremote
['remote "myremote"']
    url = gs://your-bucket-name/mlops-practice
```

```bash
# Push all tracked files to GCS
dvc push

# Pull them back on a fresh clone
dvc pull
```

***

## 🧠 Key Learnings & Mistakes Made

This section is the **most honest part of the README**. Every mistake below was a real lesson.

### 🔴 Absolute vs Relative Paths in Linux (Big One)

This caused silent pipeline failures. In Linux:

| Type | Example | Behaviour |
|------|---------|-----------|
| **Absolute path** | `/path/to/data` | Always starts from filesystem root `/`. Works from anywhere. |
| **Relative path** | `path/to/data` | Relative to **current working directory**. Breaks if you run from a different folder. |

**In `dvc.yaml`**, always use **relative paths** — DVC resolves them relative to the project root, not your shell's `$PWD`. Using absolute paths like `/home/user/project/data/raw` breaks reproducibility across machines entirely.

```yaml
# ❌ Wrong — absolute path, breaks on any other machine
outs:
  - /home/yourname/MLOpsPipelinePractice/data/raw

# ✅ Correct — relative path, works everywhere
outs:
  - data/raw
```

### 🟡 DVC Lock File Confusion

`dvc.lock` is auto-generated. **Never manually edit it.** If it gets out of sync with your pipeline, just delete it and run `dvc repro` to regenerate.

### 🟡 `dvc exp run` vs `dvc repro`

| Command | When to use |
|---------|------------|
| `dvc repro` | Reproduce the pipeline (respects cache, skips unchanged stages) |
| `dvc exp run` | Run as a tracked experiment (saves to experiment history, creates a Git stash internally) |

Use `dvc exp run` when you want to compare this run against previous ones later.

### 🟡 Deleting Experiments the Right Way

Running lots of experiments creates noise. To clean up:

```bash
dvc exp remove exp-abc123          # Remove specific experiment by name
dvc exp gc --workspace             # Remove all experiments not referenced by a branch/tag
```

> Don't just `git stash drop` DVC experiments — it leaves orphaned DVC cache entries.

### 🟢 YAML Indentation is Not Optional

YAML fails **silently or with cryptic errors** when indentation is off. Two spaces, consistently. No tabs.

```yaml
# ❌ Broken — mixed indentation
model_building:
n_estimators: 100    # this is NOT nested under model_building

# ✅ Correct
model_building:
  n_estimators: 100
```

### 🟢 IAM Scope Matters

Initially used a broad project-level `Storage Admin` role. Later narrowed it to `Storage Object Admin` scoped only to the specific bucket — this is the right production practice.

***

## 📝 YAML Config Notes

The `dvc.yaml` pipeline and `params.yaml` together act as the **single source of truth** for the entire ML workflow.

- **`dvc.yaml`** defines *what runs* and *in what order*
- **`params.yaml`** defines *with what configuration*
- **`dvc.lock`** records *exact hashes* of every input/output for reproducibility

Any time you add a new hyperparameter, add it to `params.yaml` first, then reference it in the relevant stage in `dvc.yaml` under `params:`.

***

<div align="center">

Built while learning by breaking things. 🔥

**[TheHashiramaSenju](https://github.com/TheHashiramaSenju)**

</div>