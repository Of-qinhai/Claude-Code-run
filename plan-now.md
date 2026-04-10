# Super Agent WebUI Short-Term Execution Plan (Scheme C / Now)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在本仓库下交付 `apps/super-agent-web` 的首个可用版本（V1），实现本地 WebUI、双模型标准（OpenAI/Anthropic）和真实工具调用闭环。

**Architecture:** 采用“独立新项目 + 核心流程复刻”。不直接重耦合现有超大模块，先复刻高价值能力：Agent Loop、工具编排、权限模式、流式事件。

**Tech Stack:** Bun、TypeScript、React、Vite、Hono、SSE、Zod、Vitest。

---

## 0) V1 范围边界

- 必做：
  - 本地 WebUI 可聊天
  - 模型配置支持 `provider/baseURL/apiKey/model`
  - 支持 Anthropic 与 OpenAI 两种模型标准
  - 至少 5 个核心工具可调用
  - 流式输出 + 工具时间线
- 暂不做：
  - 多代理编排
  - MCP 生态全量接入
  - Bridge/remote-control
  - 历史压缩（snip/compact）

---

## 1) 近期核心提炼（V1必须）

### A. Agent 主循环
- 来源：`src/QueryEngine.ts`、`src/query.ts`、`src/cli/print.ts`(runHeadless)
- 落地：`apps/super-agent-web/server/src/agent/agentLoop.ts`

### B. 工具编排语义
- 来源：`src/services/tools/toolOrchestration.ts`、`toolExecution.ts`、`StreamingToolExecutor.ts`
- 落地：`apps/super-agent-web/server/src/tools/toolOrchestrator.ts`、`toolExecutor.ts`

### C. 权限模式
- 来源：`src/Tool.ts`、`src/cli/structuredIO.ts`
- 落地：`apps/super-agent-web/server/src/permissions/permissionManager.ts`

### D. 流式事件协议
- 来源：`src/entrypoints/sdk/coreSchemas.ts`
- 落地：`apps/super-agent-web/shared/src/events.ts`

### E. 双 Provider 适配层
- 新建：`anthropicAdapter.ts`、`openaiAdapter.ts`

---

## 2) 近期目录结构

```txt
apps/super-agent-web/
  web/
    src/
      pages/ChatPage.tsx
      components/ChatPanel.tsx
      components/ToolTimeline.tsx
      components/ModelConfigForm.tsx
  server/
    src/
      routes/chat.ts
      routes/config.ts
      agent/agentLoop.ts
      llm/adapters/anthropicAdapter.ts
      llm/adapters/openaiAdapter.ts
      tools/toolRegistry.ts
      tools/toolOrchestrator.ts
      tools/toolExecutor.ts
      tools/builtin/*
      permissions/permissionManager.ts
      store/configStore.ts
      store/sessionStore.ts
  shared/
    src/
      schemas.ts
      events.ts
      types.ts
```

---

## 3) 近期任务（按顺序执行）

### Task 1: 初始化工程骨架

**Files:**
- Create: `apps/super-agent-web/**`
- Modify: 根 `package.json`（workspace scripts）

- [ ] **Step 1:** 创建目录结构
- [ ] **Step 2:** 初始化依赖与启动脚本
- [ ] **Step 3:** 验证 `bun run dev` 可同时拉起 web/server
- [ ] **Step 4:** 提交骨架

### Task 2: 共享协议（前后端类型统一）

**Files:**
- Create: `apps/super-agent-web/shared/src/{types,schemas,events}.ts`

- [ ] **Step 1:** 写失败测试（schema）
- [ ] **Step 2:** 实现事件和请求 schema
- [ ] **Step 3:** 固化事件集合：`assistant_delta/tool_start/tool_output/tool_end/run_end/run_error`
- [ ] **Step 4:** 提交

### Task 3: 模型适配层（Anthropic + OpenAI）

**Files:**
- Create: `apps/super-agent-web/server/src/llm/**`

- [ ] **Step 1:** 定义统一 `LLMAdapter` 接口
- [ ] **Step 2:** 实现 `anthropicAdapter`
- [ ] **Step 3:** 实现 `openaiAdapter`
- [ ] **Step 4:** mock 测试通过
- [ ] **Step 5:** 提交

### Task 4: 工具系统 V1

**Files:**
- Create: `apps/super-agent-web/server/src/tools/**`

- [ ] **Step 1:** 注册机制 + schema 校验
- [ ] **Step 2:** 并发安全/串行执行语义
- [ ] **Step 3:** 实现 5 工具：`bash/read_file/write_file/edit_file/search`
- [ ] **Step 4:** 统一错误结构
- [ ] **Step 5:** 单测通过
- [ ] **Step 6:** 提交

### Task 5: 权限策略层

**Files:**
- Create: `apps/super-agent-web/server/src/permissions/permissionManager.ts`

- [ ] **Step 1:** 支持 `ask/auto_allow/auto_deny`
- [ ] **Step 2:** `ask` 回路事件化（前端确认）
- [ ] **Step 3:** 权限测试通过
- [ ] **Step 4:** 提交

### Task 6: Agent 主循环 + SSE

**Files:**
- Create: `apps/super-agent-web/server/src/agent/agentLoop.ts`
- Create: `apps/super-agent-web/server/src/routes/chat.ts`

- [ ] **Step 1:** 实现 loop 状态机
- [ ] **Step 2:** 事件流 SSE 输出
- [ ] **Step 3:** 边界：maxTurns/abort/error
- [ ] **Step 4:** 测试通过
- [ ] **Step 5:** 提交

### Task 7: WebUI 首版

**Files:**
- Create: `apps/super-agent-web/web/src/components/**`

- [ ] **Step 1:** 聊天流式渲染
- [ ] **Step 2:** 工具时间线
- [ ] **Step 3:** 模型配置面板
- [ ] **Step 4:** `ask` 权限弹窗
- [ ] **Step 5:** 前端测试通过
- [ ] **Step 6:** 提交

### Task 8: 配置与会话持久化

**Files:**
- Create: `apps/super-agent-web/server/src/store/{configStore,sessionStore}.ts`

- [ ] **Step 1:** 模型配置持久化
- [ ] **Step 2:** 会话持久化（至少最近会话恢复）
- [ ] **Step 3:** 存储测试通过
- [ ] **Step 4:** 提交

### Task 9: 集成验收（V1 Gate）

**Files:**
- Create: `apps/super-agent-web/docs/v1-acceptance.md`
- Create: `apps/super-agent-web/README.md`

- [ ] **Step 1:** 验收场景 A：读文件 -> 编辑 -> 总结
- [ ] **Step 2:** 验收场景 B：bash 执行 -> 解析 -> 结论
- [ ] **Step 3:** 记录 known issues
- [ ] **Step 4:** 提交

---

## 4) 里程碑（Now）

- M1：双 Provider 可流式对话（无工具）
- M2：5 个工具闭环可执行
- M3：WebUI 时间线 + 权限交互
- M4：会话/配置持久化 + V1 验收通过

---

## 5) 完成定义（V1 DoD）

- 本地启动可交互
- OpenAI/Anthropic 端点可切换
- 5 个核心工具可真实调用
- 工具调用链完整可见
- 权限模式可切换并生效
- 能稳定完成至少一个端到端任务

