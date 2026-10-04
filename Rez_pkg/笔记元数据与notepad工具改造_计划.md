# 笔记元数据约定 与 notepad 工具改造（计划）

> 状态：**已实施**（2026-10-04；格式约定 + 读侧 + 写侧 + 权限规则均已落地并验证 —— 完成情况见 §5）。
> 本文定的是**格式约定**（笔记顶部元数据）与两个工具（`notepad_read` 读侧兜底、`notepad_modify` 写侧）的行为。
> 相关：`Rez-Docs/Rez_pkg/知识库归档删除同步与索引清理_计划.md`（删除只显式做、归档只增）。

---

## 1. 已核实的事实（2026-10-04 实测）

| 事实 | 依据 |
|---|---|
| `notepad_read(rel, kb)` 走 `GET /api/kb/{kb}/workspace/file` | `l_agent_chat/notepad_knowledge.py::notepad_read`（:206） |
| 该端点读的是**本机工作区目录**（`_workspace_base(conn,kb)` → `read_text`），**不读云归档** | `l_notepad_server/routers/kb.py::api_kb_workspace_file`（:537） |
| 云归档另有端点 `GET /api/kb/{kb}/depot/file` | `kb.py:291` |
| 搜索索引读的是**归档** | `search_index.py` → `depot_map.list_tree`（已改 `live_only=True`） |
| 归档里的"删除"是**状态**（`action=delete`），blob 与历史版本保留、可 revert | `depot_map.is_deleted_state()` / `/api/depot/delete` |

**后果**：本机删掉文件 → 搜索还命中（读归档）、`notepad_read` 404（读本机工作区）。
工具层表现为「**搜得到、读不到**」的不一致。

## 2. 读侧：`notepad_read` 增加归档兜底（推荐）

**目标**：本机工作区没有该文件时，**不要直接 404**，而是退回读**归档**那一份，
并在返回里**标明来源是归档**（而不是本机），这样：

- 内容在版本库里还在 → 「删除不会有啥印象」成立；
- 调用方（智能体/人）能一眼看出"这是归档版、本机已无"，不会误以为本机还有这个文件。

**行为**：`rel` 先查本机工作区 → 命中即返回（`source="workspace"`）；未命中 → 查归档
（`/api/kb/{kb}/depot/file`）→ 命中返回（`source="archive"`，可再带 `rev`）；都没有 → 404。

**注意**：归档里**处于删除状态**的文件**不兜底**（否则又变成"读得到却已删"的怪状态）；
判断用 `depot_map.is_deleted_state()`。

## 3. 笔记顶部元数据：固有格式（本文的核心约定）

### 3.1 位置与形态

**必须是文件最开头的第一个块**，用一个 HTML 注释包裹（渲染时不显示、不干扰 Markdown）：

```md
<!-- lugwit-note
title: 知识库搜索接口
created: 2026-09-20
updated: 2026-10-04
updated_by: admin01
note: 合并了四篇搜索改造文档；接口默认 mode 改为 auto
tags: l_notepad,搜索
-->
```

### 3.2 字段（**这一版只定这几个，别自作主张加**）

| 字段 | 必填 | 含义 |
|---|---|---|
| `title` | 否 | 笔记标题（缺省用文件名） |
| `created` | 否 | 创建日期 `YYYY-MM-DD` |
| `updated` | **是** | 最后一次修改日期 |
| `updated_by` | **是** | 最后一次修改人（lugwit 用户名） |
| `note` | **是** | **本次修改备注**（一句话：改了什么、为什么） |
| `tags` | 否 | 逗号分隔 |

可选历史（看到"改过几次、最近都为什么"）：

```md
<!-- lugwit-note-history
2026-10-04 admin01 合并四篇搜索文档、接口默认 mode 改 auto
2026-09-28 fqq     重排默认关；订正 DB 体积结论
-->
```
（最多保留最近 **5** 条，倒序，新的在上。**只增不删**旧条目。）

### 3.3 三条硬规则

1. **元数据不是正文**：`notepad_read` 必须把它**剥离**出来，单独放在返回体的 `meta` 字段里，
   **不要**混进 `content`（否则每份文档都白白多占上下文，且引用行号会整体偏移）。
2. **不认识就原样保留**：写入方遇到未知字段（别人加的新字段）**必须保留**，不得丢弃。
3. **行号口径**：`meta` 之外的正文行号**从元数据块之后重新起算**（这样 `chunk_line` /
   `notepad_read(offset=…)` 与"我在编辑器里看到的第 N 行"要说明口径 —— 约定
   **统一按去掉元数据块后的行号**，并在返回里给 `meta_lines` 让调用方能换算）。

### 3.4 为什么要有它（与删除的关系）

**删除前先读修改备注**：删一个笔记/文档前，`notepad_read` 返回的 `meta.note` +
`lugwit-note-history` 直接告诉决策者「**这是谁、什么时候、为什么改的**」——
这正是判断"能不能删"的依据。配合"归档删除只显式做"（见另一篇计划），
删除动作就有了**可追溯的现场记录**，而不是一个孤零零的消失。

## 4. 写侧：新增 `notepad_modify`

### 4.1 接口（建议）

```
notepad_modify(rel, kb="", *, content=None, note="", patch=None, expect_rev=None,
               target="auto") -> dict          # target: auto | workspace | archive（见 §4.5）
```

