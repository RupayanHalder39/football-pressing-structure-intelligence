# Press Labeling Protocol

2026-09-08. Written to be precise enough that two independent
annotators, working separately with only this document and the
labeling tool (Part 12,
`pressing_structure/tools/press_labeling_tool/`), label the same clip
consistently. Nothing in this protocol is inferred from or biased
toward the pipeline's own heuristic output — an annotator uses the raw
video + tracking overlay, not the FSM's state banner.

## 1. Outcome taxonomy

Exactly one label from this list per episode, chosen by what actually
happened to the ball/possession within the observation window (§3),
not by how "good" the press looked:

| label | definition |
|---|---|
| `BALL_REGAIN` | The pressing team wins the ball (interception, tackle, or the carrying team's pass/touch goes directly to a pressing-team player) within the window. |
| `FORCED_BACKWARD` | The carrying team retains the ball but the next action (pass or dribble) moves the ball toward their own goal, net of at least ~3m depth (matches the pipeline's own `progress <= -300cm` convention — see §5). |
| `FORCED_LATERAL` | The next action stays within ~3m of forward/backward depth (neither clearly forward nor backward) — a sideways pass/dribble under pressure. |
| `FORCED_LONG_CLEARANCE` | The carrying team plays a long, low-control clearance (typically a defender under pressure hoofing the ball forward/away with no clear intended receiver) rather than a controlled pass — distinguish from `REDUCED_FORWARD_PROGRESSION` by the visibly low intent/control, not just distance. |
| `REDUCED_FORWARD_PROGRESSION` | The ball does move forward, but by visibly less than it would have with a clean, unpressured touch (a rushed short pass instead of the killer ball that looked available before the press arrived). Judgment call — see §6 for the tie-break rule against `DANGEROUS_FORWARD_PROGRESSION`. |
| `FORCED_LOW_VALUE_PASS` | The carrying team retains possession via a pass that is safe but tactically unambitious (backward/square to a deep/wide teammate) specifically because the press removed better options — distinct from `FORCED_BACKWARD` (this can be roughly the same depth as a "lateral" pass, the key criterion is "options were removed", not depth alone). |
| `PRESS_ESCAPED` | The carrying team plays through/past the press cleanly — a successful line-breaking pass, turn, or dribble that the pressing shape did not meaningfully affect. |
| `DANGEROUS_FORWARD_PROGRESSION` | The carrying team progresses significantly forward into a genuinely threatening area (final third / behind the press) despite the pressure — the press notably failed. |
| `UNCERTAIN` | Ball out of view, ambiguous action, or the annotator genuinely cannot tell which of the above applies. Never guess to avoid this label. |

**Grouping note (documented explicitly per the research-phase brief,
used only by Part 6's binary target, never silently assumed
elsewhere)**: a candidate "successful pressure" grouping = `{BALL_REGAIN,
FORCED_BACKWARD, FORCED_LATERAL, FORCED_LONG_CLEARANCE,
REDUCED_FORWARD_PROGRESSION, FORCED_LOW_VALUE_PASS}`; the complementary
"escape/progression" grouping = `{PRESS_ESCAPED,
DANGEROUS_FORWARD_PROGRESSION}`; `UNCERTAIN` is excluded from the
binary target entirely (not folded into either side).

## 2. Three-tier outcome recording (never conflated)

- `heuristic_outcome`: whatever the pipeline's automatic evidence
  matching produced (`pressing_features.py`'s
  `attribute_outcomes_all_episodes`, mapped through §5's table into
  this taxonomy). An annotator must NOT be shown this label before
  recording `human_outcome`, to avoid anchoring.
- `human_outcome`: this annotator's independent judgment, from this
  protocol.
- `final_adjudicated_outcome`: filled in later, after ≥2 annotators'
  `human_outcome`s are compared — equal to the agreed value, or a
  third reviewer's decision on disagreement. Left null for a
  single-annotator pass.

## 3. Observation window

- `press_start_frame`: the frame at which the annotator judges
  organized pressure genuinely begins — at least one defender clearly
  closing down the ball carrier with intent, not merely being nearby.
  This is independent of (may differ from) the pipeline's own
  `onset_frame`, which is a threshold-crossing artifact of the
  smoothed intensity score, not a human judgment.
- `press_end_frame`: the frame at which the pressing action's direct
  consequence is decided — the ball leaves the presser's control
  radius via the chosen outcome (a pass is released, a tackle
  completes, the carrier clearly breaks away). Do not extend past the
  action that decides the outcome (e.g. do not include the receiving
  player's subsequent, separate action).
- Outcome is judged from `press_start_frame` to at most
  `press_end_frame + 3s` (matches the pipeline's own evidence-matching
  window, §5) to allow the deciding action's confirmation to land just
  after `press_end_frame`.

## 4. Required fields per label (the tool enforces this)

| field | type | notes |
|---|---|---|
| `press_start_frame` | int | per §3 |
| `press_end_frame` | int | per §3, must be ≥ `press_start_frame` |
| `pressing_team` | `0` or `1` | the team applying the pressure |
| `ball_carrier_team` | `0` or `1` | must be the OTHER team; a same-team value is a data-entry error the tool rejects |
| `pressing_players` | list[track_id] | every defender the annotator judges as actively part of the pressing action (not merely nearby) |
| `primary_presser` | track_id or null | the single player who most directly applies the deciding pressure (closest/most impactful) — null if genuinely ambiguous (e.g. simultaneous double-team), never guessed |
| `support_pressers` | list[track_id] | `pressing_players` minus `primary_presser` |
| `outcome` | one of §1's 9 labels | — |
| `confidence` | `HIGH` / `MEDIUM` / `LOW` | the annotator's own certainty in `outcome` (not the pipeline's `confidence` field — a separate, human concept) |
| `notes` | free text, optional | anything that doesn't fit the fixed fields (e.g. "possible offside on the eventual pass", "ball briefly off-screen at frame 1751") |

## 5. Heuristic-to-taxonomy mapping (for `heuristic_outcome` only —
never used to auto-fill `human_outcome`)

The pipeline's existing evidence-matching (`pressing_v4.
attribute_outcomes`'s pass/turnover window logic, generalized in
`pressing_features.py` to run on every episode, not just
`ACTIVE_PRESS`-confirmed ones — see `RESEARCH_DATASET_PLAN.md` §9) only
distinguishes a coarser set: `BALL_REGAIN`, `BACKWARD_ACTION`
(`progress <= -300cm`), `LATERAL_ACTION` (`-300 < progress < 300cm`),
`FORWARD_PROGRESSION` (`progress >= 300cm`), `UNCERTAIN`. It cannot
currently distinguish `FORCED_LONG_CLEARANCE` /
`REDUCED_FORWARD_PROGRESSION` / `FORCED_LOW_VALUE_PASS` /
`PRESS_ESCAPED` / `DANGEROUS_FORWARD_PROGRESSION` from each other
(those require judging *intent* and *quality*, not just displacement) —
mapped conservatively:

| pipeline label | mapped `heuristic_outcome` |
|---|---|
| `BALL_REGAIN` | `BALL_REGAIN` |
| `BACKWARD_ACTION` | `FORCED_BACKWARD` |
| `LATERAL_ACTION` | `FORCED_LATERAL` |
| `FORWARD_PROGRESSION` | `PRESS_ESCAPED` if `progress >= 300cm`, else n/a (this branch always has `progress>=300` by construction) — **documented limitation: the heuristic cannot distinguish a genuine escape from a merely-adequate forward pass; both map to `PRESS_ESCAPED` today.** `DANGEROUS_FORWARD_PROGRESSION` is never auto-assigned — it requires a human judgment of "genuinely threatening", not just a displacement threshold. |
| `UNCERTAIN` | `UNCERTAIN` |

This is exactly why `human_outcome` exists as a separate tier — the
finer taxonomy (§1) is designed to be human-judged, not automatable
from displacement alone with current data.

## 6. Tie-break rules (for annotator consistency)

- `REDUCED_FORWARD_PROGRESSION` vs `DANGEROUS_FORWARD_PROGRESSION`: if
  the ball ends the window inside the final third AND with a clear
  path toward goal, label `DANGEROUS_FORWARD_PROGRESSION`. Otherwise,
  if forward but not clearly threatening, `REDUCED_FORWARD_PROGRESSION`.
- `FORCED_LOW_VALUE_PASS` vs `FORCED_LATERAL`: if the pass is
  materially backward-and-sideways to a deep, low-threat option purely
  because better options were pressed out, prefer
  `FORCED_LOW_VALUE_PASS`; if it's a simple sideways ball with no clear
  sign options were removed, prefer `FORCED_LATERAL`.
- `FORCED_LONG_CLEARANCE` vs `FORCED_BACKWARD`: a hoofed clearance that
  travels FORWARD (typical defensive clearance direction) is still
  `FORCED_LONG_CLEARANCE`, not `FORCED_BACKWARD` — the label is about
  the low-control/desperation nature of the action, not its direction.
- When two labels seem equally defensible after applying the above:
  choose the more conservative one for the pressing team (i.e., don't
  over-credit the press) and note the ambiguity in `notes`.

## 7. Inter-annotator reliability

Once ≥2 annotators have labeled overlapping episodes: report Cohen's
κ on `outcome` and on `pressing_team`/`primary_presser` agreement in
`pressing_structure/evaluation/evaluate_outcomes.py`'s output. No κ has
been computed yet (single-annotator state today) — reported honestly as
"awaiting a second annotator" wherever this would otherwise be claimed.
