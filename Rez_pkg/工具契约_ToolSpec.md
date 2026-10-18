# 工具契约（ToolSpec）v1

> 2026-10-13。**这是 S0（定契约）的产物**：只定**字段 / 词表 / 落点 / UI 同源口径 / 判据**，实现分 S1–S4（见 §8）。
> 用户口径（原话）：「我希望工具可以是 python 类代码，也可以是节点类网络，工具可以在 pygraph 中显示并配有 ui，
> l_agent 的工具 ui、pygraph 中的工具节点 ui 完全一致；一套 pygraph 的 agent 跑完就相当于是完成一次对话，
> 其中**思考内容组件也是一个工具**，只是比较特殊。」

## 0. 一句话

**工具 = 一份 `ToolSpec`**：两种来源（**节点图** / **python 类**）产同一种 spec，四处消费
（画布节点 UI · 聊天工具卡 UI · agent 函数 schema · 执行器）。
**"完全一致"的判据** = 同一份 spec 喂同一套渲染器（容器 → 控件）出同一种卡。

## 1. 现状：为什么现在做不到

| # | 现状 | 证据 |
|---|---|---|
| 1 | **工具资产零落地**：PUT 不认 `kind:"tool"` → 这类资产会被**当普通图存**；无列表接口、无 🧰、"落成工具节点"没写 | `l_mindmap_mmd/server.py` PUT 校验只认 `struct`/`class`/`project`；`l_mindmap_mmd.md`「还没做」条 |
| 2 | **参数 schema 三套形状**（且都不是 JSON Schema） | agent 侧扁平 `[{name,type,desc,required}]`（`l_agent_tool/agent_tools.py` `_param`）／图侧 `graph_tool_spec`（`l_node_engine/pygraph.py`）／画布 `object_info`（`l_node_engine/node_base.py` `get_node_info`） |
| 3 | **图工具的参数类型被压平**成 `number`/`string`，默认值只拼进描述文本 | `l_agent_tool/pygraph_tools.py` 的 `_parameters` |
| 4 | **聊天工具 UI 是按工具名 if-else 硬编码**，不是 schema 驱动；工具选择器只有参数**名**数组 | `l_agent_chat/web/src/new/NewApp.jsx` 的 `ToolCard`；`components/ToolPicker.jsx` |
| 5 | **文档锚点悬空**：`l_mindmap_mmd.md` 那句"见**设计要点那节**"——**该章节不存在**（本文即补上它） | grep「设计要点」只命中引用行自身 |

另有两条事实：
- `Tools/` 目录已随工程建齐（`DEFAULT_PROJECT_DIRS` 含它），但**没有任何 kind 落到该目录的逻辑**；
- python 类工具"存 `module:Class`"目前**只是一句概念**：字段名 / schema / 运行时加载都没定义。

## 2. 术语（一套词，三处不许再各叫各的）

| 词 | 定义 |
|---|---|
| **工具（tool）** | 能被 agent 调用、也能在画布上出现的一次"能力"。两种来源：**图工具** / **python 工具**；另有**内置工具**（含思考） |
| **工具资产** | 落在 `<工程>/Tools/` 里的那一个文件（图或 `kind:"tool"` JSON） |
| **ToolSpec** | 由工具资产（或内置工具）算出的**一份**规格：名字 / 说明 / 参数 / 输出 / 来源 / 属性 / 显示 |
| **容器（container）** | **显示语义**：这个参数用什么控件画（`text`/`checkbox`/`select`/…）。**唯一**控件标识，两处 UI 都按它画 |
| **端口类型（port type）** | **数据语义**：`STRING`/`INT`/… 或 `*`（只连线的对象端口） |
| **标准类型** | 参数的**权威**类型写法 = python 类型表达式（§5） |
| **思考工具** | 内置工具的一种：把"模型思考内容"当成一次可显示、可组图的调用（§7） |

## 3. 落点：`<工程>/Tools/`

用户已拍板（`l_mindmap_mmd.md`）：**节点网络工具与 python 类工具都在 `Tools/` 这一层**，`Functions/` 只放普通函数图。

