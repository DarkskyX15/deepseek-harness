# Agent Note：pi-ai 模型模态的发现与模型编辑器

Status: implemented

[English](2026-09-10-pi-ai-model-modalities-discovery-and-editor.md) | 中文

## Problem

两个 adapter 早已依据声明的模态列表判定图像输入——`dsh-llm-pi-ai` 把 profile 条目的 `input` 依次对内置 catalog 与 route 的 `defaultInput` 求解，`dsh-llm-deepseek` 则校验每个 catalog 条目的 `inputModalities`——运行时也据此行动：未声明 `image` 的 route，其每个请求图像都会被替换为文本占位符，`read_image` 拒绝执行，携带图像的提交也会被拒。但有两个界面无法表达这一声明。「获取可用模型」返回的候选只带 id、名称、上下文窗口、输出上限与思考强度，于是 catalog 里的视觉模型被以纯文本形态采纳，其图像静默变成占位符；Models 页的每一行只提供容量与思考强度，要声明模态只能手改 `settings.yaml`。Remote 边界第三次丢掉了该字段：`LlmRuntime.discoverModels` 逐字段重建每个候选，而这份清单此前也从未带上 `reasoningEfforts`。

## Decision

发现候选现在携带其来源披露的模态，Models 页逐行编辑它们。

`LlmDiscoveredModel` 新增可选的 `inputModalities`。`dsh-llm-pi-ai` 的 `discoverModels` 对 catalog route 经 `modalityCandidate` 作答——它把 catalog 模型的 `input` 按 profile 词汇表过滤；对被询问的 listing，则经 `listingModalities` 读取 `architecture.input_modalities`、`input_modalities` 或 `modalities`：只保留 pi-ai profile 可以声明的取值（OpenRouter 的 `file`、`audio`、`video` 被丢弃），且不读取 `architecture.modality` 字符串——其语法属厂商约定，不是本构建能够复述的披露。`LlmRuntime.discoverModels` 现在把 `inputModalities` 与 `reasoningEfforts` 一并带过它的逐字段重建，因此候选字段在声明它的同一次改动中抵达界面。

Models 页在每个模型行的展开区渲染「每个模态一个复选框」，pi-ai profile 与 DeepSeek catalog 一致。未声明的行把两个 adapter 共同的默认值 `text` 显示为已勾选，于是勾选 image 是在请求本就携带的文本上追加，而不是替换它；全部取消勾选则删除该字段，而不是存入空列表——pi-ai 把空列表读作「什么都不接受」，DeepSeek 的 schema 也拒绝它。DeepSeek 编辑器按该 adapter 的规则校验手改的值（非空、唯一、仅 `text`/`image`）。采纳时在候选带有 `inputModalities` 时复制该字段。两个表单各自写入自己 adapter 读取的字段——pi-ai 条目把该列表拼作 `input`，DeepSeek catalog 拼作 `inputModalities`，采纳时把候选字段改写成写入侧表单的拼写。产品文案归 locale 所有，两个编辑器共用同一个控件与同一组布局类。

## Alternatives considered

| 已否决 | 理由 |
|---|---|
| 只在 `settings.yaml` 中编辑 `inputModalities` | 自定义网关用户实际走的是发现路径，而 Models 页是唯一既能抓取模型又能写入 profile 的界面；把编辑器排除在外会重新引入本 note 要关闭的手改缺口 |
| 用单个下拉框（`未设置`／文本／文本+图像） | 增加第三种模态就要新增选项与语法；复选框组与思考强度编辑器同构，新增一个模态只是多一行 |
| 解析 OpenRouter 的 `architecture.modality` 字符串（`text+image->text`） | 其语法属厂商约定，猜测会把端点从未作出的声明写进用户即将保存的字段 |
| 未声明的行让所有复选框都留空 | 那样勾选 image 会存入 `['image']`，静默丢掉每个请求本就携带的文本；显示共同默认值让第一次编辑是追加式的 |

## Consequences

- 采纳闭环保留披露的模态：获取 → 采纳 → 编辑，使 catalog 视觉模型保持图像能力，而不是让 route 默认值继续生效。
- `LlmDiscoveredModel.inputModalities` 属于预稳定的 Remote 视图，Web 阅读端的 `api-catalog` 声明同样携带它；任何披露模态的 adapter 都经同一字段呈现。
- `LlmRuntime.discoverModels` 的逐字段重建由此成为候选字段的既定归属地，并顺带修复了发现流程一直在那里丢失的思考强度。
- 写入的模态是对端点的声明而非校验：声称支持图像而网关拒绝的模型，仍会在请求中途被 provider 拒绝。

## Related

- [统一图像请求管线](2026-08-20-unified-image-request-pipeline.zh.md)——被接受的图像如何成为按 route 的请求版本。
- [pi-ai 思考强度发现与模型编辑器](2026-09-10-pi-ai-reasoning-efforts-discovery-and-editor.zh.md)——同一套「发现 + 编辑器」模式用于另一个能力字段。
- [草稿 provider 端点询问](../architecture/2026-08-04-draft-provider-endpoint-interrogation.zh.md)——发现候选及其元数据规则的归属。
