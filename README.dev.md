# dev 分支本地安装与启动指南（README.dev）

> 本文件只适用于**本地 fork 仓库**（`D:\Ds_Projects\TypeScript\deepseek-harness`，dev 分支）的安装与日常启动说明，与上游发布版行为无关。内容包含自制特性（pi-ai 思考档位发现与编辑器、请求头 `${sessionId}` 模板）与本地布局打磨的启动方式。

## 仓库即 dsh 本体

仓库 `apps/cli` 就是 `@deepseek-ai/dsh` 包（bin 指向已构建的 `apps/cli/lib/bin.js`，当前版本 `0.1.5-alpha.2`）。全部依赖是 pnpm workspace 协议，无法用 npm 全局安装；以下包装脚本等价于 `pnpm dsh`，但可从任意目录直接调用。

## 已安装的启动命令

全局 bin 目录（`D:\nodejs`，npm 全局前缀，已在 PATH）下创建了 4 个包装脚本：

| 命令 | 模式 | 用途 |
|---|---|---|
| `dsh-dev` | 构建产物（`apps/cli/lib/bin.js`） | 日常使用，启动快；改代码后需重新构建才生效 |
| `dsh-dev-src` | 源码直跑（tsx 加载 `apps/cli/src/bin.ts`） | 开发迭代，改源码立即生效，无需构建 |

两者都会自动：切到仓库目录 → 若环境变量 `DSH_HOME` 未设置，则默认使用 `%USERPROFILE%\.dsh-dev`（独立 home，不碰主实例 `C:\Users\Darksky\.dsh`）。

## 启动方式

```powershell
# 启动 Web UI（默认 DSH_HOME=%USERPROFILE%\.dsh-dev，浏览器自动打开）
dsh-dev web --port 3081

# 不自动打开浏览器（复制输出的 token URL 手动访问）
dsh-dev web --port 3081 --no-open

# 源码模式（改代码即时生效，无需 build）
dsh-dev-src web --port 3081 --no-open

# 其他 dsh 命令同理
dsh-dev --version                  # 0.1.5-alpha.2
dsh-dev --profile headless "任务"   # 无头任务
```

启动输出示例：

```
dsh web: http://127.0.0.1:3081/?token=<每次随机>
```

**token 每次启动都不同**（安全设计，无固定 token 参数）：自动打开浏览器时无需处理；`--no-open` 模式复制该 URL 或手动拼上 token 即可。

## 注意事项

1. **DSH_HOME 隔离**：脚本默认用 `~\.dsh-dev`，与 3080 主实例（`~\.dsh`）完全分离，互不干扰。需要其他 home 时：
   ```powershell
   $env:DSH_HOME = 'C:\Users\Darksky\.dsh-dev-015'
   dsh-dev web --port 3081
   ```
2. **端口**：不要使用 3080（主实例占用）；用 3081/3082/... 或 `--port 0` 让系统自动分配。启动前若端口被占，先确认没有残留实例（`Get-NetTCPConnection -LocalPort <port> -State Listen`）。
3. **构建模式 vs 源码模式**：
   - `dsh-dev` 运行最近一次 `pnpm run build` 的产物（dev 分支特性已构建，开箱即用）。
   - 修改仓库代码后：`dsh-dev-src` 立即反映；`dsh-dev` 需 `pnpm run build`，或仅重建单个 client 包 `pnpm --filter @deepseek-ai/dsh-client-ui-settings-models bundle`。
   - client bundle 改动必须重新打包对应包并**重启实例**（实例按 `lib/client.js` 发布，不热更新）。
4. **自制插件不在仓库内**：m3-theme、notification 等自制插件位于旧 `~\.dsh` 的 profile patch 层（`profiles/web/cordis.patch.yml`），不随仓库构建、也不随新 home 生效；如需要，复制 `cordis.patch.yml` 到 `~\.dsh-dev` 对应 profile。
5. **卸载**：删除 `D:\nodejs\dsh-dev.cmd|.ps1` 与 `dsh-dev-src.cmd|.ps1` 四个文件即可（不影响全局 `dsh` 0.1.0-rc.6）。

## 为什么不用 npm / pnpm 全局安装

`apps/cli` 的依赖全部是 `workspace:^` 协议（pnpm workspace），npm 无法全局安装；`npm link` 会把 `workspace:` 依赖解析失败。包装脚本等价于 `pnpm dsh`（其内部即 `node --import tsx/esm apps/cli/src/bin.ts`），且免去每次先 cd 进仓库。

正式发布走仓库 `pnpm run release:dsh`（bump 版本）后由 CI 发布 npm 包——但发布版不含 fork 特性，因此**保留本地包装脚本是继续使用自制改动的正确方式**。

## dev 分支自制特性一览

- pi-ai 思考档位：`LlmDiscoveredModel.reasoningEfforts` 发现透传 + Models 页逐模型思考强度编辑器（`packages/llm/llm`、`packages/llm/llm-pi-ai`、`packages/client/ui-settings-models`）
- pi-ai 请求头模板：`${sessionId}`/`${provider}`/`${model}` 请求时替换（`packages/llm/llm-pi-ai/src/adapter.ts`、`config.ts`）
- 对应 Agent Notes：`.agents/notes/implemented/feature/2026-09-10-pi-ai-reasoning-efforts-discovery-and-editor.{md,zh.md}`、`2026-09-10-pi-ai-request-header-templates.{md,zh.md}`