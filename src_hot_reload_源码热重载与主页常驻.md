# src_hot_reload 源码热重载 + 主页常驻守护（现行机制）

状态：**现行主文档，截至 2026-09-17**。`.dev_mod` → `L_DEV_MOD=1` 的启动门控与 wuwo `ENV_MODIFIERS` 约定仍保留；早期 uvicorn `--reload` 方案仅作历史背景，见[dev_mod_热更新机制_fa50f01f.md](dev_mod_热更新机制_fa50f01f.md)。

**重要实测警示**：`l_notepad_server` 的 `L_SRC_WATCH` 对 Python 改动仍不可靠；改动 `.py` 后必须手动重启服务，不能把自动自重启视为已验证保障。

## 背景：uvicorn --reload 的三个坑（对全部后端服务通用）

早期各服务用 `.dev_mod`（→ `L_DEV_MOD=1`）让 uvicorn 开 `--reload` 做热更新
（见《dev_mod_热更新机制》）。**实践后确认该方案对"常驻 + 单实例守卫"的服务不可靠**：

1. **reload 孤儿 worker 让 `.solo` 守卫失明 → 双实例抢端口**（最致命）：
   - uvicorn `--reload` 的 worker 子进程命令行是 `python -c spawn_main --multiprocessing-fork`，**不含 app 名**；
   - `.solo` 守卫按命令行匹配 app 名（`l_app_ready.find_running`），**永远看不到这个 worker**；
   - 结果 `.solo` 放行新实例，而 reload 孤儿 worker 占着端口跑旧代码 → **两套实例抢同一端口、请求随机分流**（详见《solo_单实例守卫模式》§3 事故）。
2. **无 watchfiles 时 `StatReload` 只监听 `*.py`**：`reload_includes`/`reload_excludes` 完全无效，改模板/静态 `.html/.css/.js/.json/.svg` 不触发重载（见《dev_mod_热更新机制》§2.5）。
3. **watchfiles watcher 首次重载后可能停摆**：touch 源文件不再触发 worker 重建，"热加载看似正常实则失效"，掩盖问题直到行为明显异常。

> 结论：**凡是 `.dev_mod` + `.solo` 组合且服务真的开 `--reload` 的，都有双实例风险**。
> 主页面板已实测（8090 出现双监听 = reload 监督父+worker 或孤儿）；其他服务当前多为单进程（未真开 reload）所以暂未暴雷，但配置层面隐患相同。

---

## 更优方案：`src_hot_reload`（通用工具，落地于 `l_app_ready`）

服务改为**单实例普通进程运行**（不开 uvicorn `--reload`），热重载拆成两部分，都"即时生效"：

| 变更类型 | 处理 | 生效方式 |
|---|---|---|
| 模板 `.html`/`.j2`（A 类） | Jinja `auto_reload` | 刷新页面即变，**无需重启** |
| 代码 `.py` / 启动期配置（B 类） | `SrcHotReload` 监视线程 | 检测到变化 → 触发 `restart_cb` 自重启 |
| `__pycache__`、`.pyc`、`.git`、`node_modules`、`logs` | **排除** | 不理会（否则重启自产文件导致**重启死循环**） |

### 实现位置与用法

- 文件：`wuwo/packages/l_app_ready/1.0.0/src/l_app_ready/hotreload.py`
  - `SrcHotReload` 类 + `src_hot_reload()` 便捷函数 + 常量 `SRC_WATCH_ENV`
- 统一环境变量：**`L_SRC_WATCH`**（`"1"`=开/默认，`"0"`=关）——**所有服务共用这一个名字，不再每包一个前缀**。

**服务一行接入**：

```python
from l_app_ready.hotreload import src_hot_reload

_src_watch = src_hot_reload(
    watch_root=Path(__file__).resolve().parent,  # 源码根目录（包 src 下）
    restart_cb=restart_self,                     # 检测到代码变化时调用（服务自定自重启）
    on_toggle=lambda on: set_templates_auto_reload(on),
)
```

方法/属性（供服务对接）：
- `is_enabled()` / `set_enabled(on)` —— 开关（运行时切换，即时生效，无需重启）
- `state()` —— 返回 `{enabled, env_var, watch_root, watch_exts, restart_exts}`
- `restart_exts`（默认 `.py`）+ `extra_restart_names`（额外视为"要重启"的具体文件名，如 `services.json`）
- 可调参数：`watch_exts` / `exclude_dirs` / `exclude_suffixes` / `interval`(1s) / `debounce`(2s) / `cooldown`(10s) / `debounce_max`(60s)

