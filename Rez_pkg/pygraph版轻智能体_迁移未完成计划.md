# pygraph 版轻智能体 —— 迁移未完成清单与计划

> 2026-10-12。用户口径：**要判断"把智能体控制流编成图"值不值，并且真的把控制流搬进图**。
> 这份文档只记「做到哪 / 还差什么 / 判据是什么 / 按什么顺序」；通用背景看
> `l_node_engine.md`（引擎）、`流程图智能体_图驱动控制流.md`（现状控制流，权威）、
> `pygraph版轻智能体_双跑对照.md`（Arm A/B 对照与坑清单）。

## 0. 一句话现状

| 阶段 | 状态 |
|---|---|
| ① 引擎侧控制流（分支 / 人在环 / 跨轮状态 / 结构校验 / 轨迹回放 / 沙箱 A/B / 策略包） | **完成 + 自证**：`wuwor l_node_engine -- l_node_engine_selftest` 全绿（子图是第 21 组）、web `node --test` 43 项 |
| ② 第一版对照：Arm B（小图）vs Arm A（老 agent） | **已收口**：n=3 表在对照文档 §4.0、原始样本 `tools/ab_runs/2026-10-12_n3.jsonl`、复跑脚本 `tools/run_arm_a.cmd` / `run_arm_b.cmd`（见 §1） |
| ③ 换发动机：`plan` / `tools` 步循环搬进图 | **第 1 步（影子）✅、第 2 步的等价证据 ✅（5 场景两档对拍）、第 3 步起点（flow→pygraph 镜像）✅**（2026-10-12）；**执行本体已收敛成唯一一份 ✅**（2026-10-14，`tool_turn.py`）；**goal 模式已对齐 ✅ + 换发动机 n=3 对照已出 ✅**（2026-10-14）；**翻默认档位**留人拍板 |
| ④ 子图 / FX-Map 宿主接线 | **已完成**（引擎 + 宿主 + `TexFxMap`；见 `子图与FX-Map_计划.md`） |
| ⑤ 阻断项：增量缓存 × 括号节点 | **已修**（2026-10-12）：`IfStart` 与循环括号一并排除出缓存；自证 + 真跑连跑 2 次都过（见 §4） |

## 1. P0：② 已收口（2026-10-12）

### 1.1 n=3 表已搬进对照文档 ✅

- `pygraph版轻智能体_双跑对照.md` §4.0 = **n=3 中位数表**（§4.1 那组 n=1 保留为"第一次长什么样"），
  §3 标题已无「（待跑）」。数据：Arm A 中位 14.2s / 4 次模型往返 / 3 次工具；Arm B 5.33s / 2 次往返 / 1 次工具。
- ⚠ 这张表只能读成**基线**，别读成"图编排更好"（理由三条在对照文档 §4，别删）。

### 1.2 原始数据已归档 ✅

- `rez-package-source/l_mindmap_mmd/999.0/tools/ab_runs/2026-10-12_n3.jsonl`（7 行 = Arm A 3 + Arm B 4，
  其中 1 行是关增量前的首次成功样本）；对照文档 §4 指了这个路径。
- 后续样本按同形状追加（一行一 JSON，字段：
  `arm/ts/dur_s/plan_steps/steps/tools/approvals/chars/answer/errors/incremental`）。

### 1.3 采集脚本已固化 ✅

- `tools/run_arm_a.cmd`（探活 → 必要时重启隔离实例 → 跑 1 样本 ×AB_N → 追加 `ab_runs/arm_a.jsonl`）与
  `tools/run_arm_b.cmd`（同题 ×AB_N → 追加 `arm_b.jsonl`）；样本级守卫写在 `run_arm_a.cmd` 里。
- 隔离实例是 `.soloignore` 豁免，与 `wuwo svc` 不冲突。

## 2. P1：把"可比"做实（1~2 天）

### 2.1 多步问题集（当前唯一题看不出差异）

- 现状：唯一题是**单跳**"列出 `<目录>` 顶层有哪些 .py 文件"——两臂答案都是"3 个 .py"，
  差异只体现在固定开销上，**看不出控制流能力**。
- 待做（每题都要"多步、且答案可机检"）：

| 题 | 形态 | 考点 |
|---|---|---|
| Q2 二跳 | 统计 `<目录>` 下每个子包里 `*.py` 的个数并按数量排序 | 多轮工具 / 汇总 |
| Q3 需审批 | 写类操作（新建文件再读回内容） | Arm A 会触发审批；Arm B 目前只读 → 要么出**只读等价题**，要么给 Arm B 加写节点并明确"审批"这一维怎么记 |
| Q4 需跨轮 | 第二轮问"比上一轮多了几个" | 跨轮状态（`StateIn`/`StateOut`） |

