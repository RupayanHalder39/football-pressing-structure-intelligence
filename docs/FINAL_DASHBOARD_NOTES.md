# Final Dashboard Visual Redesign (Pressing)

Pure visual/layout restyle -- **no analytics logic changed**. Same V2
functions (`build_possession`, `build_pressing_features`,
`detect_pressing_events`, `classify_press_outcomes_v2`) reused
unmodified from `pressing_structure/analytics/pressing.py`. See
`pressing_v2_summary.md` for the actual episode/outcome findings,
which are unchanged by this restyle.

## What changed

- New dark-card visual language (`pressing_structure/dashboard/
  dashboard_style.py`, shared with the offside final dashboard) modeled
  on two locked reference mockups the user supplied -- layout,
  hierarchy, panel sizing, color treatment, and typography only. No
  value, label, threshold, or event category from those mockups was
  used; every number on screen comes from the real V2 analytics above.
- Canvas stays 2304x1204 (unchanged). Layout: dark header strip (title
  + live episode/outcome status + clock, no fake score) -> top row
  (video feed + radar, 730px tall = 60.6% of the canvas height, per the
  locked layout rule) -> 4 metric tiles (pressers/nearest defender/
  pitch control/compactness) -> 5 graph panels (nearest defender
  distance, closing speed, pitch control at carrier, local compactness,
  and a press-events-per-10s bar histogram) -> full-width event
  timeline (V2's 5-state outcome taxonomy, color-coded, legend
  swatches).
- Radar pitch-control wash was tuned more vivid (higher blend weight)
  to match the reference's clear cyan/magenta split -- purely a
  rendering parameter, not a change to `compute_pitch_control_grid`
  itself (imported read-only, untouched).

## QA-driven fixes made during this restyle

- Current-time marker on scrolling graphs was low-contrast against a
  bright bar-chart background (magenta bars in the events-per-10s
  panel) -- fixed by drawing a black outline behind the white marker
  line in `dashboard_style.py`, applied to every graph type.
- The events-per-10s histogram's timeline-style background is now
  rendered ONCE (static bars don't change across the video) with only
  the marker redrawn per frame, for performance.

## Output

`pressing_structure/outputs/final_dashboard/pressing_dashboard_120s_final.mp4`
(full 120s, 3600 frames, 2304x1204). `pressing_structure/outputs/
v2_120s/pressing_dashboard_120s_v2.mp4` (V2's own layout) is superseded
by this restyle and was deleted once this file passed full QA -- its
hash/size is recorded in the final report for this stage.
