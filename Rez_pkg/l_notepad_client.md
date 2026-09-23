# l_notepad_client 使用与崩溃排查文档

> `l_notepad_client` 是 L Notepad 的 Qt 桌面客户端（PySide6）。**纯 PC 模式**：本地文件笔记 +
> HTTP 走远端 `l_notepad_server`，不拉起本地后端进程。本页记录启动方式、日志位置，以及
> 「运行中窗口突然消失、无任何提示」这类**静默崩溃**的分层排查方法。
>
> 更新：2026-09-23（代码位置以函数名为准，行号会漂移）

## 1. 启动方式

- 包 alias：`l_notepad_client`（无边框标题栏）/ `l_notepad_ori`（系统原生标题栏）
- wuwo 启动：`wuwor l_notepad_client -- l_notepad_client`
- `commands()` 注入：`PYTHONPATH={root}/src`、`L_NOTEPAD_ROOT`、`PYTHONIOENCODING=utf-8`
- 真正入口：`python -m l_notepad_client.local_main`（`local_main.py::main`）

### 坑：用 alias 启动会被 detach

`wuwor <包> -- <alias>` 走的是 rez alias，进程会**脱离启动器父进程**。用后台/进程管理器跟踪时，
父进程先退出 → 被判定「进程已消失」，很容易**误判成崩溃**。
需要稳定跟踪或常驻托管时，改用**非 alias 的阻塞形式**：

```bat
wuwor l_notepad_client -- python -m l_notepad_client.local_main
```

## 2. 日志位置

| 日志 | 路径 | 说明 |
|------|------|------|
| console 日志 | `D:\Temp\Log\l_notepad\notepad_console.log` | 当前这次运行，启动时重建；由 `logger.py::setup` 配置 |
| 崩溃兜底日志 | `D:\Temp\Log\l_notepad\crash_<YYYYMMDD>.log` | 由 `local_main.py::_install_crash_handlers` 写：**每次进程启动一行 + Python 未捕获异常** |
| 按天业务日志 | `D:\Temp\Log\rez_pkg_log\l_notepad_client\*.log` | pytracemp 落盘，按天追加 |
| 崩溃 dump | `D:\Temp\Log\l_notepad\crashdumps` | WER LocalDumps 产物（见 §4.2） |

日志根目录可用环境变量 `L_NOTEPAD_LOG_DIR` 覆盖。

## 3. 为什么「崩溃没有任何提示」

要分两层看——**`crash_*.log` 里没有 traceback 不代表没崩**：

1. **Python 层异常** → 被 `sys.excepthook` 捕获，写进 `crash_<日期>.log` 的 `===== 未捕获异常 =====`。
   一般只打印、不致命（Qt 槽里抛出后事件循环继续）。
2. **原生层崩溃（Qt/PySide C++）** → Python 完全抓不到：`sys.excepthook` 不触发，
   `qInstallMessageHandler` 也到不了，`faulthandler` 对 fail-fast / 栈溢出还可能写不出。
   进程直接消失 → **无弹窗、无 traceback**。

### 已知原生崩溃签名（Windows 事件查看器 / CrashDumps 实测）

| 模块 | 异常码 | 含义 | 出现 |
|------|--------|------|------|
| `Qt6Core.dll` | `0xc0000409` | fail-fast（C++ 未定义行为/缓冲区） | 2026-09-10 / 09-17 |
| `Qt6Gui.dll` | `0xc00000fd` | 栈溢出（深递归） | 2026-09-15（多次） |
| `pyside6.abi3.dll` | `0xc0000005` | 访问违例 | 2026-09-15 |
| `SogouPY.ime` | `0xc0000005` | 输入法注入崩溃 | 2026-09-17 |
| `d3d11.dll` | `0xc0000005` | 显卡渲染访问违例 | 2026-09-19 |

> 结论：这类「静默消失」是**原生崩溃**，不是业务代码异常。截至 2026-09-23 **根因未定位**，
> 只能靠 dump 现场继续查。

