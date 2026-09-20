# CodeMaker 扩展能力清单与移植评估

> **性质**：外部组件调研 + 移植候选评估。**不是本仓现状文档，也不是已批准计划**，不得据此认为任何东西已实装。
> **调研对象**：`rez-package-source/codemaker-26.9.4`（CodeMaker VS Code 扩展的 VSIX 解包快照）。
> **抽样日期**：2026-09-20（引用位置以函数/类名与字符串为准；该文件是压缩产物，字节偏移会随版本变化）。
> **一句话结论**：这个快照的价值不在 `extension/` 树（**里面没有任何 TS/JS 源码**），而在它捆绑的 Agent 包 `resources/language-server/codemaker-agent-v26.8.12.zip`。解开后可从 `dist/node/index.js` 挖出完整的**策略/治理层行为**；其中规则注入、ignore、hooks、MCP、spec 解析这几层恰好是 `l_agent_chat` 的空白。

---

## 0. 这份结论是怎么来的（复现方法）

前提：`extension/` 树全目录只有一个 js（`pnpm_v6.js`），`package.json` 的 `main` 指向未打包的 `./out/extension.js`。所以**扩展源码不在这里**，只能挖捆绑 Agent。

```powershell
# 1) 解包捆绑 Agent（29 个条目）
tar -xf "<repo>\rez-package-source\codemaker-26.9.4\extension\resources\language-server\codemaker-agent-v26.8.12.zip" -C <out>
```

```python
# 2) 按 [Tag] 前缀归属日志 —— index.js 压缩后日志里有完整英文文案 + 类名 + 配置键
s = open("dist/node/index.js", encoding="utf-8", errors="replace").read()
# 模块普查
Counter(re.findall(r"\[([A-Z][A-Za-z0-9]{2,28})\]", s))
# 行为文案（按 Tag 归属）
lits = re.findall(r'"((?:[^"\\\n]|\\.){6,220})"', s)      # + 反引号版
m = re.match(r"\[([A-Za-z][A-Za-z0-9 _-]{2,40})\]\s*(.{4,190})", lit)
# 构造器窗口（拿默认值/目录名）
[i for i in range(len(s)) if s.startswith('se("RulesHandler")', i)]  # 然后取该处后 600 字符
```

**踩过的坑（别再走）**：

- **按 logger 变量归属会错**。`loggerVar = se("RulesHandler")` 这个映射看似能用，但绝大多数类是 `this.logger = se("X")`，变量名统一是 `logger`，全被归到第一个模块头上。**必须按 `[Tag]` 前缀归属**。
- `len(re.findall(r'"([^"]+)"', s))` 会漏转义引号，务必带上 `\\` 分支。
- `python` 不在 PATH；用 `rez-package-3rd\python\3.12.10\python.exe`。
- 别对 `index.js` 直接 grep 后打印——单行可达数万字符，会把输出打爆。先切窗口再读。

---

## 1. 包里到底有什么（29 项）

| 条目 | 大小 | 性质 |
|------|------|------|
| `dist/node/index.js` | 3.08 MB | **CodeMaker 的 Agent 本体**，单文件、已压缩混淆 |
| `dist/node/win-ca/roots.exe` | 83 KB | 证书根采集 |
| `rg/{win32-x64,darwin-x64,darwin-arm64,linux-x64,linux-arm64}/rg[.exe]` | 4.5–5.7 MB | 真 ripgrep 二进制，5 平台齐 |
| `resources/doctor-agent-extensions/doctor-tools.ts` | 7.8 KB | **可读 TS 源码**，Doctor Agent 的 4 个工具 |
| `wasm/tree-sitter-*.wasm` + `wasm/tree-sitter.wasm` | **131 B** | **Git-LFS 指针，不是真 wasm** |
| `version.json` | 24 B | 版本 |

`tree-sitter-python.wasm` 全文就三行，实测：

```
version https://git-lfs.github.com/spec/v1
oid sha256:72d0f97ba6c3134d7873ec5c9d0fd3c1f5137f4eac4dda0709993d92809e62b6
size 474189
```

**推论**：本副本无法运行 tree-sitter/Code Map（语法文件缺失）。`extension/tree-sitter/` 下那批多半同理。

---

## 2. 运行时骨架

| 项 | 证据 |
|----|------|
| 运行时是第三方框架 **Pi**（`@mariozechner/pi-coding-agent`） | `spawn(piBinary, {systemPrompt, provider, model, sessionFile, initialPrompt})`；`[PiProcess] spawning: ${piBinary} ... (sessionFile=...)`；`failed to write initial prompt to stdin` |
| 插件装载点 | `{PI_CODING_AGENT_DIR}/extensions/`，用 Pi 的 `registerTool` 注册 |
| Doctor 会话落盘 | `~/.codemaker/doctor-agent/sessions/<sessionId>.json` |
| 父子 IPC | LSP 的 `extension_ui_request` / `extension_ui_response`，magic title `__codemaker_kb_lookup__` / `__codemaker_plugin_context__` / `__codemaker_origin_session_context__` |
| Doctor provider 约定 | `provider="codemaker-doctor-agent"`、默认模型 `qwen3.6-plus`、header `mode_type=doctor`、`x-opencode-session-id` |
| 健壮性 | `EventLoopBlockMonitor`；`Uncaught exception swallowed to keep process alive` / `Unhandled rejection swallowed to keep process alive` |

捆绑的 Doctor Agent system prompt 是明文，可取全文。两条要点：

- `permission_tool(tool_name, reason, command?)` — "Request explicit user authorization before using bash, write, or edit."
- `Before every tool call, narrate your intent in one short Chinese sentence (5-15 words)` — 与本仓 `l_agent_chat` 的「工具调用必须说明原因并置顶」是同一套设计。

---

## 3. 规则注入层（`l_agent_chat` 全缺）

`RulesHandler`：`rulesDirectory=".codemaker/rules"`，watcher 默认开，workspace 变更即重init。**六个来源**：

| 来源 | 证据 |
|------|------|
| `.codemaker/rules` | `Loading rules from ${n} - directory:` |
| 用户级 CodeMaker rules | `User-level CodeMaker rules loaded - count:` / `User-level CodeMaker rules directory not found` |
| **Cursor rules** | `Found ${n.length} .cursor/rules directories`、`Found mdc files in ${n}/.cursor/rules`、`Reached maximum recursion depth (${n})`、symlink realpath 去重 |
| `.codemaker.codebase.md` | `Loading Original codebase rule...` |
| `AGENTS.md` | `Loading AGENTS.md...`，实际是 `join(workspaceRoot, "AGENTS.md")` |
| `CLAUDE.md` | 同族来源 |

