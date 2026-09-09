# solo 模式：单实例守卫与孤儿 worker 处理

## 背景与目标

`wuwor.BAT <pkg> .solo -- <script>` 中的 `.solo` 是**单实例守卫修饰符**：启动前检测同名应用是否已有实例在跑，有则交互确认"结束旧实例并继续启动"，防止同一服务被重复拉起。

动机是一次真实事故：`l_mindmap_mmd`（8110 端口）出现**两套服务实例抢同一端口**——uvicorn reload 模式的孤儿 worker 占着端口跑旧代码，.solo 守卫却放行了新实例。本文记录 `.solo` 的实现链路、该事故的成因，以及 server 侧的彻底加固方案。

## 1. 实现链路（wuwo_rez.py）

文件：`wuwo/py_modules/wuwo_rez.py`

- `.solo` 属于 `GUARD_MODIFIERS`（L58），由 wuwo.bat 剥离，不注入 rez 环境。
- 命令模式下（L799-858）若带 `.solo`：
  1. `app_name = command_args[0]`（即 rez 别名，如 `l_mindmap_mmd_server`）；
  2. **与 auto_fetch 并行**异步起守卫子进程 `_sg_guard_async`（L530-548）：在 rez env 里跑 `l_app_ready.find_running(app_name)`——打印 JSON（运行中进程信息）并以退出码 1 表示"已在运行"，退出码 0 表示"未运行"；
  3. auto_fetch 完成后 `guard_proc.wait()` 收结果（L815-819）；
  4. 已在运行 → 打印旧实例 pid/cmdline，进入**交互循环**（L852-879）：`_ask_solo_action`（L556-602，msvcrt 单键读取，5 秒无输入默认重启）。若服务包已通过 `l_app_ready.register_url` 登记访问 URL（web 服务），提供 **[R]重启 / [O]打开已运行网址 / [n]保留旧实例退出** 三选；未登记则维持原 [Y/n]。选 R → `_kill_pid_tree`（**taskkill /F /T 杀整棵进程树**）→ 重新探测 → 旧实例已清则继续启动，仍检测到则再次询问；选 O → `webbrowser.open(url)` 打开旧实例网址后本次启动退出；选 n → 保留旧实例、本次启动退出。

## 2. l_app_ready 的进程匹配机制

文件：`wuwo/packages/l_app_ready/1.0.0/src/l_app_ready/__init__.py`

- `find_running(app_name)`（L111-142）：psutil 遍历全部进程，`_is_real_app` 要求 **cmdline[0] 含 "python" 且整条命令行包含匹配模式**；跳过当前进程的**祖先链**（`_get_ancestor_pids`，避免把 rez 启动器链误当实例）；多个命中取 **PID 最大**的（rez 启动器先启动、PID 较小，真正的应用子进程 PID 更大）。
- 匹配模式 = `%TEMP%/lugwit_ready/{app_name}.match`（托盘 `register_launch` 写入）或直接 app_name。
- **匹配弱点（事故根源）**：按命令行匹配 app 名——uvicorn reload 的 **worker 子进程命令行是 `python -c "...spawn_main..." --multiprocessing-fork`，不含 app 名**，守卫永远看不见它。

## 3. 事故复盘：双实例抢 8110

时间线：

1. `taskkill /F /PID <uvicorn 监督父>`（**绑定同端口（SO_REUSEADDR 语义下 uvicorn 双绑不报错）→ 新旧实例静默共存，请求随机分流——落到孤儿的响应全是旧代码行为。

叠加因素：uvicorn + watchfiles 在本环境 **watcher 首次重载后停摆**（touch 源文件不再触发 worker 重建），"热加载看似正常实则失效"，掩盖了问题直到行为明显异常。

## 4. server 侧加固（l_mindmap_mmd/999.0/src/l_mindmap_mmd/server.py）

### 4.1 启动前端口自检

`_port_in_use(port)`（connect 探测 127.0.0.1:port，0.6s 超时）。`main()` 在 `uvicorn.run` 之前探测：端口被占（无论孤儿 worker 还是无关进程）→ 打印 `netstat`/`taskkill` 提示并以退出码 1 拒绝启动。Windows 允许重复绑定，所以**必须主动探测**，靠 bind 报错是不行的。

### 4.2 worker 孤儿自毁线程

`_start_orphan_watch()`：每 3 秒 `OpenProcess(ppid)` 查询父进程存活（PROCESS_QUERY_LIMITED_INFORMATION），父死 → `os._exit(0)` 释放端口。孤儿存活窗口从"永久"缩到 3 秒，.solo 守卫随后即可正确判定"无实例"。

**启动判据**（关键细节）：挂在模块加载处，条件为 `__name__ == "l_mindmap_mmd.server"`——

- reload 的 worker 以 import 方式（import string）加载本模块 → `__name__` 正是 `"l_mindmap_mmd.server"` → 看门狗启动；
- `-m` 监督父进程虽也执行模块代码，但 `__name__ == "__main__"` → 不启动（父进程是必须存活的监督者，杀它=杀服务）；
- 普通第三方 import → `__name__` 同为模块名，理论上也会启动看门狗，但监控的是其自身父进程，语义无害。

