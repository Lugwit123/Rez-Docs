# 子图（引擎）+ FX-Map（贴图算子）—— 计划与现状

> 2026-10-12。用户口径：**B 真·子图版 FX-Map** —— 先给引擎 / pygraph 打通"一张图引用另一张图"
> （子图 / 图实例），再把 FX-Map 当第一个用家。**这份文档只记"做到哪、还差什么、判据是什么"**，
> 别在这里堆通用背景（背景见 `l_node_engine.md` / `l_tex_nodes.md`）。
> **四块都已做完并实测（2026-10-12 上午）**；下面 §三 的勾都是"跑过的"，判据与证据逐条写在同一条里。

## 0. 一句话现状

| 阶段 | 状态 |
|---|---|
| ① 引擎：`SubGraph`（把另一张图当一步跑） | **完成 + 自证通过**（21 组全绿）；图名→内联的通用实现 `resolve_graph_names` 也在引擎 |
| ② 宿主：图名 → prompt 内联、端口、编辑器 UI | **完成 + 真跑实测通过**（内联 / 参数端口 / 图名下拉 / 「↗ 子图」/ 按名字拦环 / 缓存复用口径） |
| ③ `TexFxMap`（象限树 / 迭代） | **完成 + 自证通过 + 宿主真跑一次（图案子图被调 4 次）** |
| ④ 文档与索引 | **完成**（引擎文档 / 贴图文档 §7.4 / 索引 / CHANGELOG） |

自证：`wuwor l_node_engine -- l_node_engine_selftest` → `SELFTEST PASS`（第 21 组即子图那组）；
`wuwor l_tex_nodes -- l_tex_nodes_selftest` → `SELFTEST PASS`（第 9 组即 FX-Map 那组）。

## 一、已完成（阶段 ①）

| 文件 | 内容 |
|---|---|
| `l_node_engine/999.0/src/l_node_engine/subgraph.py`（新） | `SubGraph` 节点 + `prompt_hash` / `inject_params` / `collect_outputs` / `SubGraphError` / `MAX_DEPTH=8` / 调用栈 ContextVar |
| `l_node_engine/.../graph.py` | `_ACTIVE_GUARD` + `active_guard()`、`_ACTIVE_EMIT` + `active_emit()`、`_HASH_LITERAL_INPUTS` + `_literal_fingerprint()`（`sub` 走整体 sha1） |
| `l_node_engine/.../__init__.py` | `discover_module(_subgraph)`（导入即注册） |
| `l_node_engine/.../selftest.py` | 第 21 组 `_check_subgraph()`：参数注入 / Output 取值 / 成环 + 深度 / 内层失败外抛 / 事件前缀 / **指纹随内容变** / 循环体内逐轮执行 / 栈不泄漏 |

## 二、定死的设计（别再重新讨论）

1. **契约 = 嵌套图的 `Input` / `Output` 节点**，与 `pygraph.py` 当 agent 工具时**完全同一条**
   （一张图既能当工具给 agent 调，也能当子图给别的图调；不新造 `SubInput` / `SubOutput`）。
2. **`sub` 是内联 prompt，不是图名**。理由两条：引擎**宿主无关**（不认 pygraphs 目录 / 文件名）；
   更要紧 —— 增量烹饪的指纹按字面量算，内联进来"子图内容改了 → 指纹跟着变"，
   只放图名就会**名字没变、内容变了 → 复用旧结果**（静默出错）。
3. **三道护栏**：深度 `MAX_DEPTH=8` / **按内容哈希**判成环 / 护栏接力（`active_guard()`，
   显式传入的 guard 不能被子图绕过）。三者都用 ContextVar，多会话共进程不互相看见。
4. **指纹**：`sub` 走**整体 sha1**（与 `subgraph.prompt_hash` 同口径）。**不能**走 `_safe_repr`
   ——它给 dict 只展开 20 项 × 40 字符，两份不同的子图会算出同一个指纹。
