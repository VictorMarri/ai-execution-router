# ai-execution-router

A [Claude Code](https://claude.com/claude-code) skill that decides **how** an AI agent should work on each software-engineering issue: which kind of model, how much reasoning, which workflow, and how hard the result must be verified. It then resolves that decision to the best concrete model available **on the day the work runs**, and records both on the GitHub issue.

> **Status: v0.** The policy has been audited against scripted scenarios and blind readings, but the skill has not yet run against a real repository. Expect changes after the first real runs.

## Contents

- [Why it exists](#why-it-exists)
- [The core idea: policy now, model later](#the-core-idea-policy-now-model-later)
- [A quick example](#a-quick-example)
- [How a run works](#how-a-run-works)
- [Classification](#classification)
- [Roles, effort and effort control](#roles-effort-and-effort-control)
- [The model registry](#the-model-registry)
- [Resolving a model](#resolving-a-model)
- [Ties and preferences](#ties-and-preferences)
- [What gets written to GitHub](#what-gets-written-to-github)
- [Optional: evaluating models](#optional-evaluating-models)
- [Installation](#installation)
- [Usage](#usage)
- [Configuration](#configuration)
- [Repository layout](#repository-layout)
- [Design principles](#design-principles)
- [Current status and limitations](#current-status-and-limitations)
- [License](#license)

## Why it exists

Handing issues to AI agents raises the same questions every time:

- **Which model?** A frontier model on a one-line rename wastes money. A fast model on concurrent message handling ships bugs.
- **How much reasoning?** Most providers now expose an effort or reasoning setting. The right level depends on the task, not on the model.
- **Who checks the work?** Code that touches money, security or concurrency needs a stronger, independent review than a text change.
- **For how long is the answer valid?** Model names change every few months. An issue that says "use model X" is stale before it is implemented.

The router answers the first three questions from the issue itself, with a written rubric, and makes the fourth one irrelevant: the issue stores **capabilities**, never model names.

## The core idea: policy now, model later

Model names are configuration. Model capabilities are domain logic.

```mermaid
flowchart LR
    I["GitHub issue"] --> C["Classifier<br/>rubric + roles"]
    C --> D["Routing decision<br/>role, effort and effort control<br/>per phase"]
    D -->|"stored in the issue body"| S[("Issue block<br/>permanent policy")]
    S -->|"at execution time"| R["Resolver"]
    G[("Model registry<br/>refreshed from official docs")] --> R
    R --> M["Provider, model, reasoning setting<br/>audit only, re-resolved on every run"]
```

- The **routing decision** is the policy. It says things like "implementation: balanced role, medium effort; review: frontier verifier, high effort". It stays valid for as long as the issue lives.
- The **resolution** turns that policy into a concrete provider, model and reasoning setting, using the registry of that moment. It is kept only as audit and is replaced on every run.

So an issue classified today runs on tomorrow's best model without anyone editing it.

## A quick example

Issue: *"Implement idempotent event processing with retries and concurrent consumers."*

The router classifies it and stores this decision in the issue:

```yaml
status: ready

execution_profile:
  workflow: tdd
  autonomy: balanced

analysis:
  complexity: medium
  ambiguity: low
  hidden_edge_cases: high
  verification_requirement: high
  cost_of_failure: high
  scope: multi-file

risk:
  impact: high
  likelihood: high
  overall: high

implementation:
  model_role: balanced
  effort: medium
  effort_control_requirement: preferred

review:
  model_role: frontier-verifier
  effort: high
  effort_control_requirement: required

rationale:
  - Requirements are sufficiently specified.
  - Implementation complexity is moderate.
  - Idempotency and concurrency create hidden edge cases.
  - Strong independent verification is recommended.
```

Then it resolves the models, writes labels (`ai:profile:balanced`, `ai:workflow:tdd`, `risk:high`) and prints a summary:

```text
Issue #123
Ready
Implementation: Balanced / Medium
Review: Frontier Verifier / High
Workflow: TDD
Risk: High
Current resolution: <provider> / <model> → <provider> / <model> for review
```

The work is moderate, so a balanced model implements it. The hidden edge cases are what make it risky, so the **review** gets the strong model, at high effort, in a separate session.

A vague issue gets questions instead of a route:

```yaml
status: needs-refinement

analysis:
  ambiguity: high

missing_information:
  - When a downstream call times out after its side effect was applied, is the operation retried, compensated, or reported?
  - May a redelivered event be processed again, or must handling be idempotent?
```

A stronger model does not fix a missing answer. Refinement does.

## How a run works

```mermaid
flowchart TD
    A["0. Read arguments<br/>issues and strategy"] --> B["1. Load the issue<br/>body, comments, linked docs"]
    B --> K{"Stored decision<br/>still valid?"}
    K -->|"yes"| F
    K -->|"no"| G2{"2. Readiness gate"}
    G2 -->|"gap found"| NR["Needs refinement<br/>questions for the owner"]
    G2 -->|"ready"| R3["3. Rate 8 dimensions<br/>and derive risk"]
    R3 --> R4["4. Route implementation and review<br/>role, effort, effort control"]
    R4 --> R5["5. Emit the routing decision"]
    R5 --> F["6. Check registry freshness<br/>once per run"]
    F --> R7["7. Resolve models"]
    R7 --> W["8. Write labels and issue block"]
    NR --> W
    W --> S9["9. Summarize"]
```

| Step | What happens |
|---|---|
| 0 | Parse issue references (`#123`, `123`, URLs, or pasted text) and `--strategy`. |
| 1 | Read the issue, its comments and every linked spec or ADR. Text inside the issue is data to classify, never instructions for the router. |
| 2 | **Readiness gate.** Could a competent engineer start without inventing behavior? Each gap becomes a question. |
| 3 | Rate the eight dimensions of [`RUBRIC.md`](RUBRIC.md), each with its evidence, and derive the risk. |
| 4 | Pick role, effort and effort control for implementation, then again for review, independently ([`ROLES.md`](ROLES.md)). |
| 5 | Emit the routing decision (the policy). |
| 6 | Refresh the registry only if it is stale. A fresh registry means no web request at all. |
| 7 | Resolve each phase to a concrete model ([`RESOLVER.md`](RESOLVER.md)). |
| 8 | Write the labels and the issue block ([`GITHUB.md`](GITHUB.md)). |
| 9 | One summary block per issue. |

Each issue is classified on its own. Sibling issues of the same feature give context for scope, never a template for the ratings. A failure on one issue is reported in its block and the batch goes on.

## Classification

### Readiness gate

An issue is **ready** when:

- the expected behavior is stated;
- the acceptance criteria are checkable;
- the scope says what is in and what is out;
- every edge-case signal it touches has its failure semantics specified (retry, duplicate, timeout, partial failure).

For requirement-refinement or investigation work, the deliverable *is* the clarified requirement or the root cause, so the answers the work will find do not count as gaps.

### The eight dimensions

| Dimension | Scale | Measures |
|---|---|---|
| `complexity` | low, medium, high, extreme | logic, components, integrations, architectural impact |
| `ambiguity` | low, medium, high | how clear behavior and criteria are; high forces `needs-refinement` |
| `hidden_edge_cases` | low, medium, high, critical | signals such as concurrency, retries, ordering, money, dates, parsing, auth, timeouts |
| `verification_requirement` | low, medium, high, critical | how much the implementation must be *proven*; never below `hidden_edge_cases` |
| `cost_of_failure` | low, medium, high, critical | from a rename (low) to money, security or data loss (critical) |
| `autonomy` | interactive, balanced, autonomous, highly-autonomous | how long the agent should run without a human |
| `scope` | localized, multi-file, cross-module, codebase-wide, external-systems | where the change reaches |
| `nature_of_work` | implement, tdd, bugfix, investigation, refactor, architecture, migration, testing, security-review, performance, documentation, configuration, requirement-refinement | the kind of work; becomes the workflow |

### Risk

Risk keeps **impact** and **likelihood** apart, because a failure can be likely and cheap, or unlikely and ruinous.

- **impact** = `cost_of_failure`.
- **likelihood** starts at the higher of `hidden_edge_cases` and `complexity`, and rises one level (once) for amplifiers: residual ambiguity, legacy code, an undocumented integration, failures that are hard to reproduce.

| impact \ likelihood | low | medium | high | critical |
|---|---|---|---|---|
| **low** | low | low | medium | medium |
| **medium** | low | medium | medium | medium |
| **high** | medium | high | high | high |
| **critical** | high | critical | critical | critical |

`risk.overall` becomes the `risk:*` label.

## Roles, effort and effort control

**Role** follows how the work runs. **Effort** follows how hard the model must think inside that role. **Effort control** says whether that effort must be applied through an explicit reasoning setting. The three are decided separately.

### Roles

| Role | For | Weighted capabilities |
|---|---|---|
| `fast` | deterministic, localized, clear, low-risk work | speed, coding |
| `balanced` | ordinary engineering in any layer | coding, agentic |
| `frontier-interactive` | hard work with a human in the loop | reasoning, interaction |
| `frontier-verifier` | adversarial review and strong verification | verification, reasoning |
| `deep-autonomous` | long independent investigation | autonomy, agentic, long context |

Implementation role, first match wins:

```mermaid
flowchart TD
    S{"nature_of_work is<br/>security-review?"} -->|"yes"| FV["frontier-verifier"]
    S -->|"no"| D{"investigation or unknown-cause bugfix,<br/>autonomous or more,<br/>scope cross-module or wider?"}
    D -->|"yes"| DA["deep-autonomous"]
    D -->|"no"| X{"complexity extreme, or interactive with<br/>high complexity, architecture<br/>or requirement-refinement?"}
    X -->|"yes"| FI["frontier-interactive"]
    X -->|"no"| L{"complexity, edge cases and cost of failure<br/>all low, scope localized or multi-file?"}
    L -->|"yes"| FA["fast"]
    L -->|"no"| BA["balanced"]
```

The review is chosen independently, from `verification_requirement`, `hidden_edge_cases` and `cost_of_failure`: **frontier-verifier** when any of them is high or critical (or the work is a security review), **fast** when all are low, **balanced** otherwise. The review runs in a session separate from the implementation, so it verifies instead of agreeing.

### Effort

| Effort | When |
|---|---|
| `low` | small scope, clear requirements, low risk |
| `medium` | the default: normal feature, normal TDD, well-specified rules |
| `high` | edge cases or correctness matter: concurrency, messaging, legacy code, a hard bug, adversarial review |
| `max` | reserved: broad autonomous investigation, critical security analysis, extreme debugging |

A hard but well-specified task is often `balanced` at `high` effort: difficulty raises effort before it raises role. Importance alone earns a `high` review, not `max`.

### Effort control

Some models expose an explicit effort or reasoning setting, some do not. `effort_control_requirement` says how much that matters for a phase:

| Value | Meaning | Effect on a model without the control |
|---|---|---|
| `none` | the provider's default reasoning serves the work | ignored in eligibility and ranking |
| `preferred` | the effort helps, and a shallow run would still be caught later (the review, a human) | stays eligible; loses only a tie against a model that honors the effort |
| `required` | the phase is the last check on correctness, so a shallow run would ship unnoticed | ineligible, unless accepted measured evidence shows its default configuration reaching the role bar |

Per phase, first match wins:

1. **required**: the phase verifies (the review, or security-review work) and `verification_requirement` or `cost_of_failure` is high or critical.
2. **none**: the phase's role is `fast`.
3. **preferred**: otherwise.

No effort level implies a requirement: a `balanced` implementation at `high` effort stays `preferred`.

## The model registry

[`registry/models.yaml`](registry/models.yaml) is the **only** place model names live. It holds:

- **providers**, each with its official sources (models API, models page, deprecations page, effort page) and its models;
- per model: lifecycle, capability grades, `supported_effort` (normalized level to the provider's native value, or `none`), list price and speed data;
- **roles**, each with an incumbent and its candidates, and each candidate's fitness evidence.

```yaml
claude-sonnet-5:
  lifecycle: { status: active, retirement_not_before: 2027-06-30 }
  capabilities: { reasoning: medium, coding: high, agentic: high, ... }
  supported_effort: { low: low, medium: medium, high: high, max: max }
  relative_cost: 10          # USD per 1M output tokens
  evidence: [official-docs]
```

### Freshness

```mermaid
flowchart TD
    S["Start of run"] --> M{"Registry exists?"}
    M -->|"no"| RF["Refresh"]
    M -->|"yes"| A{"User asked<br/>for a refresh?"}
    A -->|"yes"| RF
    A -->|"no"| W{"A needed incumbent is under<br/>lifecycle watch or no longer active?"}
    W -->|"yes"| T1{"Older than 1 day?"}
    W -->|"no"| T7{"Older than 7 days?"}
    T1 -->|"yes"| RF
    T1 -->|"no"| USE["Use it as is<br/>no web request"]
    T7 -->|"yes"| RF
    T7 -->|"no"| USE
    RF --> OK{"Any provider<br/>reachable?"}
    OK -->|"yes"| NEW["Update lifecycle, effort and price<br/>new models enter as challengers<br/>queue evaluation triggers"]
    OK -->|"no"| STALE["Keep entries<br/>warning stale-registry"]
```

A refresh is cheap by design: official sources only, one read each, no benchmarks, no trial runs. A model new to the registry enters as a **challenger** with low confidence. The incumbent keeps its role until evidence moves it. A model whose retirement date, or the provider's earliest possible retirement date, is within 60 days goes under **lifecycle watch**: it stays eligible, but every resolution that picks it carries a warning.

## Resolving a model

```mermaid
flowchart TD
    P["Phase<br/>role, effort, effort control"] --> E["Eligible candidates<br/>active; confidence medium or high, or incumbent;<br/>meets the effort control requirement"]
    E -->|"none left"| U["unresolved, with the reason"]
    E --> ST{"routing_strategy"}
    ST -->|"best-fit"| BF["incumbent while eligible,<br/>else highest ranking confidence"]
    ST -->|"quality-first"| QF["highest measured score,<br/>else role profile"]
    ST -->|"cost-aware"| CA["lowest measured cost,<br/>else list price"]
    ST -->|"speed-first"| SF["fastest on shared evidence,<br/>else best-fit"]
    BF --> T{"Tied?"}
    QF --> T
    CA --> T
    SF --> T
    T -->|"no"| PICK["Pick, with resolved_reasoning"]
    T -->|"yes"| TC["Tie chain<br/>evidence layers, effort control if preferred,<br/>user preference, incumbent, lower price"]
    TC -->|"decided"| PICK
    TC -->|"unbroken"| TIE["Tied candidates<br/>one tie label each, picked at execution"]
```

### Evidence

Fitness comes only from the registry's evidence fields. What a model was used for before, in any project or conversation, is history, not evidence. Evidence has three layers:

- **global**: generic engineering evidence for the role. Required for every candidate.
- **category_specific**: evidence for one task category (code review, database reasoning, distributed systems, ...).
- **optional_specialization**: evidence from one specialization pack (a language, a framework). Only breaks ties for issues in that specialization.

Each claim carries a confidence: **low** (name, presence, third-party info), **medium** (official docs position the model for this work), **high** (a benchmark or eval pack confirmed it).

### Strategies

| Strategy | Picks |
|---|---|
| `best-fit` (default) | the incumbent while it is eligible; otherwise the highest ranking confidence |
| `quality-first` | the highest measured score when every candidate has one from the same suite; otherwise the best on the role profile |
| `cost-aware` | the lowest measured cost per case, else the lowest list price, else best-fit |
| `speed-first` | the fastest on comparable, shared evidence; otherwise best-fit |

Speed and cost rank only on evidence that every candidate shares. Qualitative labels such as "fastest" stay descriptive, because providers grade them on different scales.

### Reasoning setting

Each phase keeps both sides of the effort:

```yaml
requested_effort: high
effort_control_requirement: required
resolved_reasoning:
  provider_effort: high     # the provider's native value; null when unsupported
  status: supported         # supported | unsupported
```

The requested level is sent as asked or not at all. It is never swapped for another level.

### Implementation and review

Implementation is resolved first, then review, each on its own, so an efficient model can implement and a stronger one can verify. Model diversity is a tie-breaker, not a requirement: when the review ties with the implementation's model, an independent model of comparable fitness is preferred. A materially worse verifier is never chosen for diversity alone.

## Ties and preferences

A tie is broken, in order, by: ranking confidence; secondary-category confidence; specialization confidence; the model that honors the requested effort (only when effort control is `preferred`); the user's preference; the incumbent; the lower list price. A criterion is skipped when a candidate has no value for it, so an unknown value never counts as worse; `user_preference: null` is a value and means "not preferred".

A tie that survives every rule is **the user's decision**. Registry order and provider carry no weight. The router does not stop to ask: it lists every tied model in the resolution and puts one tie label per model on the issue (`ai:tie:review:<model-a>`, `ai:tie:review:<model-b>`), so whoever picks the issue up chooses which one runs. A run with nobody watching behaves the same way.

A tie and a standing preference are different things:

| | A tie | A preference |
|---|---|---|
| How it starts | a tie survives every rule | the user says so explicitly, e.g. "on ties, prefer <provider>" |
| Recorded as | `tied_candidates` on the phase, one `ai:tie:<phase>:<model-id>` label per candidate | `user_preference: { preferred: true }` on the matching candidates in the registry |
| Who chooses | whoever executes the issue, among the labeled models | the registry, on every later tie |
| Next run | lists the same candidates while the tie stands | decides the tie, no tie labels |

```mermaid
sequenceDiagram
    participant U as User
    participant R as Router
    participant I as Issue
    participant G as Registry
    Note over R: Run 1, issue 12, review tie between A and B
    R->>I: Labels ai:tie:review:A and ai:tie:review:B
    Note over G: Unchanged, user_preference stays null
    U->>I: Picks B when executing the issue
    Note over U,R: Later, the user states a preference
    U->>R: On ties, prefer the provider of B
    R->>G: user_preference preferred on that provider's candidates
    Note over R: Next run, B wins the tie and the tie labels come off
```

When the implementation is tied too, the review stays tied as well, and you prefer a review model other than the one you picked for implementation.

## What gets written to GitHub

### Labels

Labels are for filtering and automation. Everything else lives in the issue block.

| Family | Labels |
|---|---|
| Profile (implementation role) | `ai:profile:fast`, `ai:profile:balanced`, `ai:profile:frontier`, `ai:profile:autonomous` |
| Workflow | `ai:workflow:<name>` for tdd, implement, bugfix, investigation, refactor, architecture, review, refinement |
| Risk | `risk:low`, `risk:medium`, `risk:high`, `risk:critical` |
| Gate | `ai:needs-refinement` |
| Tie (only when a tie survives every rule) | `ai:tie:impl:<model-id>`, `ai:tie:review:<model-id>`, one per tied model |

The router owns exactly these labels: the fixed ones by full name, the tie labels by their `ai:tie:impl:` and `ai:tie:review:` prefixes. Any other label stays untouched, even with a shared prefix (`risk:compliance`). A resolved model is never a label: labels persist, models change. Tie labels are the exception, because they show a choice left to you; every run replaces them. Missing labels are created on first use.

### The issue block

The block sits between markers at the end of the issue body and is the **canonical routing state**. The author's text outside the markers is never touched.

```markdown
<!-- ai-execution-router:start -->
## AI Execution

Status: Ready

### Classification
- Complexity: Medium
- Hidden Edge Cases: High
- Risk: High (impact High, likelihood High)
  ...

### Implementation
- Profile: Balanced
- Effort: Medium
- Effort Control: Preferred

### Review
- Profile: Frontier Verifier
- Effort: High
- Effort Control: Required

### Current Model Resolution
_Temporal: resolved <date> with registry v1 generated <date>, strategy best-fit.
The execution profile above is permanent; the models below are re-resolved at execution time._
  ...

<!-- ai-execution-router:decision
schema: 1
classified_at: ...
source_sha256: ...
decision: { ... }
-->
<!-- ai-execution-router:end -->
```

- **Reuse.** The hidden decision carries a SHA-256 of the normalized issue (title plus body without the block, LF line endings, trimmed). When the hash matches, no comment is newer and the user did not ask to reclassify, the stored decision is reused and only the resolution is refreshed.
- **Idempotent writes.** The body is written only when the block changed in more than the resolution date. Labels are added and removed by difference. An unchanged issue on an unchanged registry produces no edit.
- **Consent.** The router writes only to the issues the user named. Reached any other way, it shows what it would write and asks first.

## Optional: evaluating models

Evaluation is a separate branch, off the routing path. It never runs on its own: only when the user asks, or accepts an `evaluation-recommended` warning, and only after confirming the cost. It answers one question per role: **which configuration (a model plus its reasoning setting) fits this role, on evidence?** It never asks which model is best in general.

```mermaid
flowchart TD
    TR["Triggers found by the refresh<br/>new model, deprecation, price or effort change,<br/>stale evidence, unbroken tie"] --> QU[("evaluation.queue")]
    QU -.->|"warning evaluation-recommended"| USR["User asks to evaluate"]
    USR --> SC["1. Scope<br/>roles, challengers, incumbent, cases"]
    SC --> BG{"2. Budget and privacy gate<br/>user confirms the estimate"}
    BG -->|"declined"| X["Nothing runs"]
    BG -->|"confirmed"| RUN["3. Run<br/>each configuration, repeated,<br/>with provenance"]
    RUN --> SCO["4. Score<br/>deterministic checks first,<br/>blind judges for the rest"]
    SCO --> GT{"Hard gates pass?"}
    GT -->|"no"| RET["retain-incumbent"]
    GT -->|"yes"| DEC{"5. Decide<br/>coverage, consistency,<br/>no regression, relevant gain"}
    DEC -->|"strong evidence"| PRO["promote-challenger"]
    DEC -->|"below the automatic bar"| REC["recommend-promotion<br/>waits for approval"]
    DEC -->|"no consistent gain"| RET
    DEC -->|"too little coverage"| INS["insufficient-evidence"]
    PRO --> WB["6. Write back<br/>registry evidence only"]
    REC --> WB
    RET --> WB
    INS --> WB
```

Key rules:

- **Challenger versus incumbent.** A new model is a challenger until evidence supports promotion. Availability is not fitness.
- **Hard gates.** A model cannot compensate for failing a critical capability by scoring high elsewhere. Each role has minimum scores per dimension (for example, a verifier needs correctness, verification quality and edge-case discovery), and each case can list `must_pass` checks that must hold in every repetition.
- **Blind judging.** Judges see anonymous, shuffled responses with provider and model names removed, and no cost, latency or incumbent status. Deterministic checks (executed I/O, SQL results) score before any judge.
- **Confidence.** Medium needs at least 3 cases and 2 repetitions. High needs 6 cases across 2 categories, 3 repetitions, half of the cases with an executed check, and a small spread across repetitions. Self-judged runs or partial blinding cap it at medium.
- **Promotion.** Automatic only with high confidence and a clear gain (at least 0.08 in role score, or 40% savings). Otherwise it is only recommended and waits for the user. One good run never promotes.
- **Privacy.** Only packs marked `synthetic` or `authorized` leave the machine, only to allowed providers, and only the case's task and context: no file from your repository, no secret.
- **Budget.** Default limits: USD 25 and 12 cases per run, 3 repetitions, 3 configurations per candidate.

### The generic pack

[`eval-packs/generic/`](eval-packs/generic) holds nine synthetic cases written in neutral pseudocode or I/O contracts, free of any real project:

| Case | Category | Roles it feeds |
|---|---|---|
| RD-001 | requirement-discovery | frontier-interactive |
| FI-001 | feature-implementation | fast, balanced |
| BF-001 | bugfix | fast, balanced, deep-autonomous |
| CR-001 | code-review | balanced, frontier-verifier |
| DB-001 | database-reasoning | fast, balanced |
| DS-001 | distributed-systems | frontier-interactive, frontier-verifier, deep-autonomous |
| AR-001 | architecture | frontier-interactive |
| SR-001 | security-review | frontier-verifier |
| LI-001 | long-horizon-investigation | deep-autonomous |

Category and specialization packs (a language, a framework, your own project) can be added beside it. They refine the evidence in their own layer and never define the global policy. See [`eval-packs/README.md`](eval-packs/README.md).

## Installation

Clone into your Claude Code skills folder:

```bash
git clone https://github.com/VictorMarri/ai-execution-router ~/.claude/skills/ai-execution-router
```

On Windows the folder is `%USERPROFILE%\.claude\skills\ai-execution-router`.

Requirements:

- [Claude Code](https://claude.com/claude-code).
- The [GitHub CLI](https://cli.github.com/) (`gh`), authenticated, to read issues and write labels and blocks. Without it, paste the issue text: the router prints the block and labels for you to apply.
- Web access for registry refreshes (once a week by default).
- Optional: provider API keys (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or whatever `key_env` names) let the refresh read each provider's models API. Evaluations send cases to the providers, so they need API access too.

## Usage

```text
/ai-execution-router #123
/ai-execution-router #123 #124 #125 --strategy cost-aware
/ai-execution-router https://github.com/<owner>/<repo>/issues/42
```

You can also paste an issue's text, or just ask: *"route these issues before I hand them to an agent"*, *"which model and effort should implement #42?"*, *"evaluate models for the frontier-verifier role"*.

The run ends at the summary. Implementing the issue is a separate request.

## Configuration

**[`registry/models.yaml`](registry/models.yaml)**

- `max_age_days` and `watch`: registry freshness.
- `providers`: add a provider as one more entry with its official sources; the next refresh fills in its models.
- `roles.<role>.candidates[].user_preference`: `null` by default. Set it only by telling the router explicitly, e.g. *"on ties, prefer <provider>"*, optionally naming the roles.
- `evaluation.role_bar` and `evidence_max_age_days`: the minimum measured score for a fit, and how long measured evidence stays fresh.

**[`eval-packs/config.yaml`](eval-packs/config.yaml)**

- `budget`: cost and size limits per evaluation run.
- `judges`: fixed judge models, or empty to use the resolver's verifier pick among models not under test.
- `role_requirements`: hard gates per role.
- `confidence` and `promotion`: every evaluation threshold.
- `privacy`: which providers cases may be sent to, and which packs may leave the machine.

## Repository layout

```text
.
├── SKILL.md              the entry point: steps 0 to 9
├── RUBRIC.md             the eight dimensions and the risk matrix
├── ROLES.md              roles, effort and effort control
├── REGISTRY.md           registry schema, lifecycle and refresh
├── RESOLVER.md           evidence, eligibility, strategies, ties, effort mapping
├── GITHUB.md             labels, issue block, reuse and writing
├── EVALUATOR.md          the optional evaluation procedure
├── registry/
│   └── models.yaml       the only place model names live
└── eval-packs/
    ├── README.md         pack and case schema
    ├── config.yaml       budget, judges, gates, confidence, promotion, privacy
    └── generic/          the baseline pack: nine synthetic cases
```

## Design principles

- **Project-, language-, framework- and provider-agnostic.** The routing policy works the same for backend, frontend, mobile, infrastructure, data, embedded and libraries. Project-specific evidence may refine a recommendation, but never defines the global policy.
- **Late binding.** Issues store capabilities; concrete models are resolved as late as possible.
- **Evidence over habit.** Past use of a model is history, not evidence. A preference exists only when the user states it.
- **Questions, not guesses.** A gap in the issue becomes a question. A gap filled with a guess is a defect shipped.
- **Challengers earn their role.** New models start as challengers; the incumbent keeps the role until evidence moves it.
- **Idempotent and minimal.** The same issue on the same registry produces no edit. The router owns only its labels and the text between its markers.

## Current status and limitations

- **Not yet run against a real repository.** The rules were checked with scripted scenarios (classification, eligibility, ties, freshness, promotion, GitHub parsing and labels) and blind readings of the documents.
- **Nothing is measured yet.** Every role mapping in the registry rests on official documentation (`unevaluated`). Three roles (`balanced`, `frontier-verifier`, `deep-autonomous`) are tied between providers, so an issue that needs them gets both models as tie labels and you pick one at execution, until you evaluate them or state a preference.
- **The registry is a snapshot** from 2026-09-26 with two providers. The first run after a week refreshes it from official sources.
- **No automated evaluation runner.** The agent follows the evaluation procedure itself.
- **The generic pack alone reaches medium confidence at most** (each role has fewer than six cases), so it can recommend a promotion but never apply one automatically.

## License

[MIT](LICENSE)
