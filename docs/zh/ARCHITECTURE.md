# Pi 项目架构概览

## Pi 是什么

Pi 是一套用于构建 AI agent 和管理 LLM 部署的开源工具集，采用 monorepo 结构组织，由 7 个包组成。其核心产品是一个终端交互式编程 agent（`pi-coding-agent`），支持通过扩展、技能、提示模板和主题进行深度定制。

Pi 的设计哲学是**极致可扩展、核心极简**。它不内置子 agent、计划模式、权限弹窗等功能，而是通过扩展机制让用户按需构建或安装第三方包。

## 整体架构

```
┌────────────────────────────────────────────────────────────────┐
│                        用户交互层                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ coding-agent │  │   web-ui     │  │        mom           │  │
│  │ (终端 CLI)    │  │ (浏览器组件)  │  │   (Slack 机器人)      │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
├─────────┼─────────────────┼──────────────────────┼─────────────┤
│         │          Agent 运行时层                  │             │
│         └─────────────┐   │   ┌──────────────────┘             │
│                  ┌────▼───▼───▼────┐                           │
│                  │   agent-core    │                            │
│                  │ (agent 循环、    │                            │
│                  │  工具调用、      │                            │
│                  │  状态管理)       │                            │
│                  └────────┬────────┘                            │
├───────────────────────────┼────────────────────────────────────┤
│                     LLM 抽象层                                  │
│                  ┌────────▼────────┐                            │
│                  │     pi-ai       │                            │
│                  │  (统一多提供商   │                            │
│                  │   LLM API)      │                            │
│                  └────────┬────────┘                            │
│                           │                                    │
│            ┌──────────────┼──────────────┐                     │
│            ▼              ▼              ▼                      │
│        OpenAI        Anthropic       Google    ...更多提供商     │
├────────────────────────────────────────────────────────────────┤
│                       基础设施层                                │
│  ┌──────────────┐                    ┌──────────────┐          │
│  │     tui      │                    │     pods     │          │
│  │ (终端 UI 库)  │                    │ (GPU 部署管理) │          │
│  └──────────────┘                    └──────────────┘          │
└────────────────────────────────────────────────────────────────┘
```

## 各包详解

### 1. pi-ai — 统一 LLM API 层

**包名**: `@mariozechner/pi-ai`

**作用**: 提供一个统一的多提供商 LLM 接口，屏蔽各家 API 的差异。

**核心功能**:
- **流式输出与完整输出**: 通过 `stream()` 和 `complete()` 函数与 LLM 交互
- **自动模型发现**: 内置各提供商可用模型列表，仅包含支持工具调用（function calling）的模型
- **工具定义**: 使用 TypeBox schema 定义工具参数，支持类型安全和验证
- **跨提供商切换**: 上下文（Context）可序列化，支持在会话中途切换模型
- **token 与成本追踪**: 自动统计输入/输出 token 和费用
- **思考/推理模式**: 统一的 thinking level 接口，适配各提供商的推理能力

**支持的提供商**: OpenAI、Anthropic、Google Gemini、Google Vertex、Azure OpenAI、Amazon Bedrock、Mistral、Groq、Cerebras、xAI、OpenRouter、GitHub Copilot、Kimi For Coding、MiniMax 等。

**关键数据结构**:

```typescript
// 上下文 - 一次 LLM 对话的完整状态
interface Context {
  systemPrompt: string;
  messages: Message[];   // 用户、助手、工具结果消息
  tools: Tool[];         // 可用工具定义
}

// 模型 - 标识一个特定的 LLM
interface Model<T extends Api> {
  provider: string;
  id: string;
  api: T;  // API 类型标识（如 "openai-chat"、"anthropic" 等）
}
```

**工作流程**:
1. 通过 `getModel(provider, modelId)` 获取模型引用
2. 构建 `Context` 包含系统提示、消息历史和工具定义
3. 调用 `stream(model, context, options)` 获取事件流
4. 事件流产生 `text`、`tool_call`、`thinking`、`usage`、`stop` 等事件

### 2. pi-agent-core — Agent 运行时

