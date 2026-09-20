# 托盘应用 l_tray（宿主能力 / 统一登录 / 服务管理 / 排错）

> 2026-09-20 更新。相关文档：`服务发现与IPC.md`（本机怎么找到并调用服务）、
> `solo_单实例守卫模式.md`（`.solo` / `.soloignore` 修饰符）、
> `../lugwit_auth统一用户授权服务设计.md`（§9 P5 统一登录窗口）。

## 1. 入口与两条硬约束

| 项 | 值 |
|---|---|
| 启动别名 | `wuwor l_tray -- start_tray`（`start_tray.bat` → ConEmu 新窗口 → `src/l_tray/plugSync.py`）；`st` = 直接跑 `plugSync.py` |
| 重启（接管式） | `set L_TRAY_RESTART=1` + `wuwor l_tray -- st` → 新实例的单实例守卫干掉旧进程后接管，不出现双托盘（`TrayIcon.restart()` 内部就是这么做的） |
| 日志 | `<wowo_log_dir>/rez_pkg_log/l_tray/l_tray_st_<YYYYMMDD>.log`（默认 `D:/Temp/Log`） |

**约束 1：Tray.py 是"顶层模块"被导入的**（`plugSync.py:146 from Tray import TrayIcon`，`src/l_tray`
在 `sys.path` 上）。因此 Tray.py 里**不能用 `from . import xxx`**（报
`attempted relative import with no known parent package`）——取同目录模块统一走
`Tray._load_sibling_module(name)`（先试包内相对导入，再试顶层 import，最后按文件路径加载）。

**约束 2：托盘菜单只在启动时构建**。改了菜单相关代码/配置（`smallProgramList.yaml`、
`sys_tools.yaml`、`auto_start.json`、`Tray.py`）必须**重启托盘**才生效。

## 2. 托盘进程自身提供的服务

菜单那行按钮里点「服务管理」可实时查看（`services_panel.py`，4 秒自动刷新，双击行=打开地址）：

| 服务 | 类型 | 地址 | 说明 |
|---|---|---|---|
| 进程内执行服务 **ExecServer** | HTTP · 仅本机 | `http://127.0.0.1:19527` | 在托盘进程内执行 Python 函数，避免 spawn 开销。GET `/health` `/docs` `/list_packages` `/worker_stats` `/`（帮助页）；POST `/run`（预注册动作）/`/register`。浏览器跨源只放行白名单（`DEFAULT_WEB_ORIGINS` + `L_TRAY_EXEC_ORIGINS` 追加） |
| **备用解释器池 WorkerPool** | 进程池 | — | 预启动空闲解释器，小工具网格里 `use_worker: true` 的项"秒开"；池大小由托盘按机器决定 |
| **统一登录回环页** | HTTP · 临时 | `127.0.0.1:<随机端口>` | 仅登录期间存在（见 §3） |
| **启动管理调度** | 调度器 | — | 托盘启动 ~1.5s 后按包内 `config/startup_builtin.json` + 用户 `~/.lugwit/l_tray/auto_start.json`（按 id 覆盖内置项）拉起启用的项 |
| **进程监督 plugSync** | 外部守护 | 父进程 | 托盘被异常结束会被拉起；托盘「重启」会先清理进程内派生的子服务 |

ExecServer 里注册的动作分两类：`action_registry`（本机脚本/命令行可调，如 `depot_*` 之外的自定义动作）、
`web_actions`（**网页可调白名单**，当前 16 个：`depot_*`、`kb_ws_*`）。查当前清单：

```cmd
curl.exe -s http://127.0.0.1:19527/health          :: registered_actions / web_actions
curl.exe -s http://127.0.0.1:19527/worker_stats    :: 备用解释器池
curl.exe -s http://127.0.0.1:19527/docs            :: 端点说明（浏览器看）
```

## 3. 统一登录（P5）

**点菜单里的「登录」**，三种走向都汇到同一处会话：

1. **浏览器授权码（主路径）**：托盘起一个本机回环页 → 打开
   `<auth>/api/v1/auth/authorize?client_id=desktop&redirect_uri=http://127.0.0.1:<port>/callback&...&code_challenge=<S256>`
   → 302 回本机页 → 页内服务端用 `code_verifier` 兑换 token。
2. **回环页上直接输账号密码**：同一个页面带表单（`POST /login` → 本机进程转发给 auth）；
   授权码没走完 / `state` 不符 / 兑换失败时，页面就是这张表单（错误直接回显，不白屏）。
3. **账号密码 API**（无 UI 场景）：`LUGWIT_USER` + `LUGWIT_PASSWORD` 环境变量。

登录态存放（**P5 §7 口径：refresh 落盘、access 只留内存**）：