- 判据：每题两臂答案都能对拍出**真值**（脚本/`dir` 算一遍），"看着像对"不算。

  **✅ 判据那半边已落地（2026-10-12）**：`l_mindmap_mmd/999.0/tools/ab_truth.py` —— 纯标准库
  （不联网、不花 token），把三道题的**真值现算出来**并给一份"答案要点"，再用 `--check-file` 判对错：

  ```bat
  :: 算真值（q2 = 子包 .py 计数排序；q3 = 写入内容；q4 = 两轮差，用 --prev 喂第一轮真值）
  wuwor l_mindmap_mmd -- python <包>/999.0/tools/ab_truth.py --q q2 --dir <目录>
  :: 判一份答案（**中文答案写文件**，别走命令行 —— cmd 码页会把参数搞坏，实测踩过）
  wuwor l_mindmap_mmd -- python <包>/999.0/tools/ab_truth.py --q q2 --dir <目录> --check-file D:\Temp\答案.txt
  ```
  退出码 0 对 / 1 错 / 2 用不了；实测两极性都对（正确答案 `ok:true`、含糊答案 `ok:false`）。
  q2 的判据片段故意**不含"子包总数"**：真实目录里有 `.git`/`.venv`/`--version` 这类噪声（实测 84 个），
  两臂对它们的取舍本来就不同 —— 那样判的是"数目录的耐心"，不是"多跳汇总"的能力。

  **还差的**：三题各跑两臂 n=3（花 token，走 `run_arm_a.cmd` / `run_arm_b.cmd` 的样本级守卫）。

  **Arm B 的 Q4（跨轮）已就位且跑通（2026-10-12）**：`tools/agent_lite_ab.py --q q4`
  —— **同一个 session 跑两轮**，两轮之间采集器往顶层放一个 `.py`（这样"多了几个"的真值是 **1**，
  蒙不出来）；图里加四个节点：`prev`(`StateIn`) / `keep`(`StateOut`) / 两段带标签的清单 /
  `answer.prompt` 接"本轮 + 上一轮"。跨轮状态**全由宿主存取**（跑时把会话里的值灌进 `StateIn.default`、
  跑完收 `StateOut` 输出），图只 declare 键名 —— 引擎不碰磁盘。
  实测：两轮都 `✔ 对`（第一轮 17 步 / 2 次模型往返、第二轮 17 步 / `if_taken=True` 真重新列了一次），
  跑完**探针文件与会话都清掉**。两个踩出来的坑（都写进代码注释）：
  ① **`StateIn.default` 是 JSON 文本** —— 写 `[]` 会得到 list，下游 `operator:concat` 直接炸
  （`can only concatenate str (not "list") to str`）→ 空字符串要写 `""`；
  ② **第二问的措辞必须含"再列一次"** —— 只问"比上一轮多了几个"时模型直接 `answer`（`if_taken=False`），
  `dump` 是 `null`，谈不上对比。

  **Arm B 的 Q2 已就位、且 2026-10-13 起"图自己算"**：`tools/agent_lite_ab.py --q q2`
  —— `glob('*/*.py')` → **`GroupCount(key_sep="/")`**（按子包分组计数、按个数降序）→ `Sum` 出合计 →
  `ToText` 拼进提示；模型只**照着念**（"别自己重算"）。实测 `answer_ok=True`、17.8s，
  表格与真值逐项一致（`_tmp 9 / l_test_code 2 / repo_tools 2 / Test 2 / 合计 15`）。
  （此前是"把清单塞给模型让它数"：提示词大、还会算错 —— 那正是 §3.2 里"引擎无数值原语"的后果，
  2026-10-13 补了 `Calc/Count/Sum/GroupCount/ToText` 五个节点后这条不再成立。）
  过程里踩过的两次墙（留痕）：`**/*.py` 递归 → 提示词爆炸（调到 300s 也回不来，真因是题目/模式写错）；
  宿主原先只有 env 一条超时路 → 已补 per-request `{"timeout": N}`。
  中途撞的三次墙（都留了痕，别重复踩）：
  ① 原写 `**/*.py`（递归整棵树）→ **提示词爆炸**，作答那步连 300s 都回不来；而且**口径也不对**
  （真值数的是"子包**直接**放的 .py" = `*/*.py`）。**换成一层通配后 9.6s 就答完了** ——
  "撞超时"当时的症状，真因是题目/模式写错。
  ② 宿主原先**只有 env 一条超时路**（`L_MINDMAP_RUN_TIMEOUT`，60s）→ 已补 **per-request
  `{"timeout": N}`**（夹 1..3600；env 仍是默认）并用 `Ask` 图实测：给 2s → `dur=2.03s` +
  `执行超时（2s）`，证明认的是请求里那个值。
  ③ 采集器现在默认给长题 `--timeout 300`（q2/q4），客户端等待也跟着放宽。

### 2.2 判读口径写死

- 现状：只有"别读成图编排更好"三条。
- 待做：明确**哪些开销是该省的**（读操作免审批、短问免复述、规划往返数）与**哪些不许省**
  （护栏、收尾、痕迹、审批本身），每条能指回表里某个量。
- 判据：换发动机后能用这组口径判"劣化/持平/改善"，不靠感觉。

## 3. P1：换发动机（大项，未起步）

### 3.1 现状（先别搞错）

- `flow_enabled=1` 只驱动**收尾路由**；再开 `flow_full=1` 才把 `branch_calls`/`branch_steps`
  的迭代决策也交给图。
- `plan` / `tools` 两个动作节点的**执行**仍由 `app.py` 步循环托管（规划是 LLM 往返，
  工具执行要审批 / 流式 / 痕迹）。
- 所以"pygraph 具备智能体能力"目前 = **图能表达控制流 + 有沙箱可验**，但**还没接管任何一轮真对话**。

### 3.2 三步走（每步都留退路）