> FastAPI 开关接口：**不要在工具内部用函数内局部定义的 `pydantic.BaseModel` 挂路由**
> （会被 FastAPI 当成 query 参数，报 `loc:["query","body"]`）——请在服务里用**模块级路由**
> + 手动 `await request.json()`（主页样板见下），或模型定义在模块级。
> 这是 FastAPI 对"嵌套函数内局部模型"的陷阱。

### 设计要点（踩坑沉淀）

- **线程常驻模型**：监测线程 `daemon=True` 常驻，`stop()` 只"暂停检测"（不杀线程），
  避免"开关快速 on/off 时旧线程未退出、`start()` 误判已活着而跳过重启"的竞态。
- **防抖防死循环**：
  - 检测到 `.py` 变化 → 等 `debounce` 秒确认稳定（无新变化）→ 才触发重启；
  - 触发后 `cooldown` 秒内不再次触发（罩住重启过渡期/新进程重建）；
  - `exclude_dirs` 必含 `__pycache__`（否则重启自产 `.pyc` 触发重启死循环）。
- ⚠ **"防抖窗口内第二次写入会吞掉改动"的坑（2026-09-16 修复）**：
  旧实现（`wuwo/packages/l_app_ready/1.0.0/src/l_app_ready/hotreload.py::_tick`）在
  "rescan 发现还在变"时就把快照更新成**已变状态**，于是本轮改动永久丢失 ——
  之后不再有文件事件、下一轮 diff 又是空的 → **永远不重启**（而且没有任何日志，
  表现就是"改了 .py 但服务没动静"）。
  触发条件很常见：debounce(2s) 窗口内发生第二次写入 —— 分块写文件（我们自己的
  8764 `/upload_folder` 上传器就是分块写）、落盘后改 mtime、编辑器多段保存，都会踩到。
  现在改成：**循环等到"连续 debounce 秒无新变化"再触发，期间不更新快照**，
  并用 `debounce_max`(60s) 兜底（防止文件一直在写导致永不重启）。
  回归测试：`wuwo/packages/l_app_ready/1.0.0/tests/test_hotreload_debounce.py`（7 个用例，
  含"防抖窗口内两次写入必须仍然重启"这条回归点）。
- **重启回调由服务决定**：`restart_cb` 是服务自定的自重启方式（主页用"独立进程杀旧+起新"）。

---

## 主页落地（l_homepage / homepage_cli.py）

- 主页 `start()` **去掉 uvicorn `--reload`**，改为**单实例普通进程**（uvicorn reloader 父子结构消失）。
- 主页 `_src_watch` 接入 `src_hot_reload`：
  - `watch_root = Path(__file__).resolve().parent`（包 `src/l_homepage`）
  - `restart_cb = _spawn_self_restart`（拉起独立进程执行「停旧+起新」重启，见下）
  - `on_toggle` → 设置 `templates.env.auto_reload`（**模板 auto_reload 跟随 `L_SRC_WATCH` 开关**）
- 统一开关接口（模块级定义，供前端显示/切换）：
  - `GET  /__dev__/src_watch` → 状态
  - `POST /__dev__/src_watch` `{enabled:bool}` → 切换（运行时即时生效，无需重启）
- 前端：主页卡片「♻ 热加载」按钮 = **开关 + 状态显示**（on 绿色/off 灰色），点击即时切换。
- 主页卡片默认命令改为 `wuwor l_homepage .solo -- homepage_start`（不再带 `.dev_mod`）。

**`_spawn_self_restart`（独立进程自杀式重启，避免杀自己导致响应丢失）**：
- 用 `subprocess.Popen([sys.executable, "-m", "l_homepage.homepage_cli", "restart_self_cli"], CREATE_NEW_PROCESS_GROUP|CREATE_NO_WINDOW)` 拉起独立进程；
- `restart_self_cli` 子命令 → `_restart_self()`：sleep 1s 给响应留返回窗口 → 杀 `_listener_pids(DEFAULT_PORT)` 及父 → `wuwor l_homepage .solo -- homepage_start` 重新拉起。

> ⚠️ 端口环境：主页 `package.py` 的 `commands()` 强制 `env.L_HOMEPAGE_PORT="8090"`，会**覆盖外部传入的端口**。
> 所以 `_restart_self` 经 `homepage_start` 拉起时固定用 8090；隔离测试想改端口时需注意此覆盖（否则新进程起回 8090，测不到隔离端口）。

