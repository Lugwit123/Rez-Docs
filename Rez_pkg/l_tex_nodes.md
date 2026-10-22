# `l_tex_nodes` —— 程序化贴图算子（Substance Designer 那类）

> 更新：2026-10-12。**先读本文再改算子** —— 每个算子的语义（值域、方向、单位）都在这里定死，
> 改之前也要看 §5「踩过的坑」。
>
> 上游是 **`l_node_engine`**（节点图内核，纯标准库），下游是 **`l_mindmap_mmd`** 的
> `/pygraph/<图名>` 编辑器（画布上多出 **46** 个节点：39 个基础 `Tex*` 算子 + **`2d_view` / `3d_view`** + `TexSbsar` + 4 个 FX 内层 `Fx*`）。**引擎里没有一行图像代码**：
> 本包是"外面挂进来的算子库"。

## 一、这是什么 / 为什么这么分层

| 层 | 文件 | 职责 |
|---|---|---|
算子 | `src/l_tex_nodes/ops.py` | **纯 numpy**：吃 `TexImage` 出 `TexImage`，没有"节点"概念，可单独 import 调用（所以好测） |
值类型 | `src/l_tex_nodes/image.py` | `TexImage`（内存里的图）+ 读写盘 + 输入归一化 `as_image()` |
节点 | `src/l_tex_nodes/nodes.py` | 把算子包成 `BaseNode`，**导入即注册**；宿主 discover 它 |
示例 | `src/l_tex_nodes/samples.py` | 每个算子一段极小图（调色板预览图用它现烤，见 §6.2） |
自证 | `src/l_tex_nodes/selftest.py` | 逐像素断言 + 示例图逐个真跑（`wuwor l_tex_nodes -- l_tex_nodes_selftest`） |

**为什么不做进引擎**：`l_node_engine` 刻意只用标准库（宿主无关、可被任何进程 import）；
图像算子要 numpy + Pillow（几十 MB，还有 ABI 问题）。所以按仓库既有机制拆包：
**依赖写进本包 `requires`，宿主 `requires` 里带上本包，启动时 `discover_nodes("l_tex_nodes.nodes")`。**

## 二、值约定（改算子前必读）

| 约定 | 取值 | 为什么 |
|---|---|---|
像素值域 | **`0..1` 的 `float32`** | Substance / 大多数图像算法的口径；导出时再乘 255。越界由 `TexImage` 夹紧 |
形状 | `(H, W, C)`，**`C ∈ {1, 3, 4}`** | 灰度 / RGB / RGBA。`(H, W)` 的数组会被自动补成灰度 |
节点之间传什么 | **`TexImage` 对象**（不是文件路径） | 实时调参要的是"内存里过一手"；每步落盘等于掐死实时性。**只有 `TexPreview` / `TexExportImage` 落盘** |
端口类型 | 统一 **`IMAGE`** | 与 AI 图像节点同类型才能互连；**输入一律过 `as_image()`**：`TexImage` / 文件路径字符串 / `None` 都认 —— 所以"AI 出的图（路径）"能直接接进来 |
引擎里的 `outputs` | 值是 `TexImage`，SSE 序列化会退化成 `repr()` | 宿主 `_sse()` 对不可序列化值有 `default=repr` 兜底 —— 所以 `__repr__` 必须短（`TexImage(1024x1024 rgb)`），别塞数组 |
落盘目录 | **宿主媒体目录**（`l_node_engine.ai_nodes.media_dir()`） | 宿主的 `/api/pygraphs/media/<名>` 只从那儿读；另建目录等于"图写了但画布看不见" |
掩码类输入 | 标 `forceInput: True`（如 `TexBlend.mask`） | 引擎的 `container_for` 会把 `IMAGE` 判成"文本输入框 + 手柄"（可以填路径也可以连线）；要求**必须连线**的端口就加这个 |

**尺寸：绝对 / 相对父级**（2026-10-14 用户口径：「所有 SD 相关节点的尺寸参数都可以设置为相对于父级」）：
每个带 `width`/`height` 的节点（生成类 `TexGradient`/`TexNoise`/`TexShape`、`TexResize`/`TexCrop`/
`TexPixelProcessor`/`TexFxMap`、FX 内层 `FxQuadrant`/`FxIterate`/`FxTile`，共 10 个）都多两个参数：

- `size_mode`：`absolute`（默认，= 老行为，用 width/height）/ `parent`；
- `size_scale`：只在 `parent` 时有用（1.0 = 跟父级一样大，0.5 = 一半）。

`parent` 的"父级"取谁：**有输入图 → 第一张能用的输入图**（"跟着上游走"）；**没有输入图 → 图级基准**
（SD 的 base resolution：图文件顶层 `size: [宽, 高]`，编辑器里在 **「⇄ 接口」格的「基准尺寸」** 改，
宿主跑图时读它传给引擎 `execute_graph(base=…)`，缺省 1024×1024）；两边都没有 → 落回 width/height。
嵌套里（FX 子图 / 子图节点）"父级"= **这一块容器的尺寸**（`TexFxMap` 把象限尺寸传进 `run_nested(base=…)`），
与 SD 一致。实现只有一份：`image.resolve_size()` + 引擎的 `graph.active_base()`。

⚠ 尺寸各维一律夹进 `SIZE_MIN..SIZE_MAX`（4..4096，见 `image.SIZE_MAX` 那段）——
"打字中间态被自动存盘"实测能让后端去要 78 GiB（`width: 10242048`），夹一刀成本为 0。
判据：`selftest.py::_check_size_mode`（absolute 不变 / 跟输入图 / 跟基准 / 取不到回落 / 整图与嵌套两条端到端）。

**确定性**：噪波一律由 `seed` 决定的整数哈希生成（不用全局 `np.random`）—— 同一参数两次跑**逐像素一致**
（不然实时重算会自己抖，也没法写测试）。`TexNoise` / `TexWarp` 都有 `seed`；
`TexNoise` 另有 **`phase`（相位，0..1）**：采样坐标整体按格平移（**x/y 同向 = 斜着挪**），
**不动 `seed`**。实测（2026-10-14，96²/scale24 逐像素对拍）：`value`/`perlin`/`worley` 是**精确平移**
（`phase=1` 整图斜挪一格后 mean|Δ| = **0.00000**；`0.02` 只是轻轻挪一小步，mean|Δ| ≈ 0.01，
与"换 seed"的 ~0.27 差两个数量级）—— 所以它是 seed 的补充（seed 换一套哈希，phase 在这套哈希上挪位置、
可做细微调整）；`fbm` 是每层各按自己的格子挪，合起来约等于换花样（不是单一位移）。
（语义与判据写在 `selftest.py::_check_noise` 里，改这块先看它。）

## 三、节点清单（**44** 个，画布分类 `l_tex_nodes`：下表 **40** 个 + §7.5 的 4 个 FX 内层 `Fx*`）

| 分类 | 节点 | 要点 |
|---|---|---|
生成 | `TexGradient` | `linear / radial / angular / diamond`；`angle` 单位度 |
 | `TexNoise` | `value / perlin / worley / fbm`；`scale` 越大越粗；`octaves` 只对 fbm 有意义；`seed`、`phase`（相位 0..1，按格平移采样坐标）；**`seamless`（默认**开**，2026-10-15）**：格点按整幅折回 ⇒ **整幅就是一个周期**，平铺/滚动无缝；关掉 = 老口径（散着采，边上不接）。代价：周期梯度噪波整幅均值恒 ≈ 0.5（"拿均值当签名"的判据会假失败，见 `selftest.py` 那条注） |
 | `TexShape` | `rect / disc / ellipse` 遮罩；`size` 占短边比例、`hardness` 边缘软化；**`disc` 永远是正圆**（忽略 `ratio`） |
滤镜 | `TexBlur` | `gaussian`（**三次方框近似**，不用 scipy）/ `box` / `radial`（绕图心角度模糊，单位度）。**`tiling = clamp / repeat`（2026-10-16，默认 clamp）**：平铺纹理（`TexNoise` 的输出）要无缝就得 `repeat`（`np.pad(mode="wrap")`）—— clamp 会在四边各留一圈"被边缘拉住"的像素（实测半径 5 的无缝噪波首末行落差比内部大 **8~10 倍**，半径 10 到 **15~25 倍**；`repeat` 后回到 0.6~1.5 与基准同量级）。⚠ **`radial` 例外**：它绕图心**转**、旋转不是平移 ⇒ 即便 `repeat` 也换不来平铺无缝（实测 3.1/5.1），只让它不再被边缘夹取污染。⚠ `tiling` **只给 clamp / repeat 两档**：模糊是"邻域取平均"，mirror 会让镜像面重复计数、black 会改归一化口径 —— 都不是"平铺"该有的语义（非周期素材的无缝另有 `TexSeamless`） |
 | `TexDirectionalBlur` | 方向（拖尾）模糊：`angle` 度、`length` 像素、`samples` 档数（直线平均，保能量）；`tiling = clamp / repeat`（同 `TexBlur`，平移和 ⇒ repeat 后平铺无缝） |
 | `TexEdgeDetect` | `sobel`（梯度模，默认）/ `laplacian`（细二阶）/ `emboss`（浮雕，平地 = **0.5** 灰）/ `outline`（平地 = 白） |
 | `TexSharpen` | 反锐化掩模：`原图 + amount × (原图 − 模糊)`；`tiling = clamp / repeat`（传给里面那层模糊） |
 | `TexLevels` | 输入黑白场 → `gamma` → 输出黑白场（**0..1 口径**，UI 上常说 0..255） |
 | `TexHistogramScan` | 按画面**自己的亮度百分位**归一（`position` = 新中点、`contrast` = 窗口宽度）；按亮度算一刀、三通道共用 → **色相不动** |
 | `TexCurve` | `points = "0,0; 0.5,0.6; 1,1"`；**Fritsch–Carlson 单调插值**（普通三次样条会过冲出脏边） |
 | `TexHSL` | `hue` 是**偏移量**（度）、`saturation`/`lightness` 是**乘数** |
 | `TexInvert` / `TexThreshold` | 阈值 `softness>0` 为 smoothstep 软过渡 |
 | `TexGradientMap` | 灰度 → 彩色（Substance 的 Gradient Map）：`points = "0:#000; 0.5:#ff8800; 1:#fff"`；`alpha_from_gray` |
 | `TexQuantize` | 压到 `levels` 档；`dither = none / ordered`（4×4 Bayer）/ `noise`（确定性噪点，抖动**不多出档**） |
 | `TexMorphology` | 膨胀 / 腐蚀（灰度也适用）；半径上限 16（迭代实现，很大就慢） |
| `TexSeamless` | **无缝纹理**：错开半张 + 按边缘权重混回来 ⇒ 平铺时边界与内部连续（值不跳）、画面中心不动。`strength` 整体强度（0 = 原样）、`power` 越大越只在最边上动手。⚠ 它改的是**像素值**，对噪波/高度/遮罩这类内容均匀的图最合适；有明确构图的图边上会出现「十字」痕迹（这套方法的固有代价，SD 的 Make It Tile 也一样）。**什么时候要它**：源**本身不是周期的**（AI 出的图、照片、手画的遮罩）才需要这一手；`TexNoise` 2026-10-15 起默认周期化、放大配 `TexResize(tiling=repeat)` 就够，**不用**再套它（套了反而把噪波糊一下） |
 | `TexPixelProcessor` | **逐像素表达式**（整张图当数组一起算）：`a/b/c` 输入图、`x/y` 是 0..1 坐标；可用 `clip/min/max/abs/sqrt/hypot/where/sign/floor/round/sin/cos/exp/log/tanh` 与 `pi`。例 `clip(hypot(x-0.5, y-0.5) + a*0.35, 0, 1)`；**不是**任意代码入口（白名单函数 + 空 `__builtins__`） |
混合/通道 | `TexBlend` | **13 种模式**（normal/multiply/screen/overlay/soft_light/hard_light/add/subtract/difference/darken/lighten/min/max/divide）+ `opacity` + `mask`；`bottom`=底、`top`=顶 |
 | `TexChannelSplit` / `TexChannelMerge` | 拆 R/G/B（1 通道）与合回 RGB；尺寸以 `bottom`/`r` 为准，其余自动缩放 |