`.mdc` 解析：`extractMetaData` / `extractContent` / `Failed to stringify MDC content`（frontmatter 元数据）。

### 3.1 `.mdc` frontmatter 契约（已落地到 `rules.py`）

分隔符常量 `FRONT_MATTER_DELIMITER = "---"`。解析规则：

- `\r\n` 归一为 `\n` 后 trim；**不以 `---` 开头**则视作无 frontmatter，`metaData = {alwaysApply: false}`，全文当正文，`success = true`
- 行数 < 3 或找不到收尾 `---` → 视为格式错误（`Invalid MDC format: ...`）
- 字段只有三个 + globs：`name`、`description`、`alwaysApply`、`globs`（都要 trim）
- `alwaysApply` 取布尔真值；`globs` 支持字符串（逗号分隔）与数组
- 解析前会**把未加引号的 globs 值补上引号**（`globs: a/**` → `globs: "a/**"`）再喂 YAML —— 因为 `**` 开头的值容易把 YAML 解析搞崩
- 解析失败**不抛**：返回 `{metaData:{alwaysApply:false}, content:"", success:false, error}`，调用方只记错误

作用域语义（Cursor 那套）：`alwaysApply: true` = 常驻；有 `globs` = 按文件匹配；只有 `description` = 模型按需取；都没有 = 手动。

本仓实测样本 `.cursor/rules/rez-package-source-wuwor.mdc`：

```yaml
---
description: rez-package-source 下的 Rez 包使用 wuwor 运行与测试
globs: rez-package-source/**
alwaysApply: true
---
```

---

## 4. ignore 治理

| 项 | 证据 |
|----|------|
| 文件与 mode 头 | `".codemaker/.codemakerignore"`；`/^#\s*mode\s*:\s*(allowlist|denylist)\s*$/i` |
| 状态字段 | `localMode` / `remoteRules` / `remoteMode` / `mode="denylist"` / `loadedMtimeMs` / `ruleIgnoreCache` / `remoteLastFetchMs` |
| 热重载 | `refreshIfChanged()` 按 **mtime** 判断，未变不重算 |
| 空 allowlist 语义 | `allowlist mode with no rules: all paths will be blocked` |
| 远端规则 | `CodemakerIgnoreRemote`：按 **department 路由**拉配置；`no routing config; remote ignore off`、`no department in validate response; remote ignore off`、`remote config loaded: mode=${p??"undeclared"}, rules=`；拉失败**保留本地规则** |
| 执行点 | `SearchHandler`: `READ_FILE blocked by .codemakerignore: ${s}`；`searchGrep`: `protected dir pass skipped for "${g}"` |

---

## 5. Hooks 层（`l_agent_chat` 无）

| 项 | 证据 |
|----|------|
| 四类 handler | `command` / `http` / `mcp_tool` / `prompt`（条件求值） |
| command shell 探测 | `shell:'powershell' but no PowerShell (pwsh/powershell) found — failing open`、`Git Bash not found — falling back to PowerShell`、`No usable shell (Git Bash / PowerShell) on Windows — failing open` |
| HTTP 白名单门 | `HTTP hook blocked by allowedHttpHookUrls gate` |
| **统一 fail-open** | `prompt eval threw ... (fail-open)`、`mcp_tool hook skipped [hookId=]: MCP call entry not wired (fail-open)`、`HOOK 超时` 亦 fail-open |
| 可阻断性 | `Event ${r} is not blockable, but triggerBlockable was called` |
| matcher | `HookMatcher`：exact / regex / `invalid regex` |
| 双轨校验 | `dualtrack diff source= zodOk= legacyOk=`、`Both zod and legacy validation failed` |
| 迁移 | `HookConfigMigrator`：`Renaming legacy event key`、迁移 tool matcher、丢弃不支持字段 |
| 回滚 | `ConfigChange denied for ${o} — rolling back to pre-reload config` |
| 输出校验 | `hookSpecificOutput.hookEventName mismatch: expected '...', got '...'. Ignoring hookSpecificOutput.` |
| 进程管理 | `killProcessTree` → `taskkill.exe /pid <pid> /t /f`（win32 分支） |

---

## 6. Claude Code 兼容层（可直接抄的资产）

`CCSettingsLoader` 路径族：

```
managed : win32  C:\Program Files\ClaudeCode\managed-settings.json
          darwin /Library/Application Support/ClaudeCode
          linux  /etc/claude-code
user    : <ClaudeUserConfigDir>/settings.json
project : <workspaceRoot>/.claude/settings.json
local   : <workspaceRoot>/.claude/settings.local.json
```

- 非法 JSON / schema 不过 → **拒整个 source**（`rejecting source`），不是只跳坏条目。
- `allowManagedHooksOnly ignored — only honored from managed (policy) settings`：特权开关只认 policy 层。
- `ClaudeAliasEnvCache` 带文件 watcher，刷新失败**保留旧缓存**。

---

## 7. Skills / Commands / Agents / SubAgent

| 模块 | 证据 |
|------|------|
| `SkillsHandler` | 四张表 `skills` / `pluginSkills` / `commands` / `pluginCommands`；多 source；无 workspace 时只载用户级；`RELOAD_DEBOUNCE_MS=300`；`MAX_SKILL_DIRS_PER_ROOT` 目录预算；`Max scan depth reached`；symlink realpath；`Skipping plugin folder (has manifest)` |
| `SkillFeature` | `INSTALL_BUILTIN_SKILL` / `UPLOAD_SKILL` / `CHECK_SKILLS_VERSION` / `RELOAD_SKILLS` + skill 模板 |
| `AgentsHandler` | `agents` / `pluginAgents`；`HUB_REFRESH_TIMEOUT_MS=5000`；hub 刷新超时→**保留旧快照**；`timedOutHubRefreshes` 计数 |
| `AgentParser` | agent 定义 = md + YAML frontmatter；`mcpServers has null entries ... Check YAML indentation`；unsupported field/tool 忽略而非报错 |
| `AgentConfigSubagentRuntime` | subagent **私有 MCP + skills + resource 作用域**；`open agent=${n} resource=${i} skills=${c.length} mcp=${u.length}/${l.length}` |
| `CommandRegistry` | `duplicate command dropped (first-wins)` |
| `AgentSwarmRegister` | 注册到 swarm hub；心跳失败退避；secret 落 `~/.codemaker/plugin-bridge/agent-secrets.json`，**0o600 + tmp rename 原子写** |

