# Submission: ready-to-list

## 1. Team

- **Team / solo name:** ready-to-list
- **Members:** Pranav (add teammates here)
- **Complexity level claimed:** L3

## 2. One-line summary

List only the hearings that will actually happen and move the case forward, pack them into the judge's day with a constraint solver, give every advocate a real time window, and set the next date from what the case needs, never a flat 60 days.

## 3. The approach

**In plain language.** Every evening the scheduler looks at the cases due and asks three questions. Is this case ready: has its summons or warrant come back, and did counsel confirm? How likely is the hearing to be substantive, given its hearing type and what happened last time? How many minutes will it take? Ready cases are packed into the next day's two sittings (10:30-12:30 and 13:30-17:00), best value per minute first. Bail goes first and 25% of each day is locked for 4+ year cases. Cases of the same type and the same advocate are grouped, so the judge switches context less and advocates make one trip. After each hearing, the next date comes from the next purpose's reference gap, or from the failure reason (a warrant not back: when it is due back).

**Inputs used (all six files):**

| File | How it is used |
|---|---|
| `roster_sample_100.csv` | The cases. Scaled to 3,000 with your own `scripts/generate_roster.py`. `last_hearing_summary` is read for signals: who was absent, what is awaited ("Await warrant", "not ready", "last chance") |
| `court_calendar.csv` | Working days for the simulation and for next dates |
| `hearing_type_reference.csv` | Minutes per hearing type, and the gap to the next hearing for each purpose |
| `substantiveness_by_hearing_type.csv` | The real probability each type moves the case forward: the truth model is calibrated to it |
| `hearing_failure_reasons.csv` | Why hearings fail, grouped into process (summons/warrant not back), absence, not ready, court, unclear. Each group is a different lever |
| `sample_causelist_2026-09-22.csv` | Used as the shape of today's cause list (baseline lists about 60 a day) |

**Core logic (`core/pucar_engine.py`):**

1. **Truth model.** A hearing of type T is substantive with the real probability p(T). If not, it fails for a reason drawn from the real reason mix for T. "Awaiting process" is a state that persists until the process comes back, not a coin flip. The last hearing's note raises the matching risk (x2.5), normalised so each type's average stays at the real rate. Check: under today's rules, with nothing tuned to match, the model gives 60 listed, 26 reached and 12.5 substantive a day, close to the case study's 60 / 20 / 10.
2. **Readiness levers:**
   - **Process tracking:** list a case only once its summons or warrant is back (status known 90% of the time).
   - **T-2 intent check:** half of the "not ready" failures surface before listing.
   - **Fixed slot and advocate clustering:** a third fewer absences.
   - **Reading the last hearing's note** when ranking.
3. **Optimiser.** CP-SAT knapsack per sitting. Value = priority x P(substantive) x minutes^0.75, so the ratio per minute favours likely, short hearings without starving long arguments and judgments. Capacity is 95% of the net minutes, with a same-day waitlist. Bail goes first; the ageing quota is locked.
4. **Next date:** after a substantive hearing, the reference gap for the next purpose. After a failure, the gap for its reason: process 21 days or when it is due back, absence 7, not ready 10.

**The full product** (`app.py`, Streamlit) adds a multi-day planner. Stage 1 picks the day for every case across three courts with CP-SAT or MILP. Stage 2 sets exact times with CP-SAT interval scheduling: no advocate is due in two courtrooms at once, with travel time between them, and changeovers are sequence-dependent. It also has a pre-filing defect check with dummy filings, a judge dashboard with an override impact meter, a court master screen with one-tap outcomes and next dates, a calendar, and model-accuracy pages. The page "On the organisers' data" runs everything in this submission.

**Key decisions:**
- **Calibrate to your rates.** The baseline must reproduce your substantive rates before any claim about improvement means anything.
- **Treat "awaiting process" as state, not luck.** It is the largest preventable failure: 67% of failed WARRANT hearings and 51% of failed ADMISSION hearings.
- **Score per minute, not per case.** A 30-minute evidence hearing and a 5-minute admission compete fairly.
- **Lock the ageing quota and bail-first.** Judges can configure everything else.

**Assumptions (explicit):**
- **Lever strengths** (process status accuracy 90%, intent check catches half of "not ready", fixed slots remove a third of absences) are assumptions set in `config/pucar.yaml`. The ablation below shows how much each one matters.
- **Minutes** are your estimates x lognormal noise (sigma 0.35). An adjournment costs 2 minutes.
- **Process return time:** a pending process comes back in 3 to 25 working days.
- **Court day:** 10:30-12:30 and 13:30-17:00, which is 330 minutes. We also report the 420-minute day from your README.
- **Initial due dates:** the roster's cases are spread evenly over the horizon.

