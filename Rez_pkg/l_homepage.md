# l_homepage 主页卡片 / 局部刷新开发笔记

> 源码热加载、`L_SRC_WATCH`、守护和自重启的现行说明集中在[src_hot_reload_源码热重载与主页常驻.md](../src_hot_reload_源码热重载与主页常驻.md)。本页仅保留主页专属的卡片、deps、SSE 局部刷新与前端排错；`l_notepad_server` 的 Python 改动必须手动重启，勿以本页历史热加载描述作为保障。


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

---
# l_homepage 主页卡片 / 热加载 / 局部刷新 开发笔记

> 上面一节是 deps 页面的笔记；本节记录主页（`home.html`）的卡片配置模型、源码热加载、局部刷新迭代，供后续维护参考。

## 卡片配置：包内默认 + 用户只存差异

- **唯一默认来源**：`src/l_homepage/config/services_builtin.json`（16 张，只读、随包发布；改它等于改所有用户的默认卡）。
- **用户配置**：`~/.lugwit/l_homepage/runtime/services.json`，只存三类：自定义卡、对默认卡的覆盖（**同名即覆盖**）、删除墓碑 `{"name": "...", "removed": true}`。
- `ServiceCard`（dataclass）字段：`name, url, origin_url, desc, icon, newtab, port, kind, auto_start, packages, run_args, run_cmd, reload_args, reload_cmd, depends, builtin, overridden, overridden_fields`
  方法：`from_raw`（校验 + 历史迁移 + 补 `.solo`）、`to_payload`（写盘形状）、`to_builtin_item`（写进包内默认的形状）、`to_dict`（API/模板形状：内容 + `builtin/overridden/overridden_fields/override_tip`）、`differs`、`diff_fields`、`override_tip`（悬停提示文本，后端拼好）。
- `load_services()`：包内默认 + 用户差异合并（内置在前、自定义在后）；`save_services()`：只落「自定义卡 + 与包内不一致的覆盖」——与默认一致时**自动不落盘**，改回默认即消失；列表里缺失的默认卡自动记墓碑。
- 卡片名是不可变锚点（`PUT` 要求 URL 名 == body 名），故覆盖以 `name` 关联，不需要额外 id 字段。
- 界面标识：`⚙ 服务 / 🔗 链接`；`📦 默认 / ✎ 覆盖默认`（悬停显示逐字段「原值 → 现值」）；`覆盖为系统设置` 按钮。
- `POST /api/v1/services/promote-builtin?name=<卡名>`：把当前卡片写回包内默认文件（同名替换、无则追加），随后该卡即「默认」，用户侧覆盖记录被清掉。
- deps 页节点也带 `builtin/overridden`（图例「📦 包内默认卡 / ✎ 覆盖默认」）。

## 源码热加载（watchfiles → watchdog → 轮询）

- 组件：`l_app_ready.hotreload.SrcHotReload`；`l_homepage` 侧用 `L_SRC_WATCH*` 接入。
- **驱动自动降级**：`watchfiles`（Rust 事件）→ `watchdog`（纯 Python，`ReadDirectoryChangesW`）→ 轮询；`L_SRC_WATCH_BACKEND` 可固定其中一种。
- 事件只负责**叫醒**主循环；是否重启仍由一次 mtime 扫描 diff + debounce(2s) + cooldown(10s) 判定；事件漏报由 **10s 兜底轮询**补（`_FALLBACK_SCAN_SECS`）。
- 改动分两类：`.py`（`restart_exts`）→ `restart_cb` → 整进程重启；`.html/.j2` 等 → `on_frontend_change(changes)`（可选回调，l_homepage 用它做 SSE 推送），**不重启**。
- 事件源刚装上时可能还没"武装"完 → `start()` 后立刻补扫一次，避免"刚启动就改文件"被漏掉。
- **开关**：`L_SRC_WATCH`（默认 1）+ 运行时切换（`GET/POST /__dev__/src_watch`）；状态持久化到 `runtime/src_watch.json`，**显式 env 优先于存档**（guard、重启执行进程会显式给 0）。
- `.py`（`restart_exts`）→ 整进程重启；`.html/.j2` 等 → **不重启**（Jinja `auto_reload` 下次渲染即用新版）。
- **主页自重启链路**：`_spawn_self_restart(trigger)`（独立进程 + `L_SRC_WATCH=0` + 输出写入 `homepage.log`）→ `python -m l_homepage.homepage_cli restart_self_cli` → `_restart_self()`：
  1) 杀 8090 监听进程 + 其父进程；2) **清掉所有匹配别名的 `.solo` 启动链残留**（`_kill_stale_solo_wrappers`）；3) 等端口真正释放（最多 5s）；4) `wuwor l_homepage .solo -- homepage_start` 起新进程；5) 同步记一条重启历史（等出新 PID 再落盘）。