## 4. 怎么抓原生崩溃现场

### 4.1 Windows 事件查看器

```powershell
Get-WinEvent -FilterHashtable @{LogName='Application'; ProviderName='Application Error'} -MaxEvents 40 |
  Where-Object { $_.Message -match 'python' } |
  Select-Object TimeCreated, @{n='Msg';e={($_.Message -replace "`r`n",' | ')}}
```

`Windows Error Reporting` provider（Id 1001）里有对应的 APPCRASH / BEX64 详情。

### 4.2 WER 崩溃转储（LocalDumps）

默认 `C:\Users\<user>\AppData\Local\CrashDumps\python.exe.*.dmp` 只在部分崩溃生成、且常缺。
要稳定拿 full dump：

```powershell
New-Item -ItemType Directory -Force -Path "D:\Temp\Log\l_notepad\crashdumps"
reg add "HKLM\SOFTWARE\Microsoft\Windows\Windows Error Reporting\LocalDumps\python.exe" /v DumpFolder /t REG_EXPAND_SZ /d "D:\Temp\Log\l_notepad\crashdumps" /f
reg add "HKLM\SOFTWARE\Microsoft\Windows\Windows Error Reporting\LocalDumps\python.exe" /v DumpType   /t REG_DWORD /d 2 /f   # 2 = full
reg add "HKLM\SOFTWARE\Microsoft\Windows\Windows Error Reporting\LocalDumps\python.exe" /v DumpCount  /t REG_DWORD /d 20 /f
```

> **实测（2026-09-23）：HKCU 键位不生效**——写入后强制崩溃仍无 dump；改 **HKLM** 后立即产出
> `python.exe.<pid>.dmp`（约 26 MB）+ 事件查看器 Application Error（退出码 `0xC0000409`）。HKLM 需管理员。
> 验证方法：`python -c "import os; os.abort()"` 应生成 dump。
> **没有「复现时间点 + dump」，根因定位无从谈起。**

### 4.3 faulthandler（仅 Python 层原生信号）

`local_main.py::_install_crash_handlers` 已启用，输出到 `crash_<日期>.log`。
**必须保留目标文件句柄引用**，否则可能被 GC 关闭、崩溃时反而写不出（见 §5）。

## 5. 已修复（2026-09-23）

| 位置（函数名） | 问题 | 修复 |
|------|------|------|
| `ui.py::_collect_folder_entries` | 方法误标 `@staticmethod`，内部却用 `self._file_entry_marker` → 文件夹悬停弹窗抛 `NameError: name 'self' is not defined` | 去掉 `@staticmethod`，改为实例方法 |
| `ui.py::_on_selection_changed_inner` | 选中 `__folder__:*` / `__empty__*` 等非数字 id 时 `int(item_id)` 抛 `ValueError` | 先拦截特殊前缀，非数字字符串直接返回，并加 `try/except` 兜底 |
| `local_main.py::_install_crash_handlers` | `faulthandler.enable(<临时句柄>)` 未持引用，有被 GC 关闭风险 | 用模块级 `_FAULTHANDLER_FILE` 持句柄后再 `enable` |

> 这两处 Python 异常本身只刷日志、不致命，但会**掩盖真正的崩溃信号**，故一并修掉。

## 6. 排查清单（下次窗口又消失时）

1. 看 `crash_<日期>.log`：只有「进程启动」→ 走原生崩溃方向；有「未捕获异常」→ 先按 Python 异常查。
2. 同时刻查事件查看器 `Application Error` / WER，确认出错模块与异常码。
3. 看 `crashdumps` 有无新 `.dmp`；没有就确认 LocalDumps 键位（HKLM）与权限。
4. 按签名判方向：`SogouPY.ime` → 换/更新输入法；`d3d11.dll` → 关 GPU 渲染；`Qt6Gui 0xc00000fd` → 查深递归/大列表构建。
5. 拿到**复现步骤 + dump** 再定根因。