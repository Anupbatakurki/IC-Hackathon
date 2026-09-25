# ◈ Campus Twin AI

**Dual-Twin Counterfactual Engine · Zero IoT Sensors · Local-Only**

A campus energy, water, and e-waste sustainability platform where **every modeled saving is verified against a parallel "ghost twin" simulation** — rather than being presented as an unverified spreadsheet estimate. If the live twin applies an approved action and the ghost twin doesn't, the gap between them *is* the measured saving.

---

## Table of Contents

- [Why this exists](#why-this-exists)
- [How it works](#how-it-works)
- [Features](#features)
- [Architecture](#architecture)
- [Data sources](#data-sources)
- [Models](#models)
- [Project structure](#project-structure)
- [Quick start](#quick-start)
- [Configuration](#configuration)
- [Deployment](#deployment)
- [Limitations & honest disclosures](#limitations--honest-disclosures)
- [Roadmap](#roadmap)
- [License](#license)

---

## Why this exists

Most campus sustainability dashboards report savings like this:

> "We turned the AC off for 3 hours, so we saved 4.5 kWh."

That number is a **claim**, not a measurement — it's computed from nameplate wattages and occupancy guesses. There is no way for a facilities manager, sustainability officer, or judge to verify the kWh was actually avoided.

Retrofitting a campus with IoT smart meters fixes this — at ₹15,000–₹40,000 per meter, plus gateway hardware, network provisioning, and months of deployment. Most institutions never do it.

**Campus Twin AI** demonstrates that counterfactual verification is achievable **without any new hardware**, using real-world historical smart-meter data (BDG2) as the baseline, real-world operating patterns represented through simulation, and one webcam for a single room's observed occupancy. The verification *mechanism* is identical to what you'd use on real meters — swapping in live hardware requires **zero changes to the rest of the codebase**.

---

## How it works

Two simulations run in lockstep:

```
┌──────────────────┐       ┌──────────────────┐
│   LIVE TWIN      │       │   GHOST TWIN     │
│  Actions applied │       │  No actions      │
│  Recommendations │       │  Same baseline   │
│  executed        │       │  same noise      │
└────────┬─────────┘       └────────┬─────────┘
         │  measured kW             │  baseline kW
         └──────────┬───────────────┘
                    ▼
         ┌────────────────────────┐
         │  delta = verified save │
         │  × 0.71 kg CO₂/kWh     │
         │  = CO₂ avoided (kg)    │
         └────────────────────────┘
```

The **OFF effect is perturbed ±15% per tick** rather than subtracted exactly, so the measured delta is a *noisy estimate* of a real effect — the way a smart-meter reading would behave — not a self-fulfilling readback of a constant the simulator wrote. Everything else in the platform hangs off this single mechanism.

---

## Features

### 🌐 Live tab
- YOLOv8n webcam stream — real-time person detection written into the same SQLite table the simulator uses
- Accelerated twin — 60× speed, 2-second tick, pause/resume/speed controls
- Telemetry charts — power (kW), water (m³), occupancy per room
- Distinguishable sources — camera-sourced rooms carry an emerald outline and tooltip
- Live event feed — recommendations, refusals, verifications, alerts

### 🧑‍⚖️ Decide tab
- Human-in-the-loop gate — every recommendation sits in `PENDING` until a person decides
- Refusal log with structured reasons (`duplicate_pending`, `protected_window`, `policy_override`)
- Alert dispatch (Telegram / in-app) with delivery status
- Duplicate suppression — a device already in the queue doesn't spam fresh refusals

### 🏛️ Departments tab
- Per-department verified kWh, energy CO₂, e-waste CO₂
- Energy CO₂ (measured) and e-waste CO₂ (estimated) shown **side-by-side — never stacked**
- Top-contributor highlighting, refusal counts by department

### ✓ Verify tab
- Live-vs-Ghost twin overlay — the core proof of the platform
- Verified ledger — every approved action, its measured delta, its CO₂ equivalent
- Per-building counterfactual chart

### 🎯 Allocation tab
- CP-SAT multi-objective solver balances five objectives: wasted seats, energy, relocations, peak load, department affinity
- Hard constraints: capacity, room type, no double-booking
- Before/after comparison on waste seats and energy

### ♻️ E-Waste tab
- Inventory scan — flags by EOL age, non-functional condition, or stale servicing
- Reuse-path assignment: refurbish · donate · parts-harvest · certified recycle
- CO₂ recovery — path-specific percentages applied to embodied-carbon figures
- Revenue estimate — material recovery × per-kg rate

### 🌱 Sustainability tab
- 14-day forward CO₂ forecast (LightGBM)
- Relatable equivalents — tree-years, km driven, LED bulb hours, phone charges
- Combined tree-years shown as a labelled sum of two components

### 📊 Evidence tab
- F1 / precision / recall / confusion matrix — scored against the simulator's own occupancy log
- Energy forecast metrics — per-building MAPE, MAE, residual σ
- Water forecast metrics — with dead-meter and sparse-coverage caveats surfaced inline

### 📄 Report tab
- One-click session report — Markdown + JSON download
- Executive summary — template fallback (works offline), optional Gemini enrichment
- Measured energy CO₂ and assumption-based e-waste CO₂ reported separately

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        GRADIO UI                             │
│   Live · Decide · Departments · Verify · Allocation ·        │
│   E-Waste · Sustainability · Evidence · Report               │
└────────────────────────┬─────────────────────────────────────┘
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                         │
│  Simulator · Decision Gate · Verifier · YOLOv8n · CP-SAT     │
│  E-Waste Planner · LightGBM energy · LightGBM carbon         │
└────────────────────────┬─────────────────────────────────────┘
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                      DATA LAYER                              │
│  SQLite (WAL mode, thread-safe)                              │
│    telemetry · ghost_telemetry · water_telemetry             │
│    occupancy · buildings · departments · rooms · devices     │
│    device_inventory · recommendations · refusals             │
│    verified_ledger · alerts_sent · e_waste_flags · reuse     │
│  Historical BDG2 CSVs (real-world electricity, water, metadata)         │
└──────────────────────────────────────────────────────────────┘
```

---

## Data sources

| Source | Usage | Licensing |
|---|---|---|
| **Building Data Genome Project 2 (BDG2)** | Historical hourly electricity and water telemetry for 5 education buildings (14,142 rows) | Public research dataset |
| **YOLOv8n pretrained weights** | Person detection from webcam | AGPL-3.0 (Ultralytics) |
| **Real-world simulated device inventory** | 90 modeled devices across 30 rooms, using device categories and embodied-carbon ranges grounded in real-world equipment and LCA references | Generated at runtime |
| **Real-world simulated room schedule** | ~20 modeled teaching sessions representing realistic campus operating patterns for the allocation solver | Generated at runtime |

**Grid emission factor:** 0.71 kg CO₂ / kWh (CEA India baseline).

---

## Models

### Energy forecast (LightGBM)
- Features: `hour`, `dow`, `lag1`, `lag24`, `roll24`
- Hyperparameter search: 15 random trials per building
- Train/test split: 90/10 time-based holdout
- Results: MAPE 0.97%–30.98% per building (B2 best, B4 worst — high variance on small loads)

### Water forecast (LightGBM)
- Same feature set, resampled hourly
- Caveats surfaced inline: B1 has a dead meter (0% coverage), B4/B5 have significant gaps (<80% coverage)
- Results: MAPE 9.4%–42.6% on buildings with usable data

### Carbon forecast (LightGBM)
- Trained on 585 days of 5-building aggregate
- Features: `doy`, `dow`, `is_weekend`, `lag1`, `lag7`, `roll7`
- Results: MAE 349.4 kg/day, MAPE 2.96%

### Anomaly detection (rule-based + YOLO)
- Rule: empty room with AC state=1 → flag for recommendation
- Detection: YOLOv8n counts people (class 0) at 640px, conf≥0.5
- F1 test: self-consistency check against the simulator's own occupancy log — *not* external ground truth

### Allocation (OR-Tools CP-SAT)
- Decision variables: `x[session, room] ∈ {0,1}`
- Hard: exactly-one-room per session, capacity ≥ attendance, matching room type, no overlap in any hour
- Soft: weighted objective across waste, energy, relocations, peak load, dept affinity
- Time limit: 8 seconds
- Result: typically OPTIMAL or FEASIBLE in <100 ms

---

## Project structure

```
campus-ai-final-version.ipynb     # the whole project, one notebook
├── Cell 1  — installs, imports, path setup
├── Cell 2  — BDG2 loader (5 education buildings)
├── Cell 3  — LightGBM hyperparameter search (energy + water)
├── Cell 4  — YOLOv8n download & load
├── Cell 5  — SQLite schema + building/dept/room seeding + 90-device inventory
├── Cell 6  — simulator state, timetable, tick loop, verifier
├── Cell 7  — e-waste scan & reuse planner
├── Cell 8  — carbon model training
├── Cell 9  — all UI helper functions
├── Cell 10 — Gradio UI (this is the last cell you run)
└── Cell 11 — (empty)

/kaggle/working/
├── campus.db                     # SQLite database (WAL mode)
├── data/
│   ├── elec_5b.csv               # 5-building hourly electricity
│   └── water_5b.csv              # 5-building hourly water
├── models/
│   ├── forecast_B1.pkl ... B5.pkl
│   ├── forecast_water_B1.pkl ... B5.pkl
│   ├── carbon_forecast.pkl
│   ├── metrics.json
│   ├── water_metrics.json
│   ├── carbon_stats.json
│   └── best.pt                   # YOLOv8n weights
└── report_HHMMSS.md / .json      # generated session reports
```

---

## Quick start

### Option 1 — Run on Kaggle
1. Upload the notebook to Kaggle
2. Add the BDG2 dataset: `claytonmiller/buildingdatagenomeproject2`
3. Run cells in order — the last cell launches Gradio with a public share URL
4. Click the share link → the UI opens in your browser

### Option 2 — Run locally
```bash
# 1. Clone / download the notebook
# 2. Install deps:
pip install lightgbm pyarrow gradio ultralytics requests plotly ortools
pip install torch torchvision  # for YOLO (CPU or CUDA)

# 3. Edit BASE in cell 1 to point at your local BDG2 CSV folder
# 4. Convert notebook → script:
jupyter nbconvert --to script campus-ai-final-version.ipynb
# 5. Run:
python campus-ai-final-version.py
```
The UI opens at `http://127.0.0.1:7860`.

---

## Configuration

| Setting | Location | Default | Purpose |
|---|---|---|---|
| Tick interval | UI slider | 2.0 s | Simulator speed |
| Baseline buildings | Cell 2 | 5 education buildings | Which BDG2 buildings to load |
| Hyperparameter trials | Cell 3 | `N_TRIALS = 15` | LightGBM search budget |
| Grid emission factor | Cell 8 / report | 0.71 kg/kWh | CEA India baseline |
| CP-SAT time limit | Cell 9 `run_optimizer` | 8 s | Solver budget |
| OFF-effect perturbation | Cell 6 `tick()` | ±15% | How noisy the counterfactual delta is |
| Telegram alerts | Env: `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID` | unset | Optional push notifications |
| LLM report enrichment | Env: `GEMINI_API_KEY` | unset | Optional exec-summary rewrite |

Secrets are read via `kaggle_secrets.UserSecretsClient` on Kaggle, or from environment variables otherwise. All secret-dependent features degrade gracefully offline.

---

## Deployment

### Kaggle (default)
`demo.launch(share=True)` produces a public Gradio URL valid for 1 week.

### Hugging Face Spaces
```bash
pip install gradio
gradio deploy
```
Then add the BDG2 dataset as a Space asset and adjust `BASE` in cell 1.

### Docker (self-hosted)
```dockerfile
FROM python:3.11-slim
RUN apt-get update && apt-get install -y libgl1 libglib2.0-0 ffmpeg
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 7860
CMD ["python", "campus-ai-final-version.py"]
```

---

## Data integrity & disclosure

This hackathon prototype intentionally distinguishes three data categories:

1. **Observed real-world data** — historical electricity and water measurements from the Building Data Genome Project 2 (BDG2).
2. **Real-world simulated data** — modeled campus devices, room schedules, occupancy patterns, and operational events based on realistic campus scenarios.
3. **Assumption-based outputs** — items such as e-waste recovery rates and embodied-carbon estimates that depend on published factors or configured assumptions.

The system does not present simulated or assumption-based values as measured sensor readings. This distinction is maintained in the UI, verification ledger, and generated reports.

## Limitations & honest disclosures

This is a **twin**, not a deployed system. Three things would need to change before it could be called a real-world verified deployment:

1. **One-room camera only.** Only one room's occupancy comes from a live webcam. Every other room's occupancy is a real-world simulated timetable, used purely to demonstrate the system at campus scale. Adding more cameras is a straight swap — the DB schema doesn't change.

2. **E-waste CO₂ is an assumption-based estimate.** The 70% / 65% / 40% / 15% recovery percentages are drawn from published LCA literature, not measured on our specific devices. This is why the UI and every report keep it **numerically separate** from measured energy CO₂ — they are never summed into a single "total impact" figure.

3. **F1 is a self-consistency check.** The adversarial test scores the anomaly detector against the simulator's own occupancy log. This is honest as a sanity check, but it is *not* validation against independent external ground truth. The UI labels it as such.

**What would make it a real deployment:** swapping the historical BDG2 baseline for live smart meters. Every other line of code — the ghost twin, the verifier, the ledger, the reports — stays identical. That's the entire point of the architecture.

---

## Roadmap

- [ ] Multi-camera occupancy — support N webcams mapped to N rooms via config
- [ ] Real MQTT ingestion — subscribe to smart-meter topics instead of CSV replay
- [ ] Webhook-based alerting — Slack, MS Teams, PagerDuty in addition to Telegram
- [ ] Anomaly detector upgrade — replace the rule with IsolationForest or a small transformer
- [ ] Persistent experiment tracking — log every verification to MLflow across sessions
- [ ] Multi-campus support — parameterize buildings, departments, schedules per site
- [ ] SSO / auth — protect approve/reject behind role-based access
- [ ] EnergyPlus co-simulation — replace historical replay with a physics simulator for what-if scenarios

---

## License

Provided as-is for research and demonstration purposes. See individual dependencies for their licenses:

- Ultralytics YOLOv8 — AGPL-3.0
- LightGBM — MIT
- Gradio — Apache-2.0
- OR-Tools — Apache-2.0
- BDG2 dataset — see original publication for terms
- Plotly — MIT

---

## Acknowledgements

- **Building Data Genome Project 2** (Miller et al., 2020) — for the open smart-meter dataset that makes counterfactual verification demonstrable without hardware
- **Ultralytics** — for YOLOv8n and the pretrained weights
- **Google OR-Tools** — for the CP-SAT solver
- **Gradio** — for a UI framework that handles streaming webcam input, live plots, and file downloads without ceremony

---

*Campus Twin AI · counterfactual verification · no IoT hardware*  
*LightGBM forecasting · CP-SAT allocation · e-waste reuse*


