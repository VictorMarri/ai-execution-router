# Registry

[`registry/models.yaml`](registry/models.yaml) is the only place model names live. This file covers **availability**: which models exist, their IDs and lifecycle, and how the registry stays fresh. Which model serves which role is **fitness**, in [`RESOLVER.md`](RESOLVER.md).

## Providers

```yaml
version: 1                          # schema version; bump on a breaking change
generated_at: 2026-09-26T00:00:00Z  # last successful refresh, UTC
max_age_days: 7                     # TTL
watch:
  window_days: 60                   # an active model this close to retirement goes under watch
  max_age_days: 1                   # TTL while an incumbent this run needs is under watch
evaluation:
  role_bar: 0.7                     # minimum measured role score for a fit
  evidence_max_age_days: 90         # measured evidence older than this is stale
  queue: []                         # { role, candidates, reason, queued_at }; the refresh adds, the evaluator clears
  pending_promotions: []            # { role, challenger, incumbent, run_id, confidence, reasons }; recommended, not applied
diversity_bonus:                    # measured effect of cross-model review, see EVALUATOR.md
  status: unmeasured                # unmeasured | helps | neutral | hurts
  confidence:
  evaluated_at:

providers:
  <provider>:
    key_env: <ENV_VAR>              # when set, the models API is readable
    sources:                        # every official page the refresh reads
      models_api: <official endpoint listing models>
      models_doc: <official models page>
      deprecations_doc: <official deprecations page>
      effort_doc: <official effort page, when models_doc omits it>
    models:
      <model-id>:                   # the exact ID sent to the API
        lifecycle:
          status: active            # active | deprecated | retired | unsupported
          deprecated_at:            # date the provider deprecated it
          retirement_at:            # announced retirement date
          retirement_not_before:    # the provider's floor, when no date is announced
          replacement:              # <provider>/<model-id>, once announced
        lifecycle_watch: false      # set by the refresh, see Lifecycle
        capabilities:               # low | medium | high, relative across the registry
          reasoning: high
          coding: high
          agentic: high
          interaction: high
          verification: high
          autonomy: high
          long_context: high
        supported_effort:           # normalized level: the provider's native value
          low: low
          medium: medium
          high: high
          max: max
        relative_cost: 20           # list price, USD per 1M output tokens
        speed:                      # external measurement only; eval-pack latency lives in measured evidence
          latency_p50_s:            # median seconds on the source's reference workload
          source:                   # benchmark:<name> | official-quantitative
        evidence: [official-docs]   # what the capability grades rest on
```

An empty field means unknown.

Effort is normalized to low, medium, high and max. `supported_effort` maps each level a model accepts to its provider's own control (an effort value, a reasoning setting, a thinking budget). A level absent from the map is unsupported; `supported_effort: none` means the model has no effort control and runs at its default.

A new provider is one more entry under `providers` with its sources; the next refresh fills its models.

## Lifecycle

`lifecycle.status` alone decides availability. An `active` model whose `retirement_at` or `retirement_not_before` falls within `watch.window_days` gets `lifecycle_watch: true`: it stays active and eligible, the registry refreshes on the shorter `watch.max_age_days`, and every resolution that picks it carries a warning. The model leaves service only when the provider changes its status.

## Refresh

When to refresh is decided in [`SKILL.md`](SKILL.md), step 6.

Cheap by design: official sources, one read each, no benchmarks, no trial runs. In-depth fitness evaluation is a later stage.

The registry tracks each provider's current lineup for text and coding work, as its models page presents it. Legacy, audio, realtime, image and embedding models stay out, unless a role still lists them.

Per provider:

1. Read every source listed, `models_api` only when `key_env` is set. The provider's own release notes count as official. Third-party sources fill only a gap the official ones leave, recorded as `third-party`.
2. Known models: update `lifecycle`, `supported_effort`, and any capability or cost the docs state. `speed` changes only on quantitative data.
3. Models new to the registry: add them with their lifecycle. When the docs position one for a kind of work (fastest, best for coding, most capable), list it under the matching role as `evaluation_status: challenger` with global confidence low and `user_preference: null` (see [`RESOLVER.md`](RESOLVER.md)). A new model enters as a challenger; the incumbent keeps the role until evidence moves it.
4. Recompute `lifecycle_watch` for every model.
5. Incumbents no longer `active`: replace them by the rule in [`RESOLVER.md`](RESOLVER.md).
6. Mark `stale` every candidate whose measured evidence is past `evaluation.evidence_max_age_days` or its suite version, and add each evaluation trigger ([`EVALUATOR.md`](EVALUATOR.md), Triggers) to `evaluation.queue`, once per role and reason. Queuing is all the refresh does; no evaluation runs here.
7. Set `generated_at` to now.

Done when every source was read or recorded as unreachable. A provider whose sources all fail keeps its entries unchanged. When every provider fails, `generated_at` stays put and the run continues with the warning `stale-registry`.
