# Predictive Health Twin: Build Instructions for an AI Coding Tool

Use this file with GitHub Copilot (agent mode), Claude Code, Cursor or any similar tool.
Put it in the repo root and tell the tool: **"Read instructions.md. Do Phase 0 only, then stop and report."** Then continue one phase at a time.

---

## 1. Project brief

**Name:** Predictive Health Twin
**Context:** SDV Hackathon @ NE-PS, Challenge 1 "Vehicle Digital Twin & Predictive Diagnostics".
**Goal:** A working proof of concept (PoC) that shows an EV battery and drive thermal system with:

1. A **digital twin** that estimates a hidden temperature (virtual sensor).
2. **Predictive diagnostics** (anomaly alerts, health score, days to service).
3. A **cloud backend** that receives vehicle telemetry.
4. A **dashboard** for the fleet and for a single vehicle.
5. An **OTA fix** pushed to the vehicle, after which health visibly recovers.
6. A light **continuous learning** step (fleet retrains the model).

**SDV enablers demonstrated:** cloud connectivity, fleet data, OTA update, continuous learning, personalization per vehicle.

**Customer value:** the driver and the OEM get a warning weeks ahead of a thermal failure, and a software update mitigates it before a breakdown.

**Important:** All vehicle data is **simulated**. No real vehicle is needed. State this clearly in the README and the demo.

---

## 2. Working agreements (the AI tool must follow these)

1. **No manual coding by the human.** The AI writes, runs and fixes all code and tests.
2. **One phase at a time.** Finish a phase, run its tests, report results, then **stop and wait** for "continue".
3. **Keep it simple.** Prefer clear, readable code over clever code. The team must be able to explain every file to judges.
4. **Comment the why**, not the what. Add a short docstring to every module and function.
5. **Every phase ends with:** (a) tests passing, (b) a short summary of what was built, (c) the exact command to run or see it, (d) known limitations.
6. **Do not add features that are not in this file** without asking.
7. **If something is ambiguous,** choose the simplest reasonable option, state the assumption in one line, and continue.
8. **Reproducibility:** use fixed random seeds. Same seed must give the same data.
9. **Never commit secrets.** There are none in this project; keep it that way.
10. **If a test fails,** fix the code (or the test if it is wrong, explaining why) before moving on. Never skip or delete failing tests.

---

## 3. Tech stack and constraints

| Area | Choice |
|---|---|
| Language | Python 3.11 |
| Backend | FastAPI + Uvicorn |
| Storage | SQLite (via SQLAlchemy or plain `sqlite3`) |
| ML | scikit-learn, NumPy, pandas |
| Dashboard | Streamlit + Plotly |
| Tests | pytest |
| HTTP client | requests (simulator and dashboard to API) |
| Run | One command: `./run.sh` (and `run.bat` for Windows if easy) |

Must run **locally, offline**, on a normal laptop, in under a few minutes of setup.

---

## 4. Target repository structure

```
predictive-health-twin/
  README.md
  instructions.md
  requirements.txt
  run.sh
  config.yaml                 # all tunable numbers in one place
  docs/
    architecture.md
    demo_script.md
  src/
    common/
      config.py               # loads config.yaml
      schemas.py              # Pydantic models shared by all parts
    simulator/
      vehicle_model.py        # "true" physics incl. faults
      driving_cycle.py        # speed/power profiles
      fleet.py                # runs 20 vehicles, writes data / posts to API
    twin/
      thermal_model.py        # RC network (physics)
      residual_model.py       # ML correction
      train.py                # trains and saves model
    diagnostics/
      anomaly.py
      health.py               # score + days to service
    backend/
      main.py                 # FastAPI app
      db.py
      ota.py
    dashboard/
      app.py                  # Streamlit
  models/                     # saved model files (versioned)
  data/                       # generated data, sqlite db
  tests/
    test_simulator.py
    test_twin.py
    test_diagnostics.py
    test_api.py
    test_ota.py
```