---

## 共享封装：`l_app_ready.hotreload_service.SrcWatchService`

前 4 个服务（l_homepage / lugwit_auth / lugwit_baidu_netdisk / l_WChat）各自抄了一整套
Windows 动作（门控 / 状态文件 / 按端口停旧 / 清 `.solo` 残留 / wuwor 原样拉起 / 开关接口）。
接入第 5~10 个服务前先把它收成一处：**`wuwo/packages/l_app_ready/1.0.0/src/l_app_ready/hotreload_service.py`**。

服务侧从此只要 ~15 行：

```python
from l_app_ready.hotreload_service import PORT_ENV, SrcWatchService

_PORT = int(os.environ.get(PORT_ENV) or 8462)          # 重启执行进程靠它知道停哪个端口

_sw = SrcWatchService(
    module="l_model_hub.server",  # 重启执行入口：python -m <module> restart_self_cli
    pkg="l_model_hub",            # rez 包名（wuwor 第一个参数）
    alias="l_model_hub_server",   # 启动别名（也是 .solo 判重的别名）
    port=_PORT,
    watch_root=Path(__file__).resolve().parent,
    runtime_dir=_runtime_dir(),   # 开关状态；**不要写进 rez 包目录**
    jinja_env=templates.env,      # 有 Jinja 模板就传 → auto_reload 跟随开关
    label="l_model_hub",          # 日志前缀
)
_sw.mount(app)                    # 挂 GET/POST /__dev__/src_watch（app 建好后调）
_sw.start()                       # 起监视线程（模块级调一次）

def main():
    if _sw.handle_restart_argv(sys.argv[1:]):   # python -m xxx restart_self_cli
        return 0
    ...
    _sw.port = args.port          # 端口以实际启动参数为准
```

`SrcWatchService` 负责：`L_SRC_WATCH` 硬门控（非 `.dev_mod` 一律关，页面上也打不开）、
状态文件 `runtime_dir/src_watch.json`、`/__dev__/src_watch` 的 GET/POST、独立进程自重启
（`/F` 单杀 → 等端口释放 → 清 `.solo` 残留 → `wuwor <pkg> [.dev_mod] .solo -- <alias>`）、
可选 `owns_port` 判定（占端口者是无关进程就拒绝误杀）、`jinja_env` 跟随开关切 auto_reload。

**两个必须记住的坑（都踩过）**
1. **顺序**：`SrcHotReload.__init__` 就按 `L_SRC_WATCH` 决定初始开关，所以门控必须**先**解析并写进 env、
   **再**构造监视器。写成"先构造、`start()` 里再解析"→ 非 dev 启动也会起监视线程，硬门控失效。
2. **`mount()` 的注册方式**：FastAPI 与 FastHTML 都用 `app.get(path)(fn)` / `app.post(path)(fn)`，
   但 **FastHTML 的 `add_route(route)` 只收一个参数**（签名与 Starlette 不同），别照 Starlette 写。

**收编记录（2026-09）**：新接入的 6 个服务全部走 helper；随后 `lugwit_auth`、`lugwit_baidu_netdisk`、
`l_WChat` 也从各自的私有实现**收编到 helper**——各删掉约 200 行重复代码（门控/状态/自重启/清 `.solo`/
开关接口），只保留差异参数：auth 的 `endpoint="/api/v1/auth/src_watch"`、
网盘的 `restart_exts=(".py",".html")` + `owns_port=service_cli._is_ours` + `runtime_dir=state_dir()`、
l_WChat 的 `exclude_dirs=DEFAULT_EXCLUDE_DIRS+("wchat-android",)`。
收编后逐服务回归验证：门控、端点路径、`restart_exts`、`owns_port`、重启命令（含 `.dev_mod` 保留）与收编前**完全一致**。
只剩 `l_homepage` 保持独立实现（SSE 块刷新 + 重启历史 + 自卡分流，超出 helper 通用范围）。

---

## 接入服务清单（2026-09 全量迁移）

