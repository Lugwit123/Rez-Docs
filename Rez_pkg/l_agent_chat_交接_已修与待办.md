# l_agent_chat 交接：已修 / 已验 / 待办（2026-09-29）

> 给接手者的文档。**先读第 1 节的三条陷阱** —— 今天我在这个工程上犯了三次同类错，
> 不读那节，你很可能把已经修好的东西再"修"一遍，或者把现象归因到错的层。
>
> 涉及包：`l_agent_chat`（对话 agent，:1250）、`l_notepad_server`（知识库，:8765）、
> `l_agent_tool`（工具库）、`l_repo_sync_gui`（直推部署通道）。
> 全部路径在 `D:\TD_Depot\Software\Lugwit_syncPlug\lugwit_insapp\trayapp\rez-package-source\`。

---

## 0. 目标与验收标准（先看这节）

### 0.1 目标

**让 `l_agent_chat` 在真实任务上"说到做到"——它承诺的是「查清原因并修掉」，
验收就按这个承诺收，不按"给出了合理分析"收。**

今天暴露的本质：agent 表面上像个"不听话的模型"，实际是 **harness 在替它做决定**
（工具一失败就终止整轮、收尾门挂在单个出口上、信号传不到模型）。目标是把这些
**结构性坑**堵掉，让模型的努力能真正落到交付上。

一句话：**目标是「任务闭环」，不是「现象消失」。**

### 0.2 验收标准（每条都可复现、且**不能自证**）

| # | 目标 | 验收标准（可复现的判据） | 判据来源 |
|---|---|---|---|
| **G1** | **窄屏下消息卡片可正常滚动**（原始需求） | Playwright `375×667` 打开 :1250 → 找一条超过屏高 2/3 的长消息 → 在卡片上滑到边界后继续上滑 → **断言外层会话容器（`.au-viewport`）确实跟滚**；再点卡片确认「展开/收起」仍正常 | 断言结果 + 截图 |
| **G2** | agent 不再**提前终止**本轮 | 构造一次可恢复的工具失败（如给 `read_file` 一个不存在的路径）→ 事件流 `stop": true` **为 0**，且**该轮之后仍有新的 `tool_start`**（证明它换手段继续了） | `events/<session>.jsonl` |
| **G3** | **收尾门**三条路径都通 | 同一轮里：①它想收尾 → `收尾前请用户确认` ≥ 1；②点「继续做」→ `ask_required` +1 **且 `tool_start` 增加**（证明真接着干，不是原地再问）；③点「可以结束」→ `done": 1` | 同上（今天已验，需回归） |
| **G4** | 失败**有上限**、不会无限重试 | 造连续失败（≥3 次同一操作失败）→ 到上限后**主动收尾并说明**，而不是继续空转 | 同上 |
| **G5** | **等人回答时断线**不留僵尸会话 | 发起提问 → agent 调到 `ask_user` → **强行关掉浏览器** → 重新打开该会话：状态应为**「已中断」**（不是永远"运行中"、不是空白） | 会话界面 + 事件流 |
| **G6** | 远端与本地**代码一致**且服务在线 | `missing_files()` 对四包（`l_agent_chat` / `l_notepad_server` / `l_agent_tool` / `l_agent_market`）返回**缺 0 / 不一致 0**；远端 8765 / 1250 均 HTTP 200 | `deploy_verify` 判据 |
| **G7** | 假错误为 0 | 正常流程（含门控触发）事件流里 `"type": "error"` **为 0** | 事件流 |
| **G8** | 文档无欠账 | `l_agent_chat` 的 `doc/CHANGELOG.md` 补一条本次改动（`l_notepad_server` 的 v3.5.0~v3.5.5 已覆盖 §3.2，不重复补）；本文件末尾「未验」项清零或降级为「已知取舍」 | 文件存在 + 内容 |

### 0.3 完成定义（DoD）

**全部 G1~G8 都有实测判据即可交付**；任一条只有"看起来好了"没有判据，
**不算完成** —— 今天三次误判都是这么来的（见 §1 陷阱 1）。

