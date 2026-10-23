# `l_mesh_nodes` —— 程序化 3D 建模（`l_node_engine` 的**引擎外插件**）

> 2026-10-21 从 `l_node_engine` 核心**拆出来**成独立包（照 `l_tex_nodes` 的分层），
> 引擎只保留接口：注册表 / 插件壳 / 值契约。**插件形态 B**：宿主 `requires` 里没有它，
> 启用时按 rez 包名解析路径再 import。

## 1. 包结构

```
l_mesh_nodes/999.0/
  package.py                      requires = [python, numpy, numba, l_node_engine]
    alias l_mesh_nodes_selftest            几何算子自证（python -m l_mesh_nodes.mesh3d）
    alias l_mesh_nodes_plugin_selftest     插件层自证（72 节点注册 / 真跑图 / 上限 / 停用摘干净）
    alias l_mesh_nodes_test                回归网（tests/test_mesh3d.py，200 条）
    alias l_mesh_nodes_gpu_bench           numpy / numba / torch 基准
    alias l_mesh_nodes_three_addons        生成示例 demo_three_addons（抄 three.js examples/jsm 那批）
  src/l_mesh_nodes/
    mesh3d.py            纯 numpy 几何算子（图元/细分/置换/雕刻/倒角/UV/绑定/散布/重拓扑/QEM/凸包/NURBS/MC…）
    mesh3d_plugin.py     72 个节点类 + `Mesh3DPlugin`（**导入即注册**）
    meshio.py            外部模型读入（OBJ / PLY / STL / glb·gltf）+ 二进制 STL 写出
    gltf.py              glTF 2.0（glb）自写 writer（多槽 / 蒙皮 / 动画 / 八面体法线压缩）
    mesh3d_gpu.py        GPU 可选后端（torch 优先 / cupy 备选；插件「依赖目录」里有才用）
    mesh3d_njit.py       CPU JIT（numba `@njit`：taubin + remesh 栅格化）
    plugin_selftest.py   插件层自证
  tests/test_mesh3d.py   200 条回归网
  tools/mesh3d_gpu_bench.py
  tools/three_addons_demo.py
  tools/shapes_demo.py
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
4. 宿主照旧 `bootstrap()` / 面板「重载」→ **72 个节点**进节点库。

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

### 6.0c 毛发壳层渲染（`3d_view.fur`，2026-10-23）

`3d_view.fur = "layers:4; length:0.05; opacity:0.32; slots:1"` —— 只对**指定材质槽**做壳：

- 取该槽的三角形（按 `mesh.groups` 的槽范围），非索引化成独立几何；
- 按 `layers` 复制、沿**顶点法线**外推 `k*length/layers`，透明度逐层递减（`opacity * (1-(k-1)/layers)`），
  `depthWrite=false` + 双面 → 毛绒边缘；
- 判据：`canvas.dataset.fur` = 层数（实测卡通狮子 `"4"`）。

为什么这算"比 three.js 强"：three.js 要做同样效果得**手写 shell shader 或自建多层几何**；
这里是一个图里的字符串参数，跟着图走、换个人打开一样。

### 6.0d 按槽着色预设（`3d_view.slot_shade`，2026-10-23）

`slot_shade = "1:fuzz; 6:fuzz; 2:skin"` —— 每个材质槽一套**PBR 预设**（只改参数，不动 shader，
跨 three 版本安全）：

| 预设 | 参数 |
|---|---|
| `fuzz` | roughness 1.0 / metalness 0 / env 0.25（毛皮、绒面，压掉高光） |
| `skin` | roughness 0.62 / env 0.55（皮肤那种半哑光） |
| `silk` | roughness 0.22 / env 1.45（丝绸高光） |
| `metal` | roughness 0.28 / metalness 1 / env 1.7 |
| `glow` | 额外给 `emissive = color × 0.55`（自发光） |

视口「槽」那排每槽还有一个下拉可临时改（本地覆盖，不写图）。判据：`canvas.dataset.slotshade`。
实测卡通狮子：`"1:fuzz,2:skin,6:fuzz,7:fuzz"`（鬃毛+两个毛壳绒面、口鼻皮肤）+ `fur=4` 层壳 + 8 个槽。



### 6.0e 渲染档：材质族 / 后处理 / 投影（2026-10-23）

用户口径：「**我们再 vs 一下 threejs**」→ 逐目录对完，结论是"造形状我们赢、看形状 three 赢"，
最扎眼的是**渲染**：64 个建模节点产出的东西，最后只有一个 `MeshStandardMaterial` 在显示。
所以这一档照 three 补齐（**都用它的现成模块，不自己写 shader**）：

| `3d_view` 入参 | 口径 | 判据（画布 dataset） |
|---|---|---|
| `mat_type` | `standard`（默认）/ **`toon`**（`MeshToonMaterial` + 3 阶 `DataTexture` 渐变，暗部留 0.42 不压死）/ **`physical`**（`MeshPhysicalMaterial`） | `dataset.mattype` = 真类名（`MeshToonMaterial` / `MeshPhysicalMaterial`） |
| `slot_type` | 按槽换档：`1:toon; 2:physical`（**槽号**，不是材质数组下标） | `dataset.mattype` 出现多个类名（逗号分隔） |
| `sheen` / `clearcoat` | 0~1，只对 `physical` 生效（毛皮边缘绒光 / 清漆层） | `dataset.sheen` / `dataset.clearcoat` |
| `post` | `bloom:0.35; outline:2; ao:0.8`（0/缺 = 关）→ `EffectComposer`：`RenderPass → SSAO → UnrealBloom → OutlinePass → OutputPass` | `dataset.post` = `bloom,outline` 这种标签串；全关 = `off` |
| `shadow` | `1` = 主光当太阳（`castShadow` + 阴影相机按 bbox 罩住 + 2048 贴图），网格投射/接收 | `dataset.shadow` = `1` / `0` |

- **关掉后处理时是零开销**：`composerRef` 置 null → `tick` 直接 `renderer.render`（不是"跑一条空链"）。
- **面板「🎨 渲染」折叠条**：材质档 / sheen / 清漆 / 泛光 / 描边 / AO / 投影 —— 改的是**本地覆盖**（不写图），
  要跟着图走就写进 `3d_view` 节点（同一套参数）。
- **卡通狮子已切**（`demo_lion_cartoon` 的 `130`）：`mat_type=physical` + `slot_type="1:toon; 2:physical"`
  + `sheen=0.35` + `clearcoat=0.18` + `post="outline:1.5; bloom:0.12"` + `shadow=1`；
  实测整图 154 节点 `node_errors {}`。
- **自证**：探针场景 `render`（`node web_probe.cjs render`）**7/7** —— 含"整轮无 JS 报错"这条。

**踩到的三条**：
1. **`new klass(kind)({…})` 少一层括号** ⇒ 运行期 `TypeError: et is not a constructor`（压缩后的名字），
   React 整块面板崩掉 → 画布 dataset **全空**、探针报"参数没传进去"。看 `JS 报错：…` 那一行才知道是崩了，
   不是没传。
2. **"看文件"那条分支也得有渲染档**：`Viewer3DPanel` 有两条 return（节点几何 / 模型文件），
   我第一版只给节点那条加了 `{renderBar}` ⇒ 拿 glb 试的时候面板上根本点不到「🎨 渲染」。
3. **槽号 ≠ 材质数组下标**：`slot_type` 要按 `mesh.mats[idx].slot` 反查（与 `slot_shade` 同一套口径），
   直接拿下标当槽号在"槽不连续"的图上会串档。

### 6.1 拖动中的实时预览（2026-10-23）

顶点编辑与 gizmo 都改成**拖动中即时改画面**（改 `geometry.position` / `object` 的 transform），
**松手才撤预览 + 写参数 + 重跑** —— 这条补的正是"手感不如 three.js 手搓"的最大短板。
判据：`dataset.vertexpreview`（拖动中递增，实测 10 次 move → `"10"`）与 `dataset.gizmopreview`。

### 6.0b 顶点编辑（`MeshPointMove` + 「✱ 顶点编辑」，2026-10-23）

用户口径：「手捏」——把 three.js 那种"选点拖动"搬进来，但**结果仍然写进图**（可复现、可回滚）。

- 节点 `MeshPointMove`（第 54 个）：`moves` 文本 `12: 0.1,0,0; 40: 0,0.2,0`（下标:`dx,dy,dz`），
  同下标累加；越界/格式错明确报错。算子 `move_vertices()`，判据
  `tests/test_mesh3d.py::SculptLatticeCreaseTest::test_move_vertices`（只动指定点 / 累加 / 空串原样 / 越界报错）。
- 视口：「🖌 视口工具」里勾 **✱ 顶点编辑** + 选目标节点 → 画布画出**点云**（`PointsMaterial`，绿色 4px）；
  点住一个点拖动，**松手**才把**像素位移换算成世界位移**（按该点深度、fov、视口高）写进 `moves` 并重跑。
- 判据：`canvas.dataset.vertexedit = "on"` + 拖完 `dataset.vertexmoves` 递增 + 图文件里出现 `moves`。
  实测（`demo_point_move`）：拖 40px → `165: 0.0521,0.0740,-0.1030`。
- **拖动中实时预览（2026-10-23）**：拖的过程里**直接改顶点位置**（点云 + 网格本体一起改），
  松手才撤掉预览、写参数、重跑 —— 以前拖动时画面完全不动，这是"手感不如 three.js 手搓"的最大一条。
  判据：`dataset.vertexpreview` 随 pointermove 递增（实测 10 次移动 → `"10"`），
  松手后 `dataset.vertexmoves` = 1（只写一次）。gizmo 那条同理（`dataset.gizmopreview`）。
- ⚠ 踩过两条：① `vertexEdit` 传**内联对象**会让 effect 每次渲染重建（点云永远刚建就拆，`dataset` 恒 `off`）
  —— 要传 `true/null`；② 生成几何**没有 uv** 时 `meshesRef` 是空的（那是贴图清单），形状源要退回 `rootRef`。



### 6.0 视口 gizmo（2026-10-22）

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

**图里指定"每槽用谁的材质"**（2026-10-23）：`3d_view.slot_mat = "1:128; 6:128; 7:128"`
（`槽号:材质节点 id`）→ 宿主把那个节点的 `basecolor/normal/roughness` 三张挂到该槽上，
没写的槽退回「槽 0 吃全局材质，其余纯色」。判据：`canvas.dataset.slotmaps`（例：`1,6,7`）。
卡通狮子就用它给鬃毛+两个毛壳槽单独配了深色毛皮材质（节点 128）。

### 8.1d three.js 对齐件（2026-10-23，53 个节点）

补齐 three.js 几何 API 里我们没有的那几样（判据：`tests/test_mesh3d.py::ThreeParityTest` 4 条）：

| 节点 | 对应 three.js | 口径 |
|---|---|---|
| `MeshTorusKnot` | `TorusKnotGeometry` | `p`/`q` 缠绕次数 + `radius`/`tube`/`segments`/`sides`；用 `_frames` 平行移动框架，绕向稳 |
| `MeshPolyhedron` | `PolyhedronGeometry` | `tetra / cube / octa / dodeca / icosa`；**三角面由"最小边长的三元组"现算**（不手抄面表），十二面体走二十面体的对偶 |
| `MeshExtrudeShape` | `ExtrudeGeometry` | 2D 轮廓（`x, y` 用 `;` 分隔）+ `depth` 沿 Z 挤出 + `bevel`（内侧再加一圈收边）；耳切法三角化 |
| `MeshSweep.radius_profile` | `TubeGeometry` 的变径 | `1; 1.6; 0.8` 逐点倍数线性插值（肌肉/腿那种中段粗两头细） |

⚠ 踩过的坑：我给 2D 三角化取名 `_triangulate`，**撞了 `new_mesh` 里那个三维多边形三角化** →
`box()` 都造不出来、142 个用例一起红；现在叫 `_triangulate_2d`。
⚠ 正多面体顶点表里的 φ 是四舍五入的 1.618，判"等长边"的容差必须给到 0.5%（严格 1e-6 时二十面体只认出 8 个面）。

### 8.1c 毛发壳层：`MeshShell`（第 50 个节点）
`shell(mesh, distance)` = 顶点沿法线推 `d`（负 = 往里收），拓扑与属性不变。做毛的套路：
同一件（例如鬃毛）复制 2~3 份 → `MeshShell(d=0.01/0.02/0.03)` → 各占一个材质槽 →
视口「槽」那排把**外层不透明度**调低 ⇒ 毛茸茸的边缘。判据：
`tests/test_mesh3d.py::SculptLatticeCreaseTest::test_shell`（体积涨/缩 + 属性长度不变）。

### 8.1e 四边配对 `MeshQuads`（55 个节点，2026-10-23）

**先给结论**：**"真·四边重拓扑"在我们现在这套数据模型里做不了** —— `new_mesh()` 会把面表
**一律三角化**（`_triangulate(faces)`），全库 `_arrays()` 有 **53 个调用点**、另有 ~20 处
`f[:, 0..2]` / `*3` 的逐角假设；要让四边形一路活着，等于换掉面表示（一次性重构，不是加节点）。

所以落的是**能马上用的那一半**：`MeshQuads` = **四边配对**（quad-dominant）

- 算法：边→相邻面，按"法线夹角"从最平的一对开始贪心配成四边形（`angle` = 硬边阈值，超过不配）；
- 值里同时给两份拓扑：**`quads`**（真四边形，4 个索引）+ `faces`（扇形三角化）——
  下游算子零改动（它们只看 `faces`），而 **`MeshSave(fmt=obj)` 见到 `quads` 就写 `f a b c d`**，
  Blender 一进去就是四边网格；
- 好处：对角线由配对决定（跨整片、同向），比随机三角化光顺得多；
- 判据：`tests/test_mesh3d.py::QuadifyTest`（立方体 12 三角 → **6 四边形**、四边合法、
  OBJ 里 6 行 `f a b c d`、角度参数 0°/180° 都能跑）。

**A 换面表示：第一步已落（2026-10-23）** —— `ngons` 通道 + 真 CC：

| 件 | 说明 |
|---|---|
| `ngons` | 值里除 `faces`（**永远三角**，所有老算子照旧吃）外，另存一份**多边形拓扑**；`ngons_of(mesh)` 取它，没有就退回三角 |
| `MeshQuads` | 产出 `ngons`（四边形 + 没配上对的三角） |
| `MeshSubdivideSurface` | **见到 `ngons` 就走真 CC**（面点/边点/顶点 `Q+2R+(n-3)S`；每个 n 边面 → n 个四边形），输出仍是 `ngons` |
| `MeshSave(obj)` | 优先写 `ngons`（`f a b c d`） |
| 其它算子 | 不认识 n-gon ⇒ 自动降级成三角（**不报错**），这就是"边界处降级"的约定 |
| **`MeshExtrude` / `MeshInset`** | 第三块（同日）：网格带 `ngons` 时走 `_extrude_ngon` —— 复制面环 → 沿面法线推 `distance` + 朝面心收 `inset` → 顶面 + **只在选择边界上建侧壁**（内部边不建墙）。`MeshInset` 就是 `distance=0` 的那一档。实测 `demo_quad_extrude`：方块→四边→选顶面→内缩 0.15→CC 1 层→OBJ = **40 个面全是四边形** |
| **`MeshUnwrap`** | 第四块（同日）：`_unwrap_ngon` —— 岛按**面**（不是三角）判接缝、岛内平面投影 + 等比归一 + 货架打包、沿缝分裂顶点；`detail.islands` 给岛数。实测立方体四边 → **6 岛 / 24 顶点 / uv∈[0,1]**，OBJ 里 `f v/vt …` 成对写（78 条 `vt`） |
| **`MeshBevel`** | 第五块（同日）：`_bevel_ngon` —— 缩面（角点朝面心收 `width`）+ 硬边两侧桥四边 + 顶点补多边形（绕点角度排序）。实测立方体 → **26 个面**（6 缩面 + 12 桥 + 8 角），体积 1.0 → 0.7x；没有硬边时明确报错 |
| **`MeshProjectUV`** | 小改：投影不改拓扑 ⇒ 直接把 `ngons` 原样带出去（以前会悄悄退化） |
| **`MeshQuadRemesh`**（第 56 个节点） | 第六块（同日）：`quad_remesh()` = `remesh()` 体素重采样（内部拿 `_grid_out` 复用**占用格**）→ **`_voxel_quads()` 直接出四边形**（每个体素面一张，100% 四边，不再"三角化再配对"）→ `_laplacian_quads` 平滑。实测球 → `demo_quad_remesh` 全链（→CC→Unwrap→OBJ）：**9 712 个面全是四边形 + 9 714 条 vt**。⚠ 均匀四边，锐边会被体素磨圆（特征对齐要 QEx 那类算法） |

实测（`demo_quads`：方块 → `MeshQuads(45)` → `MeshSubdivideSurface(1)` → `MeshSave(obj)`）：
`quad_cube.obj` 里 **24 行 `f a b c d`（全是四边形）**，`node_errors {}`。

**属性与选择集也要跟上（同日第二块）**：
- **属性随新顶点走**：四边 CC 的新顶点 = `[顶点新位置, 面点, 边点]` 三段，点属性按"顶点留自己值 / 面点取该面平均 /
  边点取两端平均"铺开，面属性按"每老面长 n 个四边形"重复 —— 少这一步 `Cd`/`uv` **长度对不上**，
  宿主 payload 判定失败就当没颜色（**静默丢**，很难查）。判据 `QuadifyTest::test_cc_keeps_attrs`。
- **选择集带 `ngfaces`**：`MeshSelect` 输出的选择集除三角面号 `faces` 外，另给一份**多边形面号**
  （`_tri_to_ngon` 按 `n-2` 累加映射）—— 认识 n-gon 的算子按它选面，老算子继续用 `faces`。
  判据 `QuadifyTest::test_selection_has_ngface_numbers`。


⚠ 两个坑：① `subdiv_surface` 必须在 **`weld` 之前**判 `ngons`（`weld` 重建几何会丢它，实测 24 变 72）；
② 四边 CC 的体积比三角 CC 小（连接性不同，方块 0.42 vs 0.62）—— 不是 bug。

### 8.1f 抄 three.js `examples/jsm`（2026-10-23，+8 个节点 = **64 个**）

用户口径：「**Three.js 是开源的，我们不来抄过来么**」。协议是 **MIT**（保留版权声明即可，合法）。
关键分辨：**它的渲染层我们本来就 `import`**（`MeshViewer` 就是 three 本体）——那部分不用抄；
要抄的是 `examples/jsm` 里那几个**算法模块**。原版是 JS + 逐顶点对象，在我们这套 dict/numpy 里
没法用，所以逐条**重写成 numpy**、口径对齐，每条都有单测钉住（`tests/test_mesh3d.py::ThreeAddonsTest`）。

| 节点 / 参数 | 抄的源码 | 口径 | 判据 / 实测 |
|---|---|---|---|
| **`MeshSimplify`** | `modifiers/SimplifyModifier` | **QEM**（顶点二次误差矩阵）边折叠到目标面数，`ratio` / `target` / `passes` | 球 3 968 面 → 1 190 面 **0.05 s**，体积 0.9012 → 0.8985（形状误差 0.3%） |
| **`MeshConvexHull`** | `math/ConvexHull` | QuickHull 三维凸包 | 方块 → 8 顶点 / 12 面 / 体积 1.0；1 986 顶点的球 **0.32 s** |
| **`MeshOBB`** | `math/OBB` | 候选轴 = 凸包面法线，取**体积最小**那个朝向；`align` = 摆正 | 转 30° 的 1×2×0.5 盒：OBB 体积仍是 **1.0**；摆正后 bbox 边长 = 0.5/1/2 |
| **`MeshCurveDeform`** | `CurveModifier` | 长度轴当 `t`，另两轴偏移进曲线的**法线/副法线**平面；`twist` 累计扭 | 2.0 高的直杆 → x 向跨度 >0.9、截面仍细 |
| **`CurveNurbs`** | `curves/NURBS*` | 夹紧均匀节点 + Cox–de Boor；`weights` 给了就是有理 NURBS | 过首末控制点；权重 8 把中点明显拉向那个控制点 |
| **`MeshNurbs`** | `NURBSUtils` 张量积 | 控制网格（行 `;` / 行内 `|`）→ **全四边形** `ngons` | (su−1)×(sv−1) 个四边 |
| **`MeshTessellate`** | `modifiers/TessellateModifier` | 只劈**最长那条边**、**原顶点一个不动**（保形加密） | 方块 `max_edge=0.6` → 最长边 ≤0.6，8 个原顶点仍在 |
| **`MeshEdgeSplit`** | `modifiers/EdgeSplitModifier` + `toCreasedNormals` | 按二面角分**平滑组**再按组复制顶点 ⇒ 顶点法线在硬边上不再被抹平 | 方块 35° → **6 组 / 24 顶点**；100° → 1 组 / 8 顶点 |
| **`MeshDisplace.noise=simplex`** | `math/SimplexNoise` | 四面体梯度噪声（向量化，种子 = 置换表） | 同种子恒定、值域 ±1、相邻点差 <1e-2 |
| **`MeshSave.compress`**（glb） | `GeometryCompressionUtils` | 法线**八面体编码** 2×uint8（`normalized` accessor） | 1 986 顶点球：96 300 → **76 460** 字节（省 20.6%） |

**等价件不重复抄**：`math/MeshSurfaceSampler` 已有等价物（`scatter` 按面积撒）；
`CurveModifierGPU`（要 GPU compute）、`SceneOptimizer`（我们没场景图）不抄。

示例图：`demo_three_addons`（生成器 alias `l_mesh_nodes_three_addons`）——四支（形状链 / 凸包 /
NURBS / 曲线弯曲）→ `MeshGroup` 并成一件 → **一个** `3d_view`（一张图只许一个它）。
实测：validate `ok: true`、`node_errors {}`、产出 `three_addons.glb`（60 764 B）+ `three_addons_all.obj`。

**踩过的三条**：
1. **QuickHull 地平线新面的绕向**必须与**被删的可见面同向**（不是与幸存邻居反向）：反了会造出
   "穿越内部"的假面，方块凸包的体积只剩 **1/6**（0.1666）、法线一半朝里。这是本次唯一一次算错几何。
2. **朴素 QuickHull 太慢**：每轮重建"边→面"表 + 逐面 numpy 判可见，1 986 顶点的球 **42 s**。
   改成**增量邻接表 + 最大堆 + 每个点只挂最远那个面**（不是挂到所有可见面）→ **0.32 s**（132×）。
3. **QEM 的批内计数要用"活着的面"**：删掉的面留 `None` 占位，`len(faces)` 会骗人
   ⇒ 目标 211 面实塌到 146 面（现在维护 `live` 计数）。

### 8.1g 图元补遗 + 数据进出 + 融合球（2026-10-23，+8 个节点 = **72 个**）

同一轮把"three 有我们没有"的最后几个几何件、以及**外部模型读入**补齐。

| 节点 / 算子 | 抄的源码 | 口径 | 判据 / 实测 |
|---|---|---|---|
| **`MeshCapsule`** | `CapsuleGeometry` | 中段圆柱 `length` + 两端半球（`cap_segments` / `radial_segments`） | 总高 = `length + 2r`；体积 ∈ [圆柱, 圆柱+球] |
| **`MeshCircle`** | `CircleGeometry` | 圆盘/扇形（`theta_length` < 360）；`plane=xz/xy` | 整圆面积 = πr²、扇形 = 按角度比例 |
| **`MeshRing`** | `RingGeometry` | 内外半径之间的**四边形**环（`phi_segments` 径向切几段） | 面积 = π(R²−r²)；`ngons` 全四边 |
| **`MeshShape`** | `ShapeGeometry` | 任意简单多边形 → 耳切三角化 → 平面片（不挤出） | 正方形面积 1.0；法线全 +Y |
| **`MeshRoundedBox`** | `RoundedBoxGeometry` | SDF 圆角：顶点落到"内盒外 `radius`"处，16 个角成球面片 | 每个顶点到内盒距离 = `radius`；无"三面贴合"的尖角 |
| **`MeshParametric`** | `ParametricGeometry` | `u,v → x,y,z` 三条表达式 → **四边形**网格（沙箱 `eval`，与 `MeshPose` 同一套白名单） | 默认单位球：所有点 \|p\| = 1 |
| **`MeshImport`** | `loaders/*`（OBJ/PLY/STL/glb） | 读模型目录或绝对路径 → MESH；带 `uv` / `Cd` / 多材质槽；二进制 STL 自动 `weld` | 四种格式**往返**：OBJ 24v/12f 一致、PLY 带 `Cd`、STL 立方体 36 点 → 8 点、glb 带 uv |
| **`MeshSave(fmt=stl)`** | `STLExporter`（二进制） | 80 字节头 + 每三角 50 字节（法线 + 三点） | 12 三角 → `84 + 12×50 = 684` 字节，读回来 8 顶点 |
| **`MeshMetaball`** | `MarchingCubes` | 球集距离场 `Σ(1−d²/r²)²` → **四面体版 MC** 抽等值面 | 单球半径 = `r·√(1−√iso)`（iso .5 ⇒ 0.5412r）、半径起伏 < 0.01（光滑）、体积对得上解析值 |

**为什么 MC 不用 three 那套查找表**：标准三维 MC 要 256 项边表 + 256×16 三角表，手抄没法自证。
这里把每个体素切 **6 个四面体**、走四面体 16 种情形（1/3 内 = 一三角；2/2 内 = 一四边形按绕质心角成环）——
同样闭合光滑，三角形约 1.5×，代码几十行且判据写死。

示例图 `demo_shapes`（alias `l_mesh_nodes_shapes`，17 节点）：融合球 / 圆角盒 **导出→再导入** / 圆环 /
参数曲面 / 圆盘 / 胶囊 → `MeshGroup` → **一个** `3d_view`（physical + 描边 + 投影）。
实测 `node_errors {}`、`22` 确实排在 `21` 之后（`MeshImport.path ← MeshSave.path` 是真依赖）。

**明确不做（写在这里免得下次重猜）**：
- **`TextGeometry`**：three 的字体是 `typeface.json`（包内 `examples/fonts` 是空的）⇒ 要用户自备字体资产，
  且文字轮廓有**洞**（O/A/B），我们的耳切三角化不支持挖洞。要做先得补"带洞多边形三角化"。
- **USDZ / DRACO 导出**：USDZ 要 zip + usda 文本、DRACO 要原生编码器（本环境没有）；要压缩就用
  `glb + compress`（八面体法线，实测省 20.6%）。
- **Water / Sky / Reflector / Decal**：都要 viewer 侧的 shader / 反射相机 / 投影贴花，属渲染档的下一轮。

### 8.1h 融合球版卡通狮子 `demo_lion4`（2026-10-23，78 节点）

用户口径：「**现在我们做狮子能比 threejs 做得更好了么**」→「**继续拧外观，现在太丑了**」。
造型管线用新补的能力重做一版：`demo_lion4`（生成器 alias `l_mesh_nodes_lion4`，`tools/lion4_demo.py`）。

| 件 | 做法 | 说明 |
|---|---|---|
| 身体/脖子/头/四肢/尾根 | **一颗 `MeshMetaball`**（18 加 + **3 挖**，椭球场，`resolution=56`） | 一件**无缝**；`manifold3d` 没装时这是"把零件焊成一件"的替代路 || **腿缝 / 裆** | **负椭球（`-rx,-ry,-rz` = 挖）**：前腿之间 / 后腿之间 / 腹下各一颗 | 这是"腿和身子糊成一坨"的解 —— 光靠加球永远粘在一起（新增能力，判据 `test_metaball_carve`） |
| **头 = 圆角方**（2026-10-23 晚） | `MeshRoundedBox(1.10, 1.10, 1.02, r=.42)` + `MeshTransform(rx -8)` | 球头**永远是正圆**、脸上没有平面 ⇒ 五官只能"贴"上去（用户原话「像烂泥做的」）；圆角方正面是一块平面，眼睛/吻部才坐得住 |
| **玩具风（去泥）** | `resolution 30` + **去掉**置换 + `MeshEdgeSplit(28°)` | 之前 `res 56 + 加密 + 噪声 0.006` = 均匀细疙瘩，就是"泥/石膏"质感；粗网格 + 硬边 = 面片分明 |
| **鬃毛成束** | `MeshScatter(count 18, scale 2.8)` | 480 片小簇 = 噪点绒毛（远看一团脏色）；18 个大毛束才读得出"一撮一撮" |
| 鬃毛 | 环面（孔朝 +X，**退到脸后面** x=0.14）→ simplex 置换 → **`MeshScatter` 480 片毛簇** | 模板 = **压扁的锥**（`sy 0.32 / sz 1.5`）；**`bias="-1,0.35,0"` + `bias_amount 0.35` + `bias_falloff 0.55`** 让毛往后倒（2026-10-23 新参数） |
| 尾巴 | `CurvePath`（S 形）→ `MeshSweep(taper .65)` + 毛球 | 改曲线即改形 |
| **米色胸腹** | `MeshSelect(mode=box)` → `AttrSet(Cd=米色, selection)` → 再选一次 → `AttrSet(mat=2)` | ⚠ 用**球域**选（r .44）会把四条腿一起刷白（像穿白裤）；**盒域** z 半宽 ≈0.15 才只吃到"胸腹 + 内侧腿" |
| **嘴（微笑弧）** | `CurvePath`（5 点，两头高中间低）→ `MeshSweep(r .017)` → 深色槽 | 扁球是直条（像胡子）；弧线要按吻部表面算 x，埋 0.03 就看不见 |
| **前爪脚趾** | 一颗小球 → `MeshArray(3, offset_z .056)` → `MeshTransform` → `MeshMirror(z)` → 米色槽 | 一个 `MeshArray` 顶三份，别手摆三个球 |
| 渲染 | `mat_type=toon` + `slot_type="1:physical"`（鬃毛 sheen .4）+ `post="outline:2; bloom:.08"` + `shadow=1` | 6 个材质槽 |

**调形的六条经验（照这个顺序改，别乱拧）**：
1. **metaball 视觉半径 = `0.5412 × r`**（`iso=0.5` 时表面在 `d=√(1-√iso)`）；`blob(中心, 半轴)` 就是"填半轴、内部除 0.5412"。
   多球叠加会**互相鼓大**，所以五官位置要按**渲出来的**表面 `x` 定，不要按"球心 + 半径"算。
2. **五官贴球面要解椭球方程**：`x = cx + Rx·√(1 - ((y-cy)/Ry)² - ((z-cz)/Rz)²)`。
   按"球心 + 半径"估会把眼睛放**进**头里（实测：眼白变成一抹白绷带）。
3. **眼球别做球**：沿 X 压扁成镜片（`sx 0.45`），瞳孔再压一点并前移 0.05；球贴在球面上永远像护目镜。
4. **吻部别伸太长**：`0.66` 倍长 → 像白鸭嘴；`0.5` 才收得住。鼻子压在吻部**上缘**。
   **嘴 = `CurvePath` + `MeshSweep` 的弧线**（两头高、中间低 = 微笑，`radius .017`），
   用压扁的扁球是**直条**、像胡子（实测）。弧线点要按吻部表面算 `x`（`y=1.22` 时表面 x≈1.055），
   埋进去 0.03 就整个看不见 —— 每个点往外放 0.01 正好露出一半。
5. **耳朵要离开鬃毛环**：鬃毛环面退到 x=0.14（脸后面），耳朵在 x≈0.40 就自然露出来；耳朵别拉太长（`sy 1.20` 像兔耳、`0.95` 才像狮耳）。
6. **看形要看两个角度**：`--yaw -32`（3/4 后左）+ **`--yaw -95`（正面）** + `--pitch -22`（低角度看肚子）——
   正面才能暴露"吻部太小 / 耳朵太长 / 瞳孔偏低"，低角度才看得见胸腹补色有没有糊到腿上。
7. **按区域补色别用球域**：`MeshSelect(mode=sphere)` 在一个"腿贴着身子"的模型上必然连腿一起吃
   （实测 r .44 把四条腿刷成米色 = 白裤）；用 `box` 并把 z 半宽收到 0.15（≈ 腿内侧面）才干净。
   补色区域还要**落在看得见的那面**上：一开始盒子放在胸口，结果从侧面看是一道"白渍"，挪到腹下才对。
8. **毛的走向靠 `bias`，不是靠 `tilt`**：`MeshScatter` 新增 `bias`（方向）/ `bias_amount`（0~1）/
   `bias_falloff`（沿轴梯度）。纯随机 `tilt` = 炸毛球；`bias_amount` 压到 0.55 又塌成一圈平毛 ——
   **0.35 + tilt 52 是"蓬松且有走向"的中间值**（狮子鬃毛实测三档：0 / 0.35 / 0.55）。
9. **生成器里的节点 id 必须全局唯一**：头这块我用了 `95~98`，其中 `96` 早被"脚趾球"占了 ⇒
   后定义覆盖前定义，表现是**头根本没进图**（渲出来只有鬃毛环 + 五官浮在洞里），而日志一句错都没有。

**自证**：`node_errors {}`（78 节点全链）；探针 `lion4` **6/6**（`matslots=6` / `mattype=MeshToonMaterial,MeshPhysicalMaterial` / `shadow=1`）；
外观复核：`record_timeline.py --node 80 --static --yaw -32|--yaw -95`（离线光栅，flat shading，不是最终材质效果）。

### 8.1i 写实雄狮 `demo_lion5`（2026-10-23 晚，178 节点）

用户给了**参考图**（真实雄狮：站立、张口咆哮、大鬃毛）⇒ 卡通那条路作废，重开一条"按解剖搭"的：

| 部位 | 做法 | 关键数字（`tools/lion5_demo.py`） |
|---|---|---|
| 躯干 | **`MeshLoft` 7 个椭圆截面** | 前胸(0.88, ry .24) → 胸最深(.34) → 肩后(.36) → **腰(.31，收)** → 腹后(.33) → **臀(.36，鼓)** → 尾根(.22) |
| 四条腿 | **`CurvePath` + `MeshSweep(radius_profile)` 折线扫掠** | 前腿 肩(0.72,1.02)→肘(0.70,0.66)→腕(0.72,0.34)→(0.74,0.10)；**后腿必须反关节**：髋(-0.54,1.00)→膝前(-0.40,0.64)→跗后(-0.64,0.38)→(-0.60,0.10) |
| **腿部肌肉** | 肩三角肌 / 前臂 / 臀股 / 小腿 **各一颗椭球**（右半 → `MeshMirror(z)` 成对） | 肩 `(0.66,0.90,.20)` 缩 `(1.05,1.55,0.95)`；前臂 `(0.71,0.50,.20)` `(0.85,1.20,0.80)`；臀股 `(-0.50,0.84,.23)` `(1.35,1.70,1.15)`；小腿 `(-0.52,0.48,.22)` `(0.90,1.30,0.85)`（球 `r 0.12`，倍数即半轴） |
| **颈** | `MeshLoft` 3 截面接躯干与头（否则头是"长在鬃毛球上"） | (0.86,0.98,ry .22) → (0.92,1.18,.20) → (0.98,1.34,.17) |
| **颅骨** | `MeshLoft` 4 个**超椭圆**截面（`ring(..., power=)`，2 = 正椭圆、越大越方） | power **2.2→2.6**、rz `.155/.205/.195/.165`；⚠ **power 3.0 方成箱子**（v10 实测）、2.0 又回软球 |
| **颧弓** | 两侧**薄片**椭球（横向骨棱） | `(1.12,1.335,.172)` 半轴 `.115/.048/.026`；⚠ 厚了正面读成"腮帮子/耳朵"（v11） |
| **吻管** | 独立 `MeshLoft` 方管（power 3.2→3.6），比颅骨窄、略低 | (1.18,1.360,ry .135,rz .115) → (1.26,.112/.102) → (1.32,.090/.086)；前端球域选面色 `PALE`（半径 .072） |
| **胡须垫** | 吻两侧扁椭球 | `(1.298,1.342,.052)` 半轴 `.041/.036/.036` |
| **鼻镜** | **超椭圆放样**的方鼻垫（power 2.8→3.2，横向宽 > 纵向） | rings `1.318/1.344/1.366`、cy 1.392、ry `.032/.034/.028`、rz `.042/.048/.042`；⚠ 球做大了就是"罐盖/猪鼻" |
| **下颌 / 开口** | 独立 `MeshLoft` 方管张开 + 4 獠牙（锥）+ 粉舌 + **暗红口腔内壁**（椭球垫在开口里，槽 6） | 铰链 (1.10,1.30,.075/.105) → (1.29,1.19,.050/.072)；上牙 (1.30,1.345,±.062,r .018,h .072)、下牙 (1.29,1.225,±.052)；口腔 `(1.185,1.262,0)` 半轴 `.106/.074/.072` |
| **眼 / 眼窝 / 眉** | **眼窝衬**（比眼珠大的深色球垫在后面）+ 金黄眼珠 + 小黑瞳 + 深色薄眉骨，各一颗椭球镜像成对 | 眼窝 `(1.178,1.452,.152)` 半轴 `.060`；眼 `(1.185,1.452,.155)` 半轴 `.050`；瞳 `(1.208,1.453,.190)` 半轴 `.020`（**必须偏出到眼珠表面之外**）；眉 `(1.185,1.496,.148)` 半轴 `.055/.016/.040` |
| **耳** | **扁盘朝前** + 内耳深色（`BROW`） | 耳 `(1.090,1.585,.155)` 半轴 `.066/.062/.041`；内耳 `(1.112,1.585,.155)` 半轴 `.029/.038/.022`；⚠ 贴额角=角、球=泰迪，两版都踩过 |
| 鬃毛 | 椭球壳（`MeshDisplace` simplex 0.035/freq 3.4）+ **900 缕弯曲渐细的扁毛** | 壳 `(0.66,1.14)` 缩放 `(0.94,1.84,1.22)`（**sy 1.84 才垂到胸**）；**毛缕模板不是锥**：`CurvePath`（`0,0,0 → .030,.090 → .058,.190 → .046,.262`，samples 5）→ `MeshSweep(r .030, taper .22, sides 5)` → `MeshTransform(sz 0.45)` 压成扁毛；`scale 1.0`、`scale_var .35`、`tilt 16`、`bias="0,-1,0.15"`、**`bias_amount .80`**、`falloff .55` |
| **贴图管线**（2026-10-23） | `MeshProjectUV`（柱面，绕 Y）→ `TexNoise`→`TexGradientMap`（毛皮基色）+ 细噪 overlay（毛丝）+ `TexNormalFromHeight`（毛的凹凸）+ `TexLevels`（粗糙度）→ `TexMaterial` → `3d_view.material`；**鬃毛（槽 1）另一套**（更暗基色 + 更粗毛丝法线）→ `3d_view.slot_mat="1:325"` | UV：`mode="cylindrical"`、**`scale 3`**；身体基色 `0:#5a4022; .45:#8b6437; .78:#c2a06d; 1:#e3d3ac`、细噪 `scale 42` overlay `0.38`、法线 `strength 2.4`、粗糙 `out .68~.95`；鬃毛基色 `0:#0f0a05; .5:#2a1c10; .85:#4a3320; 1:#6a4a2c`、法线 `strength 3.2`；⚠ 见下面坑 11~14 |
| 尾巴 | 曲线扫掠 + 末端毛球（simplex 置换） | (-0.82,0.96) → (-1.05,0.80) → (-1.16,0.58) → (-1.10,0.44) |
| 配色 | **7 槽** | 0 沙褐 `0.72,0.56,0.36` / 1 深棕鬃 / 2 浅色(吻·腹·腿肌) / 3 牙白 / 4 鼻·瞳黑 / 5 舌粉 / **6 口腔暗红 `0.26,0.09,0.11`**（眼珠金黄 `0.80,0.58,0.16`、眉骨·内耳·眼窝深一档 `0.46,0.34,0.22` 走 0 槽）；`3d_view.slot_shade = "0:fuzz; 1:fuzz; 2:skin; 6:skin"` |

**和参考图对上了**：四足站姿、躯干有转折（不是一颗球）、后腿反关节、**张口露牙**、鬃毛罩颈肩并垂到胸、
**腿有肌肉鼓包**、**头是按解剖分件的**（方颅 + 颧弓 + 方吻管 + 胡须垫 + 方鼻镜 + 张开的下颌 + 4 獠牙 +
**暗红口腔内壁** + 扁耳带内耳 + **眼窝衬 + 金眼黑瞳** + 眉骨）、尾巴 + 深色毛球、沙褐配色、
**鬃毛是 900 缕下垂扁毛（有走向，不是锥堆）** —— 正视/侧视都复核（`--yaw -50 --pitch -4`、`--yaw -95 --pitch 2`）。  
**还差的**：毛缕仍是散件（没有根部起毛）；眼珠在 flat shading 下只剩一圈细金环
（真材质里会亮）；贴图只覆盖**槽 0**（身体/腿/脸），鬃毛要单独一套材质（宿主的 `slot_mat` 机制能接，
本轮没做）。

**判据（贴图这条腿）**：宿主 `mesh_payload` 里 **`uv` 81084 / `count` 81084**（齐了才会 `dataset.uv=1`）、
`groups 28 / mats 7`；离线带贴图出片 `record_timeline.py --node 124 --mat "0:309, 1:325"` →
`Temp\lion5_tex6.gif`（身体毛皮渐变 + 鬃毛更暗的毛丝层 + 牙白/鼻黑/眼亮，说明两套材质各在各的槽）。

**踩到的**：
1. `MeshLoft` 的文本是**截面间 `|`、点间 `;`**（`loft()` 里 `ring()` 生成一条，别写反）。
2. `MeshSweep` 的 `radius_profile` 是**逐控制点倍数**（`1.0; 0.92; 0.58; 0.70` = 肩粗→肘细→腕更细→掌略粗）。
3. **超椭圆 `power` 是"方/圆"的唯一旋钮**：头骨 2.2~2.6、吻 3.2~3.6；`3.0` 用在颅骨上会方成箱子。
4. **贴片件的半轴必须对着宿主曲面算**：眼/眉/耳/颧弓埋进去就"看不见"，凸太多就是"瘤"
   （超椭圆面上一点：`(|Δy|/ry)^p + (|Δz|/rz)^p = 1`）。
5. **毛缕模板要用 `CurvePath`+`MeshSweep(taper)`、别用 `MeshCone`** —— 锥是"刺"，820 根锥 = 海胆。
6. `bias_amount` 才是"倒向"主旋钮（.45 还炸着、.80 顺下来），`tilt` 只是抖动（30° 就炸）。
7. 鬃毛壳的 simplex 幅度**别超 0.04**（0.055 时壳体方块化，远看像岩石）。
8. **离屏光栅是 flat shading**，看着比真宿主粗一档。
9. **新加材质槽要同时改 `3d_view.slot_shade`**（口腔用槽 6，得写 `6:skin`；漏了它那块按默认档渲染）。
10. **口腔内壁要垫到牙齿后面**（内壁 x 上界 ≈ 獠牙位置），否则从开口里看到的是"空的黑洞"或背面。
11. **贴图先有 UV**：生成几何默认**没有 uv**（`MeshViewer` 只在 `uv` 齐了才 `dataset.uv=1` 并挂贴图）。
12. **别对这个网格用 `MeshUnwrap`**：5.5 万面的四边/ngon 拼装体会走 `_unwrap_ngon`（逐岛展开 + 沿缝裂点），
    实测 **8.1 万顶点 / 15.8 万面 → 11.9 万顶点**，demo 里不划算；**柱面投影**（顶点不分裂）就够贴噪声毛皮。
13. **基色贴图落地的语义**：宿主把 `material.color` **刷白**（乘数 = 1）再贴 `map` ⇒ 离线复核（`--mat`）
    也必须**只用贴图色**，不能乘模型自带顶点色（乘了整只变暗，像抹了泥）；且**只有写了的那几个槽**吃贴图，
    其余槽（牙/鼻/舌/眼）保持纯色 —— 离线采样要按 `attrs.prim.mat` 过滤，否则鬃毛糊成一片。
14. **一槽一套材质要 `3d_view.slot_mat="1:325"`**（`槽号:材质节点 id`）；没写 `slot_mat` 的槽恒为纯色，
    写错的槽号**静默不生效**（面板「槽」那排能看出哪几个槽吃到了图）。离线对拍用
    `record_timeline.py --mat "0:309, 1:325"`（同一份口径）。

### 8.2 导出（2026-10-22 / 23）

- **`fmt=glb`（2026-10-23，glTF 2.0）**：`gltf.py` 自己写二进制容器（JSON + BIN 两个 chunk），
  带 **POSITION / NORMAL / TEXCOORD_0（有 uv 时）/ COLOR_0（有 Cd 时）**、按 `prim.mat` **每槽一个 primitive + 一份 material**、
  **蒙皮**（`attrs.point["W:<关节>"]` → 每点取最大 4 个 → `JOINTS_0`/`WEIGHTS_0`，`inverseBindMatrices = T(-P)`）、
  **动画**（`rotations` 文本按 fps 逐帧求值 → 每关节一条 `rotation` 通道，四元数）。
  `MeshSave` 新增 `rig` / `rotations` / `fps` / `duration` 四个入参。
  判据：`tests/test_mesh3d.py::GltfTest`（容器头 + chunk + 静态无 skin/anim + 蒙皮/动画结构与帧数）。
  实测 `demo_lion3`：`lion3_skin.glb` **4.1 MB · 7 个 primitive · 1 个 skin · 1 段动画（6 条通道 / 73 帧）**，
  three.js 的 `GLTFLoader` 直接吃下（标题显示 `lion3_skin.glb`、无报错、动画下拉里有 1 段剪辑）。
  ⚠ 内部旋转矩阵是**行向量约定**（`p' = p@W`），转四元数前要先转置（见 `gltf._quat_from_row_matrix`）。
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
