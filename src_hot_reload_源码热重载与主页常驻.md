# src_hot_reload 源码热重载 + 主页常驻守护（替代 uvicorn --reload）

路径：`D:\TD_Depot\Software\Lugwit_syncPlug\lugwit_insapp\trayapp\rez-package-source\Rez-Docs`

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
- 可调参数：`watch_exts` / `exclude_dirs` / `exclude_suffixes` / `interval`(1s) / `debounce`(2s) / `cooldown`(10s)

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
