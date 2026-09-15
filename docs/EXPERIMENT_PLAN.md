# Pressing Research — Experiment Plan

2026-09-08. None of these experiments have been RUN yet in the sense of
producing a reportable result — the framework each depends on is
implemented and unit-tested (Parts 3–9), but every experiment below
needs either more human labels or more matches (usually both) before
its output would be a real, non-degenerate measurement rather than a
demonstration on 19 episodes from one match. Where a real, honestly-
computed number already exists from the current single-match dataset,
it is stated; where it doesn't, that is stated too — never a filled-in
placeholder.

## Experiment 1 — Can geometry distinguish successful vs escaped presses?

- **Hypothesis**: `PressingBaseline`-style geometric features
  (`pressing_features.py`) separate `SUCCESSFUL_PRESSURE_OUTCOMES` from
  `ESCAPE_PROGRESSION_OUTCOMES` (`models.short_horizon_outcome`) better
  than chance.
- **Input**: `pressing_episode_features.parquet` rows +
  `final_adjudicated_outcome` (or `human_outcome`).
- **Output**: binary classifier accuracy / macro F1 / ROC-AUC.
- **Split**: match-grouped (`fit_eval_grouped`), leave-one-match-out
  until ≥5 matches exist.
- **Metric**: macro F1 (accuracy alone is misleading under class
  imbalance — this clip's own heuristic outcomes are already skewed
  16 `UNCERTAIN` / 2 `BALL_REGAIN` / 1 `FORCED_BACKWARD`).
- **Baseline**: Baseline A (nearest-presser distance only).
- **Failure condition**: Baseline D does not beat Baseline A by a
  margin outside the fold-to-fold variance — would mean the richer
  feature set adds no real signal over the trivial one, a genuine and
  reportable negative result, not something to suppress.
- **Required sample size**: with a 9-way (or even the 2-way grouped)
  taxonomy, a bare minimum of ~30 labeled episodes per class is a
  reasonable statistical floor for a first pass — today's real count is
  0 human-labeled, 3 heuristic-labeled non-`UNCERTAIN`. **Not runnable
  today.**

## Experiment 2 — Does the full feature set outperform nearest-distance baselines?

- **Hypothesis**: Baseline D (full feature set) > Baseline C > Baseline
  B > Baseline A on the SAME match-grouped split.
- **Input/Output/Split/Metric**: same as Experiment 1.
- **Baseline**: Baseline A is the reference point for every comparison.
- **Failure condition**: no monotonic improvement A→B→C→D — a real,
  useful finding (identifies which feature group actually matters)
  rather than a reason to keep adding features blindly.
- **Required sample size**: same floor as Experiment 1. **Not runnable
  today** (`models.baselines.run_on_real_pipeline_output` already
  reports `INSUFFICIENT_LABELED_DATA` on the real parquet, verified).

## Experiment 3 — Which features are most associated with restricted progression?

- **Hypothesis**: a subset of `geom_*`/`escape_*`/`field_*` features
  (e.g. `n_within_5m`, `mean_lane_pressure`) has significant marginal
  association with `FORCED_BACKWARD`/`REDUCED_FORWARD_PROGRESSION`-type
  outcomes.