| 服务 | 卡/别名 | 端口 | 入口模块（重启执行进程） | 备注 |
|---|---|---|---|---|
| l_homepage | homepage_start | 8090 | l_homepage.homepage_cli | 独立实现（SSE 块刷新 + 重启历史 + 自卡分流），未用 helper |
| lugwit_auth | lugwit_auth_server | 1027 | lugwit_auth.auth_server | helper（已收编）；**endpoint `/api/v1/auth/src_watch`**（nginx 前缀归属） |
| lugwit_baidu_netdisk | baidu_netdisk_web | 1028 | lugwit_baidu_netdisk.web_server | helper（已收编）；`restart_exts` 含 `.html`（页面是 import 期烘死常量）；`owns_port=service_cli._is_ours`；endpoint `/api/src_watch` |
| l_WChat | l_wchat_backend | 1234 | l_WChat.app（别名改 `python -m l_WChat.app`） | helper（已收编）；`exclude_dirs` 加 `wchat-android` |
| l_model_hub | l_model_hub_server | 8462 | l_model_hub.server | helper；`keys.py` 写包内 `config.json`，**别进 restart_exts** |
| l_mindmap（目录 `l_mindmap_fasthtml`） | l_mindmap_server | 8100 | l_mindmap.server | helper；app 是 **FastHTML**；页面头部是 Python 常量 → 只监视 .py |
| l_mindmap_mmd | l_mindmap_mmd_server | 8110 | l_mindmap_mmd.server | helper；Jinja 模板（auto_reload） |
| l_notepad_server | l_notepad_api | 8765 | l_notepad_server.backend_server | helper；app 在 `create_app()` 里建 → 在那里 `mount`；数据在用户目录 |
| l_agent_chat | l_agent_chat | 1250 | l_agent_chat.app | helper；launcher 原来**硬编码 `reload=True`**，已改单进程 |
| ChatRoom 后端 | chatroom_backend | 1026 | app.main（cwd=backend，与 .bat 一致） | helper；app 在 `__main__` 里建 → 在那里 `mount`；`exclude_dirs` 加 `logs/Log/static/data/temp/test`（运行期往包内写日志与缓存） |

- **ChatRoom 前端（chatroom_frontend, 1025）不在此列**：它是 Vite dev server（Node），热更新靠 Vite HMR，
  与 Python 进程内热重载无关。
- 各服务开关接口路径按 nginx 归属选：`/__dev__/src_watch`（l_WChat 经 `/l_wchat/` 剥前缀）、
  `/api/src_watch`（网盘）、`/api/v1/auth/src_watch`（auth）、`/__dev__/src_watch`（其余直连即可）。
- **`L_SRC_WATCH_PORT` env**：helper 用它把"该停哪个端口"传给重启执行进程（执行进程没走 `--port` 解析）。
- **别再留 `--reload` 别名/参数**：本轮把 `l_notepad_api_reload`、`l_mindmap_dev` 里失效的
  `--reload/--reload-dir` 都改成了普通启动（保留别名只为兼容主页卡片的「♻ 热更新」按钮）。

---

## 接入服务的启动门控：**必须带 `.dev_mod` 才有热重载**（auth / netdisk / l_WChat / 其余全部）

主页的规则是"`L_SRC_WATCH` 默认 1"（不带 `.dev_mod` 也热重载）。**auth 与 netdisk 反过来做了硬门控**，
理由：热重载是开发态能力，生产/日常启动不该带一台监视线程；且它让主页卡片的
「🔥 热启动」与 `hot` 图标（读进程 `L_DEV_MOD`，`homepage_cli.py:612`）重新自洽。

```python
def _dev_mode() -> bool:                     # 是否带 .dev_mod 启动
    return os.environ.get("L_DEV_MOD") == "1"

def _resolve_src_watch(use_env: bool) -> str:
    if not _dev_mode():
        return "0"                           # 硬门控：非 dev 一律关，显式 env 也不放行
    if use_env:
        cur = os.environ.get("L_SRC_WATCH")   # dev 模式内：显式 env > 存档 > 默认开
        if cur in ("0", "1"):
            return cur
    saved = _load_saved_src_watch()
    return "1" if (saved is None or saved) else "0"
```

- `use_env=False` 供**重启执行进程**用：它自己的 env 里是 `L_SRC_WATCH=0`（故意，免得过渡期再触发一次），
  不能拿它当用户意图。
- `POST /…/src_watch` 在非 dev 模式**拒绝打开**（`want=True` 被压回 `False`），
  响应额外带 `dev_mod` 与 `note` 说明原因。
