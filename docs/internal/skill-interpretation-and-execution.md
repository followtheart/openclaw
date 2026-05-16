---
summary: "Internal analysis: how OpenClaw parses, surfaces, and executes skills end-to-end"
title: "Skill interpretation and execution (internals)"
audience: "implementers / contributors"
status: "internal — excluded from docs publish"
---

> 内部实现文档。来源是对 `src/agents/skills/**`、`src/auto-reply/skill-commands*`、`src/plugin-sdk/skill-commands-runtime.ts`、`src/agents/openclaw-tools.ts`、`src/agents/tools/**`、`extensions/**/openclaw.plugin.json`、`skills/**/SKILL.md` 的逐文件分析。所有 `path:line` 引用都对仓库根。本文不进入 Mintlify 公网发布。

## 0. 一句话总览

OpenClaw 的 skill 系统由 **目录契约 + frontmatter 路由 + 按需正文加载 + 三态命令派发** 组成。核心运行时把 `SKILL.md` 编成最小提示注入 LLM，而 slash 命令通过同一份 `SkillCommandSpec` 派发到三条路径之一：tool dispatch（绕开 LLM）、prompt template（确定性 prompt）、裸 skill（LLM 自取正文）。

---

## 1. 磁盘契约

### 1.1 目录结构

每个 skill 是一个目录，必须含 `SKILL.md`：

```
skills/<skill-name>/
├── SKILL.md          # YAML frontmatter + Markdown 正文（必需）
├── scripts/          # 可选脚本资源
├── references/       # 可选参考材料
└── assets/           # 可选媒体资源
```

仓库内示例：`skills/apple-notes/`（无脚本）、`skills/slack/`（JSON API 风格）、`skills/skill-creator/`（带 `scripts/init_skill.py` 等）。

### 1.2 SKILL.md frontmatter 字段

解析器：`src/agents/skills/frontmatter.ts:24` `parseFrontmatter` → `parseFrontmatterBlock`。

核心字段：

| 字段 | 类型 | 用途 |
|---|---|---|
| `name` | string | skill 标识符 |
| `description` | string | LLM 用于判断何时调用，会注入 prompt |
| `metadata.openclaw.emoji` | string | 终端/UI 标识 |
| `metadata.openclaw.os` | string[] | 平台白名单 (`darwin` / `linux` / `win32`) |
| `metadata.openclaw.requires.bins` | string[] | 必需二进制（缺则 skill 不暴露） |
| `metadata.openclaw.requires.env` | string[] | 必需环境变量 |
| `metadata.openclaw.requires.config` | string[] | 必需配置路径，如 `channels.slack` |
| `metadata.openclaw.install[]` | object[] | 安装规范（brew / node / go / uv / download） |
| `user-invocable` | bool | 暴露为 slash 命令 |
| `disable-model-invocation` | bool | 禁止 LLM 自动选用 |
| `command-dispatch` | `"tool"` | 派发模式（见 §6 模式 A） |
| `command-tool` | string | dispatch 时的目标工具名 |
| `command-arg-mode` | `"raw"` | 当前仅支持 raw |

frontmatter 内的 install/metadata 都做了防御性校验（路径遍历、特殊字符），见 `frontmatter.ts:112-185`。

### 1.3 正文

仅在 skill 被选中后由 LLM 通过 `read` 工具显式加载——**默认不进入系统提示**。这是控制 token 成本的核心设计。

---

## 2. 发现与加载

### 2.1 入口

公共 API：`src/agents/skills.ts`

- `loadWorkspaceSkillEntries(workspaceDir, opts?)`
- `buildWorkspaceSkillSnapshot(workspaceDir, opts?)`
- `buildWorkspaceSkillsPrompt(workspaceDir, opts?)`

内部实现：`src/agents/skills/workspace.ts:606` `loadSkillEntries`。

### 2.2 六类来源（按优先级合并，后者覆盖前者）

