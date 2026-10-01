# wuwo 多实例与迁移手册

> 2026-10-01 起：**公共层与 junction 已全部摘除**，老路径兼容改由**一处 resolver** 承担
> （`l_app_ready.paths.pkg_data_dir()`，§3 / §7）—— 这是本次模型改版的重点。
> 面向"同机多份安装共存 / 整棵树换盘 / 个人数据放哪"的实操文档；本文是**操作手册 + 踩坑清单**。
> 概念与目录归属见 `wuwo_实例与数据分区.md`；实现记录见 `wuwo/doc/CHANGELOG.md` 的 `2026-09-30` 条目。
> 实测环境：2026-09-30 ~ 10-01，D:（机械盘 ST4000NC001）→ E:（NVMe Predator GM7000 4TB）。

## 1. 三个区，各绑不同的东西

| 区 | 内容 | 绑什么 | 位置 |
|---|---|---|---|
| **程序区** | trayapp 树、货架、包源码、`wuwo/py_312` | **不绑用户**（每份可装不同盘） | 相对路径解析（基准 = trayapp 根） |
| **用户数据区** | 个人设置、卡片表、相册 DB、笔记、产物 | **绑用户**（多实例严格隔离、换安装盘不丢） | `~/.lugwit/<实例目录>/<包>/` |
| **共享只读大件区** | 模型权重、独立 venv（**只读**） | **绑盘容量/速度**（多实例共享读一份） | `E:/lugwit_rez/homes/<包>/` |

一句话：**安装盘只放程序；个人设置放 `~/.lugwit/main/…`；几十 GB 权重放 `E:/lugwit_rez/homes/…` 共享只读**。

- **`~/.lugwit` 根下只剩基础设施**（实机核对）：

```
~/.lugwit/
  instances.json          实例注册表（键 ↔ 安装路径 ↔ port_offset ↔ last_seen）
  main/                   主实例目录：各包的真数据（+ instance.json / migration_undo.json）
  run/                    服务发现 json（工具文件；本机根下为空壳，服务的 run/ 已随实例落
                          <数据根>/run/，如 ~/.lugwit/main/run/script_editor.json）
  config/oriEnvVar.json   托盘的环境变量快照（工具文件）
```

- **没有 `<包>` 目录、没有任何 junction**：以前 `~/.lugwit/<包>/` 那份"公共层"（迁移期靠
  `mklink /J` 兼容老代码）**已于 2026-09-30 夜全部删除**，junction 一并摘除。
  `~/.lugwit` 根下现在只剩上表这些东西。为什么不再需要 junction → §7。
- **只读大件也不在这里**：模型权重 / 独立 venv 已于 2026-10-01 剪切到 `E:/lugwit_rez/homes/<包>`（§4）。
- **数据按实例严格隔离**：类比 Autodesk Maya 的用户数据 —— 一个实例一份，互不可见。

## 2. 配置键（`wuwo/config/config.yaml`）

```yaml
# **键名带 `l_` 前缀 = Lugwit 自己的设置**（避免和各包内部的 data_dir / shared_home_dir 变量混淆）。
l_data_dir: "{user}/.lugwit/main/{pkg}"    # 用户数据目录模板：{user} / {instance} / {pkg}
l_shared_home_dir: "E:/lugwit_rez/homes"   # 共享只读大件根（模型权重 / 独立 venv）；留空 = 不提供
instance: "main"      # 留空 = 自动 inst-<md5(安装目录真路径)[:8]>（分支实例）
                      # 写名字 = 用原名；写 main 就是 ~/.lugwit/main/（主实例）
data_root: ""         # 留空 = ~/.lugwit/<实例目录名>；也可用 env LUGWIT_DATA_ROOT
port_offset: 0        # 本实例端口 = 卡片端口 + 该值（只给"要并行的那一份"加）
```

