# Roles and effort

**Role** follows how the work runs: who stays in the loop, how long the horizon, whether the job is building or proving. **Effort** follows how hard the model must think inside that role. A hard but well-specified task is often `balanced` at `high` effort: difficulty raises effort before it raises role.

## Roles

- **fast**: deterministic, localized, clear, low-risk work.
- **balanced**: the default for ordinary engineering (a normal feature in any layer, a moderate refactor, normal tests).
- **frontier-interactive**: hard work with a human in the loop (requirement discovery, architecture discussion, interactive debugging, complex implementation).
- **frontier-verifier**: adversarial review and strong verification (critical review, concurrency, idempotency, messaging, security, financial logic, hidden edge cases).
- **deep-autonomous**: long independent investigation (root cause, broad codebase exploration, multi-system investigation, long-horizon debugging).

## Effort

- **low**: small scope, clear requirements, low risk; fast iteration beats exploration.
- **medium**: the default (normal feature, normal TDD, moderate integration, well-specified business rules).
- **high**: edge cases or correctness matter; concurrency, messaging, legacy code, a hard bug, a high-risk feature, adversarial review.
- **max**: reserved for broad autonomous investigation, critical security analysis, extreme debugging, architecture-wide autonomous reasoning. Importance alone earns a `high` review, not `max`.

## Implementation

Role, first match wins:

1. `nature_of_work` is security-review: the work itself is a review, so **frontier-verifier**, with the review's effort rule below.
2. `nature_of_work` is investigation or a bugfix with unknown root cause, `autonomy` is autonomous or higher, and `scope` is cross-module or wider: **deep-autonomous**.
3. `complexity` is extreme; or `autonomy` is interactive with `complexity` high or `nature_of_work` architecture or requirement-refinement: **frontier-interactive**.
4. `complexity`, `hidden_edge_cases` and `cost_of_failure` all low, `scope` localized or multi-file: **fast**.
5. Otherwise: **balanced**.

Effort: start at `medium`. Drop to `low` when rule 4 matched. Raise to `high` for `complexity` high, legacy code, or a hard bug; `hidden_edge_cases` critical raises it only together with `risk.impact` high or critical. Edge cases on their own are the review's job: implementation stays at `medium` and the review goes to frontier-verifier. Raise to `max` only for the cases listed under Effort.

## Review

Chosen from `verification_requirement`, `hidden_edge_cases` and `cost_of_failure`, independently of the implementation choice. The review runs in a session separate from the implementation, so it verifies instead of agreeing.

Role:

- **frontier-verifier** when any of the three is high or critical, or `nature_of_work` is security-review.
- **fast** when all three are low.
- **balanced** otherwise.

Effort: `high` for frontier-verifier, `medium` for balanced, `low` for fast. Raise to `max` only for critical security analysis (`verification_requirement` critical with security, authorization or money at stake).

## Effort control

`effort_control_requirement` says whether a phase needs the model to apply its effort through an explicit reasoning control. It is decided per phase, after role and effort and apart from them: no effort level implies a requirement, so a `balanced` implementation at `high` effort stays `preferred`.

- **none**: the provider's default reasoning serves the work; a missing control changes nothing.
- **preferred**: the requested effort helps, and a shallow run would still be caught later (the review, a human in the loop). A model without the control stays eligible; between candidates of equal fitness, the one that honors the requested effort wins.
- **required**: the phase is the last check on correctness, so a shallow run would ship unnoticed. A model that cannot apply the requested effort is ineligible, unless accepted measured evidence clears its default ([`RESOLVER.md`](RESOLVER.md), Effort).

Per phase, first match wins:

1. **required**: the phase verifies (the review, or an implementation whose `nature_of_work` is security-review), and `verification_requirement` or `cost_of_failure` is high or critical.
2. **none**: the phase's role is `fast`: clear, localized, low-risk work.
3. **preferred**: otherwise.