---

## 8. MCP 层（`l_agent_chat` 最大缺口）

| 项 | 证据 |
|----|------|
| 管理器 | `McpManager`：mutex 串行、`connections`、`reconnectInFlight`、transport 根因/stderr 尾/spawn 路径缓存、env 刷新时间表、`credentialPolicy` |
| **自愈** | `MCP callTool on disconnected stdio server, reconnecting first: {}`、`MCP callTool self-heal retry: {} on {} -> success`、`spawn ENOENT → refreshed PATH → retrying` |
| 配置治理 | `McpConfigHandler`：user + project 配置；**plugin server 只读**；`Invalid format, expected "mcpServers" object`；配置不存在则创建；父目录 watcher 兜底 |
| 工具面 | `use_mcp_tool`、`access_mcp_resource` |
| hook 联动 | hooks 的 `mcp_tool` 类型直接调 MCP（未接线时 fail-open） |

---

## 9. Spec 解析层

`SpecHandler` 维护 `knownChanges` / `knownFeatures` / `knownArchives` / `knownPlans`；目录不存在则 watch 其创建。四个 parser：

| parser | 证据 |
|--------|------|
| `OpenSpecParser` | `framework="openspec"`, `rootDirName="openspec"`；`specs/<capability>/spec.md` + `changes/*.md`；`Detected version` / `capabilities` / `activeChanges` |
| `SpecKitParser` | `constitution.md` + features |
| `SuperpowersParser` | plans |
| 编排 | `WebviewFeature`：OpenSpec setup / **OpenSpec 0.23 → 1.x 升级** / SpecKit setup |

---

## 10. 检索层

`SearchFeature` 三个操作：**GLOB / GREP / READ_FILE**（参数 `pattern` / `path` / `offset` / `limit`）。

- `Ripgrep`：`search timed out after ${n}ms, killing rg`；按 `@vscode/ripgrep-${process.platform}-${arch}` 解析二进制。
- 另有 `RecentlyChangedCodeSearch`（近期改动检索）、`DeclarationsSnippetsProvider`（声明片段）。

---

## 11. 编辑应用层

`ApplyFeature` / `ApplyHandler` / `SmartApplyFeature`：`APPLY_WRITE` / `APPLY_EDIT` / **`ACCEPT_EDIT`**，per-webview 定向广播，窗口已关则回退广播；三个入口都记 `encoding=`（编码保真）。

与 `l_agent_chat` 的 `edit_file`（含 CRLF 保真）对等，差别在**多了逐次 accept 流**。

---

## 12. 明确不可移（IDE 宿主绑定）

`WorkspaceTunnel`、`ChannelRouter`、`WorkerClient`、`LocalServer`、`DiscoveryReporter`、`PluginBridge`、`WebviewFeature`、`AgentHubClient`、`HubDetect`、`IDEWebview`，以及 `ApplyHandler` 的前端一半。理由：全部依赖 VS Code 宿主 / Webview panel 语义。

---

## 13. 与 `l_agent_chat` 的差距对照与优先级

| # | 可移植项 | `l_agent_chat` 现状 | 备注 |
|---|----------|---------------------|------|
| 1 | **RulesHandler 多源** | 零（只有单一全局 `AI_PREFERENCE`） | 不止 AGENTS.md：还吃 Cursor rules 递归 + codebase.md + 用户级；本仓一堆 `AGENTS.md` 可直接当样本 |
| 2 | **ignore（mode 头 + 远端路由）** | 检索无 ignore 感知 | 字段与语义已取全；`READ_FILE blocked` 证明是工具层强制 |
| 3 | **Hooks（CC 兼容 schema）** | 无 | 抄 schema 即得 Claude Code 生态兼容；fail-open 语义明确 |
| 4 | **CCSettingsLoader** | 无 | 路径族已取全，等于白拿「读用户已有配置」 |
| 5 | **MCP（客户端 + 治理 + 自愈）** | **完全没有 MCP 客户端** | 需用 Python MCP SDK 重写，不是搬码；自愈策略最值得抄 |
| 6 | **OpenSpecParser** | 无 | 本仓正在跑 OpenSpec，接上后 agent 能直接读 changes/specs |
| 7 | **shell env 采集** | 无 | 证据是那条 PowerShell 读 Machine + User PATH 的双段输出；本仓是托盘 GUI 启动，PATH 缺失是真实痛点 |
| 8 | **ripgrep 后端** | `search_text` 是纯 Python 正则逐行扫 | 按仓规不能硬编码外部路径，需走资源/依赖治理 |

**不要移植**（已有对等物或会重复）：工具定义与执行（`l_agent_tool.DEFAULT_TOOLS`）、工具审批与 trace UI、会话/历史、设置注册表（`config.py._SCHEMA`）、文档检索（`rez_knowledge.py`）、端口注册表（`service_ping`）。

---

## 14. 可顺手抄走的安全设计模式

1. **白名单门**：`allowedHttpHookUrls`——外呼前先过配置白名单。
2. **统一 fail-open**：hook / 评估类功能失败一律不阻断主流程，但要打日志区分。
3. **schema 不过就拒整个 source**，不在坏配置上做部分降级。
4. **特权开关只认 policy 层**：`allowManagedHooksOnly` 只在 managed settings 生效。
5. **配置只读边界**：plugin 提供的 MCP server 不可被用户侧修改/删除。
6. **凭据落盘**：`0600` + 临时文件 + rename 原子写。
7. **快照保留**：远端刷新失败/超时保留上一份快照，不清空。

---

## 15. Hooks 规格（可直接当移植底稿）

以下全部来自压缩码里的 zod schema 与 `executeHooksCore` 主循环，字段名逐字抄录。`Q.object({...}).loose()` 表示**未知键放过**。

### 15.1 事件枚举（内部常量 `Nc`）

左侧是源码里的语义键名，右侧是字符串值（配置里写的是右侧）：