1. **影子模式**：同一轮里由**图**决定"下一步走 tools 还是收尾"，`app.py` 照旧执行；
   分歧只写进 `_traces` 与一条 status，**不改行为**。
   判据：真对话 n 轮，给出"图决策 vs 现状决策"的分歧率 + 分歧样例；**对用户的可见差异 = 0**。

   **✅ 已落地（2026-10-12）** —— `l_agent_chat/flow_shadow.py` + `app.py` 循环体末尾的采样点：

   - 开关 `flow_shadow`（settings / env `AGENT_CHAT_FLOW_SHADOW`，默认 0）；`GET /api/flow/status`
     回 `flow_shadow`。**热生效**：`POST :1250/api/settings {"settings":{"flow_shadow":1}}` 即可
     （watcher 把值同步到 `config.FLOW_SHADOW`），不用重启。
   - 每圈**跑过工具**的那一步：用同一份状态在**新鲜** context 上重跑一遍图（桩护栏全放行 +
     空动作 handler），入口 = **`tools` 之后那条分支**（`tools_exit()`；默认图 = `branch_steps`）。
   - 一行三个决策：`graph`（图怎么判）/ `baseline`（**硬编码那套判据**怎么判：`fail_limit_hit` /
     `stop_loop` / 步数用尽 → 收尾）/ `actual`（这一轮**真正驱动**的一方给的）——
     `diverge = graph != baseline`，**两种档位下都是同一个问题的答案**："图与硬编码在这圈上
     会不会走出不同的路"（`flow_full=0` 时 `baseline` 就是现状；`flow_full=1` 时 `actual` 就是图、
     `baseline` = "关掉图会怎样"）。
   - **采样点必须在两种驱动方式都会经过的地方**（循环体里"这一步工具跑完、各档出口之前"）：
     早先放在循环体末尾，而 `flow_full=1` 那条路必定 `break`/`continue` → **永远到不了** →
     "开着图驱动时影子一行都不落"（2026-10-12 实测踩到；**本机 `flow_full` 恰好就是 `true`**）。
   - 落 `<flows>/_traces/_shadow.jsonl`（一行一步，留 **4000** 行 —— 按"一天的量"给的，一行 ~300B）。
     一致 → 界面**什么都不显示**；有分歧 → 一条 `🧪 影子：N 步可比 M · 分歧 K（x%）` + 样例。
   - **怎么读**（攒一天后）：`wuwor l_agent_chat -- python <包>/999.0/tools/shadow_report.py --since <日期>`
     —— 分歧率 + 按图分组 + 分歧样例（`--json` 给脚本吃；退出码 0 = 有样本且零分歧 / 1 = 有分歧 / 2 = 没样本）。
   - 两条刻意的保守（写进模块文档，别当 bug）：**护栏用桩**（"门拦不拦"这一层看不见，`gates`
     字段给出路径上的门让人分辨；桩按**图里声明的护栏名**现生成，所以**用户图里新加的门也覆盖得到**——
     实测本机图多了一道 `contract_restate`，影子照样跑通）；**只采样跑过工具的圈**（没调用那圈本来就
     由图驱动，无可比性）。
   - 自证：`tests/test_flow_shadow.py`（**14 例**，含"从 start 开跑永远只得 loop"、"硬编码基线 vs 图"
     的判据、落盘截断）+ `tests/test_flow_shadow_turn.py`（**5 例**端点级：默认关零影响 / 一致时静默 /
     **改图后真有分歧行与 status 且行为不变** / n 轮分歧率 0 / **`flow_full=1` 那档也采样**）；
     相关面回归 33 例 + `test_flow_full_equivalence` 5 例全绿。
   - 现在能抓到什么：默认图上两边恒等（= "可以安全接管"的证据）；有人给 `branch_steps` 加一条
     "预算见底 → 收尾"之类的改动 → 影子立刻报分歧。
   - 主文档：`流程图智能体_图驱动控制流.md` §2.1。
   - **真对话分歧率：已量到（2026-10-13）n=21 · 分歧 0（0.0%）** —— 11 轮真模型对话（读/写/搜/多读/
     无工具/大文件/失败路径 + 3 轮步数上限=1），**两支出口都覆盖到**：`graph=loop×18` + `finish×3`
     （`budget_exhausted` 那 3 行），`baseline` 分布完全一致 → **图（full 档）与硬编码基线在真对话上一致**。
     怎么凑出来的（**这条比数字重要**）：本机 `l_agent_chat` 一夜**零自然对话**，光开着影子只攒到 1 行
     → 用 **`tools/shadow_feed.py`** 主动喂（真模型/真工具/`persist:false` 不落会话），
     再用 `tools/shadow_report.py` 读。
     ⚠ 三条边界别读过头：① 这些轮次的问题**是我给的**（不是自然用法）；
     ② 护栏是桩 → "门拦不拦"那一层看不见；③ `fail_limit_hit`（熔断）与 `stopped`（工具要求停）
     **没采到**（要专门造：连续失败 / 让工具要求停）。

2. **接管分支决策**：等价 `flow_full`，但决策由图上的门表达（不再散落成 `params` 阈值）。
   判据：`turn_replay.py` 沙箱 A/B 通过（**不拆护栏、不绕远、能收尾**）+ 真跑 n=3 不劣化。

   **确定性等价证据已就绪（2026-10-12）**：`tests/test_flow_full_equivalence.py` —— **5 个场景**
   在两档各跑一遍（`flow_full=0` / `=1`），断言**步数 / 工具 / 答复 / `trace.steps` 全等、零 error**：

   | 场景 | 覆盖的图上分支 | 结果 |
   |---|---|---|
   | 一次工具 + 正常收尾 | `branch_calls` / `branch_work` / `gate_*` → `end` | 等价（`end` 两边都是 `end`） |
   | `max_tool_steps=1` | `branch_steps` 的 `steps_exhausted` | 等价（1 步收尾） |
   | 连续失败 ×2（`max_consecutive_failures=2`） | `fail_limit_hit` | 等价 |
   | 完全没用工具 | 收尾路由（两档共用） | 等价 |
   | `confirm_before_finish=1` | `branch_confirm` + `gate_confirm` | 等价 |

   两处**记录差异**（不是行为差异，别当 bug）：① 步数用尽时**两档都会**发一条
   `type:error`「工具调用步数已达上限（1 步），本轮到此为止」（基线行为，实测两档一字不差）；
   ② 轨迹里的 `end`：硬编码 break 那条路记 `""`，图驱动那档记 `end_abort`/`end` —— 图那一档记得更细。

   ⚠ 老实说：`turn_replay.py` 是**"候选图 vs 现状图"**的工具（`--flow-a/--flow-b`），
   不是"同一张图两档"的 A/B —— 所以第 2 步的沙箱判据落在上面的场景表上；
   **真模型 n=3** 与"翻默认档位"仍待做（后者会改用户可见行为，**留给人拍板**）。
