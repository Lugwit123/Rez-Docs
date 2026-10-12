# pygraph 版轻智能体 · 只读双跑对照

> 目的：给"把智能体的控制流编成图"这件事**先拿到可比数据**，再决定要不要动 `l_agent_chat/app.py`
> 那套硬编码步循环。**不改线上路径、不碰线上会话**。
>
> 状态（2026-10-12）：**两臂 n=3 已跑齐**（Arm A 走隔离实例 `:1299`，Arm B 走本包 harness），
> 中位数表见 §4；原始样本已归档 `rez-package-source/l_mindmap_mmd/999.0/tools/ab_runs/2026-10-12_n3.jsonl`，
> 复跑脚本 `tools/run_arm_a.cmd` / `tools/run_arm_b.cmd`。顺带抓到节点缓存的一个真 bug（见 §4 末节）。

## 1. 两臂是什么

| | Arm A（老 agent） | Arm B（pygraph 版轻智能体） |
|---|---|---|
| 控制流 | `app.py` 步循环 + flow 图（谓词/门） | 一张 pygraph 图：`AIText 规划 → Compare/If 分支 → 只读工具 → AIText 成句` |
| 工具 | `l_agent_tool` 全套（带审批、足迹） | 图里的 `CallFunction`（**只用只读 stdlib**：`glob:glob` / `json:loads` / `json:dumps`） |
| 会话/审批 | 有 | **无**（这版故意不碰） |
| 跑法 | 隔离实例 + 它的 API（配方见 §3） | `tools/agent_lite_ab.py`（本包内，`wuwor` 跑） |

Arm B 的图结构（脚本里 `build_graph()` 就是权威定义）：

```
Input(question) → AIText(规划: 只输出 {"tool":"list_py","dir":…} 或 {"tool":"answer"})
                → json:loads → GetItem(tool) / GetItem(dir)
                → Compare(tool == "list_py") → IfStart
                     └ glob:glob(pathname="*.py", root_dir=dir)   ← 区内
                   IfEnd
                → json:dumps(files) → concat(question, dump) → AIText(成句) → Output
```

两个刻意的设计（都踩过才这么写）：

- **串化放在 if 区外面**：区被跳过时区内节点输出 `(None,)`，区外的 `json.dumps(None)` = `"null"` ✓
  所以两个分支都跑得完（区里放 dumps 就会先被跳过、没得串）。
- **只读清单写死在脚本里**（`READ_ONLY_TOOLS`）：这份清单就是"这一臂不碰副作用"的凭据。

**同一套 harness 上的下一件事：多步问题集 + 真值机检（2026-10-12）** —— 现在这唯一一道题是**单跳**，
两臂答案都是"3 个 .py"，差异只剩固定开销，**看不出控制流能力**。计划 §2.1 定了三道多步题
（q2 二跳计数 / q3 写+审批 / q4 跨轮），真值机检器已落地：`l_mindmap_mmd/999.0/tools/ab_truth.py`
（现算真值 + `--check-file` 判对错，纯标准库、不花 token）。Arm B 侧已接上其中两道：
`--q q2`（递归 glob + 模型汇总；⚠ 会撞宿主 60s 墙钟超时，见计划 §2.1）与
`--q q4`（**跨轮**，两轮同 session，两轮都 `✔ 对`）。

## 2. 怎么跑 Arm B

```cmd
wuwor l_mindmap_mmd -- python rez-package-source\l_mindmap_mmd\999.0\tools\agent_lite_ab.py --dir E:\lugwit\trayapp\rez-package-source
```

- 默认问题 = `列出 <--dir> 顶层有哪些 .py 文件？`（**别在命令行传中文**：cmd 码页会把参数搞坏，实测踩到）。
- `--dump-events` 打 SSE 原始事件（排错用）；`--print-arm-a` 只打印 Arm A 配方。
- 脚本自己登录（`admin01/666` → cookie）、PUT 图、跑一轮、打指标；**不动别的图**。

## 3. 怎么跑 Arm A

按 `Rez-Docs/Rez_pkg/l_agent_chat_隔离实例A_B实测.md` 的配方起一份独立端口实例（**不带 `.dev_mod`**
+ `.soloignore`，否则改源码会热重启、两臂可能跑的不是同一份代码），再用它的 API 问**同一句**问题，
把「轮次 / 工具调用 / 耗时 / 是否收尾」抄进下面这张表。

已固化成仓库脚本（**从干净控制台**跑，判据看落盘产物不看回显）：

