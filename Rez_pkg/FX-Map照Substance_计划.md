# FX-Map 照 Substance Designer 改 —— 计划（先方案，后动手）

> 2026-10-14。用户口径（原话）：「**fxmap 节点的内部的逻辑和节点样式请完全使用 substance designer 的**，
> 节点竖着连接，每个节点四个输入四个输出端口」。
> 这份文档 = **方案 + 判据 + 分步 + 台账**。口径已拍定（§6），P0 已落地（§4 打勾），
> 后面每步做完就回这儿打勾 + 写实测证据。
> 相关：`子图与FX-Map_计划.md`（现有 `TexFxMap` 是怎么来的）、`l_tex_nodes.md`（值约定 / 算子）、
> `l_node_engine.md`（子图与图内核）。

## 0. 一句话现状 vs SD

| 层 | 现在的样子（`子图与FX-Map_计划.md`） | SD 那边 | 差什么 |
|---|---|---|---|
| 算子 | `TexFxMap`：`mode=quadrant|iterate` + `pattern_graph`（图案**子图**）+ 递进变换 | 只有"一个图案被反复实例化"；**没有** SD 的 Quadrant/Iterate/Switch **三类内层节点**，也没有"根"（**2026-10-14 P0 已补** `FxQuadrant` / `FxIterate` / `FxSwitch`；"根"与 FX 变量上下文归 P1） |
| 引擎 | 子图 = **另一张普通图**（`Input` 喂变量 → `Output` 取结果），多条护栏（深度/成环/指纹） | 内层**不是**一条 FX 链，没有"按象限四叉走一遍"的求值；节点端口数由类写死，SD 的 4 出是"一层树"的语义 |
| 宿主 | 图名内联（`resolve_graph_names`）、双击进内层图、`graph_skeleton` 骨架 | 没有「设为根」、没有图元数据 `root`、没有 FX 变量上下文 |
| 编辑器 | 横向（左→右）连线；节点卡片一行一个手柄 | **竖连**（上入下出）、**4 入 4 出**的端口排布、SD 那种小方块节点样式 |

## 1. SD 的 FX-Map 到底长什么样（事实，不是印象）

出自 Adobe 文档与 Overdraw 的 fBm 教程（<https://www.overdraw.xyz/blog/2018/7/14/creating-fbm-noise-with-the-fx-map-node>，
文中引了 Allegorithmic 官方文档原文）：

- **一个 FX-Map 图里只有三类节点**：`Quadrant`（最常用）、`Iterate`、`Switch`。
- **Quadrant**：「把这一级的图切成四个象限」；**每个 Quadrant = 四叉树的一层（octave）**。
  文中原话：把上一个节点的**四个输出**接到新节点的**四个输入** —— 所以 **Quadrant = 4 入 4 出**。
  为什么不是"每层摆 4 个节点"：4 层就是 4×4×4×4 = 256 个节点，SD 用一个节点"自己知道自己在第几层"来省掉它。
- **Iterate**：「把**右手边接进来的图**重复叠到**左手边接进来的图**上，重复 N 次」→ **左/右两个输入，1 个输出**；
  每轮想变样就配 **Dynamic Function**（逐轮转一点、缩一点）。
  （另一篇教程里说 Iterate 通常要右键 **Set as Root** 才生效 —— 即"根"是**节点属性**，不是固定某个类。）
- **Switch**：两个输入 + `Selector` 设置，选一个出去。
- **节点属性**：每个节点有 `Pattern`（图案来自哪个输入 / 内建图案）与**逐实例**的变换（rotation/scale/…），
  这些"逐实例"的值可以挂 Dynamic Function。
- **FX-Map 自身属性**：`Color Mode`（Grayscale / Color，灰度输入配 Color 模式会**什么都不显示**）、
  `Roughness`（fBm 那套用）、以及若干**全局输入**（`Input Image 0..N` 之类）。
- **求值**：引擎按 quad-tree 走，**从上到下、从左到右**；图上那条链是 **Markov chain** 画法，不是真树。
- **进内层**：双击节点（或 Ctrl+E）进内层编辑器；内层里**节点是竖着连的**（下一个接在上一个下面）。

⚠ 用户说的"每个节点四个输入四个输出"与 SD **只在 Quadrant 上成立**（Iterate 2 入、Switch 2 入）。
这一条列进 §6 待拍板，动手前必须定。

