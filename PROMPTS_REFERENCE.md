# OpenClaw 提示词系统完整参考

## 目录

1. [概览](#概览)
2. [提示词模式](#提示词模式)
3. [硬编码提示词部分](#硬编码提示词部分)
4. [工作空间上下文文件](#工作空间上下文文件)
5. [特殊提示词](#特殊提示词)
6. [提示词参数结构](#提示词参数结构)
7. [提示词执行流程](#提示词执行流程)
8. [特殊标记](#特殊标记)
9. [工作空间文件位置](#工作空间文件位置)
10. [配置相关](#配置相关)

---

## 概览

OpenClaw 的提示词系统包含三个核心层级：

1. **硬编码提示词部分** - 在代码中定义的固定提示词
2. **工作空间上下文文件** - 用户编辑的 markdown 文件
3. **特殊提示词** - 心跳、额外上下文等动态提示词

### 核心源文件

| 文件 | 用途 | 位置 |
|------|------|------|
| **system-prompt.ts** | 系统提示词构建器（24 个部分） | `src/agents/system-prompt.ts` |
| **workspace.ts** | 工作空间文件加载器 | `src/agents/workspace.ts` |
| **heartbeat.ts** | 心跳提示词处理 | `src/auto-reply/heartbeat.ts` |
| **system-prompt-params.ts** | 运行时参数构建 | `src/agents/system-prompt-params.ts` |

---

## 提示词模式

### PromptMode 类型定义

**源文件**：`src/agents/system-prompt.ts:8-14`

```typescript
export type PromptMode = "full" | "minimal" | "none";
```

| 模式 | 描述 | 使用场景 | 包含部分 |
|------|------|---------|---------|
| **full** | 完整模式（默认） | 主 agent | 所有 24 个部分 |
| **minimal** | 最小化模式 | 子 agent (subagent) | 仅 Tooling、Workspace、Runtime |
| **none** | 无部分模式 | 特殊场景 | 仅基础身份行 |

**调用位置**：
- `src/agents/pi-embedded-runner/run/attempt.ts` - 根据会话类型设置 promptMode
- `src/agents/system-prompt.ts:348` - 提示词构建时使用

---

## 硬编码提示词部分

这些部分由 `buildAgentSystemPrompt()` 函数（`src/agents/system-prompt.ts:164-608`）自动组装。

### 1. 基础身份 (Basic Identity)

**源位置**：`src/agents/system-prompt.ts:376-380`

**代码**：
```typescript
const lines = [
  "You are a personal assistant running inside OpenClaw.",
  "",
  ...
]
```

| 属性 | 值 |
|------|-----|
| 内容 | "You are a personal assistant running inside OpenClaw." |
| 作用 | 定义 agent 基本身份 |
| 模式 | full / minimal / none（始终包含） |
| 条件 | 无（始终）|
| 参数 | 无 |

---

### 2. Tooling 工具部分

**源位置**：`src/agents/system-prompt.ts:382-406`

**代码**：
```typescript
lines.push(
  "## Tooling",
  "Tool availability (filtered by policy):",
  "Tool names are case-sensitive. Call tools exactly as listed.",
  toolLines.join("\n"),
  "TOOLS.md does not control tool availability; it is user guidance for how to use external tools.",
  "If a task is more complex or takes longer, spawn a sub-agent. It will do the work for you and ping you when it's done. You can always check up on it.",
  "",
)
```

| 属性 | 值 |
|------|-----|
| 作用 | 列出可用工具及其描述 |
| 模式 | full / minimal |
| 条件 | 始终 |
| 参数 | `toolNames?: string[]`、`toolSummaries?: Record<string, string>` |
| 工具列表位置 | 行 247-271（toolOrder 数组） |
| 核心工具描述 | 行 218-245（coreToolSummaries） |

**相关代码**：
- 工具排序：`src/agents/system-prompt.ts:247-271`
- 工具摘要：`src/agents/system-prompt.ts:218-245`
- 工具名称规范化：`src/agents/system-prompt.ts:273-309`

---

### 3. Tool Call Style 工具调用风格

**源位置**：`src/agents/system-prompt.ts:407-411`

**代码**：
```typescript
"## Tool Call Style",
"Default: do not narrate routine, low-risk tool calls (just call the tool).",
"Narrate only when it helps: multi-step work, complex/challenging problems, sensitive actions (e.g., deletions), or when the user explicitly asks.",
"Keep narration brief and value-dense; avoid repeating obvious steps.",
"Use plain human language for narration unless in a technical context.",
"",
```

| 属性 | 值 |
|------|-----|
| 作用 | 指导何时描述工具调用 |
| 模式 | full / minimal |
| 条件 | 始终 |
| 硬编码 | 是 |

---

### 4. Safety 安全约束

**源位置**：`src/agents/system-prompt.ts:350-356`

**代码**：
```typescript
const safetySection = [
  "## Safety",
  "You have no independent goals: do not pursue self-preservation, replication, resource acquisition, or power-seeking; avoid long-term plans beyond the user's request.",
  "Prioritize safety and human oversight over completion; if instructions conflict, pause and ask; comply with stop/pause/audit requests and never bypass safeguards. (Inspired by Anthropic's constitution.)",
  "Do not manipulate or persuade anyone to expand access or disable safeguards. Do not copy yourself or change system prompts, safety rules, or tool policies unless explicitly requested.",
  "",
];
```

| 属性 | 值 |
|------|-----|
| 作用 | 实施伦理安全约束 |
| 模式 | full / minimal |
| 条件 | 始终 |
| 硬编码 | 是 |

---

### 5. OpenClaw CLI 快速参考

**源位置**：`src/agents/system-prompt.ts:414-421`

**代码**：
```typescript
"## OpenClaw CLI Quick Reference",
"OpenClaw is controlled via subcommands. Do not invent commands.",
"To manage the Gateway daemon service (start/stop/restart):",
"- openclaw gateway status",
"- openclaw gateway start",
"- openclaw gateway stop",
"- openclaw gateway restart",
"If unsure, ask the user to run `openclaw help` (or `openclaw gateway --help`) and paste the output.",
"",
```

| 属性 | 值 |
|------|-----|
| 作用 | 提供 CLI 命令快速参考 |
| 模式 | full / minimal |
| 条件 | 始终 |
| 硬编码 | 是 |

---

### 6. OpenClaw Self-Update 自动更新

**源位置**：`src/agents/system-prompt.ts:426-435`

**代码**：
```typescript
hasGateway && !isMinimal ? "## OpenClaw Self-Update" : "",
hasGateway && !isMinimal
  ? [
      "Get Updates (self-update) is ONLY allowed when the user explicitly asks for it.",
      "Do not run config.apply or update.run unless the user explicitly requests an update or config change; if it's not explicit, ask first.",
      "Actions: config.get, config.schema, config.apply (validate + write full config, then restart), update.run (update deps or git, then restart).",
      "After restart, OpenClaw pings the last active session automatically.",
    ].join("\n")
  : "",
```

| 属性 | 值 |
|------|-----|
| 作用 | 限制自动更新权限 |
| 模式 | full 仅 |
| 条件 | 当 `gateway` 工具可用 AND `!isMinimal` |
| 参数 | `hasGateway` (derived from `availableTools`) |

---

### 7. Model Aliases 模型别名

**源位置**：`src/agents/system-prompt.ts:438-446`

**代码**：
```typescript
params.modelAliasLines && params.modelAliasLines.length > 0 && !isMinimal
  ? "## Model Aliases"
  : "",
params.modelAliasLines && params.modelAliasLines.length > 0 && !isMinimal
  ? "Prefer aliases when specifying model overrides; full provider/model is also accepted."
  : "",
params.modelAliasLines && params.modelAliasLines.length > 0 && !isMinimal
  ? params.modelAliasLines.join("\n")
  : "",
```

| 属性 | 值 |
|------|-----|
| 作用 | 教 agent 如何使用模型别名 |
| 模式 | full 仅 |
| 条件 | `modelAliasLines` 非空 AND `!isMinimal` |
| 参数 | `modelAliasLines?: string[]` |
| 配置来源 | `config.agents.defaults.modelAliases` |

---

### 8. 当前时间提示

**源位置**：`src/agents/system-prompt.ts:448-450`

**代码**：
```typescript
userTimezone
  ? "If you need the current date, time, or day of week, run session_status (📊 session_status)."
  : "",
```

| 属性 | 值 |
|------|-----|
| 作用 | 告诉 agent 如何获取当前时间 |
| 模式 | full / minimal |
| 条件 | 当 `userTimezone` 已设置 |
| 参数 | `userTimezone?: string` |
| 来源 | `config.agents.defaults.userTimezone` |

**时间构建相关代码**：
- `src/agents/system-prompt-params.ts:45` - 解析用户时区
- `src/agents/system-prompt-params.ts:46` - 解析时间格式
- `src/agents/date-time.ts` - 时间相关工具函数

---

### 9. Workspace 工作目录

**源位置**：`src/agents/system-prompt.ts:451-454`

**代码**：
```typescript
"## Workspace",
`Your working directory is: ${params.workspaceDir}`,
"Treat this directory as the single global workspace for file operations unless explicitly instructed otherwise.",
...workspaceNotes,
"",
```

| 属性 | 值 |
|------|-----|
| 作用 | 指定 agent 工作目录 |
| 模式 | full / minimal |
| 条件 | 始终 |
| 参数 | `workspaceDir: string`（必需）、`workspaceNotes?: string[]` |

---

### 10. Documentation 文档链接

**源位置**：`src/agents/system-prompt.ts:146-161`

**函数**：`buildDocsSection()`

**代码**：
```typescript
function buildDocsSection(params: {
  docsPath?: string;
  isMinimal: boolean;
  readToolName: string
}) {
  const docsPath = params.docsPath?.trim();
  if (!docsPath || params.isMinimal) {
    return [];
  }
  return [
    "## Documentation",
    `OpenClaw docs: ${docsPath}`,
    "Mirror: https://docs.openclaw.ai",
    "Source: https://github.com/openclaw/openclaw",
    "Community: https://discord.com/invite/clawd",
    "Find new skills: https://clawhub.com",
    "For OpenClaw behavior, commands, config, or architecture: consult local docs first.",
    "When diagnosing issues, run `openclaw status` yourself when possible; only ask the user if you lack access (e.g., sandboxed).",
    "",
  ];
}
```

| 属性 | 值 |
|------|-----|
| 作用 | 提供文档和社区资源 |
| 模式 | full 仅 |
| 条件 | `docsPath` 已设置 AND `!isMinimal` |
| 参数 | `docsPath?: string`、`readToolName: string` |

---

### 11. Skills 技能部分

**源位置**：`src/agents/system-prompt.ts:16-37`

**函数**：`buildSkillsSection()`

**代码**：
```typescript
function buildSkillsSection(params: {
  skillsPrompt?: string;
  isMinimal: boolean;
  readToolName: string;
}) {
  if (params.isMinimal) {
    return [];
  }
  const trimmed = params.skillsPrompt?.trim();
  if (!trimmed) {
    return [];
  }
  return [
    "## Skills (mandatory)",
    "Before replying: scan <available_skills> <description> entries.",
    `- If exactly one skill clearly applies: read its SKILL.md at <location> with \`${params.readToolName}\`, then follow it.`,
    "- If multiple could apply: choose the most specific one, then read/follow it.",
    "- If none clearly apply: do not read any SKILL.md.",
    "Constraints: never read more than one skill up front; only read after selecting.",
    trimmed,
    "",
  ];
}
```

| 属性 | 值 |
|------|-----|
| 作用 | 教 agent 如何选择和使用技能 |
| 模式 | full 仅 |
| 条件 | 有 `skillsPrompt` AND `!isMinimal` |
| 参数 | `skillsPrompt?: string`、`readToolName: string` |
| 来源 | 由 skills.ts 中的 resolveSkillsPromptForRun() 构建 |

**相关文件**：
- `src/agents/skills.ts` - 技能管理
- 文件名模式：`SKILL.md` (用户工作空间)

---

### 12. Memory Recall 记忆回忆

**源位置**：`src/agents/system-prompt.ts:40-66`

**函数**：`buildMemorySection()`

**代码**：
```typescript
function buildMemorySection(params: {
  isMinimal: boolean;
  availableTools: Set<string>;
  citationsMode?: MemoryCitationsMode;
}) {
  if (params.isMinimal) {
    return [];
  }
  if (!params.availableTools.has("memory_search") && !params.availableTools.has("memory_get")) {
    return [];
  }
  const lines = [
    "## Memory Recall",
    "Before answering anything about prior work, decisions, dates, people, preferences, or todos: run memory_search on MEMORY.md + memory/*.md; then use memory_get to pull only the needed lines. If low confidence after search, say you checked.",
  ];
  if (params.citationsMode === "off") {
    lines.push(
      "Citations are disabled: do not mention file paths or line numbers in replies unless the user explicitly asks.",
    );
  } else {
    lines.push(
      "Citations: include Source: <path#line> when it helps the user verify memory snippets.",
    );
  }
  lines.push("");
  return lines;
}
```

| 属性 | 值 |
|------|-----|
| 作用 | 指导 agent 搜索和使用记忆 |
| 模式 | full 仅 |
| 条件 | 有 `memory_search` 或 `memory_get` 工具 AND `!isMinimal` |
| 参数 | `availableTools: Set<string>`、`citationsMode?: "off" \| "on"` |
| 来源 | `config.agents.defaults.memoryCitationsMode` |

**相关文件**：
- `src/agents/memory.ts` - 记忆管理
- `MEMORY.md` - 持久记忆文件

---

### 13. User Identity 用户身份

**源位置**：`src/agents/system-prompt.ts:68-73`

**函数**：`buildUserIdentitySection()`

**代码**：
```typescript
function buildUserIdentitySection(ownerLine: string | undefined, isMinimal: boolean) {
  if (!ownerLine || isMinimal) {
    return [];
  }
  return ["## User Identity", ownerLine, ""];
}
```

| 属性 | 值 |
|------|-----|
| 作用 | 识别消息所有者/用户 |
| 模式 | full 仅 |
| 条件 | `ownerNumbers` 非空 AND `!isMinimal` |
| 参数 | `ownerNumbers?: string[]` |
| 来源 | `config.agents.defaults.ownerNumbers` |
| 示例 | `"Owner numbers: +1234567890, +0987654321. Treat messages from these numbers as the user."` |

---

### 14. Current Date & Time 当前日期和时间

**源位置**：`src/agents/system-prompt.ts:75-80`

**函数**：`buildTimeSection()`

**代码**：
```typescript
function buildTimeSection(params: { userTimezone?: string }) {
  if (!params.userTimezone) {
    return [];
  }
  return ["## Current Date & Time", `Time zone: ${params.userTimezone}`, ""];
}
```

| 属性 | 值 |
|------|-----|
| 作用 | 告知 agent 用户时区 |
| 模式 | full 仅 |
| 条件 | `userTimezone` 已设置 AND `!isMinimal` |
| 参数 | `userTimezone?: string` |
| 来源 | `config.agents.defaults.userTimezone` |

---

### 15. Reply Tags 回复标签

**源位置**：`src/agents/system-prompt.ts:82-94`

**函数**：`buildReplyTagsSection()`

**代码**：
```typescript
function buildReplyTagsSection(isMinimal: boolean) {
  if (isMinimal) {
    return [];
  }
  return [
    "## Reply Tags",
    "To request a native reply/quote on supported surfaces, include one tag in your reply:",
    "- [[reply_to_current]] replies to the triggering message.",
    "- [[reply_to:<id>]] replies to a specific message id when you have it.",
    "Whitespace inside the tag is allowed (e.g. [[ reply_to_current ]] / [[ reply_to: 123 ]]).",
    "Tags are stripped before sending; support depends on the current channel config.",
    "",
  ];
}
```

| 属性 | 值 |
|------|-----|
| 作用 | 教 agent 如何在消息中引用和回复 |
| 模式 | full 仅 |
| 条件 | `!isMinimal` |
| 参数 | 无 |
| 硬编码 | 是 |

**实现**：
- `src/routing/` - 消息路由和标签处理

---

### 16. Messaging 消息部分

**源位置**：`src/agents/system-prompt.ts:97-133`

**函数**：`buildMessagingSection()`

**参数**：
```typescript
{
  isMinimal: boolean;
  availableTools: Set<string>;
  messageChannelOptions: string;
  inlineButtonsEnabled: boolean;
  runtimeChannel?: string;
  messageToolHints?: string[];
}
```

**内容**：
```typescript
"## Messaging",
"- Reply in current session → automatically routes to the source channel (Signal, Telegram, etc.)",
"- Cross-session messaging → use sessions_send(sessionKey, message)",
"- Never use exec/curl for provider messaging; OpenClaw handles all routing internally.",
"- Use `message` for proactive sends + channel actions (polls, reactions, etc.).",
...
```

| 属性 | 值 |
|------|-----|
| 作用 | 指导跨频道消息传递和消息工具 |
| 模式 | full 仅 |
| 条件 | `!isMinimal` |
| 参数 | `availableTools`、`messageChannelOptions`、`inlineButtonsEnabled`、`runtimeChannel`、`messageToolHints` |
| 来源 | `src/utils/message-channel.js:listDeliverableMessageChannels()` |

**相关文件**：
- `src/channels/` - 频道实现

---

### 17. Voice (TTS) 语音部分

**源位置**：`src/agents/system-prompt.ts:135-143`

**函数**：`buildVoiceSection()`

**代码**：
```typescript
function buildVoiceSection(params: { isMinimal: boolean; ttsHint?: string }) {
  if (params.isMinimal) {
    return [];
  }
  const hint = params.ttsHint?.trim();
  if (!hint) {
    return [];
  }
  return ["## Voice (TTS)", hint, ""];
}
```

| 属性 | 值 |
|------|-----|
| 作用 | 提供 TTS（文本到语音）功能指导 |
| 模式 | full 仅 |
| 条件 | 有 `ttsHint` AND `!isMinimal` |
| 参数 | `ttsHint?: string` |
| 来源 | 由 TTS 相关配置提供 |

---

### 18. Sandbox 沙箱部分

**源位置**：`src/agents/system-prompt.ts:457-498`

**代码**：
```typescript
params.sandboxInfo?.enabled ? "## Sandbox" : "",
params.sandboxInfo?.enabled
  ? [
      "You are running in a sandboxed runtime (tools execute in Docker).",
      "Some tools may be unavailable due to sandbox policy.",
      "Sub-agents stay sandboxed (no elevated/host access). Need outside-sandbox read/write? Don't spawn; ask first.",
      params.sandboxInfo.workspaceDir
        ? `Sandbox workspace: ${params.sandboxInfo.workspaceDir}`
        : "",
      params.sandboxInfo.workspaceAccess
        ? `Agent workspace access: ${params.sandboxInfo.workspaceAccess}...`
        : "",
      params.sandboxInfo.browserBridgeUrl ? "Sandbox browser: enabled." : "",
      // ... 更多沙箱信息
    ]
      .filter(Boolean)
      .join("\n")
  : "",
```

| 属性 | 值 |
|------|-----|
| 作用 | 提供沙箱运行时约束和能力信息 |
| 模式 | full / minimal |
| 条件 | `sandboxInfo.enabled === true` |
| 参数 | 详见下表 |

**sandboxInfo 结构**：

| 参数 | 类型 | 作用 |
|------|------|------|
| `enabled` | boolean | 是否启用沙箱 |
| `workspaceDir` | string? | 沙箱工作目录 |
| `workspaceAccess` | "none" \| "ro" \| "rw"? | 工作空间访问权限 |
| `agentWorkspaceMount` | string? | agent 工作空间挂载点 |
| `browserBridgeUrl` | string? | 浏览器桥接 URL |
| `browserNoVncUrl` | string? | 浏览器 noVNC URL |
| `hostBrowserAllowed` | boolean? | 是否允许主机浏览器控制 |
| `elevated.allowed` | boolean? | 是否允许提升权限 |
| `elevated.defaultLevel` | "on" \| "off" \| "ask" \| "full"? | 默认提升级别 |

**相关文件**：
- `src/infra/sandbox-runner.ts` - 沙箱运行时实现
- `src/docker/` - Docker 相关

---

### 19. Reactions 反应部分

**源位置**：`src/agents/system-prompt.ts:524-546`

**代码**：
```typescript
if (params.reactionGuidance) {
  const { level, channel } = params.reactionGuidance;
  const guidanceText =
    level === "minimal"
      ? [
          `Reactions are enabled for ${channel} in MINIMAL mode.`,
          "React ONLY when truly relevant:",
          "- Acknowledge important user requests or confirmations",
          "- Express genuine sentiment (humor, appreciation) sparingly",
          "- Avoid reacting to routine messages or your own replies",
          "Guideline: at most 1 reaction per 5-10 exchanges.",
        ].join("\n")
      : [
          `Reactions are enabled for ${channel} in EXTENSIVE mode.`,
          "Feel free to react liberally:",
          "- Acknowledge messages with appropriate emojis",
          "- Express sentiment and personality through reactions",
          "- React to interesting content, humor, or notable events",
          "- Use reactions to confirm understanding or agreement",
          "Guideline: react whenever it feels natural.",
        ].join("\n");
  lines.push("## Reactions", guidanceText, "");
}
```

| 属性 | 值 |
|------|-----|
| 作用 | 指导 agent 何时使用表情反应 |
| 模式 | full / minimal |
| 条件 | `reactionGuidance` 已配置 |
| 参数 | `reactionGuidance?: { level: "minimal" \| "extensive"; channel: string }` |
| 来源 | `config.channels[channel].reactions` |

**相关文件**：
- `src/telegram/` - Telegram 反应实现（主要支持的频道）

---

### 20. Reasoning Format 推理格式

**源位置**：`src/agents/system-prompt.ts:547-548`

**代码**：
```typescript
if (reasoningHint) {
  lines.push("## Reasoning Format", reasoningHint, "");
}
```

**reasoningHint 构建** (行 321-332)：
```typescript
const reasoningHint = params.reasoningTagHint
  ? [
      "ALL internal reasoning MUST be inside <think>...</think>.",
      "Do not output any analysis outside <think>.",
      "Format every reply as <think>...</think> then <final>...</final>, with no other text.",
      "Only the final user-visible reply may appear inside <final>.",
      "Only text inside <final> is shown to the user; everything else is discarded and never seen by the user.",
      "Example:",
      "<think>Short internal reasoning.</think>",
      "<final>Hey there! What would you like to do next?</final>",
    ].join(" ")
  : undefined;
```

| 属性 | 值 |
|------|-----|
| 作用 | 指导 agent 使用思维标签进行推理 |
| 模式 | full / minimal |
| 条件 | `reasoningTagHint === true` |
| 参数 | `reasoningTagHint?: boolean` |
| 来源 | `config.agents.defaults.reasoning.tagHint` |

**相关文件**：
- `src/auto-reply/thinking.ts` - 推理相关

---

### 21. Project Context 项目上下文

**源位置**：`src/agents/system-prompt.ts:551-568`

**代码**：
```typescript
const contextFiles = params.contextFiles ?? [];
if (contextFiles.length > 0) {
  const hasSoulFile = contextFiles.some((file) => {
    const normalizedPath = file.path.trim().replace(/\\/g, "/");
    const baseName = normalizedPath.split("/").pop() ?? normalizedPath;
    return baseName.toLowerCase() === "soul.md";
  });
  lines.push("# Project Context", "", "The following project context files have been loaded:");
  if (hasSoulFile) {
    lines.push(
      "If SOUL.md is present, embody its persona and tone. Avoid stiff, generic replies; follow its guidance unless higher-priority instructions override it.",
    );
  }
  lines.push("");
  for (const file of contextFiles) {
    lines.push(`## ${file.path}`, "", file.content, "");
  }
}
```

| 属性 | 值 |
|------|-----|
| 作用 | 注入工作空间上下文文件 |
| 模式 | full / minimal |
| 条件 | 有 `contextFiles` |
| 参数 | `contextFiles?: EmbeddedContextFile[]` |

**EmbeddedContextFile 结构**：
```typescript
{
  path: string;      // 文件路径
  content: string;   // 文件内容
}
```

**包含的文件**（详见 [工作空间上下文文件](#工作空间上下文文件) 部分）

---

### 22. Silent Replies 无声回复

**源位置**：`src/agents/system-prompt.ts:571-586`

**代码**：
```typescript
if (!isMinimal) {
  lines.push(
    "## Silent Replies",
    `When you have nothing to say, respond with ONLY: ${SILENT_REPLY_TOKEN}`,
    "",
    "⚠️ Rules:",
    "- It must be your ENTIRE message — nothing else",
    `- Never append it to an actual response (never include "${SILENT_REPLY_TOKEN}" in real replies)`,
    "- Never wrap it in markdown or code blocks",
    "",
    `❌ Wrong: "Here's help... ${SILENT_REPLY_TOKEN}"`,
    `❌ Wrong: "${SILENT_REPLY_TOKEN}"`,
    `✅ Right: ${SILENT_REPLY_TOKEN}`,
    "",
  );
}
```

| 属性 | 值 |
|------|-----|
| 作用 | 教 agent 如何发送无声回复 |
| 模式 | full 仅 |
| 条件 | `!isMinimal` |
| 特殊标记 | `SILENT_REPLY_TOKEN`（来自 `src/auto-reply/tokens.js`） |

**相关文件**：
- `src/auto-reply/tokens.js` - 定义 SILENT_REPLY_TOKEN
- `src/auto-reply/reply.ts` - 无声回复处理

---

### 23. Heartbeats 心跳部分

**源位置**：`src/agents/system-prompt.ts:588-598`

**代码**：
```typescript
if (!isMinimal) {
  lines.push(
    "## Heartbeats",
    heartbeatPromptLine,
    "If you receive a heartbeat poll (a user message matching the heartbeat prompt above), and there is nothing that needs attention, reply exactly:",
    "HEARTBEAT_OK",
    'OpenClaw treats a leading/trailing "HEARTBEAT_OK" as a heartbeat ack (and may discard it).',
    'If something needs attention, do NOT include "HEARTBEAT_OK"; reply with the alert text instead.',
    "",
  );
}
```

其中 `heartbeatPromptLine` 定义在行 337-339：
```typescript
const heartbeatPromptLine = heartbeatPrompt
  ? `Heartbeat prompt: ${heartbeatPrompt}`
  : "Heartbeat prompt: (configured)";
```

| 属性 | 值 |
|------|-----|
| 作用 | 教 agent 如何处理定期心跳检查 |
| 模式 | full 仅 |
| 条件 | `!isMinimal` |
| 参数 | `heartbeatPrompt?: string` |
| 来源 | `config.agents.defaults.heartbeat.prompt` |

**相关文件**：
- `src/auto-reply/heartbeat.ts` - 心跳处理

---

### 24. Runtime 运行时部分

**源位置**：`src/agents/system-prompt.ts:601-605`

**代码**：
```typescript
lines.push(
  "## Runtime",
  buildRuntimeLine(runtimeInfo, runtimeChannel, runtimeCapabilities, params.defaultThinkLevel),
  `Reasoning: ${reasoningLevel} (hidden unless on/stream). Toggle /reasoning; /status shows Reasoning when enabled.`,
);
```

**buildRuntimeLine() 函数** (行 610-645)：
```typescript
export function buildRuntimeLine(
  runtimeInfo?: {
    agentId?: string;
    host?: string;
    os?: string;
    arch?: string;
    node?: string;
    model?: string;
    defaultModel?: string;
    repoRoot?: string;
  },
  runtimeChannel?: string,
  runtimeCapabilities: string[] = [],
  defaultThinkLevel?: ThinkLevel,
): string {
  return `Runtime: ${[
    runtimeInfo?.agentId ? `agent=${runtimeInfo.agentId}` : "",
    runtimeInfo?.host ? `host=${runtimeInfo.host}` : "",
    runtimeInfo?.repoRoot ? `repo=${runtimeInfo.repoRoot}` : "",
    runtimeInfo?.os
      ? `os=${runtimeInfo.os}${runtimeInfo?.arch ? ` (${runtimeInfo.arch})` : ""}`
      : runtimeInfo?.arch
        ? `arch=${runtimeInfo.arch}`
        : "",
    runtimeInfo?.node ? `node=${runtimeInfo.node}` : "",
    runtimeInfo?.model ? `model=${runtimeInfo.model}` : "",
    runtimeInfo?.defaultModel ? `default_model=${runtimeInfo.defaultModel}` : "",
    runtimeChannel ? `channel=${runtimeChannel}` : "",
    runtimeChannel
      ? `capabilities=${runtimeCapabilities.length > 0 ? runtimeCapabilities.join(",") : "none"}`
      : "",
    `thinking=${defaultThinkLevel ?? "off"}`,
  ]
    .filter(Boolean)
    .join(" | ")}`;
}
```

| 属性 | 值 |
|------|-----|
| 作用 | 提供系统运行时信息 |
| 模式 | full / minimal |
| 条件 | 始终 |
| 参数 | 详见下表 |

**runtimeInfo 结构**：

| 参数 | 类型 | 作用 |
|------|------|------|
| `agentId` | string? | Agent ID |
| `host` | string | 主机名 |
| `os` | string | 操作系统 |
| `arch` | string | CPU 架构 |
| `node` | string | Node.js 版本 |
| `model` | string | 当前模型 |
| `defaultModel` | string? | 默认模型 |
| `channel` | string? | 运行频道 |
| `capabilities` | string[]? | 频道能力列表 |
| `repoRoot` | string? | 仓库根目录 |

**相关文件**：
- `src/agents/system-prompt-params.ts` - 构建运行时参数

---

## 工作空间上下文文件

这些是用户可编辑的文件，位于 `~/.openclaw/workspace/` 中。

### 文件加载流程

**源位置**：

1. **加载文件** - `src/agents/workspace.ts:237-291`
   - 函数：`loadWorkspaceBootstrapFiles(dir: string)`
   - 从工作空间目录读取文件

2. **转换为上下文** - `src/agents/bootstrap-files.ts`
   - 函数：`buildBootstrapContextFiles()`
   - 检查文件大小、截断超大文件

3. **过滤会话类型** - `src/agents/workspace.ts:295-303`
   - 函数：`filterBootstrapFilesForSession(files, sessionKey)`
   - 子 agent 仅获得特定文件

### 文件列表

#### 1. SOUL.md - Agent 灵魂与个性

**路径**：`~/.openclaw/workspace/SOUL.md`

**默认模板**：`docs/reference/templates/SOUL.md`

| 属性 | 值 |
|------|-----|
| 作用 | 定义 agent 个性、价值观、行为准则 |
| 何时注入 | 每次请求（主 agent） |
| 子 agent 可见 | ❌ 否 |
| 最大字符数 | 无限 |
| 是否强制 | ❌ 否 |
| 特殊处理 | 检测 SOUL.md 并添加特殊指导 |

**模板内容要点**：
- Core Truths（核心真理）
- Boundaries（边界）
- Vibe（风格）
- Continuity（连续性）

**源代码位置**：
- 模板：`docs/reference/templates/SOUL.md`
- 加载检测：`src/agents/system-prompt.ts:553-562`

---

#### 2. BOOTSTRAP.md - 初始化脚本

**路径**：`~/.openclaw/workspace/BOOTSTRAP.md`

**默认模板**：`docs/reference/templates/BOOTSTRAP.md`

| 属性 | 值 |
|------|-----|
| 作用 | 新工作空间首次运行时的初始化指导 |
| 何时注入 | 每次请求（主 agent） |
| 子 agent 可见 | ❌ 否 |
| 最大字符数 | 20 KB（`DEFAULT_BOOTSTRAP_MAX_CHARS = 20_000`） |
| 是否强制 | ⚠️ 仅新工作空间 |
| 截断策略 | 70% 头 + 20% 尾 |

**模板内容要点**：
- The Conversation（对话建议）
- After You Know Who You Are（后续步骤）
- Connect（可选连接）

**源代码位置**：
- 模板：`docs/reference/templates/BOOTSTRAP.md`
- 最大字符常量：`src/agents/pi-embedded-helpers/bootstrap.ts:61`
- 大小限制逻辑：`src/agents/pi-embedded-helpers/bootstrap.ts:80-120`

---

#### 3. MEMORY.md - 主要记忆文件

**路径**：`~/.openclaw/workspace/MEMORY.md`

**默认模板**：`docs/reference/templates/MEMORY.md` (实际为空)

| 属性 | 值 |
|------|-----|
| 作用 | 持久化记忆和重要信息 |
| 何时注入 | 每次请求（主 agent） |
| 子 agent 可见 | ❌ 否 |
| 最大字符数 | 无限 |
| 是否强制 | ❌ 否 |
| 特殊处理 | 通过 `memory_search` 工具搜索 |

**使用方式**：
- 通过 `memory_search` 和 `memory_get` 工具访问
- Agent 可以通过 `memory_write` 更新

---

#### 4. memory/*.md - 分类记忆文件

**路径**：`~/.openclaw/workspace/memory/*.md`

**模式**：任何 `memory/` 子目录中的 .md 文件

| 属性 | 值 |
|------|-----|
| 作用 | 组织化的记忆（如 memory/people.md、memory/projects.md） |
| 何时注入 | 每次请求（主 agent） |
| 子 agent 可见 | ❌ 否 |
| 最大字符数 | 无限 |
| 是否强制 | ❌ 否 |
| 检测方式 | `resolveMemoryBootstrapEntries()` 检查重复 |

**源代码位置**：
- 检测逻辑：`src/agents/workspace.ts:200-235`

---

#### 5. IDENTITY.md - Agent 身份

**路径**：`~/.openclaw/workspace/IDENTITY.md`

**默认模板**：`docs/reference/templates/IDENTITY.md`

| 属性 | 值 |
|------|-----|
| 作用 | 定义 agent 名称、描述、表情符号 |
| 何时注入 | 每次请求（主 agent） |
| 子 agent 可见 | ❌ 否 |
| 最大字符数 | 无限 |
| 是否强制 | ⚠️ 可选 |
| 特殊处理 | 无 |

---

#### 6. USER.md - 用户信息

**路径**：`~/.openclaw/workspace/USER.md`

**默认模板**：`docs/reference/templates/USER.md`

| 属性 | 值 |
|------|-----|
| 作用 | 用户名、偏好、时区、特殊说明 |
| 何时注入 | 每次请求（主 agent） |
| 子 agent 可见 | ❌ 否 |
| 最大字符数 | 无限 |
| 是否强制 | ⚠️ 可选 |
| 特殊处理 | 无 |

---

#### 7. AGENTS.md - 子 Agent 列表

**路径**：`~/.openclaw/workspace/AGENTS.md`

**默认模板**：`docs/reference/templates/AGENTS.md`

| 属性 | 值 |
|------|-----|
| 作用 | 列出可用的子 agent 及其用途 |
| 何时注入 | ✅ 每次请求（包括子 agent） |
| 子 agent 可见 | ✅ 是 |
| 最大字符数 | 无限 |
| 是否强制 | ❌ 否 |
| 特殊处理 | 在 `SUBAGENT_BOOTSTRAP_ALLOWLIST` 中 |

**源代码位置**：
- 允许列表：`src/agents/workspace.ts:293`

---

#### 8. TOOLS.md - 工具使用指南

**路径**：`~/.openclaw/workspace/TOOLS.md`

**默认模板**：`docs/reference/templates/TOOLS.md`

| 属性 | 值 |
|------|-----|
| 作用 | 对外部工具的使用指导（不控制可用性） |
| 何时注入 | ✅ 每次请求（包括子 agent） |
| 子 agent 可见 | ✅ 是 |
| 最大字符数 | 无限 |
| 是否强制 | ❌ 否 |
| 特殊处理 | 在 `SUBAGENT_BOOTSTRAP_ALLOWLIST` 中 |

**源代码位置**：
- 允许列表：`src/agents/workspace.ts:293`
- 说明：`src/agents/system-prompt.ts:404`

---

#### 9. HEARTBEAT.md - 定期任务

**路径**：`~/.openclaw/workspace/HEARTBEAT.md`

**默认模板**：`docs/reference/templates/HEARTBEAT.md`

| 属性 | 值 |
|------|-----|
| 作用 | 定期任务和检查项（通过心跳触发） |
| 何时注入 | 每次请求（主 agent） |
| 子 agent 可见 | ❌ 否 |
| 最大字符数 | 无限 |
| 是否强制 | ❌ 否 |
| 特殊处理 | 空文件会跳过心跳 API 调用 |

**源代码位置**：
- 空检测：`src/auto-reply/heartbeat.ts:22-52`

---

### 文件加载流程详解

**源位置**：`src/agents/workspace.ts:237-303`

```typescript
// 1. 加载文件
export async function loadWorkspaceBootstrapFiles(dir: string): Promise<WorkspaceBootstrapFile[]>

// 2. 转换为上下文
// 来自：src/agents/bootstrap-files.ts
export function buildBootstrapContextFiles(files: WorkspaceBootstrapFile[]): EmbeddedContextFile[]

// 3. 过滤会话类型
export function filterBootstrapFilesForSession(
  files: WorkspaceBootstrapFile[],
  sessionKey?: string,
): WorkspaceBootstrapFile[]
```

**完整调用链**：
```
LLM 请求
  ↓
pi-embedded-runner/run/attempt.ts (行 373)
  resolveBootstrapContextForRun()
    ↓
  bootstrap-files.ts: buildBootstrapContextFiles()
    ↓
  workspace.ts: loadWorkspaceBootstrapFiles()
    ↓
  读取: SOUL.md, BOOTSTRAP.md, MEMORY.md, memory/*,
       AGENTS.md, TOOLS.md, HEARTBEAT.md, IDENTITY.md, USER.md
    ↓
  filterBootstrapFilesForSession() [子 agent 过滤]
    ↓
  传递给 buildAgentSystemPrompt()
    ↓
  在 "Project Context" 部分中渲染
```

**源代码位置**：
- `src/agents/pi-embedded-runner/run/attempt.ts:373`
- `src/agents/bootstrap-files.ts`
- `src/agents/workspace.ts`

---

## 特殊提示词

### 1. Heartbeat 提示词

**源文件**：`src/auto-reply/heartbeat.ts:1-6`

**默认值**：
```typescript
export const HEARTBEAT_PROMPT =
  "Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.";
```

| 属性 | 值 |
|------|-----|
| 作用 | 告诉 agent 如何处理周期性心跳检查 |
| 何时使用 | 每 30 分钟执行一次（`DEFAULT_HEARTBEAT_EVERY = "30m"`） |
| 模式 | 始终在系统提示词中（在 "Heartbeats" 部分） |
| 可配置 | ✅ 是 - `config.agents.defaults.heartbeat.prompt` |
| 来源 | `src/auto-reply/heartbeat.ts:54-57` 中的 `resolveHeartbeatPrompt()` |

**配置位置**：
- 配置文件：`~/.openclaw/openclaw.json` 或环境变量
- 配置键：`agents.defaults.heartbeat.prompt`

**相关文件**：
- `src/auto-reply/heartbeat.ts` - 心跳处理和提示词管理
- `src/infra/heartbeat-runner.ts` - 心跳定时器

---

### 2. 额外系统提示词 (Extra System Prompt)

**源位置**：`src/agents/system-prompt.ts:518-523`

**代码**：
```typescript
if (extraSystemPrompt) {
  const contextHeader =
    promptMode === "minimal" ? "## Subagent Context" : "## Group Chat Context";
  lines.push(contextHeader, extraSystemPrompt, "");
}
```

| 属性 | 值 |
|------|-----|
| 作用 | 为特定上下文注入自定义提示词 |
| 模式 | full / minimal |
| 条件 | 当提供了 `extraSystemPrompt` 参数 |
| 参数 | `extraSystemPrompt?: string` |
| 来源 | 由调用者在构建系统提示词时提供 |

**使用场景**：
- 群聊上下文
- 特定频道上下文
- 自定义工作流

**调用位置**：
- `src/agents/pi-embedded-runner/run/attempt.ts` - 运行时提示词构建

---

### 3. 技能提示词 (Skills Prompt)

**源位置**：`src/agents/system-prompt.ts:16-37`

**构建位置**：`src/agents/skills.ts`

| 属性 | 值 |
|------|-----|
| 作用 | 教 agent 如何选择和使用技能 |
| 模式 | full 仅 |
| 参数 | `skillsPrompt?: string` |
| 来源 | 由 `resolveSkillsPromptForRun()` 构建 |

**工作空间集成**：
- 文件名：`SKILL.md`（位置可配置）
- 用户工作空间：`~/.openclaw/workspace/`
- 扩展工作空间：`extensions/*/` (如果存在)

**源代码位置**：
- `src/agents/skills.ts` - 技能管理和提示词构建

---

## 提示词参数结构

### buildAgentSystemPrompt() 完整签名

**源位置**：`src/agents/system-prompt.ts:164-217`

```typescript
export function buildAgentSystemPrompt(params: {
  workspaceDir: string;
  defaultThinkLevel?: ThinkLevel;
  reasoningLevel?: ReasoningLevel;
  extraSystemPrompt?: string;
  ownerNumbers?: string[];
  reasoningTagHint?: boolean;
  toolNames?: string[];
  toolSummaries?: Record<string, string>;
  modelAliasLines?: string[];
  userTimezone?: string;
  userTime?: string;
  userTimeFormat?: ResolvedTimeFormat;
  contextFiles?: EmbeddedContextFile[];
  skillsPrompt?: string;
  heartbeatPrompt?: string;
  docsPath?: string;
  workspaceNotes?: string[];
  ttsHint?: string;
  promptMode?: PromptMode;
  runtimeInfo?: {...};
  messageToolHints?: string[];
  sandboxInfo?: {...};
  reactionGuidance?: {...};
  memoryCitationsMode?: MemoryCitationsMode;
}): string
```

### 参数详细说明

| 参数 | 类型 | 必需 | 作用 | 来源 | 配置键 |
|------|------|------|------|------|--------|
| `workspaceDir` | string | ✅ | 工作目录 | runtime | agents.workspaceDir |
| `defaultThinkLevel` | ThinkLevel? | ❌ | 默认思考级别 | config | agents.defaults.thinking |
| `reasoningLevel` | ReasoningLevel? | ❌ | 推理模式 (off/on/stream) | config | agents.defaults.reasoning.level |
| `extraSystemPrompt` | string? | ❌ | 自定义系统提示 | runtime | 无 |
| `ownerNumbers` | string[]? | ❌ | 所有者电话号码 | config | agents.defaults.ownerNumbers |
| `reasoningTagHint` | boolean? | ❌ | 使用 `<think>` 标签 | config | agents.defaults.reasoning.tagHint |
| `toolNames` | string[]? | ❌ | 可用工具列表 | runtime | 无 |
| `toolSummaries` | Record<string, string>? | ❌ | 工具描述 | config/plugins | 无 |
| `modelAliasLines` | string[]? | ❌ | 模型别名 | config | agents.defaults.modelAliases |
| `userTimezone` | string? | ❌ | 用户时区 | config | agents.defaults.userTimezone |
| `userTime` | string? | ❌ | 当前用户时间 | runtime | 无 |
| `userTimeFormat` | ResolvedTimeFormat? | ❌ | 时间格式 | config | agents.defaults.timeFormat |
| `contextFiles` | EmbeddedContextFile[]? | ❌ | 工作空间文件 | runtime | 无 |
| `skillsPrompt` | string? | ❌ | 技能指导 | runtime | 无 |
| `heartbeatPrompt` | string? | ❌ | 心跳提示 | config | agents.defaults.heartbeat.prompt |
| `docsPath` | string? | ❌ | 文档路径 | config | agents.defaults.docsPath |
| `workspaceNotes` | string[]? | ❌ | 工作空间备注 | config | 无 |
| `ttsHint` | string? | ❌ | TTS 指导 | config | 无 |
| `promptMode` | PromptMode? | ❌ | 提示词模式 | runtime | 无 |
| `runtimeInfo` | Object? | ❌ | 运行时信息 | runtime | 无 |
| `messageToolHints` | string[]? | ❌ | 消息工具提示 | config | 无 |
| `sandboxInfo` | Object? | ❌ | 沙箱配置 | runtime | 无 |
| `reactionGuidance` | Object? | ❌ | 反应指导 | config | channels.[name].reactions |
| `memoryCitationsMode` | MemoryCitationsMode? | ❌ | 记忆引用模式 | config | agents.defaults.memoryCitationsMode |

### buildSystemPromptParams() 构建器

**源位置**：`src/agents/system-prompt-params.ts:33-58`

```typescript
export function buildSystemPromptParams(params: {
  config?: OpenClawConfig;
  agentId?: string;
  runtime: Omit<RuntimeInfoInput, "agentId">;
  workspaceDir?: string;
  cwd?: string;
}): SystemPromptRuntimeParams
```

**作用**：从配置和运行时信息构建系统提示词参数

**返回值**：
```typescript
{
  runtimeInfo: RuntimeInfoInput;
  userTimezone: string;
  userTime?: string;
  userTimeFormat?: ResolvedTimeFormat;
}
```

---

## 提示词执行流程

### 完整调用链

**源位置**：见各阶段位置

```
用户消息到达
  ↓ src/channels/*/receive.ts
客户端调用 agent 运行时
  ↓ src/agents/pi-embedded-runner/run/attempt.ts: runReplyAgent()
解析会话信息
  ↓ src/agents/pi-embedded-runner/run/attempt.ts:300-330
加载系统提示词参数
  ↓ src/agents/system-prompt-params.ts:buildSystemPromptParams()
构建运行时信息
  ├─ agentId、host、os、arch
  ├─ node 版本、model、channel
  └─ capabilities
  ↓ src/agents/pi-embedded-runner/run/attempt.ts:340-375
加载工作空间文件
  ↓ src/agents/bootstrap-files.ts:resolveBootstrapContextForRun()
  ├─ loadWorkspaceBootstrapFiles()
  ├─ buildBootstrapContextFiles()
  └─ filterBootstrapFilesForSession() [子 agent 过滤]
  ↓ src/agents/pi-embedded-runner/run/attempt.ts:373
加载工具列表
  ├─ src/agents/pi-embedded-runner/run/attempt.ts:352
  └─ 工具名称和描述
加载高级配置
  ├─ reasoningLevel、defaultThinkLevel
  ├─ skillsPrompt、heartbeatPrompt
  ├─ extraSystemPrompt
  └─ 沙箱配置、反应指导等
  ↓ src/agents/system-prompt.ts:buildAgentSystemPrompt()
构建系统提示词
  ├─ 基础身份 (1)
  ├─ Tooling (2)
  ├─ Tool Call Style (3)
  ├─ Safety (4)
  ├─ OpenClaw CLI (5)
  ├─ Self-Update (6)
  ├─ Model Aliases (7)
  ├─ Time Hint (8)
  ├─ Workspace (9)
  ├─ Documentation (10)
  ├─ Skills (11)
  ├─ Memory (12)
  ├─ User Identity (13)
  ├─ Current Date & Time (14)
  ├─ Reply Tags (15)
  ├─ Messaging (16)
  ├─ Voice (17)
  ├─ Sandbox (18)
  ├─ Reactions (19)
  ├─ Reasoning Format (20)
  ├─ Project Context (21)
  │   ├─ SOUL.md
  │   ├─ BOOTSTRAP.md
  │   ├─ MEMORY.md
  │   ├─ AGENTS.md
  │   └─ TOOLS.md
  ├─ Silent Replies (22)
  ├─ Heartbeats (23)
  └─ Runtime (24)
  ↓
返回完整系统提示词
  ↓
结合消息历史
  ↓
发送给 LLM
  ↓
LLM 响应
```

### 关键源文件清单

| 阶段 | 文件 | 关键函数 | 行号 |
|------|------|---------|------|
| 消息接收 | `src/channels/*/receive.ts` | 各频道接收函数 | - |
| Agent 运行 | `src/agents/pi-embedded-runner/run/attempt.ts` | `runReplyAgent()` | 行 185-400+ |
| 参数构建 | `src/agents/system-prompt-params.ts` | `buildSystemPromptParams()` | 行 33-58 |
| 文件加载 | `src/agents/bootstrap-files.ts` | `resolveBootstrapContextForRun()` | - |
| 工作空间 | `src/agents/workspace.ts` | `loadWorkspaceBootstrapFiles()` | 行 237-291 |
| 工作空间过滤 | `src/agents/workspace.ts` | `filterBootstrapFilesForSession()` | 行 295-303 |
| 提示词构建 | `src/agents/system-prompt.ts` | `buildAgentSystemPrompt()` | 行 164-608 |
| 运行时信息 | `src/agents/system-prompt.ts` | `buildRuntimeLine()` | 行 610-645 |

---

## 特殊标记

### SILENT_REPLY_TOKEN

**源文件**：`src/auto-reply/tokens.js`

**符号**：`⏸️`（实际值从 tokens.js 读取）

**作用**：无声回复标记

**使用位置**：
- 系统提示词：`src/agents/system-prompt.ts:574`
- 回复处理：`src/auto-reply/reply.ts`
- 验证：`src/auto-reply/tokens.ts`

**规则**：
- 必须是回复的**全部内容**
- 不能追加到其他内容后
- 不能包装在 markdown 或代码块中

---

### HEARTBEAT_TOKEN

**源文件**：`src/auto-reply/tokens.js`

**符号**：`HEARTBEAT_OK`

**作用**：心跳确认标记

**使用位置**：
- 系统提示词：`src/agents/system-prompt.ts:594`
- 心跳处理：`src/auto-reply/heartbeat.ts`
- 心跳验证：`src/auto-reply/heartbeat.ts:96-157`

**规则**：
- 标记会被从回复边缘移除
- 支持 HTML/Markdown 包装（如 `<b>HEARTBEAT_OK</b>`）
- 最大确认字符数：`DEFAULT_HEARTBEAT_ACK_MAX_CHARS = 300`

---

## 工作空间文件位置

### 默认工作空间路径

**解析函数**：`src/agents/workspace.ts:9-18`

```typescript
export function resolveDefaultAgentWorkspaceDir(
  env: NodeJS.ProcessEnv = process.env,
  homedir: () => string = os.homedir,
): string {
  const profile = env.OPENCLAW_PROFILE?.trim();
  if (profile && profile.toLowerCase() !== "default") {
    return path.join(homedir(), ".openclaw", `workspace-${profile}`);
  }
  return path.join(homedir(), ".openclaw", "workspace");
}
```

| 场景 | 路径 |
|------|------|
| 默认 | `~/.openclaw/workspace/` |
| 自定义 Profile | `~/.openclaw/workspace-${OPENCLAW_PROFILE}/` |
| 环境变量 | `$OPENCLAW_PROFILE` |

### 模板文件路径

**位置**：`docs/reference/templates/`

| 文件 | 路径 | 默认内容位置 |
|------|------|-----------|
| SOUL.md | `docs/reference/templates/SOUL.md` | 行 1-43 |
| BOOTSTRAP.md | `docs/reference/templates/BOOTSTRAP.md` | 行 1-62 |
| HEARTBEAT.md | `docs/reference/templates/HEARTBEAT.md` | 行 1-12 |
| AGENTS.md | `docs/reference/templates/AGENTS.md` | - |
| AGENTS.dev.md | `docs/reference/templates/AGENTS.dev.md` | - |
| TOOLS.md | `docs/reference/templates/TOOLS.md` | - |
| TOOLS.dev.md | `docs/reference/templates/TOOLS.dev.md` | - |
| IDENTITY.md | `docs/reference/templates/IDENTITY.md` | - |
| IDENTITY.dev.md | `docs/reference/templates/IDENTITY.dev.md` | - |
| USER.md | `docs/reference/templates/USER.md` | - |
| USER.dev.md | `docs/reference/templates/USER.dev.md` | - |

### 文件常量定义

**源位置**：`src/agents/workspace.ts:20-29`

```typescript
export const DEFAULT_AGENT_WORKSPACE_DIR = resolveDefaultAgentWorkspaceDir();
export const DEFAULT_AGENTS_FILENAME = "AGENTS.md";
export const DEFAULT_SOUL_FILENAME = "SOUL.md";
export const DEFAULT_TOOLS_FILENAME = "TOOLS.md";
export const DEFAULT_IDENTITY_FILENAME = "IDENTITY.md";
export const DEFAULT_USER_FILENAME = "USER.md";
export const DEFAULT_HEARTBEAT_FILENAME = "HEARTBEAT.md";
export const DEFAULT_BOOTSTRAP_FILENAME = "BOOTSTRAP.md";
export const DEFAULT_MEMORY_FILENAME = "MEMORY.md";
export const DEFAULT_MEMORY_ALT_FILENAME = "memory.md";
```

---

## 配置相关

### 配置文件位置

**配置文件**：`~/.openclaw/openclaw.json` 或环境变量

**类型定义**：`src/config/config.ts`

### 系统提示词相关配置

| 配置键 | 类型 | 说明 | 来源 |
|--------|------|------|------|
| `agents.defaults.userTimezone` | string | 用户时区 | `src/agents/system-prompt-params.ts:45` |
| `agents.defaults.timeFormat` | ResolvedTimeFormat | 时间格式 | `src/agents/system-prompt-params.ts:46` |
| `agents.defaults.ownerNumbers` | string[] | 所有者电话号码 | `src/agents/system-prompt.ts:316` |
| `agents.defaults.thinking` | ThinkLevel | 默认思考级别 | `src/agents/system-prompt.ts:166` |
| `agents.defaults.reasoning.level` | ReasoningLevel | 推理模式 | `src/agents/system-prompt.ts:167` |
| `agents.defaults.reasoning.tagHint` | boolean | 使用思维标签 | `src/agents/system-prompt.ts:170` |
| `agents.defaults.heartbeat.prompt` | string | 心跳提示词 | `src/auto-reply/heartbeat.ts:54` |
| `agents.defaults.heartbeat.every` | string | 心跳周期 | `src/auto-reply/heartbeat.ts:7` |
| `agents.defaults.modelAliases` | Record<string, string> | 模型别名 | `src/agents/system-prompt.ts:438` |
| `agents.defaults.docsPath` | string | 文档路径 | `src/agents/system-prompt.ts:368` |
| `agents.defaults.memoryCitationsMode` | "on" \| "off" | 记忆引用模式 | `src/agents/system-prompt.ts:43` |
| `agents.defaults.repoRoot` | string | 仓库根目录 | `src/agents/system-prompt-params.ts:65` |
| `channels.[name].reactions` | Object | 反应配置 | `src/agents/system-prompt.ts:524` |

### 环境变量

| 环境变量 | 说明 | 源位置 |
|---------|------|--------|
| `OPENCLAW_PROFILE` | 工作空间 Profile | `src/agents/workspace.ts:13-16` |
| `HOME` / `USERPROFILE` | 家目录 | `src/agents/workspace.ts:11` |

---

## 附录：提示词调试

### 查看完整系统提示词

**方法 1**：使用 CLI 状态命令
```bash
openclaw status --deep
```

**方法 2**：查看 session 日志
```bash
~/.openclaw/agents/<agentId>/sessions/*.jsonl
```

### 了解当前配置

**配置读取**：
```bash
openclaw config get agents.defaults
```

**配置修改**：
```bash
openclaw config set agents.defaults.userTimezone "America/New_York"
```

### 提示词报告生成

**源文件**：`src/agents/system-prompt-report.ts`

**用途**：生成系统提示词的详细报告

---

## 总结表

### 按类型分类的所有提示词

| 类型 | 数量 | 来源 | 模式 |
|------|------|------|------|
| 硬编码系统提示词部分 | 24 | system-prompt.ts | full/minimal/none |
| 工作空间上下文文件 | 9 | workspace.ts | varies |
| 特殊提示词 | 3 | 各文件 | varies |
| 配置参数 | 12+ | config.ts | 动态 |

### 信息流

```
用户配置 (openclaw.json)
  ↓
系统提示词参数构建
  ↓
工作空间文件加载
  ↓
提示词组装
  ↓
LLM 系统提示词
```

---

**文档版本**：1.0
**最后更新**：2025-02-07
**涵盖代码版本**：openclaw main branch