- **改名 = 不兼容**：`data_dir` → `l_data_dir`、`shared_home_dir` → `l_shared_home_dir`。
  旧名**被直接忽略**（不报错、不回退）—— 只留着老键，包就落到兜底路径（`~/.lugwit/<包>`）。
- **两层配置规则**：程序层（树内 `wuwo/config/config.yaml`）提供默认；用户层只写**覆盖项**。
- **用户实例层别再写 `l_data_dir`**：用户层 = `<数据根>/wuwo/config.yaml`（本机 `~/.lugwit/main/wuwo/config.yaml`），
  由 wuwo 首次启动从程序层复制一份、只改你要覆盖的键即可。**在里面再写一份 `l_data_dir` 会覆盖程序层**
  （写死的老值会把数据带回公共路径 ✗）。本机该文件里的 stale 旧键已删除，现在只剩 `instance` / `data_root` / `port_offset` 等覆盖项。
- **`main` 只给一套安装写**：两套都写会共享同一份数据根（wuwo 会警告、注册表记 `also_seen_at`、`doctor --instance` 会报）。
- **端口归属**：想保持标准端口（8090/8475/8476/8470…）的那份留 `0`；要与它**同时运行**的另一份加偏移（如 `1000`）。
- **搬家后想继续用旧数据**：在 `instance` 写死一个稳定名字（否则 md5 变了就成了新实例）。
- **改实例名要同步 `l_data_dir`**：模板里硬编码了 `main`，改 `instance` 忘了改模板 → `wuwo doctor --instance` 会报不一致。

**与数据/盘符有关的另外两个键**（都在 `config.yaml`，**盘符不存在时自动回落**，换机/上公网服务器不会崩）：

| 键 | 本机值 | 盘符不存在时回落到 |
|---|---|---|
| `packages.third_party` | `E:/lugwit_rez/rez-package-3rd` | `<trayapp>/rez-package-3rd` |
| `wowo_log_dir` | `E:/lugwit_rez/_logs_e` | `%LOCALAPPDATA%\Lugwit\logs\rez_pkg_log` |

wuwo 启动时注入的环境变量（`wuwo/py_modules/wuwo_rez.py::_apply_wuwo_env`；临时覆盖用）：

```
LUGWIT_INSTANCE           实例键（config.instance 或 md5）
LUGWIT_DATA_ROOT          实例数据根（如 C:/Users/x/.lugwit/main）；**包只需拼 /<包名>**
LUGWIT_DATA_DIR_TEMPLATE  = l_data_dir 已展开 {user}/{instance}，**仍留 {pkg} 给包填**
LUGWIT_SHARED_HOME        = l_shared_home_dir（**盘符不存在就不注入**，包应回落实例目录）
LUGWIT_LEGACY_ROOT        = ~/.lugwit（老数据 / 老路径兜底所在）
LUGWIT_PORT_OFFSET        端口偏移
WUWO_LOCK_DIR             %TEMP%/lugwit/<实例键>（重启锁 / .solo 守卫，按实例隔离）
WUWO_ROOT(= LUGWIT_ROOT)  程序区锚点（= trayapp 根）
WUWO_THIRD_PARTY_DIR      第三方货架实际落点
```

## 3. 用户数据目录：只有一处 resolver（junction 的替代品）

`wuwo/packages/l_app_ready/1.0.0/src/l_app_ready/paths.py` 的 **`pkg_data_dir(pkg)`** —— 包**不该自己算**这个路径。

> 旧模型是"实例目录 + 在 `~/.lugwit/<包>` 留 junction 兼容老代码"。**junction 已全部摘除**，
> 兼容改由这**一个函数**承担 —— 它自带老路径兜底，所以老代码/老数据照样落到同一处。

解析顺序（第一个能的赢）：

