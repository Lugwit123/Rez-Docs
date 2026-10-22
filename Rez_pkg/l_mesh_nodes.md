# `l_mesh_nodes` —— 程序化 3D 建模（`l_node_engine` 的**引擎外插件**）

> 2026-10-21 从 `l_node_engine` 核心**拆出来**成独立包（照 `l_tex_nodes` 的分层），
> 引擎只保留接口：注册表 / 插件壳 / 值契约。**插件形态 B**：宿主 `requires` 里没有它，
> 启用时按 rez 包名解析路径再 import。

## 1. 包结构

```
l_mesh_nodes/999.0/
  package.py                      requires = [python, numpy, numba, l_node_engine]
    alias l_mesh_nodes_selftest            几何算子自证（python -m l_mesh_nodes.mesh3d）
    alias l_mesh_nodes_plugin_selftest     插件层自证（48 节点注册 / 真跑图 / 上限 / 停用摘干净）
    alias l_mesh_nodes_test                回归网（tests/test_mesh3d.py，147 条）
    alias l_mesh_nodes_gpu_bench           numpy / numba / torch 基准
  src/l_mesh_nodes/
    mesh3d.py            纯 numpy 几何算子（图元/细分/置换/雕刻/倒角/UV/绑定/散布/重拓扑…）
    mesh3d_plugin.py     48 个节点类 + `Mesh3DPlugin`（**导入即注册**）
    mesh3d_gpu.py        GPU 可选后端（torch 优先 / cupy 备选；插件「依赖目录」里有才用）
    mesh3d_njit.py       CPU JIT（numba `@njit`：taubin + remesh 栅格化）
    plugin_selftest.py   插件层自证
  tests/test_mesh3d.py   147 条回归网
  tools/mesh3d_gpu_bench.py
```

**依赖纪律**：`numpy` / `numba` 写在 `requires`（本包是"插件可以重依赖"的那一侧）；
`manifold3d`、`torch`、`cupy` 一律走插件面板「依赖目录」（`pip --target`），宿主 `requires` 里永不出现。

## 2. 引擎侧接口（核心只剩这些）

| 接口 | 位置 | 用途 |
|---|---|---|
| `register_node` / `unregister_source` | `l_node_engine.registry` | 节点进表 / 按 `NODE_SOURCE` 摘干净 |
| `Plugin` / `register_plugin` | `l_node_engine.plugins` | 插件壳（字段/状态/`apply`） |
| `plugins.load_external()` | 同上 | **引擎外插件加载**：读声明 → rez 解析 → `import` |
| `plugins_rez.resolve(pkg)` | `l_node_engine.plugins_rez` | `wuwor <pkg> -- python …` 取 `PYTHONPATH`（进程内 + 磁盘缓存） |
| 值契约 | — | `MESH` 端口上是**普通 dict**（`{kind, format, vertices, faces, attrs}`），引擎只透传 |

## 3. 接入方式（形态 B）

`<插件配置目录>/external_plugins.json`：

```json
[{"id": "mesh3d", "module": "l_mesh_nodes.mesh3d_plugin", "rez": "l_mesh_nodes",
  "deps": ["C:\\Users\\<你>\\.lugwit\\main\\l_node_engine\\mesh_deps"]}]
```

加载链路：
1. 引擎进程启动时 `plugins._prefetch_external()` **后台线程**预热（有声明才起）；
2. `load_external()`：`plugins_rez.resolve("l_mesh_nodes")` → 跑一次
   `wuwor l_mesh_nodes -- python <打印 sys.path>` → 挑出本包 payload → 追加 `sys.path` 末尾；
3. `import l_mesh_nodes.mesh3d_plugin` → 模块末尾 `register_plugin(Mesh3DPlugin())`；
4. 宿主照旧 `bootstrap()` / 面板「重载」→ **48 个节点**进节点库。

缓存与失败：
- 进程内缓存 + 磁盘缓存 `<配置目录>/_rez_paths.json`（7 天；路径不存在就重解析）；
- 解析/import 失败**不抛**：原因进 `EXTERNAL_STATE["errors"]`（面板/`l_node_engine_selftest` 能看到），宿主照常跑、只是少一块节点；
- 首次启动 `_rez_paths.json` 为空时要跑一次 wuwor（实测约 3 s；之后秒开）。
  ⚠ 首个「插件列表」请求在**冷启动**时仍会等预热完成（实测 12–15 s，其中大头是 comfy 插件那 65 个节点），
  第二次 4 ms —— 这是既有行为，不是本次拆包引入的。

