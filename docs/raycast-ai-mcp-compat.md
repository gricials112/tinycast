# Raycast AI 与 MCP 兼容性：设计与修复记录

本文记录 `hark/raycast-ai-mcp`（AI.ask 兼容）与 `hark/mcp-fixes`（MCP 修复）两条分支做了什么、为什么这样做、
还有哪些没有在 macOS 上验证。英文的功能文档仍以 `docs/features/ai.md`、`docs/features/extensions.md`、
`docs/features/mcp.md` 为准；本文是中文的设计说明，只保留在 fork 中。

## 1. Raycast `AI.ask` 桥接

### 问题
上游运行时里 `AI.ask` 是一个直接 reject 的桩，`environment.canAccess(AI)` 永远返回 `false`。
凡是调用 AI 的 Raycast 扩展（包括基于 `@raycast/utils` 的 `useAI`）要么拒绝运行，要么直接抛错。

### 设计
- **JS 侧**（`Scripts/raycast-runtime/src/api/ai.js`）：`AI.ask(prompt, options)` 发起一次
  *流式* host call `ai.ask`。Swift 每产生一段文本就通过 `__tinycast.progress` 推回一个 chunk，
  最后用完整文本 settle。返回的 Promise 额外带 `.on("data")` / `.off()`，晚注册的监听器会先补收
  已到达的全部文本，因此 `await AI.ask(…)` 与 `.on("data")` 的行为都和 Raycast 一致。
  `AbortSignal` 触发时调用 `__tinycastHost.cancel` 取消 Swift 侧任务。
  `AI.Model` 由 `@raycast/api` 的类型定义生成（`enums.generated.js`）；遇到生成表里没有的名字，
  用 Proxy 推导出 `vendor-model` 形式的 id，保证至少能判断厂商。
- **宿主侧**：`ExtensionHostAPI` 增加带进度回调的 perform；`ExtensionRuntime` 在 JS 队列上把每个
  进度 payload 先于 settle 送达，并在 JS 取消时取消宿主任务。`ExtensionAIBridge` 把一次 `ai.ask`
  作为**单轮用户消息**交给 `AppCore` 解析出的 provider，逐段回传文本增量。
- **不使用 Raycast 托管模型**：回答一律来自读者在 Settings → AI 里自己配置的路由（API key、
  Codex、Claude、OpenRouter、OpenAI 兼容端点等）。AI 关闭或没有可用路由时，`canAccess(AI)` 为
  `false`，`AI.ask` 以“请在 Settings → AI 中开启”的错误结束。

### 模型映射（`RaycastAIModelMatch`，纯 Foundation）
1. `options.model` 若与读者某条路由的模型 id 完全相同（大小写不敏感），直接用它。
2. 否则从 Raycast id 解析厂商（`openai-gpt-4o-mini` → OpenAI；`groq-openai/…`、`gateway-deepseek/…`
   这类网关前缀之后的部分才代表厂商），在读者**同厂商**的路由里按 token 相似度挑最接近的一条：
   `haiku`/`mini`/`flash` 这类档位词权重高于版本号，家族名不计分，`claude-4-5-haiku` 与
   `claude-haiku-4.5` 视为相同。
3. 路由的厂商由 Settings 已知信息决定（Codex→OpenAI、Claude→Anthropic、Grok→xAI、Gemini→Google），
   网关类路由（OpenRouter、OpenAI 兼容）则从模型 id 推断。
4. 都匹配不上时用读者的默认路由。

### temperature 处理
- `creativity` 保持 Raycast 的 0–2 刻度（none 0、low 0.5、medium 1、high 1.5、maximum 2；数字则夹到 0–2）。
- `AIRequest` 新增可选 `temperature`：OpenAI 形状的请求体原样发送；Anthropic（0–1）发送其一半；
  普通聊天从不设置。
- GPT-5、o 系列等拒绝自定义 temperature 的模型：若错误信息命中 `rejectsTemperature` 且尚未输出任何
  文本，则**去掉 temperature 重试一次**。

## 2. MCP：根因与修复