判据必须**成对且不可自证**：例如 G2 要 `stop": true` 归零 **且** 后续有新的 `tool_start`；
G3 要 `ask_required` 增加 **且** `tool_start` 增加。**单看一个数一定会被骗。**

---

## 1. 三条陷阱（今天踩过，代价最大的一节）

### 陷阱 1：拿**单个指标**当"修好了"

今天三次「以为修好」，全是同一个错：

| 我当时的判据 | 真相 |
|---|---|
| 界面不再报错 = 修好了 | 实际是我删了那条 error 事件，**门还是没生效** |
| `done=1` = 会话正常收尾 | 实际是**被一次工具超时掐断**，回答不完整 |
| `远端缺 0` = 部署完整 | 那只是**存在性**，二进制字节坏的它看不出来 |

**正确做法：判据必须成对，且都不能自证。** 例：收尾门是否生效，要看
`stop": true` 归零 **且** `收尾前请用户确认` 非零 —— 两个数同时成立才算通。

### 陷阱 2：把 harness 的行为归因成**模型的性格**

我在第 2~7 轮反复看到「agent 失败一次就写报告收尾」，据此写了长篇分析说这是
「模型不重试」的习惯问题，还提了两轮**提示词修法**（插催促、堵"只输出结论文本"）。
**那些修法不可能生效** —— 因为真相是：

```python
# app.py（改前）：工具抛异常 / 编辑预检失败 → 直接终止整轮
agent_client._append_tool_turn(plan_messages, step_out, call, failed)  # 已经把失败喂给模型
stop_loop = True        # ← 然后立刻把门关了
break
```

**是 harness 主动终止，模型根本没机会重试。** 注释自己写着「可换别的工具」，代码下一行就把它掐了。
→ 教训：**看到"模型做了坏事"，先去循环控制里找有没有人在替它做决定。**

### 陷阱 3：跨版本线用「git 差集」推送 = 制造线上事故

今天真的把远端知识库推挂了（详见 §3.3）。**这条通道是给"同一条线的增量同步"设计的，
不是通用发布工具。** 跨版本线必须走**全量直传**（本地清单驱动）。

---

## 2. 一句话现状

agent 主循环的硬伤（**提前终止 / 收尾门被绕过 / 假错误 / 连败空转 / 断线僵尸**）今天已修并实测通过；
知识库检索的排序与默认档已调过；部署通道补了完整性校验与分批优化。
**原始需求「手机窄屏下消息卡片滑不动」也已闭环**（真因是 `overscroll-behavior:none`，见 §5.1）。
本轮收尾（2026-09-29）额外完成：连续失败计数器、断线中断标记验证、远端四包一致性 +
收尾门远端复验 + `skip_unchanged` 大包实测、`l_agent_chat` 补 CHANGELOG（§5.1~§5.5 均已办结）。

---

## 3. 已修（附实测判据）

### 3.1 agent 主循环（`l_agent_chat/999.0/src/l_agent_chat/app.py`）

