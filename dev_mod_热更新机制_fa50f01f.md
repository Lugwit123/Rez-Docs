# dev_mod 特殊包名 + 服务热更新机制

## 背景与目标

现状：auth 导航首页卡片（`SERVICES` 注册表）经 `/api/v1/services/{name}/start|restart|stop` 用 `_run_wuwor(packages, run_args)` 按别名启动各服务；热更新能力参差不齐——auth 有 `--reload`（默认开）、note 后端有 `--reload`（默认关）、**netdisk 无**、**ChatRoom 后端无 CLI 参数**（bat 启动=生产模式硬关）。

目标：wuwo 识别特殊包名 `.dev_mod`（照 `.script_server` 的 `ENV_MODIFIERS` 机制），注入 `L_DEV_MOD=1`；各后端服务识别该变量启用 uvicorn 热更新（改代码自动重启生效）。热更新变为**按需启用**：启动请求带 `.dev_mod` 才开，生产默认零开销。

用法示例：`wuwor lugwit_baidu_netdisk .dev_mod -- baidu_netdisk_web`；卡片「♻ 热更新」按钮 = 停进程树 + 重启（优先用卡片配置的专用热更新别名 `reload_args`，缺省用 `run_args`），始终带 `.dev_mod`。

## 1. wuwo 注册 .dev_mod 环境变量修饰符

文件：`d:\TD_Depot\Software\Lugwit_syncPlug\lugwit_insapp\trayapp\wuwo\py_modules\wuwo_rez.py`

- `ENV_MODIFIERS`（L62-67）新增一项（注释照 `.script_server` 风格）：
  ```python
  # 特殊虚拟包 .dev_mod：识别后不注入 rez 环境，仅设置环境变量，
  # 各后端服务（auth/netdisk/chat/note）识别该变量启用 uvicorn 热更新
  ".dev_mod": {"L_DEV_MOD": "1"},
  ```
- 机制已验证：`_parse_terminal_virtuals`（L180-207）命中即 `os.environ[k]=v` 并从 clean_requests 剥离；`ALL_VIRTUAL_MODIFIERS`（L69）自动包含；cmd_env 在当前进程内执行目标命令，子进程自动继承。
- 可选：`wuwo.bat` 帮助文本（L85/L122-130 附近）补 `.dev_mod` 一行。**注意该 bat 为 GBK 编码**，若改须保持 GBK + CRLF（参照记忆「编辑 GBK bat 后须修复编码与行尾」）；不改也不影响功能。

## 2. 四个后端服务识别 L_DEV_MOD 启用热更新

统一模式：`--reload` / `--no-reload` 双向 CLI 参数 > 服务专属环境变量 > `L_DEV_MOD`；uvicorn `reload=True` 时配 `reload_dirs`（只监控自家 src）+ `reload_excludes`（logs/temp/test）。

### 2.1 lugwit_auth（1027）
文件：`lugwit_auth/999.0/src/lugwit_auth/auth_server.py`（L1118-1122）
- `--reload` 默认值改为：`os.environ.get("LUGWIT_AUTH_RELOAD", "1" if os.environ.get("L_DEV_MOD") == "1" else "0") == "1"`
- **行为变化**：非 dev 模式下 auth 默认不再热重载（生产更安全）；开发用 `.dev_mod` 或 `LUGWIT_AUTH_RELOAD=1` 恢复。需在注释中说明。

