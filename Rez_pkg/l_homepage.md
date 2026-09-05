# l_homepage 依赖关系图页（deps）开发笔记

- 页面：`/homepage/deps`
- 模板：`l_homepage/999.0/src/l_homepage/templates/deps.html`
- 后端：`l_homepage/999.0/src/l_homepage/homepage_cli.py`
- 定位：展示服务/链接卡片的依赖拓扑，支持拖拽建边、右键启停、双击编辑、新建卡片。

> 说明：本节笔记记录对 deps 页面的迭代改动，供后续维护参考。

---

## 一、连线流动美术特效（血管/叶绿素）

### 结构分层
- 默认层 `.simple-flow`：低噪流动效果，`opacity:0`，`body` 下 `.flow` 时显示。每条边有 4 个发光小点（`animateMotion` + `mpath` 沿 `#edge-i` 运动，`dur=5s`，`begin` 按 `fi*0.85` 错开）。
- 增强层 `.bio-layer`：默认关闭，由 `body.flow-on` 门控开启。包含血管外壁/内壁/蠕动波 + 血细胞或叶绿素粒子。

### 主题切换（body 级 class）
- `body.flow-on`：开启增强生物特效（`bio-layer.opacity=.8`）。
- `body.flow-chloro`：切换为叶绿素配色（绿），否则为血红配色（红）。
- 血量/营养粒子通过 `bloodLayer(i,e)` / `chloroLayer(i,e)` 生成，分别绘制血细胞与叶绿素细胞。

### 血管蠕动实现
- 多条同心 path：`.vessel-outer` / `.vessel-inner`（血管壁）、`.vessel-peri` / `.vessel-peri2`（蠕动波）。
- 用 `pathLength="100"` 归一化 + `stroke-dasharray` 制造波峰波谷，如 `stroke-dasharray="10 40 4 46"`。
- `stroke-dashoffset` 动画（`values="0;-100"`）产生沿路径流动的蠕动波；`peri2` 用更快周期（2.2s）叠加次波，更生动。

### 关键 CSS
```css
.vessel-outer { stroke:rgba(220,38,38,0.22); stroke-width:6; }
.vessel-inner { stroke:rgba(153,27,27,0.30); stroke-width:3; }
.vessel-peri  { stroke:rgba(220,38,38,0.42); stroke-width:7.5; }
.vessel-peri2 { stroke:rgba(248,113,113,0.30); stroke-width:6.2; }
.flow-halo { fill:rgba(147,197,253,0.18); }
.flow-light { fill:rgba(147,197,253,0.9); filter:drop-shadow(0 0 3px rgba(147,197,253,0.85)); }
body.flow-chloro .flow-halo { fill:rgba(134,239,172,0.18); }
body.flow-chloro .flow-light { fill:rgba(134,239,172,0.9); }
```

### 导航按钮
- `#toggleFlowFx`：增强特效 开/关（切换 `body.flow-on`）。
- `#toggleFlow`：血红胞/叶绿素 切换（`flowMode` 在 `"blood"`/`"chloro"` 间翻转，改 `body.flow-chloro`）。
- `.flow` class 由 `refreshStatus()` 依据 from 节点在线状态切换：`ed.classList.toggle("flow", on)`。

---

## 二、新建卡片 + 预设选择

- 导航增加「＋ 新建卡片」按钮 `#addCard`。
- `openEditor(null)` 走 `isNew` 分支，`POST /api/v1/services` 新建；编辑走 `PUT`。
- `_SvcIn` Pydantic 模型含 `reload_args: list[str]=[]` 与 `reload_cmd: str=""`，新建时分别置 `[]` 与 `""`。
- 新建表单顶部有「选择预设」下拉 `#ed_preset`，**数据源来自主页卡片**（`GET /api/v1/services/deps` 返回的完整 `nodes`，含 name/kind/port/url/desc/icon/newtab/depends/packages/run_args/run_cmd）。
- 选择预设后通过 `change` 事件回填全部表单字段，可再修改；若名称与现有卡片重复需改名（否则后端 409）。