```
LUGWIT_DATA_ROOT            wuwo 注入的实例数据根（正常路径走这一档）
LUGWIT_DATA_DIR_TEMPLATE   代入 {pkg}（{user}/{instance} 已由 wuwo 展开；不在 wuwo 环境时的兜底）
树内 wuwo/config/config.yaml 的 l_data_dir 模板（自己向上找树根）
兜底 ~/.lugwit/<pkg>        ← **老路径，这一档就是"以前靠 junction 兼容"的替代**
```

用法就一行：

```python
from l_app_ready.paths import pkg_data_dir as _lugwit_pkg_root
```

**现状**：全仓 **39 处** import（分布 17 个包：`ChatRoom`、`l_WChat`、`l_agent_chat`、`l_agent_market`、`l_agent_tool`、
`l_indextts2`、`l_log`（6 处）、`l_mindmap_fasthtml`、`l_mindmap_mmd`、`l_model_hub`（4 处）、`l_notepad_server`（3 处）、
`l_qframelesswindow`、`l_repo_sync_gui`、`l_tray`、`lugwit_auth`（5 处）、`lugwit_baidu_netdisk`（4 处）、`lugwit_netdisk_client`）
已经统一用它；原先散在各包里的**本地实现已全部删除** —— 各包自己算路径正是"有的落 `main`、有的落旧路径"
那次空目录事故的根因，边界只允许一处。（`l_homepage` 的 `_runtime_dir()` 是同一套阶梯的另一处落地：
老位置 → 实例根，"存在才用"。）

> **维护约束（踩过）**：`l_app_ready` 是 **wuwo 自有包**（`wuwo/packages/`，`cachable=False`），
> `paths.py` 是我们加的**本地增量**、不是上游的。升级/重装 wuwo 若覆盖该目录，
> 现象是**全树** `ImportError: No module named 'l_app_ready.paths'`；
> 修法 = **把这一个文件补回去**（不需要动其它任何包）。

## 4. 包侧 home 解析阶梯（模型/大件包）

```
L_<包>_HOME              env（临时/单次）
<包>/deploy_home.txt     部署级：一行绝对路径（大件常用）
实例私有目录（存在才用）   <实例数据根>/<包>/   ← 小包/未部署大件的包命中
老位置（存在才用）        ~/.lugwit/<包>/        ← 历史数据兜底
实例私有目录（新建）      都没有时新建在这里
```

**"存在才用"是关键**：没写 `deploy_home.txt` 时行为与以前一致；写了就优先指到指定盘。

**共享只读大件（2026-10-01 实况）**：`~/.lugwit/*` 下的大件已**剪切**到
`E:/lugwit_rez/homes/{l_wanvideo,l_ltxvideo,l_indextts2}`（共 **76.5GB**；`l_hunyuanvideo` 本来就在那儿）。
为什么分开：单份 18–32GB，**按实例各存一份要浪费几十 GB** —— 这类**只读**大件共享，可写的用户数据才实例隔离（Maya 式）。
声明方式 = 包根 `rez-package-source/<包>/<版本>/deploy_home.txt`（一行绝对路径，**部署级文件：不入库、不部署到别的机器**）：

```
E:/lugwit_rez/homes/l_wanvideo        :: deploy_home.txt 的内容 = 一行绝对路径
```

搬完仍"能读"：`l_indextts2` 的独立 venv 直接可用（`pyvenv.cfg` 的 `home=` 指向 uv 的 base 解释器，
与盘符无关），所以搬目录不影响；只有 `Scripts\*.exe`（`pip.exe` 之类 shim）烤死了旧绝对路径
—— **需要装包时重跑一次它的 setup 即可**。

- **`LUGWIT_SHARED_HOME`** = `l_shared_home_dir`，是留给包的入口（**盘符不存在时不注入**）：
  想按共享根拼路径的包读它、拼不出就回落实例目录；当前落地做法是各包 `deploy_home.txt` 直接写绝对路径。

## 5. 整棵树迁移（D: → E: 实操）

### 5.1 迁移步骤

