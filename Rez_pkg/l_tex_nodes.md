# `l_tex_nodes` —— 程序化贴图算子（Substance Designer 那类）

> 更新：2026-10-12。**先读本文再改算子** —— 每个算子的语义（值域、方向、单位）都在这里定死，
> 改之前也要看 §5「踩过的坑」。
>
> 上游是 **`l_node_engine`**（节点图内核，纯标准库），下游是 **`l_mindmap_mmd`** 的
> `/pygraph/<图名>` 编辑器（画布上多出 38 个 `Tex*` 节点）。**引擎里没有一行图像代码**：
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

**确定性**：噪波一律由 `seed` 决定的整数哈希生成（不用全局 `np.random`）—— 同一参数两次跑**逐像素一致**
（不然实时重算会自己抖，也没法写测试）。`TexNoise` / `TexWarp` 都有 `seed`。

## 三、节点清单（38 个，画布分类 `l_tex_nodes`）

| 分类 | 节点 | 要点 |
|---|---|---|
生成 | `TexGradient` | `linear / radial / angular / diamond`；`angle` 单位度 |
 | `TexNoise` | `value / perlin / worley / fbm`；`scale` 越大越粗；`octaves` 只对 fbm 有意义；`seed` |
 | `TexShape` | `rect / disc / ellipse` 遮罩；`size` 占短边比例、`hardness` 边缘软化；**`disc` 永远是正圆**（忽略 `ratio`） |
滤镜 | `TexBlur` | `gaussian`（**三次方框近似**，不用 scipy）/ `box` / `radial`（绕图心角度模糊，单位度） |
 | `TexDirectionalBlur` | 方向（拖尾）模糊：`angle` 度、`length` 像素、`samples` 档数（直线平均，保能量） |
 | `TexEdgeDetect` | `sobel`（梯度模，默认）/ `laplacian`（细二阶）/ `emboss`（浮雕，平地 = **0.5** 灰）/ `outline`（平地 = 白） |
 | `TexSharpen` | 反锐化掩模：`原图 + amount × (原图 − 模糊)` |
 | `TexLevels` | 输入黑白场 → `gamma` → 输出黑白场（**0..1 口径**，UI 上常说 0..255） |
 | `TexHistogramScan` | 按画面**自己的亮度百分位**归一（`position` = 新中点、`contrast` = 窗口宽度）；按亮度算一刀、三通道共用 → **色相不动** |
 | `TexCurve` | `points = "0,0; 0.5,0.6; 1,1"`；**Fritsch–Carlson 单调插值**（普通三次样条会过冲出脏边） |
 | `TexHSL` | `hue` 是**偏移量**（度）、`saturation`/`lightness` 是**乘数** |
 | `TexInvert` / `TexThreshold` | 阈值 `softness>0` 为 smoothstep 软过渡 |
 | `TexGradientMap` | 灰度 → 彩色（Substance 的 Gradient Map）：`points = "0:#000; 0.5:#ff8800; 1:#fff"`；`alpha_from_gray` |
 | `TexQuantize` | 压到 `levels` 档；`dither = none / ordered`（4×4 Bayer）/ `noise`（确定性噪点，抖动**不多出档**） |
 | `TexMorphology` | 膨胀 / 腐蚀（灰度也适用）；半径上限 16（迭代实现，很大就慢） |
 | `TexPixelProcessor` | **逐像素表达式**（整张图当数组一起算）：`a/b/c` 输入图、`x/y` 是 0..1 坐标；可用 `clip/min/max/abs/sqrt/hypot/where/sign/floor/round/sin/cos/exp/log/tanh` 与 `pi`。例 `clip(hypot(x-0.5, y-0.5) + a*0.35, 0, 1)`；**不是**任意代码入口（白名单函数 + 空 `__builtins__`） |
混合/通道 | `TexBlend` | **13 种模式**（normal/multiply/screen/overlay/soft_light/hard_light/add/subtract/difference/darken/lighten/min/max/divide）+ `opacity` + `mask`；`bottom`=底、`top`=顶 |
 | `TexChannelSplit` / `TexChannelMerge` | 拆 R/G/B（1 通道）与合回 RGB；尺寸以 `bottom`/`r` 为准，其余自动缩放 |
几何 | `TexTransform` | `translate_*` 是**图宽百分比**（0.5 = 挪半张）；`rotate` 度；**`scale` >1 = 放大内容**、<1 = 缩小（露出边缘，靠 `tiling` 决定露什么）；`tiling = clamp/repeat/mirror/black` |
 | `TexWarp` | 按一层噪波沿 `angle` 推挤（`amount` = 图宽百分比） |
 | `TexResize` / `TexCrop` | 改尺寸 / 裁剪（越界用边缘像素补，**尺寸恒等于请求值**） |
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


## 四、怎么用 / 怎么起

```cmd
wuwor l_tex_nodes  -- l_tex_nodes_selftest      :: 自证（11 组，逐像素）
wuwor l_tex_nodes  -- python -c "from l_tex_nodes import ops; print(ops.noise(256,256,'fbm',48,5,7))"
```

- **画布上**：`http://<入口>/mindmap-mmd/pygraph/demo_tex_basics`（现成示例图，见下），
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
  滤镜类统一喂**标准素材** `SOURCE`（perlin 噪波）—— 拿一片纯色去看"模糊/色阶/法线"是看不出差别的，
  这是这张表里唯一值得讲究的地方。
- `TexLoadImage` / `TexExportImage` / `TexInfo` / `TexMaterial` / `TexExportMaterial` **故意没有示例**：
  第一个要磁盘文件，后三个的产物分别是**文件路径 / 材质字典 / 路径列表**（**都不是图**），那一格没图可放。
  调色板里它们只显示名字（前端 `onError` 塌成空，不留破图标）。
  （`TexMaterialChannel` **有**示例 —— 它那张图里先搭了一个 `TexMaterial`，顺便把材质打包也跑了一遍。）
- 门槛：示例图要跑节点，所以和跑图同一个开关（`L_MINDMAP_ALLOW_RUN`）—— 关着执行面的实例
  （公网那张卡）调色板只显示文字，不影响画布。

### 6.3 3D 预览贴 PBR（已做，2026-10-12）——「竹编」示例

入口两个，**同一个 `MeshViewer`**（`docked` 属性区分浮层/停靠 —— 光照、贴图、取景只此一份）：
① `Input(type=model)` 节点上的 **👁**（浮层）；② 底栏标签 **「🧊 3D 视图」**（常驻，不必去点节点：
模型取图里**第一个** `Input(model)` 的值，材质同样自动选第一个 `MATERIAL` 节点；切到这个标签时
底栏会自动加高到 340px，否则模型就一个小点）。
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
（节点数 37 → **38**、示例图 32 → **33** 张）。

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
- `TexBlend` 的 per-channel 混合模式（现在三个通道用同一个模式）。
- 取景器的"两个节点对比"（A/B 分屏）、节点卡片缩略图的分辨率/显隐偏好。
- `TexFloodFill` 是"只在实心像素上跑并查集"：满屏实心的 1024² 会慢到秒级（它的用法本来就是
  "给遮罩里的岛上色"）；真要常态处理满屏，得换连通域标记的 scipy 路子 —— 但那要引重依赖，先不。
- 依赖**只到 numpy + Pillow**：不引 scipy / opencv（模糊用累加式、形态学用平移取极值、缩放用自己写的
  双线性）—— 想加"更快的卷积"再考虑，别顺手拖个重依赖进来。