- **重启历史**：`runtime/restart_history.jsonl`（上限 400），字段 `ts, name, op, trigger, port, pid_before, pid_after, restarted, ok, error, files, secs`；`trigger` 枚举：`user / card / header / watchdog / src-watch / guard`。
  - `GET /api/v1/services/history?name=&limit=` → `{events, file, src_watch, templates_stamp}`。
  - 卡片 🕘 弹窗：3 秒自刷 + 展示驱动模式/监视目录/扩展名/兜底周期/记录文件 + 每条事件「PID 旧 → 新」。
- 卡片状态行**常显** `旧 PID → 新 PID`（数据来自状态接口的 `last_restart`）；检测到新事件时 🕘 闪 5 秒（`src-watch` 另弹 toast）。
- 🕘 图标随驱动模式换样式：`⚡watchfiles / 👁watchdog / ⏳poll / 🕘关闭`（数据来自状态接口的 `src_watch`，零额外请求）。

## 局部刷新（Jinja block + 哈希 diff）

- **规则**：把区域包成 `{% block 名 %}`，外层元素 id 用 `blk-名` → **新增可刷新区域只需改模板 1 处**，前后端零改动。
  现有区域：`brand(#blk-brand)`、`user_bar(#blk-user_bar)`、`foot(#blk-foot)`、`grid(#blk-grid)`、`page_style(#blk-page_style)`、`deleted(#blk-deleted)`。
- 后端**自动发现**块名（`_home_block_names()` = `templates.get_template("home.html").blocks`），不维护名单。
  注意：**渲染单个块不会执行模板顶部的 `{% from %}`**，所以块内用到的宏要自带 import（`grid` 块里就是这么写的）。
- **触发**（两级）：
  1. **SSE 推送（主）**：`l_app_ready.hotreload` 新增可选回调 `on_frontend_change(changes)`，`_tick()` 检测到**非重启类**改动（`.html/.j2` 等）时调用它（`.py` 仍只走 `restart_cb`，两者互不混淆）；`l_homepage` 侧 `_on_frontend_change` 广播到订阅队列，`GET /api/v1/homepage/events` 用 `StreamingResponse` 推 `{"kind":"frontend"}`，前端 `EventSource` 收到即 `refreshBlocks()`。响应头带 `X-Accel-Buffering: no`（nginx 见到会对该响应关 `proxy_buffering`）→ **不必改 nginx 配置**；每 15s 一个 `ping` 兼作 `proxy_read_timeout` 保活。
  2. **`/stamp` 轮询（兜底）**：SSE 连通时放宽到 10s，断线回落 1s；标签页隐藏时跳过，且不依赖「自动刷新」开关。
  实测（线上，watchfiles）：touch 模板后 **0.15s** 收到推送事件；从存盘到换块总延迟 ≈0.2s（早期 5s 轮询版 / 1s 轮询版分别是 ≤5s、~1.05s）。
- **决策**：`POST /api/v1/homepage/blocks`，请求带上次各块哈希 → 响应 `{changed:{块:HTML}, hashes, shell_hash, reload_required}`，只回变化块。
  `shell_hash` = 页面 `<script>` 段 + **不属于可热替换块**的 `<style>` 段（算指纹前先把 `page_style` 块挖空）。
  所以：改 CSS → 只换 `page_style`（前端 `OUTER_BLOCKS` 用 `outerHTML` 替换 `<style>` 元素，**无需刷新**、立即生效）；
  改 `<script>` 或 `_log_viewer.html` 内的样式 → `reload_required=true` → 整页刷新（有弹窗时改为提示手动 F5）。
- 卡片 HTML 只有一份：`templates/_grid.html` 的宏，被 `grid` 块引用；卡片增删改 / 覆盖为系统设置 / 模板改动**统一走 `refreshBlocks()`**。
- 页面初始化时调一次 `refreshBlocks()` 建立基线（同时建立 shell 基线、验证接口连通）。

## 布局：卡片 4 列（卡片宽度不变）

