# Smart Logistics — Late Delivery Prediction System

An end-to-end machine learning system that predicts, at dispatch time, whether a
cold-chain shipment will miss its delivery promise — with an API, a dashboard,
explainability, monitoring and tests.

> **Data honesty notice.** The business data in this repository is **synthetic**.
> It imitates a Delhi-based cold-chain distributor (1.5M shipments, 12 hubs, two
> years of scans) so the system can be built and demonstrated without exposing
> anyone's commercial data. Nothing here is real shipment data, and no result in
> this repo should be read as a real-world logistics claim. See
> [`docs/DATASET.md`](docs/DATASET.md).

## Project status

| Phase | Scope | Status |
|---|---|---|
| 1 | Project setup + dataset generation | ✅ complete |
| 2 | Exploratory data analysis | ✅ complete |
| 3 | Preprocessing pipeline | ⏳ next |
| 4 | Baseline models (LR, DT, RF, XGBoost) | planned |
| 5 | Hyperparameter tuning + threshold economics | planned |
| 6 | SHAP explainability | planned |
| 7 | FastAPI service | planned |
| 8 | Streamlit dashboard | planned |
| 9 | Drift monitoring | planned |
| 10 | Tests, Docker, benchmarks | planned |

## Quickstart

Requires Python 3.11+.

```bash
git clone <your-repo-url> smart-logistics-late-delivery
cd smart-logistics-late-delivery

make run       # setup (first time) + data + tests + audit notebook, ~2 min
make eda       # run the Phase 2 EDA notebook and save its charts
make lab       # open the notebooks in JupyterLab
```

Expected output of `make data`:

```
Calibrated delay offset = -0.086 -> late rate 20.04%
Wrote shipments         rows=  1506000  84.1 MB
Wrote warehouse_scans   rows=  3396171  86.8 MB
Done in 34.7s. Late rate 20.04% | splits {'test': 0.248, 'train': 0.1758, 'validation': 0.2432}
```

Then open `notebooks/01_data_audit.ipynb` (locally or in Colab) to audit the data
and `notebooks/02_eda.ipynb` for the Phase 2 analysis.

## What is in the box

```
├── src/logistics/
│   ├── config.py              # paths, seed, simulation window, splits, business constants
│   ├── logging_utils.py
│   └── data/
│       ├── catalog.py         # 12 hubs, lanes, cargo types, vehicles, renames
│       ├── schema.py          # canonical columns + leakage contract + splits
│       └── synthesize.py      # the dataset simulator
├── scripts/generate_data.py   # CLI
├── tests/test_synthesize.py   # 21 tests over the data contract
├── notebooks/
│   ├── 01_data_audit.ipynb    # Phase 1: data contract audit
│   └── 02_eda.ipynb           # Phase 2: EDA + baseline
├── docs/
│   ├── DATASET.md             # source, design, defects, limitations
│   └── BASELINE.md            # measured baseline and the 20% target
├── data/raw/                  # generated, git-ignored
├── Makefile
└── requirements.txt
```

## The dataset at a glance

| Table | Rows | What it holds |
|---|---|---|
| `shipments` | 1,506,000 | dispatch, lane, cargo, vehicle, driver, conditions |
| `deliveries` | 1,506,000 | actual arrival, delay minutes, `is_late` |
| `warehouse_scans` | 3,396,171 | OUTBOUND / TRANSFER / INBOUND events |
| `hubs` | 12 | network master |
| `routes` | 120 | lanes with distance and reliability |
| `hub_renames` | 4 | old → new hub names with effective dates |

**Target:** `is_late = actual_delivery_ts > promised_delivery_ts + 15 min`.
Overall late rate 20.0%, rising from ~10% in the quiet season to a ~42% weekly
peak during the monsoon and 24.8% across the Sep–Dec test window.

The raw layer is intentionally messy — missing values, duplicate rows, three
time-zone conventions, out-of-order scans, renamed hubs and entry outliers — so
the Phase 3 pipeline has real work to do. Every defect is listed in
[`docs/DATASET.md` §6](docs/DATASET.md).

## Reproducibility

- One seed (`RANDOM_SEED`, default 42) controls the entire simulation.
- `data/raw/manifest.json` records seed, row counts, label rates and library
  versions for every generated dataset.
- `tests/test_synthesize.py` asserts that regenerating with the same seed gives
  identical data.
- Data is never committed: clone → `make data` reproduces it byte-for-byte on the
  same library versions.

## What the data says (Phase 2)

Everything below is measured, not assumed; `notebooks/02_eda.ipynb` reproduces it.

| Finding | Number |
|---|---|
| Overall late rate (1.5M shipments) | 20.0% |
| Late rate: train / validation / test | 17.6% / 24.3% / 24.8% |
| Weekly late-rate range | 7.9% → 42.0% (monsoon peaks in August) |
| Strongest rank correlations with `is_late` | weather 0.28, planned transit 0.20, stops 0.19, distance 0.17, traffic 0.16, driver experience −0.09 |
| Weather quartile Q1 → Q4 | 11.4% → 40.9% |
| Traffic quartile Q1 → Q4 | 12.2% → 28.8% |
| Hub spread | 13.5% (H09 Mohali) → 34.5% (H08 Jaipur) |
| Lane spread (lanes with ≥3,000 shipments) | 5.3% → 43.2% |
| Missingness by feature | 0.8%–11%, stable across splits |

**Baseline:** "always on time" scores 75.2% accuracy, zero recall, ROC-AUC 0.5
and PR-AUC 0.2480 (= the positive rate). The business metric is **PR-AUC**; the
project goal is to beat 0.2480 by at least 20% relative, i.e. **≥ 0.2976** on the
held-out test window. Details in [`docs/BASELINE.md`](docs/BASELINE.md).

## Development commands

```bash
make run         # one command: setup, data, tests, audit notebook (~2 min)
make setup       # environment only
make data        # full 1.5M dataset
make data-small  # 100k rows for fast iteration
make test        # pytest
make notebook    # run the audit notebook end-to-end and save its outputs
make eda         # run the EDA notebook end-to-end and save its outputs
make lab         # open the notebooks in JupyterLab
make lint        # ruff
make clean       # delete generated data
```

