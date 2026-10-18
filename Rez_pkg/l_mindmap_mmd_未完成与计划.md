# l_mindmap_mmd 编辑器：现状数字 · 已知坑 · 自证命令（参考）

> **计划/待办看 openspec**：`openspec/changes/archive/2026-10-12-mindmap-editor-followups/`
> （`proposal.md` 为什么做 / `design.md` 拍板的取舍 / `tasks.md` 已完成与未做的分项 / `specs/` 行为判据；
> **2026-10-12 已归档 —— 该 change 的 S0–S5 与其验收项全部勾选**，剩下两处诚实的保留：`.fbx` 真资产几何未验、
> 真机帧预算只跑到这台 60Hz 显示器）。
> 这份文档只放**参考事实**（现状数字、踩过的坑、可直接抄的自证命令），不放任务清单 —— 免得两边漂。

## 〇、已完成（截至 2026-10-12，逐条有自证）

| 已完成 | 自证（可复跑） |
|---|---|
| **canvas 渲染后端 P3/P4/P5**：>150 节点走 canvas，且**可编辑** —— 点选/框选/拖节点/拖端口连线/右键菜单（节点·边·空白）/双击居中/滚轮缩放/小地图（可点可拖）/分组框拖动（**框里的节点跟着走**）与缩放/循环区凸包/标志位徽标/耗时徽标/运行中脉冲/热边高亮 | `node …/tools/web_probe.cjs perf`（下面 1.2 的数字）+ `… web_probe.cjs group`（两后端的整框搬运）；编辑器手点 |
| **2000 节点性能收口** | `web_probe perf`：整理布局 **141ms**、稳态 `fillRect` **1/帧**、JS+提交 **0.58ms/帧**；`l_mindmap_mmd_web_test` 里 `layoutGraph` 2000 节点不超时 + 幂等 |
| **`Input` 全 python 类型 + 自由类型表达式**：标量存真标量、容器存 **JSON 文本**、执行时 `coerce_param()` 转真对象；`type` 可写 `list[int]` / `dict[str, int]` / `np.ndarray` | `wuwor l_node_engine -- python …/tools/param_types_check.py` → **35 项全过**（含真跑图下游拿到真 `dict`/`list`/`tuple`、`base64:` 解码） |
| **容器参数的结构化编辑**（S1）：`list` 类给数组行、`dict` 给键值行，JSON 文本域退成兜底视图（可切）；落盘**仍是 JSON 文本** | `web_probe types`（结构化行 + 切回文本视图 + 存回 `["x","y"]`）；`l_mindmap_mmd_web_test` 里 `containerShapeOf/rowsOfText/textOfRows/pairsOfText/textOfPairs` |
| **`bytes` 两个入口**：文本框（utf-8）+ 📄 选文件（存 `base64:<…>`，引擎解码） | `web_probe types`（bytes 控件 + 选文件按钮）；`param_types_check.py` 的 `base64:aGk=` 断言 |
| **连线类型软提示**（design §2.4 A）：canvas 拖线不再按类型硬拦，落点画警示色 + 一句"类型不符：str → int" | `CanvasSurface.linkTarget` 的 `soft` + `typeMismatchHint`（单测）；`web_probe value` 顺带覆盖 canvas 绘制路径 |
| **多文件 glTF（S2）**：一次选 `.gltf`+`.bin`+贴图 → 服务端建 `<uuid8>/` 子目录 → 值取主文件相对名；`.bin`/贴图按同目录相对路径解析 | `web_probe model`：值 `<uuid8>/tri.gltf`、预览 `3 顶点 · 1 面` |
| **`.fbx`（S2）**：白名单 + `FBXLoader` 分支（懒加载 chunk）。⚠ 真资产几何**未验**：仓里那三个 `.fbx` 现在是 106 字节占位 | `web_probe model`（默认只验"不报『预览不支持』"；`set MM_FBX=<真文件>` 可补真预览） |
| **本机路径受控读口（S3）**：`L_MINDMAP_MODEL_ROOTS` 白名单根 + 只认 3D 扩展名 + `resolve()` 后在根内 + 普通文件；👁 不再置灰、根外给明确提示 | `wuwor l_mindmap_mmd -- python …/tools/model_gates_check.py` → **14 项全过**；`web_probe model` 的 HTTP 反例 |
| **canvas 值行（S4）**：`Input` 节点标题栏下画单行截断文本（容器给 `[…] N 项` / `{…} N 键`），LOD `zoom ≥ 0.45` | `web_probe value` → 4 项过（值行 `[…] 2 项` + 标题/行名）；`pySchema.inputValueLine` 单测 |
| **循环区凸包真图对拍（S4）**：canvas 与 DOM 的顶点一致（不是"照抄没验证"） | `web_probe hull` → 8 顶点、归一化最大偏差 **0.000** |
| **`right`/`down` 布局的零重叠 + 幂等**（之前只测了 `flow`） | `l_mindmap_mmd_web_test` 两条新用例 |
| **`type=model` 3D 预览**：原始 body 流式落盘 + three.js **懒加载**（glb/gltf/obj/fbx/stl/ply）+ OrbitControls | `web_probe model` → 15 项过 |
| **文档与索引（S0）**：`AGENTS.md` 必读表加 `web_probe.md`；`l_mindmap_mmd.md` 补 `models/` 四路由 + 值口径 + `L_MINDMAP_MODEL_ROOTS`；源码里三处"canvas 只读"旧注释改掉 | `AGENTS.md` / `l_mindmap_mmd.md` / `CanvasSurface.jsx` |
| **前端纯逻辑**：布局防重叠 + 幂等、参数口径、结构化行、凸包、分组框、策略包形状 | `wuwor l_mindmap_mmd -- l_mindmap_mmd_web_test` → **56 项全过** |
| **多引擎（远端宿主）**：脑图可接多个后端引擎（`local` + 登记的远端），图随编辑器走、对象留在宿主进程、能力按 `features` 降级 | 契约：`Rez-Docs/Rez_pkg/远端引擎契约_v1.md`；自证：`tools/engine_contract_check.py`（本地 vs 假宿主 18 项）+ `web_probe engine`（下拉/切引擎/远端跑图/切回） |
| **搜宿主的类 + 节点标引擎名**：左栏搜索框会多一节「🧩 类」（搜宿主**已导入模块**里的类，点一下落 `MakeInstance`）、一节「**ƒ 函数**」（2026-10-17 加：同一批模块里的函数，点一下落 `CallFunction`，见 `l_node_engine.md` 的 `functions.find_functions`）与「🔀 别的引擎才有的节点」；不属于当前引擎的节点卡片上标 `🔗 <引擎名>` | `GET /api/engines/nodes-index` + `GET /api/pygraphs/search`（⚠ 不是 `/classes`：那个路径被"类资产列表"占着，同路径 GET 先注册的赢 ⇒ 搜索那条以前从没被调到过）；自证：`web_probe muse`（"local 下标出 muse"与"搜类落节点"断言）+ `web_probe fnsearch`（搜函数 → 落 `CallFunction` → 端口按签名展开） |
| **属性面板：本轮值**（连线字段不再只有"← 节点 x 第 n 个输出"）：一行摘要（类型+条目数+体积）+ 左侧 `▸` 展开 + `⧉` 复制；节点底部有「本轮输出」/`ui` | `web_probe muse`（摘要正则 / 没有铺开的 `<pre>` / 点 ▸ 看全）；`web/src/RunValue.jsx` |
| **混引擎（一张图多台引擎）**：节点上标 `engine` → 跑图按**同引擎连通块**切片，段内交给那台引擎照旧跑（执行器不重造），**段间值走 JSON 交接**；UI：属性面板「引擎」下拉 + 卡片徽标 + 日志"▶ 段 x" | 口径与限制见 `l_mindmap_mmd.md`「混引擎」；自证：`web_probe mix`（绑引擎的卡片徽标 / 面板默认值 / 跑通 / 日志有段）+ `pySchema.test.mjs` 的"engine 不能丢"用例 |
| **节点来源标记 + CSV 表格预览**：`/object_info` 带 `source`（`function`/`class`）→ 节点列表 `ƒ`/`◇`；`Preview(mode="csv")` 把 CSV 画成表格（`ui.table`） | `preview_csv_check.py`（10 项）+ `web_probe muse`（"CSV 预览画出表格"） |