```
SessionStart              SessionStart
SessionEnd                SessionEnd
MessageUserSubmit         UserPromptSubmit
MessageBeforeSend         message.beforeSend
MessageComplete           message.complete
ToolBeforeExecute         PreToolUse
ToolAfterExecute          PostToolUse
FileBeforeEdit            file.beforeEdit
FileAfterEdit             file.afterEdit
FileBeforeCreate          file.beforeCreate
FileAfterCreate           file.afterCreate
TerminalBeforeExecute     terminal.beforeExecute
TerminalAfterExecute      terminal.afterExecute
PreCompact / PostCompact  PreCompact / PostCompact
UserPromptExpansion       UserPromptExpansion
PostToolBatch             PostToolBatch
Stop / StopFailure        Stop / StopFailure
SubagentStart / SubagentStop
Setup / InstructionsLoaded
MessageDisplay
PreToolUseFailure / PostToolUseFailure
PermissionRequest / PermissionDenied
Notification
TaskCreated / TaskCompleted / TeammateIdle
ConfigChange / CwdChanged / FileChanged
WorktreeCreate / WorktreeRemove
Elicitation / ElicitationResult
SessionTaskStart / SessionTaskComplete / SessionClose / SessionUnreadIdle
```

注意命名不统一：前 12 个是**语义键名**（`ToolBeforeExecute`），后 24 个键名与字符串值同名。配置里一律用字符串值。

### 15.2 可阻断事件集合（`oM`，18 项）

```
UserPromptSubmit, PreToolUse, file.beforeEdit, file.beforeCreate,
terminal.beforeExecute, PreCompact, UserPromptExpansion, PostToolBatch,
Stop, SubagentStop, PermissionRequest, ConfigChange, WorktreeCreate,
TaskCreated, TaskCompleted, TeammateIdle, Elicitation, ElicitationResult
```

`D_(event)` 即"是否可阻断"。**其余事件返回 deny 会被忽略并 warn**（`Hook returned deny for non-blockable event`）。

### 15.3 配置 schema

顶层 hook 文件：

```
{
  version?:             string,
  description?:         string,
  syncCcHooksConfigs?:  boolean,           // 是否并入 Claude Code 的 settings.json 各层
  hooks?:               Record<EventName, HookEntry[]>
}
```

`HookEntry`：

```
{ matcher?: string, hooks: Handler[] }
```

`Handler` 按 `type` 判别：

| type | 字段 |
|------|------|
| `command` | `command: string, args?: string[], name?, timeout?: number, async?: boolean, asyncRewake?: boolean, shell?: "bash"\|"powershell", if?: string, statusMessage?, env?: Record<string,string>, cwd?: string` |
| `http` | `url: string, name?, timeout?, async?, headers?: Record<string,string>, allowedEnvVars?: string[]` |
| `mcp_tool` | `server: string, tool: string, input?: Record<string,unknown>, name?, timeout?, async?, if?` |
| `prompt` | `prompt: string, model?, name?, timeout?, if?, statusMessage?, once?: boolean` |

设置门（独立 schema，写在同一个文件顶层）：

```
{
  disableAllHooks?:        boolean,
  allowManagedHooksOnly?:  boolean,
  allowedHttpHookUrls?:    string[],
  httpHookAllowedEnvVars?: string[],
  enabledPlugins?:         Record<string, string[] | boolean | undefined>
}
```

### 15.4 输出 schema

顶层返回（loose）：

```
{
  decision?:        string,      // 语义值见下
  reason?:          string,
  continue?:        boolean,
  stopReason?:      string,
  suppressOutput?:  boolean,
  systemMessage?:   string,
  terminalSequence?: string,     // 内部做了白名单变换 cK()
  hookSpecificOutput?: { hookEventName: <必填，判别键>, ...逐事件字段 }
}
```

`hookSpecificOutput` 是**按 `hookEventName` 判别的联合**，逐事件可选字段：

| hookEventName | 可选字段 |
|---------------|----------|
| `SessionStart` | `additionalContext`, `initialUserMessage`, `watchPaths[]`, `sessionTitle`, `reloadSkills` |
| `SessionEnd` | （无） |
| `UserPromptSubmit` | `additionalContext`, `suppressOriginalPrompt`, `sessionTitle` |
| `PreToolUse` | `permissionDecision: enum(allow\|deny\|ask)`, `permissionDecisionReason`, `updatedInput: Record<string,unknown>`, `additionalContext` |
| `PostToolUse` | `additionalContext`, `updatedToolOutput` |
| `PreCompact` / `PostCompact` | （无） |
| `message.beforeSend` / `message.complete` | （无） |
| `file.beforeEdit` | `updatedContent` |
| `file.afterEdit` | （无） |
| `file.beforeCreate` 起后续 | **本次未取全**（窗口被截断），需再挖 |

### 15.5 决策合并语义（`executeHooksCore`）

- 按匹配到的配置顺序遍历，`decision` 字符串 switch：
  - `"deny"` → 仅当事件可阻断才生效；否则 warn 忽略
  - `"ask"` → **不能覆盖已定的 deny**（源码即 `a!=="deny" && (a="ask")`）
  - 其他/缺省 → 若尚未定则置 `"allow"`
- 连带效应收集（各自累积进独立数组，链尾统一应用）：`newCustomInstructions`、`hookSpecificOutput.additionalContext`、`updatedToolOutput`、`stopReason`、`suppressOutput`、`systemMessage`、`initialUserMessage`、`watchPaths`、`reloadSkills`、`sessionTitle`、`updatedInput`、`displayContent`、`terminalSequence`
- 任一 handler 返回 `continue === false` → **立即中断整个链**
- 条件：`if` 在非 tool 事件上无法求值 → 跳过该 handler（`cannot be evaluated on non-tool event`）
- 短路：所有 config 都被设置门关掉 → 直接 allow（`All N configs gated off — short-circuit allow`）
- `async: true` 的 handler 走另一条路（`Firing async <type> hook for <event>`），不阻塞主链
- 输出校验：`hookSpecificOutput.hookEventName` 与当前事件不符 → **丢弃整个 hookSpecificOutput**（只 warn）

### 15.6 配置来源与合并（`HookConfigLoader`）

| 源 | 路径 / 条件 |
|----|-------------|
| `projectSettings` | `<workspace>/.codemaker/hooks.json` |
| `userSettings` | `~/.codemaker/hooks.json` |
| CC 各层 | 仅当 `syncCcHooksConfigs` 为真时加载，见 15.7；源名含 `policySettings` |
| plugin | 来自插件清单，见 15.9 |