- `content` 与 `patch` 二选一；**两者都给 → 报错**（避免歧义）。
- `note`（必填）：本次修改备注 → 写入 `meta.note`，并往 `lugwit-note-history` 顶部追加一条
  `日期 用户 备注`（超 5 条截断尾部）。
- `expect_rev`（可选）：期望的版本号/内容哈希，不一致就**拒绝**（并发保护，避免覆盖别人的改动）。

### 4.2 行为

1. 读现有内容 → 解析 `meta`（不认识就按"只有正文"处理）；
2. 应用 `content`/`patch`；
3. **重写元数据**：`updated`=今天、`updated_by`=当前用户、`note`=`note` 参数、历史追加上一条；
4. 写回本机工作区（**只写工作区**；上传仍由实时同步器按 20s/10s 节奏做）；
5. 返回新的 `meta` + 新 `rev`/哈希。

### 4.3 护栏

- **必须带 `note`**：没有备注的修改 → 拒绝（这是"修改备注"能长期存在的前提）。
- 只允许改**文本类型**（沿用 `WORKSPACE_EXTS`）。
- 路径必须落在工作区内（复用 `_safe_ws_path`），**禁止**越界。
- 并发：`expect_rev` 不匹配 → 拒绝并回传当前 `rev`。

### 4.4 与删除的关系

`notepad_modify` **不做删除**。删除是另一个显式动作（见另一篇计划 §3），
且**应当先 `notepad_read` 拿 `meta.note` + 历史**再决定 —— 这条要写进删除工具/界面的提示里。

### 4.5 `target`：写工作区还是写归档（2026-10-04 实测定型）

| `target` | 行为 |
|---|---|
| **`auto`（默认）** | **本机工作区里有这篇** → 写工作区；**没有** → **直写归档** —— 与 `notepad_read` 的兜底口径一致（判的是"有没有这个文件"，**不是**"有没有配工作区"） |
| `workspace` | 强制写工作区；本机未配工作区（服务端 400）→ 明确报错 |
| `archive` | 强制直写归档（`POST /api/kb/{kb}/depot/submit`，**body 是原始文本**；服务端写完 `notify_kb_change` **立即重索引**） |

- **`expect_rev` 只在 `target="archive"` 时校验**：rev 是归档的概念，写工作区时归档 rev 本就可能落后（本地在前）→ 校验会误拒。
- **写归档且本机还有工作区副本时：允许 + 返回体 `warning` 强警告**（同步是「工作区 → 归档」单向，下一轮可能把本地那份传上去覆盖）。
- 直写归档时把提交 `description` 设为 `"<用户>: <修改备注>"`（同步历史里能看出谁、为什么）。

### 4.6 权限门（2026-10-04 已加）

- **模式裁剪本就有**：`chat_modes.py` 的 `READ_TOOLS` 只读白名单 —— `ask`/`plan`/`review` 模式下**写工具不可用**。
- **规则引擎本没有覆盖它**：`permissions.py` 内置规则只覆盖 受保护路径 / `mcp:*` / `task` / `execute` → `notepad_modify` 无命中 → 走旧兜底（`AUTO_APPROVE=1` 默认自动批准）。
- **已加**：内置规则 `{"action": "notepad_modify", "effect": "ask"}` → **default 模式下执行前需确认**。
  实测：`evaluate()` → `ask`；`effective(..., "default")` → `ask`；`"allow_all"` → `allow`（引擎设计如此）；`notepad_read` 仍 `none`（没误伤）。
  （**没**用 `APPROVAL_TOOLS`：那条路要同时关掉全局 `AUTO_APPROVE`，影响所有工具。）
- **已知残余**：引擎的 `resource` 维度是"文件路径 / 命令 / URL"，**不认工具参数** → 无法只对 `target="archive"`（直写共享版本库）加严。要做需给引擎加参数级匹配。

## 5. 实施情况（2026-10-04）

| # | 项 | 状态 |
|---|---|---|
| 1 | 格式约定 + 读侧返回 `meta`（剥离、不混正文） | ✅ 已实施（探针验过：有元数据/无元数据/只有 note/块未闭合 四种边界） |
| 2 | 读侧归档兜底（标 `source`） | ✅ 已实施，端到端验过（`SOURCE archive`，读到已删文档全文） |
| 3 | `notepad_modify`（含 §4.5 的 `target`） | ✅ 已实施（**写路径未做真实端到端**：会写真实工作区/共享归档，只验到"写入前中止"的分支） |
| 4 | 权限门（§4.6） | ✅ 已加 `ask` 规则并实测 |
| 5 | 显式删除入口（kb 路由补 delete） | ⬜ 未做（当前用 depot 的 `/api/depot/delete` 手工做，见另一篇计划） |
| 6 | 索引清旧行 | ✅ 已实施（2026-10-04 收尾：**词法侧早就在跑**，漏的是**向量侧** —— `_drop` 只删 `search_docs`/`search_fts`，`vec_docs`/`vec_chunks` 要等下一次嵌入才消失。已改成 `_drop` 两侧同删，并补 `purge_orphan_index` 兜底。见另一篇计划 §2.4） |


## 6. 不要做的事

- ❌ 不要把元数据当普通正文返回（浪费上下文、行号漂移）。
- ❌ 不要在读取时**静默改写**元数据（只有 `notepad_modify` 能写）。
- ❌ 不要让 `notepad_read` 兜底返回**删除状态**的归档文件。
- ❌ 不要为"省事"在元数据里塞大块内容（它是元数据，不是存档区）。