## 4. 自证

```cmd
wuwor l_mesh_nodes -- l_mesh_nodes_selftest          :: 几何（mesh3d: OK）
wuwor l_mesh_nodes -- l_mesh_nodes_plugin_selftest   :: 插件层（49 节点/真跑图/上限/停用）
cd rez-package-source\l_mesh_nodes\999.0 && wuwor l_mesh_nodes -- python -m unittest tests.test_mesh3d
wuwor l_node_engine -- l_node_engine_selftest        :: 引擎侧（已不含 mesh3d，只验外部插件接口）
```

## 5. 毛发实例化（`MeshScatter(instances=true)` + `MeshBakeInstances`，2026-10-22）

- **值**：`{"kind":"instances","format":"mesh3d/inst1","template":<网格>,"matrices":[[16 个数],…],"count":N}`
  —— 只发「模板一份 + N×16」，不再按件复制顶点。
- **宿主**：`mesh_preview` 转成 `{kind:"instances", template:{…}, matrices:[…]}`；`MeshViewer` 建
  `THREE.InstancedMesh`（`setMatrixAt` + `frustumCulled=false`），`canvas.dataset.instances` = 件数（探针判据）。
- **算子层不收窄**：`mesh_of()` 见到实例值会**自动烘**（等于旧行为），要显式控制就用 `MeshBakeInstances`。
- 实测（200 件 × 24 顶点柱子）：payload 49 KB / `data-instances=200`（若按真几何发，是 4 800 顶点 ≈ 3 倍大小）。
- 什么时候该烘：要 `MeshSave` / `MeshSkin` / `MeshRemesh` / 布尔 这些**非渲染**通道时；只看/摆场景时别烘。

## 6. 视口点选（`MeshSelect.picked` + 3D 视图「🖌 视口点选」，2026-10-22）

### 6.0 视口 gizmo（2026-10-22，先于点选上线）

用户口径：「把'盲调数字'变成'手动物体'」。3D 视图那排勾 **✥ 视口拖拽** + 选目标节点（`MeshTransform` / `MeshSculpt`）+ 选模式（**平移 / 旋转 / 缩放**）：

- **点住模型拖** = 平移那个节点（`MeshTransform` 写 `tx/ty/tz`，`MeshSculpt` 写 `cx/cy/cz`，都是"当前值 + 位移"累加）；
- 三个模式共用同一套 three `TransformControls` 手柄（箭头 / 圆环 / 方块）做**轴向锁定**；松手时按模式回：
  平移回 `dx/dy/dz`、旋转回 `drx/dry/drz`（角度）、缩放回 `dsx/dsy/dsz`（比例）；
- 落地：平移加、旋转加、**缩放乘**（`sx/sy/sz`）—— 旋转/缩放只对 `MeshTransform` 生效（`MeshSculpt` 只有中心）；
- **抬手才结算**（中途重跑太慢），然后走 `onGeoPick({op:"translate"})` → `mutate` → `geoPending` 触发重跑；
- 拖的时候临时把 OrbitControls 关掉（否则会一边转视角）。

实测：合成 `pointerdown/pointerup` 拖 90px → `✥ 位移 (-0.271, 0.545, -0.271) → 节点 2`，
落盘 `tx -0.30 → -0.571`、`ty 0.78 → 1.325`。⚠ 在曲面拖时三个分量都会变（沿视线方向也位移）。
旋转/缩放的**手柄渲染**已确认（圆环/方块），但 headless 里拖不动那几条细环 —— 这条只做了目视确认。

**问题**：`MeshSculpt` 每笔要手填 `cx/cy/cz`、`MeshSelect` 只能按方向/盒子/球筛 —— 想"雕哪儿就是哪儿"没法做。

**做法**（三处各一小块，都不新增节点）：

