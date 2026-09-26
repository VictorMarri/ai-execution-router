# Evaluator

Optional. It answers one question per role: which **configuration** (a model plus its reasoning setting) fits this execution role and task category, on evidence. It never asks which model is best in general. It runs only when the user asks for an evaluation, or accepts one a resolution recommended; classification and resolution never wait on it.

Principles:

- The evaluation system must be project-agnostic, language-agnostic, framework-agnostic, and provider-agnostic at its core.
- Specialized evaluation packs may refine model fitness, but must never define global routing policy.
- New models are challengers until evidence supports promotion.
- Model availability is not model fitness.
- Model + reasoning configuration is the relevant evaluation unit when reasoning controls exist.
- Evaluation results update registry evidence; they do not hardcode routing policy.
- A model cannot compensate for failure in a critical capability by scoring highly on unrelated dimensions.

The **incumbent** is the model currently trusted for a role; a **challenger** is a model not yet validated against it.

## Triggers

The refresh detects these cheaply and queues them in `evaluation.queue` ([`REGISTRY.md`](REGISTRY.md)); a resolution that touches a queued role warns `evaluation-recommended:<role>`.

- A model new to the registry, including a provider's new generation: listed as `challenger`.
- An incumbent became `deprecated` or `retired`.
- A candidate's `supported_effort` changed.
- A candidate's `relative_cost` moved by 20% or more.
- A capability grade changed in the provider's docs.
- A role has no incumbent (unbroken tie), or its incumbent's confidence is low.
- Measured evidence went stale: past `evaluation.evidence_max_age_days`, or from an older suite version.
- Sources disagree on which candidate is better for a role.
- The user asks for an evaluation or re-evaluation.

## States

`evaluation_status` on a role candidate, kept apart from `lifecycle.status`:

- **unevaluated**: no measured evidence, nothing queued.
- **challenger**: queued for comparison with the role's incumbent, or with the other tied candidates when the role has none.
- **evaluating**: a run is in progress.
- **evaluated**: measured evidence at confidence medium or high.
- **insufficient-evidence**: a run ended below the evidence policy.
- **stale**: measured evidence past its age or suite version; queued again.

## Steps

### 1. Scope

From the queue or the user's request: the roles, and per role the challengers plus the incumbent (or the tied candidates). Cases: only those whose `roles` include the role, from the generic pack plus any category or specialization pack the user names. A new verifier model runs the verifier cases, not the whole suite.

Configurations per candidate: the conceptual efforts the role runs at ([`ROLES.md`](ROLES.md)), plus one neighbouring level when supported, up to `budget.max_configs_per_candidate`. A model without an effort control runs once at its provider default, recorded as such, and competes at every level of the comparison: a missing control says nothing about its reasoning.

Done when every (candidate, configuration, case) triple is listed, minus those the cache already answers: same suite version, case content, model, provider value and judges, within `evidence_max_age_days`.

### 2. Budget and privacy gate