几何 | `TexTransform` | `translate_*` 是**图宽百分比**（0.5 = 挪半张）；`rotate` 度；**`scale` >1 = 放大内容**、<1 = 缩小（露出边缘，靠 `tiling` 决定露什么）；`tiling = clamp/repeat/mirror/black` |
 | `TexWarp` | 按一层噪波沿 `angle` 推挤（`amount` = 图宽百分比） |
 | `TexResize` / `TexCrop` | 改尺寸（`tiling = clamp/repeat/mirror/black`）/ 裁剪（越界用边缘像素补，**尺寸恒等于请求值**）。⚠ **周期纹理（`TexNoise` 的输出）要"放大后仍无缝"必须 `tiling=repeat`**：`clamp` 会把边界拉成边缘值，接 `TexTransform(repeat)` 一滚就是一道可见的**横缝**（2026-10-15 用户报过这道缝，`w-1` 那一夹把分数量丢了 —— 平铺采样要**折回坐标 + 跨边界那一拍索引接回 0**，两条都做才是周期延拓；判据见 `selftest.py::_check_geometry` 与探针 `seam`） |
遮罩派生 | `TexNormalFromHeight` | 切线空间法线；`invert_y` 切 DirectX 那种上下翻转 |
 | `TexCurvatureFromHeight` | `concave`（凹/谷）/ `convex`（凸/脊）/ `both`（绝对曲率） |
 | `TexDistance` | 遮罩 → 距离场（`outside` / `inside` / `signed`）；**归一化**到 0..1（`signed` 时 0.5 = 中线） |
 | `TexBevel` | 遮罩 → 倒角高度：边缘在 `radius` **像素**内斜下去、内部平；剖面 `linear/smooth/round/cone`。用**像素级**带符号距离算（和 `TexDistance` 的归一化不是一回事，所以半径真是像素） |
 | `TexFloodFill` | 连通域填色（4 连通）：`random` = 每个岛一个**确定性**随机亮色 / `index` = 按编号出灰（相邻岛颜色差得远） |
 | `TexSplatter` | 在背景上撒 `count` 个 `disc/square`；接了 `density` 就**按它的亮度加权落点**（拒绝采样）；`seed` 定一切 → 同参数逐像素一致 |
子图/FX-Map | `TexFxMap` | **把一张图 / 一份图案子图反复实例化**（引擎 `SubGraph` 的用家，用法见 §7.4）：`mode = quadrant`（四等分递归 `depth` 层，矩形不重叠、正好铺满；每块跑一次图案）/ `iterate`（`iterations` 轮，**上一轮输出当下一轮输入**，再按 `blend_mode` 叠回）；`pattern_graph` = 图案**子图名**（宿主内联成 `pattern`），留空用 `builtin`；变量按子图 `Input` 名喂：`index`/`number`/`x`/`y`/`size`/`pos`；每档按 `t=(i+0.5)/N` 递进位移/旋转/缩放/上色（`tint`） |
材质 | `TexMaterial` | 把 `basecolor/normal/roughness/metallic/height/ao` 接成一份材质（值是**普通 dict**，端口类型 `MATERIAL`，**不是图**） |
 | `TexMaterialChannel` | 从材质里取一个通道 → `IMAGE`（可继续算子） |
 | `TexExportMaterial` | 每个通道各导一张 PNG（`prefix_通道名`），返回路径列表 |
输入输出 | `TexLoadImage` | 磁盘图 → `TexImage`（上游已是图就不必用） |
 | `TexPreview` | **原样透传**（插在链中间不断流）+ 写 PNG 给画布内联预览 |
 | `TexExportImage` | 写 PNG（给下游 AI 节点用 —— 它们要的是**路径**），返回路径 |
 | `TexInfo` | 尺寸/通道/每通道 min·mean·max（排错用；返回字典 + `ui.text`） |
 | `TexFromComfy` | **ComfyUI 出图转贴图**（2026-10-18）：把 ComfyUI 的 `IMAGE`（**torch 张量**，BCHW/CHW）转成 `TexImage` —— 之后就能接 `Tex*` 算子或 `2d_view` 在 pygraph 里直接看（`/render` 出 PNG）。⚠ **鸭子类型、不 import torch**（pygraph 不依赖它）：`detach()/cpu()/numpy()` 有就调，已是 `TexImage` 原样透传。 |
 | `2d_view` | **2D 视图**（类名 `View2D`，2026-10-16；2026-10-17 收 `SCENE`）：**原样透传**、**不落盘**、不给 `ui`。输入端口是 `*`：接 `IMAGE`（`TexImage`/路径）走老路；接 **`SCENE`（`Mg*` 运动图形）原样吐回去**（宿主不光栅，矢量留给取景器）。它现在的意义在编辑器那一侧 —— **画布上没选中节点时**，底栏「🖼 2D 视图」看的就是它。⚠ **一张图只允许一个**（类上标了 `SINGLETON = True`，引擎校验报 `not_singleton`）；注册名就是 `2d_view`（**故意不带 `Tex` 前缀**：编辑器按键名认它）——详见 §6.2b |
 | `3d_view` | **3D 视图**（类名 `View3D`，2026-10-17）：给底栏「🧊 3D 视图」**指定看什么** —— `model`（模型名，可连线）/ `mesh`（接一个出几何的节点）/ `material`（`MATERIAL`）**三路都是可选**，**接了就以那路为准、没接的那路回落到面板自己的设置**（用户口径："三个输入都接，如果某个端口没接就使用 3D 视图的设置"）。输出 = `model` 字符串原样透传（插链中间不断流）、不落盘。⚠ 同样是**单例**（`SINGLETON`）、注册名 `3d_view`；**没有示例图**（输出是字符串）→ 调色板那格靠 `ICON`（线框立方体）——详见 §6.2c |
SBSAR | `TexSbsar` | **Substance 归档（`.sbsar`）当节点**（2026-10-14）：参数 = 归档自己的参数、输出 = 它的每路 `OUTPUT`（`port_spec` 的端口表）。点卡片上「🔎 读一次 sbsar」→ 宿主 `GET /api/pygraphs/sbsar?path=…` 把两张表填好。⚠ **参数行一律 `source:"port"`**（用户 2026-10-15："参数都应该显示输入端点，有端点用端点输入，没端点自动使用手动输入"）：**每个参数长一个输入端口**，连了用连线值 → 没连用手填值（卡片上就是那个控件）→ 都空 = **归档自己的默认值**（`_value_arg` 空 = 不传 `--set-value`）。两张表同时存在时**输出听端口表**（别按"每行一个输出"长假端口），**输入仍带上 `port` 行**（`pySchema.toFlow` 的合并规则）。⚠ **出图必须有 Substance 引擎**（`sbsrender.exe`，随 **Substance 3D Designer / Automation Toolkit** 分发，官方要有效订阅；没有免费/开源替代）：模块按 `L_MINDMAP_SBSRENDER` → PATH → 常见安装目录（含 `C:\ProgramData\PCGM_Apps\App\Adobe\…`）找它，找不到时节点/读口**点名报错**。渲染一次拿全部输出（`--output-name "{outputNodeName}"` + JSON 报告），缓存在 `<pygraphs>/_renders/sbsar/<sha1>/`（键 = 路径+mtime+图名+尺寸+参数值）。⚠ 两个实测坑：`$outputsize` 可能是 `INTEGER2`（要 `6,6` 只给一个会被忽略）；`sbsrender info` 给不出 min/max/default（要滑块范围得走 pysbs，本机 12.x 的 pysbs 在 py3.12 上缺 `distutils`，没接） |


## 四、怎么用 / 怎么起

```cmd
wuwor l_tex_nodes  -- l_tex_nodes_selftest      :: 自证（11 组，逐像素）
wuwor l_tex_nodes  -- python -c "from l_tex_nodes import ops; print(ops.noise(256,256,'fbm',48,5,7))"
```

- **画布上**：`http://<入口>/mindmap-mmd/pygraph?name=<工程名>%2Fdemo%2Flava_flow&tab=viewer2d`
  （⚠ 入口地址首段必须是**真工程名**，别名 `pro` 只在图内引用里成立 —— 写 `?name=pro%2F…` 服务端 400），
  或新建图 → 左侧调色板 `l_tex_nodes` 分类。
- **宿主怎么挂上**：`l_mindmap_mmd` 的 `requires` 带本包，且 `server._node_engine()` 启动时
  `discover_nodes("l_tex_nodes.nodes")`。**为什么默认挂而不是靠 env**：服务卡片只能带启动参数、
  带不了环境变量。缺包时静默跳过；要关掉设 `L_MINDMAP_TEX_NODES_OFF=1`。
- ⚠ **改本包代码后要 reload 宿主**（`wuwo svc reload l_mindmap_mmd_server`）——
  它在服务的 rez 环境里，热更只盯 `l_mindmap_mmd` 自己的源码目录（同 `l_node_engine` 那条坑）。

**示例图 `demo_tex_basics`**（11 节点，**不需要任何密钥**，全 CPU）：

```
Input(seed) ─┬─→ TexNoise(fbm 48 / 5 层)
             │                        └─→ TexBlend(multiply) ←─ TexGradient(linear 90°)
             └───────────────────────────→ TexWarp(45°) → TexLevels → ┬→ TexNormalFromHeight → TexPreview / TexExportImage / Output
                                                                       └→ TexPreview
```

实测：256×256 整条链 **132 ms**（每个节点 3–5 ms）—— 所以"改参数重算"是即时的手感，
**增量烹饪（只重算脏节点）暂时不急**（图更大时再说，见 §6）。

## 五、踩过的坑（都是实测）

1. **HSL 的色段判断必须互斥**：`hp<1 / hp<2 / …` 是**重叠**条件，用 `np.where` 逐条覆盖会让后面的段
   盖掉前面的（实测：纯红 +120° 变成**蓝**）。要用 `hp >= 上一段` 切开条件 + `np.select`。
2. **普通节点返回元组，带 ui 的返回字典**：`{"result": (…), "ui": …}` 是"要内联预览"的形状
   （宿主会拆 `result`，把整只 dict 当值传下游会炸）。第一版我把所有节点都写成字典，于是
   下游拿到的 `outputs[id][0]` 直接 `KeyError: 0`。
3. **引擎 guard 是三个位置参数**：`guard(node_id, class_type, resolved_inputs)`；
   写成 `lambda nid, node, **kw` 会 TypeError（自证里就踩了）。
4. **`scale` 的语义别想当然**：本包是 **>1 = 放大**（采样坐标除以 s），我第一版 docstring 写反了。
   自证用"两个亮点的间距"判单点灰度可靠。
5. **低频图上测 `warp` 会假失败**：默认位移只有零点几像素，均值差被抹平；要拿**高频条纹**量最大差。
6. **预览会写文件，得修剪**：贴图在实时模式里"每个按键重跑一次"，每轮留一张准撑爆媒体目录
   → `_write_media` 对自动命名的预览剪到最新 40 张（**只动 `preview_*.png`**，别的文件一律不碰）。
7. **`forceInput` 决定"能不能手填"**：引擎把 `IMAGE` 判成"文本框 + 手柄"（能填路径也能连线）——
   这正好和 `as_image()` 认路径一致；要"必须连线"的端口（如 mask）才加 `forceInput`。
8. **折回坐标的 float32 进位会越界**（2026-10-16 做 `lava_flow` 时炸出来的，**真 bug**）：
   `np.mod(np.float32(-1.5e-15), 1024)` 会舍成 **`1024.0`**（`1024 − 1.5e-15` 在 float32 里就是 `1024`）
   ⇒ `floor` = 1024 ⇒ `IndexError`。触发一点都不苛刻：`TexDirectionalBlur(angle=90)` 里
   `dx = cos90° = 6.1e-17` 乘上**负**半径就造出这种微小负值（`tiling=repeat` + 1024² 必挂；
   angle 不是 90 时随机挂）。`mirror` 那条同源（退化成 `-1`，**不报错但取错像素**），
   numba 那版更坏 —— 关着边界检查，**静默越界读**。
   修法：折回后**把"等于周期整数倍的值"按 0 处理**（周期语义下 `1024 ≡ 0`，丢的分数量是 1e-15 级）；
   `ops.sample_bilinear` 与 `accelerate._sample_px` 两处都改。自证：`_check_geometry` 里
   `x=-1.5e-15`（repeat / mirror）＋ `x=-1` 应当落在最后一列。

## 六、2D 取景器（已做，2026-10-12）

Substance 那个"点节点看结果"的 2D 视图。**贴图节点不必先接 `TexPreview` 就能看**：

- **入口**：节点卡片头上的 **👁**（按钮**只出现在输出里有 `IMAGE` 端口的节点上**）。
- **取景器**：夹在画布与属性面板之间的第三栏（`[画布 | 取景器 | 属性面板]`）——
  一边看图一边调参数；要更大按 ⛶ 铺满画布区；Esc / ✕ 关掉。滚轮缩放（以光标为锚点）、
  拖拽平移、双击=适应窗口、`1:1` 看 texel（放大时 `image-rendering: pixelated`）、
  棋盘格底看透明。**通道切换 RGB / R / G / B / A 在浏览器侧做**（宿主只给一张 RGBA PNG）。
