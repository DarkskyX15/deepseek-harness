# Agent Note: pi-ai reasoning-efforts discovery and model editor

Status: implemented

English | [中文](2026-09-10-pi-ai-reasoning-efforts-discovery-and-editor.zh.md)

## Problem

The pi-ai adapter already accepted per-model `reasoningEfforts` configuration — `false` for a non-reasoning model or a level dict mapping the seven pi-ai levels to their wire spellings — through `resolveModelReasoning`, and the zod profile schema validated it. Two surfaces could not see it. "Fetch available models" returned candidates carrying only id, name, context window, and output cap, so an endpoint or catalog model that disclosed its thinking levels (OpenAI-style `reasoningEfforts`, pi-ai's `thinking_levels`, or a `reasoning: false` flag) lost that metadata on adoption. The Models page's per-model editor exposed capacities only, so a user had to hand-edit `settings.yaml` to declare a custom model's available thinking levels, and a custom-gateway workflow that started from discovery could never round-trip the levels the endpoint advertised.

## Decision

Discovery candidates now carry the levels an endpoint or catalog discloses, in the exact `reasoningEfforts` configuration shape, and the Models page edits them per model row.

`dsh-llm-pi-ai`'s `discoverModels` reads each listing entry's `reasoningEfforts` or `thinking_levels` field into the candidate's `reasoningEfforts` (an unrecognizable or empty dict, and an undisclosed capability, stay absent so a candidate never carries a value its profile would refuse), and a `reasoning: false` flag becomes `false`. The installed-catalog answer normalizes each catalog model's `thinkingLevelMap` with `reasoningCandidate`: a `null` wire means unsupported (key absent), an absent key means `off: null` (send nothing), `xhigh`/`max` stay absent, the other base levels spell themselves, and a string wire is carried. `LlmDiscoveredModel` gained the optional field, which the Typert Remote protocol and the web viewer's `api-catalog` declaration carry unchanged.

`dsh-client-ui-settings-models` edits the field with a Thinking-model switch (`false` ↔ a level dict) plus one row per level with its wire spelling, inside each model row's disclosure, with a row-level summary tag showing the enabled levels. Adoption copies `candidate.reasoningEfforts` when present; the shared model validator checks the value the way `resolveModelReasoning` will (non-off levels need a non-empty wire, only `off` may be `null`, at least one non-off level, known levels only) and reports the failure copy per row. All product copy is locale-owned.

## Alternatives considered

| Rejected | Reason |
|---|---|
| Edit `reasoningEfforts` only in `settings.yaml` | The discovery workflow is the path users actually take for custom gateways, and the Models page is the only surface that both fetches models and writes profiles; keeping the editor out reintroduces the hand-edit gap this note closes |
| Carry the raw listing object through to the model row and let the adapter validate on resolution | A candidate that adoption stores and the profile schema then refuses turns "fetch" into a trap; the editor validates before any write, and the adapter's gates remain the final authority on requests |
| Reuse the DeepSeek editor's capacity pattern for levels | The levels are a dict of booleans plus wire strings, not K/M counts; the dedicated control states every level explicitly so the user sees exactly what the adapter will dispatch |

## Consequences

- Adoption rounds-trips disclosed thinking levels: fetch → adopt → edit stays correct for OpenAI-style gateways and catalog routes.
- `LlmDiscoveredModel.reasoningEfforts` is part of the pre-stable Remote view; any adapter that discloses levels now surfaces them through the same field.
- The catalog-normalization rules are stated once in `reasoningCandidate`; pi-ai `thinkingLevelMap` defaulting asymmetries stay invisible to the user.
- The editor's summary tag and validation copy add to the Models page's locale dictionary and component tests; the adapter remains the final gate for what a request actually sends.

## Related

- [Draft provider endpoint interrogation](../architecture/2026-08-04-draft-provider-endpoint-interrogation.md) — where discovery candidates and their metadata rules live.
- [Twin LLM adapters](../architecture/2026-06-13-twin-llm-adapters.md) — the pi-ai adapter family and its reasoning vocabulary.