## 2. 现状 → SD 的差距（逐条，可勾）

- [ ] **内层节点族**：没有 `Quadrant` / `Iterate` / `Switch`（也没有 SD 那些 FX 输入节点）。
- [ ] **4 入 4 出的端口语义**：现在节点的端口数由类写死（`RETURN_TYPES`），且连线是"值流"；
      SD 的四个出是"**同一层的四个象限**"，接回下一个 Quadrant 的四个入 = 造出下一层的四个子树 ——
      语义是**树结构**，不是四份并列的值流。
- [ ] **根（Set as Root）**：现在靠子图里的 `Output` 节点；SD 是"图上任意节点右键设为根"。
- [ ] **竖排连线 + 4/4 端口画法**：两个后端（DOM / canvas）都得改，且必须同口径。
- [ ] **FX 变量上下文**：现在靠子图里摆 `Input` 节点接收 `index/number/x/y/size/pos`；
      SD 的逐实例值（Pattern 变换）是**节点属性**上的动态函数，不必先摆 Input。
- [ ] **全局属性**：`Color Mode` / `Roughness` / 全局输入 —— 现在只有 `width/height/builtin/…`。
- [ ] **示例图**：`demo_fxmap` 现在是三条 `TexFxMap` 支路（内建图案 / 图案子图 / iterate），
      摆不出 SD 那种"Quadrant×5 叠 octave"的经典用法。

## 3. 改法（三层，别混）

### 3.1 算子里加一族 FX 节点（`l_tex_nodes`，新文件 `fx_nodes.py`）

都吃/出 `TexImage`（沿用 §二 的值约定），**先只做 CPU、确定性**（同参数两次跑逐像素一致）：

| 节点 | 端口 | 语义 |
|---|---|---|
| `FxQuadrant` | **4 入 `in0..in3`（IMAGE）/ 4 出 `out0..out3`（IMAGE）** | 每出 = 本层的第 i 个象限；入 = 上一层的四个子树（没接就按 `pattern` 参数出图案） |
| `FxIterate` | 入 `background`（左）+ `pattern`（右）+ `iterations` / 逐轮变换 → 1 出 | SD 的 Iterate：把 pattern 反复叠到 background |
| `FxSwitch` | 入 `input0` / `input1` + `selector` → 1 出 | SD 的 Switch（selector 可以是变量） |
| `FxInputImage` / `FxNumber` / `FxColor` | 0 入 / 1 出 | SD 的全局输入在图上就是一个节点。**挪到 P1**：它们要"FX 变量上下文"那条线才有意义，P0 先不做空壳 |
| `FxRoot`（或"根"是元数据，见 §6） | 1 入 | 哪颗是输出 —— **§6 已拍：用图元数据 `root`，不加 `FxRoot` 节点** |

- 变量（`index/number/x/y/size/pos`）**照旧能当端口**，另外新增**节点属性里的表达式**（`$index`、`$x`…）
  给"逐实例变换"用 —— 两条都留，端口给连线派、表达式给手填派。
- 判据（自证，加进 `l_tex_nodes` 那套）：
  ① `FxQuadrant` 四个出**正好是四个象限**（尺寸 + 内容 = 对应入缩到象限大小）、`fx_tile` 拼回的图铺位对；
  ② **顺序 = raster**（out0 左上 / out1 右上 / out2 左下 / out3 右下），链两级不重排；
  ③ 逐档变换/上色生效，且同参数两次跑**逐像素一致**；
  ④ `FxIterate` 轮数变了输出跟着变；`FxSwitch` 按 selector 选对（偶→input0、奇→input1）且缺图**当场报错**；
  ⑤ 结构：`FxQuadrant` 就是 4 出、名字 `out0..out3`（防以后被改坏）。
  （"没有根 → 跑图前 400"是**宿主**那半，归 P1。）

### 3.2 宿主接线（`l_mindmap_mmd`）

- 内层图照旧走**子图内联**那条（`resolve_graph_names` + `run_nested_value`）—— 不新造机制。
- **根**进图元数据（`root: "<节点id>"`），`/validate` 与跑图前都检查"有且只有一个根"；
  编辑器右键菜单加 **「设为根」**（两个后端都要）。