| 工具来源 | 文件形态 | 参数从哪来 |
|---|---|---|
| **图工具** | `Tools/<名>.json` **就是一张普通 pygraph**（与 `Functions/` 同格式） | 照旧：`Input`/`Output` 现算（`graph_tool_spec`） |
| **python 工具** | `Tools/<名>.json` = `{"kind":"tool", "source":{"type":"python", …}, …}` | 反射（`inspect.signature` / 类方法 + 类型注解），见 §4.3 || **内置工具**（思考等） | 无文件（代码内置） | 代码写死 |

**判别唯一入口**：`tool_spec_of(addr, data)`（S1 实现）——
- `data` 有 `nodes` 且地址在 `Tools/` 下 → 图工具；
- `data.kind == "tool"` → 看 `source.type`（`python`）；
- 其他 → 不是工具。

**可选顶层 `tool` 块**（不新增资产类型）：图工具想加显示名 / 图标 / 属性时，在图上写
`{"nodes": {…}, "tool": {"display_name": "…", "flags": {…}, "ui": {…}}}`。

## 4. ToolSpec 字段（权威）

### 4.1 形状

```json
{
  "name": "glob_files",                  // 稳定标识：agent 工具名 / 节点类型名都用它派生
  "display_name": "列文件",
  "description": "按通配符列文件…",       // 给模型看的那一句
  "kind": "graph",                       // graph | python | builtin
  "addr": "<工程名>/Tools/glob_files",     // 资产地址（内置工具 = 空串）：编辑器靠它打开/定位。⚠ 首段是**真工程名**（`pro/…` 只在图内引用里成立，当入口地址 = 400）
  "source": {"type": "graph", "addr": "<工程名>/Tools/glob_files"},
  "params": [                            // ← §4.2
    {"name": "pattern", "type": "str", "description": "通配符",
     "required": true, "default": "*.py", "container": "text"}
  ],
  "outputs": [{"name": "files", "type": "list"}],
  "flags": {"read_only": true, "approval": false, "blocking": false, "deadline": 0},
  "ui": {"icon": "🧰", "group": "文件", "order": 10}
}
```

### 4.2 `params[]`（唯一参数口径）

| 字段 | 必填 | 说明 |
|---|---|---|
| `name` | ✅ | 参数名（一段，`{A-Za-z0-9_.-}`） |
| `type` | ✅ | **标准类型**（§5）；不写 = `string` |
| `description` | | 给模型看的说明（子图取 `Input.description`） |
| `required` | | 缺它 / 空值是否报错；**图工具默认全 `false`**（`Input` 有初值，报必填是假警报） |
| `default` | | **结构化**默认值（不再拼进描述文本） |
| `container` | | 控件（§5.3）；缺省由 `type` 推 |
| `values` | | COMBO 候选（`container:"select"` 时用） |
| `multiline` | | `text` ↔ `textarea` |

**存值形态**（与引擎/编辑器现有约定一致，别改）：标量存真标量；**容器存 JSON 文本**
（`Input.value` 端口是 `*`，存数组会被当成"连线"）；执行时按 `type` parse 回真对象。

### 4.3 python 工具怎么写

> **口径（用户 2026-10-13 拍板）**：python 工具**是一个类**，不是一个函数 ——
> `{"module": …, "class": …}` 是标准写法（`source.func` 保留兼容，但新工具别用）。
> 样板见 `l_node_engine/demo_tools.py`；入口按 `source.method` → `run` → `execute` → `__call__` 找。
> ⚠ 类的包必须能被节点引擎导入：**动态导入与"节点里 import"同一道闸**（`l_node_engine.guard`，
> 默认只放行引擎自身与已注册节点模块）—— 自己的包要加进 `L_NODE_ENGINE_ALLOW_PKGS`。

```json
{
  "kind": "tool",
  "name": "run_muse",
  "display_name": "跑 Muse",
  "description": "…",
  "source": {"type": "python", "module": "lugwit_tools.muse", "class": "RunMuse", "method": "run"},
  "params": [{"name": "task", "type": "str", "required": true, "container": "text"}],
  "flags": {"read_only": false, "approval": true}
}
```
- `source` 标准写法：`{"module": "pkg.mod", "class": "Cls"}`（+ 可选 `"method"`）；
  `{"module": …, "func": …}` 仅兼容旧写法。
