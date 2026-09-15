# Pressing Structure — Methodology

Separate research prototype, built alongside the existing PhD Problem 1
pipeline. Does not modify, retrain, or reinterpret any PhD conclusion.

## Research question

"Who is pressing, how is the space being closed, and what spatial
structure is associated with a successful or failed press?"

## Input data

`outputs/tracking/testVideo1_120s/tracking.parquet` (RAW, read-only,
never modified) + the EXISTING, CANONICAL, UNCHANGED derived files
under `outputs/analytics/testVideo1_120s_v3/`:
`ball_trajectory.parquet`, `passes.parquet`, `turnovers.parquet`,
`homography_transformers.pkl`. No new model inference is run anywhere
in this project -- everything here is geometry over already-tracked
positions.

## Assumptions

1. A "ball carrier" only exists when the EXISTING PhD `analytics.
   possession` module reports `possession_state == CONTROLLED`. Any
   other state (UNCERTAIN/FREE_BALL/BALL_LOST) means NO pressing
   features are computed for that frame -- never guessed.
2. "Defending team" = whichever team is NOT the carrier's team, only
   when the carrier's team itself is confidently known.
3. Player positions/velocities come from this project's own
   `cleaned_tracking_view.py` (see below), not raw `tracking.parquet`
   track_id sequences directly.
4. All thresholds below are fixed, stated constants, chosen from
   either (a) values the PhD pipeline itself already validated
   (the 2000 cm/s impossible-jump threshold, reused verbatim from
   `scripts/quality_report.py`), or (b) simple, round, disclosed
   choices (3/5/8m presser radii, 100 cm/s closing-speed trigger) --
   never fit to make the results "look right."

## Cleaned analysis view (shared with `offside_break/`)

`pressing_structure/analytics/cleaned_tracking_view.py`. Reuses the
EXISTING PhD `analytics.position_fill.build_full_display_frame`
(role/team stabilization + short-gap fill, `apply_smoothing=False`)
unchanged, then adds ONE new thing those utilities don't do: explicit
rejection of frame-to-frame pitch transitions exceeding 2000 cm/s (the
SAME threshold `scripts/quality_report.py` already flags with, measured
independently to affect 50% of raw tracks -- see `outputs/tracking/
testVideo1_120s/quality_summary.md`). A rejected transition starts a
new `cleaned_segment_id` for that track; smoothing (causal EMA, reused
from `analytics.temporal_utils.causal_ema`) and velocity are computed
PER SEGMENT, never bridging a rejected jump. Measured effect on this
clip: 136 raw tracks -> 1348 cleaned segments (mean 9.91 segments per
track) -- i.e. the average continuously-trustworthy trajectory piece is
short (~2s). **This directly limits how much confidence any velocity-
based feature (closing speed, run velocity in the offside project) can
carry over a long span** -- disclosed, not hidden.

Note on the "1212 vs 773 flagged transitions" discrepancy: `quality_
report.py` counts jumps between consecutive NON-NULL raw values only
(which can span a long null gap in one big step); this module counts
jumps AFTER short-gap interpolation (max 3 frames, reused from
`position_fill.py`), which can split one long-gap jump into several
flagged interpolated sub-steps of the same anomaly. Both numbers are
correct for what they each measure; they are not directly comparable.

Ball position: loaded read-only from the EXISTING `ball_trajectory.
parquet` (OBSERVED/PREDICTED/LOST distinction preserved exactly as the
main dashboard already validated it) -- no new ball logic.

## Exact feature definitions

