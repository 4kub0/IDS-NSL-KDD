# Hierarchical Intrusion Detection System (H-IDS) on NSL-KDD

An intelligent, multi-tier Network Intrusion Detection System built on the **NSL-KDD** benchmark dataset. The system is designed around a **hierarchical edge-to-cloud cascade** that balances line-rate packet inspection latency at the edge with deep multiclass threat categorization in the cloud.

---

## 1. System Architecture

Real-world high-throughput networks cannot run complex multiclass models on every incoming packet without overflowing hardware ring buffers. This project decouples threat detection into two coordinated tiers:

```
                  Incoming Network Traffic (NSL-KDD)
                                   │
                                   ▼
         ┌──────────────────────────────────────────────────┐
         │       LEVEL 1: Edge Binary Filter (Gateway)       │
         │  • Binary Classification: Normal vs. Attack      │
         │  • Strict SLA: <= 50 µs latency, >= 99.5% Recall  │
         │  • Models: Decision Tree, Extra Trees            │
         └─────────────────────────┬────────────────────────┘
                                   │
                     ┌─────────────┴─────────────┐
                     │                           │
              [ Normal Traffic ]          [ Flagged Attack ]
                     │                           │
                     ▼                           ▼
                 Permitted             ┌──────────────────────────────────────────┐
                                       │    LEVEL 2: Cloud Multiclass Classifier   │
                                       │  • Detailed Attack Triage:               │
                                       │    DoS, Probe, R2L, U2R                  │
                                       │  • Optimized for Imbalanced Macro-F1     │
                                       │  • Models: Random Forest, LightGBM,      │
                                       │    XGBoost, CatBoost                     │
                                       └──────────────────────────────────────────┘
```

---

## 2. Model Tracks & Registry Architecture

To prevent code conflicts and maintain clean separation of concerns, models inherit from a shared contract and register dynamically at runtime:

- **Shared Contract ([`src/contracts.py`](src/contracts.py))**: Defines `BaseIDSModel` with `.fit(X, y)`, `.predict(X)`, and `.evaluate(X, y)`.
- **Shared Mixin ([`src/models/base.py`](src/models/base.py))**: Implements `IDSModelMixin` with high-resolution `time.perf_counter()` latency benchmarking and confusion-matrix FPR calculation.
- **Dynamic Registry ([`src/models/__init__.py`](src/models/__init__.py))**: Automatically scans model subclasses and populates `MODEL_REGISTRY`.

### Collaborative Model Tracks

| Track | Owner | Assigned Models | Hardware Target |
|---|---|---|---|
| **Tree & Instance Ensembles** | **`4kub0`** | `DecisionTree`, `RandomForest`, `ExtraTrees`, `BaselineNB` | Level 1 Edge Gateway ($\le 50 \mu s$ SLA) & Level 2 Cloud |
| **Gradient Boosted Trees** | **`lemonkartikeya`** | `LightGBM`, `XGBoost`, `CatBoost` | Level 2 Cloud / Fog Multiclass Categorizer |

---

## 3. Repository Structure

```
IDS-NSL-KDD/
├── .agent/                      # Multi-developer agent coordination logs
│   ├── sync_4kub0.md            # Track A progress & decisions
│   └── sync_lemonkartikeya.md   # Track B progress & decisions
├── boosting_dev/                # Research scratchpad for gradient boosting & feature selection
│   └── feature_selection_evolopy.ipynb
├── results/                     # Benchmark comparison tables, ROC curves, and confusion heatmaps
│   ├── comparison_results.csv   # Baseline experimental benchmarks
│   └── *.png
├── src/
│   ├── config.py                # Dataset paths, column mappings, attack categories, seed
│   ├── contracts.py             # BaseIDSModel abstract interface
│   ├── models/
│   │   ├── __init__.py          # Dynamic MODEL_REGISTRY auto-discovery
│   │   ├── base.py              # IDSModelMixin (latency profiling, macro-F1, FPR)
│   │   ├── tree_models.py       # [Track A] DecisionTree, RandomForest, ExtraTrees, NaiveBayes
│   │   └── boosting_models.py   # [Track B] LightGBM, XGBoost, CatBoost
│   ├── data_loader.py           # NSL-KDD ingestion & validation
│   ├── preprocessing.py         # LabelEncoding and feature scaling
│   └── smote_handler.py         # Class imbalance balancing
├── tests/
│   ├── test_tree_models.py      # Unit tests for tree models & registry contract (14 tests)
│   └── test_ids_pipeline.py     # Pipeline validation tests
├── implementation_plan.md       # Full multi-step collaborative architecture roadmap
├── PROJECT_REPORT.md            # Detailed report on preliminary baseline experiments
├── pytest.ini                   # Pytest configuration (pythonpath = .)
└── requirements.txt             # Project dependencies
```

---

## 4. Setup & Installation

### Prerequisites
- Python 3.10+
- Git

### Installation
```bash
# Clone the repository
git clone https://github.com/4kub0/IDS-NSL-KDD.git
cd IDS-NSL-KDD

# Install dependencies
pip install -r requirements.txt
```

---

## 5. Running Tests

Unit tests verify that every model complies with the `BaseIDSModel` contract, handles binary (Level 1) and multiclass (Level 2) targets, supports hyperparameter overrides, and reports microsecond latency:

```bash
# Run all model unit tests
pytest tests/test_tree_models.py -v

# Or run bare pytest from repository root
pytest
```

---

## 6. Dataset (NSL-KDD)

The project benchmarks against the **NSL-KDD** dataset, an improved standard over KDD Cup 1999 that removes duplicate records and balances difficulty:

- **Features**: 41 continuous and categorical network traffic attributes (byte counts, error rates, protocol types).
- **Difficulty Column**: Dropped at ingestion to prevent data leakage.
- **Attack Classes (5)**:
  - **Normal**: Legitimate network traffic.
  - **DoS**: Denial-of-Service attacks (`neptune`, `smurf`, etc.).
  - **Probe**: Reconnaissance and port scans (`ipsweep`, `portsweep`, `nmap`).
  - **R2L**: Remote-to-Local unauthorized access (`guess_passwd`, `warezmaster`).
  - **U2R**: User-to-Root privilege escalation (`buffer_overflow`, `rootkit`).

---

## 7. Documentation & Roadmap

- **[implementation_plan.md](implementation_plan.md)**: Full 6-step engineering plan covering evolutionary feature selection (`EvoloPy`), capped SMOTE, and the hierarchical cascade triage engine.
- **[PROJECT_REPORT.md](PROJECT_REPORT.md)**: Comprehensive report documenting the initial flat baseline experiments that informed our two-tier design.