- `params` **可以省略** → S1 用反射补（类：显式 `tool_params()` 优先，否则 `run/execute/__call__` 的签名；函数：签名 + `typing` 注解）；
- 反射不出来的注解一律回落 `string`，并**在 spec 里记 `"inferred": true`**（UI 上给个"猜的"标记，别装作精确）。

## 5. 类型词表（唯一权威表）

### 5.1 标准类型 = python 类型表达式

理由：图侧 `Input.type` **已经是**自由类型表达式（`l_node_engine/pygraph.py` 的 `_PARAM_TYPES` +
`norm_param_type` / `_PARAM_TYPE_RE`），能表达 `list[int]` / `dict[str,int]` / `np.ndarray` 这类；
再引入第二套"标准"就是给三套形状又加一套。

### 5.2 映射表（**这张表是权威，别在别处再写一份**）

| 标准 type | 端口类型（ComfyUI） | 容器（缺省） | agent/OpenAI type | 存值 |
|---|---|---|---|---|
| `str` / `string` | `STRING` | `text` | `string` | 文本 |
| `str` + `multiline` | `TEXT` | `textarea` | `string` | 文本 |
| `int` | `INT` | `number` | `integer` | 数字 |
| `float` / `number` | `FLOAT` | `number` | `number` | 数字 |
| `bool` | `BOOLEAN` | `checkbox` | `boolean` | 布尔 |
| `None` | `*` | `none` | `null` | `null` |
| `list` / `tuple` / `set` / `dict` / `object` / `list[int]`… | `*` | `json` | `string`（JSON 文本） | JSON 文本 |
| `bytes` / `bytearray` | `*` | `bytes` | `string` | utf-8 文本或 `base64:<…>` |
| `Path` | `STRING` | `text` | `string` | 路径文本 |
| `image` / `model` | `STRING` | `image` / `model` | `string` | 路径文本 |
| COMBO（`values`） | `COMBO` | `select` | `string` + `enum` | 候选之一 |

反向（端口类型 → 标准）：`STRING/TEXT→str`、`INT→int`、`FLOAT→float`、`BOOLEAN→bool`、
`COMBO→str(+values)`、**其余（`LIST`/`DICT`/`IMAGE`/`VIDEO`/`AUDIO`/`*`/自定义）→ `object`，端口类型保持 `*`**。
⚠ 现有 `_SUBGRAPH_PORT_TYPES`（`l_mindmap_mmd/server.py`）只有 9 条、其余落 `*` ——
**S1 已并进本表**：宿主只剩 `_port_type_of()` 一层转发（引擎缺包时回落 `*`），别再写第二张。
**本节的表已落在代码里**：`l_node_engine/tool_spec.py`（`_PORT_BY_TYPE` / `_CONTAINER_BY_TYPE` /
`_JSON_BY_TYPE` + `comfy_port_type()` / `container_of()` / `json_schema_type()`）——文档与代码不一致时以代码为准。

### 5.3 容器词表 = 渲染器清单（两处 UI 必须都实现）

引擎权威来源是 `l_node_engine/node_base.py` 的 `container_for`（`text`/`textarea`/`number`/`checkbox`/`select`/`link`
+ 编辑器专用 `code`/`hidden`）；编辑器另外按 `Input.type` 派生出 `image`/`model`/`json`/`bytes`/`none`
（`l_mindmap_mmd/web/src/pySchema.js` 的 `paramEditorOf`）。

⇒ **合并后的容器全集**（S1 落成一张常量表，两侧共用）：
`text` · `textarea` · `number` · `checkbox` · `select` · `link` · `code` · `hidden` · `image` · `model` · `json` · `bytes` · `none`

