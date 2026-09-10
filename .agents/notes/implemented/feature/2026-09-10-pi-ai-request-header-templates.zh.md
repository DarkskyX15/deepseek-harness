# Agent Note: pi-ai 请求头模板

Status: implemented

[English](2026-09-10-pi-ai-request-header-templates.md) | 中文

## 问题

`llm-pi-ai` profile 可以给提供方路由配置静态 `headers`，且 `GenerateOptions.sessionId` 已经到达适配器（agent loop 在每个请求上盖上持久化的 `Session.id`）。但两者无法结合：想要会话级请求头的部署（网关追踪头、由对话派生的租户头）只能在「无法标识会话的静态值」与「新写一个适配器」之间二选一。网关支持团队通常要求一个贯穿整个对话的相关请求头，而只有活请求携带当前会话 id。

## 决策

`dsh-llm-pi-ai` 在请求时解析配置头值里的 `${sessionId}`、`${provider}`、`${model}` 模板，紧接着在 `requestHeaders()` 合并 attribution 并剔除保留名前执行。`resolveHeaderTemplates` 仅在 `options.sessionId` 存在时替换会话 id（否则为空），未知 token 保持字面量，让笔误在线上可见而不是静默清空。Profile 头仍属部署所有：相同的保留名冲突策略继续生效，发现探测保持静态头，因为探测没有会话上下文。

配置不变：现有 `headers` 字典接受模板拼写，已在 profile 字段上记录。路由声明 `X-Session-Trace: dsh-${sessionId}` 后，该路由的每个请求都携带解析后的值。

## Alternatives considered

| 否决 | 理由 |
|---|---|
| 单独的 `dsh-llm-request-headers` 提供方路由 | 第二条路由为单一请求头特性复制整个适配器；在现有路由上做模板覆盖所有 pi-ai 提供方且配置零迁移 |
| 在 settings 层写入时替换 | 会话 id 只能在请求时知道；存入的替换会冻结为过期或空值 |
| 增加专用 `sessionIdHeader` 配置字段 | 通用模板词汇覆盖追踪、租户、相关头，无需为每个用例新增字段 |

## Consequences

- 任意 pi-ai 提供方路由都能携带逐会话请求头，无需自定义适配器。
- 未知变量名的字面 `${name}` 保持原样，拼错的 token 由接收网关诊断而不是静默消失。
- Attribution 头仍然赢得冲突；部署头不能伪造 Harness 身份。
- 发现探测不解析模板——它们没有会话——因此 `GET /models` 上的模板头保持字面量；这与本特性取代的本地 fork 先例一致。