- 设置门合并：`disableAllHooks` / `allowManagedHooksOnly` **任一为真即为真**；`allowedHttpHookUrls` / `httpHookAllowedEnvVars` 取**并集**
- `warnIfAllowManagedHooksOnlyMisplaced`：非 `policySettings` 源设了它 → warn 并忽略
- `snapshot()` / `restore()`：供 `ConfigChange` 被拒时回滚到 pre-reload 配置
- 校验为**双轨**（zod 与 legacy 各跑一遍，对比结果：`dualtrack diff ... zodOk= legacyOk=`），两者都失败才判无效

### 15.7 Claude Code settings 层（`CCSettingsLoader`）

```
managed : win32  C:\Program Files\ClaudeCode\managed-settings.json
          darwin /Library/Application Support/ClaudeCode
          linux  /etc/claude-code                  （其他平台一律按 linux）
user    : <ClaudeUserConfigDir>/settings.json
project : <workspaceRoot>/.claude/settings.json
local   : <workspaceRoot>/.claude/settings.local.json
```

校验策略：gate 键非法 → **删键并记** `hooks-gate:<key>`；`hooks["<event>"]` 非数组 → 记错；单条 entry 校验不过 → 记 `hooks["<event>"][<idx>]`。**非法 JSON 的整个源被拒**。

### 15.8 旧格式迁移（`HookConfigMigrator`）

- 配置路径：`<workspace>/.codemaker/hooks.json` + `~/.codemaker/hooks.json`（常量 `NOe=".codemaker"`、`MOe="hooks.json"`）
- `migrateEventKeys`：查 legacy → 新事件名映射表（源码变量 `OM`，**映射内容本次未取到**），命中则改键并**合并数组**，不命中则原样保留
- `migrateMatcherField`：老 `tool` 字段 → 新 `matcher`（数组 join 成 `a|b`）；若同时存在 `path` / `command` → **无法自动迁移**，warn 要求人工改
- 迁移会丢弃不支持字段（`dropped unsupported field(s)`）

### 15.9 插件清单默认值（`PluginScanner`）

```
skillsDirRoots   : ./skills/
agentsFiles      : ./agents/
commandsPaths    : ./commands/
hooks            : ./hooks/hooks.json
mcpServers       : ./.mcp.json
```

声明为 `inline` 时直接用清单内联对象；显式列出时与默认值**并集去重**。已知不支持项会被记录：`outputStyles`、`lspServers`、`experimental.themes`、`experimental.monitors`。

### 15.10 工具匹配器（命令路径守卫，即前面 #3 候选的实现）

```
x1t = new Set(["PreToolUse", "PostToolUse", "PostToolUseFailure",
               "PermissionRequest", "PermissionDenied"])
```

- 只对这 5 个事件构造 matcher
- 取值用工位回退：`tool_name || toolName`；输入取 `tool_input ?? args ?? toolInput ?? {}`；再补 `path` / `filePath` / `command`（`command` 还会从 `input.command` 兜底）
- 匹配选项带 `bashParseFailureMatches: true` → **复合命令解析失败时也允许匹配**，不因解析异常漏拦

### 15.11 执行环境与信任

- command hook 的 shell 探测顺序：git-bash → powershell/pwsh；Windows 两者都没有 → **fail-open**
- 进程树清理：`taskkill.exe /pid <pid> /t /f`（win32）
- HTTP hook 超时默认 `10s`（`e.timeout || 10`），且受 `allowedHttpHookUrls` 白名单门控
- **hook 信任台账**：`~/.codemaker/`，结构 `{version:1, records:{<id>:{trusted, revision, updated_at}}}`；`set` 时 `revision` 自增；原子写（`<file>.tmp-<uuid>` → fsync → rename），目录模式 `0700`、文件 `0600`
- 配置目录 `~/.codemaker/agent-config/`，另见 `.agent-lock.json`

---

## 16. MCP 规格

### 16.1 配置位置与合并（`McpConfigHandler`）

| 源 | 路径 |
|----|------|
| user | `~/.codemaker/mcps.json` |
| project | `<workspace>/.codemaker/mcps.json` |

- 顶层键必须是 `mcpServers` 对象；格式非法 → 记错（`expected "mcpServers" object`）并可创建模板文件
- 插件提供的 server **只读**：不可新增/删除（`plugin servers are read-only`）
- 合并顺序 user ← project 覆盖，结果变化才发 `configChanged`
- `RELOAD_DEBOUNCE_MS = 300`；两个文件都 watch，文件/父目录不存在则 watch 祖先目录等创建

### 16.2 传输三型（`McpManager.createTransport`）

| type | 实现要点 |
|------|----------|
| `stdio` | `command` / `args` 做 `${WORKSPACE}` 变量替换；env 由凭据策略合成（见 16.4）；cwd 取 `workspacePath` 否则版本目录；`stderr` 走 pipe 并抓尾巴；`onerror` 区分 **spawn 错误**（有 `syscall`/`errno` 或 message 以 `spawn ` 开头）与 **stderr 错误**，日志前缀不同 |
| `sse` | EventSource；`max_retry_time: 5000`；仅当有 `Authorization` 头才 `withCredentials` |
| `streamableHttp` | 专用 client + 自定义 dispatcher；`onerror` 记 `MCP server HTTP error` 并追加到 server.error |
| 其他 | `throw new Error("Unknown transport type: " + type)` |

### 16.3 配置字段白名单（校验）

- `type` 缺省但有 `command` → **按 stdio 处理**
- `http` / `streamable-http` → 归一为 `streamableHttp`
- stdio 允许字段：`type, command, args, env, disabled`
- 远程允许字段：`type, url, headers, timeout, disabled`
- 出现白名单外字段 → 直接报错（`has unsupported field "<f>"`），不是忽略
- `disabled` 必须是 boolean；server 名必须非空、不含 `\0`；每个 server 必须是对象

### 16.4 凭据策略（`credentialPolicy`，默认实现 `UQ`）

stdio env 合成顺序（后者覆盖前者）：

```
configuredEnv  ←  平台注入(kind=mcp-stdio, workspacePath)  ←  WORKSPACE
              ←  AUTH_USER(server.user)  ←  ACCESS_TOKEN(server.token)  ←  PATH
```

远程 headers：