约定：
- **两处 UI 只按 `container` 分支**，不再 `if (toolName === "read_file")` 这种按名字硬编码；
- 容器是**显示语义**，与"数据语义"（type / 端口类型）解耦：同一个 `str` 可以 `text` 也可以 `textarea`；
- 新增容器 = 两侧同时加渲染器，**不许只加一边**（否则"完全一致"立即破功）。

### 5.4 `std` 的**流通链**（端口配色 / 形状靠它；2026-10-14 落地）

**为什么要有这一格**：按 §5.2，`list` / `tuple` / `set` / `dict` / `object` 在**端口类型上全是 `*`**
—— "这到底是数组还是元组"只有 python 类型表达式知道。编辑器拿它给端口**配色 + 画形状**：

| 形状 | 标准类型 | 色 |
|---|---|---|
| 圆点 | 标量（`int`/`float`/`str`/`bool`/`Path`/`None`…） | 黄 / 橙 / 绿 / 天蓝 |
| 三道竖条 | `list` / `list[int]` | 紫 |
| 3×3 九点 | `tuple` | 蓝紫 |
| 花括号 | `dict` | 粉 |
| 双环 | `set` / `frozenset` | 靛 |
| 虚线圈 | `bytes` / `bytearray` | 灰 |
| 方角 | `*`（未知，保底观感） | 灰 |

**唯一实现**：`l_node_engine/introspect.py` 的 **`std_of_annotation(ann) -> (标准类型, 可空)`**
（`Optional[X]` / `Union[X, None]` / `X | None` 会**剥成 `X`**；认不出回 `""` = 不知道）。
**别在别处再写一份映射表。** 前端那张"类型 → 颜色/形状"的表在
`l_mindmap_mmd/web/src/portStyle.js`（DOM 与 canvas 两个后端**共用**它）。

**⚠ 图案型端口不画边框**（2026-10-17 用户：「这个端点图标…只去边框」）：`bars` / `grid` / `brace`
这三个的图案**本身就是形状**，再套一圈彩色边框 = 10px 里框和图案叠着画（`dict` 端口那圈粉框就是这么来的）。
实现口径：`PyNode.jsx` 给 `Handle` 的内联样式**只给 `backgroundColor`（实心标量用）+ `color`（图案走
`currentColor`）**，**不要 `borderColor`** —— 形状类里对图案型写的 `border-color: transparent` 没有
`!important`，内联的 `borderColor` 会把它覆盖回去。要不要边框由形状类决定（`ring`/`dashed` 自己声明
`2px`，实心的 `dot`/`square` 吃 `.pyg-handle` 的 `1px solid var(--fx-bg)` 暗描边）。
canvas 那边 `drawPortMark` 本来就只有图案、不画框，所以这条只改 DOM。判据：`web_probe types`
（`.is-brace` / `.is-bars` 的 `border-top-color` 必须是 `rgba(0, 0, 0, 0)`，`color` 仍是类型色）。

各环节字段名与来源 —— **认不出就不发这一格**，消费端一律回落"按端口类型画"：

| 环节 | 字段 | 来源 |
|---|---|---|
| 引擎·输入端口 | `opts.std_type` | `introspect.params_from_callable()`（按**参数注解**，含函数/方法/类生成的端口） |
| 引擎·输出端口 | 节点元数据 `output_std[i]` | `callables._return_std()`（按**返回注解**）→ 生成类的 `RETURN_STD_TYPES` → `node_base.get_node_info()` |
| 宿主·函数 | `/ports?kind=function` → `outputs:[{name,port,std}]` | `callables.return_spec(func)` |
| 宿主·方法 | `/ports?kind=class&method=…` → 同上 | `callmethod.method_return_spec(target, method)` |
| 宿主·工具 | `/ports?kind=tool` → 同上 | 工具 `ToolSpec.outputs[].type`（**本身就是标准类型**，见 §4.2） |
| 宿主·子图 | `/ports?kind=subgraph` → 同上 | 顺着内层 `Output` 节点的**连线**找源头节点的输出类型；源头是 `Input` 就用它自己的 `type` |
| 宿主·**宿主步骤**（`HostStep.step`） | `/ports?kind=hoststep&target=<步>` → `outputs[].std` | 那一步**注册时声明**的 `params` / `outputs`（agent 侧 `graph_turn.STEP_META`，宿主 `/api/pygraphs/host-steps` 转一手） |
| 编辑器·端口表节点 | `info.output_std[i]` | 端口表里用户写的那个 `type`（`pySchema.withOutputPorts`） |
| 编辑器·`Input` 节点 | 输入/输出两端都用它自己的 `type` | 节点输入 `type`（静态就知道，不必等运行） |

