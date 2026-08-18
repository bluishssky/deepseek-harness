# Agent Note: 流式 tool-call 增量不得以空串重复覆盖已捕获的 id/name

Status: implemented

[English](2026-08-18-tool-call-empty-name-overwrite.md) | 中文

## Problem

deepseek-v4-flash 在高推理档位下流式输出工具调用时，首个 delta 携带真实的 `id` 和 `function.name`，但后续参数分片 delta 会以空字符串重复这两个字段。`translate` 中的 `!== undefined` 守卫把这些空串当作值，覆盖了已捕获的 id/name，最终组装出 `name: ""` 的 tool-call 块。dispatch 随即以 `unknown tool ""` 失败，而失败的调用/结果对写入会话日志后永久污染该会话：此后每次请求的重放都会带上畸形调用，切换到其他模型后同一份重放还会在上游触发 `Duplicate value for 'tool_call_id'` 拒绝。

## Decision

`translate` 的 `tool_calls` 循环中三处守卫（`packages/llm/llm-deepseek/src/translate.ts`）改用真值判断：`if (call.id)`、`if (call.function?.name)`，以及发出的 `tool-call-delta` 上的 `...block.name ? { name: block.name } : {}`。空串重复不再覆盖已捕获的值，首个非空 id 和 name 生效。修复位于流式翻译层而非 dispatch 层：线路缺陷是空字段重复，最早观察到它的层就是必须保住该值的层。

从头到尾都不携带 id/name 的流仍保持 `closeBlock`（`name: block.name ?? ''`）与 `BlockAssembler`（`toolCallName ?? ''`）既有的空串兜底；真正无名的调用依旧在 dispatch 处以 `unknown tool` 可见地失败，而不是被静默丢弃。

## Alternatives considered

**在 agent loop 或工具注册表中丢弃无名 tool-call。** 已否决：首个 delta 携带了真实 name，只是被后续空串覆盖；丢弃组装后的块等于丢弃模型完整声明的合法调用。这还会用执行层过滤掩盖翻译层缺陷，并在模型输出进入会话日志之前将其丢弃，违背"模型可见即可从日志重建"的约定。

**在 `closeBlock` 或 `BlockAssembler` 中补救 name。** 已否决：覆盖发生在 delta 累积过程中；下游兜底只能把 `undefined` 变成 `''`，无法找回已被覆盖掉的真实 name。

**保留 `!== undefined` 并显式排除空串。** 已否决：与 `arguments` 不同，空的 `id` 或 `function.name` 在该线路上从不携带信息，真值判断已是完整条件；写成 `!== undefined && !== ''` 只是把同一件事说两遍。

## Consequences

- 后续 delta 中空串重复的 `id`/`function.name` 被忽略；首个非空值生效。
- 在非空 name 到达之前，`tool-call-delta` 省略 `name` 字段而非携带 `name: ''`；`BlockAssembler` 对缺失与空 name 本就同等处理（`if (chunk.name)`），消费方不受影响。
- 从不提供非空 name 的流仍组装出 `name: ''` 并在 dispatch 处以 `unknown tool` 失败——可见的失败，行为不变；本修复只消除了从完好模型输出制造这类流的"空串重复覆盖"路径。

## Testing

`packages/llm/llm-deepseek/tests/translate.spec.ts` 新增回归用例，重放 v4-flash 形态——首个 delta 带真实 id/name，后续 delta 在参数分片上重复空 id/name——断言组装后的块保留 `call_00_x`/`get_weather`。既有的空串兜底与单 delta 工具调用用例保持不变。
