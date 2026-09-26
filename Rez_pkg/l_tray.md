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
`web_actions`（**网页可调白名单**，当前 22 个：`depot_*`、`depot_local_*`（6 个）、`kb_ws_*`）。查当前清单：

```cmd
curl.exe -s http://127.0.0.1:19527/health          :: registered_actions / web_actions
curl.exe -s http://127.0.0.1:19527/worker_stats    :: 备用解释器池
curl.exe -s http://127.0.0.1:19527/docs            :: 端点说明（浏览器看）
```

**本机目录树（2026-09-26 新增，`l_tray/local_tree.py`）** —— 给网页端版本库页的「工作区」标签用
（浏览器模式没有客户端桥，读不到本机路径）：

| 动作 | 参数 | 返回 |
|---|---|---|
| `depot_local_tree` | `root, depth=3, limit=2000, token` | `{root, tree:[{name,path,isdir,size,mtime,kids}], truncated, seq}`（与客户端 `bridge.treeDir` 同形） |
| `depot_local_version` | `root, token` | `{root, seq}` 变更序号，网页轮询它决定要不要重拉 |
| `depot_local_open` | `path, mode="reveal"｜"open", token` | 在托盘这台机器的资源管理器里打开 / 定位；`reveal`：目录→打开它、文件→打开所在文件夹并选中；`open`：`os.startfile` 交给默认程序（**可执行/脚本/快捷方式类扩展名被拒**，见下） |
| `depot_local_mkdir` | `parent, name, token` | 在 `parent`（目录）下新建**一层**目录；同名已存在 → 报错 |
| `depot_local_newfile` | `parent, name, text="", token` | 新建文本文件（默认空）；同名已存在 → 报错 |
| `depot_local_delete` | `path, token` | 删除文件 / 目录（目录连内容一起），**送回收站**（`winshell.delete_file`，可还原）；**不许删工作区根目录及其上层目录** |

- **边界**：`root` 必须命中**该 token 用户某个 depot 工作区的 `local_root`**，其余动作的 `path` / `parent`
  必须在这些根**之内**（拿 token 查 `GET /api/depot/workspace`，结果缓存 15 秒）；读只给名字/大小/mtime，
  写只给新建 + 删到回收站；名字只能是单层且不含 `\ / : * ? " < > |`；深度 ≤8、条目数有上限；
  这些动作**只支持 Windows**（`explorer` / `os.startfile` / `winshell`）。
- **路径比对走 realpath**（2026-09-26 加固）：`_norm()` = `realpath + normcase + normpath`，`_check_path()`
  返回**解析后**的路径。只做字符串前缀的话，工作区里放一个指向 `C:\Windows` 的 junction 就能越界；
  现在 junction / symlink 逃逸会被判成"不在工作区根目录下"。`depot_local_open(mode="open")` 另外按扩展名
  挡掉 `.exe/.com/.scr/.pif/.msi/.msp/.cpl/.jar/.bat/.cmd/.ps1/.psm1/.vbs/.vbe/.js/.jse/.wsf/.wsh/.hta/.lnk/.url/.reg`
  —— 不然往工作区丢个 `.bat` 就是"网页一键在本机执行"。
- **token 口径**（2026-09-26 改）：页面带的 token **为空时不再回落托盘自己的会话 token**（`_allowed_roots("")`
  直接回空）——那等于把 `token_override`"用网页登录态替代托盘账号"的语义反过来，让没带登录态的页面
  拿到托盘账号的工作区根。现在空 token 直接拒答「网页没带登录态（cookie 读不到？）…」；有 token 但查不到
  工作区才报「拿不到你的工作区本地路径（托盘登录态不可用？…）」。所以**浏览器模式要用这几个动作，
  页面和托盘都得是登录态**；客户端模式的开 / 定位走本地桥（`bridge.revealInExplorer`），不需要托盘，
  但**新建 / 删除目前只有托盘实现**（客户端桥没有写能力）。
- **变更序号**：`watchdog` 递归监听该 root（事件触发即自增；没装 watchdog 退化为根目录
  `mtime + 条目数`）。`l_tray` 的 `requires` 因此加了 `watchdog`，删除用到的 `winshell` 也显式写进了 `requires`
  （此前只是 Tray.py 在 import，属隐式依赖）。
- 改了动作不必重启托盘：`POST /run {"module":"l_tray.local_tree","function":"reregister","reload":true}`
  （注意：`web_actions` 只对**带白名单 Origin 的浏览器请求**开放，本机 curl 不带 Origin 会被当成动态 `/run` 而报 unknown action）。

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

## 7. 2026-09-26 变更记录

| 改动 | 说明 |
|---|---|
| 网页动作 +2（新增） | `l_tray/local_tree.py` 注册 `depot_local_tree` / `depot_local_version`（本机目录树快照 + 变更序号，供网页端版本库页在**浏览器模式**下看工作区本地目录，见 §2） |
| 依赖（新增） | `package.py` 的 `requires` 加 `watchdog`（变更序号靠它事件驱动；缺席时退化为根目录 mtime） |
| 注册点 | `Tray.py` 启动 ExecServer 时 `local_tree.register(...)`；改了动作可 `reregister` 热灌，不必重启托盘 |
| 说明 | 版本库页的工作区列表/增删改已改为**页面直连 depot 服务**，托盘的 `depot_workspace_*` 动作保留给别的调用方（见《网盘版本库Depot设计.md》§6.2） |
| 网页动作 +1（2026-09-26 补） | `depot_local_open`：在资源管理器中打开 / 定位本机文件（`reveal` / `open` 两种），供版本库页工作区树的右键菜单用；路径限工作区 `local_root` 之内 |
| 网页动作 +3（2026-09-26 续） | `depot_local_mkdir` / `depot_local_newfile` / `depot_local_delete`（删除走回收站，不许删根目录本身）；`requires` 显式加 `winshell` |
| 边界加固（2026-09-26 P0） | ① `_norm()` 加 `realpath`，`_check_path()` 回**解析后**路径 → junction/symlink 逃逸不再能蒙过前缀比对；② 空 token **不再回落托盘会话 token**（新增 `_require_roots()`），没带登录态的页面直接拒答；③ `mode="open"` 挡可执行/脚本/快捷方式扩展名（`_NO_STARTFILE_EXTS`）；④ 删根保护扩到"根本身 **及其上层目录**"。验收：假 `depot_bridge` + 临时工作区探针（含 `mklink /J` 逃逸用例）全绿 |
