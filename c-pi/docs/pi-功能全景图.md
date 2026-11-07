# Pi 仓库功能全景图

> 学习对象：earendil-works/pi（TypeScript monorepo，npm workspaces，Node ≥ 22.19.0）。
> 本文是动手复刻前的「全局认知」底稿：仓库有哪些包、各自做什么、CLI 有哪些用户可见功能、怎么跑起来、数据存哪里。
> 来源标注为 `文件路径:行号`，可在 `pi22/` 中直接核对。

## 1. 顶层结构

### 1.1 包清单

| 包 | 目录 | npm name | 作用 | 入口 |
| --- | --- | --- | --- | --- |
| 主 CLI | `packages/coding-agent` | `@earendil-works/pi-coding-agent` | Coding agent CLI（read/bash/edit/write 工具 + 会话管理） | `src/cli.ts` |
| Agent 核心 | `packages/agent` | `pi-agent-core` | 通用 agent：传输抽象、状态管理、attachment | `src/agent.ts` |
| 统一 LLM API | `packages/ai` | `pi-ai` | 多 provider LLM API + 自动模型发现 | `src/index.ts` |
| TUI 库 | `packages/tui` | `pi-tui` | 差分渲染终端 UI 库 | `src/index.ts` |
| Chord | `packages/chord` | `@earendil-works/chord` | 跨进程对象运行时（contracts/documents/store/plugins/HTTP/workers） | 多子路径 exports |
| Durable | `packages/durable` | `pi-durable` | 可持久化对话/任务/文档运行时（Experimental） | `src/index.ts` |
| MCP 客户端 | `packages/mcp` | `pi-mcp` | 独立 MCP 客户端（不依赖官方 SDK） | `src/index.ts` |
| 远程执行环境 | `packages/env` | `pi-env` | SSH daemon + ExecutionEnv 客户端（daemon 为 Rust） | `src/index.ts` |
| JSON Schema | `packages/schema` | `pi-schema` | 无 eval 的 JSON Schema 校验（draft 2020-12 / 7） | `src/index.ts` |
| 遥测 | `packages/telemetry` | `pi-telemetry` | 厂商中立遥测契约 | `src/index.ts` |
| Codemode | `packages/codemode` | `pi-codemode` | QuickJS 沙箱执行模型写的 JS，唯一能力是调工具 | `src/index.ts` |
| Evals | `packages/evals` | `pi-evals` | 评测框架（private） | `src/cli.ts` |
| p2 应用 | `apps/p2` | `p2` (private) | 自托管 Pi Durable Web UI（实验性原型） | `src/hosts/server.ts` |

### 1.2 依赖关系

```
telemetry ── (无内部依赖)
schema   ─── (无内部依赖)
chord ────── pi-schema, pi-telemetry
ai ───────── pi-schema, pi-telemetry + 各家 SDK
codemode ─── (无内部依赖; 仅 quickjs-wasi)
mcp ──────── (无内部依赖; 仅 cross-spawn)
tui ──────── (无内部依赖)
durable ──── chord, pi-ai, pi-codemode, pi-schema, pi-telemetry
env ──────── chord, pi-durable
agent ────── pi-ai, pi-schema
coding-agent ─ pi-agent-core, pi-ai, pi-codemode, pi-mcp, pi-tui
```

构建顺序（`pi22/package.json:16`）：telemetry → schema → chord → tui → codemode → mcp → ai → durable → env → agent → coding-agent，与依赖图一致。

## 2. coding-agent 用户可见功能清单

### 2.1 CLI 用法（`packages/coding-agent/docs/cli.md`）