| Feature | Definition |
|---|---|
| ball_carrier | `possessing_track_id` where `possession_state==CONTROLLED` (PhD `analytics.possession`, reused) |
| nearest_defender_distance_cm | min Euclidean pitch distance, carrier to any opposing player |
| nearest_defender_closing_speed_cm_s | component of the nearest defender's velocity vector directed toward the carrier's current position (positive = approaching) |
| n_pressers_3m/5m/8m | count of opposing players within 300/500/800cm of the carrier (position only, not closing speed) |
| local_numerical_balance | (attacking teammates within 1000cm of carrier) − (defenders within 1000cm) |
| n_passing_options | teammates within 1500cm of the carrier |
| n_open_lanes / n_blocked_lanes | a lane carrier→teammate is "blocked" if any defender is within 200cm of the straight-line segment between them (point-to-segment distance) -- a geometric proxy, NOT a modeled interception probability |
| defending_team_width/depth_cm | max−min y / max−min x among ALL currently-visible defending outfield players (global, not local) |
| local_compactness_cm | mean pairwise distance among defenders within 1500cm of the carrier |
| pitch_control_for_carrier_team | the EXISTING PhD `analytics.pitch_control` continuous surface (unmodified), sampled at the grid cell nearest the carrier's own position |
| escape_space_score_cm | == nearest_defender_distance_cm. Explicitly a simple proxy, not a separately modeled "space" metric -- larger distance is read as "more escape space" |

## Event detection rules

- **PRESS ACTIVE** (a frame is "under press"): >=1 defender within
  500cm of the carrier (PRESS_TRIGGER_RADIUS_CM) with closing speed
  >=100 cm/s toward the carrier.
- Consecutive under-press frames form an episode; a gap of <=15 frames
  (0.5s) between two under-press stretches is bridged (treated as one
  episode) to avoid slicing a real press into micro-episodes on
  single-frame jitter -- the underlying feature values in the bridged
  frames are real, not fabricated.
- **PRESS START/END** = the episode's first/last under-press frame.
- **Outcome**, decided ONLY from the EXISTING passes/turnovers data
  (never re-derived from raw positions):
  - SUCCESS: a turnover with the pressed team as source starts within
    [press end, press end + 3.0s].
  - FAILED: the pressed team completes a pass (in `passes.parquet`)
    ending within that same window (they kept the ball).
  - UNCERTAIN: neither -- never forced into SUCCESS/FAILED.

## Uncertainty handling

Every stage refuses to guess: no carrier -> no features; no opposing
player visible -> no features; ambiguous outcome -> UNCERTAIN, not a
coin-flip. Nothing here claims possession, a press, or an outcome that
the underlying data does not support.

## What is analytics vs. what is learned

**Everything in this project is rule-based analytics** -- geometry,
thresholds, and reuse of already-validated PhD modules. No model was
trained. See "Is supervised modeling justified" in `outputs/
pressing_summary.md` (answer: no, only 11 events on one clip).

## Known limitations

1. Ball-possession sparsity (258/3598 frames, 7.2%) is the dominant
   constraint on coverage.
2. 11 total detected episodes -- far too few for any statistical claim
   about "the best" pressing structure; this is a prototype capable of
   showing individual episodes clearly, not a generalizable finding.
3. Escape space and passing-lane blockage are simple geometric proxies,
   explicitly not validated space-control or interception models.
4. Press outcome quality is capped by the existing pass/turnover
   pipeline's own measured precision (~0.80) and recall (~0.53) --
   documented in the main PLAN.md.
5. Dense multi-presser frames can crowd on-screen labels (3+ "PRESSER"
   tags stacked close together) -- readable but not elegant; not fixed
   in this prototype pass.
6. Only a 25-second window was rendered to video; the remaining ~95s of
   the clip's pressing events (6 more, per `pressing_events.csv`) were
   analyzed but not rendered to video in this pass.

## What the dashboard shows / does NOT prove yet

Shows: a specific, inspectable, geometry-grounded account of who is
near the ball carrier, how fast they're closing, how many passing
options exist, and what happened next (per the existing pass/turnover
data) for the episodes this rule actually detected.

Does NOT prove: which spatial structure is "best" for pressing in
general (11 events, one clip, one broadcast angle, one detection
pipeline's own measured error rates) -- that would require far more
labeled data and, per the project brief, only a logistic regression or
gradient-boosted baseline even then, never a claim beyond what such a
small sample can support.