3. **整轮图化**：`plan`/`tools`/收尾都在图上，宿主只当"执行器"（LLM 往返 / 审批 / SSE / 痕迹）。

   **接线层已落地（2026-10-13）** —— 引擎侧钩子 + agent 侧一轮跑通：

   - **引擎**：`l_node_engine/host_steps.py` —— 节点 `HostStep`（`step` + 动态端口）+ 进程级注册表
     `register_host_step(name, fn, *, description, params, outputs)`；**没注册就报清楚**
     （不许静默产出 `None`）；多输出（执行器返回 tuple）按序交出去。自证 selftest 第 28 组。
   - **agent**：`l_agent_chat/graph_turn.py` —— `TurnContext`（宿主侧状态）+ 四个宿主步骤
     `plan`（默认走 `agent_client._planner_step`，真 LLM 往返）/ `tools`（**分发器由调用方注入**，
     审批/流式/痕迹不搬进图）/ `has_calls` / `is_answer`（两个**宿主谓词**，图上拿它们做
     `IfStart` 条件与 `LoopEnd.stop`）+ `run_graph_turn(graph, ctx)`（注册 → 跑 → **一定注销**）。
   - **v1 接缝（写清，别当成能用的数据流）**：图上 `HostStep` 只给 `step`；**动作之间的数据交接走
     宿主 context**（messages / 工具结果本来就在宿主侧，且是几十 KB 级）。"数据流也进图"是后续
     （给 `HostStep` 加 `ports` 动态输出 + 图内取值节点），与"控制流进图"分开做。
   - **自证**：`tests/test_graph_turn.py` **5/5** —— ①图真驱动一轮：第一圈要工具（dispatch 真被调一次）、
     第二圈不要 → `LoopEnd.stop` 收尾（`rounds=2` / `stop_reason="answer"` / 输出端口有值）；
     ②步数上限到点收尾（`stop_reason="budget"`，不是崩）；③跑完四个步骤**全注销**（不污染下一轮）；
     ④没注册 → `node_errors` 说清"宿主没注册 + 已注册哪些"；⑤`tools` 没接分发器 → 明确报错
     （**不偷偷执行**）。引擎侧 `l_node_engine_selftest` 同时覆盖 `HostStep` 三条判据。
   - **接真实执行本体（只读档，2026-10-13 同轮）**：`plan` 默认就是真 LLM 往返（`_planner_step`）；
     `tools` 的分发器由调用方注入 —— 沙箱脚本 `l_agent_chat/tools/graph_turn_demo.py` 给的是
     **只读档**（复用现成演练模式：写类在 `AgentTool.__call__` 处一律拒，读类放行 ⇒ 模型想写
     也改不了盘）。真模型自证：`--ask "读 AGENTS.md 第一行"` → **rounds=2 / stop_reason=answer /
     errors=0**，答案正确；模型一直要工具时 → `stop_reason=budget`（护栏那条也走过）。
     ⚠ 踩到的最要命一条：工具结果**必须回灌** planner 对话（`_append_tool_turn`），否则模型下一圈
     看不到结果、只能盲目重规划（实测同一个问题从"2 圈答案正确"退化成"4 圈撞上限 + 答非所问"）。
   - **还差（第 3 步的主体，不是接线）**：把 `app.py` 回合函数里**内联**的 plan / tools 执行本体
     抽成可调用函数，挂到 `plan`/`tools` 两个宿主步骤上（那函数 ~2500 行，plan/tools 与 SSE/审批/
     检查点交织 → 得先拆；沙箱里走的是"只读档 + 无审批"的简化分发器）；以及**翻默认档位**
     （会改用户可见行为 → 留给人拍板 + 真对话 n=3 对照）。

     **✅ 执行本体已收敛（2026-10-14，"真抽执行本体"）** —— 判据/记账不再是两份：
     - **新增 `l_agent_chat/tool_turn.py`**（全**同步**，无 `await`）：判据链拆成逐条小函数
       `hook_pre` / `permission_effect` / `fs_block` / `needs_approval` / `mcp_autoapprove` /
       `approval_preview` / `hook_permission_request`，记账拆成 `snapshot` / `record`，另有组合入口 `judge()`。
       **顺序照 `app.py` 现状**（PreToolUse hook → 规则引擎 → deny → 读后写闸门 → 是否审批 → 预检 →
       PermissionRequest）。**同步**是关键：pygraph 分发器（跑在宿主线程）可原样调用，
       硬编码步循环在其上 `await` 自己的部分。
     - **`graph_turn.approval_dispatch`** 改调它（顺带补齐原简版缺的五项：PreToolUse 改写 /
       PermissionRequest / edit·apply_patch 预检 / mcp `autoApprove` / PermissionDenied hook）；
       旧私有 `_needs_approval`/`_fs_block`/`_hook_deny`/`_perm_effect`/`_before_write` 删除。
     - **`app.py`** 的判据块 / 快照块 / 记录块改调同一批函数；`_diff_preview`/`_apply_patch_preview`
       从 app 删除。**异步点**（审批等待 / 监工看门狗 / SSE yield）与特殊工具分支、失败路径**仍在 app**
       —— 那正是"审批 / 流式 / 痕迹不搬进图"的既有边界。
     - **边界（本次没动）**：`plan` 本来两边就已同调 `agent_client._planner_step`；**翻默认档位**仍留给人拍板。
     - 判据：`tests/test_tool_turn.py` **20/20**（含"唯一性防回归"：app / graph_turn 都不许再定义那份逻辑）
       + `tests.test_graph_turn` **38/38** + `tests.test_flow_full_equivalence` + **端点批 103/失败 2**
       （两条为环境所致：市场接口取到非 JSON、开发机上多出的 `office-*` 技能；均与工具执行面无关）
       + 真跑三件：`l_agent_chat_graph_smoke` **4/4** / `web_probe.cjs agentgraph` **5/5** /
       `l_agent_chat_graph_step_probe`（人在环：出询问条 → 批准 → 落盘 + checkpoint + diff）PASS。
     - ⚠ **踩过的坑（端点批抓到，已修）**：`tool_turn.fs_block`/`record` 早先对**空会话**直接短路
       （"没会话就跳过"）—— 但 `app.py` 的老调用方不带 `session_id` 时 `req_sid` 就是**空串**，
       而 `fs_guard` 的观察与判据按 `""` 这一桶**成对**记账 ⇒ 短路 = 读后写闸门**静默关掉**。
       现两处均照空串判/记，并有 `test_empty_session_still_gates` 防回归。
     - 计划/设计：`openspec/changes/c9-tool-turn-consolidation/`。**纯搬家，用户可见行为不变**。

     **✅ goal 模式已对齐（2026-10-14，换发动机收尾）** —— 与硬编码那条线**同一份判据**：
     - **缺口（改前）**：图侧 `goal` 门**只看时间** ⇒ 目标已达成也会被推回去硬撑到预算耗尽；
       且推回提示只写"说一句就收尾"，**没叫模型打 `GOAL_DONE`/`GOAL_BLOCKED`** ⇒ 模型不打标记、
       一直被拦，看着像"图不认标记"（其实是提示词没要求）。
     - **改法（只动宿主 `graph_turn.py`，图资产不动）**：`TurnContext.goal_verdict`（`done`/`blocked`/`""`）
       + 模块级 `_goal_verdict()`（懒导入 `chat_modes.goal_verdict` = **唯一实现**，与硬编码同款）；
       `do_plan` 每圈从**模型正文**算结论并留一条 `🏁 目标达成 / 受阻`；`do_goal` **有结论即放行**；
       `GOAL_NUDGE` 改写成硬编码那条线的原文（逐条对照验收标准 + 单独一行打标记）。
       ⚠ 只喂 `reason`（正文）：`assistant` 是原始 dict，`str()` 会把 `reasoning_content` 一起带上，
       模型在思考里复述一句标记就会被误判。
     - 判据：`tests/test_graph_turn.py::GoalGateTest`（预算内拦 / 预算到点放行+交代进度 /
       **GOAL_DONE 提前放行** / **GOAL_BLOCKED 提前放行** / 非 goal 不看标记 / 推回提示必须含 GOAL_DONE）
       + 真跑：goal 模式预算 180s、目标门只拦 **1** 次（上限 2）、**18.6s 就提前收尾且 `停因 answer`**
       ⇒ 只可能因读到 `GOAL_DONE` 放行（复现两次）。

     **✅ 换发动机 n=3 对照（2026-10-14）** —— 同一问题 / 工作区 / 模型，**只差智能体**：
     - harness：`l_mindmap_mmd/999.0/tools/agent_arm_a.py` 新加 `--agent graph|hard`（**ASCII 键**绕开
       命令行中文码页）/ `--mode goal` / `--goal-seconds` / `--n`，并加判别器 `graph_driven`（这轮真由图
       驱动）/ `goal_gate`（目标门真拦过）。跑在隔离实例 `:1299`（当前代码 + 稳定产物，见
       `l_agent_chat_隔离实例A_B实测.md`）。⚠ `🏁` 是**过程流水**行、**不走 SSE**，哨兵别量 SSE（两臂皆然）。
     - 问题：`列出 <rez-package-source> 顶层有哪些 .py 文件？`（两臂逐字相同）。
     - 原始样本：`l_mindmap_mmd/999.0/tools/ab_runs/2026-10-14_engine_n3.jsonl`（agent 模式、两臂，靠
       `agent` 字段区分）/ `2026-10-14_engine_goal.jsonl`（goal 模式，含复跑那次）。

     | 臂 | 智能体 | 图驱动 | 工具 | 审批 | 耗时(s) | tokens | 答案(字) |
     |---|---|---|---|---|---|---|---|
     | hard | `硬编码智能体` | 否 | 3~6 | 1~2 | 15.5 / 20.6 / 24.5 | 61k / 77k / 78k | 1283~2337 |
     | graph | `default_intelligent_agent` | 是 | 2~3 | 0~1 | 9.7 / 15.8 / 19.2 | 44k / 44k / 62k | 816~1106 |

     - **判读口径（先定口径，别把变化读成"劣化"）**：
       - ✅ **正确性**：两臂都对（3 个 `.py`：`check_class_list.py` / `restart_app.py` / `test_scan.py`）。
       - ✅ **该省的**：图臂 tokens 中位 **~44k vs hard ~77k**（约 **−43%**，同题）、工具调用更少
         （2 vs 4）、答案更短（816~1106 vs 1283~2337 字）。
       - ⚠ **用户可见差异（唯一一条）→ ✅ 已关闭（2026-10-15）**：当初 hard 臂答复**带"问题复述"**
         （`contract` 那一步的可见产物）、图臂**不复述**（图臂 `contract` 节点当时只提醒、不产可见复述）。
         用户拍板**完全对齐 hard 口径**（并同时覆盖画布那条路）后已补上：图侧重新把 `restate_question`
         广告给模型（撤 `app.py` 两处刻意过滤）、`do_contract` 改成真校验（漏了照抄 `RESTATE_RETRY_NUDGE`
         重试一次）、两个分发器加"只记不发"元工具分支、作答那一刻 `compose_reply` 拼成答复第一段
         （模型漏了 → `fallback_restate` 本地兜底 ⇒ **"复述必然存在"**）+ 第 0 圈就发 `restate` 事件；
         **图资产不动**。判据：`RestateContractTest` 6 例 + `graph_smoke` 4/4（隔离实例同样 4/4，
         `read` 档新增"答复第一段必须是问题复述"）+ `step_probe` 默认/`--slots`/`--always` 全过 +
         `web_probe agentgraph` 6/6。实现与代价见 `l_agent_chat/.../doc/CHANGELOG.md`（2026-10-15 条）。
       - ✅ **收尾门的口径也补齐了（2026-10-15 同轮）**：`fact` 门（先核实）原来比 hard **松**——
         图侧 `did_read` 把 `list_dir` / `search_text` 也算"读过"，于是"只搜过就下结论"反被放过去。
         现已照 `app.py` 那处护栏的原文判据对齐（`not read_any and (located_any or not did_tools) and
         _looks_like_code_question(last_user)`，谓词/工具名表仍从 `app` 懒导入），并照 hard **只拦 1 次**；
         非代码问题这道门不再插手。判据 `GraphTurnTest::test_fact_gate_same_as_hard` 4 例。
         ⚠ 唯一剩下的差异是**收益性的**：模型第 0 圈没先复述时 `contract` 会追加纠正消息 ⇒
         那一轮多跑一圈（**与 hard 同口径**：hard 也重规划一次；模型照做时不多跑圈）。
       - ⚠ **不许省的都还在**：两臂都出审批卡、都落 `_traces`、都有收尾门（`门拦`/`提醒` 都在）。
       - ⚠ **别读成"图更快"**：耗时差受 prompt cache 命中影响（hard 先跑、后两轮多命中缓存），
         单题不足以下结论；**多步问题集**是 §2.1 的待做项。
     - **结论**：在"该省的开销"（tokens / 冗余工具调用）上**图臂占优或持平**，
       在"不许省的面"（护栏 / 审批 / 痕迹 / 收尾 / **问题复述**）上**一致**；**未发现劣化**。
       `翻默认档位`已无待拍板的口径差异（原"是否保留 restate"那条 + "`fact` 门比 hard 松"那条
       均已于 2026-10-15 关闭）。

   **起点已落地（2026-10-12）**：`l_mindmap_mmd/999.0/tools/flow_to_pygraph.py` 把默认智能体的
   **流程图**（goto 状态机）翻成 **pygraph**（拓扑求值 + 括号区），产出
   `pygraphs/default_intelligent_agent.json`【镜像】（21 节点）：

   - 步循环（`plan → tools → branch_steps →(回边) plan`）→ `LoopStart/LoopEnd`；跨轮上下文 →
     `LoopEnd.state` 回灌（`transcript`）；收尾 → `LoopEnd.stop`（模型说 `answer` 就停）→ `AIText` → `Output`。
   - 工具那一步 v0 = **只读档**（`glob:glob`）；flow 的 `tools` 背后那套带审批/痕迹的工具集**仍在宿主**
     （= 第 3 步真正要搬的东西）。
   - **两条不可平移**（都已实测，逐条落成 `fx_*` 便签，别假装平了）：
     ① 背边 → 括号区（pygraph 图必须无环，`while` 用 `LoopStart(until_stop=True)`）；② 多路 `branch` 与
     `gate` 在 pygraph 侧没有**同名原语** → 多路控制流用"嵌套 `IfStart` + `Not`"、多路选值用 `Select`
     （都 2026-10-13 起可用；**`else` 2026-10-16 起可直写** = `IfStart … ElseStart … IfEnd` 三件套，
     见引擎 §3.1 —— 这条不再必须靠 `Not` 嵌套）；门的判据结构仍归宿主。
     ~~③ `if` 不能放进循环体~~ → **2026-10-12 已由引擎放开**（见 `l_node_engine.md` §六）：
     镜像里的 `branch_calls` 现在是**真 if 区**（`has_calls` + `if_tool/end_tool`，工具只在需要时才跑），
     不再退化成"无条件工具 + stop"。
   - 判据：结构校验 errors/warnings **全空** + 覆盖表 16/16（`--verify`）；真跑 **2 圈收敛**
     （`is_answer=True` → `stop`）、答案正确、`node_errors` 空。
   -    ⚠ 镜像**不是**运行时本体：默认智能体的控制流仍跑在 `flow_engine` 上；这张图现在是"能看、能校验、
     能跑（只读档）"的**靶子**，第 3 步要做的正是把宿主的执行本体接进 `plan`/`tools` 两个位置。