| 优先级 | 来源 ID | 路径 |
|---|---|---|
| 1（最低） | `openclaw-bundled` | 仓库 `/skills/`（`bundled-dir.ts:resolveBundledSkillsDir`） |
| 2 | `openclaw-extra` | `config.skills.load.extraDirs` + 插件贡献目录 |
| 3 | `openclaw-managed` | `~/.openclaw/skills/` |
| 4 | `agents-skills-personal` | `~/.agents/skills/` |
| 5 | `agents-skills-project` | `<workspace>/.agents/skills/` |
| 6（最高） | `openclaw-workspace` | `<workspace>/skills/` |

合并在 `workspace.ts:900-919` 的 Map 上发生，同名 skill 后者覆盖前者。

### 2.3 扫描算法

两层递归（`workspace.ts:737-832`）：

1. 直接层：`<root>/<name>/SKILL.md`
2. 分组层：`<root>/<group>/<name>/SKILL.md`（前者不存在时尝试）

防护限制：

- `maxCandidatesPerRoot = 300`
- `maxSkillsLoadedPerSource = 200`
- `maxSkillFileBytes = 256KB`
- `realpath` 强制阻止 symlink 越界（`workspace.ts:313-338`）

### 2.4 插件贡献

`src/agents/skills/plugin-skills.ts:22-121` `resolvePluginSkillDirs` 读插件 `package.json` 的 `skills` 字段（相对路径数组），校验 symlink 不逃逸后收集 `<plugin-root>/<skill-path>` 加入第 2 类来源。

`plugin-skills.ts:201-268` `publishPluginSkills` 进一步把这些目录以 symlink/junction 形式发布到 `~/.openclaw/plugin-skills/`，并自动清理过期链接。

---

## 3. Frontmatter / 工具 / 脚本的提取

### 3.1 SkillEntry 结构

`src/agents/skills/types.ts`：

```ts
type SkillEntry = {
  skill: Skill;
  frontmatter: Record<string, string>;
  metadata?: OpenClawSkillMetadata;
  invocation?: SkillInvocationPolicy;
  exposure?: SkillExposure;
  syncSourceDir?: string;
  syncDirName?: string;
};
```

### 3.2 关键提取函数

- `parseFrontmatter(content)` —— 解析 YAML 块
- `resolveOpenClawMetadata(frontmatter)` —— 规范化 `metadata.openclaw.*`
- `resolveSkillInvocationPolicy(frontmatter)` —— 提取 `user-invocable` / `disable-model-invocation`
- `resolveSkillExposure(...)` —— 计算模型/用户可见性

### 3.3 脚本不自动注入

`tools-dir.ts` 把每个 skill 的脚本资源根目录映射到 `~/.openclaw/tools/<safe-skill-key>/`，但**核心运行时不会自动扫描 `scripts/` 注入 LLM 上下文**。脚本的执行由：

- LLM 通过 `Bash` 工具按 SKILL.md 正文指引调用（模式 C），或
- 通过 `command-dispatch: tool` 由专门的工具触发（模式 A）

---

## 4. 注入 LLM 提示

### 4.1 生成流程

`buildWorkspaceSkillsPrompt`（`workspace.ts:1061-1066`）→ `resolveWorkspaceSkillPromptState`：

1. `loadSkillEntries` 加载
2. requirements 过滤（bins/env/config 缺失即剔除）
3. `isSkillVisibleInAvailableSkillsPrompt`（`workspace.ts:105-113`）
4. `compactSkillPaths`（`workspace.ts:69-76`）路径缩写为 `~/...`
5. `applySkillsPromptLimits`（`workspace.ts:990-1040`）按预算二分搜索
6. `formatSkillsForPrompt`（`skill-contract.ts:44-64`）输出 XML

### 4.2 完整格式