- 只改两处 `max-width`（都在 `home.html` 的 `page_style` 块里）：`.wrap` 与 `.topbar` 由 `1080px` → **`1432px`**；`.grid` 的 `repeat(auto-fill,minmax(268px,1fr))` + `gap:16px` **不动**。
- 为什么是 1432：容器上限 1432 − 左右 padding 36 = 内容 **1396px**；4 列需 `4×268+3×16=1120 ≤ 1396` ✓，5 列需 `5×268+4×16=1404 > 1396` ✗ → 恰好 4 列，每列 `(1396−48)/4 = 337px`，与改前 3 列时的 `(1044−32)/3 ≈ 337.3px` **一致**（卡片宽度不变，内容宽 +352px）。
- 窗口变窄时 `auto-fill` 自动回落 3/2/1 列，不会挤压卡片。
- 验证方式：jinja 渲染后用正则从**实际 CSS** 取 `wrap / grid min / gap`，再算 `列数 = (内容+gap)//(min+gap)`、`每列 = (内容−gap×(列数−1))/列数`，断言 `列数==4` 且与改前宽度差 <1px。
- 纯 CSS 改动 → 走 `page_style` 块热替换，**已打开的页面 ~0.2s 自动换样式，不刷新**。
- 已知小瑕疵（未改）：`.topbar` 有两条同名规则，后一条把 padding 覆盖成 `10px 2px 0`，所以顶部品牌栏比卡片左边缘少缩进 16px；要对齐就把那条改成 `10px 18px 0`。

## 接口一览（路径受 nginx 约束）

> nginx 只把 `^/api/v1/services(/|$)`、`^/api/v1/homepage(/|$)`、`^/api/v1/nginx(/|$)` 反代到主页；其它 `/api/v1/*` 兜底转 auth(1027) → 会拿到 404。另外 `/api/v1/services/X/Y` 两段式会被通用路由 `{name}/{op}` 先匹配走。

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/api/v1/services/status` | 卡片状态 + `last_restart` + `templates_stamp` + `src_watch` |
| GET | `/api/v1/services/history` | 重启历史（`name` 可过滤，`limit` 默认 60） |
| POST | `/api/v1/services/promote-builtin?name=` | 覆盖为系统设置（写回包内默认卡） |
| GET | `/api/v1/services/deleted` | 已删除的内置卡（墓碑记录），面板数据源 |
| POST | `/api/v1/services/restore-builtin?name=` | 恢复被删的内置卡（清墓碑，回原位） |
| POST | `/api/v1/services/purge-builtin?name=` | **彻底删除**内置卡（改包内默认卡文件，不可恢复） |
| POST | `/api/v1/services/{name}/{op}` | start/stop/restart/reload/hotstart |
| POST | `/api/v1/services/restart-all` | 按拓扑顺序重启（一段式，避开 `{name}/{op}`） |
| POST | `/api/v1/services/deps/layout` | 依赖图布局（注册顺序在 `{name}/{op}` 之前） |
| GET | `/api/v1/services/{name}/log` | 卡片后台日志 |
| GET/POST | `/__dev__/src_watch` | 源码热加载开关 |
| POST | `/api/v1/homepage/blocks` | 局部刷新片段（块哈希 diff） |
| GET | `/api/v1/homepage/stamp` | 极轻量：`templates_stamp` + 驱动状态（SSE 的兜底轮询） |
| GET | `/api/v1/homepage/events` | SSE：推 `{"kind":"frontend"}`（前端文件改动）+ 15s 心跳 |
| POST | `/api/v1/homepage/restart` | 重启主页自身（独立进程） |

## 已删除的内置卡：查看 + 恢复

删除内置卡不是真删，而是往 `runtime/services.json` 写一条**墓碑**（`{"name": X, "removed": true}`，
`save_services()` 按"列表里没有的默认卡 = 被删"自动生成）。历史上没有入口看回/恢复，只能手改 JSON。

- 后端：`_removed_names()`（读墓碑）+ `deleted_builtin_cards()`（墓碑 ∩ 包内默认；包内默认已删掉的
  孤儿墓碑不展示）。`GET /api/v1/services/deleted` 列表，`POST /api/v1/services/restore-builtin?name=` 恢复。
- 恢复实现：`SERVICES.append(base)` → `save_services(SERVICES)`（名字又在列表里 → 墓碑被清）→
  `SERVICES[:] = load_services()`（重排成"包内默认在前 + 用户卡在后"）。放回的是**包内默认卡本体**，
  内容一致 → `services.json` 里不留覆盖记录。重复调用返回 `already: true`；名字不在包内默认里 → 404。
- 前端：顶部 `🗑 已删除默认卡 (N)`（仅管理员，计数来自 `deleted_cards|length`）→ `toggleDeleted()`
  给 `#blk-deleted` 加/去 `.show`；面板内容来自 `templates/_deleted.html` 宏，被 `deleted` 块引用。
  恢复后 `refreshBlocks()` 同时刷新 `deleted` 块（列表）与 `user_bar` 块（计数）。
  外层 `#blk-deleted` 不在块内，所以局部刷新**不会丢 `.show` 展开状态**。
