# Rez 包创建指导文档

路径：`D:\TD_Depot\Software\Lugwit_syncPlug\lugwit_insapp\trayapp\rez-package-source`

本文基于当前目录下已有 Rez 包结构整理，用于新建、维护、调用同类 Rez 包。

## 1. 当前仓库包结构

典型包目录：

```text
rez-package-source/
  包名/
    版本号/
      package.py
      src/
        包名/
          ...
```

示例：

```text
l_tray/999.0/package.py
l_qt_wgt_lib/999.0/package.py
Lugwit_Module/999.0/package.py
postgresql/999.0/package.py
wuzu/1.0.1/package.py
```

已有常见 Rez 包包括：

- `ChatRoom`
- `conemu`
- `l_agent_chat`
- `l_agent_tool`
- `l_everything`
- `l_fastapi_guard`
- `l_folder_favorites`
- `l_frp`
- `l_homepage`
- `l_log`
- `l_media_converter`
- `l_muse_backup_viewer`
- `l_nginx`
- `l_notepad`
- `l_notepad_client`
- `l_notepad_server`
- `l_qt_wgt_lib`
- `l_repo_sync_gui`
- `l_scheduler`
- `l_script_editor`
- `l_simple_gui`
- `l_thread_safe`
- `L_Tools`
- `l_tray`
- `l_WChat`
- `lperforce`
- `lugwit_auth`
- `lugwit_baidu_netdisk`
- `Lugwit_Module`
- `Lugwit_PackageRegistry`
- `postgresql`
- `pyfory`
- `pytracemp`
- `start_multi_app`
- `ui_validator`
- `view_pkl_tool`
- `wuzu`

## 2. 新建 Rez 包目录

以创建 `my_tool` 包为例：

```text
rez-package-source/
  my_tool/
    999.0/
      package.py
      src/
        my_tool/
          __init__.py
          main.py
```

推荐版本号规则：

- 内部开发包：`999.0`
- 正式发布包：`1.0.0`、`1.0.1` 等语义化版本
- Python 版本绑定包：可使用类似 `999.0-py3.12`

## 3. 最小 package.py 模板

```python
# -*- coding: utf-8 -*-

name = "my_tool"
version = "999.0"
description = "My Tool Package"
authors = ["Lugwit Team"]

requires = [
    "python-3.12+<3.13",
]

build_command = False
cachable = True
relocatable = True


def commands():
    env.PYTHONPATH.prepend("{root}/src")
    env.MY_TOOL_ROOT = "{root}"
    alias("my_tool", "python {root}/src/my_tool/main.py")
```

## 4. package.py 字段说明

### `name`

Rez 包名。应与包目录名保持一致。

```python
name = "l_tray"
```

### `version`

Rez 包版本。

```python
version = "999.0"
```

### `description`

包说明。用于识别包用途。

```python
description = "Lugwit Tray Application"
```

### `authors`

维护者信息。

```python
authors = ["Lugwit Team"]
```

### `requires`

运行依赖。当前仓库常见依赖写法：

```python
requires = [
    "python-3.12+<3.13",
    "pyside6",
    "pywin32",
    "Lugwit_Module",
]
```

注意：

- Rez 包依赖写包名，不写本地路径。
- Python 版本建议明确范围。
- 依赖其他内部包时，直接写对应 Rez 包名。
- **依赖（尤其是第三方 pip 库）只需在 `requires` 里声明包名即可，无需手动安装**——
  wuwo 加载包时会自动解析并补齐缺失的依赖（见下节「自动下载依赖」）。

### 自动下载依赖（不用手动装）

`wuwor <包> -- ...` 启动时，wuwo 会先检查该包 `requires` 闭包里的依赖，
**缺什么自动补什么**，无需手动 clone / pip 安装：

- **GitHub 包**：注册表里登记的内部包缺失时，自动 `git clone` 到 `rez-package-source`
- **pip 库**（fastapi / watchfiles / pyside6 等）：自动用 pip 装进
  `rez-package-3rd/<包名>/<版本>/` 并生成 rez 包装（package.py），
  之后 `requires` 里直接写包名即可解析