---

## 5. Fleet and fault definition (single source of truth)

- **Fleet:** 20 vehicles, ids `EV-01` to `EV-20`.
- **Duration:** 30 simulated days.
- **Time resolution:** 10 seconds while driving or charging. Do **not** generate 1 Hz for all 30 days (too large). Simulate **2 trips per day** (about 40 minutes each) and **1 charging session per day**, plus one daily summary row per vehicle.
- **Signals sent by each vehicle (the "available" signals):**
  `timestamp, vehicle_id, soc_pct, pack_current_a, pack_voltage_v, coolant_temp_c, motor_temp_c, ambient_temp_c, speed_kph, pump_duty_pct`
- **Hidden signal (not sent, used only to validate the twin):** `cell_temp_true_c`.
- **Healthy vehicles:** 17 (small per-vehicle differences in battery capacity, resistance, driving style).
- **Faulty vehicles:** 3 (`EV-05`, `EV-11`, `EV-17`). Fault = **coolant pump wear**.
  Pump efficiency falls from 100% to about 60% over the period, starting between day 8 and day 15 (different start per vehicle), with a gentle accelerating curve. Cell and coolant temperatures therefore drift upward for the same driving load.
- Add realistic **sensor noise** and a few **missing samples** (about 0.5%).
- Hot ambient days should occur so the fault becomes visible.

---

## 6. Phases

### Phase 0: Project setup

**Goal:** Empty but runnable skeleton.

**Tasks**
1. Create the folder structure from section 4 with placeholder files.
2. Create `requirements.txt`, `config.yaml` (fleet size, days, seeds, thresholds), and `src/common/config.py`.
3. Create `src/common/schemas.py` with Pydantic models: `TelemetryRecord`, `VehicleStatus`, `FleetStatus`, `OtaRequest`, `OtaResult`.
4. Write a `README.md` stub and a `run.sh` that prints "not yet implemented".
5. Add a trivial passing pytest so the test setup works.

**Done when:** `pip install -r requirements.txt` works, `pytest` passes, and the structure matches.

**Stop and report.**

---

### Phase 1: Vehicle simulator (the "true" world)

**Goal:** Generate realistic telemetry for 20 vehicles including 3 degrading ones.

**Tasks**
1. `driving_cycle.py`: generate speed and power-demand profiles (city, highway, mixed) with randomness per vehicle and per day.
2. `vehicle_model.py`: simulate the true physics. Use this simple lumped model (explain it in comments):
   - Battery heat: `Q_batt = I^2 * R_int(SoC, T)`
   - Motor/inverter heat: `Q_motor = loss_fraction * P_mech`
   - Cell node: `C_cell * dT_cell/dt = Q_batt - (T_cell - T_coolant) / R_cell_cool`
   - Coolant node: `C_cool * dT_cool/dt = (T_cell - T_cool)/R_cell_cool + Q_motor - pump_efficiency * k_pump * pump_duty * (T_cool - T_amb)`
   - Pump duty is set by a simple controller: duty rises with coolant temperature (parameters held in a "pump curve").
   - Faulty vehicles reduce `pump_efficiency` over time.
3. `fleet.py`: run all vehicles for 30 days, add noise and missing samples, and write to `data/telemetry.csv` (or parquet). Also write `data/ground_truth.csv` with the hidden cell temperature and pump efficiency.
4. Make the simulator **callable in two modes**: batch (write files) and stream (post day by day to the API; used in Phase 4 and 6).
5. Add a CLI: `python -m src.simulator.fleet --days 30 --seed 42`.

**Tests**
- Same seed gives identical output.
- Row counts and column names are correct.
- Temperatures stay in a plausible range (for example cells 15 to 65 C).
- Faulty vehicles show a clearly higher late-period coolant temperature than healthy vehicles under comparable load.

**Done when:** a plot or printed summary shows drift in EV-05, EV-11 and EV-17 and none in the others.