| # | 根因（上游位置） | 修复 |
|---|---|---|
| 1 | `MCPCoordinator.invoke`（约 111–123 行）与 `AIChatCoordinator`（约 345 行）通过**解析** wire name（`slug__tool`）来路由。wire name 经过 ASCII 清洗与 64 字符截断：中文服务器名 `文件` 的 slug 清洗成 `__`，解析失败；长工具名被截断后，发给服务器的是截断后的名字 | 新增 `MCPToolRoutes` 查找表：wire name → (服务器 id, 工具原名)。`MCPServerManager.route(_:)` 由已就绪连接的工具列表构建；`invoke` 与聊天的按服务器开关都改为查表，调用时发送工具**原名**。被清洗或截断的名字追加原名的 FNV-1a 哈希，避免两个工具撞到同一 wire name |
| 2 | `MCPSlug` 只保留字母数字，中文名得不到可用 handle | 非 ASCII 名先音译再去变音符：`文件` → `wen-jian`，`Café` → `cafe`。旧版本已保存的非 ASCII slug 仍可路由（查表不依赖解析）；下次保存时 store 会重新规范化 |
| 3 | `MCPHTTPTransport`（约 266 行）`notifications/initialized` 以 fire-and-forget 发送，`tools/list` 可能先到服务器 | `notify` 改为 `async` 并被 await；握手顺序固定为 initialize → 记录协商版本 → await initialized → tools/list |
| 4 | 协议版本未协商，HTTP 请求不带 `MCP-Protocol-Version` | `MCPProtocol.negotiatedVersion` 取服务器回答的版本（仅限 2025-06-18 / 2025-03-26 / 2024-11-05，未知则回退）；`didNegotiate` 交给传输层，HTTP 之后每个请求都带该头 |
| 5 | 会话 404 只报错，不重建 | 有会话时收到 404 → `sessionExpired`；连接层**重新 initialize 一次**并重试原请求 |
| 6 | SSE 响应要等整个 body 读完才解析；服务器保持流不关闭时请求一直挂起 | 按字节增量读取，遇到换行即喂给 `SSEParser`，拿到匹配 id 的响应立即返回；途中的通知转发、`ping` 请求单独 POST 回复 |
| 7 | 没有旧版 HTTP+SSE（2024-11-05）回退 | 首个 POST（initialize、尚无会话）收到 400/404/405 → `streamableHTTPUnsupported`，连接改用新的 `MCPLegacySSETransport`：`GET` 同一 URL 打开事件流 → 等 `endpoint` 事件（30 s）→ 向该地址 POST，响应以 `message` 事件从流上返回。`endpoint` 必须与流同源（`MCPEventStream.endpoint`），否则拒绝，防止凭据被带到别处 |
| 8 | `MCPStdioTransport`（约 85 行）initialize 只有 15 s，首次 `npx -y` / `uvx` 下载依赖时超时 | 统一超时：initialize 120 s、tools/call 300 s、其他 30 s（`MCPProtocol.timeout(for:)`，两种传输共用） |
| 9 | 服务器发来的 `ping` 被当作未知请求拒绝 | `MCPProtocol.reply(toRequest:method:)`：`ping` 回空结果，其他请求仍拒绝（Tinycast 不对外提供能力） |
| 10 | `MCPServerConnection` 忽略 `tools/list` 分页 | 跟随 `nextCursor` 直到为空，最多 50 页；`list_changed` 通知触发的重新列举也走分页 |
| 11 | `start()` 被取消时直接 `return`，状态永远停在 “Connecting…” | 取消（或代次未变但任务已取消）时关闭传输、清空工具、回到 `.stopped`，下次进入聊天可重新启动 |

## 3. 配置导入（Import from Clipboard）

Settings → AI → MCP 新增 **Import from Clipboard** 按钮，并加入设置搜索目录（关键词 mcp、mcpservers、
json、paste、raycast、claude、cursor、vs code、config；`Scripts/check-settings-search.js` 通过）。

`MCPServerImport` 解析：
- `mcpServers`（Raycast、Claude Desktop、Cursor）、`servers`（VS Code）、`mcp_servers`、裸映射，或单个服务器对象；
- `command` + `args` + `env` → stdio 服务器，环境变量存入钥匙串；`"command": "npx -y pkg"` 且无 `args` 时自动拆分；
- `url` / `serverUrl` + `headers` → HTTP 服务器，保留 `Authorization`（没有则保留第一个头）；
- `disabled: true` 导入为停用；
- 命令或 URL 已存在的服务器跳过（重复粘贴不会重复添加）；放不下的字段（多余的头、`cwd`）在按钮下方列出，而不是静默丢弃。