- **数据从哪来**：宿主 `GET /api/pygraphs/render?name=<图>&node=<id>` —— **上一次运行**的输出，
  **内存里按需渲染**（长边 ≤1024、RGBA），响应头带 `X-Tex-Size` / `X-Tex-Mode`，`no-store`。
  这样**不给每个节点落盘**（实时模式每个按键重跑，落盘会撑爆媒体目录）。
- **两类节点的分流**（取景器自己判）：
  - 贴图节点（输出是 `TexImage`）→ 走上面那条 render；
  - **输出是文件路径的**（如 `TexExportImage`、AI 节点）→ render 会 **404** → 回退用它的
    `ui.image`（媒体 URL `/api/pygraphs/media/<名>`）✓ 实测两类都显示正常。
  - 没跑过 → render 404 → 提示"先点一次运行"。
- 跑完图**自动刷新**（编辑器在 `result` 事件里把 `viewerKey` +1）。

⚠ 实现时踩的两个坑（都写进代码注释了）：
1. **`ImageBitmap.close()` 之后 `width/height` 会变成 0** —— 尺寸必须在 close 之前取，
   否则 `getImageData` 报 "The source width is 0"。
2. **取图失败要连 `meta` 一起清** —— 只清 error 的话，头部会挂着**上一张图**的尺寸/模式。

### 6.2 节点缩略图 + 算子库示例图（2026-10-12）

两处"小图"，都是**看**的入口（点 👁 才是看大图）：

| 位置 | 内容 | 怎么来的 |
|---|---|---|
**节点卡片底部** | 这个节点**上一次运行**的输出（≤128px） | 宿主跑完就算好放内存，画布取 `/api/pygraphs/render?thumb=1`。**没跑过不请求**（否则必然 404，白打一堆） |
**左侧算子库** | 每个贴图算子的**示例小图**（96px） | 宿主用下面的 `samples.py` **现烤**（`/api/pygraphs/palette/<类名>`，烤一次缓存） |

**为什么现烤而不是放图标文件**：图标要手工维护、改算子就过期；现烤看到的**就是当前代码的真实输出**，
顺带成了"每个算子都能独立跑通"的回归（自证里逐个真跑一遍 ✗ 写错端口名就报）。

- 示例表在 **`samples.py`**：`SAMPLES[类名] = {节点id: {class_type, inputs}}` —— 就是普通的 pygraph。
  滤镜类统一喂**标准素材** `SOURCE` —— 拿一片纯色去看"模糊/色阶/法线"是看不出差别的，
  这是这张表里唯一值得讲究的地方。
  ⚠ **素材本身不能糊**（2026-10-16 换过一次，踩过）：老素材是单层 perlin，实测 mean .500 / **std .070** /
  范围 0.31~0.71 —— perlin 原始幅度天生只有 ±0.48（`ops._perlin_noise` 的 `*0.5+0.5` 只吃到半幅），
  于是"保结构"的算子跑完跟素材**几乎逐像素一样**（`TexBlur` .076 / `TexDirectionalBlur` .078 /
  `TexCrop` .079 / `TexChannelSplit` .093），调色板一排格子长成一个灰糊，**完全看不出算子干了什么**。
  现在 `SOURCE` 是一张合成小图（`TexNoise` 粗的 perlin → `TexLevels` 把对比拉到近满幅 →
  `TexShape(rect, hardness=0)` 当硬边方块 → 方块内 `TexGradient` 压着噪波、方块外是噪波，
  `TexBlend(normal, 0.7)` 合起来），一处满足三件事：**硬边**（模糊/锐化/边缘/曲率/倒角/距离 一眼可见）、
  **大片明暗**（levels/threshold/quantize/渐变映射 有东西可压）、**细纹理**（warp/transform/通道 不面对纯色）。
  实测：素材 mean .284 / std .208；35 张示例图**没有一张 std < .10**（`TexNoise` 那条改用
  `value, scale 12`、`TexDistance` 改喂硬边遮罩 —— 见下面两点）。
  自证仍是一条：`wuwor l_tex_nodes -- l_tex_nodes_selftest`（示例表逐个真跑）。
  ⚠ 改 `samples.py` 后**宿主要重启**：示例图在 `_PALETTE_CACHE`（进程内），不重启还是旧图
  （`wuwo svc reload l_mindmap_mmd_server`）。
- **产图节点画的是自己的输出，不是图标**（用户口径 2026-10-16：「**噪波节点就应该显示噪波本身的形状**」）
  —— 所以 `TexNoise` 的示例特意用 `value, scale 12`（96² 下 std **.223**，一眼看得出细胞状图案）；
  老那组 `fbm, octaves=5` 多层按 1/2 加权平均把对比拉平（std **.058**），小格子里就是一块灰糊。
  想让噪波整体更"够"得修 perlin 的半幅归一化（`ops.py:256`，会改所有噪波的像素值），那是另一件事。
- **"跑通"不等于"看得懂"**（2026-10-16 补上的两道判据，`selftest._check_samples`）：① 每张示例图
  `std ≥ 0.10`（**无豁免**）；② 彩色类示例必须真有通道差（R−B 平均差 > 0.05）。这两条正是老判据
  （只查"无 node_errors + 出的是 TexImage"）拦不住的东西 —— 那版判据放过了"一排灰糊"好几个月。
  顺着这两条修掉的四处（都留了"为什么"在代码注释里）：
  · `TexNoise` → `value, scale 12`（std .058 → **.223**）；
  · `TexHSL` / `TexChannelSplit` / `TexChannelMerge` → 改喂**彩色素材** `COLOR`（素材过一道
    `TexGradientMap`）：灰度上 `hue`/`saturation` 恒为空操作、三路拆出来一模一样、R/B 对调不可见
    （实测彩色化之后 `TexHSL` R−B 差 **.348**、`TexChannelMerge` **.277**）；
  · `TexDistance` → 喂**硬边圆遮罩**（`TexShape hardness=0`）：它内部就是 `> 0.5` 两值化，
    拿"有明暗的素材"当输入等于白给（实测：喂素材与喂"素材过阈值"**逐像素完全相同**，std .092
    一片雾；改遮罩后 **.120**）。
  ⚠ 顺带更正一条我先前写错的判断：`TexCrop`（48²）/`TexResize`（40²）**没有被 CSS 裁** ——
  `.pyg-pal-thumb` 是 26×26 且图也是方的，`object-fit: cover` 对方形**不裁**（实测两边都是 26×26）。
- **不产图的那些用 SVG 图标**（2026-10-16，用户口径「功能性节点使用 svg」）：`FxQuadrant` /
  `FxIterate` / `FxSwitch` / `FxTile` / `TexSbsar` / `TexLoadImage` / `TexInfo` / `TexExportImage` /
  `TexMaterial` / `TexExportMaterial` / **`Mg*` 运动图形 19 个** —— 它们烤不出示例图（要磁盘文件 /
  产物是路径·材质字典 / 要外部 Substance 引擎 / 出的是**矢量场景**而不是栅格图），调色板那格以前只有名字。
  图标**直接写在节点类里**（`BaseNode.ICON`，SVG 源码，`currentColor` 跟主题），契约与消毒分工见
  `l_node_engine.md` §3.0f；图类节点照旧画现烤图（**真实输出**比图标信息量大）—— 两者同一个位置，
  有图就不用图标。
  判据：`wuwor l_tex_nodes -- l_tex_nodes_selftest` 的「调色板图标」—— 查的是**互补不变量**
  「**有图标 ⟺ 没有示例图**」（不再点名，2026-10-17 改：`Mg*` 那 19 个就是点名清单漏掉的）。
  入口那条：`l_tex_nodes` 出的图要是**矢量/场景**（`Mg*`）或者**要外部文件**，就别给它配示例图，
  配图标 —— 两边只能有一样，否则那格要么空着、要么白写。
- `TexLoadImage` / `TexExportImage` / `TexInfo` / `TexMaterial` / `TexExportMaterial` **故意没有示例**：
  第一个要磁盘文件，后三个的产物分别是**文件路径 / 材质字典 / 路径列表**（**都不是图**），那一格没图可放。
  调色板里它们只显示名字（前端 `onError` 塌成空，不留破图标）。
  （`TexMaterialChannel` **有**示例 —— 它那张图里先搭了一个 `TexMaterial`，顺便把材质打包也跑了一遍。）
- 门槛：示例图要跑节点，所以和跑图同一个开关（`L_MINDMAP_ALLOW_RUN`）—— 关着执行面的实例
  （公网那张卡）调色板只显示文字，不影响画布。

### 6.2b `2d_view` + 底栏「🖼 2D 视图」的兜底取景对象（2026-10-16）

用户口径：「节点库添加一个 2d 预览节点，没有选择节点时 2D 视图显示这个 2d 预览节点」。

- **节点**：`l_tex_nodes.nodes.View2D`，**注册名 `2d_view`**（故意不带 `Tex` 前缀 —— 编辑器按键名认它）。
  `IMAGE` 入 → 原样透传（`(image,)`，零副作用、不落盘、不给 `ui`）；`category` 仍是 `l_tex_nodes`，
  所以它就长在节点库那一栏里。**一张图只允许一个**：类上标 `SINGLETON = True`，引擎
  `validate.validate_structure` 认这个标（多个 → `not_singleton` 错），编辑器那三处落节点的地方
  （点节点库 / Tab 搜索 / 拖到画布）也各拦一道，复制节点同样拒。
  · **2026-10-17 收 `SCENE`**（用户口径「修改 `2d_view` 节点的功能，可以显示运动图形图」）：输入端口
  放开成 `*`，接 `Mg*` 的场景**原样透传**（宿主不光栅；编辑器走 `mgGpu` 矢量实时光栅，GPU 腿按
  `SCENE→IMAGE` 隐式转换出纹理，`/api/pygraphs/render` 需要 PNG 时才 Pillow 光栅一次）。
  输出端口也标 `*`（上游是什么就是什么）。所以 MG 图里摆一个它、把 `MgPreview` 接上，
  不选中节点时底栏那一格就是这张运动图形。端口类型是 `*` ⇒ 谁都能接（接模型/字符串会自己报错）。
  · 加**新单例节点**的口径：节点类自己声明 `SINGLETON`，引擎侧不再点名任何具体节点（与 `_STANDALONE_OK`
  那种"点名表"分开）；自证 `l_node_engine -- l_node_engine_selftest` 的"结构校验"组里有个探针节点钉住它。
- **编辑器兜底**：底栏「🖼 2D 视图」原来只跟**当前选中节点**走（没选中 = 空着给一句提示）。现在
  看谁 = **选中的节点** > 图里的 `2d_view` > 空（提示）。所以"放一个 `2d_view` 接上要看的图"之后，
  不用一直点着那个节点也能看它 —— 点别处取消选择，那一格回到它。
  · 实现：`PyGraphEditor.jsx` 的 `VIEW2D_CLASS` + `view2dNode` / `view2dId`（`selected || view2dNode`），
  与节点卡片上那个 👁 是**同一个取景器**（`TexViewer`，`docked`）。
- **自证**：`node tools/web_probe.cjs view2d`（12/12：没选中时看 `2d_view` / 跑一轮真出图 96x96 /
  选中优先 / 重开图回到兜底 / 单例拦第二个 / **真图 `demo/fire_time` 也吃到了**）；
  `node tools/web_probe.cjs mggpu`（7/7：运动图形那条 **`2d_view` 接 `SCENE`** ⇒ 没选中时那一格
  显示它、且是 `webgpu` 纹理、零宿主 `/run`）；
  引擎侧 `l_node_engine_selftest` 的结构校验组；贴图侧示例图 `samples.SAMPLES["2d_view"]`
  （调色板里那格有图）随 `l_tex_nodes_selftest` 一起跑。
- **示例纹理图全都补上了**（2026-10-16，用户口径「请给所有的程序纹理图都补上这个节点」）：10 张
  「给人看」的纹理图各接了一个（落点 = 源节点正右方一列 `x+360`）——`Functions/demo_tex_basics`(7)、
  `demo_tex_batch2`(10)、`demo_bamboo_weave`(hsl)、`demo_math_2d`(7)、`demo_fxmap`(5)、
  `demo/bloom_flower`·`lava_flow`·`fire_time`·`fire_keys`(都是 `out`)、`demo/demo_sbsar_patterns`(14)。
  **压测图**（`demo_perf_*`）与 **FX 内层子图**（`demo_fxmap_pattern/iterate/fxmap`）**故意没加**
  （前者是尺子、后者是被内联的图案图，钉住没意义）。
  批量工具：`wuwor l_mindmap_mmd -- python <包>/999.0/tools/add_view2d.py --path <图> [--path …] [--dry|--check]`
  （挑"最终纹理"的四条口径：`Output` 链 → `TexMaterial` 主通道 → 节点 `out` → 最右终端产图节点；
  写盘前跑引擎校验，**新出现的**错/提醒就整张拒收；`--check` 只复检盘上的图）。