**未完成**：本变更（`mindmap-editor-followups`）已归档且分项全勾；见上面归档目录的 `tasks.md`。
仅两处保留未验（`.fbx` 真资产几何、144Hz 机上的帧预算），判据与补跑开关都写在归档 `tasks.md` 的 S5/验收里。

## 一、现状（2026-10-12 这一轮的基线）

### 1.1 canvas 渲染后端（>150 节点自动启用，P3/P4/P5 之后）

能编辑：点选 / 框选 / 拖节点 / **拖端口连线** / 右键菜单（节点·边·空白）/ 双击居中 / 滚轮缩放 /
小地图导航（可点可拖）/ 分组框拖动（框里的节点跟着走）与缩放 / 循环区凸包（按 `data.hull`）/ 标志位徽标 / 耗时徽标 /
运行中脉冲 / 连线"热边"高亮 / 与 DOM 后端一致的配色 / `Input` 节点标题栏下的**值行**（LOD `zoom ≥ 0.45`，
容器给 `[…] N 项` / `{…} N 键`）。

**指针命中是线性 O(n)**（`CanvasSurface.hit()`，从后往前）—— 大图（>5000 节点）再谈空间索引，
当前**刻意不做**：命中只在指针事件里跑，比每帧绘制便宜一个量级，加索引反而多一份要同步的状态。

