# KAIROS 机制分析

> 基于 `@anthropic-ai/claude-code` v2.1.88 source map 还原源码的分析。

## 1. 概述

**KAIROS** 是 Claude Code 内部的 **助手模式（Assistant Mode）** 代号，是一个把 Claude CLI 从交互式终端工具进化为 **持久化自主 Agent 守护进程** 的架构层。启用后，Claude Code 不再是一个"用完就关"的命令行工具，而是可以：

- 作为后台守护进程（daemon）持续运行
- 通过远程 Bridge 会话接收消息
- 接收外部推送通知（MCP Channels）
- 按计划执行定时任务（Cron）
- 自动整理记忆（Dream/Memory Consolidation）
- 通过 `SendUserMessage` 工具主动向用户发送消息

涉及文件数：**约 69 个文件** 分布在 commands、tools、services、hooks、utils、components 等各个子系统中。

---

## 2. 特性标志（Feature Flags）

KAIROS 通过 Bun 的编译时 `feature()` 函数实现 **死代码消除（DCE）**。非 KAIROS 构建中，所有相关代码在编译时被完全移除。

| Feature Flag | 说明 | 控制范围 |
|---|---|---|
| `feature('KAIROS')` | **主标志**，控制整个助手模式 | 核心激活、会话发现、远程查看、团队初始化 |
| `feature('KAIROS_BRIEF')` | Brief 工具独立发布标志 | `SendUserMessage` 工具单独可用 |
| `feature('KAIROS_CHANNELS')` | MCP Channel 推送通知 | 入站推送消息、`--channels` CLI 选项 |
| `feature('KAIROS_GITHUB_WEBHOOKS')` | GitHub PR 订阅 | `/subscribe-pr` 命令、SubscribePRTool |
| `feature('KAIROS_PUSH_NOTIFICATION')` | 推送通知 | OS 级别推送通知 |
| `feature('KAIROS_DREAM')` | 自动梦境/记忆整理 | autoDream 服务 |
| `feature('AGENT_TRIGGERS')` | Cron 调度系统 | CronCreate/CronDelete/CronList 工具 |

组合使用模式：
```typescript
// KAIROS 自身依赖 Brief，所以单独 KAIROS 也需要 Bundle 它
feature('KAIROS') || feature('KAIROS_BRIEF')

// 主动模式 + KAIROS 共享 Sleep/Proactive 逻辑
feature('PROACTIVE') || feature('KAIROS')
```

---

## 3. 核心架构

### 3.1 启动流程

```
main.tsx
├── 1. 解析 `claude assistant [sessionId]` → 设置 _pendingAssistantChat
├── 2. feature('KAIROS') 条件加载：
│   ├── require('./assistant/index.js')   → assistantModule
│   └── require('./assistant/gate.js')    → kairosGate
├── 3. 权限门控检查：
│   ├── --assistant 标志 → markAssistantForced()（跳过 Gate）
│   ├── isAssistantMode()（.claude/settings.json → assistant: true）
│   ├── 信任对话框检查（防止未信任目录激活）
│   └── isKairosEnabled()（GrowthBook tengu_kairos gate）
├── 4. 激活：
│   ├── setKairosActive(true)
│   ├── opts.brief = true（强制启用 Brief）
│   └── initializeAssistantTeam()（初始化 Agent 团队）
├── 5. 系统提示词追加：
│   └── getAssistantSystemPromptAddendum()
└── 6. REPL 启动：
    └── initialState.kairosEnabled = true
```

**关键代码** (`main.tsx:1048-1087`)：
```typescript
let kairosEnabled = false;
// --assistant 标志（Agent SDK daemon 模式）
if (feature('KAIROS') && options.assistant && assistantModule) {
  assistantModule.markAssistantForced();
}
// settings.json 中 assistant: true 且不是子代理
if (feature('KAIROS') && assistantModule?.isAssistantMode() && !options.agentId && kairosGate) {
  if (!checkHasTrustDialogAccepted()) {
    console.warn('Assistant mode disabled: directory is not trusted.');
  } else {
    kairosEnabled = assistantModule.isAssistantForced() || await kairosGate.isKairosEnabled();
    if (kairosEnabled) {
      opts.brief = true;
      setKairosActive(true);
      assistantTeamContext = await assistantModule.initializeAssistantTeam();
    }
  }
}
```

### 3.2 Gate 机制

KAIROS 的权限检查分两层：

**编译时层**：`feature('KAIROS')` → 非 KAIROS 构建完全消除代码

**运行时层**（`assistant/gate.js`）：
- `isKairosEnabled()` → 检查 GrowthBook `tengu_kairos` gate
- 使用磁盘缓存：已确认的用户瞬间返回 `true`
- 未缓存时懒初始化 GrowthBook 并请求（最长 ~5 秒）
- `--assistant` 标志完全跳过 Gate（daemon 已预先检查权限）