```xml
The following skills provide specialized instructions for specific tasks.
Use the read tool to load a skill's file when the task matches its description.
When a skill file references a relative path, resolve it against the skill directory ...

<available_skills>
  <skill>
    <name>apple-notes</name>
    <description>Create, view, edit, delete, search, move, or export Apple Notes via the memo CLI on macOS.</description>
    <location>~/openclaw/skills/apple-notes/SKILL.md</location>
  </skill>
  ...
</available_skills>
```

### 4.3 紧凑降级

预算（默认 `maxSkillsInPrompt=150`、`maxSkillsPromptChars=18000`，可被 `config.skills.limits.*` 或 agent 级配置覆盖）超出时，第二阶段省略 `<description>`，仅保留 `<name>` 和 `<location>`，以保留尽可能多的 skill 数量。

### 4.4 关键设计

**仅注入 name + description + location，正文绝不预加载**。LLM 决定相关时再用 `read` 工具显式拉取完整 SKILL.md——这把"知识检索"从启动成本转为按需成本。

---

## 5. 派发链路

### 5.1 SDK Facade

`src/plugin-sdk/skill-commands-runtime.ts` 全文 5 行：

```ts
export {
  listSkillCommandsForAgents,
  listSkillCommandsForWorkspace,
} from "../auto-reply/skill-commands.js";
```

存在意义：**lazy import**。命名为 `*-runtime.ts` 暗示给 `createLazyImportLoader` 用，让 hot path（无斜杠消息）不为 skill 扫描付加载代价。

调用点示例 `src/auto-reply/reply/get-reply-directives.ts:53`：

```ts
const skillCommandsLoader = createLazyImportLoader(
  () => import("../skill-commands.runtime.js"),
);
```

只有当 `commandTextHasSlash && rawAliases.length > 0` 时才会 await。

### 5.2 命令规格构造

`src/auto-reply/skill-commands.ts:22-42` `listSkillCommandsForWorkspace`：

```ts
return buildWorkspaceSkillCommandSpecs(workspaceDir, {
  config: cfg,
  agentId,
  skillFilter,
  eligibility: {
    remote: getRemoteSkillEligibility({
      advertiseExecNode: canExecRequestNode({ cfg, agentId }),
    }),
  },
  reservedNames: listReservedChatSlashCommandNames(),
});
```

`src/agents/skills/command-specs.ts:60-199` `buildWorkspaceSkillCommandSpecs`：

- `sanitizeSkillCommandName` —— 小写、非字母数字→`_`、截到 32 字符
- `resolveUniqueSkillCommandName` —— 与保留集合/已用集合冲突时加 `_2/_3`
- 描述截断到 `SKILL_COMMAND_DESCRIPTION_MAX_LENGTH`
- frontmatter → `dispatch` 字段：

```ts
const dispatch = (() => {
  const kindRaw = normalize(fm["command-dispatch"] ?? fm["command_dispatch"]);
  if (kindRaw !== "tool") return undefined;
  const toolName = (fm["command-tool"] ?? fm["command_tool"] ?? "").trim();
  if (!toolName) return undefined;
  return { kind: "tool", toolName, argMode: "raw" } as const;
})();
```

bundle 命令（`.claude/commands/*.md`，由 `src/plugins/bundle-commands.ts:159` `loadEnabledClaudeBundleCommands` 加载）以同一份 `SkillCommandSpec` 出现，但带 `promptTemplate` 而非 `dispatch`。

### 5.3 解析入站命令

`src/auto-reply/skill-commands-base.ts:59-99` `resolveSkillCommandInvocation`：

```ts
const match = trimmed.match(/^\/([^\s]+)(?:\s+([\s\S]+))?$/);
//                            ^^^^^^^^^   ^^^^^^^^^^^^
//                            命令名      其余全部（含换行）
```

支持两种入口：

- `/<name> <args>` —— 按 `entry.name` 精确匹配
- `/skill <name> <args>` —— 经 `findSkillCommand` 做多键匹配（name / skillName / `normalizeSkillCommandLookup` 后的等价形式：空格、`_`、`-` 互通）