5. **错误不套娃**：`SubGraphError` 原样上抛（内层失败 → 外层不再包一层"子图里有 N 个节点失败"）。
6. **子图内不接增量缓存**（缓存的 key 是节点 id，嵌套 id 会和外层撞）。
7. **输出 v1 只取一个**：多个 `Output` 时按 `name` 排序取第一个，其余写进 `ui.text`。
   （多输出要动编辑器的端口声明 —— 前端按静态 `RETURN_TYPES` 画端口 —— 留到"子图多输出"那步。）

## 三、完成情况（勾选清单 + 判据 + 证据）

### 3.1 阶段 ① 收尾（小，半小时）

- [x] `__init__.py` 的 `__all__` 导出 `SubGraph` / `SubGraphError` / `MAX_DEPTH` / `prompt_hash`
      （外加 `GraphNameError` / `resolve_graph_names` / `run_nested` / `run_nested_value`）
- [x] `Rez-Docs/Rez_pkg/l_node_engine.md`：§三 节点表加 `SubGraph` 一行（契约 / 为什么内联 / 三道护栏 /
      指纹口径 / 指回本文件）、§四 坑 14/15、§六 未做两条 → 已改成"引擎 + 宿主都已做"
- [x] `l_agent_chat/doc/CHANGELOG.md` 一条

### 3.2 阶段 ② 宿主接线（`l_mindmap_mmd`，中等）——**已完成 + 真跑实测**

- [x] **解析图名 → 内联**：`server._inline_subgraphs()`（跑图与校验两个入口都走）→ 引擎
      `resolve_graph_names(prompt, _load_graph_for_subgraph, pkg="l_mindmap_mmd")`；
      宿主只提供"图名 → 图文件（带老图键迁移）"这一层，**没有**自己再写一份净化逻辑。
      实测：外层图 `SubGraph{graph: demo_tool_add, a: 2, b: 3}` → `/run` 真跑通，内层 6 个节点的
      事件带前缀（`1/1` … `1/6`），`Output sum = 5`。
- [x] **改子图内容 → 不复用旧结果**。实测三步：① 首次跑（内层事件 `1/in`/`1/n`/`1/out` 都在）
      → ② 原样重跑 `cached: true` + `cached_nodes: ["1","2"]`（内层一个都没重跑）
      → ③ 把被引图里的 `Not` 换成 `And` 再跑：`cached` 消失、结果由 `False` 变 `True`
      （指纹随内联内容变，与 §二 第 4 条一致）。
- [x] **按图名再拦一道环 / 深度**（引擎按内容拦，宿主按名字拦，报错能指名道姓）：
      实测自引用图 → `POST /api/pygraphs/_sub_cycle/validate` 回 `{"code":"subgraph_ref"}`；
      `/run` → **HTTP 400** `子图成环：_sub_cycle 已经在引用链上（_sub_cycle → _sub_cycle）（节点 1/1）`。
- [x] `GET /api/pygraphs/ports?kind=subgraph&target=<图名>`：读那张图 `Input` 的
      `name` / `type` / `default` / `description`，全部放 `optional`（子图参数都有初值，
      报"必填"是假警报）。实测 `target=demo_tool_add` → `a`(`FLOAT`, 20) / `b`(`FLOAT`, 22)，
      `signature: demo_tool_add(a, b) → sum`。
- [x] 编辑器 UI：`/object_info` 把节点类自己声明的 `GRAPH_NAME_PORT` 那个输入换成**图名下拉**
      （`container=select` + `values={api:"/api/pygraphs", value:"name"}`；宿主不点名任何节点类），
      卡片上有「↗ 子图 <图名>」一键跳过去（`PyNode.jsx` + `PyGraphEditor.jsx` 的 `openGraph`），
      换图后端口刷新（动态端口预取按 `kind: "subgraph"` 问、`target` 进 key）。
      实测：`object_info` 里 `SubGraph.graph`、`TexFxMap.pattern_graph` 都已是 `select` + 图名列表。