```bat
:: 0) 停掉要迁的那棵树（含托盘，避免两个 watchdog 抢 8090）
:: 1) 主拷贝：/XJ 跳过 junction（避免重复拷 + 死循环），排除 python 缓存
robocopy "D:\...\trayapp" "E:\lugwit\trayapp" /E /XJ /MT:8 /R:0 /W:0 ^
  /NFL /NDL /NP ^
  /XD __pycache__ .pytest_cache .mypy_cache .ruff_cache .pytest_tmp ^
  /XF *.pyc *.pyo /LOG:"%TEMP%\robo_tray.log"

:: 2) 补拷一遍（相同文件自动跳过；被占用漏掉的补上）
robocopy ... /R:1 /W:1 ...（同上）

:: 3) 重生 Scripts/*.exe shim（见 §6.1，必做）
:: 4) 配副本身份：instance / port_offset / wowo_log_dir
:: 5) 无副作用自检：用副本自己的 wuwor 跑一次解析 + import
:: 6) 启动副本托盘，验证服务
```

**实测数据**：29.46GB / 466,581 文件 → 排除后 **452,365**；第二遍只补 **2 个文件**、**0 失败**；
速度 64.6 MB/s。**关键认知：Windows 上运行中的文件禁止写/删、但允许读** → robocopy 能正常拷（不会被“跳过”）。

### 5.2 必须排除什么

| 排除项 | 原因 |
|---|---|
| `__pycache__` / `*.pyc` / `*.pyo` | 编译产物，副本首次运行自动重建，省掉小文件长尾的一大块 |
| `.pytest_cache` / `.mypy_cache` / `.ruff_cache` | 同上 |
| **junction / symlink**（`/XJ`） | 不跟进去才不重复拷、不会绕圈（**用户数据侧已无 junction**；`rez-package-3rd` 里 pyside6 装配的 junction 由装配层自愈，不必手工重建） |

### 5.3 统计文件数要给进度 + 缓存（`os.scandir`）

`os.walk`/`rglob` 在 Windows 上**会走进 junction**（`followlinks=False` 只挡 symlink），
与 robocopy `/XJ` 不一致 → 总数会把 junction 目标重复算（实测 466,581 vs 正确的 452,365）。
用 **`os.scandir`** 才能 `entry.is_junction()` 跳过；顺带 `DirEntry.is_dir()/is_file()` 不做额外 stat。
计数结果缓存到 `%TEMP%\copy_total_cache.json`（键 = 源路径 + 排除集；7 天新鲜）→ 监控窗口秒开。

### 5.4 迁移后必做的三件"隐形事"

1. **`Scripts/*.exe` shim 里的绝对路径**（§6.1）—— 不改的话副本的 `rez.exe` 跑的是**原树**的 python。
2. **包内 `commands()` 的 PYTHONPATH**（§6.3）—— `rez env` 会重建 PYTHONPATH。
3. **用户数据落点**：自查不能再有"包自己算 `~/.lugwit/<包>`"（§3），并核对 `l_data_dir` 模板里的实例名与 `instance` 一致
   （`wuwo doctor --instance` 会报不一致）。

### 5.5 迁移时 `/XJ` 仍要带（junction 已摘除，这只是防御）

用户数据侧已经没有 junction，**但货架/`Lib` 里仍可能有**（`rez-package-3rd` 装配出来的那些）。
`/XJ` 在这个前提下是纯防御：不跟进去才不会重复拷。junction 的完整历史与替代方案见 §7。

## 6. 迁移踩坑清单（都实测踩过）

### 6.1 `Scripts/*.exe` 把解释器路径烤进二进制