切分只发生一次：**第一段空白**之前是命令名，之后整段（保留换行）作为 args。

### 5.4 派发器

`src/auto-reply/reply/get-reply-inline-actions.ts:268-360` 是真正的"三态派发"分叉点：

```ts
const skillInvocation = resolveSkillCommandInvocation({ ... });
if (skillInvocation) {
  if (!command.isAuthorizedSender) return reply(undefined);  // 静默丢弃
  const dispatch = skillInvocation.command.dispatch;
  if (dispatch?.kind === "tool") { /* 模式 A */ }
  const rewrittenBody = skillInvocation.command.promptTemplate
    ? expandBundleCommandPromptTemplate(...)                 // 模式 B
    : `Use the "${skillName}" skill ...`;                    // 模式 C
}
```

---

## 6. 三态派发

### 6.1 模式 A — Tool Dispatch（绕开 LLM）

**Frontmatter**：

```yaml
---
name: notify
description: Forward the rest of the message as a notification.
user-invocable: true
command-dispatch: tool
command-tool: sessions_send
---
```

**执行**（`get-reply-inline-actions.ts:285-340`）：

```ts
const tools = createOpenClawTools({ agentSessionKey, agentChannel, ... });
const authorizedTools = applyOwnerOnlyToolPolicy(tools, command.senderIsOwner);
const tool = authorizedTools.find(c => c.name === dispatch.toolName);
if (!tool) return reply(`❌ Tool not available: ${dispatch.toolName}`);

const toolCallId = `cmd_${generateSecureToken(8)}`;
const result = await tool.execute(toolCallId, {
  command: rawArgs,
  commandName: skillInvocation.command.name,
  skillName: skillInvocation.command.skillName,
}, opts?.abortSignal);
const blocked = extractBlockedToolReason(result);
if (blocked) return reply(`❌ Tool call blocked: ${blocked}`);
return reply(extractTextFromToolResult(result) ?? "✅ Done.");
```

特点：

- LLM 完全不参与
- SKILL.md 正文从不被读取
- 工具自身闭包持有会话上下文（session/channel/owner/...）
- 成功/blocked/异常都统一通过 `{kind:"reply", reply:{text}}` 返回

### 6.2 模式 B — Prompt Template（确定性 Prompt）

来源是插件 `.claude/commands/<name>.md`（由 `src/plugins/bundle-commands.ts:159` 加载，需要插件 `bundleFormat === "claude"` 且 `bundleCapabilities` 包含 `"commands"`）。

**模板示例**：

```markdown
---
name: review
description: Senior-engineer style code review.
---

Walk through the following snippet:
1. Correctness issues
2. Hidden side-effects
3. Test gaps

Snippet:

$ARGUMENTS

End with a punch list of fixes.
```

**展开**（`get-reply-inline-actions.ts:102-111`）：

```ts
function expandBundleCommandPromptTemplate(template: string, args?: string): string {
  const normalizedArgs = normalizeOptionalString(args) || "";
  const rendered = template.includes("$ARGUMENTS")
    ? template.replaceAll("$ARGUMENTS", normalizedArgs)
    : template;
  if (!normalizedArgs || template.includes("$ARGUMENTS")) {
    return rendered.trim();
  }
  return `${rendered.trim()}\n\nUser input:\n${normalizedArgs}`;
}
```

展开结果写入 `ctx.Body` / `ctx.BodyForAgent` / `sessionCtx.Body` / `sessionCtx.BodyForAgent` / `sessionCtx.BodyStripped` / `cleanedBody` 共 6 处，让下游 LLM 引擎只看到改写版。

特点：

- LLM 参与，但 prompt 是模板预编
- `$ARGUMENTS` 是唯一占位符，可多次出现
- 无占位符时自动以 `\n\nUser input:\n<args>` 追加（兼容兜底）

### 6.3 模式 C — 裸 Skill（LLM 自取 SKILL.md）