- 自重启**保留启动模式**：dev 起的，新实例命令仍带 `.dev_mod`（`pkgs = [pkg] + (['.dev_mod'] if _dev_mode() else []) + ['.solo']`），
  否则自重启一次就把热重载门控关死了。
- 手动验证：`wuwor lugwit_auth .solo -- lugwit_auth_server` → 静态；加 `.dev_mod` → 热重载默认开。

---

## 第二个接入服务：lugwit_auth（落地于 auth_server.py）

改动文件：`lugwit_auth/999.0/src/lugwit_auth/auth_server.py`

- **去掉 uvicorn `--reload`**：删掉 `--reload/--no-reload` 参数、`reload_dirs`、
  `reload_includes`、`LUGWIT_AUTH_RELOAD` / `LUGWIT_AUTH_RELOAD_DIR` 判断
  （`.dev_mod` / `L_DEV_MOD` **保留**，升级为热重载硬门控，见上节）；
  `uvicorn.run(app, ...)` 单实例普通进程。
- **接入 `src_hot_reload`**（模块级）：
  - `watch_root = Path(__file__).resolve().parent`
  - `restart_cb = _spawn_self_restart("src-watch")`，`on_toggle = _on_src_watch_toggle`
  - `extra_restart_names=(".env",)`（启动期配置，改了要重启才生效）
  - `templates.env.auto_reload = _src_watch.is_enabled()`（A 类模板改动）
- **开关接口 `/api/v1/auth/src_watch`（GET/POST）**：**不是**主页的 `/__dev__/src_watch`
  —— nginx `location /` 归主页(8090)，非 `/api/` 路径到不了 auth；只有
  `location /api/v1/auth` 才转 auth(1027)。新服务接入时按 nginx 的归属前缀选路径。
- **自重启链路**：`_spawn_self_restart()` 拉起独立 `python -m lugwit_auth.auth_server
  restart_self_cli`（新建进程组 + `L_SRC_WATCH=0`）→ `_restart_self()`：sleep 1s →
  按 PID **单杀**（`/F`，不带 `/T`，否则会连带杀掉承载拉起的执行进程自己）→
  等端口释放（≤5s）→ `_kill_stale_solo_wrappers()`（清 `.solo` 残留包装进程，
  否则新实例被判"已有实例"静默退出）→ `wuwor lugwit_auth .solo -- lugwit_auth_server`。
- **运行期状态**：`~/.lugwit/lugwit_auth/runtime/src_watch.json`（可用
  `LUGWIT_AUTH_RUNTIME` 覆盖）；存档只在 `.dev_mod` 启动时生效（见上节门控）。
- **auth 侧"热更新其他服务"接口**（`POST /api/v1/services/{name}/reload`）不再追加
  `.dev_mod`，改为原样停旧起新（热重载归各服务进程自己管）。
- **主页卡片**：`l_homepage/config/services_builtin.json` 里两张 auth 卡的
  `.dev_mod` **保留**（它就是热重载开关），用户 runtime 里的同名覆盖若被改成不带
  `.dev_mod`，点「覆盖为系统设置」即可恢复。
- **共享工具修的一处坑**：`hotreload._keep()` 只按 `Path.suffix` 比对，点文件
  （`.env`）的 suffix 是空串 → `DEFAULT_WATCH_EXTS` 里的 `.env` 是**死配置**。
  现改为"扩展名或完整文件名"双向匹配，`_is_restart()` 同样支持按名匹配。

尚未做（auth 侧）：外部 `guard` 常驻守护（服务崩了没人拉）、`on_frontend_change`
→ SSE（无块哈希局部刷新机制，模板改动靠 `auto_reload` + 手动刷新）。

---

## 第三个接入服务：lugwit_baidu_netdisk（落地于 web_server.py）

改动文件：`lugwit_baidu_netdisk/999.0/src/lugwit_baidu_netdisk/web_server.py`

- **去掉 uvicorn `--reload`**：删 `--reload/--no-reload`、`reload_dirs`、`reload_excludes`、
  `LUGWIT_NETDISK_RELOAD` 判断（`.dev_mod` / `L_DEV_MOD` 保留为硬门控）；
  `uvicorn.run(app, ...)` 单实例普通进程（启动前的 `ensure_port_free` 自清障保留）。
- **`restart_exts=(".py", ".html")`**：本服务的页面不是 Jinja 模板，而是 import 期
  `_HTML = read_text()` 烘进来的常量 → **改 `.html` 也必须重启**（与主页/lugwit_auth
  的 auto_reload 语义不同，别照抄）。
