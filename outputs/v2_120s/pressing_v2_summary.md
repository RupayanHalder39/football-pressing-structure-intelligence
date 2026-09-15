# Pressing V2 Summary (full 120s clip)

**Scope**: rule-based analytics only. No predictive/learned model is used
anywhere in this document or its underlying code -- see
`pressing_structure/docs/METHODOLOGY.md` for the RULE-BASED vs LEARNED
separation policy. This is a V2 refinement of the v1 prototype
(`pressing_structure/outputs/pressing_dashboard.mp4`, 25s), which
remains untouched as the baseline.

## Episode count

**11 pressing episodes** detected over the full 120s clip (unchanged
from v1 -- V2 only changes how each episode's *outcome* is classified,
not detection itself; `detect_pressing_events` is reused verbatim).

## Refined outcome taxonomy: definitions

| State | Meaning | Logic |
|---|---|---|
| `BALL_REGAIN` | The pressed team lost the ball to a turnover within the outcome window | a `turnovers.parquet` row for the pressed team starts inside `[episode_end, episode_end + 8s]` |
| `FORCED_BACKWARD` | The pressed team escaped via a pass, but that pass carried the ball **backward** (toward their own goal) by >=3m | first qualifying pass's forward progress <= -300cm |
| `FORCED_LATERAL` | Escaped via a pass with less than 3m of net forward-or-backward progress | \|forward progress\| < 300cm |
| `ESCAPED_PRESS` | Escaped via a pass that gained >=3m of forward progress -- a clean escape | forward progress >= +300cm |
| `UNCERTAIN` | No turnover and no qualifying pass found in the window, or pass endpoints unknown | never forced into one of the above |

`FORWARD_PROGRESS_THRESHOLD_CM = 300.0` (3m) is a disclosed, round
threshold, not fit to this clip's outcomes. Own-goal-end direction per
team is read from `offside_break.analytics.offside.DEFENDING_GOAL_X`
(measured from GK average position, not assumed) so "forward" always
means "toward the pressed team's own goal" regardless of which side of
the pitch they're attacking.

**Optional higher-level grouping** (documented, not silently applied):
`SUCCESSFUL_PRESS = BALL_REGAIN or FORCED_BACKWARD or FORCED_LATERAL`.
This reflects a specific tactical interpretation -- forcing the ball
backward or sideways denies the escaping team forward progress even if
possession isn't won outright -- and is reported here explicitly as
that interpretation, not as a redefinition of "success" that overrides
the underlying 5-state outcome.

## Refined outcome counts (11 episodes)

| Outcome (v2) | Count |
|---|---|
| BALL_REGAIN | 3 |
| FORCED_BACKWARD | 2 |
| FORCED_LATERAL | 1 |
| ESCAPED_PRESS | 1 |
| UNCERTAIN | 4 |

Under the optional grouping: **SUCCESSFUL_PRESS = 6**, ESCAPED_PRESS = 1,
UNCERTAIN = 4.

## How this changes v1's interpretation (measured, not assumed)

v1 used a 3-state taxonomy (SUCCESS / FAILED / UNCERTAIN): 3 SUCCESS, 4
FAILED, 4 UNCERTAIN. Mapping v1 to v2 per-episode:

| Episode | v1 | v2 |
|---|---|---|
| 0 | SUCCESS | BALL_REGAIN |
| 1 | FAILED | FORCED_BACKWARD |
| 2 | FAILED | FORCED_BACKWARD |
| 3 | UNCERTAIN | UNCERTAIN |
| 4 | UNCERTAIN | UNCERTAIN |
| 5 | FAILED | FORCED_LATERAL |
| 6 | SUCCESS | BALL_REGAIN |
| 7 | UNCERTAIN | UNCERTAIN |
| 8 | UNCERTAIN | UNCERTAIN |
| 9 | SUCCESS | BALL_REGAIN |
| 10 | FAILED | ESCAPED_PRESS |

All 3 v1 SUCCESS episodes and all 4 v1 UNCERTAIN episodes map unchanged
(BALL_REGAIN and UNCERTAIN respectively). **v1's 4 "FAILED" episodes
split 3 ways**: 2 became FORCED_BACKWARD, 1 became FORCED_LATERAL, and
only 1 was a genuine clean escape (ESCAPED_PRESS). This is a real,
disclosed reinterpretation, not relabeling: v1's binary taxonomy could
not distinguish "the press forced the ball backward/sideways" (a
partial tactical win for the pressing team) from "the pressed team
escaped forward cleanly." Measured on this clip, 3 of v1's 4 "failed"
presses actually denied forward progress -- under the documented
SUCCESSFUL_PRESS grouping, successful presses effectively double (3
-> 6) while clean escapes drop from "4 FAILED" to 1 true ESCAPED_PRESS.

