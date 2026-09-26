# Rubric

Eight dimensions, each rated on its own evidence, then the derived `risk`. A rating is the level the evidence supports; when two levels fit, take the higher and say why.

## complexity: low | medium | high | extreme

Amount of logic, components touched, integration points, architectural impact, breadth.

- low: one unit of logic, no new integration.
- medium: a normal feature or use case across a few components.
- high: several interacting components, a new integration, or a non-trivial algorithm.
- extreme: reshapes architecture or spans systems.

## ambiguity: low | medium | high

How clear expected behavior, rules and acceptance criteria are.

- low: behavior and criteria fully stated.
- medium: minor details open, resolvable from codebase convention.
- high: the behavior itself is open. Forces `needs-refinement`.

## hidden_edge_cases: low | medium | high | critical

Driven by **signals**: concurrency, retries, async processing, duplicate messages, ordering, distributed systems, financial calculations, dates and timezones, persistence, partial failure, migrations, external integrations, parsing, security, authentication and authorization, timeouts, fallback logic.

- low: no signal present.
- medium: one signal, well contained.
- high: several signals, or one that interacts with state.
- critical: signals where a missed case corrupts data, money or access.

## verification_requirement: low | medium | high | critical

How much we must **prove** the implementation correct. At least the level of `hidden_edge_cases`: each hidden case needs its own proof.

- low: reading the diff is enough.
- medium: normal automated tests.
- high: a test per edge case plus an independent review.
- critical: adversarial review; each signal needs its own proof.

## cost_of_failure: low | medium | high | critical

- low: data-shape change, rename, simple mapping.
- medium: normal internal feature, internal report.
- high: important persistence, messaging, system integration, sensitive business rules.
- critical: money, security, authorization, destructive migrations, data loss, critical production infrastructure.

## autonomy: interactive | balanced | autonomous | highly-autonomous

How much the work should run without a human in the loop. Rated apart from complexity: a very complex task can stay interactive.

- interactive: decisions surface mid-task and need a human.
- balanced: the agent works, a human checks at milestones.
- autonomous: the agent works to completion, a human reviews the result.
- highly-autonomous: long-horizon run, broad exploration, no checkpoints.

## scope: localized | multi-file | cross-module | codebase-wide | external-systems

Where the change or investigation reaches.

## nature_of_work

One primary value: implement, tdd, bugfix, investigation, refactor, architecture, migration, testing, security-review, performance, documentation, configuration, requirement-refinement.

## risk: impact, likelihood, overall

Derived from the ratings above. Impact and likelihood stay separate: a failure can be likely and cheap, or unlikely and ruinous.

- **impact** = `cost_of_failure`.
- **likelihood**: how likely the implementation ships a relevant failure. Start at the higher of `hidden_edge_cases` and `complexity` (extreme counts as critical); concurrency and distributed behavior already count through `hidden_edge_cases`. Raise one level, once, capped at critical, when any amplifier is present: residual ambiguity (medium), legacy or brownfield code, an integration with unfamiliar or undocumented behavior, failures that are hard to reproduce or debug.
- **overall**, from the matrix:

| impact \ likelihood | low | medium | high | critical |
|---|---|---|---|---|
| low | low | low | medium | medium |
| medium | low | medium | medium | medium |
| high | medium | high | high | high |
| critical | high | critical | critical | critical |

Impact sets the level; a low likelihood softens it one step; a high or critical likelihood keeps it at medium or above. Edge cases raise verification (`verification_requirement`, the review in [`ROLES.md`](ROLES.md)), not impact.

`risk.overall` is the value a `risk:*` label carries.