## 4. Justify your complexity level (L3)

- **L1:** fixed rules: bail first, the ageing quota, the readiness gate, the priority formula (`config/pucar.yaml`).
- **L2:**
  - Distributions: real substantive rates and reason mixes per type, lognormal durations, process-return times.
  - A prediction step: P(substantive) per case from type plus last-note signals.
  - A constraint solver: CP-SAT.
  - Costs: changeovers, the adjournment call.
  - Judge overrides: in the app, the impact meter shows the change in utilisation, predictability and 5+ year cases before approval.
- **L3:**
  - Advocates and parties behave. They respond to fixed slots, bundling and confirmation by showing up more, and to short-notice waitlist calls by showing up less.
  - Their absences and unreadiness feed back into the next date and the next day's plan.
  - In the full app, 250 advocate agents of three types (diligent, busy, chronic adjourner) decide whether to confirm, prepare, file a cover sheet and appear. A busy advocate who confirmed and did not appear gets a costs warning and responds to it (`core/agents.py`, `core/simulate.py`).

## 5. Results

**A. The judge's full docket, 60 working days.** 3,000 cases (your generator, seed 42) from 1 Oct 2026, 330 court minutes a day, averaged over 5 seeds (`outputs/results.md`):

| Metric (case study) | Today's rules | Ready-to-List |
|---|---|---|
| Utilisation: court minutes used | 99.3% | 99.6% |
| Reach rate: scheduled cases the court gets to | 44% | 97% |
| Substantiveness: reached hearings that move the case | 47% | 82% |
| Backlog-age impact: 4+ year cases heard at least once | 29% | 49% |
| Predictability: days from first listing to the hearing that moved it | 24 | 3 |
| Next-date gap, days | 60 (flat) | 14 (purpose-based) |
| Substantive hearings a day | 12.5 | 14.6 (+17%) |
| Cases disposed in 60 days | 191 | 368 (+93%) |
| Wasted listings (trips for nothing) | 2,851 | 220 (-92%) |

With your 420-minute day (`outputs/capacity_420/`), today against Ready-to-List: 15.9 against 19.2 substantive hearings a day, 235 against 486 disposed.

**Which lever does what** (one switched off at a time, and the halves alone):

| Configuration | Substantive a day | Moves the case | Wasted listings | Disposed |
|---|---|---|---|---|
| Ready-to-List, everything on | 14.6 | 82% | 220 | 368 |
| Scheduling only: registry packing and next dates, no party input | 13.5 | 66% | 464 | 343 |
| Scheduling only + fixed slots and clustering | 13.9 | 73% | 360 | 358 |
| Readiness levers only (no optimiser) | 13.8 | 64% | 2,768 | 152 |
| Without reading the last hearing's note | 13.8 | 78% | 266 | 321 |
| Without the pre-filing check | 14.5 | 81% | 233 | 364 |

Scheduling alone, which needs only the registry's own data, delivers most of the gain.

**B. One judge for a year, the 100 real cases inside it, new complaints arriving** (`outputs/one_judge_year.md`, 250 working days, 3 seeds, about 2.5 new complaints a day). This is where pre-filing shows: 96 of your 100 cases went through Delay Condonation hearings (3.37 hearings per case, 29% substantive). When the registry computes limitation at e-filing and the condonation petition is heard with admission, new complaints stop stalling there.

| Over a year | Today's rules | Scheduling only | Scheduling + pre-filing | Ready-to-List (all) |
|---|---|---|---|---|
| Cases disposed | 524 | 639 | 639 | 864 |
| Of the 100 real cases | 16 | 21 | 21 | 29 |
| New complaints past cognizance within the year | 3% | 4% | 89% | 79% |
| Delay condonation listings | 455 | 175 | 85 | 85 |
| Substantive hearings a day | 12.6 | 14.1 | 14.1 | 15.0 |
| Wasted listings | 11,851 | 3,216 | 3,116 | 1,597 |