### 1.2 2000 节点实测（headless Edge，1410×693）

| 指标 | 数字 |
|---|---|
| 整理布局（`整理布局` 按钮） | **130–220ms**（旧实现：200 轮跑满还收敛不了、剩 13 万对重叠） |
| 稳态每帧 `fillRect` | **1**（网格 = 一次 `createPattern` 填充；旧实现 65% 缩放下 ~7000 次） |
| 稳态每帧 draw call | ~1400 `lineTo`（边，低缩放走直线 LOD）+ ~800 `rect`（可见节点，批成 3 次 `beginPath`） |
| 每帧 JS + 提交耗时 | **0.6ms**（软件光栅环境；真机只会更省） |
| 帧间隔 | **跟着显示器刷新率走**（vsync 型满帧）：60Hz 一次实测 p50 16.7ms / p95 16.8ms；早先一次 p50 7.0ms / p95 7.4ms（144Hz 预算 6.93ms）。**别拿绝对值当指标**，看是否满帧 + 调用次数 |

⚠️ headless 是软件光栅：**帧间隔只能当参考**；调用次数与 JS 提交耗时才是 CPU 侧真数字。
真机（真 GPU + 144Hz 窗口）的对照还欠一次 —— 见 tasks S5。

### 1.3 `Input` 参数（全 python 类型 + 3D 模型）

- 类型是**自由表达式**：`str/int/float/bool/list/tuple/set/bytes/None/Path/object` + 语义标签
  `image/model`，也能写 `list[int]` / `dict[str, int]` / `np.ndarray`（空或不像类型 → 退 `string`）；
- 值**标量存真标量、容器存 JSON 文本**，执行时 `coerce_param()` 转真对象（下游拿到真 `dict`/`list`/`tuple`）；
- 控件按类型给：`bool`→勾选、`int/float`→数字框、容器→JSON 文本域、`None`→只读、`image/model`→选文件；
- `type=model`：上传（**原始 body 流式落盘**到 `models/`，白名单 + 200MB + 执行面闸门）+
  three.js 预览（**懒加载**：主包 +24KB，three/GLTF/OBJ/STL/PLY/OrbitControls 各自成 chunk）。

口径细节见 `Rez-Docs/Rez_pkg/l_node_engine.md` §3.2（3D 模型）/ §3.3（值怎么存）。

## 二、踩过的坑（7 条，别再踩）

1. **容器值不能存数组**：`Input.value` 端口是 `*`，而"连线"的存储约定正是 `[源id, 下标]` 数组 ——
   存 `[1, 2]` 会被当成"连到节点 1 的第 2 个输出"，**值直接丢**，画布还看着像有连线。
   → 容器存 JSON 文本，执行时 parse。