- FX 图里的节点**固定竖排**这件事要有个判据（怎么知道"这是 FX 图"）——
  建议：**图元数据 `kind: "fx"`**（新建骨架/双击进内层图时写上），或"图里有 `Fx*` 节点"即认为是 FX 图。
- 判据：`POST /api/pygraphs/<图>/validate` 对"根缺失/多个根/根指向不存在"逐条报 400；
  跑一张 `Quadrant→Quadrant` 的图 → SSE 里能看到**逐象限**的进度事件。

### 3.3 编辑器（两个后端，必须同口径）

- **竖排**：`FxInterior` 的句柄摆上/下（React Flow 用 `Position.Top/Bottom`；`CanvasSurface` 自己画一份）。
- **4/4 端口**：`in0..in3` 在上、`out0..out3` 在下；端口名可关（SD 内层不看名字，只数格子）。
- **样式**：小方块节点（SD 那种"一个小框 + 4 个上格 + 4 个下格"），标题只留类名简写。
- 判据：探针加 **`fx` 场景**（两后端各跑）：节点数 / 边数 / 每个节点 4+4 个句柄 /
  **句柄在卡片上/下**（用 `getBoundingClientRect` 断言，别只看类名）/ 右键「设为根」生效并落盘。
- ⚠ 与既有铁律一致：**两处 UI 都得实现，不许只加一边**；改完**重建产物并开一次页面**（白屏坑见
  `子图与FX-Map_计划.md` §四.8）。

### 3.4 示例图与文档

- `demo_fxmap` 重做：外层三条支路（内建图案 / 引两张 **FX 图**），内层两张 FX 图分别演示
  `FxQuadrant` 两层链 + `FxSwitch`（`index` 驱动）与 `FxIterate`（5 轮当根）；双击建的骨架 = 一张 FX 图骨架。
- `l_tex_nodes.md` 加 §7.5（FX 节点族 + 根口径 + 示例图，节点数 38→42）；`l_node_engine.md` 补
  `ROOT_WRAP` / 图级 `root`；本文件随进度打勾；`AGENTS.md` / `Rez-Docs/README.md` 补索引。

## 4. 分步（每步可单独验收）