**注意不要用 `--multiprocessing-fork in sys.argv` 判断**：spawn worker 的 `sys.argv` 继承自监督父进程（是 server 的启动参数，不含该标志），此判断永远为假、看门狗形同虚设（已踩坑）。

### 4.3 reload 默认关闭

watchfiles watcher 首次重载后会停摆（touch 源文件不再触发 worker 重建），热加载在本环境不可靠且会遗留孤儿。`--reload` 的 argparse 默认值改为 **False**；改 `server.py` 后用 `999.0/dev_restart.bat` 一键重启（按端口 `taskkill /F /T` 清进程树 + wuwor 链重新拉起）。前端 `mm.js` / 模板改动不受影响，仍即时生效。

## 5. L_SOLO 环境变量：让服务包自行响应 .solo

`.solo` 守卫按命令行匹配 app 名，对 reload 孤儿 worker 天然失明（见 §3）；靠各服务复制 ctypes 看门狗代码又难维护。落地为 **环境变量约定**（与 `.dev_mod` → `L_DEV_MOD=1` 同模式）：

- **wuwo 侧**（`wuwo_rez.py`，`_parse_terminal_virtuals` 的 GUARD 分支）：`.solo` 剥离时同步注入 `os.environ["L_SOLO"] = "1"`——守卫匹配逻辑一行未动，子进程（cmd_env 在当前进程执行目标命令）自动继承该变量。
- **服务侧**：识别 `L_SOLO=1` 即启用自我防护（孤儿看门狗、端口自检等）。未带 `.solo` 启动则无此变量，服务按普通模式运行。

### 5.1 共享工具落在 l_app_ready

`wuwo/packages/l_app_ready/1.0.0/src/l_app_ready/__init__.py` 新增两个函数，任何 `.solo` 包两行代码即可接入：

```python
from l_app_ready import port_in_use, start_orphan_watch

if not port_in_use(8110):          # 启动前端口自检
    ...
start_orphan_watch()               # worker 孤儿自毁看门狗（仅 Windows）
```

- `port_in_use(port, host="127.0.0.1")`：connect 探测端口占用（Windows 双绑检测不到 bind 报错）。
- `start_orphan_watch(interval=3.0)`：worker 孤儿自毁看门狗（每 interval 秒 `OpenProcess` 查父进程，父死 `os._exit(0)` 释放端口；仅 Windows）。
- `register_url(app_name, url)` / `solo_open_url(app_name)`：登记/读取服务访问 URL，供 `.solo` 守卫在旧实例运行时提供"打开已运行网址"选项。

### 5.2 l_mindmap_mmd 的响应实现

`server.py`：孤儿看门狗启用条件 = `__name__ == "l_mindmap_mmd.server"`（reload worker 以 import 加载本模块时成立；`-m` 监督父进程 `__name__ == "__main__"` 不启动）**且 `L_SOLO == "1"`**；端口自检改用 `l_app_ready.port_in_use`。

当前 `l_mindmap_mmd` 默认单进程运行（reload 关），看门狗仅在手动 `--reload` 调试的 worker 中激活——正是它需要生效的位置。

> **2026-09 更新**：主页（l_homepage）已**弃用 uvicorn `--reload`**，改用 `l_app_ready` 的
> `src_hot_reload`（模板 `auto_reload` + 源码监听自重启），主页单实例运行，从而彻底避开
> "reload 孤儿 worker 让 `.solo` 守卫失明 → 双实例抢端口"这一坑；主页常驻由外部 `guard` 进程负责。
> 详见《src_hot_reload 源码热重载与主页常驻》。

## 6. 相关文件
| 文件 | 作用 |
| --- | --- |
| `wuwo/wuwor.bat` | 入口，转发 `wuwo.bat rez env ...` |
| `wuwo/py_modules/wuwo_rez.py` | `.solo` 剥离、**注入 `L_SOLO=1`**、守卫异步启动、交互循环（重启/打开网址/保留）、`_kill_pid_tree` |
| `wuwo/packages/l_app_ready/1.0.0/src/l_app_ready/__init__.py` | psutil 进程命令行匹配（find_running）；**共享防护工具 `port_in_use` / `start_orphan_watch` / `register_url` / `solo_open_url`** |
| `l_mindmap_mmd/999.0/src/l_mindmap_mmd/server.py` | 响应 `L_SOLO` 启用孤儿看门狗；启动前端口自检；reload 默认关 |
| `l_mindmap_mmd/999.0/dev_restart.bat` | 一键重启（杀树 + wuwor 链拉起） |
| `l_WChat/999.0/src/l_WChat/app.py` | 响应 `.solo`：注册 `l_wchat_backend` 的匹配模式与访问 URL，使守卫能发现运行实例并支持"打开已运行网址" |