约定：
- **两处 UI 都要实现**（同 §5.3 的纪律）：DOM 后端 = `PyNode.jsx` 的 `.pyg-handle.is-<shape>` + `style.css`
  里的 CSS 图案；canvas 后端 = `CanvasSurface.jsx` 的 `drawPortMark()`（手绘同一套图案）。
  **只加一边等于没加。**
- `output_std` / `std` 都**与端口数组同下标一一对齐**；空串 = 不知道。
- **拿不到就不猜**：`GetAttr` / `Select` / 没写注解的函数这类"输出类型由运行时决定"的端口保持 `*`
  （方角 + 灰）。要更细就得给节点加"输出类型规则"声明 —— 那是另一件事，不在本表。

## 6. UI 同源（"完全一致"怎么算成立）

| 消费方 | 现在的入口 | 改造后 |
|---|---|---|
| 画布 | `PyNode.jsx` + `PyWidgetInput.jsx`（已按 `object_info` 的 container 驱动） | 工具节点仍走这套，但**参数来自 `ToolSpec.params`** |
| 聊天 | `NewApp.jsx` 的 `ToolCard`（按工具名 if-else） | 参数区/结果区改用**同一个参数渲染器**；按名分支只允许留给"结果差异"（如 diff、截图），不留参数 |

**共享方式（唯一真解）**：两个前端（`l_mindmap_mmd/web`、`l_agent_chat/web`）**共用一份组件源码**
（新建共享前端包，两边 vite `resolve.alias` 指过去）。复制一份必然立刻漂 —— 等于没做。
共享组件的最小集：`ParamForm`（`params[]` + `container` → 控件）、`ParamRow`、`ResultView`（文本/JSON/富块）。

**共享源码的路径怎么来（2026-10-14 收紧）**：**走 rez env，不写死相对路径**。
- `l_py_ui/999.0/package.py` 的 `commands()` 里设 `env.L_PY_UI_WEB = "{root}/web/src"`
  （另有 `env.L_PY_UI_ROOT`）；
- `l_agent_chat` 与 `l_mindmap_mmd` 的 `requires` 都声明 **`"l_py_ui"`**；
- 两个 `vite.config.js` 里 `const SHARE = process.env.L_PY_UI_WEB || <相对路径>` ——
  **回退必须留**：`vite`/`node` 直接跑、或没走 `wuwor` 的 CI 里没有那个 env。
- **为什么**：把 `../../../l_py_ui/999.0/web/src` 硬写在前端源码里，等于把"两个包必须同货架 +
  版本目录名固定为 `999.0`"绑进源码 —— **任一包被单独拷走/部署 ⇒ 别名指空**（构建失败，
  或悄悄读到别的旧源码），且升版本目录还得改前端源码。走 rez 后，谁部署谁按 `requires` 解析，
  版本目录名不再进源码。

## 7. 思考内容组件 = 一个内置工具

**注册表里已经是了**：`l_node_engine.tool_spec.builtin_tool_specs()` 第一条就是它（`GET /api/pygraphs/tools`
里看得见）；契约先定形状、行为分两步落（2026-10-15 落了"两处都能显示"这半）：