| 项 | 位置/行为 |
|---|---|
| access | 仅进程内存（`depot_bridge.set_session_token` / `token()`） |
| refresh | `~/.lugwit/lugwit_auth/tray_tokens.json`，Windows 下 **DPAPI 包裹**（按"当前用户+机器"加密）、Linux 退化为 0600 明文；**删掉即强制重登** |
| 自动登录 | 托盘启动 ~2.5s 后后台 `restore_session()`：读 refresh → `POST /auth/refresh` → 换出可用 access（refresh 轮转后**回写**）→ 登录按钮直接显示 `登出(<用户名>)` |
| refresh 失效 | 清掉落盘文件、安静回到未登录（不会弹错） |
| 登出 | 撤销服务端会话 + 清内存 + **删落盘文件** |

**为什么不用 `lugwit_auth.client` SDK**：托盘环境只装 Qt + 工具包，**没有 Web 栈**
（fastapi/uvicorn/sqlalchemy…），为一次登录把整套服务端依赖塞进托盘不划算、也易与托盘既有依赖冲突。
因此 `depot_bridge.py` 用**标准库**按同一 HTTP 契约实现（与 `l_WChat` / `l_notepad_server` 的做法一致）。

**depot 调用取 token 的顺序**（`depot_bridge.token()`）：
托盘会话 token > `LUGWIT_ACCESS_TOKEN` > `LUGWIT_USER`/`LUGWIT_PASSWORD` 登录。
（`/api/v1/auth/auto` 回环自动授权已按 P0 **默认关闭**，不再作为取 token 途径。）

## 4. 菜单里那几个按钮

| 按钮/项 | 行为 |
|---|---|
| 启动多个程序 | 通过备用解释器池启动（秒开） |
| 强制结束程序 | 下拉选常见托盘程序（`l_notepad_client`/`lugwit_baidu_netdisk`/…），`taskkill /F /T /IM <名>.exe` |
| 服务管理 | 打开 §2 的面板 |
| 退出 | **先弹确认**：默认按钮与 Esc 都是「取消」；弹窗自身异常时放行退出（避免托盘退不掉） |
| 用户信息行 | 本机信息（悬停看 tooltip，右键「复制本机信息」）；紧接着就是登录按钮 |

**启动项的单实例参数**：启动管理与小工具网格里的项已从 `.solo` 改为 **`.soloignore`**
（wuwo 剥离它、恒设 `L_SOLO_IGNORE=1` 并注入 `L_SOLO_PEER_*`，**不杀旧实例**，是否接管交给包自己决定）。
差别与现状见 `solo_单实例守卫模式.md`。

## 5. 排错速查

| 现象 | 先看 |
|---|---|
| 点「登录」没反应 | 托盘日志里 `[l_tray] 登录模块不可用: attempted relative import…` → 用了 `.` 相对导入（见 §1 约束 1） |
| 弹「登录失败：No module named 'lugwit_auth'」 | 已改为 stdlib 实现，不该再出现；若出现说明有人把 SDK 依赖加回来了 |
| 重启托盘后没自动登录 | 看 `~/.lugwit/lugwit_auth/tray_tokens.json` 是否存在；日志里找 `自动恢复登录态失败`（refresh 被撤销/过期会清文件） |
| 服务管理里 ExecServer 显示"未启动" | `curl.exe -s http://127.0.0.1:19527/health`；端口被占时 ExecServer 会记 `ExecServer 端口 19527 被占用` |
| 改了菜单/配置没生效 | 托盘菜单启动时构建 → 重启托盘 |
| 强制结束程序无效 | 目标进程名不对（下拉里没有时手动输入 `<名>.exe` 的 `<名>` 部分） |

## 6. 2026-09-20 变更记录

| 改动 | 说明 |
|---|---|
| 登录入口（新增） | 菜单用户信息行加「登录/登出(<用户>)」按钮；`loginUI.py` 那套独立 MySQL/FastAPI 实验栈的 `Tray.login()`（无调用点）已删 |
| 回环登录页（新增） | `depot_bridge._LoginPageServer`：成功页 / 账号密码表单 / 错误回显 / 授权码失败兜底；成功后页面多活 120 秒 |
| refresh 持久化（新增） | DPAPI 落盘 + 启动自动续期 + 登出清盘（§3） |
| 服务管理面板（新增） | `services_panel.py`，列 §2 五项托盤自身服务 |
| 强制结束程序（补实现） | 之前是**没有处理函数**的空按钮 |
| 退出确认（新增） | `_confirm_quit()`，默认取消、Esc 取消、异常放行 |
| 启动项参数 | 启动管理/小工具网格里 `.solo` → `.soloignore`（含用户 `auto_start.json`） |
| 模块加载 | 新增 `_load_sibling_module()` 统一兼容顶层模块加载（修掉"登录点了没反应"的真因） |