### 2.2 lugwit_baidu_netdisk（1028）
文件：`lugwit_baidu_netdisk/999.0/src/lugwit_baidu_netdisk/web_server.py`（`main()` 在文件末尾 `if __name__ == "__main__"` 之前）
- 加 `--reload`/`--no-reload`（dest=reload，default=None）；生效值 = CLI > `LUGWIT_NETDISK_RELOAD` > `L_DEV_MOD`
- `uvicorn.run(..., reload=on, reload_dirs=[<包 src 目录>] if on else None)`
- **模块级**加 Windows Selector 事件循环策略（asyncpg 与 Proactor 不兼容；reload 子进程不执行 main()，必须 import 时生效——照 auth_server.py L46-51 的写法与注释）
- **启动自清障（2026-09 新增）**：`main()` 在 `uvicorn.run` 前调 `service_cli.ensure_port_free(port)`——端口被**本包旧实例**占用（含 reload 父链）先整树清掉再绑定；占端口的是**无关进程**（如别的机器上 cer_service.exe 恰好占 1028）则拒绝误杀、明确报错并以退出码 2 终止（此时绑定也必然失败，报 WinError 10013/10048）。安全边界：`_kill_tree` 只杀自身+后代（taskkill /T），**绝不沿祖先链追杀**——旧实例常是主页 spawn 的，追杀祖先会误伤主页进程树（已踩坑）；判定"自己人"用 cmdline 含 `lugwit_baidu_netdisk`/`web_server`。
- **反代前缀改 middleware 自剥（2026-09 新增）**：不再把 `LUGWIT_NETDISK_PREFIX` 传给 uvicorn `root_path`——uvicorn ≥0.38 会把 root_path **prepend 进 scope.path**（starlette 再剥一次）：经 nginx（剥前缀转发）恰好抵消没问题，但浏览器**直连**本端口时页面 `__API_BASE__="/baidu"` 拼出的 `/baidu/api/*` 会被再 prepend 成 `/baidu/baidu/*` → 404。改为：`_strip_proxy_prefix` middleware 自行剥离带前缀 URL（兼容直连），页面 `__API_BASE__` 由 `_page_base()` 渲染（auth 挂载场景取 scope root_path，standalone 取 env）；登录跳转 `next=` 只在路径被代理剥掉前缀时补回。

### 2.3 ChatRoom 后端（1026）
文件：`ChatRoom/999.0/src/ChatRoom/backend/app/main.py`（L507-560）+ `run_backend_server.bat`（L58）
- `__main__` 里用 `argparse.parse_known_args` 加 `--reload`/`--no-reload`；默认 = CLI > `CHATROOM_RELOAD` > `L_DEV_MOD` > 原 `is_debug_user and not from_bat`
- 三个 uvicorn.run 分支合并为一次调用：`reload=on`，on 时带 `reload_dirs=["app"]` + 现有 `reload_excludes` 列表（L545-557 保留）
- bat L58 改 `python -m app.main %*`（透传参数；`pause` 保留，DETACHED/DEVNULL stdin 下读 EOF 即过）

### 2.4 l_notepad_server 后端（8765）
文件：`l_notepad_server/999.0/src/l_notepad_server/backend_server.py`（L744-753）
- `--reload` 默认值扩展为同时识别 `L_DEV_MOD`（`L_NOTEPAD_RELOAD` 仍优先）
- `local_main.py`（l_notepad_client）**无需改**：`_start_backend_subprocess`（L1736）`env = os.environ.copy()`，卡片以 `l_notepad_server .dev_mod` 启动时内嵌后端自动继承 `L_DEV_MOD`
- **模板热更兜底（2026-09 新增）**：`.dev_mod`（`L_DEV_MOD=1`）下 `templates.env.auto_reload = True`——改 `.html` 后刷新页面即生效，无需进程重启（缓解 §2.5 所述无 watchfiles 时 StatReload 不监听模板的问题）
- **全文件类型 reload 配置（2026-09 新增）**：`uvicorn.run` 配 `reload_includes=["*.py","*.html","*.css","*.js","*.json","*.svg","*.mmd"]` + `reload_excludes`（logs/temp/test/pycache/隐藏文件）——**装了 watchfiles 后**即可对全部前端文件改动触发完整进程重载

## 2.5 ⚠️ 通用注意：uvicorn --reload 默认只监听 *.py（无 watchfiles 时退化为 StatReload）

实测（2026-09-03，l_notepad_server 8765）：运行环境（`rez-package-3rd` 的 python 3.12.10）**未安装 `watchfiles`**，此时 uvicorn 的 `--reload` 退化为 `StatReload`（`uvicorn/supervisors/statreload.py`）：
- 只 `rglob("*.py")`（`statreload.py:49-51`），且 `__init__` 明确 `if reload_excludes or reload_includes: warning("... have no effect unless watchfiles is installed")`（`statreload.py:25-26`）——**`reload_includes`/`reload_excludes` 在无 watchfiles 时完全无效**。
- 后果：改 `.py` → 自动重启 ✓；改模板/静态（`.html/.css/.js/.json/.svg`）→ **不触发重载** ✗。

**对"所有文件改动都热更"的两条路**（适用所有基于 uvicorn --reload 的 rez 后代服务）：
1. **首选：运行环境装 `watchfiles`**——uvicorn 改用原生 watchfiles 监听，`reload_includes` 即生效，任意前端文件改动触发完整进程重载（重启后模板缓存也一并刷新）。安装属 rez 部署级改动（pip 装入运行时 python），本机尚未执行。
2. **不装依赖的兜底（已用于 l_notepad_server §2.4）**：Jinja 模板开 `auto_reload`（改 `.html` 刷新即生效、无需重启）；`.py` 仍由 uvicorn 自动重启；静态文件由 `StaticFiles` 每次读盘（浏览器刷新即可见，注意浏览器缓存需强刷）。