```json
{
  "name": "thinking", "display_name": "思考", "kind": "builtin",
  "source": {"type": "builtin", "builtin": "thinking"},
  "params": [],
  "outputs": [{"name": "text", "type": "str"}],
  "flags": {"read_only": true, "approval": false, "blocking": false}
}
```
- 画布上 = 一个"思考"节点（无输入、一个文本输出；跑图时把 reasoning 流写进输出）—— **仍没做（S4）**；
- 聊天里 = 现有那块思考内容 UI；**画布的「🧰 工具卡」这侧 2026-10-15 补上了** ——
  数据：agent 把每段规划思考挂进 `/api/pygraph-live` 的 `reasoning`（`graph_turn.TurnContext.reasonings`，
  `{t, text, secs, round}`，空思考不记）；组件：共享件 **`l_py_ui` 的 `ThinkingCard`**（正文走
  `ResultView`，与上面那句"同一个 `ResultView` 渲染"对齐；折叠/秒数/可选「📜 提示词」）。
  实测：真跑一轮 → live 里 1 段思考（`secs 3.4`）→ 画布那张卡显示同样文字，探针 `panes` **14/14**。
- **还没同源**（下一步）：聊天那侧仍是它自己那块 `Reasoning`（`l_agent_chat/web/src/new/NewApp.jsx`，
  带翻译 / 朗读 / 流式秒表 / 与消息状态联动的自动折叠）。要真正"一份渲染"，得把这几个能力也搬进
  `ThinkingCard`（或明确砍掉），再让聊天改用它 —— **没做，别当成已经同源**。
- 它是**工具**的意义：能被 `tools` 步"调用"、能被图编排、能被审计/回放（前两条还没做）。

## 8. 落地阶段与判据（S1–S4）

| 阶段 | 内容 | 判据（可自证） |
|---|---|---|
| **S1 工具资产落地**（后端）✅ | ① 词表 + `ToolSpec` 的**唯一实现** = `l_node_engine/tool_spec.py`（宿主与 agent 都依赖它）；② 宿主认 `kind:"tool"`（PUT 校验走 `tool_spec_of`，形状坏当场 400）；③ `Tools/` 下的图当工具；④ 新出口 `GET /api/pygraphs/tools`（所有工程 + 内置工具，坏资产进 `errors` 不 500）；⑤ 端口表并进引擎（`_port_type_of`） | **实测**：`GET /api/pygraphs/tools` → `demo_tool_add`（图工具，`a/b` 带结构化默认 20/22、`container:number`）+ `describe_nodes`（python 工具，参数反射出来、标 `inferred`）+ `thinking`（内置）；PUT `kind:"tool"` → 200 `params:1`，**不再出现在 `/api/pygraphs` 的图列表里**；`/api/pygraphs/ports?kind=subgraph&…` 200；`wuwor l_node_engine -- l_node_engine_selftest` **全绿**（27 组） |
| **S2 agent 接线** ✅ | `l_agent_tool` 读工具资产（**默认工程平铺图 + 各工程 `Tools/`**；判别在引擎）；`kind:"tool"` 的 python 工具**真加载**（`load_python_tool`，过导入白名单）；读/写与审批来自 `flags`（`AgentTool.blocks_in_dry_run()` **声明优先**，老名单只做回落）；参数用 `ToolSpec.params`（`agent_params()`，**不压平**）；`/api/tools` 加 `params_detail` + `flags` | **实测**：注册到 `pygraph_demo_multi_engine`（历史平铺图）+ `pygraph_demo_tool_add`（工程 `Tools/`）+ `python_list_graphs`（python 工具）；图工具真跑 `add(a=1,b=2)` → `sum=3`；演练模式图工具**被拒** / `read_only` python 工具**照跑**；`:1250` `/api/tools` 三条都在、`params_detail` 带 `default`、`flags` 到位（166 个工具）<br>⚠ **分类后果**：`Functions/` 里的图不再当工具（要当工具得进 `Tools/`）<br>**回归**：`tests/test_pygraph_tools.py` **14 项全绿**（老契约保住三条：`OFF` 优先 / 显式目录只扫那一层 / 无 Input+Output 不算工具；`skipped` 键改成**资产地址**） |
| **S3 UI 同源** ✅ | **A（共享渲染器）**：新建包 `l_py_ui`（`containers.js` 容器词表+值编解码 / `tools.js` spec 归一 / `ToolUi.jsx` = `ParamForm`·`ToolHeader`·`ResultView`·`ParamRow`）；两个 vite 各自 `resolve.alias @pyshare` 指过去（同一个货架，构建期直读，**只有一份源码**）；聊天的 `ToolCard` 已改用它画参数行<br>**B（画布工具节点）**：引擎 `Tool` 节点（`tool_node.py`，`DYNAMIC_PORTS`，静态只有 `addr`）+ **宿主改写**（`_inline_tools`：图工具 → `SubGraph`；python 类工具 → `CallMethod`）+ `ports?kind=tool` 现算参数端口 + 内容浏览器列 🧰 行（`listTools()`）/ 点一下落 `Tool` 节点 + 节点参数区挂 `l_py_ui.ParamForm`（`PyNode` 的 `.pyg-tool-params`） | **A**：`l_py_ui` 单测 5/5；两个 web 构建通过<br>**B 后端**：`_inline_tools` 改写后**真跑**（图工具输出 `15`=7+8；python 类工具真返回图文件列表）；`/pro` 别名地址同样改写；缺 `addr`/非工具 → 400；`l_node_engine_selftest` PASS<br>**B 前端（真浏览器）**：新探针场景 **`tools` 13/13 PASS** —— 内容浏览器 `🧰 demo_tool_add` / `🧰 list_graphs`（`Tools/` 3 项）→ 点一下画布 `13 → 14` 节点、标题 `Tool` → `ports?kind=tool` **200** → `.pyg-tool-params` 在、参数 `a,b` 容器 `number,number`（= 工具 spec 现算）<br>⚠ **运维教训**：引擎包（`l_node_engine`）改完 **宿主不会**跟着热更（热更只覆盖自己那个包）→ 必须 `POST /api/v1/services/l_mindmap_mmd/reload`；否则 `object_info` 里还是旧元数据，前端认不出"这个端口引用工具"，**一步都不动**（这次实测：探针红在"没发 `ports?kind=tool`"，重启后 13/13） |
| **S4 整轮图化** | 迁移计划 ③ 第 3 步：`plan`/`tools` 执行本体进图；思考工具节点接 reasoning 帧 | 迁移计划 §3.2 既有判据（结构校验 0 error、真跑 2 圈收敛） |

