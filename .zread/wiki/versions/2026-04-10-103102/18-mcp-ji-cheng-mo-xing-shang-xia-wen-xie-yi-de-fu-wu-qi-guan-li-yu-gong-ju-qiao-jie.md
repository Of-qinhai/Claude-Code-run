MCP（Model Context Protocol，模型上下文协议）是 Claude Code 扩展能力的核心通道——它将外部工具服务器、远程 API 和第三方能力以标准化协议桥接到 AI 会话中，使 Claude 不再局限于内置工具集，而是能够动态发现、连接和调用任意 MCP 服务器暴露的工具与资源。本文将深入剖析 MCP 集成的完整架构：从连接管理器的生命周期控制，到工具桥接的动态注册机制，再到认证流、权限门控与 UI 交互层的协作全貌。

## 架构总览：三层分离的 MCP 集成栈

Claude Code 的 MCP 实现遵循 **传输层 → 管理层 → 表现层** 的三层架构。传输层负责进程间通信（stdio）与远程通信（HTTP/SSE/WebSocket）；管理层负责连接生命周期、工具发现、权限审批与配置持久化；表现层则在终端 UI 中呈现服务器状态、工具详情与审批对话框。这种分层使得 MCP 的核心逻辑不受传输方式变化的影响，也令 UI 交互可独立演进。

```
┌─────────────────────────────────────────────────────────┐
│                    表现层 (UI)                           │
│  MCPListPanel · MCPServerMenu · MCPToolDetailView       │
│  MCPSettings · ElicitationDialog · MCPServerApproval    │
├─────────────────────────────────────────────────────────┤
│                    管理层 (Service)                       │
│  MCPConnectionManager · useManageMCPConnections          │
│  config · auth · channelPermissions · elicitationHandler │
│  normalization · officialRegistry · oauthPort            │
├─────────────────────────────────────────────────────────┤
│                    传输层 (Transport)                     │
│  InProcessTransport · SdkControlTransport                │
│  client · mcpWebSocketTransport · vscodeSdkMcp           │
└─────────────────────────────────────────────────────────┘
```

Sources: [MCPConnectionManager.tsx](src/services/mcp/MCPConnectionManager.tsx), [useManageMCPConnections.ts](src/services/mcp/useManageMCPConnections.ts), [InProcessTransport.ts](src/services/mcp/InProcessTransport.ts), [SdkControlTransport.ts](src/services/mcp/SdkControlTransport.ts)

## 连接管理器：MCPConnectionManager 的生命周期控制

`MCPConnectionManager` 是整个 MCP 集成的中枢控制器，它以 React 组件的形式存在（`.tsx` 后缀暗示其挂载在组件树中），这意味着 MCP 连接的状态天然与 React 渲染周期绑定。管理器承担以下核心职责：**服务器进程的启动与停止**（针对 stdio 类型服务器）、**连接健康监测与自动重连**、**工具列表的动态发现与注册**，以及**服务器配置的热更新响应**。

MCP 服务器连接支持两种传输模式：**stdio 模式**——Claude Code 作为父进程拉起 MCP 服务器子进程，通过标准输入/输出进行 JSON-RPC 通信；**远程模式**（HTTP/SSE）——连接到已部署的 MCP 服务器端点，无需本地进程管理。`InProcessTransport` 则提供了第三种可能：将 MCP 服务器直接嵌入同一进程内运行，消除进程间通信开销，适用于高性能内置工具的桥接场景。`SdkControlTransport` 为 VS Code SDK 等外部宿主提供了控制通道，允许 IDE 插件直接参与 MCP 会话管理。