- 用户自定义卡（非包内默认）删除后无法恢复：它们不进墓碑，直接从 `services.json` 移除。
- **彻底删除**（面板里「恢复」右边的按钮 `purgeBuiltin()`）：内置卡只是"记了墓碑"，想让它永远别再出现
  必须改**包内容**——`remove_builtin_card()` 从 `config/services_builtin.json` 里移除该条目
  （原子写 `.tmp` → `os.replace`，保留 `version`/`note`，其它卡不动），随后 `save_services(SERVICES)`
  顺手把墓碑也清掉（该卡已不是默认卡，不会再被判成"被删的默认卡"）。
  守卫：卡不在包内默认里 → 404；卡**还在主页上**（没先删除）→ 400 提示先删卡片。
  与「删除卡片」（只写墓碑、可恢复）是两回事，所以确认框用 `danger` 并把后果写清楚。

## 覆盖明细浮层（悬停「✏️ 覆盖默认」）

- **要解决三次**（同一个交互，被报了三个 bug）：
  1. 角标 `✎`(U+270E) 缺字形 → 显示成 `?`（见下方字形小节）。
  2. 浮层**再也弹不出来** —— 监听用 `querySelectorAll(...).forEach(badge.addEventListener("mouseenter"))`
     一次性绑在角标元素上，而 `refreshBlocks()` 初始化与每次模板改动都整体替换 `#blk-grid` 的 innerHTML，
     角标被换成新元素 → 监听随之消失。
  3. 浮层**一移开角标就没了，里面的按钮点不到** —— 隐藏判定依赖 `mouseover`/`mouseleave` 的触发顺序：
     指针移进浮层时 `document` 上的 `mouseover`（委托处理器）会**再排一次隐藏**，浮层自身的
     `mouseenter` 那次 `clearTimeout` 已经被它排在后面，240ms 后浮层消失 → 按钮永远点不到。
- **最终实现**：
  - **显示**：事件委托（`document` `mouseover` + `closest(".src-badge.over[data-override]")`）→ 换块后照样能弹。
  - **保持/隐藏**：`document` `mousemove` + **几何判定** —— 只有当指针**既不在角标 rect、也不在浮层 rect**
    内（外扩 3px）时才 `hideLater(240)`（给跨 6px 缝隙留时间）；在任一 rect 内则 `cancelHide()`。
    浮层自身再挂 `mouseenter`/`mouseleave` 兜底"指针停在原地不动、`mousemove` 不再触发"的情况。
    **不要用 `mouseover`/`mouseleave` 的先后顺序来判"该不该收"** —— 顺序不可依赖，这就是 bug 3。
  - **内容**：`字段 / 包内默认 / 当前值` 三列表格（用户诉求），且**只用 CJK+ASCII**，从根上避免缺字形。
  - **数据**：角标带 `data-fields='{{ s.overridden_fields | tojson }}'` —— **必须单引号**，Jinja `tojson`
    不转义 `"`（只转义 `< > & '`），双引号会把属性截断；解析失败回退 `data-tip` 多行文本。
  - 定位仍挂 `body` + `position:fixed`（卡片 `overflow:hidden` 会裁掉面板），宽度 470px。

---

## 应用内确认框 `uiConfirm`（替代 window.confirm / alert）

原生 `confirm`/`alert` 的标题栏、按钮是浏览器/系统样式，无法换主题，与页面观感完全脱节（"js 的原生通知太丑了"）。
改为页面自己的确认框：

- `uiConfirm({title, lines: [...], okText, danger})` → `Promise<boolean>`（`false` = 取消）。定义在 `home.html`。
- 复用 `.modal-mask` / `.modal` 样式（玻璃卡片，与卡片编辑弹窗一致）；DOM **动态创建并挂 body** →
  不受 `refreshBlocks()` 换块影响，也不用改模板。
