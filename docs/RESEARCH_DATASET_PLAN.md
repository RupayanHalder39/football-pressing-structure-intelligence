# Pressing Research Dataset Plan

2026-09-08. Written at the start of the pressing research phase (dashboard
now frozen). Every number below was measured by actually running the
current pipeline against the real 120s clip
(`outputs/tracking/testVideo1_120s/tracking.parquet`,
`outputs/analytics/testVideo1_120s_v3/`) — nothing here is estimated or
assumed.

## 1. Current data

**Source footage**: one 120s (3600-frame, 30fps) clip,
`ExternalDownlaodVideo/testVideo1_120s.mp4`, 2 periods collapsed to one
(`PitchConfig.periods = ((0, True),)` — no real half-time break inside
this clip). One match only.

**Tracking**: `outputs/tracking/testVideo1_120s/tracking.parquet` — per-
frame per-object rows (`track_id`, `display_team_id`,
`display_object_type`, `x_pitch`/`y_pitch`, `vx_cm_s`/`vy_cm_s`,
`motion_valid`, `team_confidence`, `cleaned_segment_id`, provenance
fields), produced by `tactical_shared.tracking.build_quality_view`.
Ball view: `outputs/analytics/testVideo1_120s_v3/` via
`pressing_structure.analytics.cleaned_tracking_view.load_ball_view`.

**Current carrier/possession resolution**
(`tactical_shared.tracking.current_roles`): a nearest-player-within-
`CONTROL_RADIUS_CM` rule reused from `analytics.possession.
compute_possession`, reporting `possession_state ∈
{CONTROLLED, EXTENDED_UNCERTAIN, FREE_BALL, NO_VISIBLE_PLAYERS,
BALL_UNAVAILABLE}`. Measured on the real clip: `CONTROLLED` (instant
carrier resolved) in **144/3600 frames (4.0%)**. A separate,
already-documented **trailing-4-second majority hold**
(`historical_context_role`) additionally resolves a defending team in
**1,532/3,600 frames (42.6%)** with no instant carrier — used by the
dashboard's continuous field, and reusable for research feature
attribution too. The remaining ~53.4% have no team attribution.

