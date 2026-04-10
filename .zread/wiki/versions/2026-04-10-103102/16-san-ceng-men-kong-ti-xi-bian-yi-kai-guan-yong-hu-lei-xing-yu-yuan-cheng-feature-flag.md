Claude Code 的功能可见性并非铁板一块，而是由**三层独立但协同的门控机制**精密调控：编译开关在构建期执行死代码消除，用户类型在运行时区分内部与外部受众，远程 Feature Flag 则从服务端实时推送灰度策略。三层叠加，形成了一套从"代码是否存在"到"代码是否执行"再到"执行哪条分支"的完整梯度管控体系。理解这套体系，是解码哪些功能"表面不存在但实际存在"、哪些功能"仅对特定人群开放"、以及哪些功能"随时可能被远程点亮"的关键。

Sources: [commands.ts](src/commands.ts#L1-L200), [betas.ts (constants)](src/constants/betas.ts#L1-L53), [betas.ts (utils)](src/utils/betas.ts#L1-L200)

## 第一层：编译开关 — `feature()` 与死代码消除

### 机制原理

Claude Code 使用 `bun:bundle` 提供的 `feature()` 函数作为编译期门控的核心原语。当 `feature('X')` 返回 `false` 时，Bundler 的 dead code elimination 会将条件分支整体剥离——这不是运行时的 `if` 判断被短路，而是**代码在产物中物理不存在**。这意味着即使逆向工程 CLI 产物，也无法从代码中找到被裁剪功能的任何痕迹。

Sources: [commands.ts](src/commands.ts#L59-L123)

### 特性标志清单

从 `commands.ts` 的条件导入中，可以提取出当前已知的编译期特性标志：

| 特性标志 | 关联命令/模块 | 语义推断 |
|---|---|---|
| `PROACTIVE` | `/proactive` | 主动式助手干预 |
| `KAIROS` | `/brief`, `/assistant`, `/proactive` | 跨会话持久助手核心 |
| `KAIROS_BRIEF` | `/brief` | Kairos 的摘要子功能 |
| `KAIROS_GITHUB_WEBHOOKS` | `/subscribe-pr` | Kairos 的 GitHub Webhook 订阅 |
| `BRIDGE_MODE` | `/bridge`, `/remoteControlServer` | 远程桥接模式 |
| `DAEMON` × `BRIDGE_MODE` | `/remoteControlServer` | 守护进程+桥接双开关 |
| `VOICE_MODE` | `/voice` | 语音交互模式 |
| `HISTORY_SNIP` | `/force-snip` | 历史片段裁剪 |
| `WORKFLOW_SCRIPTS` | `/workflows` | 工作流脚本系统 |
| `CCR_REMOTE_SETUP` | `/remote-setup` | 远程环境配置 |
| `EXPERIMENTAL_SKILL_SEARCH` | skillSearch 缓存清除 | 实验性技能搜索 |
| `ULTRAPLAN` | `/ultraplan` | 云端深度规划 |
| `TORCH` | `/torch` | 内部调试/性能工具 |
| `UDS_INBOX` | `/peers` | Unix域套接字对等消息 |
| `FORK_SUBAGENT` | `/fork` | 子Agent派生 |
| `BUDDY` | `/buddy` | 终端AI电子宠物 |
| `CONNECTOR_TEXT` | beta header 条件 | 连接器文本摘要 |
| `TRANSCRIPT_CLASSIFIER` | beta header + auto mode | 对话分类器（AFK模式等） |

Sources: [commands.ts](src/commands.ts#L59-L123), [betas.ts (constants)](src/constants/betas.ts#L23-L30)

### 组合门控模式

编译开关支持布尔组合，最典型的模式是**双开关交叠**——两个独立特性必须同时启用，功能才被编译进入产物。例如 `remoteControlServer` 命令要求 `DAEMON` **且** `BRIDGE_MODE` 同时为真：

```
const remoteControlServerCommand =
  feature('DAEMON') && feature('BRIDGE_MODE')
    ? require('./commands/remoteControlServer/index.js').default
    : null
```

这种组合门控确保了功能间的依赖关系在编译期即被强制执行——如果 `BRIDGE_MODE` 未开启，即使 `DAEMON` 已编译，远程控制服务器也不会出现在命令注册表中。

Sources: [commands.ts](src/commands.ts#L76-L79)

## 第二层：用户类型 — `USER_TYPE` 与内部/外部隔离

### 核心判别式

用户类型门控的判别式极为简洁：`process.env.USER_TYPE === 'ant'`。`'ant'` 代表 Anthropic 内部员工，所有其他值（包括未设置）均为外部用户。这个环境变量在身份认证阶段被注入，随后在整个应用生命周期中被广泛引用。

Sources: [commands.ts](src/commands.ts#L48-L52), [utils/betas.ts](src/utils/betas.ts#L166-L194)

### 门控生效的三种形态

**形态一：命令级完全屏蔽**。整个命令仅对 `ant` 用户注册，外部用户的命令列表中根本不存在该入口。典型例证是 `agents-platform` 命令：

```
const agentsPlatform =
  process.env.USER_TYPE === 'ant'
    ? require('./commands/agents-platform/index.js').default
    : null
```

与编译开关不同，此处使用 `require()` 而非 `feature()`——代码仍存在于产物中，只是运行时加载路径被短路为 `null`。

Sources: [commands.ts](src/commands.ts#L48-L52)

**形态二：功能分支的条件放行**。同一功能对内部用户和外部用户暴露不同的行为边界。最复杂的案例是 `modelSupportsAutoMode()`——内部用户走"拒绝列表"逻辑（默认允许，仅排除已知不支持的模型），外部用户走"允许列表"逻辑（默认拒绝，仅放行指定模型）：

```
if (process.env.USER_TYPE === 'ant') {
  // Denylist: block known-unsupported claude models, allow everything else
  if (m.includes('claude-3-')) return false
  if (/claude-(opus|sonnet|haiku)-4(?!-[6-9])/.test(m)) return false
  return true
}
// External allowlist (firstParty already checked above).
return /^claude-(opus|sonnet)-4-6/.test(m)
```

Sources: [utils/betas.ts](src/utils/betas.ts#L183-L193)

**形态三：Beta Header 的条件注入**。API 请求中某些 beta header 仅对内部用户附加，用于激活服务端尚未公开的能力。例如 `CLI_INTERNAL_BETA_HEADER` 的值根据 `USER_TYPE` 动态决定——外部用户获得空字符串，该 header 不会被发送：

```
export const CLI_INTERNAL_BETA_HEADER =
  process.env.USER_TYPE === 'ant' ? 'cli-internal-2026-02-09' : ''
```

Sources: [constants/betas.ts](src/constants/betas.ts#L29-L30)

## 第三层：远程 Feature Flag — GrowthBook 与灰度放量

### 运行时特性开关的加载链

远程 Feature Flag 在应用启动后从 GrowthBook 服务端拉取，缓存在本地供后续查询。核心 API 包括两个函数：

- **`checkStatsigFeatureGate_CACHED_MAY_BE_STALE`**：布尔型特性门，判断某个开关是否打开
- **`getFeatureValue_CACHED_MAY_BE_STALE`**：值型特性门，返回结构化配置对象

"缓存可能过期"的命名约定揭示了这层门控的关键设计折衷——Flag 值在应用生命周期中仅周期性刷新，单次查询可能返回过期数据。这是用一致性换取响应速度的典型策略。

Sources: [utils/betas.ts](src/utils/betas.ts#L4-L6)

### 叠加门控的典型决策路径

远程 Flag 通常不单独生效，而是叠加在前两层门控之上形成**三重判定链**。以自动模式（Auto Mode）的支持判定为例，完整路径如下：

1. **编译开关**：`feature('TRANSCRIPT_CLASSIFIER')` 必须为 `true`，否则整个函数直接返回 `false`
2. **用户类型**：内部用户可跳过 `firstParty` 提供商限制，外部用户仅限 `firstParty`
3. **远程 Flag**：`getFeatureValue_CACHED_MAY_BE_STALE('tengu_auto_mode_config')` 可强制放行特定模型，覆盖后续的名单判定

```
if (feature('TRANSCRIPT_CLASSIFIER')) {
  // 编译开关通过
  if (process.env.USER_TYPE !== 'ant' && getAPIProvider() !== 'firstParty') {
    return false  // 用户类型门控
  }
  const config = getFeatureValue_CACHED_MAY_BE_STALE('tengu_auto_mode_config', {})
  if (config?.allowModels?.some(am => ...)) {
    return true   // 远程 Flag 强制放行
  }
  // ... 继续执行名单判断
}
return false  // 编译开关未通过
```

Sources: [utils/betas.ts](src/utils/betas.ts#L160-L195)

## 三层门控的交叉关系与决策流程

```mermaid
flowchart TD
    A[功能请求到达] --> B{编译开关<br/>feature()}

    B -->|false| C[代码不存在<br/>功能彻底不可达]
    B -->|true| D{用户类型<br/>USER_TYPE}

    D -->|ant| E[内部通道<br/>Denylist 逻辑]
    D -->|非 ant| F[外部通道<br/>Allowlist 逻辑]

    E --> G{远程 Flag<br/>GrowthBook}
    F --> G

    G -->|放行| H[功能激活]
    G -->|拒绝| I[功能未开放]

    style C fill:#ff6b6b,color:#fff
    style H fill:#51cf66,color:#fff
    style I fill:#ffd43b,color:#333
```

三层门控的设计遵循**漏斗式收窄**原则：编译开关决定"代码是否存在于产物中"（最粗粒度），用户类型决定"哪些人群可见此功能"（中粒度），远程 Flag 决定"特定个体是否获得功能"（最细粒度）。每一层都是前一层的前置依赖——如果编译期代码已被消除，后两层无从谈起；如果用户类型不匹配，远程 Flag 的值再怎么设置也不会生效。

Sources: [commands.ts](src/commands.ts#L59-L123), [utils/betas.ts](src/utils/betas.ts#L160-L195)

## Beta Header：门控与 API 协议的交汇点

Beta Header 是门控体系与 API 协议的交汇面——它既是客户端门控决策的输出（决定哪些 header 被附加到请求中），也是服务端能力协商的输入（决定 API 返回哪些增强特性）。理解 Beta Header 的条件注入逻辑，等于掌握了客户端如何向服务端"声明自己有资格使用哪些未公开能力"。

| Beta Header | 门控条件 | 语义 |
|---|---|---|
| `cli-internal-2026-02-09` | `USER_TYPE === 'ant'` | CLI 内部特性集 |
| `summarize-connector-text-*` | `feature('CONNECTOR_TEXT')` | 连接器文本摘要 |
| `afk-mode-*` | `feature('TRANSCRIPT_CLASSIFIER')` | 离席模式（依赖对话分类器） |
| `claude-code-20250219` | 无条件 | Claude Code 基础协议版本 |
| `interleaved-thinking-*` | 模型能力判定 | 交织思维输出 |
| `context-1m-*` | 模型+用户+上下文联合判定 | 1M token 上下文窗口 |

其中 `context-1m` 的判定逻辑最为复杂——它需要同时考虑模型是否支持、提供商是否兼容、用户是否为 Claude.ai 订阅者、以及当前上下文是否确实需要 1M 扩展。这是三层门控叠加模型能力判定的四维决策。

Sources: [constants/betas.ts](src/constants/betas.ts#L1-L53), [utils/betas.ts](src/utils/betas.ts#L1-L200)

## SDK Beta 注入：门控体系的安全边界

对于通过 SDK 嵌入 Claude Code 的场景，门控体系引入了一道额外防线——`ALLOWED_SDK_BETAS` 白名单。即使 SDK 调用方尝试注入任意 beta header，也仅有白名单内（当前仅为 `context-1m`）的值会被保留：

```
const ALLOWED_SDK_BETAS = [CONTEXT_1M_BETA_HEADER]
```

更严格的是，若用户本身是 Claude.ai 订阅者（`isClaudeAISubscriber()`），则任何 SDK 自定义 beta 都会被忽略——因为订阅者的 beta 集由服务端决定，不允许客户端覆盖。这套安全边界确保了门控体系的最终仲裁权始终在服务端，而非客户端。

Sources: [utils/betas.ts](src/utils/betas.ts#L37-L87)

## 三层门控的对比总览

| 维度 | 编译开关 `feature()` | 用户类型 `USER_TYPE` | 远程 Flag GrowthBook |
|---|---|---|---|
| **生效时机** | 构建期 | 运行时（认证后） | 运行时（周期性刷新） |
| **门控粒度** | 功能模块级 | 用户群体级 | 个体/灰度级 |
| **可逆性** | 不可逆（需重新构建） | 可逆（重新认证） | 可逆（服务端即时变更） |
| **代码可见性** | 代码不存在于产物 | 代码存在但路径短路 | 代码存在且可执行但走不同分支 |
| **绕过难度** | 极高（需修改构建管线） | 中等（需伪造环境变量） | 低（需拦截网络响应） |
| **典型用途** | 未发布功能裁剪 | 内部工具隔离 | A/B 测试与渐进灰度 |

Sources: [commands.ts](src/commands.ts#L1-L200), [utils/betas.ts](src/utils/betas.ts#L1-L200), [constants/betas.ts](src/constants/betas.ts#L1-L53)

## 延伸阅读

- 关于门控体系在命令注册表中的完整实践，参见 [命令系统：斜杠命令注册、Feature Gate 与用户类型过滤](6-ming-ling-xi-tong-xie-gang-ming-ling-zhu-ce-feature-gate-yu-yong-hu-lei-xing-guo-lu)
- 关于通过隐藏 CLI 参数绕过部分门控的技巧，参见 [隐藏命令与秘密 CLI 参数全览](17-yin-cang-ming-ling-yu-mi-mi-cli-can-shu-quan-lan)
- 关于 Bridge 模式（受 `BRIDGE_MODE` 编译开关门控）的完整架构，参见 [Bridge：远程遥控终端的 WebSocket 双向通道](15-bridge-yuan-cheng-yao-kong-zhong-duan-de-websocket-shuang-xiang-tong-dao)
- 关于 Buddy（受 `BUDDY` 编译开关门控）的实现细节，参见 [Buddy：终端 AI 电子宠物系统](11-buddy-zhong-duan-ai-dian-zi-chong-wu-xi-tong)