- 交互：Esc = 取消（`keydown` 用 **capture + stopPropagation**，否则会和卡片弹窗/历史弹窗的 Esc 处理器打架）、
  Enter = 确认（确认钮自动聚焦）、点遮罩 = 取消。
- `danger:true` → 确认钮变红，用于删除/停止/重启类。
- 已全量替换：**覆盖为系统设置、删除卡片、恢复内置卡、重启主页、停止服务、重启全部**；
  失败/成功提示统一走 `showToast` / `refreshBlocks(msg)`，不再有 `alert`。

---

## 模板字形：非 emoji 的 dingbat 会显示成 `?`

`✎`(U+270E)、`⧉`(U+29C9) 这类字符没有 emoji 变体，浏览器落到缺字形的字体上就渲染成 `?`。
「✏️ 覆盖默认」角标的 `✎` 就是这么变成问号的。**规则：图标一律用 emoji（可带 VS16 `\uFE0F`）**，
如 `✏️`、`📦`、`🗑`、`📜`、`🕘`；纯文本符号只在确定字体覆盖时用。

判断依据（实测）：`✎` 不在 GBK/常用中文字体里（U+270E），而 `·`(U+00B7)、`→`(U+2192) 在 GBK 内，
且混有中文的行会走中文字体，所以那两个符号安全；`✏️`(U+270F+VS16) 走 emoji 字体，也安全。
**新一代码位（U+2xxx 以上、非 emoji）最容易踩这个坑。**

补充实测（同一台机、Chromium，把候选字符排一排截图对比）：

| 字符 | 结果 |
|---|---|
| `🗑`(U+1F5D1，无 VS16) | 12px 下退回文本字形，看着就是个**灰块**（"已删除默认卡"按钮上的那个方块就是它） |
| `🗑️`(带 VS16) | 能出彩色 emoji，但 12px 小字下**细弱发灰**，观感仍差 |
| `↩`(U+21A9)、`♻`(U+267B)、`✕`(U+2715)、`⟲`、`⧉`、`📦`、`✏️` | 清晰可用 |

**结论：小尺寸按钮/角标优先"纯文本"或上表里已验证清晰的符号；别在小字号上用冷门 emoji。**
「已删除默认卡」按钮、面板标题、以及新加的「彻底删除」按钮最后都改成**纯文本**了。

---

## 文件与环境变量

- 代码/模板：`homepage_cli.py`、`templates/home.html`、`templates/_grid.html`、`templates/deps.html`、`templates/_log_viewer.html`、`templates/login.html`、`config/services_builtin.json`。
- 运行时目录（`~/.lugwit/l_homepage/runtime`，可用 `L_HOMEPAGE_RUNTIME` 覆盖）：`services.json`、`src_watch.json`、`restart_history.jsonl`、`homepage.log`、`watchdog.log`、`.homepage.pid`、`.homepage.guard.pid`、`.homepage.port`、`deps_layout.json`。
- 环境变量：`L_SRC_WATCH`、`L_SRC_WATCH_BACKEND`、`L_SRC_WATCH_INTERVAL`、`L_HOMEPAGE_PORT`、`L_HOMEPAGE_SERVICES`、`L_HOMEPAGE_RUNTIME`、`L_HOMEPAGE_BASE_URL`、`L_HOMEPAGE_NGINX_PORT`、`L_HOMEPAGE_WATCHDOG_INTERVAL`、`L_HOMEPAGE_WATCHDOG_WAIT_READY`、`L_HOMEPAGE_RESTART_TRIGGER`（内部：标记重启来源）。

## 日志窗口：增量渲染 + 性能显示（`_log_viewer.html`）

**问题**：老实现每次刷新都对整段日志重新 `split` + 高亮 + `innerHTML` 重建。日志一大
（反复重启时一次就是几百行 wuwo/rez 启动日志）每秒重建几千行 → 浏览器直接卡死。

**改法**（全在前端；后端接口没动——它本来就支持 `offset` 增量）：

- **只渲染新增部分**：日志按"块"存 `_blocks`，`_nodes` 与之一一对应（null = 被过滤，不占 DOM）；
  新行解析完直接 `appendChild(fragment)`，**绝不再重建已显示内容**。