- [x] **渲染 / 缓存只认外层 id**：`sub` 事件的 id 形如 `<外层>/<内层>`，而 `outputs` 的键只有外层 id，
      所以 `_cache_renders` / `_TEX_RENDER` / 缩略图天然不会把 `1/2` 当成图里的节点（只认外层）；
      `_RUN_CACHE` 也不受影响（嵌套图 `cache=None`）。实测重跑那轮 `cached_nodes` 里只有 `1`/`2`。
- [x] 顺带确认：`l_mindmap_mmd` 的 `requires` / discover **没改**（`SubGraph` 是引擎自带；
      贴图那组照旧 `discover_nodes("l_tex_nodes.nodes")` 默认挂上）。

### 3.3 阶段 ③ `TexFxMap`（`l_tex_nodes`，大）——**已完成 + 自证 + 宿主真跑**

- [x] 节点 `TexFxMap`，`mode = quadrant | iterate`：
      - `quadrant`：四等分递归 `depth` 层（`ops.fxmap_quadrant_rects` 按深度分配、**不重叠、正好铺满**），
        每块跑一次图案并落在它自己那块矩形里；位移 / 旋转 / 缩放 / 上色按档位递进
        （`_fx_transform`：`t = (i+0.5)/N`）；
      - `iterate`：`iterations` 轮，**上一轮输出当下一轮输入**（子图 `Input(name="image")` 拿它），
        再按 `blend_mode` 叠回。
      判据（自证第 9 组）：象限矩形不重叠且铺满 / 逐档变换生效 / 轮数变了输出跟着变 / 轮内不引随机。
- [x] 变量进子图：`_fx_vars()` 给 `index` / `number` / `x` `y`（中心 0..1）/ `size` / `pos`（= `[x,y]`），
      `$` 前缀可省（`normalize_var_name` 归一）。⚠ 只喂给**能收值的端口** ——
      子图里那个 `Input` 要是被控件（`TexNoise.seed` 那类）占着，就收不了变量，得让它当端口用。
- [x] `pattern` **可以不接**：不接就用 `builtin`（`disc / square / gradient / noise`，
      `ops.fxmap_builtin`）或接进来的 `image` —— 单节点也能用，不必先建一张子图。
- [x] 示例图（调色板预览用）：`samples.SAMPLES["TexFxMap"]`（象限递归 + 逐档旋转/缩放/上色），
      自证里"算子库示例图 33 张全跑通"就含它。
- [x] 自证：**pattern 子图真的被调用 N 次**（按事件计数）+ 变量真进子图 + 跑完调用栈不泄漏。
      实测（宿主，`depth=1` + `pattern_graph=_fx_pattern`）：`1/g`、`1/out` 各出现 **4 次**，
      输出 `TexImage(128x128 rgb)`。
- [x] `pattern` 走**整体 sha1** 指纹（`register_hashed_inputs("TexFxMap", "pattern")`，与 `sub` 同口径）。

### 3.4 阶段 ④ 文档与索引

- [x] `l_tex_nodes.md` §三：节点数 37 → **38**、加 `TexFxMap` 一行；新增 **§7.4 FX-Map / 子图实例化**
      （两种模式 / 变量 / `pattern` 可不接 / 判据 / 指纹口径）；§四 自证组数改 11；§八 改成"三块都已做"
- [x] `l_node_engine.md`：§三 `SubGraph` 行补上 `resolve_graph_names` / `GraphNameError` 与宿主落地情况；
      §六 那条"只做了引擎那一半"已改成"引擎 + 宿主都已做（仍只取一个 `Output`）"
- [x] 索引已补：`AGENTS.md` 必读表 + `Rez-Docs/README.md`（本文件一行；`l_node_engine.md` 那行同步成
      "17 基础 + 6 AI"并点出 `SubGraph`）
- [x] 本文件 §0 / §三 逐条打勾，并写清实测证据

### 3.5 实测记录（2026-10-12 上午，本机 `:8110`）