- **命令总览**（`cli.md:7-19`）：`pi [options] [messages...]`、`pi install/remove/update/list/config`、`pi auth login/logout/check/...`、`pi mcp add/remove/list/login/logout`。
- **模式**（`cli.md:23-53`）：默认 TUI；`--print` 一次性输出；`--mode json` JSONL 事件流；`--mode rpc` 长驻 JSONL 进程；终端重定向时自动 print 模式。`@path` 附加文件、管道 stdin 作为首条 prompt、`--export` 导出 HTML。
- **模型选项**（`cli.md:57-76`）：`--provider`、`--model`（模糊匹配/`provider/id`/`:<thinking>` 后缀）、`--api-key`、`--thinking`、`--models`、`--list-models`。
- **会话选项**（`cli.md:80-109`）：`-c/--continue`、`-r/--resume`、`--session`、`--fork`、`--session-dir`、`--no-session`、`-n/--name`。
- **工具选项**（`cli.md:113-184`）：内建工具 read / bash / powershell(Win) / edit / write / grep / find / ls（默认 read,bash,edit,write）；内建扩展工具 codemode、tool_search（默认关）；`-t/--tools` 增量语法 `+name`/`-name`/`*`、`-xt`、`--no-builtin-tools`。
- **资源选项**（`cli.md:188-219`）：`-e/--extension`、`-ne`、`--no-mcp`、`--skill`、`--no-skills`、`--prompt-template`、`--theme`、`--use-theme`、`--no-themes`、`-nc/--no-context-files`。
- **其他**（`cli.md:223-250`）：`--system-prompt`、`--append-system-prompt`、`--tui-mode`、`--verbose`、`-a/--approve`、`--offline`。

### 2.2 交互模式（`docs/usage.md`）

- 界面：transcript（对话/工具调用）+ 编辑器 + footer（目录/会话/模型/上下文用量/费用）。
- 输入：`Enter` 发送、`Shift+Enter` 换行、`Ctrl+G` 外部编辑器、`@` 搜文件、`Tab` 补路径、粘贴/拖拽图片。
- 运行中引导：`Enter` steering、`Alt+Enter` follow-up 排队、`Alt+Up` 退回、`Escape` 中止。
- shell 集成：`!cmd` 运行并送入对话；`!!cmd` 运行不送模型。
- 复制/导出/分享：`Ctrl+X`/`/copy`、`/export`、`/share`。

### 2.3 斜杠命令全集（`docs/slash-commands.md:7-53`）

- 模型与设置：`/settings`、`/model`、`/thinking`、`/scoped-models`、`/login`、`/logout`、`/llama`
- 会话与上下文：`/new`、`/resume`、`/name`、`/session`、`/tree`、`/fork`、`/clone`、`/compact`、`/import`
- 导出与分享：`/copy`、`/export`、`/share`、`/bug`
- 运行时与项目：`/trust`、`/mcp`、`/reload`、`/hotkeys`、`/changelog`、`/quit`
- 扩展可注册自定义命令；skill 以 `/skill:name` 出现。

### 2.4 扩展机制（`docs/extensions.md`）

- TypeScript 模块 default 导出 factory，接收 `ExtensionAPI`，jiti 加载免编译（`extensions.md:186-213`）。
- 能力：注册工具、拦截/修改工具调用、注入上下文、自定义 compaction、`ctx.ui` 交互、自定义 TUI 组件、注册 `/` 命令、会话持久化状态、注册 provider/虚拟模型/MCP server。
- 加载位置：`~/.pi/agent/extensions/`、`.pi/extensions/`、`pi.extensions` manifest、settings.json（`extensions.md:123-136`），支持 `/reload` 热重载。
- 事件生命周期：`input → before_agent_start → agent_start → turns → agent_end → agent_before_settle → agent_settled`；工具链 `tool_execution_start → tool_call(可block) → tool_execution_update → tool_result → tool_execution_end`（`extensions.md:311-407`）。
- 内建扩展：`builtin:mcp`、`builtin:llama.cpp`、`builtin:codemode`、`builtin:tool-search`（`docs/settings.md:164`）。

### 2.5 RPC（`docs/rpc.md`、`rpc-commands.md`）

- `pi --mode rpc` 启动；stdin Command / stdout response + Session event / Extension UI 四类记录；严格 JSONL framing。
- 命令族：prompt/steer/follow_up/abort、get_state/get_messages、set_model/cycle_model、set_thinking_level、set_steering_mode/set_follow_up_mode、compact、set_auto_retry、bash、session 族（switch/fork/clone/get_tree/export_html...）、get_commands。
- 关键语义：`agent_end` ≠ 空闲，须等 `agent_settled`（`rpc.md:58-71`）。

### 2.6 SDK（`docs/sdk.md`）

- `createAgentSession()` → `AgentSession`，`session.prompt()` / `dispose()`。
- 可替换边界：modelRuntime、settingsManager、sessionManager、resourceLoader、tools。
- 事件订阅：`message_update` 流式、`message_end` 权威、`agent_settled` 最终空闲。
- `AgentSessionRuntime`：`newSession()` / `switchSession()` / `fork()` / `importFromJsonl()`。