2. **Vite 动态 import 不能用变量路径**（`import(SPEC[kind])`，加 `@vite-ignore` 也不行）：不打包、
   运行时按 URL 请求裸模块名 → **404**（症状：3D 预览永远"载入中"）。加载器必须写字面量分支。
3. **FastAPI 的 `@app.get` 不带 HEAD** → 前端探文件大小吃 **405**；要
   `@app.api_route(..., methods=["GET", "HEAD"])`。
4. **登录闸门对本机直连只放行 `GET/HEAD`**：页面免登录 **≠** 接口免登录。自动化里别去模拟登录表单
   （本机直连的页面**不弹**登录框），要 `POST 127.0.0.1:1027/api/v1/auth/login` 拿 token 再种
   `lugwit_token` cookie。
5. **Playwright 在 low-zoom 的小控件上会卡**：节点缩到 0.5× 时 checkbox 只有 **6.5px**，
   "可见性/稳定性"判定会超时，`force: true` 还会**点空** → 用原生 `el.click()` + 直接设值派发 `input`。
6. **挪代码后必须回读一眼**：把上一行的 `//` 注释和下一行并成一行，会静默注释掉一个 `const`
   （实测整页白屏：`ReferenceError: lastSent is not defined`）。
7. **服务热重启期间**（改了 `.py`）跑浏览器自证会以"没回 JSON / 找不到节点"失败 ——
   先 `wuwo svc status l_mindmap_mmd_server` 确认在线再跑。

## 三、自证命令（可直接抄）

```cmd
:: 浏览器侧：perf / model / types / value / hull 五个场景（headless Edge，不弹窗、不抢焦点；全套 40 项断言）
node rez-package-source/l_mindmap_mmd/999.0/tools/web_probe.cjs
node rez-package-source/l_mindmap_mmd/999.0/tools/web_probe.cjs perf          :: 只跑性能
node rez-package-source/l_mindmap_mmd/999.0/tools/web_probe.cjs value         :: 只验 canvas 值行
set MM_N=300& node rez-package-source/l_mindmap_mmd/999.0/tools/web_probe.cjs perf

:: 真机帧预算（有头窗口 = 真 GPU + 真刷新率；会开一个 Edge 窗口约 1 分钟）
set MM_HEADED=1& node rez-package-source/l_mindmap_mmd/999.0/tools/web_probe.cjs perf

:: 有真 .fbx 时补验 FBX 真预览（仓里那三个 .fbx 现在是 106 字节占位，验不了几何）
set MM_FBX=<真文件路径>& node rez-package-source/l_mindmap_mmd/999.0/tools/web_probe.cjs model

:: 引擎侧：类型转换 + 自由类型 + bytes 的 base64 + 真跑图下游拿到真对象（35 项断言）
wuwor l_node_engine -- python rez-package-source/l_node_engine/999.0/tools/param_types_check.py

:: 引擎侧：把"宿主的普通函数"批量扫成节点（端口/默认值/重名/幂等/白名单/异步/真跑图）
wuwor l_node_engine -- python rez-package-source/l_node_engine/999.0/tools/from_functions_check.py

:: 宿主侧：本机路径受控读口 + models 读口 的四道闸（14 项；探针配不了 env，白名单内正例只能在这里验）
wuwor l_mindmap_mmd -- python rez-package-source/l_mindmap_mmd/999.0/tools/model_gates_check.py

:: 多引擎：契约 v1 本地 vs 假宿主逐条对拍（18 项；不起我们的服务）
wuwor l_mindmap_mmd -- python rez-package-source/l_mindmap_mmd/999.0/tools/engine_contract_check.py
:: 想手动摆一个"远端引擎"给编辑器用：
wuwor l_mindmap_mmd -- python rez-package-source/l_mindmap_mmd/999.0/tools/engine_fake_host.py --port 8731 --token secret

:: 前端纯逻辑（布局防重叠/幂等、参数口径、结构化行、值行摘要、探针脚本本身）
wuwor l_mindmap_mmd -- l_mindmap_mmd_web_test
```

探针的用法与口径见 `rez-package-source/l_mindmap_mmd/999.0/tools/web_probe.md`。