Sources: [MCPConnectionManager.tsx](src/services/mcp/MCPConnectionManager.tsx#L1-L50), [InProcessTransport.ts](src/services/mcp/InProcessTransport.ts), [SdkControlTransport.ts](src/services/mcp/SdkControlTransport.ts), [client.ts](src/services/mcp/client.ts)

## 配置体系：分层声明与环境变量展开

MCP 服务器的配置采用**项目级 + 用户级 + 企业级**的三层声明模型。项目级配置存储在 `.claude/mcp.json` 中，随代码仓库版本化；用户级配置位于 `~/.claude/mcp.json`，跨项目共享；企业级则通过**渠道白名单**（`channelAllowlist`）机制由管理员统一管控。`config.ts` 模块负责解析和合并这三层配置，处理优先级冲突与重复定义消除。

`envExpansion.ts` 解决了一个关键问题：MCP 服务器配置中的环境变量引用（如 `${API_KEY}`）需要在运行时展开，而非静态读取。这允许用户在配置中引用系统环境变量，而无需将敏感值硬编码到配置文件中——配置文件可以安全提交到版本控制，实际凭证则从运行环境中注入。

| 配置层级 | 存储位置 | 作用域 | 优先级 |
|---------|---------|-------|-------|
| 项目级 | `.claude/mcp.json` | 当前仓库 | 最高 |
| 用户级 | `~/.claude/mcp.json` | 全局 | 中 |
| 企业级 | 远程托管配置 | 组织范围 | 基础底线 |

Sources: [config.ts](src/services/mcp/config.ts), [envExpansion.ts](src/services/mcp/envExpansion.ts), [channelAllowlist.ts](src/services/mcp/channelAllowlist.ts)

## 认证与授权：OAuth 流、IDP 集成与服务器审批

MCP 服务器的认证是安全性保障的第一道门。`auth.ts` 模块定义了认证状态的统一抽象，支持从无认证（anonymous）到 API Key 直传、再到完整 OAuth 2.0 流程的多种模式。`oauthPort.ts` 管理本地 OAuth 回调服务器的端口分配，确保多个并发 OAuth 流程不会端口冲突。`xaa.ts` 和 `xaaIdpLogin.ts` 实现了企业级身份提供商（IDP）集成，允许 MCP 服务器通过企业 SSO 进行身份验证。

**服务器审批机制**是 MCP 安全模型的关键环节。当用户首次添加一个 MCP 服务器时，系统通过 `mcpServerApproval.tsx` 弹出审批对话框，展示服务器将暴露的工具列表和请求的权限范围，要求用户显式批准。`channelPermissions.ts` 则在组织层面控制特定渠道（channel）下的服务器是否需要额外审批，或是否可以自动受信——这为企业部署提供了集中管控能力。

`elicitationHandler.ts` 处理 MCP 协议中的 **Elicitation** 机制：当 MCP 服务器在工具执行过程中需要向用户请求额外信息时（如确认操作、输入凭证），通过此处理器弹出交互式对话框，将用户输入回传给服务器。对应的 UI 组件为 `ElicitationDialog.tsx`，而 `elicitationValidation.ts`（位于 utils 目录）负责验证用户输入是否符合服务器声明的约束。

Sources: [auth.ts](src/services/mcp/auth.ts), [oauthPort.ts](src/services/mcp/oauthPort.ts), [xaa.ts](src/services/mcp/xaa.ts), [xaaIdpLogin.ts](src/services/mcp/xaaIdpLogin.ts), [mcpServerApproval.tsx](src/services/mcpServerApproval.tsx), [channelPermissions.ts](src/services/mcp/channelPermissions.ts), [elicitationHandler.ts](src/services/mcp/elicitationHandler.ts), [ElicitationDialog.tsx](src/components/mcp/ElicitationDialog.tsx), [elicitationValidation.ts](src/utils/mcp/elicitationValidation.ts)

## 工具桥接：从 MCP 工具到 Claude 内部工具的映射

MCP 的核心价值在于**工具桥接**——将 MCP 服务器暴露的工具动态转换为 Claude Code 内部的 `MCPTool` 实例，使其与内置工具（BashTool、FileEditTool 等）享有相同的调用接口与权限管控流程。

`MCPTool.ts` 是桥接的核心实现。每个被发现的 MCP 工具都会被包装为一个 MCPTool 实例，其关键设计包括：**输入 Schema 的 JSON Schema 转换**（MCP 协议的 Schema 被适配为 Claude 内部工具的参数描述格式）、**工具名称的命名空间前缀**（格式为 `mcp__<server_name>__<tool_name>`，避免不同服务器的同名工具冲突）、以及**调用结果的标准化转换**（将 MCP 工具返回的内容适配为 Claude 消息流可消费的格式）。

`prompt.ts` 负责生成 MCP 工具的系统提示注入文本——当工具列表动态变化时，需要更新系统提示中的工具描述段，使模型感知到新增/变更的工具。`classifyForCollapse.ts` 决定工具调用结果在终端 UI 中的折叠策略：高频低价值输出（如心跳检查、状态查询）默认折叠，而关键操作结果则展开显示。`UI.tsx` 提供了 MCPTool 在消息流中的渲染组件。

```
MCP Server ──(JSON-RPC)──> MCPClient ──(discovery)──> MCPConnectionManager
                                                         │
                                                    (tool wrapping)
                                                         ▼
                                                    MCPTool 实例
                                                         │
                                              ┌──────────┼──────────┐
                                              ▼          ▼          ▼
                                          prompt.ts   UI.tsx   classifyForCollapse.ts
                                           (提示注入)  (渲染)      (折叠策略)
```

此外，MCP 协议不仅暴露工具（Tools），还暴露**资源（Resources）**。`ListMcpResourcesTool` 和 `ReadMcpResourceTool` 是两个专门的内置工具，允许 Claude 列举和读取 MCP 服务器提供的资源（如文件、数据集、配置模板等），这拓展了 MCP 的能力边界，使其不仅是"远程过程调用"通道，更是"上下文供给"通道。

Sources: [MCPTool.ts](src/tools/MCPTool/MCPTool.ts), [prompt.ts](src/tools/MCPTool/prompt.ts), [classifyForCollapse.ts](src/tools/MCPTool/classifyForCollapse.ts), [UI.tsx](src/tools/MCPTool/UI.tsx), [ListMcpResourcesTool](src/tools/ListMcpResourcesTool), [ReadMcpResourceTool](src/tools/ReadMcpResourceTool)

## MCP 技能系统：从工具到技能的语义升维

在纯粹的 MCP 工具桥接之上，Claude Code 还构建了**MCP 技能系统**——将 MCP 工具组合封装为更高语义层次的"技能"（Skill），提供更友好的发现入口与使用引导。

`mcpSkills.ts` 负责从已连接的 MCP 服务器中发现可封装为技能的工具集合；`mcpSkillBuilders.ts` 则提供建造器模式，将原始 MCP 工具元数据转化为技能描述对象（包含技能名称、描述、使用示例等）。这种**工具→技能的升维**使得用户不需要了解底层 MCP 工具的具体参数和调用约定，而可以通过自然语言描述触发整个技能链。

例如，一个数据库 MCP 服务器可能暴露 `query`、`schema`、`explain` 三个独立工具；MCP 技能系统可以将它们组合为"数据库查询助手"技能，在用户问出"帮我查看用户表结构"时，自动编排 `schema` 工具的调用与结果解读，而不需要用户逐一指定工具和参数。

Sources: [mcpSkills.ts](src/skills/mcpSkills.ts), [mcpSkillBuilders.ts](src/skills/mcpSkillBuilders.ts)

## 命令行接口：MCP 管理斜杠命令

MCP 的命令行接口封装在 `src/commands/mcp/` 目录下，提供了用户直接管理 MCP 服务器的终端操作入口：

| 命令 | 实现文件 | 功能描述 |
|------|---------|---------|
| `/mcp` | [mcp.tsx](src/commands/mcp/mcp.tsx) | 主命令入口，列出已配置的服务器及状态 |
| `/mcp add` | [addCommand.ts](src/commands/mcp/addCommand.ts) | 交互式添加新的 MCP 服务器配置 |
| `/mcp xaa-idp` | [xaaIdpCommand.ts](src/commands/mcp/xaaIdpCommand.ts) | 执行企业 IDP 登录认证流程 |

`addCommand.ts` 的交互式添加流程值得特别关注：它不仅收集服务器名称和传输配置（stdio 命令或远程 URL），还会触发前述的服务器审批流程，并在配置写入前验证服务器的可达性——先尝试建立连接，确认工具列表可枚举后才持久化配置，避免用户添加一个无法连接的"幽灵服务器"。

Sources: [mcp.tsx](src/commands/mcp/mcp.tsx), [addCommand.ts](src/commands/mcp/addCommand.ts), [xaaIdpCommand.ts](src/commands/mcp/xaaIdpCommand.ts), [index.ts](src/commands/mcp/index.ts)

## UI 组件层：服务器状态面板与交互式审批

MCP 的 UI 组件层位于 `src/components/mcp/` 目录，提供终端界面中的可视化交互窗口。核心组件及其职责如下：

**`MCPListPanel.tsx`**——主面板组件，以列表形式展示所有已配置的 MCP 服务器，每条记录包含服务器名称、连接状态（已连接/断开/错误）、工具数量等摘要信息。**`MCPServerMenu` 家族**——分为 `MCPStdioServerMenu.tsx`（本地 stdio 服务器菜单）和 `MCPRemoteServerMenu.tsx`（远程服务器菜单），针对不同传输类型提供差异化操作项（如 stdio 类型可重启进程，远程类型可刷新端点）。**`MCPToolListView.tsx` / `MCPToolDetailView.tsx`**——工具列表与详情视图，前者展示单个服务器暴露的全部工具概览，后者展示选中工具的完整 Schema、描述与参数定义。**`MCPSettings.tsx`**——服务器配置编辑界面，允许用户修改环境变量、命令参数等配置项。**`MCPReconnect.tsx`**——断线重连交互，配合 `reconnectHelpers.tsx` 在连接丢失时提供手动重连按钮与自动重试逻辑。**`CapabilitiesSection.tsx`**——展示服务器声明的能力集（capabilities），如是否支持工具、资源、提示模板等。**`McpParsingWarnings.tsx`**——当 MCP 服务器返回的工具 Schema 解析异常时，展示警告信息帮助用户排查配置问题。**`MCPAgentServerMenu.tsx`**——Agent 上下文下的 MCP 服务器菜单，允许在 Agent 任务中临时启用/禁用特定 MCP 服务器。

Sources: [MCPListPanel.tsx](src/components/mcp/MCPListPanel.tsx), [MCPStdioServerMenu.tsx](src/components/mcp/MCPStdioServerMenu.tsx), [MCPRemoteServerMenu.tsx](src/components/mcp/MCPRemoteServerMenu.tsx), [MCPToolListView.tsx](src/components/mcp/MCPToolListView.tsx), [MCPToolDetailView.tsx](src/components/mcp/MCPToolDetailView.tsx), [MCPSettings.tsx](src/components/mcp/MCPSettings.tsx), [MCPReconnect.tsx](src/components/mcp/MCPReconnect.tsx), [CapabilitiesSection.tsx](src/components/mcp/CapabilitiesSection.tsx), [McpParsingWarnings.tsx](src/components/mcp/McpParsingWarnings.tsx), [MCPAgentServerMenu.tsx](src/components/mcp/MCPAgentServerMenu.tsx)

## 工具规范化与官方注册表

`normalization.ts` 解决了 MCP 生态中的一个实际问题：不同 MCP 服务器对工具名称、参数约定的命名风格差异巨大（有的用 camelCase，有的用 snake_case，甚至包含特殊字符）。规范化层在工具发现阶段统一处理这些差异，确保映射到 Claude 内部的工具标识符始终符合内部命名约束。

`officialRegistry.ts` 维护了一份**官方 MCP 服务器注册表**——这是 Anthropic 官方认证和推荐的服务器列表，在 `/mcp add` 命令的交互流程中作为快捷选项提供，降低用户寻找和配置可信 MCP 服务器的门槛。`mcpStringUtils.ts` 提供了 MCP 相关的字符串处理工具函数，包括服务器名称的合法性校验与格式化。

Sources: [normalization.ts](src/services/mcp/normalization.ts), [officialRegistry.ts](src/services/mcp/officialRegistry.ts), [mcpStringUtils.ts](src/services/mcp/mcpStringUtils.ts)

## 集成验证与诊断：Doctor 命令中的 MCP 检查

`McpAuthTool`（位于 tools 目录）是一个特殊的内置工具，用于在会话中验证 MCP 服务器的认证状态——当模型调用某个 MCP 工具但收到认证错误时，McpAuthTool 可以触发重新认证流程，而无需用户手动退出会话重新配置。`MonitorMcpTask`（位于 tasks 目录）则将 MCP 连接监控提升为后台任务级别，持续跟踪服务器健康状态并在异常时发出通知。Claude Code 的 `doctor` 诊断命令也包含 MCP 相关的检查项，验证配置文件格式、服务器可达性与认证有效性。

Sources: [McpAuthTool](src/tools/McpAuthTool), [MonitorMcpTask](src/tasks/MonitorMcpTask), [Doctor.tsx](src/screens/Doctor.tsx)

## 完整交互流程：从添加服务器到工具调用

以下流程图展示了一个完整的 MCP 服务器集成生命周期——从用户执行添加命令到模型调用 MCP 工具并返回结果的全链路：

```
┌──────────────────────────────────────────────────────────────────┐
│                     MCP 集成完整交互流程                           │
└──────────────────────────────────────────────────────────────────┘

  用户                    命令层                    管理层                      MCP 服务器
   │                       │                         │                           │
   │── /mcp add ──────────>│                         │                           │
   │                       │── addCommand.ts ───────>│                           │
   │                       │                         │── 解析配置 ──────────────>│
   │                       │                         │── 建立连接 ──────────────>│
   │                       │                         │<── 工具列表响应 ──────────│
   │                       │                         │                           │
   │<── 审批对话框 ────────│                         │                           │
   │── 批准 ──────────────>│                         │                           │
   │                       │                         │── 持久化配置              │
   │                       │                         │── 注册 MCPTool 实例       │
   │                       │                         │── 更新系统提示 ──────────>│ 模型感知新工具
   │                       │                         │                           │
   │── 用户提问 ───────────────────────────────────>│                           │
   │                       │                         │── 模型选择 mcp__* 工具    │
   │                       │                         │── 权限检查 ──────┐        │
   │                       │                         │<── 通过 ────────┘        │
   │                       │                         │── JSON-RPC 调用 ────────>│
   │                       │                         │<── 执行结果 ─────────────│
   │                       │                         │── 标准化结果格式          │
   │<── 渲染工具输出 ──────│                         │                           │
```

Sources: [addCommand.ts](src/commands/mcp/addCommand.ts), [MCPConnectionManager.tsx](src/services/mcp/MCPConnectionManager.tsx), [MCPTool.ts](src/tools/MCPTool/MCPTool.ts), [mcpServerApproval.tsx](src/services/mcpServerApproval.tsx)

## VS Code SDK 集成与 WebSocket 传输

`vscodeSdkMcp.ts` 是 Claude Code 与 VS Code 扩展的 MCP 集成桥梁。当 Claude Code 运行在 VS Code 终端内时，此模块允许 IDE 扩展直接注册和管理 MCP 服务器，实现 IDE 原生能力（如代码分析、调试器状态）作为 MCP 工具提供给模型。`mcpWebSocketTransport.ts`（位于 utils 目录）提供了基于 WebSocket 的 MCP 协议传输实现，适用于需要双向实时通信的场景（如 IDE 与终端之间的状态同步）。

Sources: [vscodeSdkMcp.ts](src/services/mcp/vscodeSdkMcp.ts), [mcpWebSocketTransport.ts](src/utils/mcpWebSocketTransport.ts)

## 关键类型定义与辅助模块

`types.ts`（位于 services/mcp 目录）定义了 MCP 集成的核心类型系统，包括 `MCPServerConfig`（服务器配置结构）、`MCPConnectionStatus`（连接状态枚举）、`MCPToolInfo`（工具元数据）等关键接口。`headersHelper.ts` 处理 HTTP 请求头的构造，为远程 MCP 服务器的认证头注入提供支持。`claudeai.ts` 实现了与 Claude.ai 平台的 MCP 服务器同步——用户在 Claude.ai 网页端添加的 MCP 服务器可以自动同步到 Claude Code CLI。`utils.ts` 提供了通用的辅助函数集，`channelNotification.ts` 则在服务器状态变更时发送渠道通知，确保多个关注方能及时感知 MCP 连接拓扑的变化。`dateTimeParser.ts`（位于 utils/mcp 目录）处理 MCP 协议中日期时间参数的解析与格式化。

Sources: [types.ts](src/services/mcp/types.ts), [headersHelper.ts](src/services/mcp/headersHelper.ts), [claudeai.ts](src/services/mcp/claudeai.ts), [utils.ts](src/services/mcp/utils.ts), [channelNotification.ts](src/services/mcp/channelNotification.ts), [dateTimeParser.ts](src/utils/mcp/dateTimeParser.ts)

## 延伸阅读

MCP 集成是 Claude Code 扩展能力的核心机制，但其运行依赖于多个关联子系统的支撑。以下页面提供了更深层的上下文：

- **[权限与沙箱：工具执行审批流与安全隔离机制](21-quan-xian-yu-sha-xiang-gong-ju-zhi-xing-shen-pi-liu-yu-an-quan-ge-chi-ji-zhi)**——MCP 工具调用同样受权限系统管控，理解审批流的完整链路
- **[插件与技能系统：插件加载、技能发现与工作流脚本](23-cha-jian-yu-ji-neng-xi-tong-cha-jian-jia-zai-ji-neng-fa-xian-yu-gong-zuo-liu-jiao-ben)**——MCP 技能是技能系统的子集，理解技能发现与编排的完整机制
- **[配置体系：分层配置、设置同步与托管环境变量](24-pei-zhi-ti-xi-fen-ceng-pei-zhi-she-zhi-tong-bu-yu-tuo-guan-huan-jing-bian-liang)**——MCP 配置的三层声明模型是配置体系的典型应用
- **[认证与计费：OAuth 流程、API 密钥验证与用量追踪](25-ren-zheng-yu-ji-fei-oauth-liu-cheng-api-mi-yao-yan-zheng-yu-yong-liang-zhui-zong)**——MCP 服务器的 OAuth 认证复用了全局认证基础设施
- **[工具系统：50+ 内置工具的注册、调度与权限管控](5-gong-ju-xi-tong-50-nei-zhi-gong-ju-de-zhu-ce-diao-du-yu-quan-xian-guan-kong)**——MCPTool 是工具系统的动态扩展，理解工具注册的统一接口