- **Input**: same feature table + outcome.
- **Output**: feature importances (Baseline D's `RandomForestClassifier.
  feature_importances_`, or permutation importance) + univariate
  correlation table.
- **Split**: match-grouped, importances averaged across folds (never
  computed on the full un-split data, which would leak).
- **Metric**: mean feature importance ± across-fold std.
- **Baseline**: none (this is a descriptive/interpretability
  experiment, not a comparison).
- **Failure condition**: importances are unstable across folds (high
  variance relative to mean) — would mean the current sample is too
  small to trust any specific feature ranking, an honest outcome to
  report rather than picking the fold that "looks best."
- **Required sample size**: same floor as Experiment 1, plus enough
  matches (≥3) for fold-to-fold importance stability to be assessable
  at all.

## Experiment 4 — How does performance change by pitch zone?

- **Hypothesis**: detection/outcome performance differs between
  `ctx_pitch_zone_third` values (e.g. presses in the defensive third
  may have different geometry/consequence than the attacking third).
- **Input**: same table, stratified by `ctx_pitch_zone_third`.
- **Output**: per-zone precision/recall/F1 (Experiment 1's metrics,
  computed separately per zone).
- **Split**: match-grouped WITHIN each zone stratum.
- **Metric**: per-zone macro F1 with a confidence interval (bootstrap
  over episodes, respecting match grouping — resample matches, not
  individual episodes, to avoid a within-match bootstrap leak).
- **Baseline**: the pooled (all-zone) Experiment 1 result.
- **Failure condition**: per-zone sample sizes too small for any CI
  narrower than the pooled estimate — report the zone breakdown
  honestly as descriptive only in that case, not as a statistically
  supported claim.
- **Required sample size**: Experiment 1's floor, multiplied by the
  number of zone strata (3) at minimum.

## Experiment 5 — How do pressing structures differ across outcome classes?

- **Hypothesis**: `pressing_archetypes.py`'s structural feature space
  separates by outcome class (e.g. `BALL_REGAIN` presses cluster
  differently from `PRESS_ESCAPED` ones).
- **Input**: `ARCHETYPE_FEATURE_COLUMNS` + outcome label.
- **Output**: cluster-vs-outcome contingency table (chi-square or
  Cramér's V for association strength).
- **Split**: not a train/test experiment — descriptive, but clustering
  is still fit ONLY on a training-fold-equivalent subset if later used
  predictively.
- **Metric**: Cramér's V (association strength, bounded [0,1]).
- **Baseline**: none (descriptive).
- **Failure condition**: `pressing_archetypes.
  MIN_EPISODES_FOR_CLUSTERING` (40) not met — **already the real state
  today (19 episodes)**, verified: `cluster_structures` returns
  `INSUFFICIENT_EPISODES_FOR_CLUSTERING`.
- **Required sample size**: ≥40 episodes (the module's own documented
  floor) before clustering runs at all; more before the resulting
  cluster-vs-outcome association is trustworthy.

## Experiment 6 — Does PSV_proxy correlate with actual short-horizon outcomes?

- **Hypothesis**: `pressing_structure_value.psv_proxy`'s `psv_proxy`
  value correlates with real `observed_progress_cm` (from matched
  evidence) and/or with the binary successful-pressure label.
- **Input**: episodes with BOTH a computed `psv_proxy` AND real matched
  evidence within the horizon (today: 3/19 episodes have any matched
  evidence at all, per the real `heuristic_outcome` distribution).
- **Output**: Pearson/Spearman correlation (`psv_proxy` vs
  `observed_progress_cm`), and point-biserial correlation (`psv_proxy`
  vs the binary label).
- **Split**: none required for a correlation, but should still be
  reported per-match once >1 match exists (a single pooled correlation
  across matches with very different pressing intensity could be
  Simpson's-paradox misleading).
- **Metric**: correlation coefficient + p-value; explicitly flagged
  "descriptive only" below n≈20.
- **Baseline**: a null model (`psv_proxy` replaced by
  `nearest_presser_distance_cm` alone) — does the proxy's extra
  structure (velocity × pressure) actually add correlation over the
  single-distance baseline?
- **Failure condition**: correlation not significantly different from
  0 — would mean `PSV_proxy` in its current form doesn't track real
  outcomes and needs revision before further use, an important,
  reportable negative result.
- **Required sample size**: with only 3 real matched-evidence episodes
  today, this cannot produce a meaningful correlation yet (n=3 is far
  below any reasonable power threshold). **Not runnable today.**

## Experiment 7 — Can counterfactual repositioning improve PSV_proxy?

- **Hypothesis**: `counterfactual_pressing.search_local_repositioning`
  finds a `delta_psv > 0` local candidate for a non-trivial fraction of
  real episodes.
- **Input**: every real episode with a resolvable onset-frame carrier +
  ≥1 presser.
- **Output**: distribution of `best_local_candidate.delta_psv` across
  episodes; fraction with `delta_psv > 0`.
- **Split**: none (this is per-episode, not a trained model).
- **Metric**: mean/median `delta_psv`, fraction improving.
- **Baseline**: `delta_psv` from a RANDOM single-defender perturbation
  (same grid, random choice instead of best) — is the search actually
  finding something better than a random nearby position, or is
  `PSV_proxy` just noisy enough that most positions "improve" it?
- **Failure condition**: the random baseline achieves a similar
  fraction of `delta_psv > 0` as the searched best — would indicate
  `PSV_proxy`'s local sensitivity to single-player position is too
  noisy for the search to be meaningful yet.
- **Required sample size**: this one IS runnable today in a
  demonstration sense (verified: episode `T0-724`, defender 49,
  `dx=-300cm` → `delta_psv=+43.4`) — but reporting it as a general
  finding (not just "it worked once") needs the full 19-episode run at
  minimum, and the random-baseline comparison above before any claim of
  "the search adds value" is defensible.

## Experiment 8 — Generalization across matches / teams

- **Hypothesis**: a model trained on match(es) A generalizes to
  held-out match(es) B (and, later, to a held-out TEAM never seen in
  training).
- **Input**: the full feature table across ≥2 matches.
- **Output**: leave-one-match-out (then leave-one-team-out) evaluation
  metrics (Experiment 1's metrics, per held-out match/team).
- **Split**: strict match-grouped / team-grouped (`RESEARCH_DATASET_PLAN.md`
  §7–8) — this is the experiment those leakage rules exist for.
- **Metric**: per-held-out-match macro F1, plus the spread across
  held-out matches (a model that works on one match and fails on
  another is not "generalizing").
- **Baseline**: within-match performance (Experiment 1's own number) —
  the generalization gap is the difference between the two.
- **Failure condition**: held-out performance collapses toward chance —
  would mean the model has overfit match-specific tracking-quality or
  tactical idiosyncrasies, not learned anything about pressing
  structure generally.
- **Required sample size**: needs a second match to run AT ALL. **Not
  runnable today** — this project has exactly one match
  (`RESEARCH_DATASET_PLAN.md` §2, "required future data" item 1).