| 步 | 内容 | 判据 | 预估 |
|---|---|---|---|
| ~~**P0**~~ ✅ | `fx_nodes.py`：`FxQuadrant`（4 入 4 出）/ `FxIterate` / `FxSwitch` + `ops.fx_instance_transform`（与 `TexFxMap` 共用一份，别再两边写） + `ops.fx_tile`（SD 的 unwrap） | **已过**：`wuwor l_tex_nodes -- l_tex_nodes_selftest` → `SELFTEST PASS`，新组"FX 内层节点"①–⑤ 全绿（其余 9 组不回归、33 张示例图照旧）；宿主 `object_info` 里三个类都在（`FxQuadrant` = 4 个 `IMAGE` 出、名字 `out0..out3`） | 半天 |
| ~~**P1**~~ ✅ | 根（图元数据 `root` + 引擎 `graph_prompt` 把它**收成 `Output`** + `validate`/`run` 三类坏根）+ FX 变量上下文 | **已过**：引擎自证第 29 组（多出→按 `ROOT_WRAP` 收口 + 单出直接接 + 无 root 走老路 + 三类坏情形当场抛）；宿主真跑 19/19（`FxQuadrant` 当根 → validate ok / run 200 且 SSE 里有合成的 `__output__`；`root` 指不存在、`kind:"fx"` 缺 root → validate 报 `graph_root`、run **400**；`TexFxMap` 引一张**带 root 的 FX 图** → 内层 `fx/__output__` = `TexImage(32x32 rgb)`，变量 `index` 进得去）。⚠ **右键「设为根」+ 卡片上的「根」标记挪到 P2**（跟竖排/4-4 端口同一个 UI 批次 + 同一个探针场景，省得两趟改两个后端） | 半天 |
| ~~**P1′**~~ ✅ | **老图迁移**：`TexFxMap` 的内层子图 → FX 口径；迁移器落清单 | **已做**：新增 `l_tex_nodes/fxmap_migrate.py`（别名 `l_tex_nodes_fxmap_migrate`）：扫全库 → 只迁**元数据**（给老内层图加 `kind:"fx"` + `root`=它的 `Output`，**逐像素不变**）→ 对拍不过**不写盘**、先备份到 `<图目录>/_backup/fxmap_migrate/<时间戳>/`；默认 dry。**实跑结论：扫 36 张图，可迁 0 / 需人看 0 / 老口径清单 0** —— 库里引用的内层图**已经是 FX 图**（用户双击 `demo_fxmap#1` 建出的 `demo_fxmap_fxmap` 就是新骨架，且落在**同工程** `Functions/` 里）。⚠ 明确**不自动**把老 `mode=quadrant\|iterate` 的分解改写成 `FxQuadrant` 链 / `FxIterate`（那会**改像素**，是"重做"不是"迁移" —— 示例图 `demo_fxmap_pattern` 就是手工重做的样子），这类只在报告里列清单给人定 | 半天 |
| ~~**P2**~~ ✅ | 编辑器看得见：**竖排**端口（输入贴上边、输出贴下边）+ **4 入 4 出** 端口条（两个后端同几何）+ 卡片「🌳 根」+ 右键「设为根」；双击 `TexFxMap` 建的骨架改成真 FX 图 | **已过**：`web/src/PyNode.jsx`（竖排时行里不画 Handle，改由上/下两条 `.pyg-hrow` 均分；卡头 `.pyg-tag-root`）、`CanvasSurface.jsx`（端口方块/命中/连线锚点全按 `isVertical` 切，与 DOM 同一份几何）、`style.css`、`pySchema.js`（`kind`/`root` 进 `normalizeGraph`/`toStorage` 白名单）、`PyGraphEditor.jsx`（`data.vertical`/`data.isRoot` + 菜单项）、`l_tex_nodes/nodes.py`（`GRAPH_SKELETON` = FX 图：`kind:"fx"` + `root` + `FxQuadrant`）。判据：`node tools/web_probe.cjs fx` → **17/17**（DOM + canvas 两后端；canvas 用"端口方块横排 + y 铺开"当几何代理，理由写在 `caseFx` 注释里）、`types` 20/20、`assets` 33/33（不回归）；骨架当图 `validate ok` / `run 200`（`__root__` → `__output__` = `TexImage(256x256 rgb)`） | 一天 |
| ~~**P3**~~ ✅ | 示例图照 FX 口径重做（`demo_fxmap` 三条支路 + 两张 **FX 图** 内层）+ 骨架改 FX（挪到 P2 做了）+ 文档/索引 | **已过**：`pro/Functions/demo_fxmap_pattern`（FX 图：变量 → `FxSwitch`（`index` 驱动）→ `q1` → 四出接 `q2` 四入 → `root=q2`）、`pro/Functions/demo_fxmap_iterate`（FX 图：`FxIterate` 5 轮当根）、外层 `demo_fxmap` 三条支路引它们。自证：三张 `validate ok` + `run 200`（`incremental:false`）+ `node_errors` 空 + `media/demo_fxmap.png` 落盘 + `pro/…` 别名也跑得通（13/13）。⚠ **那是 2026-10-14 的口径** —— 2026-10-16 起**入口地址**（URL/API 的首段）写 `pro/…` 一律 **400**，只有**图内引用**（`sub.flow` / 这些示例图里互相引用的 `pro/Functions/…`）仍可写它；入口请写真工程名。文档：`l_tex_nodes.md` §7.5（FX 族 + 根 + 示例图，节点数 38→**42**）、`l_node_engine.md`（`ROOT_WRAP` / 图级 `root` 一段） | 半天 |

顺序不能换：**P0 没自证就不许动 P2**（不然"样式改了但算子是错的"，白做工）；
**P1′ 没过就不许删老口径代码**（老图还在磁盘上，迁移器是它们的唯一出路）。

## 5. 风险与踩坑预案

1. **语义错位**：SD 的 4 出是"象限"不是"值流"。做错就是"四个一样的图"。→ P0 判据 ① 钉死。
2. **老图迁移是这一轮最危险的一步**（§6 第 5 条已拍"换成 FX 图 + 自动迁移"）：
   库里已有 `demo_fxmap` / `demo_bamboo_weave` / 调色板那 33 张示例图在吃老口径。
   → 迁移器必须**逐张真跑 + 与原输出逐像素对拍**，对不上的**不许覆盖**（只报清单，人来定）；
   迁移前把老图备份到 `<图目录>/_backup/fxmap_migrate/<时间戳>/`。