**Stop and report** (include one chart image saved to `docs/`).

---

### Phase 2: Digital twin (virtual sensor)

**Goal:** Estimate hidden cell temperature using only the signals the vehicle actually sends.

**Tasks**
1. `thermal_model.py`: implement the **same RC structure** but with **nominal healthy parameters** and a **healthy pump assumption**. Input: available signals. Output per step: predicted coolant temperature and predicted cell temperature. This is the "expected healthy behavior".
2. `residual_model.py`: a scikit-learn model (for example gradient boosting or ridge regression) that learns the **error of the physics model** versus ground truth cell temperature, using features such as SoC, current, speed, ambient, coolant, motor, and the physics prediction.
3. `train.py`: train on **healthy vehicles only** (EV-05, EV-11, EV-17 excluded), validate on a held-out period, and save the model to `models/residual_v1.joblib` with a small `models/residual_v1.json` (metrics, training date, vehicles used).
4. Expose a function: `estimate_cell_temp(df) -> series` and `expected_coolant_temp(df) -> series`.

**Tests / metrics (print and save to `docs/twin_metrics.md`)**
- Cell temperature estimate error (MAE and max error) on healthy vehicles: target **MAE under 2 C**.
- On a faulty vehicle the estimate should still track the true cell temperature reasonably (the twin estimates the real temperature), while `measured coolant - expected coolant` grows over time (this is the diagnostic signal).

**Done when:** a chart shows estimated vs true cell temperature closely overlapping, and a second chart shows the residual growing for a faulty vehicle.

**Stop and report.**

---

### Phase 3: Predictive diagnostics

**Goal:** Turn residuals into alerts, a health score and a days-to-service estimate.

**Tasks**
1. `anomaly.py`: compute a daily residual per vehicle (`measured coolant - expected coolant`, plus estimated cell temperature excess versus fleet norm). Flag an anomaly when the rolling residual exceeds a threshold for N consecutive days. Thresholds in `config.yaml`. Keep it explainable (no black box here).
2. `health.py`:
   - **Health score 0 to 100:** 100 for no residual, falling as the rolling residual grows. Document the formula.
   - **Status:** Green (80 to 100), Amber (50 to 79), Red (under 50).
   - **Days to service:** fit a simple trend (linear or exponential) to recent health scores and estimate days until the score crosses a service threshold (for example 40). Return a **low/high range**, not a single number. If there is no downward trend, return "no service needed".
   - **Explanation text:** generate a plain-language message, for example: "Coolant temperature is 6.2 C above expected for the same load. This is consistent with reduced pump efficiency. Estimated service in 18 to 26 days."
3. Add a **fleet comparison** feature: a vehicle's residual is also compared with the fleet median so that hot weather does not trigger false alarms.

**Tests**
- All 3 faulty vehicles are flagged Amber or Red before the end of the period; ideally flagged at least 7 days before they would reach Red.
- **No more than 1** healthy vehicle is flagged at any time (target 0).
- Days-to-service range is ordered (low less than or equal to high) and positive.

**Done when:** a table shows per-vehicle score, status and days to service for the final day, plus the day each faulty vehicle was first flagged.

**Stop and report.**

---

### Phase 4: Cloud backend (FastAPI + SQLite)

**Goal:** A backend that ingests telemetry and serves status, so the PoC is truly "connected".

**Tasks**
1. SQLite tables: `telemetry`, `vehicle_state` (latest score, status, days range, last update), `vehicle_config` (pump curve parameters and power limit, version), `ota_log`.
2. Endpoints:
   - `POST /telemetry` accepts a batch of `TelemetryRecord`. Stores data, then updates the vehicle's twin estimate and health.
   - `GET /fleet` returns all vehicles with score, status, days to service, sorted by risk (worst first).
   - `GET /vehicle/{id}` returns detail: latest status, explanation text, history series (measured, twin estimate, expected, residual, health) for charts.
   - `GET /vehicle/{id}/config` returns the current control parameters (the vehicle "downloads" these).
   - `POST /ota/{id}` (built fully in Phase 6; return a stub for now).
   - `GET /health` simple liveness check.
