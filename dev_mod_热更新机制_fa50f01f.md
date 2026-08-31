# dev_mod 特殊包名 + 服务热更新机制

## 背景与目标

现状：auth 导航首页卡片（`SERVICES` 注册表）经 `/api/v1/services/{name}/start|restart|stop` 用 `_run_wuwor(packages, run_args)` 按别名启动各服务；热更新能力参差不齐——auth 有 `--reload`（默认开）、note 后端有 `--reload`（默认关）、**netdisk 无**、**ChatRoom 后端无 CLI 参数**（bat 启动=生产模式硬关）。

目标：wuwo 识别特殊包名 `.dev_mod`（照 `.script_server` 的 `ENV_MODIFIERS` 机制），注入 `L_DEV_MOD=1`；各后端服务识别该变量启用 uvicorn 热更新（改代码自动重启生效）。热更新变为**按需启用**：启动请求带 `.dev_mod` 才开，生产默认零开销。

用法示例：`wuwor lugwit_baidu_netdisk .dev_mod -- baidu_netdisk_web`；卡片「♻ 热更新」按钮 = 停进程树 + 带 `.dev_mod` 重启。

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
文件：`lugwit_baidu_netdisk/999.0/src/lugwit_baidu_netdisk/web_server.py`（main 在 L1700-1707）
- 加 `--reload`/`--no-reload`（dest=reload，default=None）；生效值 = CLI > `LUGWIT_NETDISK_RELOAD` > `L_DEV_MOD`
- `uvicorn.run(..., reload=on, reload_dirs=[<包 src 目录>] if on else None)`
- **模块级**加 Windows Selector 事件循环策略（asyncpg 与 Proactor 不兼容；reload 子进程不执行 main()，必须 import 时生效——照 auth_server.py L46-51 的写法与注释）

### 2.3 ChatRoom 后端（1026）
文件：`ChatRoom/999.0/src/ChatRoom/backend/app/main.py`（L507-560）+ `run_backend_server.bat`（L58）
- `__main__` 里用 `argparse.parse_known_args` 加 `--reload`/`--no-reload`；默认 = CLI > `CHATROOM_RELOAD` > `L_DEV_MOD` > 原 `is_debug_user and not from_bat`
- 三个 uvicorn.run 分支合并为一次调用：`reload=on`，on 时带 `reload_dirs=["app"]` + 现有 `reload_excludes` 列表（L545-557 保留）
- bat L58 改 `python -m app.main %*`（透传参数；`pause` 保留，DETACHED/DEVNULL stdin 下读 EOF 即过）

### 2.4 l_notepad 后端（8765）
文件：`l_notepad/999.0/src/l_notepad/backend_server.py`（L744-753）
- `--reload` 默认值扩展为同时识别 `L_DEV_MOD`（`L_NOTEPAD_RELOAD` 仍优先）
- `local_main.py` **无需改**：`_start_backend_subprocess`（L1736）`env = os.environ.copy()`，卡片以 `l_notepad .dev_mod` 启动时内嵌后端自动继承 `L_DEV_MOD`

## 3. 卡片 reload 端点 + 首页「热更新」按钮

文件：`lugwit_auth/999.0/src/lugwit_auth/auth_server.py` + `templates/home.html`

- 新增 `POST /api/v1/services/{name}/reload`（`Depends(require_admin)`，照 L520 restart 的形态）：
  1. 按端口 `_listen_pids` + `_kill_pids` 停进程树（L388-404 已含 uvicorn reloader 父子进程处理）
  2. `_run_wuwor(svc["packages"] + [".dev_mod"], svc["run_args"])` 带 dev_mod 重启 → 热更新生效
  3. 返回结构照 restart（ok/name/port/started/start_command 含 `.dev_mod`）
- home.html：`↻ 重启` 旁加 `<button class="op reload" onclick="svcReload('{{ s.name }}')">♻ 热更新</button>`（样式照 .restart）；JS 加 `svcReload(name){svcManage(name,"reload")}`，L175 verb 映射补「热更新」。