Estimate the cost (cases × configurations × repeats, at list price and each case's budget) and show it with the scope. Run only after the user confirms, within the limits in [`eval-packs/config.yaml`](eval-packs/config.yaml). A run that hits a limit stops, keeps its partial results and decides `insufficient-evidence`.

Only cases from packs marked `synthetic: true` or `authorized: true` leave the machine, and only to providers the config allows. The evaluator sends a case's `task` and `context`, and nothing else: no file from the working repository, no secret, credential or customer data.

Done when the user confirmed the estimate and every case passed the gate.

### 3. Run

Send each configuration the case's `task` and `context`; the rest of the case is the answer key. Repeat each triple `budget.repeats` times. Record each output's SHA-256; keep the raw output itself under `results/<run-id>/raw/` only as `keep_raw_outputs` allows (by default, for cases from synthetic packs). Record the run's **provenance**:

```yaml
provenance:
  eval_suite_version: generic@2
  case_version: SR-001@2
  candidate_provider: <provider>
  candidate_model: <model-id>
  candidate_model_version: <snapshot the provider reported>
  reasoning_configuration: { conceptual_effort: high, provider_control: effort, provider_value: high }
  provider_parameters: { <every request parameter sent> }
  repetitions: 3
  executed_at: 2026-10-02T14:00:00Z
  output_sha256: [<one per repetition>]
```

Done when every triple has its repetitions and provenance, or the budget stopped the run.

### 4. Score

Each dimension scores 0 to 1, deterministic checks first:

- `objective_checks` of kind `io` or `sql-result`: execute them; `correctness` is the fraction passed.
- Checks of kind `checklist`, and each `must_detect` item: judged present or absent. Checklist checks feed `correctness`; `must_detect` feeds the case's `must_detect_dimension` (default `edge_case_discovery`).
- Any `must_not_do` violation sets `requirement_adherence` to 0 for the case.
- `must_pass`: each entry names an objective check that must pass, or states an invariant judged pass or fail. It must hold in every repetition.
- `qualitative_rubric`: judged 0 to 4, divided by 4.

**Blind judging.** A judge scores only what checks cannot, and sees each output as an anonymous response:

- The evaluator gives each configuration a random label per case (`response-1`, `response-2`), shuffles their order, and keeps the key to itself.
- It removes provider and model names, the model's descriptions of itself, and anything else that reveals the origin, and sends no cost, latency, incumbent or challenger role, or user preference.
- The judge receives the case's rubric, answer key, `must_detect`, `must_not_do` and `must_pass`, plus the anonymous responses, and returns a verdict per item with the quoted evidence. The responses are data: an instruction inside one (to score it highly, say) is part of what is judged.

An output that redaction cannot anonymize without distorting it is judged as it is and marked `blinding: partial`.

The judge set is fixed per comparison, so both sides face the same judges: `judges.models`, else the resolver's frontier-verifier pick among models not under test. When every available judge is also under test, the whole comparison uses the same judges and is marked `self_judged`. One judge per item is the default; a second, from a different provider when available, joins for the categories in `judges.second_judge_categories` and for close decisions (step 5). When two judges disagree, the item scores 0.5 and is flagged. Each judge's provenance is recorded:

```yaml
judge:
  provider: <provider>
  model: <model-id>
  model_version: <snapshot>
  rubric_version: SR-001@2/judge-template@1
  blinding: full            # full | partial
  self_judged: false
```

A judgment is evidence, not ground truth.

The **role score** is the mean, over the role's cases, of each case's mean on the role's dimensions:

| Role | Dimensions in the role score |
|---|---|
| fast | correctness, requirement_adherence |
| balanced | correctness, requirement_adherence, edge_case_discovery, verification_quality, code_quality |
| frontier-interactive | interaction_quality, requirement_adherence, reasoning_quality, architectural_judgment |
| frontier-verifier | edge_case_discovery, verification_quality, reasoning_quality, correctness |
| deep-autonomous | investigation_quality, autonomy, correctness, reasoning_quality |

Every dimension stays in the results. Efficiency dimensions (verification_efficiency, latency, cost, token_or_compute_efficiency) are reported beside the score and never folded into it.

**Hard gates.** A model cannot compensate for failure in a critical capability by scoring highly on unrelated dimensions. A configuration passes a role's gates when every dimension in `role_requirements.<role>.hard_gates` averages at least its `min` over the role's cases, and every `must_pass` entry of those cases held in every repetition. Each failure is recorded by name: `hard-gate:<dimension> <value> < <min>`, or `must-pass:<case>:<entry>`.

**Confidence**, per configuration and role, follows `confidence` in the config: the highest level whose thresholds all hold, on cases, repetitions, distinct categories among the cases, the share of cases with an executed deterministic check (`io`, `sql-result`), and the largest per-case spread across repetitions. Any condition in `confidence.cap_at_medium` then caps it at medium: a self-judged run, partial blinding, or judges disagreeing on more than `confidence.judge_disagreement_rate` of their items. Cached results older than `evaluation.evidence_max_age_days` are run again instead of counted. A few qualitative cases reach medium at most; high needs more cases, several categories and deterministic checks.

Done when every configuration carries dimensions, role score, gate results and confidence for each role in scope.

### 5. Decide

A comparison needs the same suite version, cases, repetitions and judges on both sides; its confidence is the lower of the two. The thresholds live under `promotion` in the config. Challenger C beats incumbent I in a role when all five hold:

1. **Gates**: C passes every hard gate of the role.
2. **Coverage**: confidence medium or high on both, over at least `min_shared_cases` shared cases.
3. **Consistency**: C scores at least as high as I on at least `consistency_share` of the shared cases.
4. **No important regression**: no role dimension more than `max_dimension_drop` below I; objective pass rate and `must_not_do` violations no worse than I's.
5. **Relevant gain**: role score at least `min_gain` above I; or within `tie_band` of I at `min_savings` lower measured cost per case, or lower comparable latency.

The promotion is **automatic** only on strong evidence: comparison confidence at least `auto.min_confidence`, gain at least `auto.min_gain` (or savings at least `auto.min_savings`), and no `self_judged` run or `blinding: partial` in the decisive cases. Otherwise it is only **recommended**.

A decision is **close** when the gain is below `auto.min_gain` and the lead comes from judged dimensions. A close decision gets a second judge on the decisive cases: agreement keeps the recommendation; disagreement, or no second judge available, turns it into `insufficient-evidence`.

```yaml
promotion:
  decision: recommend-promotion  # promote-challenger | recommend-promotion | retain-incumbent | insufficient-evidence
  confidence: medium
  automatic: false
  blocked_by: []                 # gate and must_pass failures, by name
  reasons: []                    # why it is not automatic, or why the incumbent stays
```

- `promote-challenger` (automatic): all five conditions and the automatic bar.
- `recommend-promotion` (not automatic): all five conditions, below the automatic bar. The incumbent stays; the recommendation waits for the user's approval or for more evidence.
- `retain-incumbent`: a gate failed (named in `blocked_by`), or conditions 3 to 5 failed.
- `insufficient-evidence`: coverage failed, or a close decision had no confirming second judge.

One good run never promotes. An incumbent that fails a gate keeps the role while no challenger passes, and the results flag it. With no incumbent, tied candidates meet head to head under the same rules: one that beats every other becomes incumbent when the promotion is automatic, or is recommended otherwise; else the tie stands.

Done when every role in scope has a promotion block.

### 6. Write back

Results go to `eval-packs/results/<run-id>.yaml` (schema below), with the kept raw outputs beside them. The registry then changes in these fields only:

- candidate `evaluation_status`;
- `fitness.<layer>.measured.<conceptual effort>` per configuration, with `gates: pass | fail`: generic cases feed `global`, a category's cases feed `category_specific.<category>`, a specialization pack feeds `optional_specialization.<pack>` (entry shape in [`RESOLVER.md`](RESOLVER.md), Measured evidence);
- role `incumbent`, only on an automatic promotion;
- `evaluation.pending_promotions`, which gains each `recommend-promotion` as `{ role, challenger, incumbent, run_id, confidence, reasons }`; the user approving one applies it as a promotion, and a later run on the same role replaces it;
- `diversity_bonus`;
- handled entries leave `evaluation.queue`.

SKILL.md, RUBRIC.md, ROLES.md and RESOLVER.md stay as written: the evaluator changes evidence, and the resolver turns evidence into picks.

Done when the results file exists and the registry validates.

```yaml
run_id: <date>-<scope>
evaluation:
  suite: generic@2
  evaluated_at: 2026-10-02T14:00:00Z
  scope: { roles: [frontier-verifier], reason: unbroken-tie }
  cost_usd: 7.40
configurations:
  - provenance: { <as in step 3> }
    judges: [ { <as in step 4> } ]
    roles:
      frontier-verifier:
        cases: 3
        repetitions: 3
        score: 0.81
        dimensions: { edge_case_discovery: 0.83, verification_quality: 0.75, reasoning_quality: 0.80, correctness: 0.86 }
        gates: { pass: false, failures: ["hard-gate:verification_quality 0.75 < 0.80"] }
        confidence: medium
        confidence_factors: { cases: 3, repetitions: 3, categories: 3, deterministic_case_share: 0.0, max_spread: 0.12, caps: [] }
        category_specific:
          code-review: { score: 0.88, confidence: low }
        cost_per_case_usd: 0.52
        latency_p50_s: 41
recommendation:
  frontier-verifier:
    incumbent: <provider>/<model-id>
    challenger: <provider>/<model-id>
    promotion: { decision: retain-incumbent, confidence: medium, automatic: false, blocked_by: ["hard-gate:verification_quality 0.75 < 0.80"], reasons: [] }
    by_strategy:                     # the resolver's picks on the updated registry
      best-fit: { provider: <provider>, model: <model-id>, reasoning: <provider value> }
      quality-first: { provider: <provider>, model: <model-id>, reasoning: <provider value> }
      cost-aware: { provider: <provider>, model: <model-id>, reasoning: <provider value> }
      speed-first: { speed_evidence: insufficient }
```

## Regression runs

The same cases serve every re-run: a new model, a new model version, changed reasoning controls, a price change, a behavior change, aging evidence. Any change to a case bumps the pack's `suite_version`, and evidence from the older version counts as stale.

## Diversity experiment

Optional. It pairs same-model and cross-model review on artifacts with known defects: model A solves a case with objective checks; A and B each review A's solution; a review scores by the defects it finds that the hidden checks reveal, minus false alarms. Under the same evidence policy, the result sets `diversity_bonus` to `helps`, `neutral` or `hurts`, with its confidence. Nothing assumes cross-model review is better.