- **缩略图（内容浏览器那个封面）也认它**（2026-10-16 用户："为这些资产图生成缩略图"）：
  宿主 `GET /api/pygraphs/asset-thumb` 的"封面挑谁"改成 **① 图里有 `2d_view` 就用它的小图**
  （= 底栏 2D 视图那张 = 这张图的最终纹理），**② 没有才**回退老的"拓扑序第一个有小图的节点"；
  响应头带 **`X-Tex-Cover: 2d_view|first`**。命令行 cook（与前端工具栏 `🍳N` 同一套口径）：
  `wuwor l_mindmap_mmd -- python <包>/999.0/tools/cook_assets.py --addr <工程/相对路径> [--addr …]`
  —— **只认 `X-Tex-Cover=2d_view` 才算烤好**，`first` 会重烤（否则那份老封面永远换不成新口径）。
  上面那 10 张图**都已 cook**（每张封面都是 `2d_view` 那张）。
  ⚠ 顺手修了一个挡路的坑：`demo_fxmap` 里有一处**图内子图引用写的是老工程名**
  （`lugwit_agent/Functions/demo_fxmap_fxmap`）—— 工程改名后它必然 `GraphNameError`（跑不起来、
  也就没封面）。**图内引用一律写 `pro/…`**（本工程别名，改名免疫），同图的另一处本来就是这么写的。

### 6.2c `3d_view` + 底栏「🧊 3D 视图」（2026-10-17）

用户口径：「我觉得统一为使用 `3d_view` 节点来显示结果」→ 追问后拍定：「**三个输入都接，
如果某个端口没接就使用 3D 视图的设置**」。所以 3D 视图从"图里找（第一个 `Input(model)` / 第一个
`MATERIAL` 节点）"改成"**节点说了算、没接的那路才回落**"：

| `3d_view` 的那一路 | 接了 → | 没接 / 图里没有 `3d_view` → |
|---|---|---|
| `model`（字符串，可连线） | 用它（连线时取源节点的**字面值**，`GetParam` 去图级 `params` 里查） | 面板「看哪个模型」下拉 → 旧兜底：图级 `params` 里第一条 `type=model`（`Input` 退休后模型住这儿）→ 再退还没迁移的 `Input(type=model)` 节点 |
| `mesh`（接出几何的节点） | 看那个节点的几何（`/api/pygraphs/mesh`） | 面板「来源」下拉（可手选，宿主 `/meshes` 报得出来的那些） |
| `material`（`MATERIAL`） | `MeshViewer` 默认贴它（`preferMat`） | 「材质」下拉（默认第一个 `MATERIAL` 节点） |

- **节点**：`l_tex_nodes.nodes.View3D`，注册名 `3d_view`，**单例**；`model`/`mesh`/`material` 全可选
  （任何一路都能单独驱动）；输出 `model` 字符串原样透传。**没有示例图**（输出不是图）→ 调色板那格
  靠 `ICON`（线框立方体）——`l_tex_nodes_selftest` 的"示例图或图标至少有一个"会盯住它。
- **实现**：`PyGraphEditor.jsx` 的 `VIEW3D_CLASS` + `view3dInfo`（逐路解析，`GetParam` 认图级
  `params`）+ `modelValue`（3d_view → params → 老 `Input`）；`Viewer3DPanel` 收 `meshNode`/`matNode`，
  `MeshViewer` 的 `preferMat` 让"接了哪份材质"赢过"第一个"。
- ⚠ **踩过的坑（一定记住）**：`Input` 节点类型已在 2026-10-17 退休（`Input` → 图级 `graph.params` +
  `GetParam`，见 `pySchema.migrateInputNodes`）——**打开就迁**。所以判"图里填了什么模型"要看
  `graph.params`，别再找 `Input` 节点（我的第一版就栽在这儿：探针图一打开 `Input` 变 `GetParam`，
  模型值"凭空消失"，3D 面板一直空）。
- ⚠ **GPU 迁表白名单**（`texGpu.js` 的 `KINDS` / `SINK_KINDS`）也要加：`canRunGpu` 要求"这段里
  每个节点都迁了"，缺一个就**整段退回 CPU**（`2d_view` 2026-10-16 已经踩过一次）。
- **自证**：`node tools/web_probe.cjs view3d`（9/9：`model` 接上 → 真渲染 `torus.obj` 且「来源」写着
  "来自 3d_view 节点"；**老图的 `Input(model)` 兜底仍能看**；单例拦第二个；**真图 `demo_bamboo_weave`**
  → `weave_plane.obj` + 材质默认就是 `mat`）。
- **示例图**：`demo_bamboo_weave` 补了 `v3d`（`model ← GetParam(model)`、`material ← mat`）；
  `demo_math_3d` 补了 `v3d`（`mesh ← 节点 3` 的参数曲面，来源下拉仍可换别的）。

### 6.3 3D 预览贴 PBR（已做，2026-10-12）——「竹编」示例

入口两个，**同一个 `MeshViewer`**（`docked` 属性区分浮层/停靠 —— 光照、贴图、取景只此一份）：
① `Input(type=model)` 节点上的 **👁**（浮层）；② 底栏标签 **「🧊 3D 视图」**（常驻，不必去点节点；
切到这个标签时底栏会自动加高到 340px，否则模型就一个小点）。
⚠ **"看哪个模型 / 哪段几何 / 哪份材质"自 2026-10-17 起由 `3d_view` 节点说了算**（没接的那路才回落：
图级 `params` 里 `type=model` 那条 / 面板「来源」下拉 / 第一个 `MATERIAL` 节点）—— 见 §6.2c；
下面这段是"材质怎么贴上去"的实现（与"看谁"无关）。
以前它只给模型套一个**单色** `MeshStandardMaterial`；现在**把节点图产出的材质实时贴上去**：

- **材质从哪来**：宿主跑完图时把 `MATERIAL` 端口那份字典（`{通道: TexImage}`）按 图 + 节点 存进内存
  （`server._cache_materials`），新端点 **`GET /api/pygraphs/material?name=&node=[&channel=&max_side=]`**：
  不带 `channel` 回 `{channels:[…]}`（这次产了哪些通道），带 `channel` 回那张通道的 PNG
  （响应头/`no-store` 与 `/render` 同一套）。**不用先导出 PNG** —— 导出是写盘动作，而调参要连着看几十次。
- **通道怎么映射**：`basecolor → map`（sRGB）、`normal → normalMap`、`roughness → roughnessMap`、
  `metallic → metalnessMap`、`ao → aoMap`；⚠ `aoMap` 要**第二套 UV**（新 three 叫 `uv1`、老的 `uv2`），
  没有就从 `uv` 复制一份，否则 AO **静默不生效**。`height` **不**接 `displacementMap`（低模会炸刺），
  照常导出 —— 要真起伏得先换细分够密的模型。
- **浮层头部多一条**：**材质下拉**（当前图里输出 `MATERIAL` 的节点；跑过一次才有内容，默认选第一个）、
  **平铺 N**（UV repeat，织席这种重复图案调大更像一整张）、**↻ 取贴图**（重跑一轮后手动拉一次）。
- **单通道查看**（看"这套图里某一张到底长什么样"）：材质条上一排小按钮（材质 / 基色 / 法线 / …，
  点谁看谁），键盘 **`C` 循环下一个通道、`M` 回材质模式**（在视图里直接按；输入框里打字不抢键）。
  实现：贴图**只下载一次**存在 `chanTexRef`，切通道只是换"挂哪个槽" —— 单通道时把它当基色、
  金属/粗糙压平，别的槽全关；`height` 这种不挂材质的通道也**按需现拉一张**（C 键能循环到它）。
  在 2D 取景器里也有同一件事（RGB / R / G / B / A 通道切换）。
- ⚠ **光要中性**（实测踩到）：原来那盏蓝色补光打在翘起来的竹条上是一片**青斑**，看着像法线坏了；
  现在是灰蓝小补光 + 中性主光。
- **示例**：`<pygraphs>/demo_bamboo_weave.json`（**斜纹 twill** 竹编：条子 45° 交叉、2 上 2 下一循环 →
  沿条子方向的**竹纤维线**（`sin((x±y)·π·200 + 噪波相位)`，比"方向模糊噪波"清楚得多）→ 高度 →
  法线 / 曲率→AO / 粗糙度 / 金属度 / 竹黄基色 → `TexMaterial` → `TexExportMaterial` 出 6 张 PNG），
  载体用内置 `<pygraphs>/models/weave_plane.obj`（2×2、40×40 细分、带 UV 的微起伏平面，
  由脚本按 `z=A·sin(kxπx)·cos(kyπy)` 生成）。跑一轮 → 3D 预览里那个材质节点自动贴好。
  ⚠ 想改这张图**先刷新页面（F5）**：编辑器不会自己重载磁盘上的图，旧标签页一保存就会把它盖回去（踩过）。
- **法线接缝的两个来源**（都踩过，别再用）：① 条子截面写成 `abs(sin(...))**0.6` —— 谷底是**尖角**
  （导数不连续），法线上就是一道硬缝；换成 `tanh(6*sin²)`（光滑平台：顶平、肩圆、谷平）就没了。
  ② 上下交替用 `where(...)` 硬切 → 切换处高度**跳变**，法线上是一道亮/暗线。现在把纤维也**只放颜色**
  （`soft_light`）+ 只给高度留 2%（`fxlv` 0.98~1.0）—— 高度一干净，法线就干净。

### 6.4 结果缓存落盘（已做，2026-10-12）——「重新打开一张图，不用再跑一遍」

用户口径：**节点结果要缓存，不用每次打开重算；但要看得出这是缓存数据**。

- 内存里那两份（`_TEX_RENDER` 图 / `_TEX_MATERIAL` 材质）**一重启就没了** —— 以前的观感是
  "关掉进程再打开一张图，节点卡片一排空白，只能再跑一遍"。现在按图落盘：
  `<pygraphs>/_renders/<图名>/meta.json` + `n<key>.t.png`（缩略图）/ `n<key>.png`（大图）/
  `m<key>.png`（材质通道）。**节点 id 只做哈希当文件名**（嵌套子图的 id 带 `/`，不能直接进路径）。
- **落盘节奏**（有意的取舍）：
  - **缩略图：跑完就落**（几 KB 一张，一张图几十个节点也就几百 KB）；
  - **材质通道：跑完就落**（≤512，6 通道 ≈ 2MB —— 换来"重启后 3D 预览照样有贴图"）；
  - **大图：第一次被看才落**（1024² 的 PNG 一张 1~4MB，全落一遍既慢又占地方，实时模式下每轮都在写）；
    ⚠ 只落过缩略图时看大图会**回退给缩略图**，响应头写 `X-Tex-Cached: thumb`（不假装它是大图）。
  - 目录按 mtime 留最新 `_RENDER_DISK_MAX_GRAPHS=60` 张图；**删图时整个目录一起删**。
- **过期判定**：`meta.graph_sha1` 记跑那一刻图文件的内容哈希。图改过 → 缓存标 `X-Tex-Stale: 1`
  （`/api/pygraphs/renders` 也回 `stale:true`），前端写"缓存（旧）"，**绝不假装是新算的**。
- **UI 上怎么标"这是缓存"**（三处）：
  - 节点缩略图右上角 **🕗 角标**（缓存 = 蓝、过期 = 琥珀），tooltip 写清"上次运行留下的、没重算"；
  - 2D 取景器头部一枚 **🕗 缓存** 药丸（读 `X-Tex-Cached` 响应头，`thumb` 时写明"只有缩略图"）；
  - 3D 预览的材质行尾巴 **🕗 缓存**；打开图时日志里也留一句（每个"图+那一次运行"只说一次）。
- 新接口 `GET /api/pygraphs/renders?name=` → `{source: memory|disk|none, ts, stale, nodes{thumb,big}, materials{channels}}`
  —— 节点卡片是 `<img>`、读不到响应头，所以**问一次**比逐张探图划算。

### 7.2 增量烹饪（已做，2026-10-12）

改一个参数不再全图重跑：**结构没变的节点直接复用上一轮的值**。实测（1024²，fbm 8 层 → 模糊 → 变形 → 色阶 → 法线，5 个节点）：

| 场景 | 耗时 | 复用 |
|---|---|---|
冷跑（全新图） | 2816 ms | 0 |
**参数没变** | **8 ms** | 5/5 |
**只改下游**（色阶 gamma） | **265 ms** | 3（上游那三步重活全复用） |
改上游（噪波 seed） | 3051 ms | 0（脏按拓扑往下传） |
关掉增量（全量） | 2770 ms | 0 |

