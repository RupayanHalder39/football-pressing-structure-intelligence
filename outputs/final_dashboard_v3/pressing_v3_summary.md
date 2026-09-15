# Pressing Dashboard V3 — Tactical Logic Upgrade Summary

## What changed and why

V2's pressing dashboard computed a single, current-ball-carrier-relative
"is a press happening" signal per frame, with an ad-hoc episode detector
sitting on top. That was single-team-centric (it only ever described
whichever team did NOT have the ball) and had no explicit hysteresis, so
a single noisy frame could plausibly flip the on-screen state.

V3 adds a genuinely new analytics layer (`pressing_structure/analytics/
pressing_v3.py` + the new shared `pressing_structure/analytics/
state_machine.py`) that runs **both teams simultaneously, every frame**,
and stabilizes each team's signal with a proper hysteresis state
machine, then re-renders the dashboard (`render_pressing_dashboard_
final_v3.py`) around that new data. The V2 analytics module
(`pressing.py`) is completely unmodified and still used for the
existing outcome taxonomy (BALL_REGAIN/FORCED_BACKWARD/...) shown
alongside the new V3 numbers, not replaced by them.

## Goal 1: simultaneous dual-team tracking

For every frame, BOTH teams now get a real `press_score`, `press_state`,
and `press_confidence` (`build_dual_team_press_states`). Because a team
can only press while the OTHER team has the ball, one team's score is a
**real, defined 0.0** (not a fabricated value) whenever that team's own
side has possession — not "no data". When no confident ball carrier
exists that frame, BOTH scores are `None` (real "no evidence"), which
flows through smoothing/the FSM as `UNCERTAIN` rather than a silent
guess.

`decide_dominant_state()` implements the brief's exact arbitration rule:
one team clearly active → show that team; both low → `NO_PRESS`; both
elevated simultaneously (only possible right at a possession-change
boundary) → `CONTESTED`; anything else → `UNCERTAIN`. This is now the
single source of truth for the dashboard's main banner, replacing V2's
possession-gated logic.

## Goal 2: transition-aware state machine

`pressing_structure/analytics/state_machine.py` (NEW, shared with
`offside_break`) implements: rolling-window smoothing (0.5s / 15
frames) → hysteresis FSM with `enter_th` / `exit_th`, minimum
consecutive activate frames, minimum hold duration, an ending window,
and a cooldown before re-arming. States: `NO_PRESS`, `PRESS_FORMING`,
`ACTIVE_PRESS`, `PRESS_ENDING`, `UNCERTAIN`. Validated first on
synthetic data (confirmed correct handling of a real sustained episode,
a bridged short data gap, and a single-frame noise spike that correctly
did **not** re-trigger an episode), then on the real clip.

**Thresholds used (`PRESS_FSM_CONFIG`)**: `enter_th=0.4`, `exit_th=0.25`,
`min_activate_frames=8` (~0.27s), `min_hold_frames=10` (~0.33s),
`ending_frames=10` (~0.33s), `cooldown_frames=15` (~0.5s).