## 9. 待拍板（用户）

1. **共享前端包的名字/位置** —— ✅ 已拍板：**`l_py_ui`**（`web/src` 一份源码，两个 vite 用 `@pyshare` 指过来；
   dev 要放开 `server.fs.allow`，否则 403 "outside of Vite serving allow list"）。
2. **`ToolSpec` 唯一实现放哪** —— ✅ 已拍板：**`l_node_engine`**（`tool_spec.py`；端口/容器/类型词表本来就在引擎里）。
3. **`Tools/` 下的图能否同时被子图/结构体引用**（建议：允许；目录只决定"是工具"，不限制引用）。
4. **agent 工具表要不要扫所有登记工程** —— ✅ 已拍板（S2 实现）：**要**，扫"默认工程平铺图 + 各登记工程
   `Tools/`"；工程登记表由引擎 `l_node_engine.paths.registered_projects()` 读（与宿主同一份文件）。
5. **思考工具的输出端口语义**（建议：无输入、`outputs=[{name:"text",type:"str"}]`）。
6. **审批字段与现有 `APPROVAL_TOOLS` 的关系** —— ✅ 已拍板（S2 实现）：`flags.approval` 为准
   （声明了就一定要人确认，连 `AUTO_APPROVE=1` 也不免）；没声明的老工具照旧走 `AUTO_APPROVE` + 那份名单。
7. **画布上的"工具节点"怎么落** —— ✅ 已拍板（用户 2026-10-13）：**B 做法 = 独立 `Tool` 节点**
   （不是"点工具直接落现成原语"）；且 **python 工具 = 类**（不是函数）。
   实现（S3-B 后端已完成）：引擎 `tool_node.Tool`（`addr` + `DYNAMIC_PORTS`）、宿主 `_inline_tools`
   改写成 `SubGraph`（图工具）/ `CallMethod`（python 类工具，类路径自动取/建实例），
   `_inline_tools` 夹在 `_inline_methods` 之后、`_inline_subgraphs` 之前。
