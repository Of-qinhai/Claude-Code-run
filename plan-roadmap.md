# Super Agent WebUI Long-term Roadmap (Scheme C / Future)

## 1) 路线目标

在 V1 基础上逐步覆盖 `src/tools` 生态能力，避免一次性过载，按业务价值和工程风险分波次推进。

---

## 2) 波次规划

### Wave A（V1.5）: Productivity Pack

目标：增强“计划-执行-确认-结构化输出”体验。

纳入工具：
- `NotebookEditTool`
- `AskUserQuestionTool`
- `SyntheticOutputTool`
- `TaskCreateTool`
- `TaskGetTool`
- `TaskUpdateTool`
- `TaskListTool`
- `TodoWriteTool`
- `VerifyPlanExecutionTool`

完成标准：
- 结构化输出可用
- 任务列表工具可用
- 需要用户确认的交互可稳定中断/恢复

### Wave B（V2）: MCP & Skills Pack

目标：接入可扩展工具生态。

纳入工具：
- `MCPTool`
- `McpAuthTool`
- `ListMcpResourcesTool`
- `ReadMcpResourceTool`
- `ToolSearchTool`
- `DiscoverSkillsTool`
- `SkillTool`

完成标准：
- 支持 MCP server 配置与状态管理
- 可列举/读取 MCP 资源
- 技能可发现并可执行

### Wave C（V3）: Multi-Agent Orchestration Pack

目标：支持子代理、任务流和团队协作。

纳入工具：
- `AgentTool`
- `SendMessageTool`
- `TeamCreateTool`
- `TeamDeleteTool`
- `TaskOutputTool`
- `TaskStopTool`
- `WorkflowTool`
- `ScheduleCronTool`
- `SleepTool`
- `RemoteTriggerTool`
- `MonitorTool`

完成标准：
- 子代理生命周期可见可控
- 任务状态完整可视化
- 工作流与定时触发可运行

### Wave D（V4）: IDE & Browser Capability Pack

目标：增强代码理解与外部信息交互。

纳入工具：
- `LSPTool`
- `WebBrowserTool`
- `WebFetchTool`（增强）
- `WebSearchTool`（增强）
- `ReviewArtifactTool`
- `SendUserFileTool`
- `TerminalCaptureTool`

完成标准：
- LSP 语义工具可用
- Browser/Fetch/Search 合规可控
- 产物审阅可进入任务/消息流

### Wave E（V5）: Platform & Expert Pack

目标：覆盖平台差异和高级工作模式。

纳入工具：
- `PowerShellTool`
- `REPLTool`
- `EnterWorktreeTool`
- `ExitWorktreeTool`
- `SnipTool`
- `BriefTool`
- `ConfigTool`
- `TungstenTool`（实验）

完成标准：
- Windows PowerShell 可用
- Worktree 隔离能力可用
- 长对话治理能力可用

---

## 3) 全量工具覆盖台账（来自 `src/tools`）

说明：`shared/testing` 为支撑目录，不作为产品功能工具。