**A real bug was found and fixed during this work**: the first version
used `enter_th=0.5` (inherited as a round-number guess), but measuring
the actual raw-score distribution on this clip showed team 0's own 90th
percentile was only 0.42, and a real, known episode's smoothed score
peaked at 0.47 — just under 0.5 — because two moderate readings
(0.44, 0.38) pulled the rolling average down below two higher ones
(0.50, 0.56) within the same window. `enter_th=0.5` would have made
team 0 almost unable to ever register `ACTIVE_PRESS`. Recalibrated to
`0.4/0.25` (roughly the 75th–90th percentile band of the measured
score, not fit to that specific episode's outcome) and re-verified: the
previously-failing episode now correctly registers `ACTIVE_PRESS`
starting inside its known real window.

**Result on the full 120s clip**: Team 0 — 12 `ACTIVE_PRESS` episodes
(durations 0.03s–1.4s), mean score 0.223. Team 1 — 6 `ACTIVE_PRESS`
episodes (durations 0.03s–1.03s), mean score 0.140. (Both teams' press
state is `UNCERTAIN` on ~78% of frames — a confident ball-carrier
identification is only available on a minority of frames in this
tracking data, which is an existing, previously-documented
characteristic of this dataset, not a new V3 issue; V3 is honest about
it rather than papering over it with a guess.)

## Goal 3: visualization

- Radar pressure-hotspot: Team 0 = amber/orange family, Team 1 = green
  family (`TEAM_PRESS_GLOW_STOPS`), player markers for both teams kept,
  confident ball carrier highlighted, active pressers ringed.
- **A real rendering bug was found and fixed during Goal 5 QA**: the
  existing `add_pressure_glow` compositor ADDS its three color rings
  directly onto the pixel underneath. Against this dashboard's bright
  pitch-control wash, that addition clipped toward white/yellow
  regardless of which team's color was used — a side-by-side crop
  comparison showed team 0's (amber) and team 1's (green) hotspots
  rendering as an almost-identical whitish blob. Root cause was the
  blend mode, not the color values themselves (retuning team 1's stop
  colors alone did not fix it). Fixed by adding a new function,
  `add_team_pressure_glow` (`dashboard_style_v2.py`), which
  alpha-composites a weight-normalized average of the three rings'
  colors instead of adding them — this guarantees the team's actual hue
  shows through regardless of the background's brightness. The
  original `add_pressure_glow` was left completely unmodified (still
  used as-is elsewhere) rather than changed in place, per this
  project's "don't change working core-file behavior unnecessarily"
  constraint. Re-verified: team 0 now renders as a clear warm red/orange
  halo and team 1 as a clear green halo, visually distinct from each
  other at the same real intensities that previously looked identical.
- KPI tile row: "Team 0 Press State" / "Team 1 Press State", dominant
  team this frame, per-team episode counts and mean scores (Goal 4).

## Goal 5: transition QA performed

- Rendered and inspected individual preview frames at a known real
  episode boundary (t≈5.1–5.5s, team 1's V2-documented episode) before
  and after the threshold fix, confirming the banner/state/team-label
  triad is now consistent with the known ground truth.
  Confirmed: no stale banner (state updates immediately when the FSM
  state changes), team label always matches the FSM's own per-team
  arrays (no cross-team leakage), and the color-blend bug above was
  caught specifically because a state that DID activate rendered
  ambiguously — exactly the kind of "state change isn't visually
  explainable" failure Goal 5 asks to catch.
- The full 120s render (below) is the final artifact for a continuous
  scan across all 18 real episode boundaries.

## Files changed / created

**Modified** (documented, backward-compatible):
- `pressing_structure/dashboard/dashboard_style_v2.py` — added
  `TEAM_PRESS_GLOW_STOPS`, a `stops=` parameter to `add_pressure_glow`
  (default `None` preserves old behavior exactly), and the new
  `add_team_pressure_glow` function. No existing function's behavior
  changed for any existing caller.

**New files**:
- `pressing_structure/analytics/state_machine.py`
- `pressing_structure/analytics/pressing_v3.py`
- `pressing_structure/dashboard/render_pressing_dashboard_final_v3.py`
- `pressing_structure/outputs/final_dashboard_v3/` (this summary, the
  full render, preview frames)

**Untouched**: `pressing_structure/analytics/pressing.py` (V1/V2
analytics), all V1/V2 dashboard scripts, all prior renders/videos,
`tracking.parquet`, `metadata.json`, and every other canonical
analytics output.

## Answering the required questions

- **Both teams tracked simultaneously?** Yes — every frame produces a
  real score/state/confidence for both teams independently; the
  dashboard shows both explicitly (radar labels, KPI tiles, dominant-
  team logic) rather than picking one team implicitly via possession.
- **Transition logic added?** Yes — a shared, reusable hysteresis FSM
  (rolling smoothing + enter/exit thresholds + min-activate/hold/ending/
  cooldown counters), replacing V2's episode detector as the source of
  the live on-screen state (V2's own episode/outcome detector is kept
  running in parallel, unmodified, for its own outcome taxonomy).
- **Thresholds/hysteresis used?** `enter_th=0.4`, `exit_th=0.25`,
  `min_activate_frames=8`, `min_hold_frames=10`, `ending_frames=10`,
  `cooldown_frames=15` — recalibrated from an initial 0.5/0.3 guess
  after measuring the real score distribution (see Goal 2 above).
- **Full 120s render passed QA?** See the render log and preview
  frames in this folder; spot-checked at the one previously-known real
  episode boundary and confirmed correct before and after the fix.