| 改动 | 位置 | 判据（实测） |
|---|---|---|
| **工具失败不再终止整轮** | 去掉 3 处 `stop_loop = True` + `break`（工具抛异常 / 两条编辑预检失败） | 成对判据（`ac_g237_out.txt`）：`stop": true` **= 0** **且** 失败后**仍有非 ask_user 的新 `tool_start`**（实测：`read_file` 失败 1 次 → 下一步 step=1 继续 `read_file`） |
| **收尾确认门**（动手的回合收尾前必须调 `ask_user` 问用户） | `if not calls:` 里的收尾点 + `ask_user` 回答处（答"继续"则 `confirm_asked=False` 重新上膛） | 三条路径全验（`ac_g237_out.txt`）：①拦住 → `收尾前请用户确认` 标记 **= 1**；②答"继续做" → `ask_required` **= 3** **且** 门控后仍有真实 `tool_start`（门控前/后 = 2/1，证明确实接着干）；③答"可以结束" → `done": 1`（回答 1030 字） |
| **收尾门不再发 `type: error`** | 同上 | `error` 由 1 → **0**（原是我发的信息性提示，界面显示成红色错误） |
| **连续失败计数器**（§5.2） | 去掉「失败就终止」后补上限：`MAX_CONSECUTIVE_FAILURES`（默认 3）；`_note_tool_result(ok, tool)` 连续计数、成功清零；到上限置 `fail_limit_hit` 走收尾通道（**不置 `stop_loop`、不发 `type:error`**） | 造连续失败：失败 `tool_result`=3 / 最长连败段=3；收尾 status=`连续 3 次工具调用失败，本轮收敛收尾并说明`；`stop": true`=0；`error`=0；`done`=1（回答 1298 字） |
| **等回答时断开 → 标记中断** | `ask_user` 的 `await` 加 `except asyncio.CancelledError:` → `interrupt_flag.set()` + `run_box["interrupted"] = True` + `_mark(...)` 后 `raise` | ✅ 已验（两种断法各 8/8 PASS）：`active` 归 `None`、`last.running=False`、`last.interrupted=True`、落盘末条 `meta.aborted=True` 且正文含「连接中断」、`steps` 含「客户端断开」、`error`=0。<br>· FIN 断（`ac_g5_check.py`，输出 `ac_g5_out.txt`）<br>· **真 RST 强断**（`ac_g5_rst.py`，裸 socket + `SO_LINGER(1,0)`，输出 `ac_g5_rst_out.txt`；`ac_g5_check.py` 的 urllib3 路径在本地版本取不到底层 socket，会退化成 `close()`=FIN，故另建裸 socket 脚本保证真 RST） |
| **内联 XML 兜底认 DSML 方言** | 新增 `_normalize_dsml()`，入口归一化；`_INLINE_TAGS` 补全角竖线变体 | 单测 13/13（样本取自真实日志的 `done.reply`） |
| **`ask_user` 应有提示词约束** | `_PLANNER_SYSTEM` 加 `ASK, DON'T END` 段 | 与门控叠加后生效 |
| **`notepad_read` 404 兜底** | 代码源命中（`kb_name` 是 rez 包名）时直接当文件读；读不到给可执行错误 | 单测 8/8 |
| **`notepad_search` 透传质量信号** | 返回体加 `confidence` / `chunk_line` / `degraded` / `unindexed_packages` | 第三轮实测：模型自己在结论里写了「本次检索 degraded（top-1 置信度 0.44）」 |

### 3.2 知识库检索（`l_notepad_server`）

- 默认档 `hybrid` → `auto`（`routers/search.py`、`routers/kb.py`）：**不传 mode 138ms，原 1365ms**
- `auto` 回退判据 40/1.3 → **25/1.0**：回退率 65% → 11%，`p50 1397ms → 230ms`，`recall` 未丢
- `search_text` 提速：`l_agent_tool` 内置 `bin/rg.exe`（原来 PATH 无 rg → 走纯 Python 兜底，**一次几分钟**）；兜底路径改 `os.walk` 目录级剪枝 + 2MB 上限
- 重排默认关（`rerank_enabled` 默认 False）、导航块降权、块长度归一、RRF 融合、`confidence` 查询内校准
- **文档**：`Rez-Docs/Rez_pkg/l_notepad_知识库检索优化_计划.md`、`..._二期方案.md`（§11 性能校准）

### 3.3 直推部署通道（`l_repo_sync_gui`）

**事故**：按 git 差集推 → 推了 `routers/kb.py`（新 `from .. import workspace_sync`）、
没推 `workspace_sync.py`（已 commit，不在差集里）→ 远端 `ImportError` → **线上 8765 宕机**。

已补（`deploy_verify.py` 新增 + `main._push_deploy_run` 接线）：

- `missing_files()`：拿本地清单问远端，把「本地有、远端没有」**并入本次上传批**
- 跳过条件加严：原「git 无差异就跳过」→「**无差异且远端不缺**才跳过」
- `DEFAULT_MAX_BODY` 60 KB → 2 MB（往返 90 → 9）；**批失败改为对半拆重试**（原整批静默丢弃）
- `skip_unchanged` **显式参数**（默认关）：先 `hash_files` 探测，内容一致的跳过（实测 76 文件 0.2s / 0 写入，见 `g6_push_align_out.txt`）
- 详细记录见 `l_repo_sync_gui/999.0/README.md` 末尾「直推部署通道：完整性加固」一节