## 3.5 P0～P3：工程 / 资产化（用户 2026-10-13 提的大方向）

口径（用户原话）：**"一个工程一个大目录，里面可以创建函数、类图、结构体……希望实现 UE 蓝图的大部分功能；
agent 的 pygraph 就是第一个工程"**。分四期，每期都能真跑、都能停：

| 期 | 内容 | 状态（2026-10-13） |
|---|---|---|
| **P0** | **工程容器**：目录即工程、地址 `工程/资产`、老图兼容、列表带 `project`、`/api/pygraphs/projects`、路由升级（13 条 `{name:path}` + 裸路由 `{name:asset}`） | ✅ **宿主 + 前端都做完并实测**（2026-10-13）：宿主侧工程内建图/跑图/列表/工程接口、老 23 张平铺图不坏；**左侧资产树**（「📁 工程与资产」按工程分组、三类资产、点图打开、点结构体/类落节点、↻）；探针场景 `assets` 9 项 PASS。**还差**：`project.json`（工程设置/依赖）、重命名/搬家 |
| **P2** | **函数库化**：编辑器按子图输出**画多端口** + 资产树分类 | ✅ **做完并实测**（2026-10-13）：宿主 `ports?kind=subgraph` 的 `outputs` + 前端 `pySchema.applyOutputPorts` / `toFlow(graph, info, portsByNode)`（**按节点**覆盖输出元数据）→ 引用的图有 2 个 `Output`，画布上就是 2 个端口（实测 handles `["out0","out1"]`、名字来自那张图）。⚠ 踩到一处：端口 effect 原先**把 `outputs` 丢了**（只存 input/methods/signature/doc）。 |
| **P1** | **结构体（Struct）**：字段表资产；运行期就是 dict；节点 `MakeStruct`（字段名作动态输入）/ `SetField`（值语义返新 dict）/ 取值用现成 `GetItem`；**按名字引用**（图里只填 `工程/名字`，宿主内联字段表） | ✅ **引擎 + 宿主 + 前端都做完并实测**（2026-10-13）：节点 `l_node_engine/struct`；资产 `kind=struct`（重名当场 400）+ `GET /api/pygraphs/structs`；名字引用 `_inline_structs`（**内联成 JSON 文本**）+ `struct_ref` 400；**字段端口 + 资产下拉**（`ports?kind=struct` + `STRUCT_NAME_PORT` 补成 select，实测 handles=`struct/fields/n/out0`）。**还差**：结构体编辑器（表格）、`project.json` |
| **P2** | **函数库化**：编辑器按子图输出**画多端口**（数据已备：`list_graph_outputs` + ports API 的 `outputs`）+ 资产树分类 | 引擎/宿主已备，差前端 |
| **P3** | **类图**：`实例 = dict（字段）+ 方法 = 图引用`；节点 `ConstructClass` / `CallGraphMethod`（**不生成真 python 类** —— 纯 JSON 资产、可热更、无动态类序列化坑）；继承/接口先不做 | ✅ **v1 引擎 + 宿主做完并实测**（2026-10-13）：类资产 `kind=class`（`fields` + `methods{方法名:图地址}`）+ `GET /api/pygraphs/classes`；`_inline_methods` 按「类→方法→图」把方法图内联成 `sub`（方法图里 `Input(self)` 拿实例），解析不到 → `method_ref` 400。实测：构造 41 → 调 `bump` → **42** ✓。v1 边界：方法名**静态**、方法图里再定义方法不支持。**还差**：类编辑器、继承/接口、按值动态分发（要就自己接 `Select`） |