- **有界**：`MAX_KEEP=4000` 行上限（超出丢最旧块并同步删 DOM，显示"丢 N 行"）；
  单次增量 `MAX_CHUNK=3000` 行上限（风暴时一次返回一大段也不至于顶死页面）。
- **轮转不再全量拉**：后端 `reset=true`（文件轮转/截断）时**不从头拉**，改回 `tail=300` 取尾部；`open()` 同样只取尾部。
- **刷新并发保护**：上一轮 fetch 未回来就跳过本轮（显示"跳过 N"），避免请求叠加。
- **整体重建只在用户操作时**：过滤/搜索变化才 `_rebuild()`，且搜索输入**防抖 180ms**。
- **性能显示**（`#lgLogPerf`）：`拉取 x ms · 解析 x ms · 渲染 x ms · +N KB/N 行 · 保留 N 行 · 丢 N 行 · Σ x ms`
  —— 卡不卡一眼可判，不用猜、不用翻 devtools。

**顺手踩到的坑**：`_rebuild()` 判断"有没有可显示内容"用了 `frag.childNodes.length`，
但 `appendChild(fragment)` 会把 fragment **掏空** → 永远判成 0 → 有匹配行时也插一句"（无匹配行）"。
**要提前用计数器（`made`）记。**

---

## 踩过的坑（含根因）

1. **热加载/重启"点了没反应、PID 不变"**（最坑）：
   `.solo` 守卫按「cmdline 含 `homepage_start`」判重，而主页启动链（`cmd.exe` → `wuwo_rez.py` → `rez.exe` → python）比监听进程多好几层；`_restart_self` 原先只杀「监听 + 一层父」，残留启动链仍匹配别名 → 新实例被守卫判为"已有实例"**静默退出**，而执行进程输出又被丢进 DEVNULL，日志毫无线索。
   修法：起新实例前 `_kill_stale_solo_wrappers()` 清掉所有匹配别名的启动链；执行进程输出改写入 `homepage.log`；`_wait_new_pid` 超时 12s → 25s。
2. **前端报 `Unexpected token '<'`**：主页没在监听 → nginx 返回 502 的 HTML，而 `_api()` 直接 `r.json()`。
   排查口诀：`curl -s -o nul -w "%{http_code}" http://127.0.0.1:8090/healthz`（`000` = 没起）→ 再看 `restart_history.jsonl` 的 `pid_after` 是否为 `null`。
3. **nginx 前缀**：新接口必须落在 `/api/v1/services/…`（**一段式**）或 `/api/v1/homepage/…` 下；`/api/v1/builtin/…` 会被兜底转给 auth → `{"detail":"Not Found"}`。
4. **两段式被抢**：`/api/v1/services/{name}/promote` 会被先注册的 `POST /api/v1/services/{name}/{op}` 匹配（`op="promote"` → 400 未知操作）。故改用一段式 `/api/v1/services/promote-builtin?name=`。
5. **主页卡片走通用启停路径会自杀**：通用路径先杀端口监听（正是当前处理请求的进程），又被 reloader/守卫拉回 → PID 不变。自卡分流到 `_manage_self_card` → 独立进程自重启；`stop` 直接拒绝并提示用「重启」。
6. **JS 注释里写字面量 Jinja 标签**（如 `{% block %}`）会导致模板编译失败 —— 注释里改成「Jinja block」。
7. `re` 未在模块顶层导入，`_shell_hash` 报 NameError；`portal_home` 漏传 `src_watch_enabled` 导致「♻ 热加载」状态显示错误（已合并到 `_home_context`）。
8. **删除内置卡后找不回来**：墓碑写了但没有任何入口能看/恢复 → 补「🗑 已删除默认卡」面板（见上节）。
9. **角标「覆盖默认」显示成 `?`**：`✎`(U+270E) 非 emoji，字体缺字形 → 换 `✏️`（同一批还修了浮层弹不出来，见第 11 条）。
10. 新增块的上下文变量必须同时加进 `_home_context()`（首屏与局部刷新共用），否则局部刷新时
    该块渲染成空 —— 用 `jinja2.StrictUndefined` 渲染整页可立刻暴露漏传。