```
User-Agent: CodeMaker-Language-Server/<version>  +  configuredHeaders
若 enableNeteaseAuth 为真 或 url 含 netease.com：
    补 X-Auth-User / X-ACCESS-TOKEN（大小写不敏感判重，已存在则不覆盖）
```

auth 变更时**重启所有** `enableNeteaseAuth` 为真或 url 含 `netease.com` 的远程 server。

### 16.5 自愈与健康（本清单里最值得抄的部分）

| 机制 | 参数 / 行为 |
|------|-------------|
| 心跳 | 默认 **10s**（可被 server 的 `heartbeatTimeout`（秒）覆盖）；`inFlightRequests > 0` 时**跳过**（busy）；`ping` 成功 → `consecutiveFailures = 0`；迟到回复也忽略 |
| stdio 发送停滞看门狗 | **15s** 内无 drain → 抛 `McpTransportStalled: stdio send stalled on <server>`，并触发 transport onerror 走自愈 |
| 连接超时 | `Promise.race` + `unref()`，超时后 `close()` 再抛 |
| 调用遇断连 | 先重连再调（`reconnecting first`），失败后再试一次（`self-heal retry … -> success`） |
| spawn ENOENT | 刷新 PATH 后重试（`spawn ENOENT → refreshed PATH → retrying`） |
| 并发去重 | `reconnectInFlight` map 保证同一 server 只跑一次重连 |
| stderr 尾巴 | 环形缓冲 **8 KiB**；输出截到 **2 KiB**；**UTF-8 解码出现 >4 个替换字符 → 回退 gb18030 解码**（中文 Windows 专用兜底） |
| 错误链展平 | 递归 `cause` 拼成 ` <- Name[code]: message` |
| 诊断缓存 | 三张 WeakMap：`transportRootCauses` / `transportStderrTails` / `transportSpawnPaths`，外加 `envRefreshAt` |

### 16.6 能力探测（`fetchServerCapabilities`）

- 超时 = `options.capabilityTimeoutMs` 或 `server.timeout ?? 默认 × 1000`（秒→毫秒）
- 先读 server 通告的 capabilities，**按通告项并发**拉：`tools/list` → `{name, description, inputSchema, autoApprove}`；`resources/list` → `{name, uri, description, mimeType}`；`resources/templates/list`；`prompts/list`
- `autoApprove` 由 server 配置的 `autoApproveTools` 白名单决定 → **与 `l_agent_chat` 的 `auto_approve` 同一思路，但粒度是"工具名白名单"**
- 单项失败 → **该项置空数组**，不影响其他项
- 超时判定：正则 `/timed?\s*out|timeout/i` 匹配错误文本

### 16.7 配置变更传播（`updateServerConnections`）

mutex 串行化，然后按 name 求差：

- 消失的 server → 删除连接
- `JSON.stringify(config)` 变化 → 记 `config changed, reconnecting` 并重连
- `disabled: true` → 加为禁用态（不连接）
- 重名 → warn `Server already exists, will replace`

### 16.8 内建重连退避（streamableHttp SDK 默认）

```
initialReconnectionDelay  = 1000
maxReconnectionDelay      = 30000
reconnectionDelayGrowFactor = 1.5
maxRetries                = 2
```

退避公式 `min(initial × grow^n, max)`；**服务端下发的 retry 值优先**（`_serverRetryMs` 非空即直接用）。

### 16.9 完整配置位置总表（`l_agent_chat` 接入时的对齐参照）

由诊断打包器 `ZOe` 逐条枚举而来，等于一份权威路径清单：

| 归属 | 路径 |
|------|------|
| user MCP | `~/.codemaker/mcps.json` |
| user hooks | `~/.codemaker/hooks.json` |
| user 规则/技能/代理 | `~/.codemaker/rules`、`~/.codemaker/skills`、`~/.codemaker/agents` |
| user 日志 | `~/.codemaker/log`、`~/.codemaker/logs`、`~/.codemaker/pluginLog`、`~/.codemaker/codemaker-language-server/logs` |
| user CC | `~/.claude/settings.json`、`~/.claude/agents`、`~/.claude/commands`、`~/.claude/skills` |
| project MCP | `<ws>/.codemaker/mcps.json`、`<ws>/.mcp.json` |
| project hooks | `<ws>/.codemaker/hooks.json` |
| project 规则 | `<ws>/.codemaker/rules`、`<ws>/AGENTS.md`、`<ws>/CLAUDE.md` |
| project 技能/代理 | `<ws>/.codemaker/skills`、`<ws>/.codemaker/agents` |
| project CC | `<ws>/.claude/settings.json`、`<ws>/.claude/settings.local.json` |
| 其它 | `~/.codemaker/agent-config/hook-trust.json`、`~/.codemaker/agent-config/.agent-lock.json`、`~/.codemaker/plugin-bridge/agent-secrets.json` |

---

## 17. 已知局限

- `index.js` 是压缩产物：**行为可挖，代码不可抄**（标识符已混淆）。
- wasm 是 LFS 指针，Code Map / tree-sitter 在本副本不可用。
- `[Tag]` 前缀覆盖 58 个模块；**无 Tag 的模块**（如 `RecentlyChangedCodeSearch`）在本清单里偏薄，需要针对单点再挖。
- `extension/` 树无源码，所有「上游路径」都是 spec 文档里引用的，不能当本机可读文件。
- 授权：捆绑 Agent 头部声明 `Licensed under the Apache License 2.0`；引用前仍应确认公司内部合规口径。

---

## 18. 移植进度

