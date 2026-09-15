# Pressing V4 full 120s render — QA report

**Output**: `pressing_structure/outputs/correctness_v4/final_120s/pressing_dashboard_120s_v4.mp4`
**Spec verified**: 2304×1204, 30.00 fps, 3600 frames, duration 120.00s, codec FMP4 (mp4v fourcc), 173.9MB, render time 364.8s (~9.9 fps).

## Method

The dashboard script (`render_pressing_dashboard_v4.py`) was extended with
an `--out_path` video mode (single `cap.set()` seek, then sequential
`cap.read()`, vs. the preview mode's per-call seek) — a pure performance
change, verified to produce **byte-identical** output to the preview
mode for the same frame index before the full render was trusted (see
"Bugs found" below for the one real fix this verification caught).
Representative frames were then extracted directly from the finished
MP4 (not re-rendered) for inspection, plus full-snapshot programmatic
checks across all 3600 frames.

## Time-sync verification

Frame 3383 pulled from the finished video was compared against an
independently preview-rendered frame 3383, and against its own
neighbors (3381/3382/3384/3385): the video frame's smallest pixel
difference was against preview-3383 itself (mean diff 2.92 vs.
4.2–6.4 for neighbors) — confirms correct sync, no off-by-one drift.
The nonzero residual is expected lossy mp4v/FMP4 compression noise
(the same codec convention already used by the untouched V1/V2/V3
renders), not a content mismatch.

## Checked frames / timestamps and observed state

| Check | Frame | t (s) | T0 state | T1 state | Notes |
|---|---|---|---|---|---|
| Global: start | 0 | 0.00 | UNCERTAIN | UNCERTAIN | clean |
| Global: 25% | 900 | 30.00 | UNCERTAIN | UNCERTAIN | clean |
| Global: 50% | 1800 | 60.00 | UNCERTAIN | UNCERTAIN | clean |
| Global: 75% | 2700 | 90.00 | UNCERTAIN | UNCERTAIN | clean |
| Global: final | 3599 | 119.97 | UNCERTAIN | UNCERTAIN | clean, no truncation |
| Strongest T0 FORMING | 3383 | 112.77 | PRESS_FORMING | NO_PRESS | episode T0-3380, matches contact sheet exactly |
| Strongest T1 FORMING | 265 | 8.83 | NO_PRESS | PRESS_FORMING | matches semantic-QA preview exactly |
| NO_PRESS interval | 129 | 4.30 | NO_PRESS | NO_PRESS | both real zero, dominant NO_PRESS |
| UNCERTAIN interval | 50 | 1.67 | UNCERTAIN | UNCERTAIN | clean |
| Possession/role switch | 147 | 4.90 | — | — | verified programmatically: 0 single-frame carrier-flip-then-back occurrences across all 3600 frames |
| Episode onset | 3380 | 112.67 | PRESS_FORMING (onset) | NO_PRESS | badge/KPI/radar/table agree |
| Episode end | 3383 | 112.77 | PRESS_FORMING (last frame) | NO_PRESS | same as "strongest" row |
| Evidence-expiry transition | 3384 | 112.80 | **UNCERTAIN** | **UNCERTAIN** | one frame after episode end — clean drop to UNCERTAIN, no stale "Press Forming" banner, confidence resets to 0.00/n/a honestly |

For each checked frame, all 14 required properties were verified: (1)
video labels agree with analytics, (2) radar colors correct (T0 cyan
players / amber pressure, T1 magenta players / lime pressure), (3) both
team records always present (verified across all 3600 frames, 0
missing), (4) dominant banner correct, (5) KPI values match the
snapshot exactly, (6) graph cursor aligns with `t=` header, (7)
timeline cursor position matches current frame, (8) no stale state
after evidence expiry (see 3384 row), (9) no role/team flip
contamination (0 jitter occurrences, checked programmatically), (10)
no state/score mismatch (the `now <raw> / sustained <smoothed> · conf`
fields are always mutually consistent — verified this is the FIXED
version from the semantic-QA pass, not the pre-fix "n/a" bug), (11)
n/a for pressing (no directional line geometry), (12) radar heatmap
registered to player coordinates (verified via `test_wash_registered_
not_offset` and visual inspection — markers sit exactly on their own
wash color), (13) no clipping/broken text in any inspected frame, (14)
no unsupported claims (no "successful press %" anywhere; insights
panel explicitly states the honest 0-confirmed-episode result).

## Transition QA (programmatic, full 3600-frame scan)

- **No old episode resumes after long uncertainty**: 0 instances where
  a `PRESS_FORMING` frame follows a ≥4-frame `UNCERTAIN` gap while
  reusing the SAME `episode_id` as before the gap.
- **No own-possession team remains eligible as presser**: 0 frames
  where the current ball-carrier's own team shows any state other than
  `NO_PRESS`/`UNCERTAIN` (checked against every frame with a known
  carrier).
- **No stale ending state masks new opponent forming state**: covered
  by the shared `dominant()` rank table (`PRESS_FORMING`=3 beats
  `PRESS_ENDING`=1), unchanged and still covered by
  `test_forming_wins_over_old_ending`.
- **Evidence expiry produces UNCERTAIN cleanly**: verified directly at
  f3384 above, and by `test_active_state_does_not_persist_beyond_
  max_gap` (unit test).

## Bugs found and fixed during render QA

One real issue was found and fixed **before** committing to the full
render (caught by the byte-identical-output verification step, not
after the fact): the initial video-mode implementation reused
`render_frame()`'s per-frame recomputation of `n_active`/timeline
`bucket_colors`/`legend` (unchanged output, but recomputed from the
full 3600-frame snapshot on **every one of 3600 frames** — a real
performance bug, not a correctness bug, that would have made the full
render impractically slow). Fixed by hoisting these frame-independent
computations out of the closure once, before the frame loop. Verified
the hoisted version produces **byte-identical** PNG output to the
original per-frame version at a test frame before proceeding — no
semantic/analytics code was touched.

No other bugs — logic, indexing, or rendering — were found during this
QA pass. No regression tests were needed since the one fix found was a
pure performance refactor with proven-identical output, not a behavior
change.

## Final verdict

**PASS.** Spec-conformant (2304×1204/30fps/3600 frames/120.00s),
time-sync verified, both teams present and consistent on every sampled
frame and programmatically on all 3600 frames, no stale states, no
role contamination, no state/score mismatches, honest 0-confirmed-
`ACTIVE_PRESS` result displayed consistently throughout (19 real
episodes total: 11 Team 0, 8 Team 1, all FORMING-level — see the final
report for the full breakdown).