- **nuget 包**：同理自动安装到 `rez-package-3rd`

实现：`wuwo/py_modules/auto_fetch_packages.py`（`--for-rez-env` 在每次
`wuwor ... rez env ...` 前解析 requires 闭包，按需补齐）。

> 实操要点：
> - 因此**不要在 package.py 里写 `env.PYTHONPATH.append(r"E:\...")` 这类
>   硬编码外部路径 hack** 去"补依赖"——正确做法是把它写进 `requires`，
>   由 wuwo 自动下载并纳入环境。
> - 首次启动某包时若需联网下载依赖会稍慢；下载后即缓存，后续秒起。
> - 自动下载依赖失败（如网络不通）时，`wuwor` 会报错提示，可按报错手动
>   补装或重试（如 `wuwor <包> .update` 强制刷新 GitHub 包）。

### `build_command`

当前仓库多数包使用：

```python
build_command = False
```

表示不执行额外构建命令。

### `cachable`

```python
cachable = True
```

允许 Rez 缓存包。

### `relocatable`

```python
relocatable = True
```

表示包可移动，不强绑定绝对安装路径。

## 5. commands() 配置

`commands()` 是 Rez 激活包环境时执行的配置入口。

常见用途：

### 添加 Python 模块路径

```python
def commands():
    env.PYTHONPATH.prepend("{root}/src")
```

如果希望当前包 Python 模块可被 `import`，必须加入 `PYTHONPATH`。

### 设置包根路径变量

```python
def commands():
    env.MY_TOOL_ROOT = "{root}"
```

建议环境变量名使用大写包名，例如：

- `l_tray` -> `L_TRAY_ROOT`
- `l_qt_wgt_lib` -> `L_QT_WGT_LIB_ROOT`
- `Lugwit_Module` -> `LUGWIT_MODULE_ROOT`

### 添加命令别名

```python
def commands():
    alias("my_tool", "python {root}/src/my_tool/main.py")
```

调用：

```bat
wuwor my_tool -- my_tool
```

### Windows 下调用 bat

参考 `postgresql` 包写法：

```python
alias("postgres_start", 'cmd /c "{root}/src/postgresql/bat/postgres_start.bat"')
```

### Windows 下添加 PATH

```python
def commands():
    env.PATH.prepend("{root}/bin")
```

## 6. 创建 Python 工具包示例

目录：

```text
my_tool/999.0/
  package.py
  src/my_tool/__init__.py
  src/my_tool/main.py
```

`src/my_tool/main.py`：

```python
def main():
    print("Hello from my_tool")


if __name__ == "__main__":
    main()
```

`package.py`：

```python
# -*- coding: utf-8 -*-

name = "my_tool"
version = "999.0"
description = "示例 Python 工具包"
authors = ["Lugwit Team"]

requires = ["python-3.12+<3.13"]

build_command = False
cachable = True
relocatable = True


def commands():
    env.PYTHONPATH.prepend("{root}/src")
    env.MY_TOOL_ROOT = "{root}"
    alias("my_tool", "python {root}/src/my_tool/main.py")
```

调用：

```bat
wuwor my_tool -- my_tool
```

或进入环境：

```bat
wuwor my_tool
python -m my_tool.main
```

## 7. 创建 GUI 工具包示例

适用于 PySide6 工具：

```python
# -*- coding: utf-8 -*-

name = "my_gui"
version = "999.0"
description = "示例 GUI 工具包"
authors = ["Lugwit Team"]

requires = [
    "python-3.12+<3.13",
    "pyside6",
    "qtpy",
    "Lugwit_Module",
]

build_command = False
cachable = True
relocatable = True


def commands():
    env.PYTHONPATH.prepend("{root}/src")
    env.MY_GUI_ROOT = "{root}"
    alias("my_gui", "python {root}/src/my_gui/main.py")
```

## 8. 创建带二进制程序的包示例

适用于工具程序、服务端程序、本地可执行文件：