| 层 | 改动 | 判据 |
|---|---|---|
| 算子 `mesh3d.py` | `SELECT_MODES` 加 `picked`；`select_faces_value(..., picked="")`、`parse_index_list("12, 40, 90-94", limit)`；`sculpt(..., points=[...])` 多笔心（权重取并集） | `tests/test_mesh3d.py::SelectionTest::test_picked_faces` / `test_parse_index_list` / `SculptLatticeCreaseTest::test_sculpt_path_ridge` |
| 插件 `mesh3d_plugin.py` | `MeshSelect` 多一个 `picked` 文本参数（`mode` 列表自动含 `picked`）；`MeshSculpt` 多一个 `path` 文本参数（`x,y,z; x,y,z`） | `l_mesh_nodes_plugin_selftest`（49 节点） |
| 宿主前端 | `MeshViewer` 加 `pick`/`onPick`（射线拾取 + 笔刷圈 + 框选方框 + **按住拖采样**）；`Viewer3DPanel` 加一排「🖌 视口点选」；`PyGraphEditor.onGeoPick` 写节点参数并重跑 | `canvas.dataset.pick/picks/pickStroke/pickFace/pickPoint/pickNormal` |

**坐标与面号**（两条必须一起看）：模型在视口里被**居中过**（`object.position.sub(center)`）⇒
引擎坐标 = 命中点世界坐标 **+ center**；面号 = three 的 `faceIndex`，因为几何按引擎 `indices` 原序装进
`BufferGeometry` ⇒ 与 `MeshSelect(picked=…)` 认的编号**同一个**。

**三种用法**：
- 雕刻（单击）：`onGeoPick({op:"sculpt"})` → 写 `MeshSculpt` 的 `cx/cy/cz` + `dx/dy/dz`（法线）+ `radius`；
- **拖一笔**（按住左键拖）：屏幕上每走 8px 采一个点（`hitAt` 打射线），抬手时整串点交给
  `op:"sculptPath"` → 写 `MeshSculpt.path`（`x,y,z; x,y,z`），引擎 `sculpt(points=…)` 一次跑完；
  **权重取并集** `1 - Π(1-w_i)`（求和会在采样点密集处把面雕穿），`pinch` 取加权质心、
  `flatten` 取权重最大那个点的平面；
- **新开一笔**（面板那个勾，默认开）：`op:"sculptNew"`（拖也认）→ **建一个 `MeshSculpt` 接在目标后面**，
  参数（`mode`/`strength`/`falloff`/`symmetry`）照抄目标；原来指向"目标第 0 路"的下游连线
  **改接到新节点**上（视图/装配才不会漏这一笔），落点走 `freeCell`（往右下挪，不压别人卡片）。
  返回新 id，面板据此把"下一笔写哪儿"切过去 —— 连点就是一条链（实测 `1 → 2 → 4 → 5`，
  `3d_view` 一路跟到链尾）；
- 选面：`onGeoPick({op:"faces"})` → 追加到 `MeshSelect.picked`（并把 `mode` 切成 `picked`），
  `Shift`+点 = 只留这一次；框选（拖框）内部是"方框里撒 10px 采样点打射线"，与点选同一条判据。

**踩过的六条**：
1. **跑图必须等 `version` 变**：`mutate` 只是 setState。早先甩一个 `setTimeout(…, 120)` 猜"应该生效了"，
   实测 React 有时还没把新节点放进 `graph` ⇒ `doRun` 里 `doSave` 把**旧图**写回磁盘
   （界面多出来的那一笔，刷新就没了）。现在走 `geoPending` + `useEffect([version])`：
   等新 `graph` 真的落地再跑。「⚡ 实时」开着时不自己跑（那条 effect 700ms 后会跑）。
2. **`pick` 不能进 `MeshViewer` 的 `key`**：进 key = 每次开/关点选都重建视口（闪一下、相机回初始位）。
3. **目标节点的默认值别取"第一个"**：早先 `choices[0]`（id 最小）⇒ "新开一笔"插到了**链中间**
   （`2 → 新节点 → 4`，看着像倒序）。现在顺序是 手选 > **3D 视图此刻在看的那个**（`meshNode`）> id 最大的。
4. **别在"重跑后列表刷新"时把 `node` 清回空**：`/meshes` 是跑完才更新的，新节点第一次回来时列表还是
   上一轮的 ⇒ 一 reset 就清空，而 `meshNode` 那条 effect 有"值没变就不重设"的守卫 → 面板永久停在
   "还没有可看的模型"（实测踩到，已删那条 reset；真没几何由 `err` 分支明说）。