### 2.7 Skills（`docs/skills.md`）

- 实现 Agent Skills 规范：`SKILL.md`（frontmatter name/description 等）+ scripts/references/assets。
- 渐进加载：启动只注入 name/description，模型任务匹配时才读全文；`/skill:name args` 强制加载。
- 发现位置：`~/.pi/agent/skills/`、`.pi/skills/`、`~/.agents/skills/`、`.agents/skills/`。

### 2.8 Prompt templates（`docs/prompt-templates.md`）

- Markdown 文件 → `/命令名`：`~/.pi/agent/prompts/*.md` 或 `.pi/prompts/`；变量 `$1..$n`、`$@`、`${1:-default}` 等 shell 风格替换。

### 2.9 Themes（`docs/themes.md`）

- 内建 `system`（从终端 ANSI 推导，WCAG 4.5:1）、`dark`、`light`；自定义 `<agent-dir>/themes/<name>.json`，支持 hex/OKLCH/OKHSL/256 色。

### 2.10 Pi Packages（`docs/packages.md`）

- 把 extensions/skills/prompts/themes 打包经 npm/git/本地分发；`pi install`、`--local` 写项目 `.pi/settings.json`、`-e npm:pkg` 试用。

### 2.11 其他子系统

- **codemode**（`docs/codemode.md`）：QuickJS 沙箱（无 Node API/网络/定时器），globals `tools`、`text()`、`image()`、`store()/load()`、`searchTools()`（BM25）等。
- **MCP**（`docs/mcp.md`）：stdio + streamable HTTP；`mcp.json` 配置；工具命名 `mcp__<server>__<tool>`；exposure 控制（direct/codemode/deferred）。
- **compaction**（`docs/compaction.md`）：接近上下文上限自动压缩；`compaction.reserveTokens` 默认 16384；扩展可自定义总结。
- **virtual-models**（`docs/virtual-models.md`）：虚拟模型路由状态存 custom entry。
- **llama.cpp**（`docs/llama-cpp.md`）：`/llama` 管理本地 GGUF router。

## 3. 其他核心包职责

### 3.1 `packages/agent`（pi-agent-core）

有状态 agent：拥有 transcript、发生命周期事件、执行工具、steering/follow-up 排队（`src/agent.ts:188-194`）。核心抽象：`AgentMessage` vs LLM Message 的转换/过滤（`README.md:43-59`）；事件流 `agent_start → turn_start → message_start/update/end → turn_end → agent_end`；依赖注入 `streamFn` + `initialState`。源码仅 6 个文件。

### 3.2 `packages/ai`（pi-ai）

统一 LLM API：provider 集合、auth 解析、token/费用追踪、跨 provider 会话交接。支持的 provider（`README.md:62-98`）：OpenAI、Azure OpenAI、OpenAI Codex、Anthropic、Google、Vertex、Bedrock、Mistral、Groq、Cerebras、Cloudflare、xAI、OpenRouter、DeepSeek、NVIDIA NIM、Together、Baseten、HuggingFace、Moonshot、GitHub Copilot、OpenCode、Fireworks、Kimi、Qwen、MiniMax、ZAI、Xiaomi MiMo 等，及任意 OpenAI 兼容端点（Ollama/vLLM/LM Studio）。关键模块：`models.ts`/`models-store.ts`/`model-catalog.ts`、`oauth.ts`、`env-api-keys.ts`、`images.ts`。

### 3.3 `packages/tui`（pi-tui）

差分渲染终端 UI：`TuiMainScreen`（主缓冲保留 scrollback）与 `TuiAltScreen`（备用缓冲）；只更新变化的行；CSI 2026 同步输出防闪烁；kitty/iterm2 内联图片。组件：Text/Input/Editor/Markdown/Loader/SelectList/Box/ScrollView 等；native 剪贴板 NAPI 模块（含 win32）。

### 3.4 `packages/chord`

跨进程对象运行时：契约（JSON Schema）、facet、文档（事务写入）、derives、jobs、principal、link（memory/ipc/webSocket）、插件热重载。子系统对应 `src/` 的 contracts/documents/host/jobs/link/registries/remote/store/state/sync/sqlite/web/react/tui/delta。

### 3.5 `packages/durable`（pi-durable，Experimental）

