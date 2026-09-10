# Agent Note: pi-ai 思考档位发现与模型编辑器

Status: implemented

[English](2026-09-10-pi-ai-reasoning-efforts-discovery-and-editor.md) | 中文

## 问题

pi-ai 适配器已接受逐模型 `reasoningEfforts` 配置——非思考模型写 `false`，或等级字典把七个 pi-ai 等级映射为线上拼写——经由 `resolveModelReasoning` 物化，zod profile schema 也完成了校验。但两个界面都看不到它。「获取可用模型」返回的候选只携带 id、名称、上下文窗口与输出上限，因此端点上或目录里揭示了思考档位的模型（OpenAI 风格 `reasoningEfforts`、pi-ai 的 `thinking_levels`，或 `reasoning: false` 标记）在采纳时丢掉该元数据。Models 页的逐模型编辑器只有容量字段，用户必须手改 `settings.yaml` 才能声明自定义模型的可用思考档位；从发现起步的自定义网关工作流永远无法把端点宣告的档位带回配置。

## 决策

发现候选现在携带端点或目录揭示的档位，形状与配置 `reasoningEfforts` 完全一致；Models 页按模型行编辑它。

`dsh-llm-pi-ai` 的 `discoverModels` 把每条列表项的 `reasoningEfforts` 或 `thinking_levels` 字段读进候选的 `reasoningEfforts`（无法识别或空的字典、以及未揭示的能力保持缺席，候选绝不会携带其 profile 会拒绝的值），`reasoning: false` 标记变成 `false`。内置目录路径用 `reasoningCandidate` 归一化每个目录模型的 `thinkingLevelMap`：`null` 线上值表示不支持（键缺席），缺席键对 `off` 表示 `off: null`（发空），`xhigh`/`max` 保持缺席，其余基础等级以自身为拼写，字符串线上值原样携带。`LlmDiscoveredModel` 增加可选字段，Typert Remote 协议与 web 查看器的 `api-catalog` 声明原样传递。

`dsh-client-ui-settings-models` 以「思考型模型」总开关（`false` ↔ 等级字典）加每个等级一行（启用勾选 + 线上拼写输入）编辑该字段，位于每行模型的展开区内，行上附带显示已启用等级的小标签。采纳时若 `candidate.reasoningEfforts` 存在则一并写入；共享模型校验器按 `resolveModelReasoning` 的方式校验该值（非 off 等级必须有非空线上拼写，仅 `off` 可为 `null`，至少一个非 off 等级，等级必须已知）并逐行报告失败文案。所有产品文案都走 locale 字典。

## Alternatives considered

| 否决 | 理由 |
|---|---|
| 只在 `settings.yaml` 里编辑 `reasoningEfforts` | 发现工作流正是用户配置自定义网关的实际路径，Models 页是唯一同时抓取模型与写 profile 的界面；保留编辑器缺失就是把这则 note 要关闭的手改缺口带回来 |
| 原样携带原始列表对象到模型行，交给适配器在解析时校验 | 采纳后存储、profile schema 再拒绝的候选会把「获取」变成陷阱；编辑器在任何写入前校验，适配器的门禁仍是请求侧最终权威 |
| 复用 DeepSeek 编辑器的容量模式来编辑档位 | 档位是「布尔 + 线上字符串」的字典，不是 K/M 计数；专用控件逐级显式表达，用户看到的就是适配器将要发送的 |

## Consequences

- 采纳可往返揭示的思考档位：获取 → 采纳 → 编辑对 OpenAI 风格网关与目录路由都保持正确。
- `LlmDiscoveredModel.reasoningEfforts` 属于 pre-stable Remote 视图的一部分；任何揭示档位的适配器现在都通过同一字段呈现它们。
- 目录归一化规则集中写在 `reasoningCandidate`；pi-ai `thinkingLevelMap` 的不对称默认行为对用户保持不可见。
- 编辑器的小标签与校验文案进入 Models 页的 locale 字典与组件测试；适配器仍是实际发送内容的最终门禁。

## 相关

- [草稿提供方端点询问](../architecture/2026-08-04-draft-provider-endpoint-interrogation.zh.md)——发现候选及其元数据规则所在。
- [双子 LLM 适配器](../architecture/2026-06-13-twin-llm-adapters.zh.md)——pi-ai 适配器家族与它的推理词汇。