```cmd
rez-package-source\l_mindmap_mmd\999.0\tools\run_arm_a.cmd   :: 探活 → 必要时重启隔离实例 → 跑 1 样本 ×AB_N(默认3) → 追加 arm_a.jsonl
rez-package-source\l_mindmap_mmd\999.0\tools\run_arm_b.cmd   :: 同题 ×AB_N(默认3) → 追加 arm_b.jsonl
```

- 样本级守卫写在 `run_arm_a.cmd` 里（每个样本先探活，死了就当场重启实例并等就绪）——
  正是为了扛「隔离实例跑一轮被硬杀」那条坑。
- 两脚本产物与原始归档都在 `tools/ab_runs/`；同一形状追加（见 §4 末的字段清单）。

## 4. 实测（2026-10-12）

### 4.0 n=3 中位数（当前基线）

同题（`列出 E:\lugwit\trayapp\rez-package-source 顶层有哪些 .py 文件？`）各跑 3 次。

| 指标 | Arm A（老 agent，隔离实例 `:1299`） | Arm B（pygraph 小图） |
|---|---|---|
| 模型往返（规划步数） | **4**（规划 9.6~11.2s + 作答 3.6~5.2s） | **2**（`plan` + `answer`） |
| 工具调用 | **3** | **1**（`glob`） |
| 审批 | **0**（3 次里 1 次为 1） | **0** |
| 总耗时（中位数） | **14.2s**（13.9 / 14.2 / 17.1） | **5.33s**（4.62 / 5.33 / 5.78） |
| 节点步数 | —（非图执行） | **12** |
| 答案 | ✅ 3 个 `.py`；中位 **1514 字**（1299 / 1514 / 1673） | ✅ 3 个 `.py`；中位 **64 字**（64 / 64 / 99） |
| usage | prompt ≈58.5k / completion ≈1.4k，total ≈60k | 未采集（AI 节点不回 usage） |

- **原始样本**（7 行 = Arm A 3 + Arm B 4，其中 1 行是"关增量前"的首次成功样本，带 `incremental` 缺省）：
  `rez-package-source/l_mindmap_mmd/999.0/tools/ab_runs/2026-10-12_n3.jsonl`。
  后续样本按**同形状追加**，一行一 JSON，字段：
  `arm / ts / question / dur_s / plan_steps / steps / tools / approvals / chars / answer / errors / incremental`
  （Arm A 行另有 `plan_s` / `answer_s` / `tool_s` / `usage` / `status_tail`）。
- 复跑：`tools/run_arm_a.cmd` + `tools/run_arm_b.cmd`（追到 `ab_runs/arm_a.jsonl` / `arm_b.jsonl`）。
- **这张表只能读成基线**（三条注意事项见下面「怎么读这张表」）。

### 4.1 第一版（n=1，历史：第一次长什么样）

**Arm B**（`--dir E:\lugwit\trayapp\rez-package-source`）：

| 指标 | 值 |
|---|---|
| 节点步数 | 12（`in>plan>parse>tool>is_list>dir>files>if>dump>qa>answer>out`） |
| 分支 | `{"if": true}`（走了列目录那一支） |
| 模型调用 | 2 次（`plan` 规划 + `answer` 成句） |
| 耗时 | 7.56s |
| 报错节点 | 无 |
| 答案 | `顶层共有3个.py文件，分别是 check_class_list.py、restart_app.py、test_scan.py。` |
| 核对 | `dir *.py` 实测 = 3 个、文件名一致 ✅ |

**Arm A**（老 agent，隔离实例 `:1299`，同一句问题；采集脚本 `tools/agent_arm_a.py`，
按隔离配方起实例 → 问一句 → 自动批准工具 / 自动回答追问 → 记指标 → 删会话）：

| 指标 | Arm A（老 agent） | Arm B（pygraph 轻智能体） |
|---|---|---|
| 模型往返（规划步数） | **4**（规划 7.7s + 作答 5.4s） | **2**（规划 + 成句） |
| 工具调用 | **3**（工具耗时 0.2s）· **审批 1 次** | **1**（`glob`，无审批） |
| 总耗时 | **16.4s** | **7.56s** |
| 答案正确性 | ✅ 3 个 `.py`（与 `dir` 一致） | ✅ 同 |
| 答案风格 | 1717 字（问题复述 + 表格 + 说明） | ~60 字（一句话） |
| usage | prompt 57513 / completion 1015（cache 命中 42240） | 未采集（AI 节点不回 usage） |

