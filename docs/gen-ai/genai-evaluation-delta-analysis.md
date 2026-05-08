# GenAI Human Evaluation: Delta Analysis

## Purpose

This document compares the internal Microsoft specification for human evaluation events
against the upstream OpenTelemetry semantic conventions for the `gen_ai.evaluation.result` event.

The goal is to identify gaps, alignment points, and candidates for upstream contribution.

## OTel Upstream Reference

- Event: `gen_ai.evaluation.result` (status: **Development**)
- Model: [`model/gen-ai/events.yaml`](../../../model/gen-ai/events.yaml)
- Registry: [`model/gen-ai/registry.yaml`](../../../model/gen-ai/registry.yaml)
- Docs: [`docs/gen-ai/gen-ai-events.md`](gen-ai-events.md)

---

## Attributes That Align

These attributes exist in OTel semconv and are reused by the internal spec.

| Attribute | OTel Stability | OTel Requirement Level | Internal Requirement Level | Delta |
|---|---|---|---|---|
| `gen_ai.evaluation.name` | Development | Required | Required | **None** |
| `gen_ai.evaluation.score.value` | Development | Conditionally Required (if applicable) | Required | **Stricter** — internal spec always requires it |
| `gen_ai.evaluation.score.label` | Development | Conditionally Required (if applicable) | Conditionally Required (dependent on evaluation kind) | **None** (semantically aligned) |
| `gen_ai.evaluation.explanation` | Development | Recommended | Optional | **None** (minor wording difference) |
| `gen_ai.response.id` | Development | Recommended (when available) | Optional (recommended when available) | **None** |
| `user.id` | Development (in `registry.user`) | Not referenced in evaluation event | Optional | **Gap** — exists in OTel registry but not in the evaluation event definition |

## Attributes in Internal Spec but NOT in OTel

These are custom extensions under Microsoft-specific namespaces or new `gen_ai.*` attributes
not yet proposed upstream.

| Attribute | Type | Requirement | Description | Upstream Candidate? |
|---|---|---|---|---|
| `gen_ai.evaluation.score.max` | double | Required | Maximum possible score value | **Yes** — universally useful for interpreting scores across different scales |
| `gen_ai.evaluation.score.min` | double | Required | Minimum possible score value | **Yes** — same rationale as `score.max` |
| `gen_ai.response.id.type` | string | Conditionally Required (when `gen_ai.response.id` is available) | Clarifies what `response.id` represents (e.g., "responses", "session") | **Maybe** — could be useful generically, but current values feel Microsoft-specific |
| `user.authenticated_id` | string | Optional (recommended for builder) | Authenticated identity of the evaluator | **Unlikely** — OTel has `user.id`, `user.name`, `user.hash`, `user.email` but no authenticated identity concept |
| `feedback.tags.<tag>` | string (template) | Optional | Extra metadata associated with the evaluation | **No** — custom namespace, no OTel equivalent pattern |
| `feedback.id` | string | Optional (recommended) | Unique ID for the feedback event itself | **No** — OTel events are identified by the log record identity |
| `feedback.source` | string | Required | Feedback provider type ("builder", "end_user") | **No** — specific to Microsoft's builder/end-user distinction |
| `feedback.kind` | string | Required | Evaluation input method ("binary", "likert_5") | **No** — specific to Foundry evaluation templates |

## Attributes in OTel but NOT in Internal Spec

| Attribute | OTel Requirement Level | Description | Recommendation |
|---|---|---|---|
| `error.type` | Conditionally Required (if operation ended in error) | Describes a class of error the evaluation operation ended with | **Add to internal spec** — important for operational observability of evaluation failures |

---

## Key Semantic Differences

### 1. `score.value` Requirement Level

OTel treats `score.value` as conditionally required ("if applicable") because some evaluations
may only produce a label without a numeric score. The internal spec makes it always required.

**Impact**: Internal spec is stricter — this is fine as a superset constraint but means
internal events always carry a numeric score even for binary pass/fail.

### 2. No `score.min` / `score.max` in OTel

The upstream convention assumes scores are self-contained or that the label provides
interpretation. The internal spec explicitly defines the range, which is necessary for
interpreting scores across different evaluation kinds (binary 0–1 vs. Likert 1–5).

**Recommendation**: Propose `gen_ai.evaluation.score.min` and `gen_ai.evaluation.score.max`
upstream. These are not Microsoft-specific and benefit any evaluation system with variable ranges.

### 3. The `feedback.*` Namespace Is Entirely Custom

OTel's `gen_ai.evaluation.result` is designed for both human and automated evaluators and
doesn't distinguish feedback source, kind, or carry a feedback ID. The internal spec layers
human-evaluation-specific semantics on top.

**Recommendation**: Keep as custom extensions. If the community shows interest in
human-evaluation-specific attributes, the `feedback.*` pattern could be proposed as a new
OTel namespace, but this is premature today.

### 4. `gen_ai.response.id.type` Not Upstream

OTel treats `gen_ai.response.id` as an opaque identifier. The internal spec adds typing
to distinguish different ID formats (e.g., "responses", "session").

**Recommendation**: Evaluate whether this is needed generically. If response ID formats
are only meaningful in Microsoft systems, keep it custom.

---

## Recommendations Summary

| Action | Priority | Details |
|---|---|---|
| Propose `gen_ai.evaluation.score.min` and `score.max` upstream | **High** | Universally useful, not vendor-specific |
| Add `error.type` to internal spec | **High** | Already in OTel, needed for error observability |
| Add `user.id` ref to OTel evaluation event | **Medium** | Attribute exists in registry, just needs a ref in the event definition |
| Evaluate `gen_ai.response.id.type` for upstream | **Low** | May be too vendor-specific |
| Keep `feedback.*` as custom namespace | — | Appropriate as Microsoft-specific extension |
| Keep `user.authenticated_id` as custom | — | No OTel equivalent concept |

## Open Questions (from Internal Spec)

1. **Should `user.id` be set for builders too?** — From OTel perspective, `user.id` is a generic
   identifier. If builders have a user ID, it should be set. `user.authenticated_id` can carry
   the authenticated identity separately.

2. **What should `score.label` be for Likert 5-point evaluations?** — OTel says labels should have
   low cardinality. Options:
   - Numeric string ("1", "2", ..., "5") — simple but redundant with `score.value`
   - Descriptive ("strongly_disagree", "disagree", "neutral", "agree", "strongly_agree") — more
     useful but ties to a specific Likert interpretation
   - Omit entirely — valid per OTel since `score.label` is conditionally required "if applicable"
