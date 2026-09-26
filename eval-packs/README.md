# Eval packs

Packs of evaluation cases produce the `fitness` evidence in the registry (layers in [`../RESOLVER.md`](../RESOLVER.md), Evidence). The procedure that runs them is [`../EVALUATOR.md`](../EVALUATOR.md).

```text
eval-packs/
  config.yaml              budget, judges, privacy
  generic/                 kind: generic         -> fitness.global, plus category entries by case category
  <category pack>/         kind: category        -> fitness.category_specific.<category>
                                                    e.g. database/, distributed-systems/, security/
  <specialization pack>/   kind: specialization  -> fitness.optional_specialization.<pack>
                                                    e.g. backend/, frontend/, languages/<language>/, frameworks/<framework>/
  results/                 one file per run; also the cache
```

- The generic pack is the baseline and the only source of `global`. Every other pack complements it; the router works with the generic pack alone.
- A pack writes only its own layer, with `source: eval-pack:<pack>@<suite_version>`.
- Cases state problems in neutral terms, free of any real project, company, product or private code. Code appears as pseudocode, or as a behavior contract (input, output) the candidate implements in any language the runner offers; language and framework skill belong to specialization packs.
- Evidence drawn from one project's own issues goes in a specialization pack for that project and never reaches `global` or `category_specific`.
- Any change to a case bumps the pack's `suite_version`.

## pack.yaml

```yaml
name: generic
kind: generic            # generic | category | specialization
suite_version: 2
synthetic: true          # every fixture was invented for the suite
authorized: false        # true only when the user cleared non-synthetic fixtures for external providers
cases: [cases/<id>.yaml]
```

## Case

```yaml
id: <PREFIX-NNN>
suite_version: 2
case_version: 1                  # bumps when this case changes; the pack's suite_version bumps with it
category: <task category>        # from the taxonomy in RESOLVER.md
difficulty: easy | medium | hard
roles: [<role>]                  # the roles this case feeds
task: |                          # sent to the candidate
context: |                       # sent to the candidate
expected_behavior: |             # answer key from here down
must_detect: []                  # items a good answer covers; judged present or absent
must_detect_dimension: edge_case_discovery
must_not_do: []                  # any violation zeroes requirement_adherence
must_pass: []                    # objective check ids, or invariants judged pass or fail; must hold in every repetition, or the case fails its roles' hard gates
objective_checks: []             # kind: io | sql-result | checklist
qualitative_rubric: {}           # <dimension>: criteria, judged 0 to 4
timeout_or_budget: { max_minutes: 10, max_output_tokens: 8000 }
```