| # | 项 | 状态 | 位置 |
|---|----|------|------|
| 1 | Rules 多源加载 + `.mdc` frontmatter 契约 | **已落地并验证** | `l_agent_chat/999.0/src/l_agent_chat/rules.py` |
| 1b | 10 个设置开关（6 源开关 + 上限/深度/TTL） | **已落地** | `config.py` 常量与 `_SCHEMA`、`templates/settings.html`「🧭 工作区规则」卡片 |
| 1c | 注入 system 提示词 | **已接入** | `agent_client.py::_build_messages`（`AI_PREFERENCE` 之后） |
| 2 | ignore 治理（mode 头；**远端 department 路由未做**） | **已落地并验证** | `l_agent_tool/999.0/src/l_agent_tool/ignore_rules.py`；挂在 `read_file` / `write_file` / `find_files` / `search_text` |
| 3 | Hooks 引擎（4 类 handler + 18 可阻断事件 + 决策合并 + 双轨校验） | **已落地并验证**；**信任台账未做**（本仓无插件体系） | `l_agent_chat/999.0/src/l_agent_chat/hooks.py`；挂点 `app.py` PreToolUse / PostToolUse |
| 4 | CCSettingsLoader（作为 hooks 来源并入） | **已落地并验证** | 同 `hooks.py` 的 `_claude_settings_paths` / `absorb`；开关 `hooks_sync_cc` |
| 5 | MCP 客户端（配置校验 + stdio/streamableHttp + 自愈） | **已落地并验证**；**sse 传输、心跳、停滞看门狗、resources/prompts 探测未做** | `l_agent_chat/999.0/src/l_agent_chat/mcp.py` |
| 6 | OpenSpec 索引 | **已落地并验证** | `l_agent_chat/999.0/src/l_agent_chat/specs.py`；注入见 `agent_client.py::_build_messages` |
| 7 | shell env 采集 | **已落地并验证** | `l_agent_tool/999.0/src/l_agent_tool/shell_env.py`；挂在 `run_command` |
| 8 | ripgrep 后端 | **已落地并验证** | `l_agent_tool/999.0/src/l_agent_tool/ripgrep.py`；`search_text` 走 rg 优先 + 纯 Python 降级 |

**已验证（2026-09-20，`wuwor l_agent_chat -- python <脚本>`）**：

- `rules.py` 自测通过：frontmatter 四字段、无 frontmatter 全文当正文、缺收尾分隔符报错、`globs` 行内/块数组两种写法
- 真工作区 `trayapp` 命中 2 条：`AGENTS.md`（`agents_md`，常驻，2471 字符）+ `.cursor/rules/rez-package-source-wuwor.mdc`（`cursor`，常驻，`globs=('rez-package-source/**',)`，865 字符）；`prompt_block` 共 3399 字符
- 10 个键全部进入 `_SCHEMA` 与 `get_runtime()`
- 开关生效：关 cursor → 只剩 1 条；关总开关 → 0 条、prompt 0 字符；恢复 → 2 条 / 3399 字符

- **#2 ignore**：造 `denylist`（`secret/` + `*.log`）工作区 → 两个路径 `blocked=True`、`src/main.py` 放行；`read_file`/`write_file` 抛 `PermissionError`（文案带 mode 与 root）；`find_files` 返回 `kept=2 blocked=2`；`search_text` 命中 0（被跳过）。切 `allowlist`（`src/`）→ 只有 src 放行；**空 allowlist → 全拦**；删掉 ignore 文件 → 完全不干预（回归安全）
- **#6 OpenSpec**：真工作区索引 2 个在途变更 + 1 个归档，含 tasks 进度（`171/174`、`55/55`、`19/19`）与 spec 子目录；`prompt_block` 951 字符并已进 system；无 `openspec/` 目录返回 `None`
- **#7 shell env**：父进程 PATH 砍到 4 条（模拟 GUI 陈旧 PATH）→ 合并后 42 条、子进程实测 `entries=42`（多出 38 条注册表条目）、`exit=0`、测后恢复 49 条
- **#8 ripgrep**：本机**没装 rg**，用从本仓 CodeMaker 包里解出的 `rg/win32-x64/rg.exe` 验 rg 分支 —— `engine=ripgrep` 命中正确行；`.codemakerignore` 仍过滤掉被忽略文件；后向断言 `(?<=NEEDLE) here`（Rust regex 不支持）→ rg 退出码 2 → 自动降级 `engine=python` 且结果正确；`AGENT_TOOL_RG=0` 也可强制降级

- **#3 hooks**：12 个用例全过 —— deny 在 `PreToolUse` 生效 / 在 `PostToolUse` 被忽略并记 note；matcher 不中不触发；`additionalContext` + `updatedToolOutput` 正确回收；`hookEventName` 不符丢弃整个 `hookSpecificOutput`；`continue:false` 中断整链；输出非 JSON、`if` 不满足、`disableAllHooks`、http 未过白名单、`mcp_tool` 未接线 —— 全部 fail-open；**stdin 载荷确实送达 handler**（`got:{"tool_name": "run_command", ...}`）
- **#4 CC settings**：项目里写 `syncCcHooksConfigs: true` → `sources=['project','claude_project']` 且 hook 生效；不写则 CC 源被忽略（`sources=[]`）；全局开关 `hooks_sync_cc=1` 可无需项目 flag；`settings.local.json` 同样生效；`allowedHttpHookUrls` 并集正确
- **#5 MCP**：6 个 server 混合配置 —— 未知字段只警告不硬拒、未知传输 `sse` 硬报错、`disabled` 跳过；`tools/list` 与 `tools/call` 打通（`echo:你好 hi`）；stdio 的 stderr 尾巴抓到中文；**坏 server 被逐个隔离**，不拖垮好 server 的工具表；未配置 server / 找不到可执行文件（含 ENOENT 刷 PATH 重试）错误可读；配置改动后缓存正确失效
- **#8 ripgrep**：本机**没装 rg**，用从本仓 CodeMaker 包里解出的 `rg/win32-x64/rg.exe` 验 rg 分支 —— `engine=ripgrep` 命中正确行；`.codemakerignore` 仍过滤掉被忽略文件；后向断言 `(?<=NEEDLE) here`（Rust regex 不支持）→ rg 退出码 2 → 自动降级 `engine=python` 且结果正确；`AGENT_TOOL_RG=0` 也可强制降级

**过程中修掉的 6 个真 bug**（都不是测试写错）：

1. `ignore_rules`：Windows 上待匹配路径已小写、模式却保留原大小写 → 含大写字母的规则永远命中不了。改编译时加 `re.IGNORECASE`
2. `ignore_rules`：只匹配路径本身，导致 `secret/` 这类目录规则拦不住 `secret/key.txt`。改为**对祖先目录也匹配**（gitignore 里命中目录即排除其下一切，而中间层级是否是目录无法逐段判断）
3. `shell_env`：Windows 上 `os.environ.copy()` 的键常是 `Path` 而非 `PATH`，按精确键取值拿到空串会把原 PATH 整个顶掉。改为大小写不敏感定位
4. `hooks`：`shutil.which("bash")` 在 Windows 上命中的是 **`System32\\bash.exe`（WSL 入口）**，hook 被送进 Linux 子系统执行，`python` 不存在 → 退出码 127。改为只认 Git 安装位置的 bash，排除 System32
5. `hooks`：缓存签名只含项目/用户 `hooks.json` 的 mtime，漏了 CC 各层 → 改 `.claude/settings.json` 不生效。签名纳入全部 CC 路径
6. `mcp`：`list_tools` 没做单机失败隔离，一个连不上的 server 会抛穿整张工具表（CodeMaker 是「单项失败只置空该项」）。改为逐个 server 隔离，另外**未知字段从硬拒降为警告** —— 本机真实 `~/.codemaker/mcps.json` 里的 server 带 `autoApprove` 字段，硬拒会把用户现有配置整条打废