```python
# -*- coding: utf-8 -*-

name = "my_binary_tool"
version = "999.0"
description = "示例二进制工具包"
authors = ["Lugwit Team"]

build_command = False
cachable = True
relocatable = True


def commands():
    env.MY_BINARY_TOOL_ROOT = "{root}"
    env.PATH.prepend("{root}/bin")
    alias("my_binary_tool", "{root}/bin/my_binary_tool.exe")
```

## 9. 创建 Web 服务型包示例

适用于 FastAPI / uvicorn 服务（参考 `lugwit_auth`、`l_notepad_server`、`ChatRoom` 后端）。

目录结构：

```text
my_web/999.0/
  package.py
  src/my_web/
    __init__.py
    auth_server.py      # FastAPI app + main()
    templates/          # Jinja2 模板
    static/             # 静态资源（CSS/JS/图片）
```

package.py：

```python
# -*- coding: utf-8 -*-

name = "my_web"
version = "999.0"
description = "示例 Web 服务包"
authors = ["Lugwit Team"]

requires = [
    "python-3.12+<3.13",
    "fastapi",
    "uvicorn",
    "jinja2",
    "pydantic",
    "asyncpg",   # 若用 asyncpg 连库必须显式声明，否则 ModuleNotFoundError
]

build_command = False
cachable = True
relocatable = True


def commands():
    env.PYTHONPATH.prepend("{root}/src")
    env.MY_WEB_ROOT = "{root}"
    alias("my_web_server", "python -m my_web.auth_server --host 127.0.0.1 --port 8000")
```

要点：

- 服务入口 alias 用 `python -m 包名.模块`（带 `--host/--port`），可配置环境变量默认值。
- 模板/静态资源：`Jinja2Templates(directory=str(Path(__file__).resolve().parent / "templates"))` + `app.mount("/static", StaticFiles(directory=...))`；路径基于 `__file__`，不要硬编码。
- ⚠️ Windows + asyncpg：与默认 Proactor 事件循环不兼容（WinError 64），必须在**模块级**设置 `asyncio.WindowsSelectorEventLoopPolicy()`（不要只放 `main()`，见 9.1）。

### 9.1 开发模式 --reload（自动重载，勿手动重启）

```python
def main():
    parser.add_argument("--reload", dest="reload", action="store_true", default=True)
    parser.add_argument("--no-reload", dest="reload", action="store_false")
    ...
    uvicorn.run(
        "my_web.auth_server:app",   # ⚠️ 必须传 import string，不能传 app 对象
        host=..., port=...,
        reload=args.reload,
        reload_dirs=[str(Path(__file__).resolve().parent)],  # ⚠️ 指向源码目录
    )
```

踩坑（反复遇到）：

- **reload 必须传 import string**（`"pkg.module:app"`），传 app 对象会警告且不生效。
- 必须用 **`reload_dirs` 显式指向源码目录**——wuwo 启动器会把子进程 cwd 切到 wuwo 目录，uvicorn 默认监视 cwd 会监视错目录。
- Windows 事件循环策略（asyncpg）要放**模块级**：reload 子进程不执行 `main()`。
- wuwo 的 `wuwo_rez.py` 现默认继承调用者 cwd（无回退），但 `reload_dirs` 仍是稳妥写法。

### 9.2 静态资源与浏览器缓存

- 修改静态 JS/CSS 后，模板里的引用要**递增版本号**（`?v=1.1` → `?v=1.2`），否则浏览器缓存旧资源不生效。
- 服务端返回自定义 HTML 时，直接读文件返回 `HTMLResponse`，静态文件通过 `StaticFiles` 挂载。

## 10. 调用已有 Rez 包

### 进入单包环境

```bat
wuwor l_tray
```

### 进入多包环境

```bat
wuwor l_tray l_qt_wgt_lib Lugwit_Module
```

### 直接调用别名

```bat
wuwor l_tray -- st
```

或：

```bat
wuwor l_tray -- start_tray
```

### 调用 Python 模块

```bat
wuwor Lugwit_Module -- python -c "import Lugwit_Module; print(Lugwit_Module)"
```

### 调用数据库包命令