## 4. 可选：stack bat 支持 dev 模式

文件：`l_nginx/999.0/l_lugwit_stack.bat`
- `start` 子命令接受第二参数 `dev`：`if /i "%~2"=="dev" set "DEVMOD=.dev_mod"`，auth/netdisk 的 wuwor 请求里追加 `%DEVMOD%`。

## 依赖关系

- 2.x 各服务改动相互独立，可并行；都依赖 1（wuwo 注入变量）才能端到端验证
- 3 依赖 1（reload 端点靠 `.dev_mod` 传参）
- 4 依赖 1

## 测试计划

1. 变量注入：`wuwor l_notepad .dev_mod -- python -c "import os;print(os.environ.get('L_DEV_MOD'))"` → `1`；不带 `.dev_mod` → `None`；rez 解析日志 clean_requests 不含 `.dev_mod`
2. netdisk：`wuwor lugwit_baidu_netdisk .dev_mod -- baidu_netdisk_web` → 日志出现 `Started reloader process`；改一行 netdisk 源码 → 自动重启；不带 `.dev_mod` → 单进程无 reloader
3. auth：带 `.dev_mod` 启动有 reloader；不带 → 单进程（验证默认翻转）；`LUGWIT_AUTH_RELOAD=1` 仍可强制开
4. chat：`wuwor ChatRoom .dev_mod -- chatroom_backend` → reload 生效（bat %* 透传验证）
5. note：`wuwor l_notepad .dev_mod -- l_notepad` → 内嵌 8765 后端带 reload（tasklist 见 reloader 父子进程）
6. 卡片：管理员登录首页 →「♻ 热更新」→ 服务以 `.dev_mod` 重启（status/日志确认）；非管理员 403
7. 回归：`l_lugwit_stack.bat start/stop/status` 不受影响；`.solo`/`.script_server` 既有修饰符行为不变

## 风险与缓解

- **auth 默认翻转**（reload 开→关）：依赖自动重载的开发者需加 `.dev_mod`；缓解=代码注释说明 + 专属 env 兜底
- **Windows + asyncpg + reload**：事件循环策略必须模块级设置（auth 已有先例注释），netdisk 照抄
- **reload 监控范围**：reload_dirs 只指自家 src + excludes 日志/临时文件，避免文件风暴反复重启
- **wuwo.bat GBK**：帮助文本可改可不改；改则保 GBK+CRLF
- **ChatRoom bat %***：用 parse_known_args 不破坏 FROM_BAT 既有逻辑

## 已否决的替代方案

- **每个服务只加 `--reload` CLI、不走环境变量**：卡片/stack 经别名启动没有顺手的参数通道，`.dev_mod` 在所有启动路径（卡片 API、stack、手动 wuwor）统一生效
- **每包新增 `<svc>_reload` 别名**：rez 别名无法作用于已运行进程；热更新靠 uvicorn 文件监控 + reload 端点即可，别名冗余
- **新建共享服务管理 Python 包**：跨包依赖改动重；环境变量方案零依赖、各服务两行代码接入

## 关键文件

- `wuwo/py_modules/wuwo_rez.py`（ENV_MODIFIERS，L62-67）
- `lugwit_auth/999.0/src/lugwit_auth/auth_server.py`（reload 默认 L1118；reload 端点加在 L520 restart 旁；SERVICES L64）
- `lugwit_baidu_netdisk/999.0/src/lugwit_baidu_netdisk/web_server.py`（main L1700）
- `ChatRoom/999.0/src/ChatRoom/backend/app/main.py`（L507-560）+ `run_backend_server.bat`（L58）
- `l_notepad/999.0/src/l_notepad/backend_server.py`（L744）
- `lugwit_auth/999.0/src/lugwit_auth/templates/home.html`（按钮 L145-147、JS L166-192）