3. Use the saved model from Phase 2 and the logic from Phase 3 (import them; do not duplicate code).
4. Update the simulator **stream mode** to post data day by day, so a live-feeling replay is possible (option: `--speed` factor).
5. Return clear errors (422 for bad data, 404 for unknown vehicle).

**Tests (use FastAPI TestClient)**
- Post a day of data, then `GET /fleet` returns all vehicles.
- Unknown vehicle returns 404.
- Posting data for a faulty vehicle over many days lowers its health score.

**Done when:** `uvicorn src.backend.main:app` runs, the simulator streams data into it, and `GET /fleet` shows 3 vehicles trending to Amber or Red.

**Stop and report** (include curl examples).

---

### Phase 5: Dashboard (Streamlit)

**Goal:** A clear, demo-ready UI. This is what the judges will see.

**Pages / sections**
1. **Fleet overview**
   - KPI cards: vehicles total, vehicles at risk, lowest health score, nearest days to service.
   - Table sorted by risk with color-coded status (use colors plus text labels, not color alone).
   - Click or select a vehicle to open its detail.
2. **Vehicle detail**
   - Chart 1: **twin estimate vs measured** (and expected healthy coolant temperature) over time.
   - Chart 2: **health score trend** with the forecast line and the service threshold.
   - Alert card with the plain-language explanation and the days-to-service range.
   - Config panel showing the current pump curve and software version.
   - **OTA button** (wired in Phase 6).
3. **Fleet insight** (small): residual of each vehicle vs fleet median, showing the 3 outliers clearly.
4. **Replay control:** a "Start live replay" button or slider that streams simulated days into the backend so the dashboard updates while the team presents.

**Rules**
- The dashboard only talks to the **API** (never reads the database or files directly), to prove the cloud architecture.
- Keep layout clean: large numbers, few colors, readable fonts, titles that say what the chart means.
- Auto-refresh every few seconds during replay.

**Done when:** `streamlit run src/dashboard/app.py` shows the fleet, a faulty vehicle detail with diverging lines, and an alert.

**Stop and report** (include screenshots saved to `docs/screens/`).

---

### Phase 6: OTA update demo

**Goal:** Close the loop: detect, push a fix over the air, and show recovery.

**Tasks**
1. Define an OTA package (a small JSON): `version`, `description`, `pump_curve` (more aggressive duty at a given temperature) and optional `power_limit_pct` (a mild derate, for example 90%).
2. `POST /ota/{id}`:
   - Validates the vehicle, stores the new config version in `vehicle_config`, and writes to `ota_log` (who, when, from version, to version).
   - Returns `OtaResult` with before and after config.
3. The simulator (in stream mode) **polls `GET /vehicle/{id}/config`** and applies new parameters in its pump controller. After the update, the faulty vehicle's temperatures drop (more pump duty compensates for the worn pump; the derate lowers heat).
4. Dashboard: OTA button on the vehicle detail page. After clicking, show a **before/after panel** (health score, temperature residual, days to service) and mark the OTA event as a vertical line on the charts.
5. Make sure the **recovery is visible within the replay** (health score rises or stops falling within a few simulated days).

**Tests**
- OTA changes the stored config version and writes a log entry.
- After applying OTA in simulation, the average residual for that vehicle over the next days is lower than before.
- OTA on a healthy vehicle either is refused politely or has no harmful effect (choose one and document it).

**Done when:** clicking the OTA button visibly improves the faulty vehicle on the dashboard.

**Stop and report.**

---

### Phase 7: Continuous learning (optional but valuable)

**Goal:** Show that the fleet makes the twin smarter over time.