- **开关接口 `/api/src_watch`（GET/POST）**：按 nginx 归属选前缀 —— `location /api/`
  归网盘(1028)，`/api/v1/` 归 auth，`/` 归主页。
- **自重启链**：同 lugwit_auth（独立进程 `restart_self_cli` + `/F` 单杀不带 `/T` +
  等端口释放 + 清 `.solo` 残留包装 + `wuwor lugwit_baidu_netdisk .solo -- baidu_netdisk_web`），
  但**复用 `service_cli._is_ours()`** 复用其"占端口者是否为本服务旧实例"判定：
  无关进程占用 → 拒绝误杀并放弃本次重启（不冒双实例风险）。
  注意 `service_cli._kill_tree()`/`ensure_port_free()` 用的是 `/T`，**不能在重启执行进程里用**
  （执行进程本身是旧实例的子进程，会被一起清掉）。
- **状态文件**：复用 `state_dir()` → `~/.lugwit/baidu_netdisk/src_watch.json`。
- **端口 env**：新增 `LUGWIT_NETDISK_PORT`（默认 1028），供重启执行进程知道该停哪个端口；
  `--port` 缺省值也读它。
- **主页卡片**：`l_homepage/config/services_builtin.json` 的网盘卡 `.dev_mod` **保留**（热重载门控）。
- **已知交互**：`lugwit_auth` 的"进程内挂载网盘"路径（`_mount_baidu_netdisk()`）会
  `import lugwit_baidu_netdisk.web_server` —— 一旦真挂上，auth 进程里也会起一份本监视线程。
  现状：auth 已不 requires netdisk，挂载通常失败（仅告警），故不构成实际问题。

尚未做（netdisk 侧）：外部 `guard` 常驻守护、SSE（页面无块哈希局部刷新机制）。

---

## 第四个接入服务：l_WChat（落地于 app.py）

改动文件：`l_WChat/999.0/src/l_WChat/app.py` + `package.py`

- **原本就没开 uvicorn `--reload`**（`package.py` 注释里写着"reload 首次重载后停摆 + 孤儿 worker
  占端口"），改 `.py` 靠手工跑 `999.0/dev_restart.bat`。现在这套手工动作由进程内 `SrcHotReload` 自动完成。
- **入口收敛**：别名 `l_wchat_backend` 从 `python -m uvicorn l_WChat.app:app --host 0.0.0.0 --port 1234`
  改成 **`python -m l_WChat.app`** —— 单进程 `uvicorn.run(app, port=_PORT)`，且 `.solo` 注册
  （`match_pattern="l_WChat.app"`）、端口自检、热重载、自重启入口全在模块里，只有一处。
- **硬门控同 auth/netdisk**：必须带 `.dev_mod`；自重启保留启动模式（带 `.dev_mod` 起的仍带）。
- **端口**：新增 `L_WCHAT_PORT`（默认 1234），模块级端口自检也从写死 1234 改成读 `_PORT`。
- **开关接口 `/__dev__/src_watch`**：nginx 里 `location /l_wchat/` → 1234 且**剥掉前缀**，
  所以对外是 `/l_wchat/__dev__/src_watch`，直连 1234 就是 `/__dev__/src_watch`。
  （**不能像网盘那样用 `/api/`**：nginx 的 `location /api/` 归百度网盘 1028。）
- **监视范围**：`watch_root = src/l_WChat`，但 `exclude_dirs=DEFAULT_EXCLUDE_DIRS + ("wchat-android",)`
  —— `wchat-android/` 是安卓壳工程（`.html/.json/.js` 一大堆），不排除会让 Gradle/前端改动触发无关重启。
- **踩点提醒**：`config.json` 就在 `watch_root` 里、且**是服务自己运行时写的**（`paths.USER_CONFIG_PATH`），
  所以它只能"被监视"，**绝不能进 `restart_exts`/`extra_restart_names`**，否则写配置 → 重启 → 再写 = 死循环。
  （实测它落在"非重启类改动"里；当前没传 `on_frontend_change`，等价于忽略。）
- **托盘路径未带 `.dev_mod`**：`l_tray/Tray.py` 用 `["l_WChat", ".solo", ".ps"]` 启动 → 静态运行；
  要热重载走主页卡片（卡片 `run_cmd` 带 `.dev_mod`）或 `999.0/dev_restart.bat`（也是 `.dev_mod .solo`）。