（第一版没做"复用节点不重渲染"那步优化时，热跑是 418 ms、改下游 457 ms —— 那几百毫秒全是
**给取景器重渲染 PNG** 的冤枉钱，见本节末尾。）

**键是"结构指纹"，不是内容哈希**：`指纹 = class_type + 自己的字面量参数 + 上游节点的指纹`（`l_node_engine.graph._node_fingerprint`）。
为什么不做内容哈希：一个 `TexImage` 十几 MB，每轮给每个节点 sha1 一遍（几十 ms × N）比重新算还贵；
而"参数没变 + 上游没变 ⇒ 这一步的输入一模一样"是**递归成立**的（算子不原地改写输入 ✓）。
参数里塞了**对象字面量**（不是连线来的）时会退化成 `repr`（带地址）→ 每轮都重算（保守，宁可重算也不许拿旧结果冒充）；
想让它可缓存就给那个对象加 `__fingerprint__()`。

**四条保守规则**（都在引擎自证里盯着）：

1. **循环体不进缓存**：体内的节点是带 `iteration` 执行的，每轮值本来就不同 —— 循环**整段**（含两个括号）跳过；
2. **`bypass` / `mute` 的节点不进缓存**：它们的输出取决于**运行时的值**（"第一个有值的输入"），指纹里没有；
3. **上游指纹未知的节点不进缓存**（比如它的上游在循环体里）→ 连带它的下游一起重算；
4. **`cache=None` = 老行为**（一字不变）：缓存策略不在引擎，宿主给就启用、不给就全量。

**宿主这边**（`l_mindmap_mmd`）：`_RunCache` 按**图名**存 `节点 → (指纹, 值)`（上限 120 个节点，满了淘汰最早的）；
画布上有个开关 **`🧩 增量：开/关`**（浏览器偏好，存 localStorage，跑图时作为 `incremental` 传下去）——
关掉就是每次全量重算。复用的节点在画布上挂 **`🧩`** 徽标、日志里报一句"复用 N 个，只重算了 M 个"。

⚠ 与另外两个缓存**不是一回事**：`_TEX_RENDER`（取景器/缩略图看的那份图，只留最近一次）、
`_PALETTE_CACHE`（算子示例图，烤一次）。别混。

⚠ 复用的图**不重渲染**（`_cache_renders(..., keep=cached_nodes)`）：它们与上一轮逐像素一致，
1024² 渲一张 PNG ~80ms × 一堆节点，正好把省下的时间又赔回去（实测踩到，改了才拿到上表的数字）。

### 7.3 第二批"更硬的算子"（已做，2026-10-12）

加了 Substance 里那类"再往前一步"的：`TexDirectionalBlur` / `TexEdgeDetect` / `TexDistance` /
`TexBevel` / `TexFloodFill` / `TexSplatter` / `TexGradientMap` / `TexHistogramScan` / `TexQuantize` /
`TexPixelProcessor`，以及**多通道材质**三件套（`TexMaterial` / `TexMaterialChannel` /
`TexExportMaterial`）—— 24 → **37** 个节点、示例图 21 → **32** 张。

**现成整图示例：`<工程>/demo/lava_flow.json`（2026-10-16，「流动的岩浆」）** —— 把上表那批算子
串成一张**能动**的图（25 节点 / 1024² / 全链只用 GPU 可跑节点 ⇒ `▶` 动画跑在 WebGPU 上）：

```
TexNoise(fbm scale4) → TexResize(1024,repeat) → TexWarp → TexTransform(←Time·慢) → TexLevels   :: 地壳大板块
       └→ TexEdgeDetect(sobel 4.5) → TexLevels(0.42~0.62,γ1.8) → TexBlur → TexMorphology(erode1) :: 细亮缝
TexNoise(fbm scale7) → TexResize → TexDirectionalBlur(78°,40) → TexWarp(108°) → TexTransform(←Time·快) → TexLevels
       └→ TexBlend(multiply 缝×熔流) → TexBlend(screen 0.14)  :: 缝里的热
TexLevels(冷壳压到 0.008~0.07) → TexBlend(screen 热) → TexGradientMap(岩浆调色板) → TexBlur(10) → TexBlend(screen 0.2)
```

⚠ **口诀：壳要暗、缝要细** —— 第一版把"环境热"给到 0.42、缝又宽，整体一提亮立刻变成**火焰**
（截图对比过）；把冷壳压到近黑、缝收窄（阈值窗口 + `erode`）、泛光降到 0.2 之后才是岩浆。
⚠ 驱动只有 `Time → Calc(*, -0.055 / -0.012)` 两条：**熔流向上漂、地壳慢慢挪**；`Time.fps`
（这张图给 60）定的是 `▶` 动画的目标帧率。
实测（`wuwor l_mindmap_mmd -- python <包>/999.0/tools/demo_graph_check.py --path <工程>/demo/lava_flow.json`
两跑对比）：两帧 **44% 像素在变**、
1024² 上 **GPU work 13ms ⇒ 49.5 fps（算力上限 156 fps）**；CPU 那条腿整图重算 ~2.1s/帧
（所以这张图默认看的是 GPU 那条；CPU 档只适合改参数看单帧）。

**现成整图示例：`<工程>/demo/bloom_flower.json`（2026-10-16，「盛开的花朵」，24 节点 / 1024²）**
—— 演示"**一朵花自己开**"：全靠**阈值**推着长，没有关键帧、没有逐帧数据。

```
TexGradient(angular) ─┐
                      ├→ TexPixelProcessor(expr="clip((b - 0.62 + 0.30*cos(a*31.4159))*9, 0, 1)")   :: 5 瓣玫瑰线
TexGradient(radial) ──┘        a=θ、b=r；`cos(5θ)` 是**周期**的 ⇒ θ 绕一圈花瓣刚好接上、不留缝
Time → Calc(*0.75) → Calc(+1) → Calc(1/x) → Calc(+0.06)  :: 绽放驱动 at = 0.06 + 1/(1+0.75t)（渐近饱和）
TexThreshold(pet, at=↑, softness=0.3)                    :: 亮区从花心往外长 = 花瓣一片片展开
TexEdgeDetect(pet) → TexBlur → TexBlend(subtract)        :: 脉络（花瓣轮廓）作减法压进花瓣
TexShape(disc 0.09) → TexBlur → TexBlend(screen 0.6)      :: 花心
TexLevels(radial, 0.34~0.92) → TexBlend(multiply)         :: 花瓣的径向明暗（花心亮、瓣尖暗）
Time → Calc(*4) → TexTransform(rotate=↑)                  :: 整朵慢转 4°/s
TexLevels(radial → 0~0.035) → TexBlend(max)               :: 暗底（径向渐变压成的微亮场）
→ TexGradientMap(紫→品红→粉→白) → TexBlur(22) → TexBlend(screen 0.15)  :: 上色 + 泛光
```

⚠ **三条踩过的**（都跟这个案例直接相关）：
1. **阈值驱动的上界必须自带**：第一版 `at = 1.05 − 0.22t` 一路减到负数 ⇒ `pet ≥ at` 恒真 ⇒
   整幅被"花瓣"淹成一块大色斑 ✗。改成 `at = 0.06 + 1/(1+0.75t)`（t→∞ 停在 0.06）⇒ 既**渐近饱和**
   又永不泛滥（`pet` 在花外恒为 0 ⇒ `0 ≥ 0.06` 不成立）。`Calc` 只有 `+ − × ÷ // % **`，
   要"夹住"就得靠**驱动函数本身**有界，或者用 `Compare + Select`。
2. **`radial` 渐变的中心是 1、边缘是 0**（`v = 1 − dist/(0.5·maxsize)`）—— 花瓣场写成
   `(0.62 + 0.30cos − r)` 就**把花心反排除在外**（中心 r=1 判成"最远"）✗；要 `(r − …)`。
   查这个别靠猜：`demo_graph_check.py --dump rad pet mask` 直接把中间节点的均值/中心值打出来。
3. **静帧要喂时钟**：宿主不喂时 `Time` 用**引擎自走**那份（每次跑图帧号 +1、`time = frame/24`）
   ⇒ 第一遍永远是 `t = 1/24 s` ⇒ 花朵静帧全是"**还没开**"（第一版就是这么误判的）。
   `execute_graph(..., clock={"time": 秒, ...})` 喂进去才截得到"开到一半"；
   `demo_graph_check.py --time <秒>` 就是干这个的。

现成示例图：`/mindmap-mmd/pygraph/demo_tex_batch2`（噪波 → 直方图扫描 → 边缘 → 模糊 → 渐变映射；
遮罩 → 倒角 → **按倒角的亮度散射** → 量化 → 逐像素 → 材质打包/取通道/导出，15 个节点）。
实测（256²，同一张图连跑两次）：

| 场景 | 耗时 | 复用 |
|---|---|---|
冷跑 | 206 ms | 0 |
**原样重跑** | **29 ms** | **15/15**（含那三个"导出"节点 —— 它们也没重写文件） |

⚠ 复用的 `TexExportMaterial` **不会重写 PNG**（值就是上一次的路径列表）。这是"结构指纹"的必然代价：
指纹只看"参数 + 上游"，看不出"你又跑了一遍"。文件没了就手动全量跑一次（关掉 `🧩 增量`）。

**这一批踩到的四个坑**（都在自证里钉住了）：

1. **`distance()` 一律加 0.5 偏置**（本意是让 `signed` 能表示负距离）→ 结果 `outside`/`inside` 的
   "贴着遮罩"也变成 0.5，断言与下游全对不上。**只有 `signed` 该居中**，另两个老老实实 0..1。
2. **`bevel` 想"以像素为半径"，就必须用像素距离**：`distance()` 是**归一化**过的（除以观测峰值），
   拿它算半径，图一大半径含义就变了。所以 `bevel` 直接用 `_chamfer`（本身就是像素）。
3. **每像素只做一点计算的算子，绝不能写 python 循环**：`_chamfer` 第一版逐像素循环，1024² 要秒级；
   改成 `minimum.accumulate(x − 3·i) + 3·i` 一次算完"这一行往左的最短路"（min-plus 卷积配线性斜坡），
   只有**行**是循环 → 几十毫秒。`quantize` 的噪点抖动同样（逐像素调 `_hash2` → meshgrid 一次算）。
   自证里给 1024² 那道**上限 5 秒**的护栏就是防这个回归的。
4. **"节点没注册上"常常不是注册的问题**：`registry.get_all_node_info()` 把每个节点的
   `get_node_info()` 异常**吞成一条 ERROR 日志**，界面上只表现为"这个节点不见了"。
   `ops.DITHER_MODES` 漏定义时就是这样 —— 现在自证发现缺失会把**真原因**抛出来。
   （顺带：`_hash2` 乘法是**故意**在 64 位回绕的，numpy 只对标量溢出告警，标量调用点已按掉。）

### 7.4 FX-Map / 子图实例化（已做，2026-10-12）

`TexFxMap` 是 `l_node_engine` 的 `SubGraph` 在贴图侧的**第一个用家**：把一张图 / 一份**图案子图**反复实例化。

- `mode = quadrant`：画布四等分递归 `depth` 层（`4^depth` 块，**矩形不重叠、正好铺满**），每块跑一次图案；
- `mode = iterate`：`iterations` 轮，**上一轮输出当下一轮输入**（子图里 `Input(name="image")` 拿它），
  再按 `blend_mode` 叠回上一轮；
- `pattern_graph` = 图案**子图名**（宿主读图目录内联成 `pattern` —— 与 `SubGraph.graph` 完全同一条路：
  `GRAPH_NAME_PORT` 声明 + `resolve_graph_names`；解析不了在跑图前就 400）；**留空**则用 `builtin`
  内建图案（`disc/square/gradient/noise`）或接进来的 `image` —— 所以单节点也能用，不必先建子图；
  **双击节点**（或点卡片上的「＋ 内层图」）= 建一张图案子图并接上再打开（Substance 那种用法）：
  骨架来自本类的 `GRAPH_SKELETON`（`/object_info` 的 `graph_skeleton`）—— 六个 FX 变量 `Input`
  + `TexGradient` → `Output(image)` + 一张写清变量含义的 `Note`；
- 变量 = 子图里 `Input` 节点的名字：`index` / `number` / `x` `y`（中心，0..1）/ `size`（边长占比）/
  `pos`（= `[x,y]`，一次拿一对），`$` 前缀可省；⚠ 只能喂给**能收值的端口**（`TexNoise.seed` 那种控件收不了连线）；
- 每档按 `t = (i+0.5)/N` 递进：位移 `offset × t`、旋转 `rotate × t`、缩放 `1+(scale-1)×t`、
  上色按 `hue_color(t)` 混 `tint`。

