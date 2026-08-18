# Agent Note: Streaming tool-call deltas must not overwrite captured id/name with empty repeats

Status: implemented

English | [中文](2026-08-18-tool-call-empty-name-overwrite.zh.md)

## Problem

deepseek-v4-flash at high reasoning effort streams tool calls whose first delta carries the real `id` and `function.name`, but whose subsequent argument-fragment deltas repeat the fields with empty strings. The `!== undefined` guards in `translate` treated those empty strings as values, overwriting the captured id/name and assembling a tool-call block with `name: ""`. Dispatch then failed with `unknown tool ""`, and the failed call/result pair entered the session log, permanently polluting the session: replay sent the malformed call on every subsequent request, and the same replay against another model surfaced as an upstream `Duplicate value for 'tool_call_id'` rejection.

## Decision

The three guards in the `tool_calls` loop of `translate` (`packages/llm/llm-deepseek/src/translate.ts`) use truthy checks: `if (call.id)`, `if (call.function?.name)`, and `...block.name ? { name: block.name } : {}` on the emitted `tool-call-delta`. An empty-string repeat never overwrites an already-captured value, so the first non-empty id and name win. The fix lives in the streaming translation layer, not in dispatch: the wire defect is repetition of empty fields, and the earliest layer that observes it is where the value must be kept.

Streams that never carry an id or name at all keep the pre-existing empty-string fallback in `closeBlock` (`name: block.name ?? ''`) and `BlockAssembler` (`toolCallName ?? ''`); a genuinely nameless call still fails visibly as `unknown tool` at dispatch rather than being silently dropped.

## Alternatives considered

**Drop nameless tool-calls in the agent loop or tool registry.** Rejected: the first delta carries a real name that later empty repeats erase, so dropping the assembled block discards a legitimate call the model fully specified. It would also hide a translation-layer defect behind an execution-layer filter and contradict the model-visible-means-logged rule by discarding model output before it reaches the session log.

**Repair the name in `closeBlock` or `BlockAssembler`.** Rejected: the overwrite happens while deltas accumulate; downstream fallbacks can only turn `undefined` into `''`, not recover the real name that was already clobbered.

**Keep `!== undefined` and add explicit empty-string exclusions.** Rejected: unlike `arguments`, an empty `id` or `function.name` never carries information on this wire, so the truthy check is the complete condition; spelling out `!== undefined && !== ''` states the same thing twice.

## Consequences

- Empty-string repeats of `id`/`function.name` in later deltas are ignored; the first non-empty values stand.
- A `tool-call-delta` omits `name` while no non-empty name has arrived, instead of carrying `name: ''`; `BlockAssembler` already treats an absent and an empty name identically (`if (chunk.name)`), so consumers are unaffected.
- A stream that never supplies a non-empty name still assembles `name: ''` and fails at dispatch with `unknown tool` — visible failure, unchanged; this fix only removes the repeat-overwrite path that manufactured such streams from well-formed model output.

## Testing

`packages/llm/llm-deepseek/tests/translate.spec.ts` adds a regression case replaying the v4-flash shape — first delta with real id/name, later deltas repeating empty id/name over argument fragments — asserting the assembled block keeps `call_00_x`/`get_weather`. The pre-existing empty-string-fallback and single-delta tool-call cases are unchanged.