**激活前置条件**：
1. `.claude/settings.json` 包含 `assistant: true`
2. 目录已通过信任对话框确认
3. GrowthBook `tengu_kairos` gate 为 true（或 `--assistant` 跳过）
4. 非子代理进程（无 `--agent-id`）

### 3.3 状态管理

KAIROS 在两个层面管理状态：

**Bootstrap State**（`bootstrap/state.ts`）：
```typescript
// 进程级全局单例
kairosActive: boolean  // getKairosActive() / setKairosActive()
```

**AppState**（`state/AppStateStore.ts`）：
```typescript
// REPL 级别 Store
kairosEnabled: boolean  // 一次性设置，不可变
isBriefOnly: boolean    // Brief 视图模式
replBridgeEnabled: boolean  // 远程 Bridge 控制
```

---

## 4. 子系统详解

### 4.1 SendUserMessage（Brief 工具）

**文件**：`tools/BriefTool/BriefTool.ts`、`tools/BriefTool/prompt.ts`

**作用**：KAIROS 的 **主要用户可见输出通道**。启用后，模型的所有面向用户的回复都必须通过此工具发送，工具外的纯文本输出对用户不可见。

**输入 Schema**：
```typescript
{
  message: string,       // 支持 Markdown
  attachments?: string[], // 文件路径（绝对或相对 cwd）
  status: 'normal' | 'proactive'  // 正常回复 vs 主动通知
}
```

**门控逻辑**：
```
isBriefEnabled()
├── feature('KAIROS') || feature('KAIROS_BRIEF')  ← 编译时
├── getKairosActive() || getUserMsgOptIn()          ← 运行时激活
└── isBriefEntitled()                                ← 权限检查
    ├── getKairosActive()                            ← KAIROS 模式
    ├── CLAUDE_CODE_BRIEF env                        ← 开发测试
    └── tengu_kairos_brief GB gate                   ← 远程开关
```

**系统提示词指令**（`prompt.ts:12-22`）：
```
SendUserMessage is where your replies go. Text outside it is visible if the
user expands the detail view, but most won't — assume unread.

Every time the user says something, the reply they actually read comes through
SendUserMessage. Even for "hi". Even for "thanks".

For longer work: ack → work → result.
```

**激活路径**：
- `--brief` CLI 标志
- `.claude/settings.json` → `defaultView: 'chat'`
- `/brief` 斜杠命令
- `/config` 中的 defaultView 选择器
- `--tools` SDK 选项中包含 SendUserMessage
- `CLAUDE_CODE_BRIEF` 环境变量
- KAIROS 模式（自动强制激活）

### 4.2 Cron 调度系统

**文件**：`utils/cronScheduler.ts`、`utils/cronTasks.ts`、`tools/ScheduleCronTool/prompt.ts`

**作用**：允许 Claude 按计划执行定时任务（一次性或重复）。

**任务存储**：
```
.claude/scheduled_tasks.json
{
  "tasks": [{
    id: string,        // 8位 hex UUID
    cron: string,       // 5 字段 cron（本地时区）
    prompt: string,     // 触发时执行的提示
    createdAt: number,  // 创建时间戳
    lastFiredAt?: number,
    recurring?: boolean,
    permanent?: boolean, // 不会过期（仅 assistant 安装任务）
    durable?: boolean    // 运行时标志：false = 仅会话内
  }]
}
```

**两种任务类型**：
- **Session-only**（`durable: false`）：仅存在于当前 Claude 进程中
- **Durable**（持久化）：写入 `.claude/scheduled_tasks.json`，进程重启后恢复

**调度器行为**：
- 每 1 秒检查一次
- 仅在 REPL 空闲时触发（非查询中）
- 使用 chokidar 监视文件变化
- 跨进程锁机制（只有一个进程持有调度权）
- 有抖动（jitter）机制防止 thundering herd

**门控**：
```typescript
isKairosCronEnabled()
├── feature('AGENT_TRIGGERS')       ← 编译时
├── !CLAUDE_CODE_DISABLE_CRON env   ← 本地禁用
└── tengu_kairos_cron GB gate       ← 远程开关（默认 true）
```

**抖动配置**（可通过 `tengu_kairos_cron_config` GrowthBook 远程调整）：
- 重复任务：前向延迟 = cron 两次触发间隔 × 10%，上限 15 分钟
- 一次性任务：在 `:00`/`:30` 分钟上触发时，最多提前 90 秒
- 重复任务 7 天后自动过期（`permanent` 标记除外）

### 4.3 Channel 通知系统

**文件**：`services/mcp/channelNotification.ts`