- 类图那条路的取舍：**实例=dict + 方法=图**（推荐）vs 生成真 python 类（更像 UE，但动态类 + 序列化 + 热更都变麻烦）。
- 老图策略：**根目录 = 兼容层**（新资产必须进工程，老图原地不动）—— 推荐，已按它实现。
- 引擎能力侧的前置（都已做）：子图 = 函数（**多输出** 2026-10-13）、`while`/嵌套 if/嵌套循环、`Select`、
  `Calc/Count/Sum/GroupCount/ToText` —— 即"**用图写函数**"这条路已经通了；**类**要 P3，**结构体**要 P1。
   判据：迁移前后同题 n=3 对照（步数 / 耗时 / 工具数 / 收尾率）+ 一份"用户可见差异"清单。

### 3.3 硬约束（写前面，别到时候才发现）

- 审批 / 流式 / 痕迹**必须仍由宿主托管**（引擎只留钩子：`set_guard` / `set_ask` / `set_stub`）。
- 沙箱**验不了**提示词 / 阈值 / 表达类改动（回放里放的是旧提示词下的响应）→ 这类改动走
  **隔离实例真采样 A/B**（`l_agent_chat_隔离实例A_B实测.md`）。
- 改图 = **人拍板**：agent 只出建议（`optimize_agent` / 会话复盘），不自动改 `flows/*.json`。

