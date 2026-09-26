# Resolver

Turns each phase's `model_role` and `effort` into a provider, a model and a reasoning setting. Model names are configuration, in [`registry/models.yaml`](registry/models.yaml); model capabilities are domain logic, here.

Fitness comes only from the registry's evidence fields. What a model was used for before, in any project, conversation or stack, is history, not evidence.

## Roles in the registry

```yaml
roles:
  <role>:
    incumbent: <provider>/<model-id>   # holds the role on ties; null until evidence decides
    candidates:
      - model: <provider>/<model-id>
        evaluation_status: unevaluated # unevaluated | challenger | evaluating | evaluated | insufficient-evidence | stale
        fitness:
          global:                      # required
            source: official-docs      # official-docs | benchmark:<name> | eval-pack:<pack>@<version> | third-party
            confidence: medium         # low | medium | high
            measured: {}               # written by the evaluator, see Measured evidence
          category_specific: {}        # <category>: same shape as global
          optional_specialization: {} # <pack>: same shape as global
        user_preference: null          # { preferred: true }, set only by an explicit user instruction
```

A model listed under a role is claimed fit for it. `evaluated` means a benchmark or eval pack tested this model on this role; a claim derived from official docs is `unevaluated`. The other states belong to the evaluator ([`EVALUATOR.md`](EVALUATOR.md), States) and change nothing here: eligibility and ranking read the confidences. Appearing in an API proves a model exists, not that it fits.

`confidence` grades each claim:

- **low**: rests on the model's presence, its name, third-party information, or a refresh placing it automatically.
- **medium**: official docs position the model for this kind of work.
- **high**: a benchmark or eval pack confirms it; `evaluated` only.

`user_preference` is configuration, not evidence: only an explicit user instruction sets it; past use, projects, conversations, stack, and a choice made to settle one tie never do. It breaks ties after the evidence.

## Evidence

Three layers:

- **global**: generic engineering evidence for the role, on any stack. Required for every candidate.
- **category_specific**: evidence for one task category. Refines global for phases of that category.
- **optional_specialization**: evidence from one specialization pack (backend, frontend, a language, a framework). Breaks ties, and only for issues in that specialization.

The routing policy works identically with only the global layer, and decides with global plus category_specific: category evidence sets eligibility, feeds quality-first, and ranks candidates when a role has no eligible incumbent. Evidence from one project's own issues stays in that project's specialization entry and never reaches `global` or `category_specific`. Where each layer comes from: [`eval-packs/README.md`](eval-packs/README.md).

Task categories: requirement-discovery, feature-implementation, bugfix, code-review, database-reasoning, distributed-systems, architecture, refactoring, security-review, performance, migration, testing, long-horizon-investigation.

A phase's **primary category**:

| Phase | `nature_of_work` | Primary category |
|---|---|---|
| implementation | implement, tdd | feature-implementation |
| implementation | bugfix | bugfix |
| implementation | investigation | long-horizon-investigation |
| implementation | refactor | refactoring |
| implementation | architecture | architecture |
| implementation | migration | migration |
| implementation | security-review | security-review |
| implementation | performance | performance |
| implementation | requirement-refinement | requirement-discovery |
| implementation | testing | testing |
| implementation | documentation, configuration | none: global only |
| review | security-review | security-review |
| review | any other | code-review |

**Secondary categories** follow the rubric's edge-case signals: database-reasoning for persistence and migrations; distributed-systems for concurrency, ordering, retries, duplicate messages, async processing and distributed systems.

A layer compares candidates only when every candidate being compared has an entry in it, as with speed. The phase's **ranking confidence** is its primary-category confidence when that layer compares, otherwise `global`.

## Measured evidence

An evaluation writes `measured` entries into a layer, one per conceptual effort it ran:

```yaml
global:
  source: official-docs
  confidence: medium
  measured:
    high: { suite: generic@2, provider_value: high, cases: 3, repeats: 3, score: 0.84, gates: pass, confidence: high,
            cost_per_case_usd: 0.42, latency_p50_s: 51, evaluated_at: 2026-10-02 }
```

At the phase's requested effort, a measured entry with confidence medium or high replaces the layer's claim. When its `score` reaches `evaluation.role_bar` and its `gates` passed, the candidate's confidence in that layer becomes the entry's confidence; below the bar or with a failed gate, the candidate leaves the eligible set, since the evidence says it does not fit. A low-confidence entry, or one measured at another effort, leaves the claim as it was. An entry older than `evaluation.evidence_max_age_days` counts at most medium.

## Role profiles

The capabilities that define each role, in order of weight:

- **fast**: `speed`, `coding`.
- **balanced**: `coding`, `agentic`.
- **frontier-interactive**: `reasoning`, `interaction`.
- **frontier-verifier**: `verification`, `reasoning`.
- **deep-autonomous**: `autonomy`, `agentic`, `long_context`.

Used by `quality-first` and when judging a model for a role. `speed` ranks by the Speed rule below.

## Picking a candidate

Eligible: the model's `lifecycle.status` is `active`; its global confidence, and its primary-category confidence when it has one, are medium or high, or it is the incumbent; and it meets the phase's `effort_control_requirement` (see Effort). A provider named for a phase, by the user or the issue, narrows that phase to it. An empty eligible set leaves the phase `unresolved`, with the reason.