```js
if (isNew) {
  var preset = overlay.querySelector("#ed_preset");
  preset.addEventListener("change", function () {
    var pk = nodes.find(function (x) { return x.name === preset.value; });
    if (!pk) return;
    // 回填 ed_name / ed_kind / ed_port / ed_url / ed_desc / ed_icon / ed_newtab / ed_depends / ed_packages / ed_run_args / ed_run_cmd
  });
}
```

---

## 三、节点状态：圆点 → 文字

- 移除 `.status-dot`（`<circle>`），改为 `.status-txt`（`<text>`），`text-anchor:end` 右对齐、`font-size:9px`。
- 位置：节点右上角 `(p.x + nodeW - 10, p.y + 15)`。
- 状态词映射：
  - `在线`（`up`，绿，HTTP 响应）
  - `监听`（`tcp`，蓝，端口通但 HTTP 无响应）
  - `等待`（`waiting`，琥珀，依赖未就绪）
  - `启动中`（`starting`，青，脉冲 `blink` 动画，含重启计时）
  - `停止`（`down`，灰）
- `link` 或无 `port` 的节点隐藏状态文字（`display:none`）。
- `applyStatus()` 中按状态设置 `.status-txt` 的 class 与 `textContent`；`moveNode` 拖拽时同步重设 `x/y`。

```css
.status-txt { fill:#64748b; font-size:9px; font-weight:600; text-anchor:end; transition:fill .3s; pointer-events:none; }
.status-txt.up { fill:var(--green); }
.status-txt.tcp { fill:#38bdf8; }
.status-txt.waiting { fill:#f59e0b; }
.status-txt.starting { fill:var(--cyan); animation:blink .7s infinite alternate; }
.status-txt.down { fill:#64748b; }
```

---

## 四、连线流动 主连线 `.edge` 由纯实线改为流动虚线：`stroke-dasharray:8 6` + `animation:dashFlow 1.1s linear infinite`。
- `dashFlow` 把 `stroke-dashoffset` 平滑推到 `-14`（等于一个 dash + 间隙周期，无缝循环），实现沿路径流动的虚线。
- `.edge.flow`（在线变红）与 `.edge.hl`（高亮）仍继承流动；`.flow-chloro` 模式变绿的 `.edge.flow` 同样流动。

```css
.edge { fill:none; stroke:rgba(91,156,255,0.45); stroke-width:1.4; stroke-dasharray:8 6; transition:stroke .15s; animation:dashFlow 1.1s linear infinite; }
@keyframes dashFlow { to { stroke-dashoffset:-14; } }
```

---

## 五、底部图例/操作说明重叠修复

- 根因：`.legend`（左，6 项图例）与 `.hint`（右，长操作说明）各自 `position:fixed` 独立定位，窄视口互相挤压重叠。
- 方案：新增 `.footer` 底部容器（`fixed`，`left/right:18px`，`flex` + `space-between` + `flex-wrap`），把 `legend` + `hint` 包入；二者改为容器内流式布局并去掉独立定位。
- `.hint` 加 `max-width:560px; line-height:1.7`，长文字自动换行。

```css
.footer { position:fixed; left:18px; right:18px; bottom:14px; z-index:10; display:flex; gap:12px; align-items:flex-start; justify-content:space-between; flex-wrap:wrap; pointer-events:none; }
.footer > * { pointer-events:auto; }
```

---

## 六、后端接口要点

- `GET /api/v1/services/deps`：返回全部服务/链接作为 `nodes`，含 `name, kind, port, depends, url, desc, icon, newtab, packages, run_args, run_cmd, reload_args, reload_cmd`。
- `POST /api/v1/services`：新建，入参 `_SvcIn`（含 `reload_args`/`reload_cmd`）。
- `PUT/DELETE /api/v1/services/{name}`：编辑/删除。
- `GET /api/v1/services/status`：轮询状态（3s），返回 `{name:{up,http,pid,hot}}`。