判据（自证第 9 组钉住）：象限矩形**不重叠且铺满**、逐档变换生效、迭代轮数变了输出跟着变且**确定性**
（轮内不引随机）、**图案子图按块数被调用 N 次**（事件计数）、变量真进了子图、跑完调用栈不泄漏。
`pattern` 与 `SubGraph.sub` 一样走**整体 sha1** 指纹（`register_hashed_inputs("TexFxMap", "pattern")`）
→ 改了图案子图**不会被复用旧结果**。调色板示例图见 `l_tex_nodes.samples.SAMPLES["TexFxMap"]`
（那一批的增量；现总数 = **39** 个基础 `Tex*` + **`2d_view` / `3d_view`** + `TexSbsar` + 4 个 FX = 46 个、调色板 **35** 张）。

**示例图（2026-10-13 晚补）**：`demo_fxmap`（文件在工程 `<真工程名>/Functions/` 下；图内引用可写 `pro/Functions/…`，但**入口地址首段必须是真工程名**）+ 它的图案子图
`demo_fxmap_pattern`（**与编辑器双击节点建出来的骨架同形**：六个 FX `Input` + `TexGradient` → `Output(image)`）。
主图三条支路一次演示三种用法 —— ① `builtin` 内建图案、② `pattern_graph` 指图案子图（带 `offset/rotate/scale/tint`
逐档递进）、③ `iterate` 叠 4 轮；末尾 `TexExportImage` 出 `demo_fxmap.png`。
自证：`POST /api/pygraphs/<真工程名>%2FFunctions%2Fdemo_fxmap/validate` → **0 错**（⚠ 首段写别名 `pro` 现在是 **400**）；
`…/run` → 200、**0.66s**、`node_errors` 空、三个 `TexFxMap` 各回 `TexImage(512x512 rgb)`。
⚠ 写这类资产的 `note` **别用 ASCII 双引号**（会把 JSON 写坏 → `/validate` 400 "Expecting ',' delimiter"，
实测又踩了一次），用「」。

### 7.5 FX 图节点族（照 Substance Designer，2026-10-14）

SD 的一个 FX-Map 图里**只有三类节点**（`Quadrant` / `Iterate` / `Switch`，见官方文档与 Overdraw 的
fBm 教程），节点**竖着连**。本包 2026-10-14 把这三类补齐（`fx_nodes.py`），口径照 SD：

| 节点 | 端口 | 语义 |
|---|---|---|
| `FxQuadrant` | **4 入 `in0..in3` / 4 出 `out0..out3`** | 一个节点 = 四叉树的**一层**：四个出就是本层的四个象限；`in_i` = **下一层**那棵子树（把上一个节点的四个出接回来 = 造下一层）。象限顺序 = raster：out0 左上 / out1 右上 / out2 左下 / out3 右下 |
| `FxIterate` | 2 入（`background` 左 / `pattern` 右）/ 1 出 | SD 的 Iterate：把右手的图案重复叠到左手底图上 `iterations` 轮，每轮按 `t=(i+0.5)/N` 递进 `rotate/scale/offset/tint` |
| `FxSwitch` | 2 入（`input0`/`input1`）+ `selector` / 1 出 | 选一个出去；**`selector` 是个端口**（接 `Input(index)` 或先过个取模节点 = SD 那个「Selector 由 Dynamic Function 给」的用法）；选中的没接 → **当场报错**（不静默出黑图） |
| `FxTile` | 4 入 `in0..in3` + `width/height` / 1 出 | 四个象限**拼回整图**（SD 的 unwrap）。**通常不用手摆**：`FxQuadrant` 当**根**时由引擎自动合成它（见下） |

**图级 `root`（"根就是这张图的输出"）**：`FxQuadrant` 声明 `ROOT_WRAP = ("FxTile", ("width", "height"))`；
引擎的 `graph_prompt`（`l_node_engine.pygraph._apply_root`）把根节点的出收成一个 `Output` ——
单出直接接，多出按 `ROOT_WRAP` 合成收口节点（`carry` 的字面量搬过去、其余口按 0..N 接上）。
`kind:"fx"` 的图**必须有 root**；`root` 缺失 / 指向不存在 / 收口不了 → 跑图与校验**当场报**
（宿主映射成 400 / `errors[].code = "graph_root"`）。
编辑器侧：`kind:"fx"` 的图**端口竖排**（输入在上、输出在下），根节点卡上挂 **🌳 根**，
右键节点「设为根」就是 SD 的 Set as Root。

**示例图**（工程 `<真工程名>/Functions/` 下，数据/资产目录里随包发；图内引用写 `pro/Functions/…` 仍成立）：

- `demo_fxmap_pattern`（**FX 图**）：FX 变量 `Input` → `FxSwitch`（`index` 驱动：偶圆盘 / 奇方块）→
  `q1`（第一层）→ 四个出接 `q2` 的四个入（第二层）→ `root = q2`；全图 4×4 = 16 块。
- `demo_fxmap_iterate`（**FX 图**）：径向渐变当底图 + 圆盘当图案 → `FxIterate`（5 轮、逐轮转 60°、
  缩 0.82、`screen` 叠）→ `root = it`（单出，不用收口节点）。
- `demo_fxmap`（外层，三条支路各一 `TexPreview`）：① 内建图案 ② 引 `demo_fxmap_pattern`（quadrant）
  ③ 引 `demo_fxmap_iterate`（iterate）→ `TexExportImage` 出 `demo_fxmap.png`。

自证：`wuwor l_tex_nodes -- l_tex_nodes_selftest`（第 10 组：四出 = 四象限且 raster 顺序 / 链两级不重排 /
逐档变换确定性 / Iterate 轮数生效 / Switch 选择与缺图报错 / Tile 落位）；引擎自证第 29 组测"根 → 输出"；
宿主真跑：三张示例图 `validate ok` + `run 200` + `node_errors` 空（FX 图那条 `__root__` → `__output__`）。
⚠ 改本包后**必须 `wuwo svc reload l_mindmap_mmd_server`**（宿主启动时就 import 了那份模块，
不 reload 跑的还是老骨架/老算子 —— 实测踩到）。

## 八、已知未做 / 下一步

> 用户口径（2026-10-12）：**A 像素算子包 + 导出** → **B 2D 取景器** → **C 缩略图 + 算子库示例图** →
> **D 增量烹饪 + 缓存** 四块完成；随后**更硬的算子 + 多通道材质**（§7.3）、**子图 / FX-Map**（§7.4）、
> **材质贴到 3D 预览**（§6.3）也已完成。下面只剩"锦上添花"。

- 子图 / 图实例：**引擎（`l_node_engine.subgraph`） + 宿主接线（`l_mindmap_mmd`：图名内联 / 参数端口 /
  图名下拉 / 「↗ 子图」跳转） + `TexFxMap` 三块都已做**（2026-10-12），用法见 §7.4；
  设计上定死的几条、逐条判据与踩过的坑见 `rez-package-source/Rez-Docs/Rez_pkg/子图与FX-Map_计划.md`。
- **3D 预览还差的**（§6.3 已做的那半之外的）：没有 **IBL / 环境光**（金属度高时看着会发死）；
  `height` 不做位移（要真起伏得先换细分够密的模型）；平铺只有**一个全局数值**（没有 per-channel UV、
  没有旋转 UV）；材质**只认当前图最近一次运行**那份（不能像 SD 那样把材质存成 `.mtl`/`.gltf` 带出去，
  现在能带出去的只有 6 张 PNG）；模型与材质没有"配对"数据结构（换图得自己在下拉里选）。
- **增量烹饪缓存（值级）仍然只在内存里**（§6.4 落盘的是**渲染结果**，两件事）：进程重启后
  **点「运行」还是会全量重算一遍**（这次重算出的结果又会覆盖磁盘上那份渲染缓存）。
  要"重跑也秒回"得把 `_RUN_CACHE` 里那些数组也落盘 —— 一个 1024² RGBA float32 就 16MB，
  先不做（真要做就上压缩 / 只留"贵"的中间结果）。
  ⚠ 2026-10-14 补了**另一半**：`_renders/<图>/values.json` 会落一份**逐节点的值 / 内联预览 / 耗时**
  （`server.py::_persist_values`，跑完就写；开图时前端读回来填进画布，面板标题写「上次运行值」）。
  那份是**只给看**的投影（`_json_safe`，复杂对象退化成 `{"__py_type__, __repr__}` 并标 `exact:false`）
  —— **不是**上面那个"跳过重算"的缓存，别拿它当原值用。
- `TexBlend` 的 per-channel 混合模式（现在三个通道用同一个模式）。
- 取景器的"两个节点对比"（A/B 分屏）、节点卡片缩略图的分辨率/显隐偏好。
- `TexFloodFill` 是"只在实心像素上跑并查集"：满屏实心的 1024² 会慢到秒级（它的用法本来就是
  "给遮罩里的岛上色"）；真要常态处理满屏，得换连通域标记的 scipy 路子 —— 但那要引重依赖，先不。
- 依赖**只到 numpy + Pillow**（模糊用累加式、形态学用平移取极值、缩放用自己写的双线性）——
  想加"更快的卷积"再考虑，别顺手拖个重依赖进来。
  ⚠ 2026-10-16 起多了一个**可选**的 `numba`（只给逐像素那几个热循环加速，装不上就回落 numpy；
  见 §十）。它靠 wuwo 自动装配，**不是**手写 `PYTHONPATH`。

## 九、浏览器侧 WebGPU 执行（2026-10-15）

**为什么有这一层**：动画（`Time` 驱动）时 CPU 那条路每帧都要"跑图 → 渲染 PNG → 前端取图 → 解码"，
2048² 一轮几十毫秒；WebGPU 那条把**整链留在显存**，2D 视图直接画那张纹理，不落 PNG、不过网络。
它**不改 CPU 那条**：只在"这段图里的节点全部已迁移"且浏览器支持 WebGPU 时才走（`canRunGpu()`），
否则整段落回 CPU（与今天之前完全一样）。**单个 `Time` 节点是共用的一份**（`l_node_engine/time_nodes.py`），
不存在 CPU/GPU 两版；GPU 侧只是用同一份 `{time, frame, phase, fps}` 语义填它的四个输出。

| 层 | 位置 | 职责 |
|---|---|---|
| 执行 + 会话 | `l_mindmap_mmd/web/src/texGpu.js` | 拓扑求值、WGSL 生成、纹理/缓冲区复用、2D 直显 |
| 接线 | `.../PyGraphEditor.jsx` 的动画循环 | 每拍先试 GPU、失败一次就整张图回退 CPU（并说一句原因） |
| 取景器 | `.../TexViewer.jsx` | 有 GPU 输出就画 GPU（`<canvas data-backend="webgpu">`），否则走老取图路 |
| 自证 | `l_mindmap_mmd/999.0/tools/web_probe.cjs webgpu` | 跨帧变化 / 不发 CPU 跑图 / 稳态零资源创建 / **逐算子与 CPU 对拍**（`set MM_DUMP=1` 会把差异像素的**下标与两侧数值**打出来，定位偏差用） |