**包名**: `@mariozechner/pi-agent-core`

**作用**: 在 pi-ai 之上构建有状态的 agent 循环，处理工具执行和事件流。

**核心概念**:

**Agent 循环**: Agent 不只调用一次 LLM，而是运行一个循环：
1. 发送用户消息给 LLM
2. LLM 返回文本或工具调用
3. 如果有工具调用，执行工具，将结果作为 `toolResult` 消息加入上下文
4. 带着工具结果再次调用 LLM
5. 重复直到 LLM 不再调用工具

```
用户消息 → LLM → 工具调用? → 执行工具 → 工具结果 → LLM → ... → 最终回复
              └── 直接回复 ────────────────────────────────────────┘
```

**消息流转换管线**:
```
AgentMessage[] → transformContext() → AgentMessage[] → convertToLlm() → Message[] → LLM
                  (可选：裁剪上下文)                      (必需：过滤自定义消息)
```

**事件系统**: Agent 通过订阅-发布模式发出事件：
- `agent_start` / `agent_end` — agent 循环开始/结束
- `turn_start` / `turn_end` — 每轮 LLM 调用
- `message_start` / `message_update` / `message_end` — 消息生命周期
- `tool_execution_start` / `tool_execution_update` / `tool_execution_end` — 工具执行

**Steering 和 Follow-up 机制**:
- **Steering（引导）**: 在工具执行期间插入消息，中断当前操作
- **Follow-up（跟进）**: 在 agent 完成后追加消息，继续工作

### 3. pi-coding-agent — 终端编程 Agent

**包名**: `@mariozechner/pi-coding-agent`

**作用**: Pi 的核心产品，一个交互式终端编程 agent。

**运行模式**:
- **交互模式**（默认）: 终端 UI，支持消息编辑器、自动补全、会话管理
- **Print 模式** (`-p`): 输出回复后退出
- **JSON 模式** (`--mode json`): 以 JSON Lines 输出事件流
- **RPC 模式** (`--mode rpc`): 通过 stdin/stdout 进行进程间通信
- **SDK**: 可嵌入到其他 Node.js 应用

**内置工具**:
- `read` — 读取文件内容
- `write` — 写入文件
- `edit` — 精确编辑文件中的特定内容
- `bash` — 执行 shell 命令

**会话系统**: 会话以 JSONL 文件存储，使用树形结构（每条记录有 `id` 和 `parentId`），支持原地分支和历史回溯。

**定制系统**:

| 机制 | 说明 |
|------|------|
| **Extensions（扩展）** | TypeScript 模块，可添加工具、命令、快捷键、事件处理器、UI 组件 |
| **Skills（技能）** | Markdown 定义的能力包，通过 `/skill:name` 调用 |
| **Prompt Templates（提示模板）** | 可复用的 Markdown 提示，通过 `/name` 展开 |
| **Themes（主题）** | 自定义终端配色方案，支持热重载 |
| **Pi Packages** | 将扩展、技能、模板、主题打包分享（npm 或 git） |

**上下文文件**: 启动时加载 `AGENTS.md`（或 `CLAUDE.md`），从 `~/.pi/agent/` 全局目录和项目目录树中收集，用于注入项目约定和指令。

**Compaction（压缩）**: 长会话超出上下文窗口时，自动或手动将旧消息压缩为摘要，保留近期消息完整。完整历史保存在 JSONL 文件中。

### 4. pi-tui — 终端 UI 库

**包名**: `@mariozechner/pi-tui`

**作用**: 提供差分渲染的终端 UI 框架，是 pi-coding-agent 的界面基础。

**核心特性**:
- **差分渲染**: 三策略渲染系统（首次渲染、宽度变化全量重绘、正常更新只渲染变化行）
- **同步输出**: 使用 CSI 2026 协议实现原子屏幕更新，无闪烁
- **组件系统**: `Component` 接口（`render(width): string[]`），内置 Text、Editor、Markdown、SelectList、Image 等组件
- **Overlay 系统**: 在现有内容之上渲染浮层（对话框、菜单等）
- **输入处理**: `matchesKey()` 辅助函数，支持 Kitty 键盘协议
- **IME 支持**: `Focusable` 接口，支持中日韩输入法定位