---

## 4. 已验 / 未验（这条线必须分清楚）

| 项 | 状态 |
|---|---|
| 收尾门三条路径（拦住 / 放行 / 打回） | ✅ 本地实测（事件流判据见 §6） |
| 工具失败不终止整轮 | ✅ 成对判据：`stop": true = 0` **且** 失败后仍有新 `tool_start`（`ac_g237_out.txt`） |
| DSML 方言 / `notepad_read` 兜底 / `rg` 内置 | ✅ 单测 |
| **连续失败计数器** | ✅ 造 ≥3 次连续失败 → 主动收尾并说明；`stop": true`=0、`error`=0（见 §3.1） |
| 远端四包代码一致性 | ✅ 缺失 0 / 不一致 0（sha256 **逐文件**比对，`deploy_verify` 判据；`l_agent_chat` 151 / `l_notepad_server` 76 / `l_agent_tool` 20 / `l_agent_market` 13 全 `bad=0 missing=0`；远端 8765 / 1250 均 HTTP 200，见 `g6_final_out.txt`） |
| 远端 `bin/rg.exe` 字节完整性 | ✅ sha256 逐位一致 + `--version` rc=0 |
| **断线中断标记（CancelledError 分支）** | ✅ 本地实测：`active=None` / `last.running=False` / `meta.aborted=True` / `last.interrupted=True` / `error=0`；FIN 断与**真 RST 强断**（`ac_g5_rst.py`，`SO_LINGER(1,0)`）各 8/8 PASS（`ac_g5_out.txt` / `ac_g5_rst_out.txt`） |
| **收尾门在远端** | ✅ 远端复验 PASS：`{"session":"20260928_234035855","turn_len":116,"turn_types":{"agent":1,"status":20,"step_start":4,"tool_start":5,"tool_result":5,"file":1,"step_finish":4,"reasoning":31,"ask_required":2,"delta":42,"done":1},"gate_text":2,"ask_required":2,"done":1,"error":0,"stop_true":0,"dsml":0}`（`g6_remote_gate_turn_out.txt`，按「最后一个 agent 事件起」切片统计） |
| **`skip_unchanged` 在大包（`l_agent_chat`，151 文件）上** | ✅ 实测：探测 151 / 跳过 150 / 上传 1 / 0.4s（`g6_push_align_out.txt`；四包合计 `written=1 skipped=259 errors=0`；`l_notepad_server` 76 文件 0.2s 亦验） |
| **MCP 真浏览器端到端实测**（本机 :1250，与脚本化 SSE 测试互为交叉验证） | ✅ 真 Chrome（MCP Playwright，`headless=false`）跑完整一轮：会话 `20260929_140455702`，下发「①读不存在文件 ②读 `ask_q.txt` 贴前 3 行 ③收尾前先问我确认」→ 规划 1/80 步并行 `read_file`×2（1 失 1 成）→ 收尾门 #1（3 选项）→ 点「先查一下…是否在别处/路径写错」→ **打回后真接着干**（新 `run_command`×2 失败 → 改 `execute`）→ `execute` 全盘 glob 240s 超时 → 缩浅层重扫 1.21s 成功 → 收尾门 #2（自检清单 + 2 选项）→ 点「可以结束」→ `done`。**成对判据**（首回合 245 事件）：`step_start/finish` 6/6、`tool_start/result` 8/8、`ask_required/作答` 2/2、`done` 1、`"stop": true` **0**、`"type":"error"` **0**、DSML **0**（`ac_browser_e2e_out.txt` §一~§三；9 张截图 `C:\Users\wb.fengqingqing\Downloads\agent_run_*.png`） |
| **`run_command` 坏点与修复复验**（端到端实测中暴露） | ✅ 真故障：:1250 进程 PID 77160 启动于 `13:21:32`，而 `settings.py`（新增 SCHEMA 键 `command_cwd`）落盘于 `13:21:40`（**晚 8 秒**）→ 进程内是旧 SCHEMA → `settings.get("command_cwd")` 抛 `KeyError` → `_resolve_command_cwd` 崩 → `run_command` 全失败（seq30/31）。`wuwo svc restart l_agent_chat`（**18.3s 就绪**，新 PID 112912 @ `14:26:00`）后真浏览器复验：`(Get-Date).ToString("o")` → `state=completed, duration=0.36s, blocked=false, exit_code=0, cwd="D:\...\trayapp"` ✅（`ac_browser_e2e_out.txt` §五） |
| **`_DANGEROUS_PATTERNS` 误伤（已修）** | ✅ **已修**（2026-09-29）：`l_agent_tool/agent_tools.py` 的 `_DANGEROUS_PATTERNS` 由**子串匹配**改为**正则 + 词边界/盘符约束**（预编译 `_DANGEROUS_RE`，`re.IGNORECASE`）；`format ` → `(?<![\w-])format\s+[a-z]:`（要求盘符；`-Format o`、`Format-Table` 不再命中）。**成对判据**：①单测 20/20 PASS —— 安全 8 条（`Get-Date -Format o` / `Format-Table` / `Format-List` …）全 `blocked=False`，危险 12 条（含 `format C:` / `FORMAT D: /q`）全 `blocked=True`（`ac_fmt_out.txt`）；②`wuwo svc restart l_agent_chat`（**15.4s 就绪**，红线⑤）后 **HTTP/SSE 端到端复验**：`Get-Date -Format o` → `{"blocked": false, "exit_code": 0, "stdout": "2026-09-29T15:21:31.98+08:00"}`、事件流 `error=0`（`ac_fmt_e2e_out.txt`）。注：本次 MCP 真浏览器被外部进程抢占（页面反复跳到 `:1234/video-editor`），故端到端改走脚本化 HTTP/SSE，与上一行「MCP 真浏览器」互为交叉验证 |
| **工具卡显示代码改动 diff（`write_file` 补齐）** | ✅ **已补**（2026-09-29）：链路本就通 —— `edit_file` / `apply_patch` 的结果自带 `diff` → `app.py::_result_diffs` → `tool_result.diffs` → 前端 `ToolCard` → `DiffBlock`（逐行 +/- 着色、`+N -M`、折叠、按扩展名高亮）。缺口在 `write_file`（`l_agent_tool`）只返回 `{path, written, chars}`，而 `AUTO_APPROVE=1`（默认）下不走审批预览 → 卡片只剩「已写入 N 字符」。现于**写盘前**（与 checkpoint 同一时机）用 `_diff_preview` 算一份挂到 `pending_diff`，复用既有回挂逻辑进 `tool_result.diffs`；另修审批预览分支对**相对路径**直接 `Path(path)` 读盘（读到进程 cwd → old 为空 → diff 全变「新增」）改走 `agent_tools.resolve_workspace_path`。**成对判据**：①`edit_file` 路径 HTTP/SSE 端到端 → `ok=True diffs=1`、diff 含 `@@`（`ac_diff_e2e_out.txt`）；②`write_file` 路径 `wuwo svc restart l_agent_chat`（**18.1s 就绪**，红线⑤）后 → `diffs=1`、diff 含 `-line2` **且** `+line2-changed`（确证读到旧内容、非整文件当新增）、事件流 `error=0`（`ac_diff_write_e2e_out.txt`）。前端零改动。**G6**：全量直传 `written=2 skipped=258 errors=0`（仅 `src/l_agent_chat/app.py` + `doc/CHANGELOG.md`；`g6_push_align_out.txt`），四包 `bad=0 missing=0`、远端 1250/8765 HTTP 200（`g6_final_out.txt`）；远端 `wuwo svc restart l_agent_chat` RC=0，**独立证据**：监听 1250 的 PID 17416 启动于 `2026-09-29T15:43:20+08:00`、探活时 age 26.5s（`g6_remote_pid_out.txt`）。 |
| **远端 33 篇语料重建索引** | ⏸️ **已知取舍**：代码对齐 ≠ 数据对齐；重建属动远端数据，按 §7-3 先拍板再做 |