**Tasks**
1. `train.py` gets a `--retrain` mode that uses all fleet data from **healthy or post-OTA healthy periods** to train a new model `residual_v2`.
2. Save model versions in `models/` with metrics. Compare v1 vs v2 on the same validation set.
3. Backend endpoint `POST /model/retrain` and `GET /model/info`. The dashboard shows the active model version and its error (MAE).
4. Add **personalization**: a small per-vehicle offset or baseline so the twin adapts to each vehicle's habits. Show that error per vehicle decreases.
5. Keep a safe rollback: if v2 is worse than v1, keep v1 and say so.

**Done when:** the dashboard shows "Model v2, MAE improved from X to Y".

**Stop and report.**

---

### Phase 8: Documentation, packaging and demo readiness

**Tasks**
1. `README.md`: what it is, the problem, architecture summary, how to run (3 commands max), how to run tests, honest limitations (simulated data, simplified physics), and the validation plan with real vehicle logs.
2. `docs/architecture.md`: components, data flow, API list, data model, and a Mermaid diagram:
   `Vehicle ECU -> Telemetry -> Cloud twin -> Diagnostics -> Dashboard -> OTA -> Vehicle ECU`.
3. `docs/demo_script.md`: the 3-minute demo with exact clicks and talking points (see section 7).
4. `run.sh`: starts the backend, waits until `/health` responds, trains the model if missing, starts the dashboard. Provide a `--reset` option that clears the database for a fresh demo.
5. Add a **"demo mode" seed** that guarantees the same visible story every time (EV-11 turns Amber by the time of the demo, Red soon after).
6. Run the full test suite and a cold-start test from a clean clone. Fix anything that breaks.

**Done when:** a teammate can clone the repo, run one command, and see the working demo.

**Stop and report** (final summary with the test results).

---

## 7. Demo script (3 minutes)

1. **Problem (20 s):** warning lights come too late, and workshop visits are costly.
2. **Fleet view (30 s):** 20 vehicles streaming in; 3 turn amber or red.
3. **Twin view (60 s):** open one vehicle; show estimated vs measured diverging; read the alert and days-to-service range.
4. **OTA (40 s):** click the button; show the version change and health recovering.
5. **Value close (30 s):** fewer breakdowns, lower warranty cost, better customer experience, and a model that improves with fleet data. State clearly that data is simulated and how real logs would validate it.

---

## 8. Acceptance checklist (all must be true)

- [ ] `./run.sh` starts everything from a clean clone.
- [ ] 20 vehicles streamed; 3 faulty ones flagged early; at most 1 false alarm.
- [ ] Twin cell temperature error (MAE) under 2 C on healthy vehicles.
- [ ] Dashboard shows fleet, vehicle detail, twin vs measured, health trend, alert text.
- [ ] OTA button changes config, logs it, and visibly improves the vehicle.
- [ ] Model version and improvement shown (Phase 7).
- [ ] All tests pass; README and architecture doc complete.
- [ ] Limitations stated honestly (simulated data, simplified physics).

---

## 9. Reusable prompts (copy and paste)

**Start a phase**
> Read instructions.md. Implement Phase N only. Follow the working agreements in section 2. When finished, run the tests, show results, summarize in a few lines, give me the command to see it, and stop.

**Fix a problem**
> The following is wrong: [describe or paste error]. Find the root cause, fix it, add or update a test so it cannot return, rerun all tests, and summarize the change.

**Explain for judges**
> Explain what Phase N built in plain language for a non-expert judge, in 6 sentences, and list the 3 questions a judge is most likely to ask with short answers.

**Improve the demo**
> Review the dashboard for a 3-minute live demo. Suggest and implement small changes that make the story clearer (labels, colors, ordering), without adding new features.

---

## 10. Out of scope (do not build)

- Real vehicle connectivity, real CAN data, or real OTA signing and security.
- User login, multi-tenant features, or cloud deployment.
- Whole-vehicle modeling; stay with the battery and drive thermal subsystem.
- Any large deep-learning model; simple, explainable models only.