## 4. P0：阻断项已修 —— 增量缓存 × 括号节点（2026-10-12）

- 症状：同一张带 `if` 的图连跑两次，第 2 次起 `dump/qa/answer/out` 全报错、答案空（0.06s 秒退）。
- 根因：`graph.py::_execute_plan` 排除缓存的集合只算了**循环**括号，漏了 `if` 括号 —— `IfStart`
  命中缓存 = 只把括号旧值塞回去，**"跑区内节点 / 整段跳过"这一步没执行**。
- 修法：`bracket_nodes = 循环括号 ∪ if 括号` 一律不进缓存（区内节点本来就不在外层 `order` 里，
  没有指纹 → 等价于"这一区每轮真跑"）。
- 判据（都满足）：① 引擎自证 `_check_incremental` 新增一张带 if 的图（真/假两支）**连跑两次**，
  两次 7 节点全产出、`node_errors` 空、结果一致、括号与区内节点不在 `cached` 里、条件侧链照旧复用；
  ② 真跑 `agent_lite_read`（宿主 `:8110`，正文 `{"incremental": true}`）**冷缓存连跑 2 次**都
  12 步 / `node_errors` 空 / 答案非空，第 2 次 2.56s 且 `cached_nodes` 非空。
- 记录：`l_node_engine.md` 坑 16、`pygraph版轻智能体_双跑对照.md` 末节（已从 "⚠ 待修" 改成 "✅ 已修"）。
- ⚠ 没修的那半边**不是 bug**：子图内**有意不接**增量缓存（key 是节点 id，会撞外层），改子图 →
  整块重跑（`l_node_engine.md` §六）。

## 5. P2：能力接线（做了才有"更聪明"的收益）

### 5.1 子图接线（阶段②③）

- 引擎 `SubGraph` + 宿主侧（图名 → 内联、参数端口、编辑器 UI）+ `TexFxMap` **均已完成**（2026-10-12，含真跑实测）。
- 与迁移的关系：子图 = **可复用的子流程**，是"换发动机"之后的模块化手段。
- 详见 `子图与FX-Map_计划.md` §3（那里有逐项判据，别在这儿抄一遍）。

### 5.2 人在环在智能体侧对齐

- 现状：引擎有 `Ask` 节点、pygraph 前端有 ask 栏 ✓；**缺**"真跑智能体时审批走哪条路"
  （老 agent 的 `approval_required` 与图的 `Ask` 是两套）。
- 判据：一轮需要审批的真对话，前端出现一次可点审批，且痕迹里能看出它属于**哪个节点**。

### 5.3 轨迹/回放用于真对话

- 现状：pygraph 轨迹只记图内 `run`；真对话的轮次轨迹在 `l_agent_chat` 侧 `_traces/<图>.jsonl`。
- 判据：一轮真对话能在编辑器里回放（或至少在控制流面板看到节点路径 + 被拦的门）。

**✅ 已落地（2026-10-15）—— 图驱动一轮的轨迹落盘**（`l_agent_chat` 侧，宿主改动、图资产不动）：

- 病根：默认档由 pygraph 驱动后，`app.py` 仍按 `flow_spec_ and flow_fctx` 往 `_traces` 写 ——
  但图驱动那轮**没跑过 `run_flow`** ⇒ `flow_fctx.visited` 恒空 ⇒ `steps:0, seq:[], segs:[], guards:{}`。
  实测该文件 **174/200 条是空的**，最后一条**非空**停在 `2026-10-14 14:27:26`（= pygraph 接管默认档之前）：
  真记录被 200 行保留上限逐轮顶出去，读端被告知"这轮什么都没跑"。