```bat
wuwor postgresql -- postgres_status
wuwor postgresql -- postgres_start
wuwor postgresql -- postgres_stop
```

### 修饰符（虚拟包）

`wuwor <包> [修饰符] -- <命令>`：包请求位置可传 `.xxx` 修饰符（虚拟包）。
修饰符会被 wuwo 识别并**剥离，不注入 rez 环境**；其中环境变量类修饰符只设置
对应环境变量，由目标程序识别后启用特定行为：

| 修饰符 | 作用 | 识别方 / 行为 |
|--------|------|--------------|
| `.script_server` | 设 `L_SCRIPT_SERVER=1` | 标题栏（`L_FramelessMainWindow`）优先识别，**总是启动脚本编辑器 HTTP 远程执行服务**（l_script_editor，默认 8764，被占用时自动向上找可用端口） |
| `.dev_mod` | 设 `L_DEV_MOD=1` | 各后端服务（auth/netdisk/chat/note/agent 等）识别后**启用 uvicorn 热更新**（reload，勿手动重启，见 9.1）；主页卡片可配专用热更新别名（`reload_args`，如 l_notepad_server 的 `l_notepad_api_reload`） |
| `.comfyui_lite` | 设 `COMFY_LITE=1` | 轻量 ComfyUI 模式 |
| `.solo` | 动作 | 单实例守卫：已有实例运行时直接退出 |
| `.update` | 动作 | 强制更新 GitHub 包（fetch + reset --hard） |
| `.ps` / `.cmd` | 终端 | 在新 PowerShell / cmd 窗口启动 |

示例：

```bat
:: 启动 l_notepad_server 并让标题栏总是启动脚本服务
wuwor l_notepad_server .script_server -- l_notepad_server

:: 开发模式启动后端服务（uvicorn 热更新）
wuwor lugwit_auth .dev_mod -- auth_server

:: 修饰符可任意位置、可叠加
wuwor l_notepad_server .solo .script_server -- l_notepad_server
```

实现位置：环境变量类修饰符在 `wuwo/py_modules/wuwo_rez.py` 的 `ENV_MODIFIERS`
中登记（解析时设环境变量并剥离，不传给 rez）；对应识别代码：

- 标题栏：`l_qframelesswindow/.../L_FramelessMainWindow.__init__` 检测
  `L_SCRIPT_SERVER` 后经 `QTimer.singleShot(0, ...)` 总是启动脚本服务
- 后端服务：各服务入口检测 `L_DEV_MOD` 后给 uvicorn 传 `--reload`
  （reload 踩坑见 9.1）

## 11. 本仓库常见写法参考

### `l_tray`

特点：

- 添加 `{root}/src` 到 `PYTHONPATH`
- 设置 `L_TRAY_ROOT`
- 添加 `start_tray`、`st` 别名
- 依赖 `pyside6`、`pywin32`、`psutil`、`Lugwit_Module` 等

### `l_qt_wgt_lib`

特点：

- GUI/Qt 组件库
- 依赖 `pyside6`、`qtpy`、`Lugwit_Module`、`l_thread_safe`
- 提供 `code_editor`、`progress_win` 别名

### `Lugwit_Module`

特点：

- 基础 Python 工具库
- 添加 `{root}/src` 到 `PYTHONPATH`
- 设置 `LUGWIT_MODULE_ROOT`

## 12. QtWebEngine（网页 UI 客户端）踩坑与解法

参考包：`lugwit_netdisk_client/999.0`（PySide6 托盘 + `QWebEngineView` 承载网页 UI）。

只要包里要用 `QtWebEngineWidgets`，下面两个坑必踩，且报错都不指向真实原因。

### 12.1 坑一：PySide6 在 rez 里被拆成三个包 → `DLL load failed`

`rez-package-3rd` 下 PySide6 是三个独立包：

```text
pyside6_essentials/6.11.0/.../python/PySide6/   # QtCore/QtGui/QtWidgets
pyside6_addons/6.11.0/.../python/PySide6/       # QtWebEngineCore/QtWebEngineWidgets
shiboken6/6.11.0/.../python/shiboken6/ + PySide6/
```