pip 生成的 console-script shim（`rez.exe` / `pip.exe` / `pyside6-*.exe` …）内嵌**绝对**解释器路径。
表现：副本里 `rez.exe --version` 报 `Rez 3.3.0 from D:\...\py_312\Lib\site-packages\rez` ✗。
修：`python -m pip install --force-reinstall --no-deps rez==<同版本>` ✓（务必钉版本，别顺手升到新版本）。
同一类问题也适用于 `l_indextts2` 独立 venv 的 `Scripts\*.exe`（搬盘后重跑它的 setup）。

### 6.2 "目标在树内、却用绝对路径写"的链接 → 改名即悬空

实测：`L_Tools/999.0/sys_tool -> D:\...\trayapp\Lib\L_Tools\sys_tool`（**绝对路径**）✗ →
原树改名后，**两棵树里都变成 `files=0`** ✗。
排查手法：`Get-Item <路径> -Force | %{ $_.Attributes -band [IO.FileAttributes]::ReparsePoint; $_.Target }`
＋ 顺着 `Target` 逐级查存在性。
（用户数据侧已无 junction，但货架/`Lib` 里仍可能有，换盘时按此法查。）

### 6.3 `rez env` 会重建 PYTHONPATH —— 包内挂路径必须写进 `commands()`

现象：`l_tray/ins.py: from tool_env import *` → `ModuleNotFoundError`（而 `tool_env.py` 明明在
`<trayapp>/wuwo/py_modules/` 里）。
根因：外部 `set PYTHONPATH=…`（甚至 `wuwor` 之前设的）会被 `rez env` **丢弃**；反证是模型包还要
`child_env()` 专门**剥掉**继承了 PYTHONPATH ✓。
修：在**包的 `package.py` 的 `commands()`** 里挂，且用 `{root}` 相对定位：

```python
env.PYTHONPATH.prepend("{root}/../../../wuwo/py_modules")   # 999.0 → trayapp 根 → wuwo/py_modules
```

### 6.4 用包/上层已经提供好的东西，别自己猜

例：起 ConEmu tab 不要去找 `ConEmuDir`/`ConEmuBaseDir` ✗ —— `conemu` 包已经在 PATH 上提供
**`conemu_lugwit`**（`l_tray/package.py` 的 `start_tray` 就是 `cmd /c conemu_lugwit -title Lugwit /cmd …`），
`/reuse` 因此必然附加到**同一个**窗口。
同理：**用户数据目录只认 `pkg_data_dir()`**（§3），别在包里再写一份解析。

### 6.5 别写死盘符（本仓自己的规矩）

- `wuwo/config/config.yaml`：货架键支持**相对路径**（基准 = trayapp 根）；跨盘才写绝对（如 E: 上的 `third_party` / `homes`）
- 占位符：`{TRAYAPP}` / `{USER_ROOT}` / `{WUWO}`（wuwo 解析路径时展开）
- 入口 `.bat` 用 `%~dp0`（换盘即断的多半就是这里）；`.bat` 必须 **CRLF + 纯 ASCII**
- 非 rez 运行（IDE / 直接 python）走 `pywin32_bootstrap` 的同源解析，不要自己推算 `trayapp/rez-package-3rd`

### 6.6 树根探测：`wuwo/` + （`Lib/` 或 `rez-package-source/`）

包内推导"trayapp 根"的那段（批替换塞进了 **87 个文件**）条件必须是**两个标记目录二选一**：

```python
_lugwit_self = __import__('pathlib').Path(__file__).resolve()
_LUGWIT_ROOT = next((q for q in [_lugwit_self, *_lugwit_self.parents]
                     if (q / 'wuwo').is_dir()
                     and ((q / 'Lib').is_dir() or (q / 'rez-package-source').is_dir())), None)
```

**为什么不能只认 `Lib/`**：公网服务器上**没有 `Lib/` 目录**，旧条件会让它返回 `None`，
后续 `_LUGWIT_ROOT / '...'` 直接 `TypeError`（托盘已在服务器上撞到过）。
`wuwo/` 与货架两个标记缺一就不是一棵可运行的树。