**Start here** (the app's first page): upload the docket file from this repo (`data/roster_sample_100.csv`, or the same columns as an Excel sheet). The app checks the columns and explains any problem in plain words. It then shows what is in the docket, reads each case's readiness from its last hearing note, plans the next week to year with time windows, and compares the results with today's rules. The pitch deck is `docs/presentation.html`: open it in a browser and use the arrow keys.

**Visualisation and workflow** (Streamlit app, page "Justice Sehgal's docket"):
- **The data:** every file and column, with the key findings (`docs/DATA_PROFILE.md`).
- **Registry intake:** the manual scrutiny of an NI Act s.138 complaint as structured e-filing fields. Statutory dates are computed (cheque validity, 30-day notice, 15 days to pay, one month from cause of action), with documents, summons details and jurisdiction (s.225 enquiry). Nothing is refused; the result says what to cure (`config/registry_ni138.yaml`, `core/registry.py`).
- **Lifecycle:** the stage graph derived from your data, with cases now at each stage, P(substantive), minutes and hearings per case.
- **Case file:** each of the 100 real cases, with hearings held per stage, the last note and the signals read from it, and its simulated next year under both approaches.
- **Plan by day, week, month or year:** the cause list with time windows; the week board by hearing type; the month's listed and moved hearings; the year's cumulative disposals and where pending cases stand.

**Against the default.** "Whatever is listed gets attempted, 60-day gap" lists 60 a day and reaches 44% of them. Ready-to-List lists about 18, reaches 97%, and hears more of them substantively.

## 6. Specs for integration

- **Input schema:** exactly your CSVs, unchanged. `core/pucar_engine.load(data_dir, roster=...)` reads them. The hearing-type names from your files are normalised to upper case with underscores.
- **Output:** `outputs/proposed_schedule.csv` with columns `date, block, expected_start, window, case_number, hearing_type, advocate_id, from_waitlist, p_substantive_planned, simulated_outcome`, plus `results.csv` and `daily.csv`. `simulated_outcome` exists only in simulation; drop it in production.
- **Scaling to every court and judge:** the unit is one judge's docket. `judge_docket()` builds it, `simulate()` plans it, and each court's rules (sitting blocks, hearing-type flow, priorities, registry checks) live in YAML (`config/pucar.yaml`, `config/registry_ni138.yaml`). Another bench is another docket and config, with no code change. Advocates shared across benches are handled by the multi-court planner in `core/optimize.py`, which never lists one advocate in two courtrooms at once.
- **Interfaces:** a Python module (`core/pucar_engine.py`), a CLI (`scripts/run_pucar.py`) and a Streamlit app (`app.py`). A DRISTI integration would call `simulate` or the day planner nightly and write the cause list plus a reason per item.
- **Dependencies:** Python 3.11+, pandas, numpy, OR-Tools (CP-SAT, SCIP), scikit-learn, PyYAML, Streamlit, Plotly, PyMuPDF, tabulate. No external services and no LLM calls.
- **Stubbed vs real:**
  - Real: the optimiser, next-date logic, calibration to your data, the ablation and the app.
  - Assumptions: lever strengths and process-return times, in config.
  - Stubbed: the LLM case summary (a cached text).
  - Simulated: process status. In production it would come from the summons/warrant tracking in DRISTI.
- **What integration would take:**
  1. Feed process-return status and advocate confirmations from DRISTI.
  2. Map DRISTI hearing purposes to the 14 types.
  3. Run the planner each evening and show the judge the draft list for approval.
  4. Write outcomes back so the models retrain.

## 7. How to run it

```bash
cd submissions/ready-to-list
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
# Score on your data (3,000 cases, 60 days, 330-minute day; add --capacity 420 for the README's day)
.venv/bin/python -m scripts.run_pucar --out outputs
# One judge for a year: the 100 real cases inside the docket, new complaints arriving, with and without pre-filing
.venv/bin/python -m scripts.run_one_judge --out outputs
# Profile every data file and column
.venv/bin/python -m scripts.data_profile
# The full app: start at 'Start here: plan a docket' and upload data/roster_sample_100.csv
.venv/bin/streamlit run app.py
```

## 8. What we'd build next

1. Fit lever strengths from real process-status and confirmation data instead of assuming them.
2. Replace keyword signals from the last hearing's note with a small classifier trained on the notes.
3. Learn minutes per hearing from court-master timestamps.
4. A two-way loop with advocates (WhatsApp confirmations) so the intent check is real, not simulated.

---
**Checklist before you open your PR:**
- [x] No real case numbers, party names, or advocate names appear anywhere in this submission. (Case numbers come from your generator; the demo filings are fictional.)
- [x] Everything lives under `submissions/ready-to-list/`.
- [x] This file is filled in, not left as a template.
- [x] Your code actually runs with the commands in section 7.