**已迁移（`GPU_MIGRATION.ready`，37 个纹理/FX 算子 + 12 个引擎节点 + `Time`）**：`TexNoise`（value/perlin/worley/fbm）、
`TexGradient`、`TexShape`、`TexLevels`、`TexInvert`、`TexThreshold`、`TexNormalFromHeight`、
`TexCurvatureFromHeight`、`TexBlend`（13 模式 + mask）、`TexGradientMap`、`TexPixelProcessor`（表达式编译成 WGSL）、
`TexHSL`、`TexCurve`（LUT 在 JS 里按同一份 Fritsch–Carlson 算）、`TexQuantize`（3 种 dither）、`TexEdgeDetect`（4 模式）、
`TexChannelMerge`、**`TexChannelSplit`（三输出）**、`TexResize`、`TexCrop`、`TexTransform`（4 种 tiling）、
`TexBlur`（box=gaussian=3 次方框/radial）、`TexSharpen`、`TexMorphology`（diamond 邻域）、`TexSeamless`（无缝纹理）、`TexDirectionalBlur`、
`TexWarp`、**`TexMaterial` / `TexMaterialChannel`（直通，不派发）**、
**FX 全家（`FxSwitch` / `FxTile` / `FxQuadrant` / `FxIterate`）+ 入口 `TexFxMap`**。
**另外还迁了引擎那批"标量节点"**（不碰纹理、不派发，直接算值）：`Input`（入参，按 `type` 转标量）、
`Calc`（7 种运算符 + `digits`）、`Compare`（8 种比较，两边能当数字就按数字）、`Not` / `And` / `Or`、
`Select`（多路：c1→c2→c3 第一个命中的赢）、`Reroute`（透传）、`Output` / `Preview`（原样透传 ——
**子图里必有 `Output`**，不认它"含子图的图"就永远进不了 GPU）、**`2d_view`（2026-10-16 补：它也是原样透传，
入口名是 `image` 不是 `value`）**、**`3d_view`（2026-10-17 补：透传 `model` 字符串，GPU 不消费它但必须在场）**。
⚠ `2d_view` / `3d_view` 必须在这张表里：`canRunGpu()` 要的是"这段 prompt 里**每个**
节点都迁了"，少一个就**整段退回 CPU** —— 示例图 2026-10-16 补了 `2d_view` 之后，实时模式实测
**0.4 fps / 2697 ms 一帧**（取景器 `data-backend="cpu"`），补上它就是 **29~33 fps / work 27 ms**，
判据 `web_probe.cjs fire`（8/8）。⚠ 顺带记一笔**还没接的**：`GetParam` / `SetParam`（`Input` 退休后的
入参引用）**不在这张表里** —— 含它们的图（= 打开就迁移过的那些）目前会整段退回 CPU，要做 GPU 实时得
把"图级 `params` 的值"也喂进 GPU 那条腿（`texGpu.js` 的 args 构造处），是独立一件事。
**子图（`SubGraph`，2026-10-15）**：`graph` 是**图名**、只有宿主认（宿主跑图前换成 `sub` 内联 prompt）。
GPU 那条不经宿主，所以**编辑器先自己内联**（`PyGraphEditor.inlineSubgraphs`：`getGraph(名字)` →
`S.toPrompt` → 递归；**按图名缓存**，保存任何图时清空），再由 `texGpu` 求值：
实参按名灌内层 `Input`（`inject_params` 口径，深拷贝不串味）、跑完整张子图、`Output` 按
**名字 → id 数字**排序取值（下游 `["sub", 1]` 取第二路）、深度 8 + **按内容**判成环（同引擎）。
自证：探针 `webgpu`（手写 `sub` 的两路对拍）+ **`subgpu`**（真建子图资产 + 引用它的图 →
2D 必须是 webgpu 画布、逐帧变、**一个 CPU 跑图请求都不发**）。
它们与引擎侧 `num_nodes` / `branch_nodes` / `pygraph` **逐条对齐**（`as_number` 里 bool/None 不算数字、
`truthy` 的 `"false"/"0"/"空"` 算假、`%` 与 `//` 按 python 语义、`round` 按四舍六入五取偶）；
**比不了的一律 throw → 整拍回退 CPU**（容器/对象值的 `Compare`、`in` 的右侧不是容器…），宁可慢不算错。
⚠ 标量节点没有纹理可对拍 → 探针是**拿它的值去驱动一个纹理节点**（`TexNoise.phase`）再比像素。
**未迁移**（`GPU_MIGRATION.pending`，撞到就整段回退 CPU）：
- `TexHistogramScan`：要**精确百分位**（numpy 的顺序统计量）—— GPU 上得排序或直方图近似，**近似就过不了对拍**（0.008），所以留 CPU；
- `TexDistance` / `TexBevel` / `TexFloodFill`：`_chamfer` 是**逐行扫描**、FloodFill 是 BFS —— 都有**顺序依赖**，不是"每像素独立"；
- `TexSplatter`：要"接受的散点清单"（拒绝采样是顺序的），得先把清单放数据缓冲再逐像素遍历（**每节点数据缓冲**这块还没做）。

**FX 全家 + 入口都迁完（2026-10-15）**：`FxSwitch` / `FxTile`（四象限拼回 = unwrap）/ `FxQuadrant`（4 出 = 本层四象限，
各自尺寸）/ `FxIterate`（N 轮 + 逐轮变换）/ **`TexFxMap`（入口：`quadrant` 递归 / `iterate` 轮回喂）**。
  ⚠ FX 这块踩过的（改之前先读）：① `FxQuadrant` 要"**每个输出各自的尺寸**"——资源按 **节点+尺寸**
  **分桶**（`resourcePool` 键是 `id@WxH`）：同一帧里同一节点在不同尺寸下跑是常态，
  只留一份就会把前面几趟还在引用的缓冲销毁掉（WebGPU 报 `Buffer used in submit while destroyed`）；
  ② 涉及"先 resize 再变换"的地方**必须两趟**（两次双线性 ≠ 一次复合坐标采样，实测差 1~15 个 LSB）；
  ③ 角度是 **`-radians(rotate)`**（`ops.transform` 里那个负号）；④ `FxIterate`/`TexFxMap(iterate)` 的
  **逐轮变换并进"叠回"那一趟**；⑤ scratch 池要 **3 张**（两张给 prev 乒乓、一张给"这一轮的图案"；
  `TexFxMap(quadrant)` 的画布乒乓**读和写不能是同一张**）；
  ⑥ `TexFxMap` 的嵌套跑图**必须排进计划**（`kind:"nested"` 那种条目、按计划顺序执行）——
  在拼计划时先跑，iterate 的第 N 轮就会读到还没写的第 N-1 轮；子图的 `image` 变量传的是**盒子**
  （`.texture`/`.width`/`.channels` 都是**取值器**，派发到哪一步就指到哪张）；
  ⑦ **每次嵌套跑图前后各 `flushEncoder()`**：命令记在 encoder 里、最后才 submit，而 `writeBuffer`
  是**立刻**发到队列的 —— 同一帧里"同一份资源被复用多遍"（`TexFxMap` 每块/每轮跑同一张子图）时，
  所有轮的 uniform 都挤在 submit 之前，**只有最后一次生效**（实测：16 块内容一模一样、量程还被压扁；
  常量图案之所以看不出来，是因为它每轮参数本来就相同）。
- 出口那几个：`TexPreview` / `TexExportImage` / `TexExportMaterial` / `TexInfo` / `TexLoadImage` / `TexSbsar`
  —— **要读写宿主的盘**，浏览器里做不到（想在动画里"顺带导出"也不该做：那是每秒写 6 张 PNG）。
⚠ 因此**含出口节点的图仍然整段走 CPU**；不过 `实时关`那一档只跑 `Time` 直连节点 + 上游，
所以"一边动、一边导出"的图（如竹编）照样能让**动的那一段**走 GPU。
**多输出**（`outCount`，2026-10-15）：一个节点可以有**多张**输出纹理（`resource.textures[]` + `views[]`）——
`TexChannelSplit` 就是三趟、每趟写自己的输出（`plan[i].dst = 输出下标`）；接出去的 `["节点", k]` 取第 k 张。
`FxQuadrant`（4 出）与 `TexMaterial`（N 通道）以后共用这套，不用再改一遍。
注意：`/render`（CPU 那档）只给**第一路**，所以对拍另两路要"接个算子再用它对拍"（探针就是这么干的）。
**多趟（`plan`）**：`TexBlur`/`TexSharpen` 这类要"上一趟的结果"的算子，一趟一个 `{body, params, src}`
（`src` = 具体纹理 / `"in"` 本节点输入 / `"prev"` 上一趟 / `null` 空纹理）——两张 scratch **乒乓**，
最后一趟写回本节点纹理；每趟**自己的 uniform 缓冲** + **自己的 bind group 缓存**（稳态零创建）。

四条口径（改之前先读）：
1. **只认已迁移节点**：`canRunGpu()` 逐节点查表，**一个不认识就整段不走 GPU** —— 半 GPU 半 CPU 的图
   不做（值没法跨过去）；`engine=远端` / 带 `flags`（bypass 等）的节点同样不走。
2. **值域与方向必须与 numpy 那版逐像素一致**：判据是探针里的**对拍**（真跑 CPU 图 → 取 PNG → 比像素），
   `max < 0.008`（8bit 量化 + float32 差异留的余量）；**加新算子必须先让它过对拍**，别只看"看着像"。
   ⚠ 探针要把 **真实 `object_info` 传进 `runGpu`**：不传的话 `points`/`tiling` 这类**字符串**参数没有兜底默认值
   （踩过：曲线对拍拿"没开 LUT"那档去比，看着像算子错了）。对拍判据是 **`max < 0.008`**，
   **或**"`badFrac < 3%` 且 `mean < 0.02`"——后一条只给"采样点落在图外就换一支"的那类
   （`tiling=black`：坐标/噪波场差 1 ULP，贴着边界的一条窄带整支翻转，`TexTransform` 10 个像素、
   `TexWarp` 66 个）。**这一档别扩大解释**：真出错时（忘写纹理、通道串了）这两个数都远不止这么大
   （踩过：`outline` badFrac 80%、方框模糊 badFrac 60%）。数字照旧全打出来，不藏。
   ⚠ 几个已经踩到的对齐点：灰度 = **Rec.709**；`np.gradient` 的边界是**单边差分**（不是复制边缘）；
   `%`/`**` 要按 python 语义展开（WGSL 没有 `%`，`pow` 负底也不等价）；
   `TexPixelProcessor` 的 `x/y` 是 `i/(n-1)`（与 `np.linspace` 同口径）；
   `np.round` 是**四舍六入五取偶**（WGSL 用 `roundEven`，自己写的取整要按这个来）；
   `TexTransform` 的 cos/sin 要**在 JS 里按 float64 算好再当 f32 传进去**（numpy 是 f32×python float → f32）；
   ⚠ **写数组索引时注意 numpy 的 `delta[-1]` 不等于 `delta[n-1]`**（踩过：曲线 LUT 的最后一个斜率写成
   `delta[n-1]` → 越界 NaN → `t ≥ 0.753` 整段坏掉，对拍 max 0.99；`m[n-1] = delta[n-2]` 才对）；
   ⚠ **着色器里别用 `return` 提前跳过 `textureStore`**（踩过：`TexEdgeDetect(outline)` 这么写 → 目标纹理留着
   上一轮内容 → 几乎整图错，`badFrac` 80%）。
   ⚠ **写之前一律 `clamp(color,0,1)`**：CPU 每张 `TexImage` 构造时都 `np.clip(0,1)`（踩过：锐化的负下冲
   GPU 写出 -0.06、CPU 是 0，对拍 max 0.1）。
   ⚠ **多趟别共用一个 uniform 缓冲**：命令全记在一个 encoder 里、最后才 `submit`，而 `writeBuffer` 是立刻
   发到队列的 → 所有趟都会读到**最后一趟**的参数（踩过：方框模糊两趟都按"竖着"跑，对拍偏 60%）。
   每趟一个缓冲（尺寸很小，代价可忽略）。
   ⚠ **方框模糊的半径别套高斯那套**：`box` 直接用 `int(round(radius))`，只有 `gaussian` 才用它 ÷1.5 的
   **等效半径**（`k = max(1, int(round(r/1.5)))`，三次方框）；两者混了就是"糊得不一样"。
   取整一律按 numpy 的**四舍六入五取偶**（JS 侧有自己的 `jsRoundEven`，别用 `Math.round`）。
3. **纹理与缓冲区一律复用**（按节点 id 存一份，尺寸变了才重建；绑定组按"管线 + 输入纹理"缓存）——
   稳态必须**零** `createTexture/createBuffer/createBindGroup`（探针有这条断言）。实测 2048² 三层 fbm：
   26.7 ms（每帧重建）→ 3–6 ms。这是**计算耗时**，不是画面 FPS；画面 FPS 还受**节流 + 跳拍**限制
   （上一拍没回来就跳过，见动画循环三条口径）。
4. **`fps` 输出两档同口径**：都是"最近 2 秒窗口平均"（`Time.fps`），不是"上一拍间隔取倒数"——
   否则节流 + 跳拍会让它 24/12/8 乱跳（用户 2026-10-15 报障过）。GPU 侧在 `runGpu()` 里自己算这份。

### 9.1 MG（2D 运动图形）也上 GPU（2026-10-17）

MG 那族（`Mg*`）原来是**纯宿主**：场景在 Python 里建、每帧一次 `/run` 往返。现在整条链在浏览器里：

| 层 | 文件 | 职责 |
|---|---|---|
| 场景求值 | `web/src/mgScene.js` | `motion.py` + `motion_nodes.py` 的 JS 版（形状/渐变/变换阵列/缓动关键帧/振荡错峰抖动/沿路径；`MgTimeline` 吃与 `Time` 同一份时钟） |
| 光栅成纹理 | `web/src/mgTex.js` | `sceneOps` → `packOps` → 离屏渲染（mgGpu 那条管线）→ 一趟 compute 精确转 **`rgba32float`**（与其它纹理节点同格式） |
| 接线 | `texGpu.registerMotionGraphics()` | `Mg*` 进 GPU 白名单、`MgPreview` 当 sink、`MgTimeline` 当时钟源；`MgPreview` 每帧的纹理挂进 `session.outputs` 给取景器 |
| 隐式转换 | 宿主 `graph.register_coercer` / GPU `raster.toValue` | **`SCENE → IMAGE`**：图形链直接接纹理算子时，宿主用 Pillow、GPU 用上面那条光栅，两条腿同一语义（编辑器 `typeCompatible` 已放开） |