| 工具目录 | 计划波次 | 默认状态 | 说明 |
|---|---|---|---|
| AgentTool | Wave C | backlog | 多代理核心 |
| AskUserQuestionTool | Wave A | planned | ask 模式核心交互 |
| BashTool | V1 | delivered/planned | 核心执行工具 |
| BriefTool | Wave E | backlog | 会话摘要增强 |
| ConfigTool | Wave E | backlog | 高级配置管理 |
| DiscoverSkillsTool | Wave B | backlog | 技能发现 |
| EnterPlanModeTool | V1 | delivered/planned | 计划模式 |
| EnterWorktreeTool | Wave E | backlog | 工作树隔离 |
| ExitPlanModeTool | V1 | delivered/planned | 计划模式退出 |
| ExitWorktreeTool | Wave E | backlog | 工作树隔离 |
| FileEditTool | V1 | delivered/planned | 核心编辑 |
| FileReadTool | V1 | delivered/planned | 核心读取 |
| FileWriteTool | V1 | delivered/planned | 核心写入 |
| GlobTool | V1 | delivered/planned | 文件发现 |
| GrepTool | V1 | delivered/planned | 内容检索 |
| LSPTool | Wave D | backlog | 语义检索 |
| ListMcpResourcesTool | Wave B | backlog | MCP 资源发现 |
| MCPTool | Wave B | backlog | MCP 调用核心 |
| McpAuthTool | Wave B | backlog | MCP 认证流程 |
| MonitorTool | Wave C | backlog | 监控类任务 |
| NotebookEditTool | Wave A | backlog | notebook 编辑 |
| OverflowTestTool | Experimental | skipped | 测试专用 |
| PowerShellTool | Wave E | backlog | Windows 平台 |
| REPLTool | Wave E | backlog | REPL 高级模式 |
| ReadMcpResourceTool | Wave B | backlog | MCP 资源读取 |
| RemoteTriggerTool | Wave C | backlog | 远程触发 |
| ReviewArtifactTool | Wave D | backlog | 产物审阅 |
| ScheduleCronTool | Wave C | backlog | 定时任务 |
| SendMessageTool | Wave C | backlog | Agent 间通信 |
| SendUserFileTool | Wave D | backlog | 用户文件发送 |
| SkillTool | Wave B | backlog | 技能执行 |
| SleepTool | Wave C | backlog | 自主循环等待 |
| SnipTool | Wave E | backlog | 历史压缩 |
| SyntheticOutputTool | Wave A | backlog | 结构化输出 |
| TaskCreateTool | Wave A | backlog | Todo v2 |
| TaskGetTool | Wave A | backlog | Todo v2 |
| TaskListTool | Wave A | backlog | Todo v2 |
| TaskOutputTool | Wave C | backlog | 子任务输出 |
| TaskStopTool | Wave C | backlog | 子任务中止 |
| TaskUpdateTool | Wave A | backlog | Todo v2 |
| TeamCreateTool | Wave C | backlog | 团队编排 |
| TeamDeleteTool | Wave C | backlog | 团队编排 |
| TerminalCaptureTool | Wave D | backlog | 终端采样 |
| TodoWriteTool | Wave A | backlog | 传统 Todo |
| ToolSearchTool | Wave B | backlog | 工具检索 |
| TungstenTool | Wave E | backlog | 实验纳管 |
| VerifyPlanExecutionTool | Wave A | backlog | 计划执行校验 |
| WebBrowserTool | Wave D | backlog | 浏览器交互 |
| WebFetchTool | V1/Wave D | delivered/planned | V1 基础版，Wave D 增强 |
| WebSearchTool | V1/Wave D | delivered/planned | V1 基础版，Wave D 增强 |
| WorkflowTool | Wave C | backlog | 工作流脚本 |

---

## 4) 长线实施任务

### Task 10: 工具迁移框架标准化

- [ ] 建立工具迁移模板（输入 schema、权限、并发、错误码、事件映射、回归用例）
- [ ] 为每个工具打标签（v1/wave-a/.../wave-e）
- [ ] 建立上线门槛（单测 + 权限测试 + 超时测试 + 事件测试）

### Task 11: Wave A 执行

- [ ] 接入 `AskUserQuestionTool` + `SyntheticOutputTool`
- [ ] 接入 `Task*` + `TodoWriteTool`
- [ ] 接入 `NotebookEditTool` + `VerifyPlanExecutionTool`
- [ ] 完成 Wave A 验收

### Task 12: Wave B 执行

- [ ] MCP 连接管理与认证流程
- [ ] 资源列举/读取
- [ ] ToolSearch + SkillTool 联动
- [ ] 完成 Wave B 验收

### Task 13: Wave C 执行

- [ ] 子代理生命周期管理
- [ ] 消息总线与任务状态系统
- [ ] Team/Workflow/Cron/Trigger 联动
- [ ] 完成 Wave C 验收

### Task 14: Wave D/E 执行

- [ ] Wave D：LSP + Browser + Artifact + TerminalCapture
- [ ] Wave E：PowerShell + Worktree + Snip + REPL + Config/Brief
- [ ] 跨平台回归（macOS/Linux/Windows）

---

## 5) 版本建议

- `v1.0.0`: Now Plan 完成（MVP）
- `v1.5.0`: Wave A
- `v2.0.0`: Wave B
- `v3.0.0`: Wave C
- `v4.0.0`: Wave D
- `v5.0.0`: Wave E

