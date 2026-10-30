# STRATOS 🏎️

**F1 race strategy simulator.** Feed it a race situation, and it simulates thousands of possible futures to tell you which pit strategy is likely to finish fastest.

> Status: working on historical races (replay mode). Live-race mode is built but needs OpenF1 credentials.

---

## What it does

- **Learns from real data**: pace, tyre wear and compound differences are calibrated from historical races instead of hand-picked numbers.
- **Simulates the race lap by lap**: tyre degradation, fuel burn, pit-stop loss and traffic.
- **Runs Monte Carlo**: randomizes traffic, pit-stop time and driver pace to give a **P10–P90 range** of outcomes, not just one number.
- **Recommends a strategy**: ranks options by your goal (e.g. minimize total race time), respects rules (mandatory compounds, pit windows), and labels its **confidence** as HIGH / MEDIUM / LOW.
- **Replays past races with no cheating**: the replay only shows data available at that moment, so there is no hindsight bias.
- **Race Engineer Console**: a React + Plotly dashboard to step through a race and see decisions.

---

## How it works

```text
OpenF1 API → MongoDB → Feature Engine → Calibrated Model
          → Simulation + Monte Carlo → Decision Engine
          → FastAPI (REST/WebSocket) → React Console
```

| Stage | What happens |
|---|---|
| **Ingestion** | Pulls race data from OpenF1 and stores raw events in MongoDB |
| **Calibration** | Fits pace offsets, compound deltas and tyre degradation using robust regression |
| **Simulation** | Steps through laps using a state machine (`CanonicalRaceState`) |
| **Monte Carlo** | Samples uncertainty to build the P10–P90 range |
| **Decision** | Scores candidate strategies, applies constraints, assigns confidence |
| **Replay** | `ReplayStreamAdapter` emits past races in order, with zero look-ahead |

---

## Quick start

**You need:** Python 3.12+, Node.js 18+, Docker (or MongoDB on port 27017).

**1. Start the database**
```bash
docker run -d -p 27017:27017 --name stratos-mongo mongo:latest
```
Then load historical race data into a database named `stratos`.

**2. Start the backend**
```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
uvicorn backend.app:app --host 0.0.0.0 --port 8000
```

**3. Start the frontend**
```bash
cd frontend
npm install
npm run dev
```

Open **http://localhost:5173**.

### Try the demo
The default demo is **Bahrain 2023**. Open the Race Console and step through the timeline. Future data stays hidden until the cursor passes each decision point.

---

## Validation results

Tested on held-out races: **Bahrain 2023, Spa 2023, Austin 2023** (Phase 5C).

| Metric | Result |
|---|---|
| Mean absolute error | 68.89 s |
| Median absolute error | 12.05 s |
| P10–P90 coverage | 30.6% |

**How to read this:**
- The calibrated model **matches the old hardcoded baseline's accuracy** while replacing hand-set parameters with learned ones.
- The gap between mean and median error means a few races/situations are predicted badly.
- A well-calibrated P10–P90 range should cover about 80% of outcomes. At 30.6%, the ranges are currently **too narrow**, so treat them as optimistic.
- These are retrospective results on three races and do not guarantee real-world performance.

---

## Tech stack

| Layer | Tools |
|---|---|
| Backend | Python 3.12, FastAPI, Pandas, Pytest |
| Database | MongoDB |
| Frontend | React, TypeScript, Vite, Plotly.js |

## Project structure

```text
analytics/   Simulation, calibration and replay logic
backend/     FastAPI service (REST + WebSocket)
database/    MongoDB connection handling
frontend/    React UI, Race Console, charts
tests/       Pytest suite
```

## Testing

```bash
pytest tests/
```

---

## Known limitations

- **Fuel burn is fixed** at `0.0600 s/lap`, since open timing data can't reliably estimate it.
- **Traffic chart** shows only adjacent cars, not the full grid gap matrix.
- **Live mode** (`LIVE_ADAPTER_IMPLEMENTED`) works in the test lab, but live race sessions need authenticated OpenF1 credentials (`CREDENTIALS_NOT_AVAILABLE`).
- **Uncertainty ranges** are currently too narrow (see coverage above).

## Roadmap

- [ ] Per-circuit pit-loss calibration
- [ ] Rust-based Monte Carlo kernel (~10x more simulations)
- [ ] Wider, better-calibrated P10–P90 ranges