11. **悬停「覆盖默认」角标：浮层弹不出来、角标显示成 `?`、浮层里的按钮点不到**（同一个交互被报了三次，三个独立原因）：
    - `?`：角标写的是 `✎`(U+270E，非 emoji、中文/GBK 字体里没有该字形) → 换成走 emoji 字体的 `✏️`。
    - 浮层弹不出来：监听是 `querySelectorAll(".src-badge.over").forEach(b => b.addEventListener("mouseenter", …))`，
      **一次性绑在角标元素上**；而 `refreshBlocks()` 在初始化与每次模板改动时都会整体替换 `#blk-grid` 的
      innerHTML → 角标变成新元素 → 监听全丢。现象具有欺骗性：`#ovPop` 节点在、`display` 为空、
      `innerHTML` 为空，好像"根本没写这个功能"。**用模拟事件也测不出来**（`dispatchEvent` 也打不到已丢的监听），
      必须真浏览器 hover。→ 改**事件委托**（`document` `mouseover` + `closest(".src-badge.over[data-override]")`）。
    - **浮层按钮点不到**：隐藏判定依赖 `mouseover`/`mouseleave` 的触发顺序 —— 指针移进浮层时 `document` 上的
      `mouseover` 委托处理器会**再排一次隐藏**，把浮层自身 `mouseenter` 的 `clearTimeout` 覆盖掉 → 240ms 后
      浮层消失，按钮永远点不到。→ 改 **`mousemove` + 几何判定**：指针不在角标 rect、也不在浮层 rect 内
      （外扩 3px）才收起；浮层 `mouseenter`/`mouseleave` 兜底"指针不动、`mousemove` 不触发"。
      **教训：不要用 `mouseover`/`mouseleave` 的先后顺序判"该不该收"，顺序不可依赖。**
    - 顺带把浮层正文从 `· 字段：旧 → 新` 拼串改成 `字段 / 包内默认 / 当前值` 三列表格（用户诉求），
      且**只用 CJK+ASCII**，从根上避免缺字形。
    - **教训：凡是被 `refreshBlocks()` 换掉的 DOM，交互监听必须挂在不被换掉的祖先上（事件委托），
      或每次换块后重新绑定。** 同类隐患：`_grid.html` 里任何 `onclick` 之外的手工 `addEventListener`。
12. 模板改动检测曾有两层（逐文件戳 + `_grid.html` 特判），已统一为「总戳触发 → 块哈希决策」。
13. **原生 `confirm`/`alert` 与页面观感脱节** → 全量换成 `uiConfirm` + `showToast`（见上节）。
    **踩坑提醒：检索时别只搜 `window.confirm` —— 库里存在裸 `confirm(`（未带 `window.`），
    我第一次只搜 `window.confirm` 就漏了两处（停止服务、重启全部）。** 正确姿势：
    `(?<![\w.])confirm\(` / `(?<![\w.])alert\(`。
14. **"删除内置卡"其实只是记墓碑**：用户看到"已删除"以为再也回不来，实际数据还在（可恢复）；
    想要真删必须改包内默认卡文件 → 补了「彻底删除」（见上节）。**命名上区分开**：
    `DELETE /api/v1/services/{name}` = 记墓碑（可恢复）；`purge-builtin` = 改包内容（不可恢复）。
15. **小字号上的 emoji 会"变灰块"**：`🗑` 无 VS16 时 12px 下退回文本字形，看着就是个方块
    （用户截图里的"已删除默认卡"按钮）。详见「模板字形」小节的实测表。
16. **日志窗口把浏览器卡死**：每次刷新全量重建 innerHTML（详见「日志窗口」小节）。
    教训：**增量数据源 + 全量重建渲染 = 卡死**；要么增量渲染（本项目的做法），要么分页/虚拟滚动。
17. **同一个服务被三路同时"停旧 + 起新"** → 双实例 / 无限重启 / 日志刷爆：
    热重载监视线程、主页 watchdog、用户点重启互不知情。修法见《src_hot_reload…》「重启的并发防护与熔断」
    （跨进程锁 + 熔断退避 + watchdog/`_svc_manage` 避让）。教训：**重启这种"全局副作用"必须有跨进程的互斥**，
    进程内标志在"每次重启都是新进程"的架构下等于没有。
18. `appendChild(fragment)` 会掏空 fragment：用 `frag.childNodes.length` 判"有没有内容"永远是 0
    （见「日志窗口」小节末尾）。

## 验证清单（可复用）

