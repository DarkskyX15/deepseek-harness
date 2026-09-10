# Agent Note: pi-ai request header templates

Status: implemented

English | [中文](2026-09-10-pi-ai-request-header-templates.zh.md)

## Problem

`llm-pi-ai` profiles can configure static `headers` for a provider route, and `GenerateOptions.sessionId` already reaches the adapter (the agent loop stamps the durable `Session.id` on every request). There was no way to tie them together: a deployment that wanted a session-scoped header (a gateway tracing header, a tenant header derived from the conversation) had to choose between a static value that cannot identify the session and writing a new adapter. Gateway support teams routinely ask for a correlation header that survives across the whole conversation, and only the live request carries the current session id.

## Decision

`dsh-llm-pi-ai` resolves `${sessionId}`, `${provider}`, and `${model}` templates in configured header values at request time, immediately before `requestHeaders()` merges attribution and strips reserved names. `resolveHeaderTemplates` substitutes the session id only when `options.sessionId` is present (empty otherwise), and keeps unknown tokens literal so a typo is visible on the wire instead of silently emptying. Profile headers remain deployment-owned: the same reserved-name collision policy still applies, and discovery probing keeps static headers because a probe has no session context.

Configuration is unchanged: the existing `headers` dict accepts the template spellings, documented on the profile field. A route declares `X-Session-Trace: dsh-${sessionId}` and every request on that route carries the resolved value.

## Alternatives considered

| Rejected | Reason |
|---|---|
| A separate `dsh-llm-request-headers` provider route | A second route duplicates the whole adapter for one header feature; templates on the existing routes cover every pi-ai provider with zero configuration migration |
| Substitute in the settings layer at write time | The session id is only known per request; a stored substitution would freeze a stale or empty value |
| Add a dedicated `sessionIdHeader` config field | A generic template vocabulary covers tracing, tenant, and correlation headers without a new field per use case |

## Consequences

- Any pi-ai provider route can carry a per-conversation header without a custom adapter.
- Header values containing a literal `${name}` for an unknown variable stay verbatim, so an accidentally misspelled token is diagnosed by the receiving gateway instead of silently vanishing.
- Attribution headers still win collisions; deployment headers cannot spoof Harness identity.
- Discovery probes do not resolve templates — they have no session — so a template header stays literal on `GET /models`; this matches the local-fork precedent the feature replaces.