### 5. pi-mom — Slack 机器人

**包名**: `@mariozechner/pi-mom`

**作用**: 一个 Slack 机器人，将消息委托给 pi agent 处理。

**特点**:
- **自管理**: 自动安装工具、配置凭据、维护工作空间
- **每频道隔离**: 每个 Slack 频道有独立的对话历史、上下文和技能
- **双文件历史**: `log.jsonl`（完整消息源）+ `context.jsonl`（LLM 上下文，可压缩）
- **记忆系统**: `MEMORY.md` 文件保存跨会话的规则和偏好
- **技能系统**: 可自行创建 CLI 工具（脚本 + SKILL.md 描述文件）
- **事件系统**: 支持定时任务、一次性提醒、周期性唤醒
- **Docker 沙箱**: 推荐在 Docker 容器中运行，隔离主机

### 6. pi-pods — GPU Pod 部署管理

**包名**: `@mariozechner/pi-pods`

**作用**: 在远程 GPU pod 上部署和管理 vLLM 实例。

**功能**:
- 自动在 Ubuntu pod 上配置 vLLM
- 为 agentic 模型（Qwen、GPT-OSS、GLM 等）配置工具调用
- 管理同一 pod 上的多个模型，自动分配 GPU
- 提供 OpenAI 兼容的 API 端点
- 支持 DataCrunch、RunPod、Vast.ai 等 GPU 提供商

### 7. pi-web-ui — Web UI 组件

**包名**: `@mariozechner/pi-web-ui`

**作用**: 可复用的 Web 组件，用于构建浏览器端的 AI 聊天界面。

**组件**:
- `ChatPanel` — 完整聊天界面（消息列表、输入框、artifacts 面板）
- `AgentInterface` — 底层聊天接口
- `ArtifactsPanel` — 展示 HTML、SVG、Markdown 等交互式内容

**特性**:
- IndexedDB 后端存储（会话、API 密钥、设置）
- 附件支持（PDF、DOCX、图片等）
- JavaScript REPL 工具（沙箱执行）
- CORS 代理处理
- 国际化支持

## 数据流：一次完整的交互

以用户在终端输入 "读取 config.json 并解释" 为例：

```
1. 用户在 Editor（pi-tui）中输入并提交
         │
2. coding-agent 将输入封装为 AgentMessage
         │
3. Agent（pi-agent-core）启动 agent 循环
         │
4. transformContext() 裁剪历史消息
         │
5. convertToLlm() 转换为 LLM 可理解的 Message[]
         │
6. pi-ai 的 stream() 调用 LLM API（如 Anthropic）
         │
7. LLM 返回 tool_call 事件: read({path: "config.json"})
         │
8. Agent 执行 read 工具，读取文件内容
         │
9. 工具结果作为 toolResult 消息加入上下文
         │
10. 再次调用 stream()，LLM 看到文件内容后生成解释
          │
11. text_delta 事件流式传输到 coding-agent
          │
12. Markdown 组件（pi-tui）实时渲染到终端
          │
13. agent_end 事件触发，会话写入 JSONL 文件
```

## 技术栈

| 技术 | 用途 |
|------|------|
| TypeScript | 所有包的主要语言 |
| Node.js (>=20) | 运行时环境 |
| npm workspaces | monorepo 包管理 |
| Biome | 代码格式化与 lint |
| TypeBox | JSON Schema 类型定义与验证（工具参数） |
| mini-lit | Web 组件框架（web-ui） |
| Tailwind CSS v4 | Web UI 样式 |
| Vitest | 测试框架 |

## 开发命令

```bash
npm install          # 安装所有依赖
npm run build        # 构建所有包
npm run check        # lint、格式化、类型检查（需先 build）
./test.sh            # 运行测试（无 API key 时跳过 LLM 相关测试）
./pi-test.sh         # 从源码运行 pi（必须在仓库根目录执行）
```

## 许可证

MIT
