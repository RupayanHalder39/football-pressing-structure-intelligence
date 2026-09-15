# Football Pressing Structure Intelligence

A football tracking-data research prototype for understanding **how a team presses**, how pressure develops over time, and how defender positioning can be evaluated through bounded counterfactual repositioning.

## Coaching Question

> **How should we press better?**

The system is designed to help coaches move beyond simply detecting whether a press happened. It analyses the structure of the press, the pressure around the ball carrier, the geometry of nearby defenders, escape lanes, and how a different defender position might improve the pressing structure.

---

## Why This Matters

Coaches often describe pressing using concepts such as:

- compactness
- pressing angle
- numerical support
- cover shadow
- nearest-presser distance
- escape lanes
- pressure at the ball carrier

These concepts are tactically meaningful, but they are difficult to compare consistently across moments.

This project turns player-tracking data into measurable pressing features and coach-facing visualisations.

---

## What the System Does

```text
Broadcast Match Video
        ↓
Player + Ball Tracking
        ↓
Spatial / Kinematic Features
        ↓
Press Episode Detection
        ↓
Continuous Pressure Field
        ↓
Pressing Structure Features
        ↓
Outcome / Pressure Analysis
        ↓
Bounded Counterfactual Defender Repositioning
        ↓
Coach-Facing Tactical Visualisation
```

---

## Core Capabilities

### 1. Press Episode Detection

A finite-state pressing model identifies pressing situations and separates them from neutral periods.

### 2. Continuous Pressure Field

The system estimates pressure around the ball carrier using defender distance and movement toward the target.

### 3. Pressing Structure Features

Examples include:

- nearest presser distance
- second-presser distance
- closing speed
- number of pressing participants
- local numerical superiority
- compactness
- convex-hull structure
- passing-lane pressure
- open / low-pressure escape lanes
- pressure at the carrier
- spatial pressure coverage

### 4. Counterfactual Repositioning

For selected pressing situations, the system tests small alternative defender positions and re-evaluates the pressing structure.

The goal is not to claim a globally optimal solution, but to answer:

> **Could this defender have stood somewhere slightly different to create a stronger pressing structure?**

---

## Coach-Facing Outputs

The project includes tactical visualisations showing:

- pressure dominance over time
- pressure intensity
- pressing episodes
- pressing outcomes
- tactical pitch geometry
- escape channels
- candidate defender repositioning

A coach can use the system to inspect:

- where the press becomes weak
- which player is late to support
- where an escape lane appears
- whether the team is compact enough
- how a small positional adjustment changes the pressure structure

---

## Example Tactical Interpretation

A team may appear to be pressing aggressively, but the ball carrier can still progress because the supporting presser is too far away.

The system can identify that:

- the nearest defender is applying pressure
- the second defender is not supporting effectively
- an escape lane remains available
- a small positional adjustment can increase modeled pressure and reduce the available escape structure

This turns:

> “The press looked weak here.”

into:

> **“The first presser arrived, but the supporting geometry left a usable escape route.”**

---

## Research Outputs

The repository includes:

- pressing-event detection
- pressure-field modelling
- pressing-structure features
- coach-facing pressure signals
- counterfactual repositioning
- evaluation utilities
- generated graphs and tactical previews
- a research-ready pressing feature dataset

---

## Repository Structure

```text
analytics/
├── pressing.py
├── pressing_v3.py
├── pressing_v4.py
├── pressure_field.py
├── pressing_features.py
├── pressing_structure_value.py
├── counterfactual_pressing.py
├── coach_pressure_signal.py
└── state_machine.py

dashboard/
├── live_graphs.py
├── dashboard_style.py
├── dashboard_style_v2.py
└── render_pressing_dashboard*.py

evaluation/
├── evaluate_press_detection.py
├── evaluate_participants.py
└── evaluate_outcomes.py

models/
├── baselines.py
└── short_horizon_outcome.py

docs/
├── METHODOLOGY.md
├── PRESSING_STRUCTURE_VALUE.md
├── PRESS_LABELING_PROTOCOL.md
├── FEATURE_SCHEMA.md
├── DATA_LEAKAGE_PROTECTION.md
└── EXPERIMENT_PLAN.md

outputs/
├── pressing_events.csv
├── pressing_features.parquet
├── research_dataset/
├── tactical graphs/
└── dashboard previews/

tools/
└── press_labeling_tool/
```

---

## Tech Stack

- Python
- NumPy
- Pandas
- Matplotlib
- OpenCV
- player / ball tracking
- spatial-temporal modelling
- geometric pressure modelling
- bounded counterfactual search

---

## Research Principles

This project follows several important constraints:

- causal analysis only
- no future-frame leakage in frame-level signals
- bounded local counterfactual search
- explicit uncertainty where evidence is weak
- no claim of globally optimal positioning
- no claim of validated causal match-outcome improvement

---

## Limitations

Current limitations include:

- short proof-of-concept match sample
- limited labelled pressing outcomes
- broadcast-video tracking noise
- heuristic tactical proxies
- counterfactual search restricted to small local positional changes

The system should therefore be interpreted as **coach-facing decision support**, not tactical ground truth.

---

## Future Work

Potential extensions include:

- multi-match validation
- expert coach annotation
- stronger pressing-outcome labels
- pressing archetype discovery
- opponent-specific pressing recommendations
- richer event-data integration
- learning which pressing structures work best against different build-up shapes

---

## Research Context

This project is part of my broader Ph.D. research on:

> **AI-Driven Tactical Intelligence in Football**

The broader goal is to convert football video and tracking data into explainable tactical recommendations that coaches and analysts can inspect and use.

---

## Author

**Rupayan Halder**  
Ph.D. Researcher — AI-Driven Tactical Intelligence in Football  
Jadavpur University  

Football Analytics Research Collaborator  

GitHub: [RupayanHalder39](https://github.com/RupayanHalder39)
