# wuwo_gui 使用文档

> wuwo 的图形管理界面：把 `doc_pkg` / 按需拉包 / 包同步 / 环境解析 / 货架浏览 / 核心包名单 /
> 解释器清理 / 配置查看收进一个 PySide6 窗口。**界面自身不实现任何命令逻辑** —— 只负责拼参数，
> 真正的活儿交给 `wuwo/py_modules/*.py`，输出实时回显。

| 项 | 值 |
|----|----|
| 包路径 | `wuwo/packages/wuwo_gui/1.0.0` |
| 依赖 | `python-3.12+<3.13` / `pyside6` |
| 启动 | `wuwor wuwo_gui -- wuwo_gui` |
| 托盘入口 | `l_tray` 工具网格「wuwo 管理界面」（`use_rez` + `.soloignore`，改完需重启托盘；重复点不再杀旧、也不自动前置） |
| 自动展开 | **不参与**（家族目录带 `.wuwo_no_auto`，见 §6） |
| 页签 | 包说明文档 / 包管理 / 环境 / 货架 / 核心包 / 解释器维护 / 配置（+ 条件出现的 AYON） |

---

## 1. 快速开始

```bash
wuwor wuwo_gui -- wuwo_gui
```

窗口标题带 wuwo 根路径（`wuwo 管理界面 — <wuwo 根>`），据此确认读的是哪份配置。

---

## 2. 页签总览

| 页签 | 干什么 | 底层命令 |
|------|--------|----------|
| **包说明文档** | 给所有包的 `package.py` / `README.md` 刷统一说明块 | `py_modules/doc_pkg.py` |
| **包管理** | 按需拉包 + 货架间包同步 | `auto_fetch_packages.py` / `sync_package.py` |
| **环境** | 解析一次请求，看解出哪些包、各来自哪个货架，并对比两个请求 | `py_312` 里用 rez API |
| **货架** | 五个货架的家族/版本，标出被遮住的重名包 | 本地扫描（无外部命令） |
| **核心包** | 勾选哪些包参与核心包自动展开 | 增删 `.wuwo_no_auto` 标记（无外部命令） |
| **解释器维护** | 清 `py_312` 里的非核心 pip 包 | `py_modules/clean_interpreter.py` |
| **配置** | 只读展示 `config.yaml` 的关键路径与就绪状态 | 读 yaml |
| **AYON** | 只读看 AYON Server（信息/项目/bundle） | `py_312` 子进程发 HTTP |

---

## 3. 各页细节

### 3.1 包说明文档

源目录可选可浏览（留空 = 用 `config.yaml` 的 `packages.source`），四个动作按钮对应
`--list` / `--check` / `--dry-run` / **写入**，另有 `--no-readme`（只处理 `package.py`）与
`-v`。

- 块内容由 `doc_pkg.py` 的 `KB_LINES` / `SE_LINES` / `PKG_LINES` / `CONVENTION_LINES` 生成，
  改措辞要改那些常量并把 `MARKER_VERSION` +1。
- `--check` 有变更时**退出码为 1**（不是错误，是「有差异」）。

### 3.2 包管理

- **按需拉包**：`--check-only` / 包名 / `--force`。名称可被 `auto_fetch_packages` 按
  「本地已存在 → `l_` 前缀(GitHub) → gitlab 探测 → PyPI 探测」推导。
- **包同步**：`sync_package <包名> [--release|--build|--local] [--version] [--force]`。

### 3.3 环境

两个请求框 A / B，各自出「包 / 版本 / **货架** / 仓库」表，A→B 差异表按
「版本变化 / 仅 A / 仅 B」分类。

- 请求留空 = `wuwor` 的默认展开（即核心包名单）；填了就等于 `wuwor <请求>`。
- **解析不在 GUI 进程里做**：这个 rez 版本没有 `rez env --json`（只有 `-o/--output`），所以起
  `py_312\python.exe` 跑同目录的 `rez_resolve.py`，用 `rez.resolved_context.ResolvedContext`
  取结构化结果，两边只通过 **stdout 上的一行 JSON** 通信（GUI 侧是 pyside6 的 rez 环境，
  没有 wuwo 的 `py_312`；反过来 `py_312` 里没有 Qt）。
- 货架顺序 = `REZ_PACKAGES_PATH` 顺序（`shelving.effective_packages_path`），所以「货架」列
  就是该包被哪个仓库满足。

### 3.4 货架

五货架（core / source / third_party / build / release）的家族数与存在性 + 全部家族的
「包 / 版本 / 说明」，并有 **「被遮住」列**。

rez 按 `packages_path` 顺序找包、**找到即用**：同名家族被靠前货架满足时，后面货架那份
永远不会被解析到。「胜出（并遮住后续货架的同名包）」标在胜出方，「被 X 遮住」标在被遮方。
可勾「只看被遮住的家族」，也可一键打开货架目录或选中家族的 `package.py`。

> 本机实测有一例：`pyfory` 在 `source/pyfory/999.0-py3.12` 与 `third_party/pyfory/1.7.0`
> 同时存在，source 排前 → 实际解析到的是**源码货架那份**。

### 3.5 核心包

表格列出 `wuwo/packages` 下的每个家族（包名 / 版本目录 / `package.py` 的 `description`），
勾选框即「是否参与自动展开」：