`pyside6` 元包只是把三者的 `python` 目录都 `append` 到 `PYTHONPATH`。

问题：三个目录下各有一个 **同名 `PySide6` 常规包**（带 `__init__.py`）。Python 只认
`PYTHONPATH` 里第一个命中的，后面两个直接不可见 → `import PySide6.QtWebEngineWidgets`
报 `ImportError: DLL load failed`。

试过但**无效**的做法（勿重复踩）：

| 做法 | 结果 |
|------|------|
| `PySide6.__path__.append(addons_dir)` | 仍 `DLL load failed while importing QtWebEngineCore` |
| 三个目录都 `os.add_dll_directory()` + 加 `PATH` | import 过了，但启动即 `ERROR:icu_util.cc:227] Invalid file descriptor to ICU data received.` 进程死 |
| 设 `QTWEBENGINE_RESOURCES_PATH` / `QTWEBENGINE_LOCALES_PATH` / `QTWEBENGINEPROCESS_PATH` | ICU 错误依旧 |

**正解：硬链接合并成一个 `PySide6` 目录。**

`qt_boot.merge()` 把三个源目录 `os.walk` 后用 `os.link()` 链到
`rez-package-3rd/.pyside6_merged/<版本>/`（同卷 → 零额外磁盘，3719 个文件全硬链接、0 复制），
写 `_lugwit_merged.ok` 标记做幂等。合并目录插到 `sys.path[0]`，ICU 错误随之消失。

要点：合并目录**必须与源包同卷**，否则 `os.link` 失败退化为 `shutil.copy2`（能跑但占空间）。

### 12.2 坑二：渲染进程 `0xC0000135` → 页面白屏

合并后 import 正常，但加载页面时：

```text
RENDER_DEAD status=CrashedTerminationStatus code=-1073741515
LOAD_FINISHED False
```

`-1073741515` = `0xC0000135` = `STATUS_DLL_NOT_FOUND`。原因是 rez 环境下
`QtWebEngineProcess.exe` 沙箱子进程拿不到父进程的 DLL 搜索路径。

**正解：关沙箱。** 在 import PySide6 **之前**设：

```python
os.environ.setdefault("QTWEBENGINE_CHROMIUM_FLAGS", "--no-sandbox --disable-gpu")
```

### 12.3 启动顺序（硬约束）

```python
from . import qt_boot
qt_boot.setup()          # ⚠️ 必须在任何 import PySide6 之前

from PySide6.QtWidgets import QApplication   # 之后才允许
```

`setup()` 内部用模块级 `_READY` 缓存做幂等；若检测到 `PySide6` 已被导入且**不是**来自合并目录，
直接抛 `RuntimeError`，避免出现"看似能跑、实际半残"的环境。

### 12.4 自检别名

包里挂一个体检别名，环境有问题时先跑它，不要直接调 GUI：

```python
alias("netdisk_client_doctor", "python -m lugwit_netdisk_client.qt_boot")
```

`doctor()` 流程：`setup()` → import → 建 `QWebEngineView` → `setHtml()` → 断言 `loadFinished`。

```bat
wuwor lugwit_netdisk_client -- netdisk_client_doctor
```

期望输出：

```text
MERGED_DIR ...\rez-package-3rd\.pyside6_merged\6.11.0
IMPORT_OK  flags = --no-sandbox --disable-gpu
LOAD_FINISHED True | TITLE 'doctor'
RESULT PASS
```

### 12.5 QWebChannel 桥（网页调本地能力）

网页 UI 要读本地文件、弹选择框，必须走 `QWebChannel`。注入方式：

```python
script = QWebEngineScript()
script.setInjectionPoint(QWebEngineScript.DocumentCreation)   # 页面 JS 之前
script.setWorldId(QWebEngineScript.MainWorld)                 # 页面能访问到
script.setSourceCode(_qwebchannel_js() + _INJECT_JS)
```