**作用**：允许外部 MCP 服务器（Discord、Slack、SMS 等）向 Claude 会话推送入站消息。

**协议**：
```
MCP Server → notifications/claude/channel → Claude Code
                                              ↓
                        包装为 <channel source="serverName">内容</channel>
                                              ↓
                        加入消息队列 → SleepTool 1 秒内唤醒
                                              ↓
                        模型决定用哪个工具回复
```

**权限审批流程**：
```
Channel 服务器声明 capabilities.experimental['claude/channel']
  → 运行时 gate（tengu_harbor）
    → OAuth 认证检查（仅 claude.ai 用户）
      → 组织策略检查（Team/Enterprise 需 channelsEnabled: true）
        → --channels 会话选项匹配
          → 白名单检查
            → 注册通知处理器
```

**Channel Permission 协议**（双向）：
- **出站**：`notifications/claude/channel/permission_request` → 发送权限请求到 Channel 服务器
- **入站**：`notifications/claude/channel/permission` → Channel 服务器返回用户审批结果（allow/deny）

### 4.4 Session History（会话历史）

**文件**：`assistant/sessionHistory.ts`、`hooks/useAssistantHistory.ts`

**作用**：`claude assistant [sessionId]` 命令的远程会话历史分页加载。

**API**：
```
GET /v1/sessions/{sessionId}/events
Headers:
  - OAuth token
  - anthropic-beta: ccr-byoc-2025-07-29
  - x-organization-uuid
Params:
  - limit: 100
  - anchor_to_latest: true   (最新页)
  - before_id: string        (更早页)
```

**UI 行为**：
- 挂载时：`fetchLatestEvents()` 获取最新一页
- 滚动到顶部（< 40 行阈值）：`fetchOlderEvents()` 获取更早页
- 内容不足填满视口时：自动链式加载（最多 10 页）
- 加载中显示哨兵消息（"loading older messages…"）
- 加载完成显示 "start of session"
- 滚动锚点补偿：在 `useLayoutEffect` 中保持视口位置不变

### 4.5 Auto-Dream（自动记忆整理）

**文件**：`services/autoDream/autoDream.ts`、`services/autoDream/consolidationPrompt.ts`

**作用**：作为后台子代理自动整理记忆文件。在助手模式中，这是 `.claude/scheduled_tasks.json` 中的永久定时任务。

**触发门控**（从便宜到贵排序）：
1. **时间**：距上次整理 ≥ minHours
2. **会话数**：在上次整理之后修改的会话 transcript ≥ minSessions
3. **锁**：无其他进程正在整理

**整理流程**：
```
Phase 1 — 定向：ls 记忆目录、读取 index.md
Phase 2 — 收集信号：日志、已有记忆、transcript 搜索
Phase 3 — 合并：写入/更新记忆文件
Phase 4 — 裁剪和索引：更新 index.md（≤ 25KB）
```

### 4.6 Bridge 远程会话

**文件**：`bridge/bridgeMain.ts`

**作用**：将 Claude Code REPL 连接到远程 Bridge 会话（CCR — Claude Code Remote）。

在 KAIROS 模式下：
- `claude assistant [sessionId]` → REPL 作为纯查看客户端
- 代理循环在远程运行
- 本地 REPL 流式传输实时事件 + 通过 POST 发送消息
- 使用 `useAssistantHistory` 延迟加载历史

**Bridge 架构**：
```
本地 REPL (viewerOnly)
  ↕ HTTP streaming / POST
远程 Bridge Session (daemon)
  ↕ API calls
Claude API
```

---

## 5. Settings 集成

### `.claude/settings.json` 中的 KAIROS 相关设置

```json
{
  "assistant": true,           // 启用助手模式
  "assistantName": "...",      // 助手显示名（claude.ai 会话列表）
  "defaultView": "chat",      // 默认 Brief 视图
  "channelsEnabled": true,     // 启用 Channel 推送（企业级）
  "allowedChannelPlugins": []  // 允许的 Channel 插件白名单（企业级）
}
```

---

## 6. CLI 命令和选项

| 命令/选项 | Feature Guard | 说明 |
|---|---|---|
| `claude assistant [sessionId]` | `KAIROS` | 连接到远程助手会话 |
| `--assistant` | `KAIROS` | 强制助手模式（Agent SDK daemon） |
| `--brief` | `KAIROS \|\| KAIROS_BRIEF` | 启用 SendUserMessage 工具 |
| `--proactive` | `PROACTIVE \|\| KAIROS` | 启动主动自主模式 |
| `--channels <servers...>` | `KAIROS \|\| KAIROS_CHANNELS` | MCP Channel 推送注册 |
| `/brief` | `KAIROS \|\| KAIROS_BRIEF` | 切换 Brief 模式 |
| `/subscribe-pr` | `KAIROS_GITHUB_WEBHOOKS` | 订阅 GitHub PR 事件 |