- 修法：`graph_turn.run_trace(ctx, box)`（纯函数）→ 复用 `_append_flow_trace` 落同形记录
  （`mode="pygraph"` / `seq`=逐节点事件（滤掉 `skip|mute|bypass|stubbed`）/ `segs`=按循环圈分段 /
  `guards`=门拦 / `steps`=圈数 / `end`=停因）；共享尾巴加 `and not _pg_drove`，**图驱动轮跳过**
  轨迹落盘 / 影子对拍 / 策略建议 / 控制流路线（它们的输入在图档恒空 ⇒ 产出的是谎报与误导）。
- 判据：`TraceRecordTest` 5 例 + 点名批 52 例 OK；`graph_smoke` 4/4、`graph_step_probe` PASS（含 `--slots`）；
  文件级取证 末行 `seq` **62–86 个节点**（含 `plan`/各门/工具）、`segs`=圈数+1；
  读端 `GET :1250/api/flow/traces?name=…` **零改动**可消费。
- ⚠ **两条没做的（记录，别当已完成）**：
  ① **FlowEditor 的节点/边热力对 pygraph 档仍对不上** —— pygraph 节点 id（`tools_read`/`g_stall`）与
  flow json 的节点名（`branch_work`/`tools`）**不是同一命名空间** + `flow_traces_api` 按 **flow json 的真实边**
  过滤 ⇒ 边热力为空、节点热力只反映 pygraph 侧计数。**跨命名空间的热力归并**才是"能在编辑器里回放"的
  最后一里（判据里那句话仍只有前半句达成）。
  ② `_traces.seq`（逐节点、含循环体）与 `flow_evolve._LAST_RUN.seq`（引擎**外层拓扑序**、只到 `LoopStart`）
  口径**不同** —— 未擅自统一（会改变 `optimize_agent` / `session_review` 的取证输入）。

## 6. P2：文档与测试缺口

- [x] `l_node_engine.md` 补 pygraph 侧索引 ——**已补（2026-10-12）**：新增 **§3.4「宿主侧的四个配套」**
      （校验 / 轨迹·回放·历史 / A-B 沙箱 / 策略包：每一项都写清"引擎给了什么契约"+"宿主在哪"，
      避免以后把宿主的事记进引擎文档）。控制流节点（`Compare/Not/And/Or`、`IfStart…IfEnd`、`LoopStart…LoopEnd`）
      与 `validate_structure` 本来就在 §3.1 / §2.4 里。
- [x] `pygraph版轻智能体_双跑对照.md`：§3 标题去掉「（待跑）」、§4 换 n=3 ——**已做**（§4.0 是 n=3，§4.1 保留 n=1 当历史）。
- [x] pygraph 编辑器**没有 e2e**（`web/tests` 43 项都是纯函数/组件单测）→ 加一条
      「建图 → 校验 → 跑 → 看轨迹 → 看 A/B 摘要」——**已补（2026-10-12）**：
      `l_mindmap_mmd/999.0/tools/web_probe.cjs` 新增场景 **`runflow`**（10 项断言，实测 PASS 10/10）：
      纯函数小图（`Input(int) → Compare → IfStart → json:loads(42) → IfEnd → Output` + `Note`）→
      点「校验」→ 点「▶ 运行」→ 轨迹面板自动多一条 → 点「回放」出现回放条。
      判据写法上踩到一条：**`IfEnd` 的输出是 `taken`（区走了没走）、不是区内值**（`graph.py::_run_if`），
      所以按**节点**断言（`← 4: [42]` 与 `← 6: [true]`）—— 先前只写 `/42/`（匹配全日志）是
      "为对的原因通过了"。
      ⚠ **`A/B 摘要` 不适用**：那是流程图编辑器的沙箱 A/B，pygraph 侧没有这个面板（已在 web_probe.md 写明）。
- [x] `l_agent_chat_隔离实例A_B实测.md` 补两条配方：**样本级守卫**与**新控制台**
      ——**已补（2026-10-12）**：新增 **§2.1「两条保命配方」**（现象 / 做法 / 参照实现 / 参照脚本），
      与 §2 的坑 4、坑 6 对应；那份文档从此是这两条的权威（对照文档那边只留"本轮踩到"的现场记录）。

## 7. 顺序与依赖

```
② 收口 ✅ ─→ P1 多步问题集 ─→ 判读口径 ─→ ③ 换发动机：影子 ✅ → 接管分支决策 → 整轮图化
缓存 bug ✅ ─┘                              （起点：flow→pygraph 镜像 ✅）
```

- ② 已收口（n=3 + 归档 + 复跑脚本）：换发动机的判据要用**同一套 harness**，所以先把这套稳定下来。
- 缓存 bug 已修 —— 前端默认开增量，用户手点第二次不再跑坏；这也是 ③ 的前置（带 if/循环的图必须能连跑）。
- ③ 第 1 步（影子）已落地：**先量等价、再接管**。第 2 步（接管分支决策 = 等价 `flow_full`）之前，
  需要一段真对话攒出的分歧率（默认图上应为 0；有分歧必是"图改了"或"门拦不拦"那类）。
- 换发动机每步**可停**：影子模式不通过就不进第 2 步；每步都要能退回"现状硬编码循环"。

## 8. 已知风险

| 风险 | 影响 | 应对 |
|---|---|---|
| 隔离实例"跑一轮被硬杀"**未根治** | n 连跑掉样本、结论残缺 | 样本级守卫 + 新控制台；用"n=6 连跑不掉样本"当判据 |
| 另一个会话正在改 `graph.py` | 两臂可能跑在**不同代码**上，结论看着有数据其实无效 | A/B 期间锁一份代码；样本行里记下文件 mtime |
| 前端默认开增量缓存 | ~~用户手点第二次就跑坏~~ **已修**（2026-10-12，见 §4） | 保留一条自证：带 if/循环的图连跑 2 次必须都对 |
| Arm B 只是"单一工具的小图" | 容易被读成"图编排更省时间" | §2.2 口径 + 对照文档 §4 的三条注意事项 |