**Current pressing participant logic** (`pressing_v4.frame_features`):
per team, per frame — nearest opponent distance/identity, opponents
within 3m/5m, relative closing speed of the nearest opponent, a
`participant_ids` list (opponents within 5m closing at ≥100 cm/s),
`support_minus_opponents` (attacking team's local numerical balance),
local visible compactness, and a partial-evidence `intensity` score
(proximity 35% + count 30% + closing 35%, closing dropped from the
weighted average — not zeroed — when motion is untrustworthy) with a
**separately tracked** `confidence` (identity/coverage reliability,
never blended into intensity).

**Current press episodes** (`pressing_v4.build_pressing_v4` →
`tactical_shared.temporal.EvidenceFSM`): a per-team evidence-hold state
machine (`NO_PRESS → PRESS_FORMING → ACTIVE_PRESS → PRESS_ENDING`,
generic thresholds — `enter=0.4`, `exit=0.25`, `min_observations=2`,
`min_supported_sec=0.25s`, `min_coverage_ratio=1/3`, `max_gap_sec=0.1s`,
`min_hold_sec=1/3s`, `cooldown_sec=0.5s` — structural, not clip-tuned;
see that class's own docstring). **Measured on the real clip: 19
episodes total (Team 0: 11, Team 1: 8), 0 of which ever reach
`ACTIVE_PRESS`** (`active_start` stays `None` for all 19 — every
episode is `PRESS_FORMING`-level evidence that later expires or is
context-interrupted, never sustained/confident enough to confirm
`ACTIVE_PRESS` under the generic bar). This is the single most
important existing constraint on this dataset (see §9).

**Current outcome labels**: `pressing_v4.attribute_outcomes` produces
`observed_outcome ∈ {BALL_REGAIN, BACKWARD_ACTION, LATERAL_ACTION,
FORWARD_PROGRESSION, UNCERTAIN}` by matching a later pass/turnover to
an episode — **but its own `events` list is built as `[e for e in
episodes if e['active_start'] is not None]`, i.e. it only ever attaches
an outcome to an episode that reached `ACTIVE_PRESS`. Measured: 0/19
episodes qualify, so `events == []` on this clip today.** No episode
currently carries a heuristic outcome under the pipeline exactly as it
stands. §4/§9 address this directly (outcome attribution is
generalized in this research phase to run on every episode by its
`termination_time`, not gated on `active_start`).

There is also a **legacy, separate V2 pipeline**
(`pressing_structure/analytics/pressing.py`:
`build_possession`/`build_pressing_features`/`detect_pressing_events`/
`classify_press_outcomes_v2`) that produced the (now-deleted, per the
2026-09-08 video-only cleanup) `event_clips/episode_XX_<LABEL>.mp4`
files, using a different event-detection method and a different,
coarser outcome vocabulary (`BALL_REGAIN`, `FORCED_BACKWARD`,
`FORCED_LATERAL`, `UNCERTAIN`, `ESCAPED_PRESS`). **This research phase
builds on the current V4 pipeline (`pressing_v4.py`), not the legacy V2
one** — V2 is noted here only because its label vocabulary partly
overlaps with §2's new taxonomy and is worth being aware of, not
confused with.

**Current feature columns available per frame/team** (from
`frame_features`, already computed, never re-derived): `
nearby_opponents_3m, nearby_opponents_5m, nearest_opponent_distance_cm,
nearest_opponent_track_id, relative_closing_cm_s,
defender_velocity_toward_carrier_cm_s, participant_ids,
support_minus_opponents, local_visible_compactness_cm, coverage,
intensity, confidence, components_used, raw_score`. Full-pitch spatial
fields (`pressure_field.py`, unchanged, reused not reimplemented):
`team_pressure_field`, `escape_value_field`, `pressure_gradient`,
`escape_directions`, `passing_lane_metrics`,
`contour_coverage_fractions`.

**Current episode-level exports**: none existed before this round.
Part 3 of this phase adds the first one
(`pressing_structure/outputs/research_dataset/pressing_episode_features.parquet`).

## 2. Required future data

To make the pipeline below (§6–§8 of this doc; Parts 4–9 of the
research phase) statistically meaningful rather than illustrative:

1. **More matches** (this is a single-clip, single-match dataset today
   — every metric in this document and every later experiment is
   reported as a measurement on ONE clip, not a generalizable result).
   At minimum 5–10 full matches, ideally from multiple teams/leagues,
   is needed before match-grouped cross-validation (§7) has enough
   groups to be meaningful.
2. **Real human labels** (Part 2's protocol) for a meaningful sample of
   candidate press windows in each match — both to validate the
   heuristic outcome labels against and to train/evaluate outcome
   models that don't just reproduce the heuristic's own biases.
3. **Denser ball-carrier resolution** — 4.0% instant / 46.6% combined
   (instant + 4s historical hold) is a real, disclosed tracking-quality
   ceiling of this specific source video, not a pipeline defect; better
   source footage or an upstream possession-model improvement would
   directly increase usable episode density.
4. **A confirmed `ACTIVE_PRESS` example** — needed to validate the
   FSM's own active/confirmed tier ever fires correctly on real,
   sustained pressure, not just to hit a count target (an evidence bar
   set to guarantee a hit is not a real validation — see
   `EvidenceConfig`'s own docstring).
5. **Team/formation metadata** (starting XI, formation, home/away) —
   not currently in the tracking parquet — needed for Experiment 8
   (generalization across teams) and any archetype-vs-team-identity
   analysis.

## 3. Minimum episode schema

One row per pressing episode (from `EvidenceFSM`'s own real output,
never invented fields):

| field | type | source |
|---|---|---|
| `episode_id` | str | `EvidenceFSM` (`T{team}-{onset_frame}`) |
| `team` (pressing team) | int (0/1) | `EvidenceFSM` |
| `onset_frame`, `onset_time` | int, float | `EvidenceFSM` |
| `active_start`, `active_end` | int or null | `EvidenceFSM` (null if never confirmed `ACTIVE_PRESS`) |
| `confirmation_time` | float or null | `EvidenceFSM` |
| `termination_frame`, `termination_time`, `termination_reason` | int, float, str | `EvidenceFSM` (`EVIDENCE_EXPIRED`/`CONTEXT_CHANGED`/`FORMING_ABORTED`/`SCORE_ENDED`/`ROLE_INELIGIBLE`/`CLIP_END`) |
| `end_frame`, `end_time` | int, float | `EvidenceFSM` (= `last_observed_frame`) |
| `supported_duration_sec`, `observations` | float, int | `EvidenceFSM` |
| `max_state_reached` | str | derived: `ACTIVE_PRESS` if `active_start` set, else `PRESS_FORMING` |
| `heuristic_outcome` | str (§2 taxonomy) | this phase's generalized `attribute_outcomes` (§9), evaluated at `termination_time`, not gated on `active_start` |
| `human_outcome` | str or null | Part 2 protocol, empty until labeled |
| `final_adjudicated_outcome` | str or null | Part 2 protocol, empty until adjudicated |

## 4. Frame-level schema

One row per (frame, team) with computable pressing evidence — the
existing `frame_features()` output, unchanged, plus the frame's
attributed episode (if any):

`frame, time_sec, team, episode_id (nullable), state, raw_score,
confidence, nearby_opponents_3m, nearby_opponents_5m,
nearest_opponent_distance_cm, nearest_opponent_track_id,
relative_closing_cm_s, defender_velocity_toward_carrier_cm_s,
participant_ids, support_minus_opponents,
local_visible_compactness_cm, coverage, components_used`

## 5. Episode-level schema (features + outcome, joined)

`episode_id` (join key) + every column from §3 + every feature column
from Part 3's `pressing_episode_features.parquet` (see
`pressing_structure/docs/FEATURE_SCHEMA.md`, generated alongside it) +
`match_id` (constant `testVideo1_120s` today, required once more
matches exist) + `period`.

## 6. Label schema

See `pressing_structure/docs/PRESS_LABELING_PROTOCOL.md` (Part 2) for
the full human-labeling schema. Summary of the 3-tier outcome design:
- `heuristic_outcome`: produced automatically by the pipeline (§9),
  never assumed correct.
- `human_outcome`: one independent annotator's judgment via the
  labeling tool (Part 12).
- `final_adjudicated_outcome`: agreed/reconciled value used for
  evaluation and modeling — equals `human_outcome` when only one
  annotator exists (current state), and a reconciled value once ≥2
  annotators overlap on the same clip.

## 7. Train/validation/test split strategy

**Not applicable to run on today's single-match dataset** — there is
exactly one `match_id`. The strategy below is specified now so it is
ready the moment a second match exists, and so no leakage-prone split
is ever accidentally used in the meantime:

- **Unit of splitting = match**, never frame or episode. All episodes
  from one match go entirely into one split.
- With ≥5 matches: match-grouped k-fold (§ Part 11), k = number of
  matches or 5, whichever is smaller.
- With <5 matches: leave-one-match-out, reporting per-match results
  individually (not pooled into one misleadingly-precise number).
- **Never**: a random frame- or episode-level split within one match —
  explicitly forbidden by Part 11 (adjacent episodes share players,
  tactical context, and tracking-quality regime; a random split leaks
  match identity into "test").
- On the current 1-match dataset, any reported number is a
  **descriptive measurement on that match**, not a train/test
  evaluation, and is labeled as such everywhere in this phase's
  reports.

## 8. Match-level grouping rules to avoid leakage

1. Split by `match_id`, enforced at the data-loading layer, not by
   convention.
2. A model trained with match A in its training fold must never see
   match A's episodes, features, or outcome labels at evaluation time
   — including indirectly via a shared team's archetype cluster fit on
   pooled data (cluster the training fold only, per Part 8/11).
3. Team identity crosses matches (the same team appears in multiple
   matches) — a stricter, later-stage held-out-team split (Part 11,
   "team-held-out evaluation later") is the correct tool for asking
   "does this generalize to a team the model has never pressed
   against", distinct from the match-level split above which only
   guarantees no exact-clip leakage.
4. Any threshold tuned on the current single clip (e.g. a future
   PSV_proxy blending weight) must be re-validated once more matches
   exist — tuning on the only available match and calling it validated
   is exactly the "Case B" self-imposed-tuning failure this project's
   own `EvidenceConfig` docstring already warns against.

## 9. Outcome-attribution generalization (this phase's one real code
change to existing logic, additive only)

`pressing_v4.attribute_outcomes` is NOT modified in place (its
`active_start`-gated `events` list stays exactly as-is for the frozen
dashboard/V4 pipeline). Instead, `pressing_features.py` (Part 3) calls
the SAME matching principle (`evidence_quality`, the pass/turnover
window logic) via a new `attribute_outcomes_all_episodes()` wrapper
that runs it against **every** episode (using `termination_time`
instead of requiring `active_start`), producing `heuristic_outcome` for
all 19 real episodes, not zero. This is additive, not a modification of
frozen V4 behavior, and is documented here precisely because it is the
one place this phase's code differs from what the dashboard already
computes.
