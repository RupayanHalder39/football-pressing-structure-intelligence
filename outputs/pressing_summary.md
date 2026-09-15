# Pressing Structure — Prototype Summary

Input: `outputs/tracking/testVideo1_120s/tracking.parquet` (READ-ONLY,
unchanged) run through this project's own `cleaned_tracking_view.py`,
combined with the EXISTING PhD `analytics.possession` and
`analytics.pitch_control` modules (unmodified) and the EXISTING
`outputs/analytics/testVideo1_120s_v3/{passes,turnovers}.parquet`
(read-only, used only to decide press outcomes).

## What was run

Full 120s clip, analytics only (1.99s runtime): possession computed for
every frame, pressing features computed for every frame with a
CONTROLLED ball carrier AND at least one visible opposing player,
events detected via a rule-based proximity+closing-speed trigger,
outcomes classified from the existing passes/turnovers data.

## Headline numbers (full 120s)

- Frames with a confident ball carrier (possession_state == CONTROLLED):
  258 / 3598 (7.2%) -- the same ball-observation sparsity already
  documented for the main PhD dashboard limits how much of the clip
  pressing can even be assessed on.
- Frames with a usable feature row (carrier confident AND >=1 opposing
  player visible): 249.
- Pressing episodes detected: **11**, across the whole 120s.
- Outcomes: 3 SUCCESS, 4 FAILED, 4 UNCERTAIN (never forced -- see
  METHODOLOGY.md for the exact turnover/pass-based rule).

## Prototype dashboard

Rendered on a 25-second window (0-25s of `testVideo1_120s.mp4`, chosen
because it contains 5 of the 11 detected episodes -- dense enough for
meaningful visual QA) rather than the full 120s, per the explicit
"prototype short, validate, then extend" instruction.

`pressing_structure/outputs/pressing_dashboard.mp4` -- 751 frames,
2044x920, ~25s.

## Visual QA (see PLAN.md/HANDOFF.md note + this project's own
## METHODOLOGY.md "Known Limitations" for the full account)

Inspected actual rendered frames at the beginning, middle, end, and 5
distinct detected events. **One real bug found and fixed**: the first
render highlighted "CARRIER" on screen for a frame where possession was
only UNCERTAIN (the metrics panel correctly refused to compute features
for that same frame) -- a genuine label/logic mismatch, not a rendering
glitch. Fixed by gating the on-screen carrier highlight on
`possession_state == CONTROLLED`, the same bar the feature/metrics code
already used. Re-rendered and re-inspected; the two panels are now
consistent everywhere checked. A second apparent anomaly (what looked
like 2 presser rings in one frame) was checked by zooming into the
actual pixels and found to be correct: 1 presser ring + 2 passing-lane
lines (1 open, 1 blocked), matching the metrics text exactly.

## Known limitations (see METHODOLOGY.md for full list)

- Ball-possession sparsity (7.2% confident-carrier coverage) is the
  dominant constraint -- most of the 120s cannot be assessed for
  pressing at all, not because pressing isn't happening, but because
  the ball carrier isn't confidently known.
- 11 events total is far too few to fit any supervised model on.
- "Escape space" is a simple nearest-defender-distance proxy, not a
  modeled space metric.
- Passing-lane blockage is a straight-line-distance geometric proxy,
  not a modeled interception probability.
- Press outcome uses only the existing passes/turnovers datasets, which
  themselves have known precision/recall limits (see PLAN.md's pass-
  pipeline diagnosis) -- an outcome label is only as good as those.

## Is supervised modeling justified right now?

**No.** 11 detected episodes on one 120-second clip is nowhere near
enough to fit even a simple logistic regression responsibly (it would
almost certainly just memorize noise). This is explicitly a rule-based
analytics + visualization prototype, and is reported as exactly that.