**实现取舍（与 CodeMaker 的差异，均已写进各模块的注释）**：

- 分档按 Cursor 语义推断：`alwaysApply: true` / AGENTS.md / CLAUDE.md / codebase.md 走常驻；有 `description` 或 `globs` 走索引；两者皆无则**不注入**（CodeMaker 对无 frontmatter 文件给的默认是 `alwaysApply: false`，本仓选择不静默注入以免噪声）
- 用**简易 YAML 解析**而非 PyYAML：`l_agent_chat` 的 `requires` 里没有 yaml，且 CodeMaker 的 schema 只用到顶层标量 + 数组
- 无文件 watcher，改 **TTL 缓存**（默认 5s）：`_build_messages` 每个 planner 步都会调，不能每步全盘扫
- **ignore 的 root 自解析**：用「从目标路径向上找最近的 `.codemaker/.codemakerignore`」（同 git 找 `.git`），这样 `l_agent_tool` 不必知道调用方的工作区概念（`l_agent_chat` 与 `l_script_editor` 的工作区来源不同）
- **ignore 匹配是 `.gitignore` 子集**（注释 / `!` 反选 / 目录尾斜杠 / 锚定 / `**` / `*` / `?`），不是全量实现；远端 department 路由规则**未做**（那需要 lugwit 后端配合）
- **shell env 改用标准库 `winreg`**：CodeMaker 用 PowerShell 读 Machine + User 两段 PATH，这里直接读注册表键，省一次进程启动；只追加缺失项、不重排，保证当前进程显式设置过的路径仍然优先
- **ripgrep 不塞二进制**：CodeMaker 按平台分发 5 个 rg 二进制（共约 25 MB）；本仓只按 `AGENT_TOOL_RG` / `PATH` 探测，`AGENT_TOOL_RG=0` 可显式关闭，探测不到或正则不兼容（如后向断言）就自动降级纯 Python
- **hooks 的信任台账没做**：`~/.codemaker/agent-config/hook-trust.json` 在本仓没有第二个调用方（无插件体系），照抄等于写死代码。真要启用须先有「不可信来源」的概念
- **hooks 的 `mcp_tool` 型**：依赖 #5，未接线前 fail-open 落 note —— 这正是 CodeMaker 自己的行为（`MCP call entry not wired (fail-open)`）
- **hooks 的 `prompt` 型**：LLM 求值由 `app.py` 通过 `set_prompt_evaluator` 注入（避免 hooks ↔ agent_client 循环导入），且**要求配置里显式写 `model`**，否则跳过。`if` 条件简化为「对 `command` / `path` / `tool_name` 做正则」
- **MCP 的 `sse` 传输跳过**：需要 EventSource 长连与 endpoint 发现，本机没有可验证的 sse server，宁可不写

---

### 18.1 顺带补的一层设置（非 CodeMaker 移植项）

#2 / #7 / #8 落在 `l_agent_tool`，而该包是被复用的库（`l_script_editor` 也在用），不能反向依赖
`l_agent_chat` 的设置注册表。所以新增 `l_agent_tool/settings.py`（最小设置层）：

- 优先级：运行时 `set()` > 环境变量 > `~/.lugwit/l_agent_tool/settings.json` > 默认值
- 5 个键：`ignore_enabled` / `ignore_root` / `shell_env_enabled` / `shell_env_ttl` / `rg_path`
  （环境变量名沿用 `AGENT_TOOL_*`，所以旧的手工设环境变量方式仍然有效）
- `l_agent_chat` 侧用 `config.py` 的 `_EXTERNAL` 映射桥接：`get_runtime()` 负责展示、
  `apply_settings()` 负责转发；**不写本包的 settings.json**，避免两处真源
- 设置页新增「🧰 工具行为」卡片，改完立即生效（`shell_env` 的 TTL 也改成每次调用现取）

验证：`apply_settings({"tool_rg_path": ...})` 后 `ripgrep.rg_path()` 立刻返回该路径；
`shell_env_ttl` 改 7 立即生效；`ignore_enabled=False` 时 `ignore_rules.check()` 返回 `None`、
置回 `True` 后恢复拦截；落盘 JSON 内容正确；未识别键被忽略不报错。

**过程中又踩一个**：`settings._selftest()` 结尾用 `os.environ.pop` 清环境变量，把调用方先设好的
`L_AGENT_TOOL_SETTINGS` 也清掉了，导致测试后续写到**用户真实**的
`~/.lugwit/l_agent_tool/settings.json`。自测应当**还原调用前的值**而不是清空。

---

## 附：本次调研的原始产物（临时目录，可重生成）
| 路径 | 内容 |
|------|------|
| `D:\Temp\Log\_cmagent\` | 解包结果 |
| `D:\Temp\Log\_cminv2.txt` | 按 Tag 归属的完整日志清单 + 关键词字面量 + 构造器窗口 |
| `D:\Temp\Log\_cminv.txt` | logger 普查 + 设置键 + fs 路径 + 工具名扫描 |
| `D:\Temp\Log\_cmprompt.txt` | Doctor Agent system prompt 全文 |
| `D:\Temp\Log\_cmhook.txt` | Hooks / MCP 候选词普查 + 路径字面量 + 传输类型窗口 |
| `D:\Temp\Log\_cmhook2.txt` | hook 输出 schema、事件枚举、CC settings 层、插件清单、配置路径总表 |
| `D:\Temp\Log\_cmhook3.txt` | 四类 handler schema、事件枚举全文、MCP 常量的原始窗口 |
| `D:\Temp\Log\_cmhook4.txt` | 完整 `Nc` 事件枚举、MCP 超时/停滞常量、工具匹配器 |

**这些在临时日志目录，随时可能被清掉**；需要长期留存请按第 0 节方法重新生成，或把它们搬进仓库。