- 勾掉 → 在家族目录写 `.wuwo_no_auto`；勾上 → 删掉它。**即时落盘**，下面有操作日志。
- 底部实时显示「当前会被 prepend 的请求」。
- 判据与 `wuwo_rez.py:_get_core_package_names` **完全一致**（含 `os` 家族硬排除 —— 裸 `os`
  会把弱平台绑定提升为强约束，界面上锁定不可勾）。
- **改动对下一次 `wuwo` / `wuwor` 启动生效**（名单是每次 `rez env` 启动时扫的，已运行的环境不受影响）。

### 3.6 解释器维护

`clean_interpreter.py --list / --dry-run / --yes`，`--exe` 自动填 `py_312\python.exe`，支持
额外 `--keep`。只保留 rez / PyYAML / requests / six / packaging 及其闭包，其余（PySide6、
pywin32、qtpy…）视为污染物卸载 —— 业务依赖应由货架提供，不该装进 `py_312`。

### 3.7 配置

只读展示：wuwo 根 / 解释器 / 配置文件 / `packages.*` 五个货架 / 日志目录 / `domain` /
`ayon.server_url`，逐项标「存在 / 缺失」。**要改配置仍去改 `config.yaml`。**

---

## 4. 实现约定

| 约定 | 说明 |
|------|------|
| 子进程统一走 `console.CommandConsole` | `QProcess` + `readyRead` 信号（事件循环内读，不开线程），同一时刻只跑一个命令，带终止/清空 |
| 子进程强制 UTF-8 | 注入 `PYTHONIOENCODING=utf-8` / `PYTHONUTF8=1`，否则 wuwo 脚本的中文输出走 GBK 变乱码 |
| 路径解析 | `paths.py`：先认 `WUWO_DIR`，否则从自身向上找含 `py_modules` 的目录；`config.yaml` 无 `pyyaml` 时退化成「两级缩进」行解析（GUI 跑在 pyside6 的 rez 环境，那儿没有 pyyaml） |
| 表格填充 | 填前关 `setSortingEnabled`，填完再开，避免边填边排序错位 |

---

## 5. 托盘入口

`l_tray` 的 `src/l_tray/config/smallProgramList.yaml` 里有一项：

```yaml
- name: wuwo 管理界面
  packages: [wuwo_gui, .soloignore]
  icon: '{package}/assets/wuwo_gui.svg'
  use_rez: true
  run_args: [wuwo_gui]
```

**2026-09-20 起改用 `.soloignore`**（原 `.solo`）：wuwo 只观测、把 peer 注入 `L_SOLO_PEER_*`，
**不杀旧实例**，接管与否交给包自己决定。注意 `wuwo_gui` 目前**没有**实现 `L_SOLO_IGNORE` 接管，
所以重复点会**开第二个界面窗口**（不再自动前置旧窗口、也不再杀旧）。
两种修饰符的差别见 `../solo_单实例守卫模式.md` §6。
**托盘菜单只在启动时构建，改配置后需重启托盘。**

---

## 6. 为什么它挂 `.wuwo_no_auto`

`wuwo/packages/<家族>/` 下的包会被 `wuwo_rez.py:_get_core_package_names` 无条件 prepend 到
每次 `rez env` 的请求里。`wuwo_gui` 依赖 PySide6，放进自动展开等于给**所有**环境挂上 Qt ——
所以家族目录放一个空标记文件 `.wuwo_no_auto`，扫描时跳过。删掉即恢复自动注入。

> 同目录的 `ayon` 包同理（它是容器工具，见 `ayon.md`）。

---

## 7. 排障

| 现象 | 处置 |
|------|------|
| 「找不到解释器 / 脚本」 | 窗口标题里的 wuwo 根不对 → 检查是否从 `wuwor` 启动（`WUWO_DIR` 由链路注入） |
| 输出中文乱码 | 不该出现（已强制 UTF-8）；若是外部程序自带编码问题，看该程序自身 |
| 环境页报「解析进程没有输出」 | 子进程连 rez 都起不来 → 用 `ayon_where` 同款方式手跑 `py_312\python.exe` 看 stderr |
| 核心包勾选后没生效 | 名单在**下一次** `wuwo` / `wuwor` 启动时才重扫 |
| AYON 页不出现 | `config.yaml` 的 `ayon.server_url` 为空 → 填上重启界面 |

---

## 8. 验证记录

| 项 | 结果 |
|----|------|
| 页签 | 8 个（含条件出现的 AYON）全部构建通过 |
| 核心包页 | 勾掉 `rez_pip_installer` → 标记文件出现、名单变 `['arch','l_app_ready','platform']`；勾回复原 |
| 环境页 | 默认展开 → 11 包（core 4 / third_party 7）；`l_folder_favorites` → 9 包；差异表 14 行（仅 A 8 / 仅 B 6） |
| 货架页 | 扫出 5 货架 206 家族，遮包 1 个（`pyfory`） |
| AYON 页 | `/api/info` 200（`1.16.6+202609071331`）；`/api/projects`、`/api/bundles` 401（未建账号） |
| 启动 | `start wuwor wuwo_gui -- wuwo_gui` 窗口正常，标题为 `wuwo 管理界面 — <wuwo 根>` |

---

## 9. 相关

- 命令语义以 `wuwo/py_modules/*.py` 的 argparse 为准，本界面不做二次解释
- 核心包自动展开机制见 `wuwo/py_modules/wuwo_rez.py` 的 `_get_core_package_names`
- AYON 服务器部署见 `ayon.md`
- 变更记录见 `wuwo/doc/CHANGELOG.md`