---

## 7. GrowthBook Gate 汇总

| Gate 名称 | 默认值 | TTL | 用途 |
|---|---|---|---|
| `tengu_kairos` | — | 阻塞检查 | 主 KAIROS 权限 gate |
| `tengu_kairos_brief` | `false` | 5 分钟 | Brief 工具权限 |
| `tengu_kairos_brief_config` | `{enable_slash_command: false}` | — | Brief 配置（斜杠命令可见性） |
| `tengu_kairos_cron` | `true` | 5 分钟 | Cron 调度系统开关/杀开关 |
| `tengu_kairos_cron_durable` | `true` | 5 分钟 | 持久 Cron 任务开关 |
| `tengu_kairos_cron_config` | 见下文 | — | Cron 抖动配置 |
| `tengu_harbor` | — | — | Channel 通知总开关 |

---

## 8. 架构总结图

```
                   ┌─────────────────────────────────────┐
                   │          Feature Flags (DCE)         │
                   │  KAIROS · KAIROS_BRIEF · KAIROS_     │
                   │  CHANNELS · AGENT_TRIGGERS · ...     │
                   └────────────────┬────────────────────┘
                                    │
                   ┌────────────────▼────────────────────┐
                   │        Gate / Eligibility            │
                   │  assistant/gate.js                   │
                   │  ├─ GrowthBook: tengu_kairos         │
                   │  ├─ .claude/settings.json: assistant │
                   │  └─ Trust dialog check               │
                   └────────────────┬────────────────────┘
                                    │
               ┌────────────────────▼───────────────────────┐
               │              main.tsx 启动                  │
               │  setKairosActive(true)                      │
               │  initializeAssistantTeam()                  │
               │  getAssistantSystemPromptAddendum()         │
               └──┬─────────┬──────────┬──────────┬────────┘
                  │         │          │          │
     ┌────────────▼──┐ ┌───▼────┐ ┌───▼────┐ ┌──▼──────────┐
     │ SendUserMessage│ │  Cron  │ │Channel │ │   Bridge     │
     │ (BriefTool)    │ │Scheduler│ │Notifs │ │Remote Session│
     │                │ │         │ │        │ │              │
     │ 主要输出通道   │ │ 定时任务│ │入站推送│ │远程会话查看  │
     └────────────────┘ └────────┘ └────────┘ └─────────────┘
                                                     │
                                          ┌──────────▼──────────┐
                                          │ Session History API  │
                                          │ /v1/sessions/…/events│
                                          │ 分页 · 延迟加载     │
                                          └─────────────────────┘
                                                     │
                                          ┌──────────▼──────────┐
                                          │  Auto-Dream          │
                                          │  记忆整理子代理      │
                                          │  .claude/memory/     │
                                          └─────────────────────┘
```

---

## 9. 安全注意事项

1. **Trust Gate**：`.claude/settings.json` 是攻击者可控的（未信任克隆）。在信任对话框出现之前，已有 ~1000 行代码执行。代码强制要求信任对话框已接受才能激活助手模式。
2. **Channel 多层门控**：capability → runtime gate → OAuth auth → org policy → session opt-in → allowlist
3. **Marketplace 验证**：Channel 插件需要验证安装来源与 --channels 标签匹配
4. **Cron 锁**：跨进程调度器锁，防止多个进程同时执行任务
5. **Durable Cron 权限**：`permanent` 标志仅可由 `assistant/install.ts` 直接写入，不可通过 CronCreateTool 设置

---

## 10. 源码中被 DCE 消除的文件（仅在引用中出现）

以下文件在代码中被 `require()` 引用，但因为 `feature('KAIROS')` 在构建时为 `false` 被消除，不在还原的 source map 中：

| 文件 | 导出/用途 |
|---|---|
| `assistant/index.js` | `isAssistantMode()`, `isAssistantForced()`, `markAssistantForced()`, `initializeAssistantTeam()`, `getAssistantSystemPromptAddendum()`, `getAssistantActivationPath()` |
| `assistant/gate.js` | `isKairosEnabled()` |
| `assistant/sessionDiscovery.js` | `discoverAssistantSessions()` |
| `assistant/AssistantSessionChooser.js` | 会话选择 UI 组件 |
| `assistant/sessionTranscript.js` | 会话 Transcript 查看 |
| `commands/assistant/index.js` | assistant 命令入口 |
| `commands/assistant/assistant.js` | assistant 命令实现 |

唯一幸存的 assistant 模块文件是 `assistant/sessionHistory.ts`（被 `useAssistantHistory` 直接引入，未被 feature gate 包裹）。
