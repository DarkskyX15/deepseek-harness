# Agent Note: pi-ai model modalities in discovery and the model editor

Status: implemented

English | [中文](2026-09-10-pi-ai-model-modalities-discovery-and-editor.zh.md)

## Problem

Both adapters already decide image input from a declared modality list — `dsh-llm-pi-ai` resolves a profile entry's `input` against the installed catalog and then the route's `defaultInput`, and `dsh-llm-deepseek` validates `inputModalities` on every catalog entry — and the runtime acts on that declaration: a route without `image` has each request image replaced by a text placeholder, `read_image` refuses, and a submitted prompt carrying an image is rejected. Two surfaces could not state the claim. "Fetch available models" returned candidates carrying id, name, context window, output cap, and thinking levels only, so a catalog vision model was adopted text-only and its images silently became placeholders. Every row of the Models page offered capacities and thinking levels, so declaring modalities meant hand-editing `settings.yaml`. The Remote seam dropped the field a third time: `LlmRuntime.discoverModels` rebuilds each candidate field by field, and that list had never carried `reasoningEfforts` either.

## Decision

Discovery candidates now carry the modalities their source discloses, and the Models page edits them per row.

`LlmDiscoveredModel` gained the optional `inputModalities`. `dsh-llm-pi-ai`'s `discoverModels` answers a catalog route through `modalityCandidate`, which filters the catalog model's `input` through the profile vocabulary, and reads an interrogated listing's `architecture.input_modalities`, `input_modalities`, or `modalities` through `listingModalities`, which keeps only values a pi-ai profile may declare — OpenRouter's `file`, `audio`, and `video` are dropped — and reads no `architecture.modality` string, whose grammar is a vendor convention rather than a disclosure this build can restate. `LlmRuntime.discoverModels` now carries both `inputModalities` and `reasoningEfforts` through its per-field rebuild, so a candidate field reaches the surface in the same change that declares it.

The Models page renders one checkbox per modality inside each row's disclosure, for pi-ai profiles and the DeepSeek catalog alike. An undeclared row shows the `text` both adapters default to as ticked, so ticking image adds to the text a request already carries instead of replacing it; clearing every box removes the field rather than storing an empty list, which pi-ai reads as "accepts nothing" and the DeepSeek schema refuses. The DeepSeek editor validates a hand-edited value the way its adapter will (non-empty, unique, `text`/`image` only). Adoption copies `candidate.inputModalities` when present. Each form writes the field its own adapter reads — a pi-ai entry spells the list `input`, the DeepSeek catalog spells it `inputModalities`, and adoption renames the candidate's field to the writing form's spelling. Product copy is locale-owned, and the two editors share one control and one pair of layout classes.

## Alternatives considered

| Rejected | Reason |
|---|---|
| Edit `inputModalities` only in `settings.yaml` | Discovery is the path users take for custom gateways, and the Models page is the only surface that both fetches models and writes profiles; keeping the editor out reintroduces the hand-edit gap this note closes |
| One dropdown (`unset` / text / text+image) | A third modality would need a new option and a new grammar; the checkbox group mirrors the thinking-level editor, where a new level is one more row |
| Parse OpenRouter's `architecture.modality` string (`text+image->text`) | Its grammar is a vendor convention, and a guess would store a claim the endpoint never made in the field the user is about to save |
| Show an undeclared row with every box clear | Ticking image would then store `['image']` and silently drop the text every request carries; showing the shared default keeps the first edit additive |

## Consequences

- Adoption round-trips disclosed modalities: fetch → adopt → edit keeps a catalog vision model image-capable instead of leaving the route default in force.
- `LlmDiscoveredModel.inputModalities` is part of the pre-stable Remote view, and the web viewer's `api-catalog` declaration carries it; any adapter that discloses modalities surfaces them through the same field.
- The per-field rebuild in `LlmRuntime.discoverModels` is now the stated home for candidate fields, and it also repairs the thinking levels that discovery had been losing there.
- Written modalities are a claim about the endpoint, not a check of it: a model claiming images its gateway refuses is still refused by the provider mid-turn.

## Related

- [Unified image request pipeline](2026-08-20-unified-image-request-pipeline.md) — how an accepted image becomes a per-route request version.
- [pi-ai reasoning-efforts discovery and model editor](2026-09-10-pi-ai-reasoning-efforts-discovery-and-editor.md) — the same discovery-plus-editor pattern for another capability field.
- [Draft provider endpoint interrogation](../architecture/2026-08-04-draft-provider-endpoint-interrogation.md) — where discovery candidates and their metadata rules live.