临时图（跑完已删，都落在 `C:\Users\wb.fengqingqing\.lugwit\main\l_mindmap_mmd\pygraphs\`）：
`_subgraph_e2e`（引 `demo_tool_add`）、`_subgraph_e2e2` + `_subgraph_e2e_inner`（改内容看指纹）、
`_sub_cycle`（自引用）、`_fxmap_e2e` + `_fx_pattern`（`TexFxMap` 引图案子图）。

⚠ 跑图 / 校验接口要**登录**：`POST http://127.0.0.1:1027/api/v1/auth/login`（`admin01` / `666`）
拿 `access_token`，请求带 `Authorization: Bearer <token>` —— 只带 `Content-Type` 会 401
`未登录 lugwit_auth`（实测踩到）。

### 3.6 双击进内层图（2026-10-12 追加，用户要 SD 那种用法）

用户问"现在能双击打开 FX-Map 么，Substance Designer 是支持的"——**原来不能**（DOM 后端只有卡片上
一个「↗ 子图」按钮；canvas 后端连按钮都没有，双击一律是居中放大）。已补上：

- **引擎侧**：节点类可声明 `GRAPH_SKELETON`（一张**图文件模板**），`node_base.get_node_info` 原样带成
  `/object_info` 的 `graph_skeleton` —— 宿主不点名任何节点类。`SubGraph` 的模板是**直通**
  （`Input x → Output y`，建完立刻能跑）；`TexFxMap` 的模板带 FX 变量（`index` / `number` / `x` / `y` /
  `size` / `image` 六个 `Input`）+ `TexGradient` → `Output(image)` + 一张 `Note`（写清变量含义）。
- **宿主侧（`PyGraphEditor.openInnerGraph`）**：双击"有图名端口的节点" →
  ① 已接图 → 打开它；② 没接 → 按 `graph_skeleton` 建一张（名字 = `<当前图名><模板后缀>`，重名自动避让）
  → 写进那个端口 → **把当前这张图也存盘**（跑图读的是服务端那份，不存等于没接）→ 打开新图。
  端口判定沿用元数据（`values.api === "/api/pygraphs"`），所以以后再加"引用另一张图"的节点类自动支持。
- **两个后端都做了**：DOM（`PyNode.onDoubleClick`，且没接图时卡片上也会有「＋ 内层图」按钮）与
  canvas（`CanvasSurface.onDoubleClick` 先判 `data.graphPort`，其余节点照旧居中放大；卡片标题后缀
  `↗ <图名>` / `＋ 内层图（双击建）` —— canvas 没有 DOM 按钮，得在卡片上写出来）。
- **实测**（`:8110`，两个后端各一次）：
  - DOM：`_dblclick_test`（一个 `TexFxMap`，没接图案）双击节点 → 建出并跳到 `_dblclick_test_fxmap`，
    外层 `1.pattern_graph` 已写上；跑外层 → 图案子图被调 **4 次**（depth=1），出 `TexImage(128x128 rgb)`。
  - canvas：切到 canvas 后端后双击同一位置 → 同样建出 `_dblclick_test2_fxmap` 并跳过去，回执
    `＋ 已建内层图 … 并接上 1.pattern_graph（这张图也一并保存了）`。
  - 两个模板单独过引擎也跑得通（`fxmap → TexImage(64x64 rgb)`、`sub → 'hello'`）。

## 四、这一轮踩过的坑（写下来，别再踩）

1. **节点自己拿不到宿主的 `emit`**：`emit` 只传给 `_safe_emit`，节点 `execute()` 里根本没有这个名字。
   第一版在 `SubGraph.execute` 里引用了不存在的 `emit` → 被 `_safe_emit` **静默吞掉**
   （图跑完了，日志里一条子图进度都没有，很难看出来）。为此加了 `graph.active_emit()`。
2. **`_safe_repr` 对 dict 会截断**（20 项 × 40 字符）→ 两份不同的子图会算出**同一个指纹**
   → "改了子图却复用旧结果"。这类"整体语义"的字面量必须整体 sha1（`_HASH_LITERAL_INPUTS`）。
   自证里有一条专门盯它：两份只差在 300 字之后的子图，指纹必须不同。