`routing_strategy` (default `best-fit`; the user may name another):

- **best-fit**: the incumbent, while it is eligible; only a promotion ([`EVALUATOR.md`](EVALUATOR.md)) moves it. Otherwise, the highest ranking confidence.
- **quality-first**: the highest measured `score` at the requested effort when every candidate has one from the same suite in the ranking layer; otherwise the highest on the role profile, compared capability by capability in order.
- **cost-aware**: the lowest measured `cost_per_case_usd` in `global` at the requested effort when every candidate has one from the same suite; otherwise the lowest `relative_cost` when every candidate has one; otherwise best-fit, with `cost_evidence: insufficient` on the phase.
- **speed-first**: the fastest by the Speed rule; without comparable evidence, best-fit.

Ties go, in order, to: the higher ranking confidence; the higher secondary-category confidence (the weakest secondary category counts); the higher confidence for the issue's specialization (a technology or domain the issue or its repository names, matching a pack); when the phase's `effort_control_requirement` is `preferred`, the candidate that honors the requested effort; the user's preference; the incumbent; the lower `relative_cost`. A criterion compares candidates only when every one of them has a value for it; otherwise it is skipped, so an unknown value never counts as worse. `user_preference: null` is a value, not preferred, so that criterion decides as soon as any tied candidate is preferred. The phase records what decided in `tie_broken_by`.

A tie that survives every rule is the user's decision; registry order and provider carry no weight. Ask which tied candidate to use. The answer is a tie resolution scoped to this execution: it serves this run only (every issue in the batch that meets the same tie), the phase records `tie_resolution: { source: user-selection, scope: execution }` in place of `tie_broken_by`, and the registry does not change: `user_preference` stays as it was, null by default, and the incumbent stays null. The next run asks again. A persistent preference exists only when the user asks for one in so many words ("on ties, prefer <provider>"): the matching candidates, in the roles the instruction names or in every role when it names none, get `user_preference: { preferred: true }`. With no user to ask, the phase is `unresolved` with the reason `unbroken-tie: <models>`.

A picked model under `lifecycle_watch` stays eligible; the output warns `lifecycle-watch:<model>`. When `evaluation.queue` holds an entry for a role this run resolves, the output warns `evaluation-recommended:<role>`; the evaluation itself runs only on request ([`EVALUATOR.md`](EVALUATOR.md)).

## Speed

Speed ranks only on comparable evidence that every candidate being ranked shares. Preferred sources, in order: measured `latency_p50_s` in `global` at the requested effort, from one suite; the model's `speed.latency_p50_s` from one `benchmark:<name>`; the same from `official-quantitative`. The first source all candidates share wins. Qualitative labels (fastest, moderate) stay descriptive, since providers grade them on different scales.

Without a shared source, record `speed_evidence: insufficient` on the phase: speed-first falls back to best-fit, and a role profile skips `speed` and compares the next capability.

## Effort

The phase's `effort` is the **requested effort**; the model's `supported_effort` gives the **provider reasoning control** for it. The output keeps both, as `requested_effort` and `resolved_reasoning`:

- Level supported: `provider_effort` is the native value, `status: supported`.
- Level unsupported (`supported_effort: none`, or the level missing from the map): `provider_effort: null`, `status: unsupported`, and the model runs at its provider default.

Whether an unsupported level matters is decided by the phase's `effort_control_requirement` ([`ROLES.md`](ROLES.md), Effort control), never by the effort level:

- **none**: eligibility and ranking ignore the control.
- **preferred**: the model stays eligible; among candidates tied on the evidence, the one that honors the requested effort wins (Picking a candidate).
- **required**: the model leaves the eligible set for this phase, the pick moves to the next candidate, and the output warns `effort-unsupported:<model>`. The exception is accepted measured evidence: an entry at the requested effort, measured at the model's default configuration, that counts under Measured evidence (confidence medium or high, score at `evaluation.role_bar` or above, gates passed) keeps the model eligible at its default, since a missing control says nothing about its reasoning.

The requested level is sent as asked or not at all; it is never swapped for another level.

## Implementation and review

Resolve implementation first, then review, each on its own: providers may differ, so an efficient model can implement and a stronger one verify.

Model diversity is a tie-breaker, not a requirement. When review lands on the implementation's model, or ties with it, prefer an independent review model whose fitness is comparable (same ranking confidence, equal on the role profile) and mark the review `selected_for_diversity: true`; the user's preference, when set, still wins. A materially worse verifier is never chosen for diversity alone: when the same model is clearly the best fit, it serves both phases. The registry's `diversity_bonus` holds measured evidence on cross-model review; the tie-break applies unless its status is `hurts`.

## Replacing an incumbent

New work goes to an `active` model whenever the role has one. When the incumbent is not `active`:

1. The best-fit pick among the eligible candidates becomes incumbent; ties go to the old incumbent's documented `replacement`, then to the tie rules above.
2. No eligible candidate: the documented `replacement`, if `active` in the registry, even at low confidence, with the warning `low-confidence-fallback`.
3. Nothing active: a `deprecated` incumbent still serves, with the warning `deprecated-incumbent`; a `retired` or `unsupported` one leaves the phase `unresolved`, with the reason.

A refresh writes the new incumbent into the registry; a run that could not refresh applies the same rule without writing it. An incumbent stays `null` while evidence leaves its role tied.
