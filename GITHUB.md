# GitHub

The github-metadata-generator. It turns a routing decision and its current resolution into a few labels and a block in the issue body, and writes them. It writes to the issues the user named when invoking the router; reached any other way (another skill, an autonomous run), it shows what it would write and asks first.

## Labels

Labels serve filtering, automation and a quick read; everything else lives in the block. The router owns exactly the labels in this table, by full name, and keeps one profile, one workflow and one risk label on a ready issue. A label outside the table stays untouched even when it shares a prefix (`risk:compliance`, `ai:workflow:manual`).

| Label | Set when | Color |
|---|---|---|
| `ai:profile:fast` | implementation role `fast` | `C5DEF5` |
| `ai:profile:balanced` | implementation role `balanced` | `1D76DB` |
| `ai:profile:frontier` | implementation role `frontier-interactive` or `frontier-verifier` | `5319E7` |
| `ai:profile:autonomous` | implementation role `deep-autonomous` | `0B1F66` |
| `ai:workflow:tdd` | `nature_of_work` tdd | `D4C5F9` |
| `ai:workflow:implement` | implement, migration, configuration, documentation, testing | `D4C5F9` |
| `ai:workflow:bugfix` | bugfix | `D4C5F9` |
| `ai:workflow:investigation` | investigation, performance | `D4C5F9` |
| `ai:workflow:refactor` | refactor | `D4C5F9` |
| `ai:workflow:architecture` | architecture | `D4C5F9` |
| `ai:workflow:review` | security-review | `D4C5F9` |
| `ai:workflow:refinement` | requirement-refinement: the issue is ready, and its work is refining requirements | `D4C5F9` |
| `risk:low` | `risk.overall` low | `0E8A16` |
| `risk:medium` | `risk.overall` medium | `FBCA04` |
| `risk:high` | `risk.overall` high | `D93F0B` |
| `risk:critical` | `risk.overall` critical | `B60205` |
| `ai:needs-refinement` | the readiness gate failed | `EDEDED` |

- **Ready**: its profile, workflow and risk labels from the table; every other table label comes off, `ai:needs-refinement` included.
- **Needs refinement**: `ai:needs-refinement` only; every other table label comes off, since the issue has no execution profile yet.
- The review profile, efforts, dimensions and model names stay in the block. A model name is never a label: labels persist, models change.
- A missing label is created on first use, with the color above and its "Set when" text as description.

## Issue block

The block sits between markers in the issue body and is the canonical routing state of the issue: the router keeps its state there and nowhere else, comments included. The router owns only the content between its markers; the author's text outside them is never touched. The first run appends the block to the end of the body, after a blank line (an empty body gets the block alone); later runs replace it in place. Every start and end pair belongs to the router: a body holding more than one (copied from another issue, say) keeps the first, rebuilt, and loses the others. A start marker without its end marker makes the extent unknowable, so the body is left as it is and the summary reports it. Values are title-cased (`frontier-verifier` shows as Frontier Verifier, `tdd` as TDD).

Ready:

```markdown
<!-- ai-execution-router:start -->
## AI Execution

Status: Ready

### Classification
- Complexity: Medium
- Ambiguity: Low
- Hidden Edge Cases: High
- Verification Requirement: High
- Cost of Failure: High
- Scope: Multi-file
- Autonomy: Balanced
- Risk: High (impact High, likelihood High)

### Implementation
- Profile: Balanced
- Effort: Medium
- Effort Control: Preferred

### Review
- Profile: Frontier Verifier
- Effort: High
- Effort Control: Required

### Workflow
TDD

### Rationale
- Requirements are sufficiently specified.
- Implementation complexity is moderate.
- Idempotency introduces hidden edge cases.
- Independent high-effort review is recommended.

### Current Model Resolution
_Temporal: resolved <date> with registry v<version> generated <date>, strategy <strategy>. The execution profile above is permanent; the models below are re-resolved at execution time._

Implementation:
- Provider: <provider>
- Model: <model-id>
- Reasoning: <native value, or "provider default">

Review:
- Provider: <provider>
- Model: <model-id>
- Reasoning: <native value>

<!-- ai-execution-router:decision
<the step 5 YAML, plus schema, classified_at and source_sha256>
-->
<!-- ai-execution-router:end -->
```

A phase that could not be resolved shows `- Unresolved: <reason in one line>` instead of provider and model.

Needs refinement:

```markdown
<!-- ai-execution-router:start -->
## AI Execution

Status: Needs Refinement

Missing Information:
- <each question from missing_information>

Recommended Action:
Run requirement clarification before implementation.

<!-- ai-execution-router:decision
<the step 5 short-form YAML, plus schema, classified_at and source_sha256>
-->
<!-- ai-execution-router:end -->
```

The hidden decision comment is the machine-readable copy of the policy:

```yaml
schema: 1
classified_at: 2026-09-26T18:00:00Z
source_sha256: <SHA-256 of the normalized issue>
decision: { <step 5 YAML> }
```

**Normalized issue**: the title, a newline, then the body with every router block removed, line endings as LF, and leading and trailing whitespace trimmed. Normalizing keeps the hash stable across the blank lines the block adds and the CRLF endings GitHub's editor may save, while a changed title or body still changes it.

## Reuse

Read the block from the body with line endings converted to LF. A stored decision still holds when its status is `ready`, the normalized issue hashes to `source_sha256`, no comment is newer than `classified_at`, and the user did not ask to reclassify. Then step 1 of SKILL.md skips classification, only the resolution is refreshed, and the decision keeps its `classified_at` and `source_sha256`. A `needs-refinement` decision is always classified again: the refinement may have answered it. Linked documents are outside the hash: when a linked spec changed, the user asks to reclassify.

## Writing

Per issue:

1. Re-read the issue right before writing (`gh issue view <n> --json title,body,labels`) and build the block on that fresh body.
2. Body: write only when the block differs from the current one in more than the resolution date. Then write the new body to a file in the scratchpad and run `gh issue edit <n> --body-file <file>`.
3. Labels: add and remove only the difference. Create any missing label first (`gh label create <name> --color <hex> --description "<text>"`), then run `gh issue edit <n> --add-label <labels> --remove-label <labels>`.

An unchanged issue on an unchanged registry therefore produces no edit. The two writes are not one transaction: the body goes first, so a label failure leaves a new block beside old labels. Each failure (permission, network) is reported in the summary with what is left to apply, and the next run converges, since both writes are idempotent. Pasted issues and other trackers get the same block and labels in the summary, with no write.