无 `command-dispatch` 且无 `promptTemplate`。两种进入路径：

**A. 隐式**：用户用自然语言，LLM 看 `<available_skills>` 描述判断是否相关，相关则用 `read` 工具加载完整 SKILL.md，再按正文指引调 `Bash` 工具。

**B. 显式 `/skill <name> <args>`**：派发器命中后落到 `get-reply-inline-actions.ts:348-353`：

```ts
const rewrittenBody = [
  `Use the "${skillName}" skill for this request.`,
  args ? `User input:\n${args}` : null,
].filter(Boolean).join("\n\n");
```

→ 改写后交给 LLM，LLM 再 `read` SKILL.md。

特点：

- 推理空间最大、token 成本最高、延迟最高
- 适合操作多样、需参数语义理解的工具集（CLI 包装、API 客户端）

### 6.4 三态对比

| 维度 | A. Tool Dispatch | B. Prompt Template | C. 裸 Skill |
|---|---|---|---|
| 触发 | `/<name> <args>` | `/<name> <args>` | LLM 主动 / `/skill <name> <args>` |
| LLM 参与 | ❌ | ✅ 接预编 prompt | ✅ 接 description，自取正文 |
| `args` 怎么用 | 全段给 `tool.execute({command,...})` | 模板 `$ARGUMENTS` 替换 | 拼成 `"User input:\n<args>"` |
| 成本 / 延迟 | 最低 | 中 | 最高 |
| 决策灵活度 | 最低（行为固化） | 中（模板预设） | 最高（LLM 推理） |
| 代码分支 | `:285-340` | `:343-347` | `:348-353` |

---

## 7. 参数提取

### 7.1 模式 A：核心层完全不解析

`command-specs.ts:144-153` 把 `argMode` 硬编码为 `"raw"`。工具收到固定三元组：

```ts
{ command: rawArgs, commandName, skillName }
```

`rawArgs` 是 `/<name>` 之后的整段字符串（保留换行）。**框架不切、不分割、不校验 schema**。

工具按需自选解析方案：

| 策略 | 用户输入示例 | 工具内部 |
|---|---|---|
| shell argv | `/ghpr review 1234 --depth full` | `parseShellArgv(command)` |
| 子命令 + JSON | `/ghpr review {"pr":1234,...}` | `command.split(" ", 2)` + `JSON.parse` |
| key=value | `/notify channel=ops priority=high build done` | tokenize + 前缀匹配 |
| 换行分隔 | `/blog title here\nbody here...` | `command.indexOf("\n")` |
| 子命令路由 | `/db query users where age > 30` | `[sub, ...rest] = argv` |

### 7.2 模式 B：单一 `$ARGUMENTS` 占位符

`expandBundleCommandPromptTemplate` 只识别 `$ARGUMENTS`，无 `$1` / `${name}` / `{{flag}}` 等多槽位语法。多参数场景的常见约定：

- **整段塞入**（最朴素）：用户写啥就让 LLM 看啥
- **模板里写"参数协议"**：用自然语言要求 LLM 按规则拆解 args
- **多处插入**：`$ARGUMENTS` 可多次出现到不同位置，构造多视角 prompt

### 7.3 关键结论

"多参数"在 OpenClaw 是**约定问题**，不是 schema 问题。框架只给一根原始字符串，由工具或模板（最终由 LLM）负责拆解。

---

## 8. 工具的定义与注册

### 8.1 通用接口

`src/agents/tools/common.ts:13-35`：

```ts
type AnyAgentTool = Omit<AgentTool<TSchema, unknown>, "execute"> &
  ErasedAgentToolExecute & {
    ownerOnly?: boolean;
    displaySummary?: string;
  };
```

每个工具对象：

```ts
{
  name: string,              // 与 command-tool 匹配的键
  label: string,
  description: string,       // 给 LLM 看的工具描述
  parameters: TSchema,       // typebox schema，LLM 调用时使用
  execute(toolCallId, params, signal?, onUpdate?): Promise<AgentToolResult>,
  ownerOnly?: boolean,
  displaySummary?: string,
}
```