1. `python -m py_compile homepage_cli.py`（顺便 `py_compile l_app_ready/hotreload.py`）。
2. 真 jinja2 渲染（仓库内已有 `rez-package-3rd/jinja2/3.1.6/...`、`markupsafe/...`）：用 `StrictUndefined` 渲染整页（漏传变量直接报错）+ 逐块渲染 + 断言首屏 `#blk-grid` 内容与 `grid` 块输出一致。
3. 内联 JS 用 `node --check`（先把 `{{ }}` / `{% %}` 替换成占位再检查）。
4. 离线验证端点逻辑：stub 掉 `fastapi/pydantic/l_app_ready`，把 `rez-package-3rd` 下所有 `python-3.12/python` 加进 `PYTHONPATH` 后导入 `homepage_cli`，直接调端点函数（可断言 `changed`/`reload_required` 各分支）。
5. 运行期：`curl` 状态与片段接口；`(Get-Item homepage_cli.py).LastWriteTime = Get-Date` 触发一次热加载，核对 `restart_history.jsonl` 出现 `pid_after` 非空、且 8090 监听 PID 已变化。
6. **DOM 交互类改动（浮层/角标/卡片按钮）必须真浏览器 hover 一次**：起 Playwright → `POST /homepage/login`
   （`{username:'fqq', password:'qwer'}`，cookie 是 httponly 所以 `document.cookie` 看不到、属正常）→ 打开 `/homepage`
   → `hover` 目标元素 → 断言 `#ovPop` 的 `display` / `.ov-fields tbody tr` 行数。
   **块刷新会换掉 DOM，`dispatchEvent(new MouseEvent('mouseenter'))` 测不出"监听丢了"**（这也正是上面第 11 条被漏掉的原因）。
   → 再 `hover "#ovPromote"`（这一步不会真点按钮、不写 `services_builtin.json`）并 **await 900ms 后重新断言
   `#ovPop` 仍是 `display:block` 且按钮还在** —— 这条专门挡"鼠标一移开浮层就收、按钮点不到"的回归。
   → 最后 `hover "h1"` 再等 600ms，断言 `display:none`（确认还能正常收）。
7. **确认框类改动**：真浏览器里**点开对话框**（点开 ≠ 确认，不会写盘）→ 断言 `.confirm-modal` 的标题/正文行/
   按钮文案与 `danger` 样式、确认钮已聚焦；`Escape` 后断言 `.confirm-modal` 数量归零。
   测试前记下目标文件的 `sha1 + mtime`，测完核对**完全未变** —— 证明测试本身零副作用（这条同时验证了"取消不会写盘"）。
8. 内联 JS 语法：抽出每个 `<script>` 段，把 `{{ }}`/`{% %}` 替换成占位后 `node --check`（仓库机器上 `node` 可用）。
9. **会写包内文件的动作（promote-builtin / purge-builtin）必须在临时副本上测**：
   导入 `homepage_cli` 后把 `H.BUILTIN_FILE` 指到 tempdir 里的副本（函数在调用时读模块全局，patch 生效），
   `L_HOMEPAGE_RUNTIME`/`L_HOMEPAGE_SERVICES` 也指 tempdir；测完用真文件的 `sha1 + mtime` 断言**没被动过**、
   且没有残留 `.tmp`。真浏览器里**只点开确认框再 Esc**（点开 ≠ 确认），同样用 sha1 证明零副作用。
10. **日志窗口类改动**：真浏览器里 `LogViewer.open('<卡片名>')` → 断言 `#lgLogPerf` 有内容、
   连续 `refresh()` 后 **DOM 子节点数只增不重建**（不会每次都换一批）、搜索过滤后
   **有匹配时不得出现"（无匹配行）"**、无匹配时才出现。
11. **重启并发/熔断**（`l_app_ready.hotreload_service`）：离线就能测——
   把锁文件写成"别的 pid"→ `spawn_self_restart()` 必须返回 False；连记 4 次 `_note_restart()`
   → `breaker()['remaining'] > 0` 且此后 `spawn_self_restart()` 仍返回 False；`state()` 里能看到 `breaker`/`restart_lock`。
   顺带离线断言模板产物：`data-fields='…'` 必须是单引号包裹、JSON 可解析、明细值里无 U+2xxx 以上的非 emoji 字形。

## 未做 / 可选

- deps 页的增删改仍是整页刷新（可复用 block 机制做局部刷新）；
- htmx 最小引入（需往包内加 `static/htmx.min.js` + 静态路由）；
- 把块基线哈希写进首屏 `data-blk-hash`（评估结论：**不必要**，服务端成本只是搬家，反而多 DOM 回写）；
- 重启并发去重（watcher 与 guard 同时触发会各写一条历史记录）。