## Coverage / limitations

- 11 episodes is a small sample; every per-outcome graph in
  `graphs/` is explicitly labeled "descriptive/exploratory only -- NOT
  statistical evidence" in its own title. No claim of statistical
  significance is made anywhere in this document.
- Outcome classification depends on `passes.parquet` /
  `turnovers.parquet` (existing PhD analytics, unmodified) and on a
  fixed 8s outcome window (`OUTCOME_WINDOW_SEC`, unchanged from v1); 4
  of 11 episodes (36%) still resolve to UNCERTAIN because no qualifying
  pass or turnover was found in that window -- this is a direct
  consequence of the underlying ball/pass detection's own sparsity
  (previously measured), not a V2 classifier weakness.
- `forward_progress_cm` requires both pass endpoints to be non-null;
  where a matched pass has a missing endpoint, the episode falls back
  to UNCERTAIN rather than guessing a direction.

## Dashboard: `pressing_dashboard_120s_v2.mp4`

Full 120s (3600 frames, 30fps, canvas 2304x1204). Layout: top-left
original video (carrier highlighted only when `possession_state ==
CONTROLLED`, pressers boxed, press-radius circle when a press is
active); top-right radar (pitch-control wash, carrier/presser markers,
open/blocked passing lanes); bottom-left 3 live scrolling graphs
(nearest defender distance, nearest defender closing speed, pitch
control at carrier -- an 8s trailing window, stable fixed y-axis,
current-time marker, gaps drawn as broken lines rather than
interpolated); bottom-right numeric metrics + episode/outcome text;
bottom timeline strip (color-coded per outcome, current-time marker,
legend with color swatches).

## Visual QA (mandatory pass, full-clip frames)

Inspected actual rendered frames at 0%, 25%, 50%, 75%, ~99.7% of the
clip, plus the midpoint of every one of the 5 distinct outcome
categories (BALL_REGAIN @3.77s, FORCED_BACKWARD @5.5s, UNCERTAIN
@16.47s, FORCED_LATERAL @44.43s, ESCAPED_PRESS @110.03s):

- Carrier is highlighted (yellow "CARRIER" box + radar marker) only
  when `possession_state == CONTROLLED`; confirmed correct at the
  FORCED_BACKWARD event frame (t=5.5s / f154, a frame that DOES have
  carrier data within that episode).
- One event-midpoint frame (FORCED_BACKWARD, exact frame 165) showed
  "NO PRESS" / "No confident ball carrier this frame" despite falling
  inside episode 1's [153, 177] frame range. Verified against the
  underlying features table this is correct, not a bug: the episode
  only has a confident-carrier feature row on 5 of its 25 frames
  (153-156, 177); the episode's start/end frames are the *bounding*
  range of qualifying frames, not a claim that every frame in between
  has carrier data. The dashboard is not fabricating carrier presence
  between sparse observations -- it honestly shows "no confident
  carrier" on frames where there genuinely isn't one.
- Presser boxes/highlighting, pitch-control wash, and open(green)/
  blocked(red) passing lanes all matched plausible on-pitch geometry at
  every inspected frame.
- Live graphs show correctly broken (not interpolated) segments
  matching the sparse confident-carrier coverage; current-time marker
  position matched the video's elapsed time at every checked frame;
  y-axis did not visibly jitter/rescale across frames (fixed range,
  1st-99th percentile + padding, computed once).
- Timeline current-time marker tracked correctly from frame 0 (left
  edge) to frame 3581 (~right edge); episode color bars and the
  color-swatch legend rendered without overlapping text at every
  checked frame.
- No stale metrics or leftover previous-episode data observed on
  transitions between episodes.

No problems requiring a fix/re-render were found in this QA pass.

## Bugs found and fixed during V2 development

- `live_graphs.py`'s per-graph `y_label` was originally drawn at
  `(width - 4, height - 6)`, which is wrong for cv2's left-anchored
  text (would render off the right edge). Fixed to left-align at
  `pad_l` before the first render; caught in code review, not at
  runtime.
- The timeline legend originally concatenated plain outcome-name text
  with no visual link to the color bars. Fixed to draw a small colored
  swatch before each label (see `draw_timeline`), re-verified via a
  fresh test render.

## Runtime / storage

- Outcome-taxonomy classification (`classify_press_outcomes_v2`) on
  the full 120s clip: ~2.1s.
- Full 120s dashboard render: ~6 minutes (3600 frames).
- Static graph generation (`generate_pressing_graphs.py`): ~7s,
  produced 9 PNGs.