---

## 5. 待办（✅ 全部办结 · 2026-09-29 收尾；原文保留以备对照）

### 5.1 手机窄屏下「消息卡片滑不动」—— ✅ 已闭环

> **办结（2026-09-29）**：真因 = `.bubble` 上叠的 `overscroll-behavior: none`
> 在不支持 `overflow: clip` 的引擎上退化成**内层滚动容器**，把手势掐断在气泡里、不往外传
> → 外层页面纹丝不动。已删掉两处 `overscroll-behavior: none`（只留 `overflow: hidden` / `clip`），
> 并在 `style.css` 就地留下「不要再叠 `overscroll-behavior:none` 当双保险」的踩坑注释。
> 375×667 实测（`ac_g1_check.mjs`，CDP `Input.dispatchTouchEvent` 派发滑动手势，**10/10 PASS**）：
> 长卡片限高生效（`clientH=438 / scrollH=1261`）；气泡不再是内层可滚容器（`overflow-y=clip`、
> `overscroll-behavior=auto`）；在卡片上滑到边界后继续上滑，外层 `.au-viewport` 跟滚
> （`300 → 494 → 600 → 600`）而气泡自身 `scrollTop` 纹丝不动（`0 → 0 → 0 → 0`）；点击展开/收起正常；
> **含负对照**：退化引擎 + 回填 `overscroll-behavior:none` → 卡死 `300 → 300 → 300`（证明断言确有分辨力）。
> 输出 `ac_g1_out.txt`（旧脚本 `ac_scroll_fix_probe.mjs` 仅作探测）。