尚未做（l_WChat 侧）：外部 `guard` 常驻守护（主页 watchdog + 卡片 `auto_start` 即可，无需自己的 guard）。

---

## 重启的并发防护与熔断（防"无限重启 + 日志刷爆"）

热重载的重启是「独立执行进程 → 停旧 → 拉起新实例」，**每次都是新进程**，所以进程内标志完全不够用。
没有防护时有三路可以同时动手：热重载监视线程、主页 watchdog（`auto_start` 掉线自拉）、用户点「重启/热更新」。
两路同时"停旧 + 起新"会各起一个实例（`.solo` 判重有竞态窗口）→ 双实例 / 反复重启；
若根因是"改坏的代码起不来"，就变成无限重启，把服务日志刷爆（前端日志窗口随之卡死）。

`SrcWatchService` 里的两道闸（均在 `l_app_ready/hotreload_service.py`）：

| 机制 | 位置 | 行为 |
|---|---|---|
| **并发锁** | `%TEMP%/lugwit_hotreload/<alias>.lock`（JSON：pid/at/trigger/pkg） | `spawn_self_restart()` 先查锁，被**别的 pid** 持有时直接跳过；执行进程 `restart_self()` 允许**父进程/自身**持锁（原实现只认自身 pid → 重启执行进程永远让位、**静默不重启**：lock 出现、breaker 计数涨、PID 不变），出口 `finally` 释放锁。TTL 30s（进程崩了也不会卡住后续重启） |
| **熔断** | `<runtime>/restart_guard.json`（events / blocked_until / backoff） | 180s 窗口内重启 ≥4 次 → 封 60s，之后按次**翻倍**（上限 15min）。熔断期间 `spawn_self_restart` 返回 False 且**限频**打日志（30s 一条，不再用日志刷爆日志） |

对外：`restart_in_progress(alias, ttl)` 供外部动作避让；`state()` 带 `breaker`（recent/blocked_until/remaining）
与 `restart_lock`，前端可提示"已熔断 / 正在重启"。

**`l_app_ready.is_restart_exec()`（服务启动期自检必须放行"重启执行进程"，2026-09-16/17）**：
服务的启动期自检（"端口被占用就 `sys.exit`"）必须放行"重启执行进程"——它是**故意**在旧实例仍占用
端口时启动的。未放行的后果同上（**静默不重启**）：import 阶段退出、`DEVNULL` 下无任何日志。
`l_WChat/app.py` 已改；`lugwit_auth` / `lugwit_baidu_netdisk` 等待办。
相关参数 `debounce_max`(60s)（见上节）；回归测试：
`wuwo/packages/l_app_ready/1.0.0/tests/test_hotreload_debounce.py`、`wuwo/packages/l_app_ready/1.0.0/tests/test_restart_lock.py`。

**主页侧的配合**（`l_homepage/homepage_cli.py`）：
- `_watchdog_tick()`：`_restart_in_progress(svc)` 为真时**跳过**（原来会在热重载过渡期把服务再拉一个起来）；
- `_svc_manage_impl()`：`start` 撞上锁直接返回 `skipped: restart-in-progress`；`restart/reload/hotstart`
  先 `_wait_restart_lock_free()`（≤8s）再动手。
- **检查周期（2026-09-20 调整）**：`WATCHDOG_INTERVAL` 默认 **60s**（旧 1800s，`L_HOMEPAGE_WATCHDOG_INTERVAL` 可覆盖）。
  一轮只是对每张卡的端口做一次 TCP connect（毫秒级），**只在掉线时**才走
  `_svc_manage(name, "start", wait_ready=WATCHDOG_WAIT_READY=20s)`；旧周期下"掉线后最长半小时才被拉回"，
  体感等于没开常驻。周期值同时驱动 UI 文案：`_watchdog_interval_text()` 注入 `_home_context()` 与 deps 上下文，
  卡片 ♻ title / 常驻图例 / 编辑表单共用（以前四处写死"每 30 分钟"，改周期必漏改页面）。
- **失败退避（同一轮调整）**：周期短了就必须防硬重试 —— `WATCHDOG_BACKOFF = (30, 60, 120, 300)` 秒，
  每卡片内存态 `_watchdog_fail[name] = {fails, next_at, error, at}`；拉起成功或端口恢复即清零；
  退避窗口内的轮次记 `skipped: "backoff"` + `fails` + `retry_in` 且**不再动手**（否则一张长期起不来的卡
  会每分钟起一条 wuwor/rez 链并等 20s，守护线程几乎一直在忙、`watchdog.log` 被刷爆）。
  退避状态在 `/api/v1/watchdog/status → last_check.checked[]` 可查（内存态，主页重启即清零）。