`_qwebchannel_js()` 从 Qt 资源读 `:/qtwebchannel/qwebchannel.js`（16509 字节），
不要自己从 npm 拷一份。`_INJECT_JS` 建好 channel 后挂 `window.lugwitBridge`
并派发 `lugwit-bridge-ready` 事件。

网页侧双用写法（浏览器打开也不崩）：

```javascript
if (window.lugwitBridge) { /* 客户端内 */ }
else { window.addEventListener("lugwit-bridge-ready", init); /* 或退化成纯网页流程 */ }
```

`Bridge`（`bridge.py`）已提供：`platform` / `pickFolder` / `pickFiles` / `revealInExplorer` /
`fileMd5`（4MB 分块 + `progress` 信号）/ `fileInfo` / `listDir` / `homeDir` / `openExternal`。

### 12.6 常驻托盘

- `app.setQuitOnLastWindowClosed(False)`
- `closeEvent` 里 `event.ignore(); self.hide()`
- `QWebEngineProfile("lugwit_netdisk")` 持久化到 `~/.lugwit/netdisk_client/webprofile`，
  `ForcePersistentCookies` → 登录态不丢
- 接 `renderProcessTerminated` → 提示 + `reload()`
- 根据环境配置调试变量、自动导入脚本、Maya 路径

### `postgresql`

特点：

- 二进制程序包
- 设置 `POSTGRESQL_ROOT`、`PGROOT`
- 添加 `pgsql/bin` 到 `PATH`
- 提供 `postgres_start`、`postgres_stop`、`postgres_status` 等别名

### `wuzu`

特点：

- Rez 基础环境包
- 读取 wuzu 配置和注册表
- 设置 `WZ_BASE_NAME`、`WZ_PIPELINE_ROOT`、`REZ_CONFIG_FILE`
- 提供 `remount` 别名

## 12. 包创建检查清单

创建新包后检查：

- [ ] 目录结构是否为 `包名/版本号/package.py`
- [ ] `name` 是否与包目录名一致
- [ ] `version` 是否与版本目录一致
- [ ] `requires` 是否包含运行所需依赖
- [ ] Python 包是否放在 `src/包名/`
- [ ] 是否在 `commands()` 中配置 `PYTHONPATH`
- [ ] 是否设置了包根环境变量
- [ ] 是否需要 `alias()` 启动命令
- [ ] Windows 路径是否正确加引号
- [ ] 是否避免硬编码用户机器路径

## 13. 测试新包

### 检查包是否可解析

```bat
wuwor my_tool
```

### 测试 Python import

```bat
wuwor my_tool -- python -c "import my_tool; print(my_tool)"
```

### 测试别名

```bat
wuwor my_tool -- my_tool
```

### 测试依赖是否生效

```bat
wowor my_tool -- python -c "import sys; print(sys.path)"
```

## 14. 常见问题

### `python` 命令找不到 / 系统未安装 Python

现象：在 PowerShell 里执行 `python`，可能解析到 `C:\Users\...\WindowsApps\python.exe`（Windows 应用执行别名 stub），运行报错或无效（退出码 9009）。

原因：系统未安装 Python 或未加入 PATH。**wuwor 自带便携 Python，整个 `trayapp/` 自包含、不依赖系统 Python**，因此统一用 `wuwor` 激活环境即可获得可用的 `python`。

另外，`wuwor` 命令本身也已加入系统 PATH（安装时写入，指向 `...\trayapp\wuwo\` 目录），因此**在任何终端直接输入 `wuwor` 即可调用**，不会出现"找不到 wuwor"的情况。若在其他机器上找不到 `wuwor`，检查 PATH 是否包含 `trayapp\wuwo` 目录。

解决：使用 `wuwor` 提供统一 Python 环境：

```bat
:: 裸 Python 环境（仅 python 3.12 + 核心包 l_app_ready / rez_pip_installer）
wuwor -- python -c "import sys; print(sys.version)"