5. **`sculpt` 的 `direction` 零向量别一律报错**：拖一笔时采样点不一定带法线 ⇒ 以前 `inflate` 也会炸
   `sculpt：direction 不能是零向量`（整条链红）。现在只有 `flatten`/`grab` 要它，别的模式零向量回落 (0,1,0)。
6. **面号是"当前这份几何"的编号**：中间插了细分/焊接/重拓扑就得重选（越界号静默丢掉，
   全越界才报错并提示"这组号是上一版网格点的"）。

实测（本机，2026-10-22）：`demo_sculpt_pick` 点一下 → 落盘 `cx 0.1985 / cy 0.5306 / cz 0.1986` +
法线 `(0.306, 0.886, 0.349)`，重跑 5 ms；`demo_select_pick` 点选 `picked="2"`、框选 24 个面后
`MeshMove` 把 `maxY` 从 0.70 抬到 0.95（= +0.25，与 `ty` 一致）。

## 7. 卡片形态：建模节点 = **表单卡**（2026-10-22）

用户口径：「模型节点不必显示所有节点端点，**使用表单那种样式**（照 AIVideo 那张表单卡），
输入端点**重置只有一个**（Houdini 那样），仅仅使用属性传递数据，输出端点 1-4 个」。

- **谁算建模节点**：`info.python_module` 以 `l_mesh_nodes` 开头（退路：类名 `^(Mesh|Curve|Attr)[A-Z0-9]`），
  且不是竖排节点（FX 图那种端口在上下边）。两处判据**必须同一份**：DOM `PyNode.jsx` 的 `meshForm`、
  canvas `CanvasSurface.jsx` 的 `meshFormNode`。
- **卡片体**：走 `.pyg-form`（标签一行、控件一行，与 AIVideo 表单卡同一套样式），参数多的节点
  参数区自己滚（`.pyg-form-scroll`，上限 360px），底部一排「▶ 预览」（跑到这个节点）。
  **行里不画任何端口** —— 参数就是表单格子。
- **端口**：只在卡片左右两端（`renderSidePorts`），左边 = **真数据口**（`container === "link"`：
  `MESH` / `SELECT` / `IMAGE` / `MATERIAL`…），右边 = 输出。`MeshSculpt` 因此是 **2 进 1 出**，
  `MeshSphere` 是 **0 进 1 出**，`MeshPose` 是 3 进 1 出。
- ⚠ **已连线的参数必须保留端口**（`data.linked[name]` 也进 `linkInSpecs`）：React Flow 找不到 handle
  会把那条边**丢掉**（`demo_lion` 的 `Time → MeshPose.t` 就是这种，实测）。
- ⚠ 与纯 DOM 的取舍：canvas 后端仍在按行画自己的几何（那边只画 `link` 行的端口，参数行不画）；
  两个后端**各自自洽**（连线端点与命中都用自己那套），视觉上不完全一致。
- 判据（探针）：`.pyg-node.is-formmesh` 存在；卡内 `.pyg-form-row` ≥ 输入参数个数；
  `.pyg-row .pyg-handle` = **0**；`.pyg-cports .pyg-handle` = 数据入参个数 + 输出个数
  （`MeshSculpt` → 3、`MeshSphere` → 1）。

## 8. 按属性选面（`MeshSelect(mode=attr)`，2026-10-22）

Houdini 那种「按属性成组」的最小版：`@Cd > 0.5` 选出一片，喂 `MeshMove` / `MeshSculpt` / `MeshDisplace`。

- 参数：`attr`（属性名，按 `point → vertex → prim → detail` 的顺序找）、`attr_op`（`gt/ge/lt/le/eq/ne`）、
  `attr_value`（阈值）、`component`（三通道取哪一路：`0/1/2`，`-1` = 长度）、`invert`。
- **每个面的代表值**：`point`/`vertex` 取三个角点平均，`prim` 是面值，`detail` 是常量（全选或全不选）。
  所以"按点属性选面"= 那片区域的点都够阈值才会中（不是任一点中）。
- 属性不存在 → 明确报错并**列出这份几何现在有哪些属性**（`select_attr：… 现在有：Cd、wear`）。
- 示例图：`demo_select_attr`（球 → `AttrSet(wear=noise)` → `MeshSelect(attr wear>0.15)` →
  `MeshMove(ty=0.15)` → 3D 视图；带 `autorun`，打开就跑）。实测 bbox 的 `maxY` 0.60 → **0.75**。