**怎么读这张表（别误读成"图编排更好"）**：

- **任务带宽不同**：Arm A 是**全能 agent**（多工具 + 审批 + 复述/表格那套作答规矩），Arm B 是
  **单一工具的小图**（只有 `glob`）。所以"省一半时间、少一半往返"是**窄任务的必然**，不是图编排的功劳。
- 这张表的用处是**定基线**：同样一句"列目录"，全能 agent 的固定开销 ≈ 4 次模型往返 + 3 次工具 + 1 次审批
  + 1717 字作答；小图 2 次往返、1 次工具、一句话。**要判断图编排值不值，得看"该省的地方省了没有"**
  （比如不需要审批的读操作、不需要复述的短问），而不是比谁快。
- **下面 §4.1 那组是 n=1 ✗**：**单次差异全是噪声**，留着只为记住"第一次长什么样"；谈方向看 §4.0 的 n=3。

### 七个新坑（本轮踩到，补进清单）

- **隔离实例会被硬杀，且杀它的人不留名**：症状 = 实例日志停在 `POST /api/chat … 200`、
  **无 traceback**、连"启动它的那个 cmd"的收尾行都没有（整棵树没了）；而**纯空闲探活 90s 全 OK**
  → 说明它不是自然退出，死因发生在"**跑一轮 chat**"或"**调用方进程树收尾**"这两处之一。
  对策：**样本级守卫** —— 每个样本先探活，死了就当场重启实例并等就绪，再跑 `--n 1`；
  这样即使"跑一轮就被杀"也拿得到逐样本数据。模板 `D:\Temp\lac_ab\arm_a_guarded.cmd`
  （`.cmd` 必须 CRLF / 无 BOM / 纯 ASCII，改完复检）。
- **cmd 的 `timeout /t` 在无控制台的管道里会直接报错**（`Input redirection is not supported`）→
  等待循环别用它，改 `ping -n 6 127.0.0.1 >nul`（或 `powershell -c "Start-Sleep 5"`）。
  报错不影响正确性，但会让"等待"变成空转（我这轮就是靠 curl 重试碰上的）。
- **PowerShell 往 `.cmd` 里"插一行"别用单引号里的 `` `r`n ``**：单引号字符串**不处理反引号转义**，
  `-replace 'a','a`r`n b'` 会把 `` `r`n `` 六个字符**字面写进文件** → cmd 读到 `set /a s=0`r`nset /a w=0`
  直接报「运算符不存在」，脚本从此静默半途而废（我这轮就栽在这：Arm A 只跑完样本 1 就停，
  退出码还是 0）。插行要么用**双引号**字符串，要么 `[IO.File]::WriteAllText` 拼 `"`r`n"` 并复检 backtick 数 = 0。
- **cmd 的 `echo` 里别出现孤立 `(`**：`echo (above: ...)` 会让 cmd 进入**续行块**（等 `)`），
  之后你敲的每条命令都被当成"块里的内容"吃掉 —— 现场看就是"命令**凭空消失**、回显还是上一条"。
  我这轮花了四条命令才认出来。补救：单发一个 `)` 收尾，再 `echo READY-OK` 确认。
  连带教训：**别把长命令拼成 `&` 链**（一条坏链能把整个控制台带进怪状态），一次一条；后台任务输出重定向到文件更稳。
- **最阴的一条：仓库根目录冒出过一个垃圾 `wuwor.bat`，它把真启动器遮住了**。
  现象 = `wuwor.bat …` 一律报 `'above:' 不是内部或外部命令` / `rc=1`，**脚本包装得再干净也没用**；
  查法 = `Get-Item wuwor.bat`（cwd 是 `e:\lugwit\trayapp`）+ 比对 mtime：根目录那个只有 **61 字节**
  （内容就是一行我失控命令的残留文本 `(above: AFTER-CALL/TAIL-OK missing = …)`），
  而真的是 `wuwo\wuwor.bat`（757 字节）。**cmd 先找当前目录**，所以 cwd 里的同名文件优先于 PATH。
  处理：删掉根目录那个（已验证非仓库内容，AGENTS.md 只认 `wuwo\wuwo.bat` / `wuwo\wuwor.bat`），
  之后同一套包装脚本立刻跑通。
  **教训**：往控制台里打的任何东西都可能被重定向落盘 —— 我那条拼接命令同时"写了个脚本 + 跑它 + 打标记"，
  结果残片写进了仓库根；**包装脚本一次只干一件事**，命令别拼 `&` 长链。