注意：**dispatch 路径不走 parameters schema 校验**。工具实现需同时兼容 LLM schema 入参和 `{command, commandName, skillName}` 字符串入参。

### 8.2 核心工具组装

`src/agents/openclaw-tools.ts:72` `createOpenClawTools` 大工厂手工 push 约 20 个内置工具：

```ts
const tools: AnyAgentTool[] = [
  ...(embedded ? [] : [nodesTool, createCronTool({...})]),
  ...(messageTool && includeMessageTool ? [messageTool] : []),
  ...collectPresentOpenClawTools([heartbeatTool]),
  createTtsTool({...}),
  ...(embedded ? [] : [createGatewayTool({...})]),
  createAgentsListTool({...}),
  createSessionsListTool({...}),
  createSessionsHistoryTool({...}),
  ...(embedded ? [] : [createSessionsSendTool({...}), createSessionsSpawnTool({...})]),
  createSessionsYieldTool({...}),
  createSubagentsTool({...}),
  createSessionStatusTool({...}),
  ...collectPresentOpenClawTools([webSearchTool, webFetchTool, imageTool, pdfTool]),
];
```

每个 `createXxxTool(opts)` 返回 `AnyAgentTool`。例（`src/agents/tools/sessions-send-tool.ts:184-197`）：

```ts
export function createSessionsSendTool(opts?: {...}): AnyAgentTool {
  return {
    label: "Session Send",
    name: "sessions_send",
    displaySummary: SESSIONS_SEND_TOOL_DISPLAY_SUMMARY,
    description: describeSessionsSendTool(),
    parameters: SessionsSendToolSchema,
    execute: async (_toolCallId, args) => {
      const params = args as Record<string, unknown>;
      const message = readStringParam(params, "message", { required: true });
      const { cfg, mainKey, ... } = resolveSessionToolContext(opts);
      ...
    },
  };
}
```

工具闭包绑住构造时的会话上下文，这是 dispatch 路径不依赖 LLM 也能正确路由消息的根因。

### 8.3 插件工具注册

`createOpenClawTools` 在 `openclaw-tools.ts:443-458` 调 `resolveOpenClawPluginToolsForOptions` → `src/plugins/tools.ts:888` `resolvePluginTools` 收集所有插件贡献的工具。

插件 SDK 入口 `definePluginEntry`。例（`extensions/canvas/index.ts:128-133`）：

```ts
api.registerTool((ctx) =>
  createLazyCanvasTool({
    config: ctx.runtimeConfig ?? ctx.config,
    workspaceDir: ctx.workspaceDir,
  }),
);
```

`createLazyCanvasTool` 自身（`extensions/canvas/index.ts:20-43`）只暴露 name/description/parameters，真正的 `createCanvasTool` 在 `execute` 第一次被调时才 lazy import。

Manifest 声明（`extensions/canvas/openclaw.plugin.json:9-11`）：

```json
"contracts": { "tools": ["canvas"] }
```

manifest-first 设计让发现/校验不必加载运行时代码。

### 8.4 工厂调用时机

每条 dispatch 都**新建一套工具**（`get-reply-inline-actions.ts:292-308`）：

```ts
const { createOpenClawTools } = await loadOpenClawToolsRuntime();
const tools = createOpenClawTools({
  agentSessionKey: sessionKey,
  agentChannel: channel,
  agentAccountId: ctx.AccountId,
  agentTo: ctx.OriginatingTo ?? ctx.To,
  agentThreadId: ctx.MessageThreadId,
  ...
  senderIsOwner: command.senderIsOwner,
  sessionId: targetSessionEntry?.sessionId,
});
```

因为每条消息的会话身份不同。所以 factory 必须便宜——重的初始化要 lazy。

### 8.5 工具的横切关注点