- 判据：`tests/test_mesh3d.py::SelectionTest::test_select_by_attr`（标量 / 三通道分量 / prim / invert / 缺属性）。

### 8.1 多材质槽的分工（2026-10-22，`demo_lion3`）

宿主 `MeshViewer` 约定：**槽 0 吃 `3d_view.material` 的贴图，槽 ≥1 只用自己的 `Cd` 纯色 + 哑光**
（建材质时给槽 ≥1 打 `userData.noMaps`，贴图那个 effect 见到就跳过）。
理由：一份 `TexMaterial` 管全模型时，鬃毛那种**薄片贴法线会泛银边**、眼珠/牙也会被刷成毛皮。

打标三条口径（都踩过）：
1. **每个面都要带 `prim.mat`** —— 少一条 `mat_slots()` 直接返回空，等于没有槽；
2. **不能对最终装配整体 `AttrSet(mat=0)`** —— 它会把上游所有槽覆盖成 0（第一次就这么写，5 个槽变 1 个）；
   要打在**各源部件**上（身体/脖子/腿/鬃毛底座/尾巴各一条）；
3. **`MeshScatter` 会重建几何** —— 模板上的 prim 属性带不过来，鬃毛的 `mat=1` 要打在**散布之后**。

`demo_lion3` 实测 5 个槽：`0 毛皮 42976 面 / 1 鬃毛 18240 / 2 眼 1248 / 3 鼻 440 / 4 牙 48`，
`canvas.dataset.matslots = "5"`。

### 8.1b 视口里**每槽可调**（2026-10-22）

`MeshViewer` 多出一排「槽」：每个槽一组 `贴图 / 粗（粗糙度） / 透（不透明度）`，只影响预览、不回写图。
缺省仍是「槽 0 吃贴图、其余纯色哑光」，所以不改也能看；要压高光/做毛发软边就在这里拖。
实测（卡通狮子 6 槽）：槽徽标 `0..5`、12 个滑块；把槽 1 的粗糙度拉到 1 → 画布像素哈希变化（`13a8…` → `cd80…`）。

### 8.1c 毛发壳层：`MeshShell`（第 50 个节点）

`shell(mesh, distance)` = 顶点沿法线推 `d`（负 = 往里收），拓扑与属性不变。做毛的套路：
同一件（例如鬃毛）复制 2~3 份 → `MeshShell(d=0.01/0.02/0.03)` → 各占一个材质槽 →
视口「槽」那排把**外层不透明度**调低 ⇒ 毛茸茸的边缘。判据：
`tests/test_mesh3d.py::SculptLatticeCreaseTest::test_shell`（体积涨/缩 + 属性长度不变）。

### 8.2 导出（2026-10-22，`demo_lion3` 的 ⑪ 段）

- `TexExportImage`（basecolor / normal 两张 PNG，落 `<pygraphs>/media/`）+ `MeshSave(fmt=obj)`：
  `texture` / `bump` **连线喂绝对路径** ⇒ `.mtl` 里 `map_Kd` / `map_Bump` / `bump` 直接指过去，
  Blender / Maya 打开就带贴图。产物：`<pygraphs>/models/lion3.obj|.mtl` + `<pygraphs>/media/lion3_*.png`。
- 多槽会写 `newmtl mat0..mat4`（`Kd` = 该槽 `Cd` 均值）—— 但 **贴图行目前每条材质都写**，
  与宿主「槽 ≥1 不贴图」的约定还不一致（已知差异，要一致得让 `save_obj` 只给槽 0 写贴图行）。
- ⚠ 路径含空格会把 MTL 行截断（本机路径没有）。

## 9. 算法 / 节点 / GPU / 踩坑

细节仍在两处，别在这里重复：
- **机制与字段（权威）**：`Rez-Docs/Rez_pkg/l_node_engine.md` §3.0d″（插件框架 + 40+ 节点的端口/口径）
- **现状 / 待办 / 实测数字**：`Rez-Docs/Rez_pkg/3D程序化建模_交接文档.md`（§1.6 GPU、§1.6b numba、§5 TODO、§3 十条坑）