- **控制台会卡在"续行态"，也可能把残缺文本喂给后来启动的子进程**：
  现场是回显反复出现同一行拼接残片（`e:\lugwit\trayapp>(above: AFTER-CALL/TAIL-OK …`），
  我先后误判成"wuwor 用了裸 `exit`"、"stdin typeahead"（加 `< NUL` 无效）—— 都不对，真因是上面那条。
  可靠对策：`Start-Process cmd.exe -ArgumentList '/c','<wrapper>.cmd' -WindowStyle Hidden -Wait`
  开**新控制台**跑包装脚本；判据永远看**落盘产物**（日志 / JSONL / mtime），不看回显。
- **引擎级坑（增量烹饪 × if 区）：同一张图连跑两次，第 2 次起下游全错 —— 2026-10-12 已修**。
  最小复现：`POST /api/pygraphs/<图>/run`，正文 `{}`（默认 `incremental: true`）—— 第 1 次正常
  （本例 12 步 / 2 次模型往返 / 7.97s / 答案对），第 2、3 次只跑 0.06s、`node_errors` 里躺着
  `dump / qa / answer / out`、答案空（缓存把上一轮的节点值复用回来，if 区里那几个节点没被重算）。
  规避曾是正文传 `{"incremental": false}`；**现在不用了**（`IfStart` 已与循环括号一并排除出缓存，
  见本文末「顺带抓到一个真 bug」）。
  **本 A/B 的 Arm B 依旧关增量**（`tools/agent_lite_ab.py:run_graph` 写死并记进行里），
  理由只剩 ①：每样本都该是"完整一轮"，才和 Arm A 同口径。
- **收实例要认人**：`netstat -ano | findstr :1299` 拿 pid → `Get-CimInstance Win32_Process`
  看 `CreationDate`/`CommandLine` —— **只收自己起的那份**（我这次 pid 36592 / 07:26:16 = 我这轮起的 ✓）。
  别人的隔离实例（可能是别的会话的实验）**别杀**。

### ✅ 顺带抓到一个真 bug：节点结果缓存 × 括号节点（2026-10-12 **已修**）

**症状**：Arm B 第一次跑（同问题、已被缓存过）出现 `steps[].cached: true` 落在 `in/plan/parse/tool/is_list/if`
上 → `if` **走缓存**、区内的"执行 / 跳过"逻辑没发生 → 区内的 `files` 既没产出也没被标 `skip`
→ 下游 `dump/qa/answer/out` 连环报「上游节点 files 没有产出」；用户手点第二次就是 0.06s 秒退、答案空
（`_traces/agent_lite_read.jsonl` 有原始记录）。

影响：任何**带 if 的**图，第二次跑（缓存命中）都会整段失效 —— 而且报的是下游的错，根因（`if` 被缓存）很远。

**根因**：`graph.py::_execute_plan` 里排除缓存的集合只算了循环括号（`plans | end_plans`），
漏了 `if_plans`。`IfStart` 一旦命中，只是把括号自己的旧值塞回去，**"跑区内节点 / 整段跳过"这一步没执行**。

**修法**（`l_node_engine`，2026-10-12）：`bracket_nodes = 循环括号 ∪ if 括号` 一律不进缓存
（区内节点本来就不在外层 `order` 里、没有指纹 → 等价于"这一区每轮真跑"）。坑记在
`l_node_engine.md` 坑 16。

**判据（都已满足）**：

- 引擎自证：`_check_incremental` 里新增一张带 if 的图（真/假两支），**连跑两次**都 7 节点全产出、
  `node_errors` 为空、结果一致、且 if 括号与区内节点**不在** `cached` 里、条件那侧的链照旧复用
  （`wuwor l_node_engine -- l_node_engine_selftest`，RC=0）。
- 真跑（`agent_lite_read`，宿主 `:8110`，正文 `{"incremental": true}`）：**冷缓存连跑 2 次**都
  **12 步 / `node_errors` 空 / 答案非空**；第 2 次缩短到 2.56s 且 `cached_nodes` 非空（复用生效）。
  修前同一场景第 2 次 = 0.06s + 四个下游节点报错 + 答案空。

**Arm B harness 仍然显式关增量**（`tools/agent_lite_ab.py:run_graph`）—— 理由 ①（每样本都该是"完整一轮"、
才与 Arm A 同口径）仍然成立；本文档不再以"bug 未修"为理由。