:: 带某 Rez 包及其依赖的环境（如需 PySide6、psutil 等）
wuwor my_tool -- python -c "import my_tool; print(my_tool)"
```

说明：

- `wuwor -- python` 会展开为 `rez env python-3.12+<3.13 l_app_ready rez_pip_installer -- python ...`，即"裸 Python"环境，**无第三方库**。
- 脚本需要第三方库（如 `pyside6`、`psutil`）时，必须指定对应 Rez 包名，例如 `wuwor start_multi_app -- python ...`。
- 激活时会清理 `PYTHONHOME` / `PYTHONPATH` / `PYTHONEXECUTABLE`，避免父环境干扰。

### import 找不到模块

原因：`commands()` 没有添加源码目录。

修复：

```python
env.PYTHONPATH.prepend("{root}/src")
```

### alias 调用失败

检查：

- `main.py` 路径是否存在
- Windows 路径是否需要引号
- Python 文件是否有 `if __name__ == "__main__"`

### 依赖包找不到

检查：

- `requires` 中包名是否写对
- 依赖包是否存在于 Rez 包路径
- 版本约束是否过窄

### 中文输出乱码

可参考 `l_tray` 设置：

```python
env.PYTHONIOENCODING = "utf-8"
```

或在命令里执行：

```bat
chcp 65001
```

## 15. 推荐 package.py 标准模板

```python
# -*- coding: utf-8 -*-

name = "package_name"
version = "999.0"
description = "Package description"
authors = ["Lugwit Team"]

requires = [
    "python-3.12+<3.13",
]

build_command = False
cachable = True
relocatable = True


def commands():
    env.PYTHONPATH.prepend("{root}/src")
    env.PACKAGE_NAME_ROOT = "{root}"
    alias("package_name", "python {root}/src/package_name/main.py")
```

## 16. 建议规范

- 包名使用小写和下划线，除已有历史包外避免大小写混用。
- Python 源码统一放到 `src/包名/`。
- 工具入口统一提供 `alias()`。
- 依赖写在 `requires`，不要在代码里临时修改外部路径。
- 路径优先使用 `{root}`，不要硬编码绝对路径。
- Windows 批处理命令使用 `cmd /c`。
- GUI 包明确依赖 `pyside6` 或 `qtpy`。
- 基础库包只配置 `PYTHONPATH`，不要自动启动程序。

## 17. 服务化架构约定（实战补充）

### 17.1 客户端通过 HTTP 访问服务，不直接依赖服务包

核心服务（如 `lugwit_auth` Auth Service）只被自己的进程 import；**客户端用 HTTP 调用，不把服务包写进 requires**：

```python
# 客户端（如 l_notepad_client）纯标准库 urllib 封装，避免拉入服务端重依赖
import json
import urllib.request

def _http_json(method, path, body=None, token=""):
    ...  # POST /api/v1/auth/login、GET /api/v1/users、POST /api/v1/auth/verify 等
```

好处：客户端 `requires` 干净（不引入 asyncpg/jose 等服务端依赖），服务可独立演进/部署，token 验证走 `POST /api/v1/auth/verify` 而非本地解析 JWT。

### 17.2 第三方依赖原则

- **Python 解释器保持纯净**：第三方包统一通过 rez `requires` 获取，不往解释器里 pip 安装。
- 需要第三方库时（pyside6、psutil、fastapi…），必须在 `requires` 声明对应 rez 包名。

### 17.3 构建与开发包

- 每个包目录下有 `build.bat`（`wuwor rez build -i`），配合 `package.py` 的 `build_command=False` 打包/安装。
- **内部开发包 `999.0` 改源码即生效**：`commands()` 里 `env.PYTHONPATH.prepend("{root}/src")`，Python 直接 import 源码目录，无需 build。
- 正式发布用语义化版本（`1.0.0` 等）。

### 17.4 端口约定汇总

| 端口 | 服务 |
|------|------|
| 1027 | Auth Service（lugwit_auth 统一用户中心） |
| 1026 | ChatRoom 后端 |
| 1025 | ChatRoom 前端（Vite） |
| 8765 | L Notepad 网页端 |
| 5432 | PostgreSQL（chatroom 库） |

> 端口通过环境变量可覆盖（如 `LUGWIT_AUTH_PORT`），默认值在各自 `config.py` / 启动脚本。