## 4. 测试

- `mcp-test`（纯模型）：102 项，覆盖音译 slug、wire name 唯一性与查表路由、版本协商、分页游标、
  超时、ping 回复、命名 SSE 事件与同源检查、配置导入与去重。
- `mcp-stdio-test`：19 项，新增 ping、分页、16 s 慢启动三种 stub 模式。
- `mcp-http-test`（新）：26 项，Node stub `Tests/ai-fixtures/mcp-http-stub.js`，覆盖 JSON 响应、
  保持打开的 SSE 流（含中途 ping）、旧协议版本、404 会话过期、分页、旧版 HTTP+SSE 回退、
  跨源 endpoint 拒绝、中文服务器名路由、取消启动。
- `raycast-ai-test`：24 项。

以上均在 Linux（swiftc 6.0.3 + 小型 Foundation 垫片）上编译运行通过。

## 5. 尚未在 macOS 上验证

- 未用 Xcode 构建整个 App；`MCPCoordinator`、`AIChatCoordinator`、`MCPSettingsSection`、
  `ExtensionRuntime` 等依赖 AppKit/SwiftUI 的文件只经过人工审阅，未经编译器检查。
- `Tinycast.xcodeproj/project.pbxproj` 为手工登记新文件（Linux 上无法运行 xcodegen），建议在 Mac 上
  执行一次 `xcodegen generate` 确认无差异。
- macOS 的 `URLSession.bytes` 与 Linux 垫片行为可能不同（重定向代理、超时语义）；`applyingTransform(.toLatin)`
  在 macOS 上的音译结果应与 Linux 一致，但未实测。
- `mcp-oauth-test` 依赖 CryptoKit，Linux 上无法运行；OAuth 路径的代码未改动语义，但未回归。
- 真实服务器：未对 npx/uvx 首次启动、真实的 2024-11-05 SSE 服务器、真实 Raycast AI 扩展做端到端测试。

## 6. 尚未实现：扩展工具进入 AI 聊天（设计）

Raycast 的 “AI Extensions” 让扩展在 `package.json` 中声明 `tools`，在 AI Chat 中用 `@扩展名` 调用。
商店构建产物里已经包含所需的一切（以 Linear 扩展为例：54 个工具）：

- `package.json` → `tools[]`：`name`、`title`、`description`、`input`（JSON Schema，构建时由 TS 类型生成）、`output`；
- `package.json` → `ai`：`instructions`（给模型的系统指令）、`skills`、`evals`；
- `tools/<name>.js`：默认导出 `async (input) => result`，可选导出 `confirmation(input)`，返回需要用户确认的信息。

拟定方案：
1. **发现**：`ExtensionManager` 读取已安装扩展的 `tools` 与 `ai.instructions`，生成
   `ExtensionTool(extensionID, name, description, inputSchema)`。
2. **命名与路由**：与 MCP 共用一套机制——wire name 由扩展 handle 与工具名组合（同样清洗、截断、哈希），
   路由**只查表**（wire name → 扩展 id + 工具原名），不解析。`@handle` 作用域与聊天工具菜单的开关同样适用。
3. **执行**：在扩展运行时中新增 `tool.invoke` host call：加载 `tools/<name>.js`，以 `input` 调用默认导出，
   结果 JSON 序列化后作为 tool result 交回 `AIToolLoopProvider`。运行时沿用扩展已有的偏好设置、
   LocalStorage、OAuth 状态。
4. **确认**：若工具导出 `confirmation`，先调用它，把返回的说明与信息项放进与 MCP 相同的三选一对话框
   （Always Allow / Allow This Chat / Don't Allow）；否则按 MCP 的信任策略默认 “Ask Each Chat”。
5. **指令**：被 `@` 选中的扩展把 `ai.instructions` 附加进本轮 instructions；未选中时只提供工具描述。
6. **CLI 路由**：Codex/Claude 等自带工具循环的路由不能直接调用 JS，需要通过本地 MCP 桥（Tinycast 以 stdio
   MCP 服务器的形式暴露扩展工具）才能提供，可作为第二阶段。
7. **测试**：用 `evals` 中的 `mocks` 驱动 `ai-fixtures` 风格的离线测试，断言 `callsTool`。