3. **两个后端漂**：`子图与FX-Map_计划.md` §四.8 的白屏坑 + 探针"两个后端各一份实现"。
4. **结构指纹**：FX 图内容改了必须重算 → 复用现成的"整体 sha1"（`register_hashed_inputs`），别用 `_safe_repr`。
5. **4/4 端口与现有容器渲染器**：端口数变了，值控件（`l_py_ui` 那套）要不要跟着画？**建议不画**（IMAGE 只在卡上留小图）。
6. **别让"竖排"变成全局开关**：只对 FX 内层图生效（见 §6 第 3 条），否则整仓图的观感一起变。
7. **宿主 discover 必须早于"建 prompt"**（P1 实测踩到）：图级 `root` 要按**节点类**问"几个出"，
   而 `_node_engine()`（里面 `discover_nodes("l_tex_nodes.nodes")`）原本在端点的**后面**才调用 ——
   于是**第一次**请求答案变成"根节点 q（FxQuadrant）没有输出，当不了根"（类还没进注册表）。
   → 修法：`_graph_to_prompt` 开头先 `_node_engine()`（幂等）；引擎那边把"类没注册"与
   "这个类没有输出"分成两句不同的错（不然同一条假消息掩盖两件事）。
8. **`Output` 回的是 `{"result": (值,), "ui": …}`**（带 ui 的节点形状）：拿它的输出要**先拆字典再索引**
   ——`dict[0]` 是 `KeyError`（`l_tex_nodes` 与引擎自证都踩过，两处都留了注释）。
9. **改了 `l_tex_nodes`（骨架在那边）必须 `wuwo svc reload l_mindmap_mmd_server`**：宿主进程里
   那份模块是启动时 import 的，不 reload 就"改完还是老骨架"（P2 实测踩到：骨架里还是 `gen`/`out`）。
10. **canvas 探针的"点一下触发重画"别点左上角**：`x+12, y+12` 落在**标尺角**上，事件不到画布 →
   不重画 → `__rects`/`__txt` 全空 → 两条**假失败**（"横排 0 条 / 没抓到 🌳 根"）。点空白处（右下角）才对。

## 6. 拍定了（2026-10-14，用户逐条确认；下表是**结论**，不再讨论）

| # | 项 | 结论 |
|---|---|---|
| 1 | 端口数 | **全按 SD**：`FxQuadrant` **4 入 4 出**；`FxIterate` **2 入 1 出**（左底图 / 右图案）；`FxSwitch` **2 入 1 出**（+ `selector`） |
| 2 | 根 | **图元数据 `root`（节点 id）+ 右键「设为根」**（DOM 与 canvas 两个后端都要）；`/validate` 与跑图前检查"有且只有一个、且指向存在的节点" |
| 3 | 竖排范围 | **只 FX 内层图竖排**（图元数据标 `kind: "fx"`）；其余图照旧横排 —— 别做成全局开关 |
| 4 | 外层 `TexFxMap` 卡片 | **不动**：它只当"进内层 FX 图"的入口，SD 那套全在内层 |
| 5 | 老 `mode=quadrant\|iterate` | **换成 FX 图 + 老图自动迁移**：老图的内层（或内建图案那条路）改写成 `Quadrant` 链；迁移必须"逐张真跑、与原输出对拍"（见 §4 P1′） |

## 7. 验证命令（改完照跑）

```cmd
wuwor l_tex_nodes  -- l_tex_nodes_selftest              :: FX 节点组（P0 判据 ①–⑤）
wuwor l_node_engine -- l_node_engine_selftest           :: 子图 / 根 / 指纹那一层
wuwor l_mindmap_mmd -- l_mindmap_mmd_web_build           :: 改过 web/ 必须重建产物
wuwo svc reload l_mindmap_mmd_server                :: 改 l_tex_nodes 不会热更宿主，必须手动
node rez-package-source/l_mindmap_mmd/999.0/tools/web_probe.cjs fx    :: 竖排 + 4/4（两后端）
```

宿主真跑（本机直连 GET 免登录，POST/PUT 要 `Authorization: Bearer <token>`，
token = `POST http://127.0.0.1:1027/api/v1/auth/login`（`admin01` / `666`））：

```cmd
curl -s -X POST "http://127.0.0.1:8110/api/pygraphs/lugwit_agent/Functions/demo_fxmap/validate" ^
     -H "Content-Type: application/json" -H "Authorization: Bearer <token>" -d "{}"
```