- **`ownerOnly`**：`applyOwnerOnlyToolPolicy(tools, senderIsOwner)` 在 dispatch 前过滤
- **`before_tool_call` hook**：`openclaw-tools.ts:477-481` 默认用 `wrapToolWithBeforeToolCallHook` 包裹所有工具，dispatch 也会触发（loop detection、policy enforcement）。可通过 `wrapBeforeToolCallHook: false` 关闭
- **重名**：`existingToolNames` 集合优先核心工具，插件同名工具被剔除
- **embedded mode**：`isEmbeddedMode()` 时跳过部分需要 gateway 的工具

---

## 9. CLI 与管理

### 9.1 `skills` 命令组

`src/cli/skills-cli.ts:91` `registerSkillsCli` 向 Commander 注册：

| 子命令 | 实现 |
|---|---|
| `skills search [query]` | `searchSkillsFromClawHub` |
| `skills install <slug>` | `installSkillFromClawHub` |
| `skills list` | `loadWorkspaceSkillEntries` |
| `skills check` | `buildWorkspaceSkillStatus`（`src/agents/skills-status.ts`） |

### 9.2 安装来源

- **ClawHub 注册表**：`src/agents/skills-clawhub.ts` `installSkillFromClawHub` —— 下载存档、根目录标记校验、写 `.clawhub/origin.json`
- **包管理器**：`src/agents/skills-install.ts` 按 `metadata.openclaw.install[].kind` 走 brew / node / go / uv / download（先调 `scanSkillInstallSource` 做安全扫描）
- **远端**：`src/infra/skills-remote.ts`

### 9.3 状态审计

`SkillStatusEntry`（`skills-status.ts`）报告：disabled、blockedByAllowlist、blockedByAgentFilter、eligible、modelVisible、userInvocable、requirements、missing 等。

---

## 10. 时序总图

```
入站消息
  │
  ▼ commandTextHasSlash?
  │   no → 跳过 skill 扫描
  │   yes
  ▼
lazy import("../skill-commands.runtime.js")
  │
  ▼ listSkillCommandsForWorkspace({workspaceDir, cfg, agentId, skillFilter})
  │     ├─ loadWorkspaceSkillEntries（6 来源合并 + 去重）
  │     ├─ requirements/eligibility/exposure 过滤
  │     ├─ sanitize / dedupe / 避开 reservedNames
  │     └─ frontmatter → dispatch / promptTemplate
  ▼ SkillCommandSpec[]
resolveSkillCommandInvocation
  │     正则 /^\/([^\s]+)(?:\s+([\s\S]+))?$/  →  {name, args}
  │     支持 /<name>、/skill <name>、空格/_/- 互通
  ▼ { command, args }
  │
  ├─ command.isAuthorizedSender? no → 静默丢弃
  │
  ├─ dispatch.kind === "tool"（模式 A）
  │     ├─ createOpenClawTools({会话身份+channel+owner+config+...})
  │     │     ├─ 核心工具：createXxxTool({...}) push 进数组
  │     │     ├─ 插件工具：resolvePluginTools → api.registerTool 回调
  │     │     └─ wrapToolWithBeforeToolCallHook 包裹
  │     ├─ applyOwnerOnlyToolPolicy（owner-only 过滤）
  │     ├─ find name === dispatch.toolName ──缺──► reply ❌ Tool not available
  │     ├─ tool.execute(toolCallId, {command:rawArgs, commandName, skillName})
  │     ├─ extractBlockedToolReason ──有──► reply ❌ Tool call blocked
  │     ├─ extractTextFromToolResult / "✅ Done."
  │     └─ catch → reply ❌ <message>
  │
  ├─ promptTemplate 存在（模式 B）
  │     ├─ expandBundleCommandPromptTemplate(template, args)
  │     │     ├─ template.replaceAll("$ARGUMENTS", args), 或
  │     │     └─ `${template}\n\nUser input:\n${args}`
  │     └─ 改写 ctx.Body 等 6 处 → 继续进 LLM 引擎
  │
  └─ 兜底（模式 C）
        ├─ `Use the "${skillName}" skill for this request.\n\nUser input:\n${args}`
        └─ 改写 ctx.Body 等 6 处 → 进 LLM
              └─ LLM 看 <available_skills> + 改写 prompt → read SKILL.md → 调 Bash 等
```