症状：窄屏（375px）下长消息卡片滑到滑不动之后，**整页也跟着滑不动**，手势像被卡片吃掉。

**已知**：
- 第 4 轮 agent 用 `apply_patch` 改过 `l_agent_chat/999.0/web/src/style.css`（+14 行），
  **那次改动从未验证过**，而 `dist` 在这之后重建过 → **线上产物里可能已带着一个未验证的改动**
- 第 7 轮它用真浏览器（窄屏 + 派发事件）复现并**定位到** `.bubble` 的
  `overscroll-behavior: none` 叠加 `overflow: clip` 在退化引擎下变 `hidden`（`style.css:1157/1251/1288`）
- 但那次 `apply_patch` **预检失败**（`old_string 找不到`），**没落地**

**接手第一步应该是**：先确认 `style.css` 当前那 14 行改的是什么、要不要留；
再按第 7 轮的定位（`overscroll-behavior` → `contain`/`auto`、`overflow: clip` 的降级回退）
做最小修复；然后**用 Playwright 在 375×667 下实测**（滑到边界后继续上滑，断言外层 `.au-viewport` 是否跟滚）。

### 5.2 「连续 N 次失败才收尾」的计数器 —— ✅ 已办

> **办结（2026-09-29）**：`config.py` 新增 `MAX_CONSECUTIVE_FAILURES`（默认 3）；
> `app.py` 加 `_note_tool_result(ok, tool)` 连续计数（成功清零），到上限置 `fail_limit_hit`
> 走收尾通道。实测见 §3.1 / §4。

今天去掉了「失败就终止」，但**没加上限**。当前是「失败不再终止、也不设上限」，
理论上存在反复失败的场景。`app.py` 3976 附近的注释里写明了设计意图（连续计数，不是一次就收）。

### 5.3 断线中断标记的验证（§4 第一行）—— ✅ 已办

> **办结（2026-09-29）**：用 `SO_LINGER(1,0)` 发 RST 强断（`ac_g5_check.py`）验证通过：
> 会话不再停在未完成态；`active` 归 `None`、`last.running=False`、`last.interrupted=True`、
> 落盘末条 `meta.aborted=True` 且正文含「连接中断」、`error=0`。
> 关键点：必须**同时**置 `run_box["interrupted"]`（正常收尾那行在 `raise` 之后跑不到）。

需要构造：发起提问 → agent 调到 `ask_user` → **强行断掉浏览器** → 看会话是否停在未完成态。
如果仍停在未完成态，说明 `raise` 之后的收尾/落盘**没被执行**（生成器被取消时不会往下走），
那就得在**生成器外层**包一层取消处理。