## 7. 兼容层：老路径、老代码怎么处理

**结论：兼容 = resolver 的末两档，不是链接。** 老代码不用改结构，只要改**一行 import**（§3）；
老数据仍在 `~/.lugwit/<包>` 时，`pkg_data_dir()` 会命中兜底档，解析出的还是那一个路径。

- **为什么摘掉 junction**（2026-09-30 夜起，2026-10-01 复核确认干净）：
  ① junction 是 NTFS 特性，复制/打包/同步工具要么跟进去重复拷、要么直接丢，跨机跨版本行为不一致；
  ② 它**掩盖了"到底谁在写哪个目录"** —— 当时正是各包自己算路径 + 公共层共享，
  才出现"有的包落 `main`、有的落旧路径"的空目录事故；
  ③ 多一层间接后，排错时"两个路径看着都对"反而更难定位。
- **现在的替代**：`l_data_dir` 模板（带 `{instance}`）+ **单一 resolver** `pkg_data_dir()`（§3）。
  包级本地实现已全部删除 —— 边界只剩一处。
- **老数据还在老路径的情形照样成立**：解析链末档 = `~/.lugwit/<pkg>`，且模板档在不经 wuwor 的进程里也能生效。
  **已在公网服务器实机验证**（服务器上数据仍留在 `~/.Lugwit/<包>`，新旧代码解析到同一路径 ✓）。
- **junction 知识留作历史**（只在迁移/统计时用得上，别再拿它做兼容手段）：
  - 迁移 `robocopy /XJ` 跳过（§5.1、§5.2）；
  - 数文件数要 `os.scandir` + `entry.is_junction()`，`os.walk`/`rglob` 会走进去重复算（§5.3）；
  - 树内绝对路径链接改名即悬空 → 两棵树都 `files=0`（§6.2）。
- **仍存在的 junction 与用户数据无关**：`rez-package-3rd` 里 pyside6 `shiboken6` 等货架装配用的 junction
  （`wuwo/py_modules/auto_fetch_packages.py` 的 `_ensure_junction`），幂等自愈，不用手工管。
- **未做（别当成已有功能）**：**"首次启动询问是否从其他实例复制数据"仍是待办，尚未实现**。
  现在改 `instance` / `data_root` 等于从零开始一份新数据，要继承旧数据只能手工搬。

## 8. `D:` 残留审计（2026-09-30 实测）

在 E 树里搜 `D:[\\/]TD_Depot|D:[\\/]Temp`：**300+ 命中极不均匀** ✗：

| 类别 | 量级 | 是否在 live 路径 | 处置 |
|---|---|---|---|
| `ChatRoom/**`（每个模块一行 `sys.path.append(r'D:\...\trayapp\Lib')` ✗） | 数百 | ✗（该包服务**未启用**） | **待办**：脚本化替换成 `{TRAYAPP}`/派生路径 |
| `ChatRoom/**/*.bat`（绝对 `python_env\python.exe` ✗） | 数十 | ✗ | 同上（该 `python_env` 目录本身也可能已不存在） |
| `**.log` / `*.bak`（历史日志、备份 ✗） | 大量 | ✗ | 可整类清理（不是代码） |
| 文档/README 里的路径示例 ✓ | 数十 | ✗ | 保留（历史记录 ✓） |
| `Lib/**`（Maya/Houdini/CGTeamwork 集成 ✗） | 数十 | ✗ | 待办（外部 DCC 集成，按需） |
| 我们自己的包 + wuwo + 根脚本 ✓ | 少数 | ✓ | **已修**（`wuwo doctor --paths` A 类清零） |

**判据**：`wuwo doctor --paths` 的 A 类（程序树内绝对路径）应为 **0**。目前该体检只覆盖
`wuwo/**` + 少量包配置文件 → **待办：扩展到 `rez-package-source/*/999.0/**`**（把上表第一二类也纳进来）。