- **就绪超时按卡片自适应（2026-09-20）**：`_ready_timeout_for(svc)` = 该卡最近成功记录中最慢耗时 ×1.5，
  夹在 `[WATCHDOG_WAIT_READY(20s), READY_TIMEOUT_MAX(120s)]`，缓存 60s；历史读 `restart_history.jsonl`
  （跨重启保留，无记录退化 20s）。原因：固定 20s 会把"起得来但慢"判成失败 —— 实测 `l_model_hub` 冷启动
  **47s**，自适应后为 70.5s。tick 记录带 `ready_timeout` 便于回看。
- **手动启停的短窗（同日）**：手动 start/restart 不等就绪（避免 UI 卡 47s），但用
  `_wait_instant_exit()`（`MANUAL_START_GRACE=4s`）兜住"子进程立刻死"——只判"进程已退出且端口没起来"，
  命中就把日志尾部一起回给前端（无输出时明说"进程在加载阶段就死了"）。见《l_homepage》踩坑第 22 条
  （实例级 spawn 失能 `0xC0000142` 就是这么被糊成 `ok:true` 的）。

---

## 主页常驻守护：必须用外部 guard 进程

**通用规律：服务无法自守护自己。** 主页的 watchdog（常驻守护）线程跑在主页进程内，
主页崩溃/被杀后 watchdog 线程也随之消失，`_watchdog_tick` 无法执行自拉。

所以主页常驻**必须由一个独立于主页进程树的外围进程**负责：

- `l_homepage.homepage_cli guard` 子命令：独立常驻进程（`CREATE_NEW_PROCESS_GROUP`，不随主页死），
  循环探测端口，主页掉线 → `_restart_self()` 拉起主页；
  `guard` 进程内设 `L_SRC_WATCH=0`（只做端口守护，不做源码热重载）。
- `start()` 后台模式下，若主页卡片开启常驻（`auto_start`），自动 `_ensure_guard(port)` 拉起 guard；
  guard 记录 pid 到 `~/.lugwit/l_homepage/runtime/.homepage.guard.pid`，已存在则复用。
- 主页卡片 `auto_start` 缺省视作 `True`（`_normalize` 对 `homepage_start` 默认常驻），可切换；
  guard 循环里若主页卡片 `auto_start` 变为 `False` 则守护自行退出。

---

## FastAPI 陷阱（本次踩坑）

**函数内局部定义的 `pydantic.BaseModel` 会被 FastAPI 当成 query 参数**：
- 报错：`{"detail":[{"type":"missing","loc":["query","body"],"msg":"Field required"}]}`
- 原因：FastAPI 用类型注解识别 body 参数，嵌套函数内局部定义的模型无法被正确推断 → 退化为 query。
- 解法：模型定义在**模块级**；或路由里手动 `data = await request.json()`（推荐，简单可靠）。

---

## 关键文件

- `wuwo/packages/l_app_ready/1.0.0/src/l_app_ready/hotreload.py`（`src_hot_reload` 工具）
- `wuwo/packages/l_app_ready/1.0.0/src/l_app_ready/__init__.py`（导出 `src_hot_reload`/`SrcHotReload`/`SRC_WATCH_ENV`）
- `l_homepage/999.0/src/l_homepage/homepage_cli.py`（主页接入：`_src_watch`、`/__dev__/src_watch`、`guard`、`_spawn_self_restart`、`_ensure_guard`）
- `l_homepage/999.0/src/l_homepage/templates/home.html`（「♻ 热加载」「♻ 常驻」按钮 + 命令显示）

## 测试要点（主页隔离实例实测）

- 主页**单实例**运行（去 `--reload` 后 8090 单监听，无 reloader 父子双监听）。
- 改 `.py` → `src_watch` 监测 → **真实自重启**（监听 PID 变化，如 82008 → 76684）。
- 改 `.html` → 刷新即变（`auto_reload`，不重启进程）。
- 开关 on/off 切换正常，`auto_reload` 跟随；快速 on/off 无竞态。
- `guard` 进程拉起并存活；主页被杀后由 guard 拉回（真实环境按 8090）。