## 3. 卡片 reload 端点 + 首页「热更新」按钮

> **2026-09 更新**：主页已从 `lugwit_auth` 拆分为独立包 `l_homepage`
> （`l_homepage/999.0/src/l_homepage/homepage_cli.py` + `templates/home.html`，
> `:8090`，nginx `/homepage` 反代）。本节所述端点与按钮现都在 `l_homepage`，
> 卡片注册表 `SERVICES` 运行时由 `~/.lugwit/l_homepage/runtime/services.json`
> 覆盖（`DEFAULT_SERVICES` 仅兜底），改默认卡片需同步运行时文件。

- `POST /api/v1/services/{name}/reload`（管理员）：
  1. **先按端口杀旧进程树**（`_kill_port_listeners` → 优先 `l_agent_tool.kill_port`，含 uvicorn reloader 父子进程）——kill-first 适用于 **start/restart/reload 全部三个操作**，否则新进程绑不上端口（WinError 10013/10048），表现为「重启后 PID 不变」
  2. 重启命令 = `wuwor <packages> .dev_mod -- <reload_args 或 run_args>`；卡片可配**专用热更新别名**（`reload_args`/`reload_cmd`，如 l_notepad_server 的 `l_notepad_api_reload`），无则用 `run_args`
  3. 返回结构含 `alias`（实际用的别名）与日志路径（`{WUWO_LOG_DIR}/{pkg}/{pkg}_{alias}_{date}.log`）
- home.html：`↻ 重启` 旁「♻ 热更新」按钮；重启/热更新期间状态组件有 busy 动画（`paintBusy`/`paintFlash`/`_transition` 状态机，退出条件 `sv.up && sv.pid`）
- deps 图页（`/homepage/deps`）：全画布节点编辑器，3s 轮询 `hot`/`pid` 状态，热更新开启时节点命令显示含 `.dev_mod` 的完整命令

## 4. 可选：stack bat 支持 dev 模式

文件：`l_nginx/999.0/l_lugwit_stack.bat`
- `start` 子命令接受第二参数 `dev`：`if /i "%~2"=="dev" set "DEVMOD=.dev_mod"`，auth/netdisk 的 wuwor 请求里追加 `%DEVMOD%`。

## 依赖关系

- 2.x 各服务改动相互独立，可并行；都依赖 1（wuwo 注入变量）才能端到端验证
- 3 依赖 1（reload 端点靠 `.dev_mod` 传参）
- 4 依赖 1

## 测试计划

1. 变量注入：`wuwor l_notepad_server .dev_mod -- python -c "import os;print(os.environ.get('L_DEV_MOD'))"` → `1`；不带 `.dev_mod` → `None`；rez 解析日志 clean_requests 不含 `.dev_mod`
2. netdisk：`wuwor lugwit_baidu_netdisk .dev_mod -- baidu_netdisk_web` → 日志出现 `Started reloader process`；改一行 netdisk 源码 → 自动重启；不带 `.dev_mod` → 单进程无 reloader
3. auth：带 `.dev_mod` 启动有 reloader；不带 → 单进程（验证默认翻转）；`LUGWIT_AUTH_RELOAD=1` 仍可强制开
4. chat：`wuwor ChatRoom .dev_mod -- chatroom_backend` → reload 生效（bat %* 透传验证）
5. note：`wuwor l_notepad_server .dev_mod -- l_notepad_server` → 内嵌 8765 后端带 reload（tasklist 见 reloader 父子进程）
6. 卡片：管理员登录首页 →「♻ 热更新」→ 服务以 `.dev_mod` 重启（status/日志确认）；非管理员 403
7. 回归：`l_lugwit_stack.bat start/stop/status` 不受影响；`.solo`/`.script_server` 既有修饰符行为不变

## 风险与缓解