⚠ **`texGpu.js` 里不许出现相对 import**：探针（`web_probe.cjs`）把它的源码塞进 `blob:` URL 去 `import()`，
blob 解析不了相对路径 —— 所以 MG 走**注册**（`PyGraphEditor.jsx` 导入后调一次 `registerMotionGraphics`）。

判据 / 自证：
- `cd web && node --test`：`mgScene.test.mjs`（与 `motion.py` **逐样例对拍**，夹具由 Python 侧生成）
  + `mgGpuPlan.test.mjs`（`demo_mg_mandala` 必须判成**纯 GPU**、`Note` 这类没接线的节点不拖后腿、
  浏览器求值跑通且换时钟会变）；
- `node tools/web_probe.cjs mggpu`（5/5）：取景器是 `canvas[data-backend="webgpu"]`、逐帧变、
  **一个宿主 `/run` 都不发**（"零往返"的哨兵）。

口径与边界：
- 动画那一拍：图里有 `Mg*` 且浏览器有 WebGPU ⇒ **直接跑整张图**（不是只跑时间子树）；GPU 那条腿
  暂时不可用时**不硬退 CPU**（整图逐帧跑宿主不现实），提示开「⚡ 实时」。
- `MgPreview.supersample` 在 GPU 腿**不参与**（渲染管线走解析式边缘羽化，见 `mgGpu.js` 顶部三条差异）。
- 卡片动画（🖼）对 MG 不逐帧换缩略图：只有取景器 / 纹理消费方拿到每帧的纹理。
- **矢量导出**：`MgExportSVG` → `motion.to_svg()`，写 `.svg` 进媒体目录（或给定绝对路径），返回路径 +
  `ui.image`（媒体口按后缀发 `image/svg+xml`，卡片能直接显示）。与 `rasterize()` **同源**：同一份
  `_outline_points` / 矩阵叠法 / 渐变参照系（形状包围盒，`gradientUnits="userSpaceOnUse"`）/
  整数描边宽 / opacity 连乘；`invert` 裁剪用 `<mask>`（`<clipPath>` 表达不了反选）。
  ⚠ 浏览器那条腿**不落盘**（动画 25fps 不该每秒写 25 个文件）：`mgScene.js` 里它只回一个空路径占位，
  真要导出就点「▶ 运行」走宿主。
- **节点名称一律 ASCII**（用户口径 2026-10-17：「所有节点名称中不能有中文」）：MG 显示名是
  `Rect` / `Ellipse` / … / `Timeline` / `2D Preview`；中文只放在 `DESCRIPTION` / 搜索别名 / 入参 tooltip
  （译名表 `node_labels.json` 是用户自选覆盖，不算包自带名）。自证见两个 selftest 的 ASCII 断言。

## 十、CPU 侧 numba 加速器（2026-10-16，可选）

**为什么有**：GPU 那层（§九）只在浏览器里、且**含出口节点的图整段走 CPU**；`Time` 驱动 + 时间线
那两档动画一旦回退 CPU，瓶颈就全在"逐像素/逐采样标量循环"上（实测 1024²：worley 782ms、
fbm 5 层 1316ms、`warp` 95ms；`flood_fill` 512² 4.3s）。这些算子在 numpy 里只能靠一堆临时数组
模拟，1M 像素全是内存流量。

**做法**：`accelerate.py` —— 把**逐像素那部分**写成一个 `@njit(cache=True, parallel=True,
fastmath=False)` 核（`prange` 分行、一次算完、不留中间数组），**装不上 numba 就返回 `None`、
调用方回落 `ops.py` 那份 numpy**（两条路都要能跑，没 numba 时行为与今天之前完全一样）。

**已接线**：`ops.noise`（value / perlin / worley / fbm）、`ops.warp` 里那层非周期 perlin
（`perlin_field`）、**`ops.directional_blur` / `ops.radial_blur`**（**`tiling="clamp"` 与 `"repeat"` 都接管**
—— 2026-10-16 补的 repeat 那版；其余平铺照旧走 numpy，别偷偷换语义）、**`ops.flood_fill`**（整数语义 ⇒ 与 numpy
**逐位相等**）、**`ops.resize` / `ops.transform` / `ops.warp` 的双线性重采样**（四种平铺都接管）。
⚠ 逐点采样**只留一份实现**（`_sample_px`，四种平铺）：blur 核早先自己抄过一份 clamp 的写法
（`_blur_px_clamp`），已在补 repeat 时删掉 —— 两份迟早漂。
实测（1024² / 512²，本机 32 线程，热身后取三次最好）：

| 算子 | numpy | numba | 加速 |
|---|---|---|---|
| value 噪波 1024² | 157 ms | 3.3 ms | ×47 |
| perlin 噪波 1024² | 240 ms | 6.9 ms | ×35 |
| worley 噪波 1024² | 759 ms | 8.8 ms | ×86 |
| fbm（5 层）1024² | 1189 ms | 23 ms | ×53 |
| **方向模糊 1024²（n=16，RGB）** | **3223 ms** | **27 ms** | **×119** |
| **径向模糊 1024²（n=12，RGB）** | **2735 ms** | **34 ms** | **×80** |
| **连通域填色 512² 满屏** | **4716 ms** | **6 ms** | **×786** |
| 连通域填色 512² 斑点 | 3090 ms | 8 ms | ×386 |
| **`resize` 512²→256²（repeat）** | 17.4 ms | **0.5 ms** | **×35** |
| `resize` 256²→1024²（clamp） | 183 ms | 7.1 ms | ×26 |
| **`transform` 512²（旋转 37°）** | 65.7 ms | **3.1 ms** | **×21** |
| **`warp` 512²** | 95.8 ms | **4.2 ms** | **×23** |
| `warp` 1024² 的噪声场那半 | 95 ms | 92 ms | ×1.0（瓶颈在采样，不在场） |

两个模糊为什么这么赚：numpy 那份**每采一次就把整张图算一遍**（1024²×16 采样 ≈ 200MB 临时数组，
纯内存带宽）；核里把 n 次采样累加在寄存器里，不落内存。`flood_fill` 则是 numpy 版用 **dict** 做
并查集（每像素几次 Python 层查找）；核里换成 int32 数组 + 三条串行扫描。
重采样那三个同理：numpy 先把**整张坐标网格**算成 float32 数组、再 4 次 gather，核里坐标现算、
不留网格。
⚠ **repeat 那版是"省时间"这个诉求逼出来的**（2026-10-16 用户报 `lava_flow` 慢）：示例图按"平铺纹理
一律 `repeat`"的规矩接线，于是那条"只接管 clamp"的边界把**整张图**送回 numpy —— `lava_flow` 2048² 的
`db_flow`（`TexDirectionalBlur` samples 16 / length 40 / repeat）**单节点 1637 ms**，占该图 CPU 一轮
（2359 ms）的七成；补上后 **45 ms**，整图 **801 ms**。自证里两个模糊的 `clamp` / `repeat` × 三种尺寸 ×
四个角度（含 `angle=90` 那条 float32 进位的老坑）全部逐像素对拍。
⚠ `_flood_kernel` **故意不开 `parallel`**：编号与发号都必须按**行主序**（顺序即语义，并行就串号）。
⚠ 重采样的**四种平铺语义逐条照抄**（`repeat` 是"折回 + 索引接回 0"、`black` 是"不折回 + 事后清 0"）——
抄错一个就是"平铺出缝 / 边缘拉丝"，而那两种错**在缩小图上量不出来**（差的是整条边），
所以自证里四种平铺 × 三种尺寸都要比。
⚠ **没接的**：`box/gaussian` 模糊（numpy 那份是**前缀和** `cumsum` 差值算的，换滚动窗口求和
等于换了一套浮点累加序 ⇒ 与 numpy 对不上，而"数值就是规格" ⇒ 不动）、`TexPixelProcessor`
（表达式按 numpy 向量化 ✓ 与核一样是带宽限制，接 numba 没收益）。

**判据 = 与 numpy 逐像素可比**（`selftest._check_accelerate()`）：
- 噪波：5 组尺寸/相位/周期开关 × 4 种 + `perlin_field` + 端到端 `ops.noise` —— **`max|Δ| ≤ 1e-6`**，
  实测最差 **1.19e-07**；
- 两个模糊：3 组尺寸/通道 × 3 档参数（含负角度、1 像素长度、2 采样）—— 同上容差，实测最差 **1.79e-07**；
- `flood_fill`：5 种形状（斑点/满屏/全空/细条纹/对角线）× 4 档（random/index/未知档/换 seed）——
  **`np.array_equal` 逐位相等**（整数语义，不能拿 1e-6 糊过去：真错了就是"岛号串了"）。
- 参考实现**不另抄一份**：测试里临时把 `accelerate.HAVE` 置 False，让 `ops` 那几个入口走真回落路
  （抄一份参考实现 = 两份实现各自漂移、测试比不出来）。
⚠ **不是位级一致**，别在注释/文档里那么写：numpy 的元素级循环在 x86 上会被 SIMD 编成 **FMA 融合**
（`a*b+c` 少一次舍入），njit（`fastmath=False`）严格分步 ⇒ 常见 1~2 个 ULP 的差。这个量级比 GPU
对拍阈值（0.008）小 **5 个数量级**，所以 §九 的 CPU/GPU 对拍不受影响（不必重跑）。

**三个"看着像优化其实更糟"的旋钮（都量过，别再试）**：
- 给 8 个 helper 加 `inline="always"` → 冷编译 **3.2s → 13.5s**，运行时不快 ⇒ 不加；
- `parallel=False` → 冷编译 2.9s（省 0.3s），但热起来慢 15 倍（fbm 1024² 23ms → 365ms）⇒ 别关；
- `fastmath=True` → 与 numpy 的差 1.19e-07 → 1.8e-07，fbm 只快 ~15% ⇒ 不值，别开。

**编译与预热（这块最容易被"第一次点很慢"咬到）**：
- 冷编译：只噪声那批 ~3.2s、连两个模糊 ~5.3s、**全部（含重采样）~7.5s**；numba 有磁盘缓存（落在本包
  `src/l_tex_nodes/__pycache__/*.nbi|nbc`），缓存命中后**每进程首调 ~0.26s**（实测 1024² fbm）。
  改了本包源码 → 缓存失效 → 又要编一次。
- `nodes.py` **导入即调 `accelerate.warmup()`**（守护线程后台编）：宿主 `:8110` 是长驻进程，
  这样"人点得慢"时基本已经热了。⚠ 但**紧接着导入就调用**仍会等编译锁（实测最差 14.5s）——
  所以开发/换环境后主动跑一次 `wuwor l_tex_nodes -- l_tex_nodes_warmup`（同步编完落缓存）。

**依赖是 wuwo 自动装配的，这里踩过一个装配层的坑**（已修，见 `wuwo/doc/CHANGELOG.md`）：
`requires` 写 `numba` 之后解析直接
`PackageNotFoundError: Package could not be found: llvmlite-0.50.0.dev0+<0.51` ——
numba 每个版本的 pip 元数据都把 llvmlite 钉成 `>=0.X.0.dev0,<0.X+1`，而 **rez 的版本序不是 PEP440**：
多出来的 `dev0` 让它排在 `0.50.0` **之后**，货架上明明装着 `0.50.0` 也解不出（换 numba 版本无用）。
修法在装配层：`auto_fetch_packages._normalize_prerelease_requirements` 把生成的 `requires` 里
`devN`/`aN`/`bN`/`rcN` 后缀去掉（PEP440 语义上 `>=0.50.0.dev0` 本来就接受 `0.50.0`），
新装与复用两条路都会治；货架上那份坏产物删掉重装一次即好。

**还没做（各条都要自带"与 numpy 逐像素可比"的自证）**：`distance` / `bevel` 的 `_chamfer`
（逐行扫描、有顺序依赖，得按 `_flood_kernel` 那种"显式状态 + 串行扫描"写）；`splatter`
（拒绝采样是顺序的，得把"接受的散点清单"先放进缓冲）；再往后才是融合 `hsl` / `seamless`
的临时数组。
⚠ **已经向量化、受内存带宽限制的那批别套 njit**：`levels` / `gradient` / `quantize` /
`pixel_processor`（表达式按 numpy 向量化，与核一样是带宽限制）；
`box`/`gaussian` 模糊**更不能换** —— numpy 那份是**前缀和差值**算的，换滚动窗口求和等于换浮点累加序
⇒ 与 numpy 对不上，而"数值就是规格"。