### 5.4 收尾门在远端复验 + `skip_unchanged` 在大包上实测 —— ✅ 已办

> **办结（2026-09-29）**：先把本机四包**全量直传**远端并 `wuwo svc restart` 重启，
> 再在远端复验收尾门 —— 判据（按「最后一个 agent 事件起」切片，`g6_remote_gate_turn_out.txt`）
> `{"session":"20260928_234035855","turn_len":116,"gate_text":2,"ask_required":2,"done":1,"error":0,"stop_true":0,"dsml":0}`
> （PASS）；`skip_unchanged` 在 `l_agent_chat`（151 文件）实测（`g6_push_align_out.txt`）：
> 探测 151 / 跳过 150 / 上传 1 / 0.4s，四包合计 `written=1 skipped=259 errors=0`。
> 复验另含远端四包 sha256 逐文件一致（缺 0 / 不一致 0）、远端 1250 / 8765 HTTP 200（`g6_final_out.txt`）。

### 5.5 本次改动未写 CHANGELOG —— ✅ 已办

> **办结（2026-09-29）**：新建 `l_agent_chat/999.0/src/l_agent_chat/doc/CHANGELOG.md`
> （该包**首个** CHANGELOG），条目 `## v1.0.0 (2026-09-29) — 主循环闭环加固…` 逐条覆盖今天这批改动
> （失败不终止 / 连败上限 / 收尾门不发 error / `except` 的 `result = failed` 修复 / 断线中断标记 /
> 窄屏滑动修复 / 技能·插件市场 / `run_command` 默认 cwd / 工作区刷新 / `MarketApiTest`）。
> **`l_notepad_server` 不重复补** —— 其 `doc/CHANGELOG.md` 的 v3.5.0~v3.5.5 已逐条覆盖 §3.2 的检索改动
> （默认档改 `auto`、判据 25/1.0、RRF、导航降权、块长归一、`confidence`、`rerank` 默认关等）。

只有 `l_repo_sync_gui/999.0/README.md` 末尾记了「直推部署通道」那一节；
`l_agent_chat` 与 `l_notepad_server` 的 `doc/CHANGELOG.md` 该各补一条（今天这批改动量不小）。

---

## 6. 复现工具与判据（都在 `d:\Temp\`）

| 脚本 | 用途 |
|---|---|
| `ac_ask.py` | 驱动浏览器问 agent。**读 `d:\Temp\ask_q.txt`（UTF-8，一行一条）**；会自动点收尾门的选项（第 1 次「继续做」、第 2 次「可以结束」） |
| `ev_answer.py` | 从事件流取 agent 的最终回答（`done.reply`）+ 工具调用清单 |
| `ev_forensic.py` | 事件流取证：类型分布 / 失败结果 / DSML 残留 / 工具时间线 |
| `test_dsml.py` / `test_notepad_read.py` | 单测（样本取自真实日志，**别用手打的近似样本**） |
| `deploy_verify` 自测 | 四包远端完整性（`missing_files`） |
| `remote_up.py` / `remote_fix.py` | 远端服务拉起 / 补推缺失文件（走 `DeployChannel`） |
| `ac_g1_check.mjs` / `ac_scroll_fix_probe.mjs` | G1 窄屏滑动（375×667 CDP 派发滚动手势，断言 `.au-viewport` 跟滚；含退化引擎**负对照**。输出 `ac_g1_out.txt`） |
| `ac_g237_check.py` | G2 / G3 / G7 回归（`stop": true`、收尾门三路径、`error`；输出 `ac_g237_out.txt`） |
| `ac_g4_check.py` | G4 连续失败上限（输出样本 `ac_g4_out.txt`） |
| `ac_g5_check.py` / `ac_g5_shot.mjs` | G5 断线中断（FIN 断 + 截图；输出 `ac_g5_out.txt`） |
| `ac_g5_rst.py` | G5 **真 RST 强断**（裸 socket + `SO_LINGER(1,0)`，`ac_g5_check.py` 取不到底层 socket 时的补充；输出 `ac_g5_rst_out.txt`） |
| `test_skip_lac.py` | `skip_unchanged` 在 `l_agent_chat` 大包上的探测/跳过实测（`test_skip_unchanged.py` 的同款） |
| `g6_push_align.py` | G6 本地清单驱动的**全量直传对齐**（四包，`skip_unchanged=True`；输出 `g6_push_align_out.txt`） |
| `g6_final_check.py` | G6 四包 `hash_files` + `missing_files` + 远端 HTTP 200 终检（输出 `g6_final_out.txt`） |
| `g6_remote_gate.py` / `g6_remote_gate_turn.py` | 远端收尾门复验（后者按「最后一个 agent 事件起」切片，避免混入历史轮次；输出 `g6_remote_gate_turn_out.txt`） |
| **MCP 真浏览器（`mcp_Playwright`）** | 真 Chrome 端到端实测（**不是**脚本化 SSE）：`playwright_navigate` 开 `http://127.0.0.1:1250/chat` → 新建会话 → 在 `div.composer-input[contenteditable="plaintext-only"]` 里输入 → `playwright_click` 点 `button:has-text("…")` 回答收尾门 → `playwright_screenshot` 留证 → `playwright_get_visible_text` 读回答。**调用前先读 schema**（`c:\Users\wb.fengqingqing\.trae-cn\mcps\...\mcp_Playwright\tools\*.json`） |
| **`ac_browser_e2e_out.txt`（证据）** | 「MCP 真浏览器端到端实测」的原始证据：会话 id / 截图清单 / 成对判据 / 工具失败 / 行为链 / `run_command` 根因 / post-restart 复验 / `format ` 误伤（§一~§六） |