Durable agent harness：对话/turn/工具调用先提交存储再展示；崩溃恢复并计失败重试。用法 `Harness.open(storage)` → `root(context)` → `submit(...)` → `submission.wait()`。工具集与 coding-agent 平行（read/write/edit/bash/image/env）；存储后端可插拔（memory/indexeddb/jsonl/sqlite）。

### 3.6 `packages/mcp`（pi-mcp）

独立 MCP 客户端：`McpClient` + 传输层（stdio/streamable-http/pipe/in-memory）；`toLlmContent()` 结果转换；完整 OAuth（discovery/flow/loopback/callback）。

### 3.7 `packages/env`（pi-env）

SSH 远程执行：Rust daemon（stdin/stdout 帧协议）、`RemoteExecutionEnv`、`connectSsh`（BatchMode、严格 host-key 校验）、`openTerminal()` 伪终端、`openProcess()` 管道进程。

### 3.8 `packages/schema`（pi-schema）

无 eval JSON Schema 校验器：draft 2020-12/7 编译为普通闭包，官方套件 2222/2223 通过。`compiler.ts`/`validator.ts`/`builders.ts`/`coerce.ts`（工具参数矫正用）。

### 3.9 `packages/telemetry`（pi-telemetry）

厂商中立遥测契约：TelemetryContext/Span 回调、跨进程传播 carrier、metrics；不含 exporter。

### 3.10 `packages/codemode`（pi-codemode）

QuickJS(WASM) 沙箱：三种实现 `CodemodeSandbox`（worker thread）/`RemoteCodemodeSandbox`（Cloudflare DO）/`InlineCodemodeSandbox`。脚本只有 `tools` 等注入能力，无 timer/fetch/process。

## 4. 运行方式

- 安装：`npm install --ignore-scripts`（根 `package.json`）；构建：`npm run build`（联网刷新模型数据）/ `npm run build:offline`（`package.json:16-17`）。
- 检查：`npm run check`（biome + pinned-deps + tsc --noEmit 等，`package.json:21`）。
- 测试：`npm test`、`./test.sh`（跳过无 API key 的 LLM 测试）、`test:mcp-conformance`。
- **从源码跑 CLI**：`pi-test.ps1`（Windows）/ `pi-test.sh`：清掉 provider API key 环境变量（`--no-env`）后，用 `node --import packages/coding-agent/src/experimental/source-resolver.ts packages/coding-agent/src/cli.ts` 直接跑 TypeScript 源码，无需构建。
- 需要模型凭据：`/login`（OAuth 或 API key）、`pi auth login <provider> --api-key`、环境变量 `ANTHROPIC_API_KEY` 等；凭据存 `~/.pi/agent/auth.json`。

## 5. 数据/状态存储

- agent 目录默认 `~/.pi/agent`（`PI_CODING_AGENT_DIR` 可改，`docs/configuration.md:9`）：settings.json、keybindings.json、mcp.json、models.json、auth.json、AGENTS.md/SYSTEM.md、extensions/skills/prompts/themes/；项目级 `.pi/` 同构，加载前需 project trust。
- 会话：`~/.pi/agent/sessions/` 下 JSONL（版本 3），路径编码 `--<path>--/<timestamp>_<session-id>.jsonl`；条目经 id/parentId 组成树；类型含 session/message/model_change/usage/compaction/context_edit/branch_summary/custom/label 等（`docs/session-format.md:62-207`）。
- 上下文重建：`buildContextEntries()`（应用 compaction）→ `buildSessionProjection()`（应用 context_edit）→ `buildSessionContext()`（产出 LLM 消息，`session-format.md:224-248`）。
- bash 工具注入给子 shell：`PI_SESSION_ID`、`PI_PROVIDER`、`PI_MODEL`、`PI_REASONING_LEVEL` 等（`docs/environment-variables.md:20-30`）。

## 6. 复刻学习切入点（初步）

1. 最小骨架链：`schema`（无依赖）→ `telemetry` → `ai` → `agent` → `tui` → `coding-agent`，按依赖序逐个复刻。
2. 协议契约先行：`docs/session-format.md`、`docs/message-types.md`、`rpc-commands.md` 是精确的可执行规格。
3. 高价值子系统：`packages/codemode`（约 10 个文件）与 `coding-agent/src/core/extensions/`（runner/jiti loader/wrapper/virtual-modules）。