---

## 11. 关键源文件清单

| 路径 | 角色 |
|---|---|
| `src/agents/skills.ts` | 公共 API 门面 |
| `src/agents/skills/workspace.ts` | 加载/合并/预算/格式化 |
| `src/agents/skills/frontmatter.ts` | YAML + metadata 解析 |
| `src/agents/skills/skill-contract.ts` | XML prompt 格式 |
| `src/agents/skills/command-specs.ts` | `SkillCommandSpec` 构造 |
| `src/agents/skills/plugin-skills.ts` | 插件 skills 发布 |
| `src/agents/skills/bundled-dir.ts` | 内置 skills 目录解析 |
| `src/agents/skills/tools-dir.ts` | 脚本工具目录映射 |
| `src/agents/skills-install*.ts` | 安装管线 |
| `src/agents/skills-clawhub.ts` | 注册表客户端 |
| `src/agents/skills-status.ts` | 状态审计 |
| `src/auto-reply/skill-commands.ts` | 命令列表生成 |
| `src/auto-reply/skill-commands-base.ts` | 入站命令解析 |
| `src/auto-reply/reply/get-reply-inline-actions.ts` | 三态派发器 |
| `src/auto-reply/reply/get-reply-directives.ts` | reply 流水线入口 |
| `src/plugin-sdk/skill-commands-runtime.ts` | SDK lazy facade |
| `src/plugins/bundle-commands.ts` | `.claude/commands/*.md` 加载 |
| `src/plugins/tools.ts` | 插件工具收集 |
| `src/agents/openclaw-tools.ts` | 核心工具大工厂 |
| `src/agents/openclaw-plugin-tools.ts` | 插件工具适配 |
| `src/agents/tools/common.ts` | `AnyAgentTool` 接口 |
| `src/agents/tools/sessions-send-tool.ts` | 典型工具实现示例 |
| `src/cli/skills-cli.ts` | CLI 注册 |

---

## 12. 设计要点小结

1. **目录契约 + frontmatter 路由**：skill 即是一个带 YAML 元数据的目录，无需代码注册
2. **按需正文加载**：系统提示只放 name+description+location，LLM 用 `read` 拉取正文
3. **lazy facade 强制懒加载**：`*-runtime.ts` 命名约定让 hot path 零成本
4. **三态派发同源**：`SkillCommandSpec` 是统一中间表示，按 `dispatch` / `promptTemplate` / 兜底分叉
5. **参数最小协议**：核心层只切一次空白，剩余字符串原样下发，由工具或模板自治
6. **工具复用零边际**：同一份 `AnyAgentTool` 既是 LLM 工具也是 dispatch 目标，通过 `{command, commandName, skillName}` 字段桥接两种入口
7. **每次派发新建工具**：会话身份在 factory 调用时绑死，工具不必从参数解析这些
8. **多层防护**：realpath 防 symlink 越界、扫描容量上限、安装源安全扫描、owner-only 工具过滤、before_tool_call hook 包裹
9. **令牌成本优化**：路径缩写、紧凑格式降级、二分搜索拟合预算
10. **manifest-first**：插件元数据驱动发现，运行时按需 lazy 加载

---

## 13. 与上层文档的关系

- 面向 skill 作者的用户文档：`/tools/skills`、`/tools/creating-skills`
- slash 命令机制：`/tools/slash-commands`
- 插件架构：`/plugins/architecture`、`/plugins/sdk-overview`

本内部文档面向**实现者**：在改动 skill 加载/派发/工具组装路径前，先在这里对齐心智模型。