- **auth 默认翻转**（reload 开→关）：依赖自动重载的开发者需加 `.dev_mod`；缓解=代码注释说明 + 专属 env 兜底
- **Windows + asyncpg + reload**：事件循环策略必须模块级设置（auth 已有先例注释），netdisk 照抄
- **reload 监控范围**：reload_dirs 只指自家 src + excludes 日志/临时文件，避免文件风暴反复重启
- **watchfiles 缺失 → reload 只监听 *.py**（§2.5）：模板/静态改动不触发重载；缓解=运行环境装 watchfiles（启用 `reload_includes`，对全部文件热更）或 Jinja `auto_reload`（模板刷新即生效）
- **wuwo.bat GBK**：帮助文本可改可不改；改则保 GBK+CRLF
- **ChatRoom bat %***：用 parse_known_args 不破坏 FROM_BAT 既有逻辑

## 已否决的替代方案

- **每个服务只加 `--reload` CLI、不走环境变量**：卡片/stack 经别名启动没有顺手的参数通道，`.dev_mod` 在所有启动路径（卡片 API、stack、手动 wuwor）统一生效
- **每包新增 `<svc>_reload` 别名**：~~已否决~~ **后被部分推翻（2026-09）**——当热更新入口与普通启动不同（如 l_notepad_server reload 走 `l_notepad_api_reload` 别名）时，别名是必要的；卡片新增 `reload_args`/`reload_cmd` 字段支持，缺省仍用 `run_args`。卡片「热更新」按钮 tooltip 会显示对应的完整 `wuwor` 命令
- **新建共享服务管理 Python 包**：跨包依赖改动重；环境变量方案零依赖、各服务两行代码接入

## 关键文件

- `wuwo/py_modules/wuwo_rez.py`（ENV_MODIFIERS，L62-67）
- `lugwit_auth/999.0/src/lugwit_auth/auth_server.py`（reload 默认 L1118）
- `lugwit_baidu_netdisk/999.0/src/lugwit_baidu_netdisk/web_server.py`（main L1704）
- `lugwit_baidu_netdisk/999.0/src/lugwit_baidu_netdisk/service_cli.py`（`ensure_port_free`/`_kill_tree`/`_is_ours`）
- `ChatRoom/999.0/src/ChatRoom/backend/app/main.py`（L507-560）+ `run_backend_server.bat`（L58）
- `l_notepad_server/999.0/src/l_notepad_server/backend_server.py`（L744；`reload_includes` 全前端类型 + `L_DEV_MOD` 下 Jinja `auto_reload`，见 §2.4/§2.5）
- `rez-package-3rd/uvicorn/0.52.4/platform-windows/.../uvicorn/supervisors/statreload.py`（无 watchfiles 时只 `rglob("*.py")` 且忽略 `reload_includes`）
- `l_homepage/999.0/src/l_homepage/homepage_cli.py`（卡片注册表/管理端点；主页已从 lugwit_auth 拆出）
- `l_homepage/999.0/src/l_homepage/templates/home.html`（卡片 UI）

## 后续演进（2026-09，fa50f01f 之后）

1. **主页拆分独立包 `l_homepage`**：卡片注册表、start/restart/reload/stop/restart-all 端点、deps 依赖图全部迁到 `l_homepage`（:8090，nginx `/homepage` 前缀）。运行时状态（`services.json`、`deps_layout.json`、日志、pid 文件）放 `~/.lugwit/l_homepage/runtime/`，**不写进 rez 包目录**；`services.json` 覆盖 `DEFAULT_SERVICES`，改默认卡片时须同步运行时文件。
2. **热更新状态实时显示**：主页状态 API 每服务返回 `hot`——用 ctypes 读目标进程 PEB 环境块（跨进程查 `L_DEV_MOD=1`，比 cmdline/父链启发可靠，因 uvicorn ≥0.30 worker 与 reloader argv 相同）。UI：`hot=true` 时卡片「♻ 热更新」按钮绿色高亮（🔥）且启动命令显示含 `.dev_mod` 版本；`hot=false` 不高亮、命令显示无 `.dev_mod` 版本。deps 图节点同步。注意：进程必须经 `.dev_mod` 启动才有 `hot=true`；重启主页自身要用 `wuwor l_homepage .dev_mod -- homepage_start`。
3. **kill-first 扩展到 start**：`_svc_manage` 的 start/restart/reload 三操作统一先清端口监听者（`l_agent_tool.kill_port` 优先，回退 GBK 解码 netstat + `taskkill /F /T`——中文 Windows netstat 输出是 GBK，按 UTF-8 解码会崩）。
4. **netdisk 启动自清障**：见 §2.2。
5. **netdisk 反代前缀 middleware 自剥**（弃用 uvicorn root_path）：见 §2.2；详见《Nginx反向代理机制》§4.1.1。