**事件流路径**：
- **本地**：`D:\Temp\Log\.l_agent_ws\events\<session>.jsonl`
- **远端**：`D:/TD_Depot/l_agent_ws/.l_agent_ws/events/<session>.jsonl`
  —— 远端 `l_agent_chat` 的工作区根 = `D:/TD_Depot/l_agent_ws`（取自 `/api/settings` 的
  `workspace_dir`），**不是** `D:\Temp\Log`；查错路径会得出「远端从没跑过 agent」的**错误结论**。

—— **每个 SSE 事件都即时落盘、无损**（见 `events.py` 头注释），所以它是排查的唯一权威来源；
界面上的显示可能滞后或被前端折叠，**不要拿界面当证据**。

> ⚠️ **`seq` 不是全局唯一键**：`wuwo svc restart l_agent_chat` 后，同一会话文件会**继续追加**新回合，
> 但 `seq` 从 1 重新计数（实测：旧回合到 245，新回合 1~147，同文件共 392 事件）。
> 事件**顺序**仍正确；若要按 seq 定位，须先按「最后一个 `agent` 事件」切片（同 `g6_remote_gate_turn.py` 的做法）。

**关键判据速查**：

```
stop": true                   应为 0     （工具失败不该终止整轮）
收尾前请用户确认               非 0      （收尾门真的走到了）
ask_required / done           成对       （问了几次、最后放行）
"type": "error"               应为 0     （不该有假错误）
DSML                          应为 0     （方言泄漏）
```

---

## 7. 红线（别碰）

1. **远端部署别用「git 差集」推跨版本线的东西** —— 今天就是这么把 8765 推挂的。
   要走**全量直传**（本地清单驱动），它的安全性来自「远端独有文件不会被覆盖或删除」。
2. **别覆盖远端 `.env`** —— 它是远端独有的，全量直传不会碰它，手工 rsync / 覆盖式同步会。
3. **远端 33 篇语料与本地不是一套**（本地 2548 篇）—— 代码对齐 ≠ 数据对齐，重建索引属动数据，先拍板。
4. **改 `.bat` / `.cmd` / `.ps1` 必须 CRLF + 无 BOM + 仅 ASCII**；`.md` 同目录保持一致的换行符
   （`Rez-Docs/` 是 CRLF，`rez-package-source/<包>/999.0/README.md` 是 LF —— 各自一致即可）。
5. **`.py` 改动不热重载**，必须 `wuwo svc restart <别名>`；服务操作一律走 `wuwo svc`，别 `wuwor` 直启。