## 9. 命令速查

```bat
:: 实例视图 + 冲突自检（抢名 / 同 offset / 路径失效 / 同路径多键 / l_data_dir 与 instance 不一致）
wuwo doctor --instance
:: 路径体检：A 类（程序树内绝对路径）应 0；B 类（程序树里引用用户数据）应逐条确认
wuwo doctor --paths
:: 只修 config.yaml 里"指回树内"的绝对路径（带 .bak）
wuwo doctor --fix
:: 数据分区迁移（默认 dry-run；--apply 真搬；--undo 回滚）
:: 注：wuwo.bat 的分发里已无 instances 子命令 → 直接跑脚本
python wuwo\py_modules\instances_migrate.py [--to <实例名>] [--apply|--undo]
:: 服务管理（在目标树里执行；连的是本实例主页）
wuwo svc list | status | start | stop | restart | reload <别名>
```

## 10. FAQ

| 现象 | 原因/处理 |
|---|---|
| 第二份安装起不来 / 抢端口 | 两份都用标准端口 → 给**要并行的那份**设 `port_offset`（如 1000） |
| `wuwo svc` 操作到了"别人家"的服务 | 两份控制面端口相同；或第二份没起主页 → 同上设 offset，或只跑一套 |
| 搬家后"个人数据不见了" | md5 变了 → 在 `instance` 写死旧名字；`instances.json` 能查到旧键对应的路径 |
| 包落到了 `~/.lugwit/<包>`（没进实例目录） | ① 还在读旧键 `data_dir`（已废弃、被忽略）→ 改成 `l_data_dir`；② 包自己算了路径 → 改用 `pkg_data_dir()`（§3） |
| 用户层写了 `l_data_dir` 后数据跑回老路径 | 用户实例层只写**覆盖项**，别重复写 `l_data_dir`（写死的老值会盖掉程序层，§2） |
| `ImportError: No module named 'l_app_ready.paths'`（全树） | wuwo 升级覆盖了自有包目录 → 补回 `l_app_ready/.../paths.py`（§3） |
| 还想靠 junction 兼容老代码 | 已摘除（2026-09-30 夜，10-01 复核）：junction 会被复制/同步工具跟进去或直接丢，还掩盖"谁在写哪个目录"。改一行 import 用 `pkg_data_dir()` 即可 —— 它自带老路径兜底（§3 / §7） |
| 换了实例名 / 数据根，旧数据没跟过来 | 预期行为：**"首次启动询问是否从其他实例复制数据"还没实现**（§7），只能手工搬 |
| `doctor` 报"实例名被多套安装抢用" | 两份 `config.yaml` 都写了同一个名字 → 只保留一份 |
| `doctor` 报 `l_data_dir` 与 `instance` 不一致 | 改了实例名没同步模板（模板里有硬编码 `main`）→ 两处一起改 |
| `doctor` A 类有命中 | 程序树里写了绝对路径 → 改相对或 `{TRAYAPP}`/`{USER_ROOT}`/`{WUWO}` |
| 模型包跑到没盘的路径去了 | `deploy_home.txt` / `L_<包>_HOME` 指到了不存在的盘 → wuwo 在**盘符不存在时不注入** `LUGWIT_SHARED_HOME`，包应回落实例目录 |
| 日志越堆越大 | `wowo_log_keep_days`（默认 14，每天最多清理一次）；日志根由 `wowo_log_dir` 指定（E: 不存在 → `%LOCALAPPDATA%\Lugwit\logs\rez_pkg_log`） |
| 托盘起不来、报 `No module named 'tool_env'` | 见 §6.3（包 `commands()` 里挂 `wuwo/py_modules`） |
| 副本的 `rez.exe` 指向原树 | 见 §6.1（重生 shim，钉版本） |
| 服务器上托盘崩在 `TypeError` | 树根探测只认了 `Lib/` → 见 §6.6 |