3. **子图报错别套娃**：8 层嵌套时每层套一句"子图里有 N 个节点失败"，最外层只剩一团乱码。
   所以 `SubGraphError` 约定**原样上抛**（检测内层失败的类型是不是自己人）。
4. **`def` 是 python 关键字**，不能当端口名 → 端口叫 `sub`。
5. **环境坑（不是代码）**：根目录 `wuwor.bat` 被执行别的任务的 agent 覆盖成一行调试文本
   （`(above: AFTER-CALL/TAIL-OK missing = exits the calling cmd)`），于是**所有** `wuwor` 调用失效、
   报"不是内部或外部命令"。已恢复成转发 `wuwo\wuwor.bat` 的 5 行（CRLF / 无 BOM / 纯 ASCII）。
   ⚠ 共用入口文件被并发改动时，**先看一眼再说"代码跑不起来"**。
6. **子图里只能用宿主注册过的节点**（2026-10-12 实测）：自检专用节点（`UpperNode` / `JoinPaths` /
   `wrap_brackets`）**不在** `l_node_engine` 的正式注册表里（那是 `selftest` 自己注册的），
   拿它们写子图 → 内层报 `ValueError: Unknown node class: UpperNode`，再被包成
   `SubGraphError: 子图里有 N 个节点失败：…`。**看到"子图失败"先看内层那句**，
   多半是被引用的图本身有问题，不是子图机制坏了（引擎自证里能用是因为它自己注册了那些节点）。
7. **验证别只靠自证**：引擎 / 贴图的组级自证都过，也可能宿主侧接线是断的（图名没内联、
   端口没暴露）。宿主那半要用**真跑**验：跑一张引子图的图（看内层事件 `1/x` 有没有）+
   再跑一次看 `cached` + 改被引图再看 `cached` 是否消失 —— 这三步才把"内联 + 指纹"钉住。
8. **前端 `useEffect` 的依赖数组会踩 const 的 TDZ**（做"双击进内层图"时踩的，2026-10-12）：
   把后面才 `const` 的 `openInnerGraph` 直接写进前面某个 `useEffect` 的**依赖数组** →
   依赖数组是**渲染期**求值的，于是 `ReferenceError: Cannot access '<minified>' before initialization`
   → **整页白屏**（浏览器控制台第一行就是这个，页面 `innerText` 全空 —— 别去怀疑后端）。
   修法与既有做法一致：回调走 ref 转发（`openInnerRef`，同 `openGraphRef` / `openSubgraph`），
   依赖数组里只留**本 effect 之前**定义的东西。**改完必须重建产物并开一次页面**（只跑 `vite build`
   成功 ≠ 页面不白屏）。

## 五、验证命令

```cmd
wuwor l_node_engine -- l_node_engine_selftest      :: 21 组（子图那组是第 21）
wuwor l_tex_nodes  -- l_tex_nodes_selftest         :: 11 组（FX-Map 那组是第 9）
wuwor l_mindmap_mmd -- l_mindmap_mmd_web_build     :: 改过 web/ 就要重建产物
wuwo svc reload l_mindmap_mmd_server_exec          :: 改 l_tex_nodes 不会触发宿主热更，必须手动
wuwo svc status l_mindmap_mmd_server_exec          :: 改宿主源码前先看：在热更就等它自己起来
```

宿主真跑（本机直连免登录；POST 要带 token，见 §3.5）：

```cmd
curl -s "http://127.0.0.1:8110/api/pygraphs/ports?kind=subgraph&target=demo_tool_add"
curl -s -N -X POST "http://127.0.0.1:8110/api/pygraphs/<图名>/run" -H "Content-Type: application/json" ^
     -H "Authorization: Bearer <token>" -d "{}"
```

- 画布：`http://127.0.0.1:8110/pygraph?name=<图名>`（本机直连免登录；经 nginx 要走登录闸门）
- 写接口（PUT / run / validate）要带 `Authorization: Bearer <lugwit 登录 token>`
  （`POST http://127.0.0.1:1027/api/v1/auth/login`，`admin01` / `666`）
