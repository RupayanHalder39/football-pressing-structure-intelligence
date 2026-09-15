# Data Leakage Protection Rules

2026-09-08. Strict rules for this research phase, each mapped to where
it is actually enforced in code (not just stated) — so a violation is a
findable bug, not a documentation gap.

| rule | enforcement |
|---|---|
| No random frame-level train/test split across the same match | `models/baselines.py::fit_eval_grouped` only accepts `groups` (match ids) and uses `GroupKFold`/`LeaveOneGroupOut` — there is no random-split code path anywhere in this phase's modules |
| No future feature leakage | `pressing_features.extract_episode_features` anchors every feature at `episode['onset_frame']` only; `models/short_horizon_outcome.py`'s own docstring states the horizon label is built strictly from evidence with `start_frame >= end_frame` (the episode's LAST observed frame), never from frames before that boundary being used as if they were after it |
| No using outcome information in feature computation | `pressing_features.py`'s GEOMETRY/MOTION/PASSING-ESCAPE/FIELD/CONTEXT functions take only `carrier`/`defenders`/`field` — none of them accept or read `heuristic_outcome`, `human_outcome`, or any pass/turnover evidence; outcome attribution (`attribute_outcomes_all_episodes`) is a SEPARATE function called only afterward, on the export script's own separate line |
| No using post-press frames as input to predict the same outcome | Same as above — the horizon label's evidence window starts strictly at `end_frame`, and `end_frame` is itself defined as the episode's `last_observed_frame` (the FSM's own real accounting, not a feature-extraction choice) |
| No tuning thresholds on the test set | `EvidenceConfig`'s thresholds (`enter`/`exit`/`min_observations`/etc.) are frozen dashboard/V4 parameters, explicitly untouched this phase (`RESEARCH_DATASET_PLAN.md` — this phase adds an ADDITIVE outcome-attribution wrapper, never edits `pressing_v4.py`'s own FSM config); any FUTURE PSV_proxy/model hyperparameter must be tuned only within `fit_eval_grouped`'s training folds, never against a held-out fold's own labels — there is no code path in this phase's modules that reads test-fold `y` before scoring |
| Prefer match-grouped split | `models/baselines.py::fit_eval_grouped`, default and only supported mode |
| Prefer leave-one-match-out | `fit_eval_grouped` automatically selects `LeaveOneGroupOut` when `n_groups <= n_splits` (true for any realistic near-term match count) |
| Team-held-out evaluation later | Not implemented yet — correctly gated: `RESEARCH_DATASET_PLAN.md` §8 documents it as a later-stage addition once team identity metadata exists across matches; `Experiment 8` (`EXPERIMENT_PLAN.md`) is where it will first run |

## What this means for today's single-match dataset

Every rule above is enforced in code that ALSO correctly refuses to
run when its precondition isn't met — `fit_eval_grouped` returns
`INSUFFICIENT_MATCH_GROUPS` with exactly one match, rather than
silently falling back to a leakier split. This was verified directly
(`models.baselines.run_on_real_pipeline_output` against the real
`pressing_episode_features.parquet`, `n_matches=1` → honest refusal).
