# Longer-Video Strategy (V2, Part 5) -- Pressing

**Status: analysis and cost estimate only. No new tracking export was
run.** Per the brief ("do NOT render full 5-minute dashboard unless
explicitly approved after seeing event counts and runtime estimate"),
this reports the decision inputs and stops there.

## Candidate segment

`ExternalDownlaodVideo/testVideo1.mp4` is 334.87s total (10,046 frames
@ 30fps) -- the source the existing 120s clip was extracted from.
Proposed candidate: **seconds 0-300 (a full 5 minutes)**, i.e. the
existing, already-tracked 0-120s clip **plus a new 120-300s (180s)
tail** that has never been processed. This is the cheapest 5-minute
option available (reuses `outputs/tracking/testVideo1_120s/
tracking.parquet` for the first 120s instead of re-deriving it) and
avoids arbitrarily cherry-picking a "more interesting-looking" segment
without evidence of what's actually in it.

## Why analytics cannot be "run over 5 minutes first" without new tracking

Pressing/offside analytics are built entirely on top of a tracked
`tracking.parquet` (player boxes + team IDs + pitch coordinates) and a
ball track. Frames 120-300s (5,400 frames) have neither yet -- there is
no shortcut that produces real analytics without first running the
tracking pipeline on that new footage. This project's own prior
measurement (`PLAN.md`, full-clip cost table, 60s-clip-derived
per-frame rates) gives the extrapolated cost for those 5,400 new
frames:

| Stage | Measured rate | 5,400 new frames |
|---|---|---|
| `export_tracking.py` | 2.011 s/frame | **~3.02 hours** |
| `detect_ball_specialized.py` (stride=2, 2,700 processed frames) | 0.973 s/processed-frame | ~0.73 hours |
| `replay_homography.py` | 0.192 s/frame | ~0.29 hours |
| `build_analytics.py` | negligible | <1 min |
| **Base tracking pipeline subtotal** | | **~4.0 hours** |

`export_tracking.py` alone is ~75% of this, for the same reason
already flagged in `PLAN.md` for the full video: its per-frame SigLIP
team-classification call runs on every visible player crop, every
single frame -- a known bottleneck this project has explicitly
deferred fixing (a temporal/cached classification redesign), not
something specific to this V2 request.

On top of the base pipeline, producing V2-quality dashboards for a full
300s clip would add (measured from this session's 120s V2 renders,
~0.1s/frame each):

| Item | Cost |
|---|---|
| Pressing V2 + Offside V2 analytics (both directions) | ~1-2 min |
| Pressing V2 dashboard render (9,000 frames) | ~15 min |
| Offside V2 dashboard render (9,000 frames) | ~15 min |
| Static graphs (both projects) | ~1 min |

**Grand total estimate: ~4.5-4.7 hours of unattended compute**, ~85% of
it the export_tracking.py step on the new 180s tail alone.

## Extrapolated event counts (from the measured 120s rate -- NOT a new measurement)

Scaling this session's actual 120s counts linearly to 300s (2.5x):

| Metric | 120s (measured) | 300s (extrapolated, x2.5) |
|---|---|---|
| Pressing episodes | 11 | ~27-28 |
| Offside attacking runs (both directions) | 374 | ~935 |
| Offside trap-break candidates | 0 | **cannot be reliably extrapolated from a zero count** -- see caveat below |

**Caveat on trap-break candidates**: 0 was observed in 120s, but the
trap-synchrony score DID reach 0.66-0.67 in brief spikes (just under
the 0.65-sustained-for-15-frames threshold) -- see
`offside_v2_summary.md`. A longer sample *might* contain a genuine
sustained TRAP_ACTIVE episode, or might not; extrapolating a rare/zero
event count from 120s of data is not statistically defensible either
way, and this document does not claim a number for it.

## Recommendation

**Do not run the ~4.5-hour export.** This report exists so the user
can decide with real numbers in hand, per the brief. If approved, the
efficient path is: run `export_tracking.py` /
`detect_ball_specialized.py` / `replay_homography.py` on only the new
120-300s tail, concatenate with the existing 0-120s tracking data, then
rerun the already-built V2 pressing/offside pipelines (no code changes
needed -- both already default to `outputs/tracking/testVideo1_120s`
but accept a `--tracking_dir` override).

## Event-centered clips: rendered from the EXISTING 120s clip instead

Since the 5-minute expansion was not run, event-centered clips were
produced from the 120s clip's own already-computed pressing episodes
(all 11 -- see `pressing_structure/outputs/event_clips/` and its
`README.md` for exactly which frames/why). This satisfies the "clips
around each detected event" deliverable using data already fully
analyzed and QA'd in this session, without waiting on the gated
5-minute decision above.
