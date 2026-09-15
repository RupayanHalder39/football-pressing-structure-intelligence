# Pressing Structure Value (PSV) — Mathematical Definition

2026-09-08. **`PSV_proxy` is a temporary, disclosed progression-based
stand-in, not the final PSV.** This project has no xT/EPV (expected
threat / expected possession value) model implemented anywhere in this
codebase — building or validating one is future work, not assumed to
exist. Do not cite `PSV_proxy` as a validated measure of pressing
quality; Experiment 6 (`EXPERIMENT_PLAN.md`) is the not-yet-run test of
whether it even correlates with real outcomes.

## Target concept

> How much the pressing structure reduces the opponent's expected
> attacking value / progression over a short horizon.

Formally, with a full value model `V` (e.g. xT):

```
PSV_t = ExpectedOpponentValue_without_effective_pressure(t, horizon)
      - ObservedOrPredictedOpponentValue_under_current_pressure(t, horizon)
```

Neither term is computable today without `V` — so this project
implements the **progression-based proxy** instead.

## `PSV_proxy` — current implementation

```
PSV_proxy_t = expected_forward_progress_without_pressure(t, horizon)
            - observed_or_predicted_forward_progress_under_pressure(t, horizon)
```

`pressing_structure/analytics/pressing_structure_value.py`'s exact,
disclosed formula for the two terms (a simple constant-velocity
physical model, NOT a trained model, NOT claimed optimal):

1. `carrier_forward_velocity_cm_s` — the ball carrier's REAL tracked
   velocity, projected onto their OWN attacking direction
   (`pitch.attacking_sign`). `0.0` (never fabricated) when motion is
   untrustworthy (`motion_valid=False`), the same convention
   `pressure_field.player_influence` already uses elsewhere.
2. `expected_forward_progress_without_pressure` =
   `carrier_forward_velocity_cm_s * horizon_sec` — a naive,
   zero-defensive-influence projection: "how far would this carrier be
   expected to move forward in this horizon if defensive pressure did
   not exist at all". This is deliberately the SIMPLEST possible
   baseline for this term (constant velocity, no acceleration, no
   decision-making model) — a stand-in until a real counterfactual
   model (Part 9) or an xT model replaces it.
3. `predicted_forward_progress_under_pressure` = the SAME projection,
   damped by `pressure_at_carrier` (the REAL, already-computed
   `team_pressure_field` value sampled at the carrier's exact position
   — the identical field the dashboard washes, never a second model):
   `expected_forward_progress_without_pressure * (1 - pressure_at_carrier)`.
4. `PSV_proxy_t = expected_forward_progress_without_pressure -
   predicted_forward_progress_under_pressure`, which algebraically
   simplifies to `expected_forward_progress_without_pressure *
   pressure_at_carrier` — i.e. **the proxy is exactly "how much of the
   carrier's own naive forward progress this frame's real, measured
   local pressure is expected to suppress."**

Units: cm of forward progress suppressed over the given horizon.
Always ≥ 0 when `carrier_forward_velocity_cm_s ≥ 0` (a carrier already
moving backward has `PSV_proxy = 0` by this formula, not negative —
documented limitation, §5).

## Real observed evidence (validation signal, not a formula input)

When real matched pass/turnover evidence exists within the horizon
(`models.short_horizon_outcome`), the REAL observed
`forward_progress_cm` (from `pitch.progress`, the same function
`pressing_v4.attribute_outcomes` already uses) is reported ALONGSIDE
`PSV_proxy` as `observed_progress_cm` — never substituted into the
formula itself (mixing a model term and an observed term inconsistently
inside one number would make it uninterpretable). Comparing
`PSV_proxy` against `observed_progress_cm` across episodes is exactly
Experiment 6.

## Components considered for a future, fuller proxy

Not implemented yet (documented here as the acknowledged next steps,
not silently dropped): final-third entry, line-breaking pass,
possession retention, dangerous-space entry, regain (these map more
naturally onto the outcome TAXONOMY's finer labels — §Part 2 — than
onto a single scalar; a future version could weight `PSV_proxy` by
which taxonomy label the episode's outcome falls into, once enough
human-labeled data exists to fit those weights honestly rather than
guess them).

## Known limitations (disclosed, not hidden)

1. **No xT/EPV** — the proxy only measures raw forward progress, not
   the VALUE of that progress (a suppressed pass into a crowded
   midfield "counts" the same as one into a dangerous space).
2. **Constant-velocity assumption** — real carriers accelerate,
   decelerate, and change direction; this formula ignores all of that.
3. **Single-frame anchor** — computed at `onset_frame` only, not
   integrated over the episode's full duration.
4. **Non-negative by construction** — cannot represent a press that
   makes things WORSE for the defending team (e.g. panics a defender
   into an even worse error) as a negative value under the current
   formula; only "how much forward progress was suppressed."
5. **Unvalidated** — Experiment 6 has not been run (requires enough
   real matched-evidence episodes to correlate against; today's clip
   has 3/19 with matched evidence within a horizon).
