# Wildlife Reserve Control & Detection System

Capstone project — Group 10, WCD Data Science Bootcamp.

Detecting and classifying wildlife activity for the King Abdulaziz Royal Reserve
(KARR, 103 camera traps) using **iWildCam2020-wilds** as a declared public stand-in.

## Layout

```
├── data/              # git-ignored; see data/README.md to obtain it
│   ├── raw/           # exactly as downloaded, never hand-edited
│   └── processed/     # regenerable outputs of src/
├── notebooks/
│   ├── 01_eda_ddcr_waterholes.ipynb   # EDA on the DDCR merged dataset
│   ├── 02_iwildcam_acquisition.ipynb  # iWildCam download / subsetting
│   └── eda.ipynb                      # main EDA, organised around the project objective
├── src/
│   ├── merge_wildlife_insights.py     # 4 raw tables -> 2 merged datasets
│   └── awk/                           # feature-engineering pipeline
├── scripts/
│   └── get_data.py                    # dataset acquisition + presence check
├── docs/
│   ├── ddcr_dataset.md                # what the merge does and why
│   ├── FEATURES.md                    # engineered feature reference
│   └── *.pptx                         # project proposal
├── models/            # git-ignored weights
└── reports/figures/   # charts ARE tracked
```

## Setup

```bash
py -m venv .venv
.venv\Scripts\activate
py -m pip install -r requirements.txt
py scripts/get_data.py --check
```

On this machine the launcher is `py`, not `python` — plain `python` hits the
Microsoft Store stub. Note the interpreter is 3.14, which is ahead of PyTorch's
published wheels; if `pip install torch` fails, create the venv against a 3.12
interpreter instead (`py -3.12 -m venv .venv`).

## Data

Neither dataset is in this repo. **Read `data/README.md` before doing
anything else** — the DDCR export is private, and iWildCam is ~101 GB behind a
rules acceptance you must click yourself.

## Conventions

- `data/raw/` is append-only. Every transformation is a script in `src/`, so any
  result can be reproduced from raw by re-running.
- Subsets and splits are seeded (`--seed 42`) so every teammate gets identical
  rows. Never share a derived split as a file when a seed will do.
- `.gitignore` excludes `*.csv` and model weights globally. If you genuinely need
  to commit a small reference table, force it: `git add -f path/to/lookup.csv`.
- Restart-and-run-all before committing a notebook, so outputs match the code.

## Team

| Name | Role |
|---|---|
| Abdulrahman Binshannar | |
| | |
| | |




## EDA (`notebooks/eda.ipynb`)

![EDA headline figures](reports/figures/00_summary.png)

**One dataset: iWildCam2020-wilds** (203,029 images, 323 camera locations, 182 classes,
Jan 2013 – Nov 2015). The DDCR Wildlife Insights export has been dropped.

> iWildCam is a declared stand-in for KARR. Every KARR figure is a **double extrapolation** —
> across camera count *and* across biome (forest/savanna to Saudi desert). Desert heat shimmer
> drives false triggers up, so the blank rate is the figure least likely to transfer.

Official WILDS splits are used throughout; the metric is **macro F1**.

### Sections

| Section | Questions | Key result |
|---|---|---|
| **S0** Data | release contents · metadata audit · official splits | images present (11.2 GB); **no coordinates, no bounding boxes** |
| **S1** Workload | volume · waste · review cost · bursts · window sweep | 34% blank, 5.56 frames/burst, **43% of bursts blank end to end** |
| **S2** Targets | species · vocabulary · diel · seasonal · animal size | **median animal is 1.6% of frame**; 81 diurnal / 38 crepuscular / 62 nocturnal |
| **S3** Stations | effort · richness · RAI · ranked table · species×location RAI | RAI reorders mid-table (Spearman 0.91); **42% of matrix cells reportable** |
| **S4** Constraints | leakage · imbalance · infrared · **class survival** · domain gap · banner | **only 102 of 182 classes reach OOD test**; 46% infrared; **camera ID burned into pixels** |
| **S5** Delivery | reviewer-hours · bandwidth · alerts · benchmark | 94% review saving · 88% bandwidth · target **ERM 31.0 macro F1** |

### Deck corrections this EDA forces

| Slide | Says | Should say |
|---|---|---|
| Deliverables | spatial hotspot map | **undeliverable — no coordinates**; ranked station table (S3.4) |
| ROI | edge filtering cuts data "up to 60%" | blanks alone ~33%; **blanks + burst de-duplication ~88%** (S5.2) |
| Success Criterion 2 | benchmark vs a platform classifier | **match or beat ERM's 31.0 macro F1 on WILDS OOD test** (S5.4) |
| Modelling | CNN on full frames | median animal ~1.6% of frame — **detect-then-crop or high resolution** (S2.5) |
| Preprocessing | *(absent)* | **crop top and bottom 5%** to remove the burned-in camera banner (S4.6) |
| Metric | accuracy | **macro F1**; accuracy overstates by >2× here (S4.2, S5.4) |

### Running it

```bash
export IWILDCAM_DATA=/path/to/iwildcam_v2.0      # metadata.csv, categories.csv, COCO json
export IWILDCAM_ZIP=/path/to/iwildcam_v2.0.zip   # optional; needed for image-level sections
python scripts/build_cache.py                    # image-feature sample (~4 min, cached)
python scripts/build_fg.py                       # animal-footprint estimate (~10 s, cached)
jupyter lab notebooks/eda.ipynb
```

Image-level sections read a **location-stratified sample of 11,904 frames** straight from the
archive and cache the results, so the notebook re-runs in minutes rather than hours. Figures land
in `reports/figures/s*_*.png`.
