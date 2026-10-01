# wuwo 实例与数据分区（多份安装共存手册）

> 2026-10-01 起：**公共层与 junction 已全部摘除**，老路径兼容改由**一处 resolver** 承担（§3）。
> 面向"同一台机装多份 wuwo / 把整棵树搬盘 / 个人数据放哪"的问题 —— **本文只讲分区概念与目录归属**。
> 操作流程（整棵树搬家全流程、踩坑清单）见 `wuwo_多实例与迁移_手册.md`。
> 实现与实测记录见 `wuwo/doc/CHANGELOG.md` 的 `2026-09-30（二）` 条目。

## 1. 三个区，各绑不同的东西

| 区 | 内容 | 绑什么 | 位置 |
|---|---|---|---|
| **程序区** | trayapp 树、货架、包源码、`wuwo/py_312` | **不绑用户**（每份可装 D: 或 E:） | 相对路径解析（基准 = trayapp 根） |
| **用户数据区** | 个人设置、卡片表、相册 DB、笔记、日志、产物 | **绑用户**（多实例**严格隔离**、换安装盘不丢） | `~/.lugwit/<实例目录>/<包>/` |
| **共享只读大件区** | 模型权重、独立 venv（**只读**） | **绑盘容量/速度**（多实例共用一份） | `E:/lugwit_rez/homes/<包>/` |

**一句话**：安装盘里只放程序；个人设置放 `~/.lugwit/main/…`（主实例）；几十 GB 的权重放
`E:/lugwit_rez/homes/…` 共享只读。

**`~/.lugwit` 根下现在有什么**（2026-10-01 实机核对）：

```
~/.lugwit/
  instances.json          实例注册表（键 ↔ 安装路径 ↔ port_offset ↔ last_seen）
  main/                   主实例目录：各包的真数据（+ instance.json / migration_undo.json）
  run/                    服务发现 json（工具文件，不是包数据；本机根下为空壳 ——
                          服务的 run/ 已随实例落在 <数据根>/run/，如 main/run/script_editor.json）
  config/oriEnvVar.json   托盘的环境变量快照（工具文件）
```

**已取消的两样东西**：

- **公共层**：以前 `~/.lugwit/<包>/` 承担"公共数据 + 只读大件"两层职责；
- **junction**：迁移期曾在老路径留 `mklink /J` 指回实例目录，让不认识新解析的老代码透明 —— **2026-09-30 夜全部删除**，
  2026-10-01 复核确认 `~/.lugwit` 根下**没有 `<包>` 目录、也没有任何 junction**。
  现在数据只有一条路：`l_data_dir` 模板 + **单一 resolver**（§3）；兼容性由 resolver 自带的**老路径兜底档**提供，不再靠链接。

**隔离语义**：类比 Autodesk Maya 的用户数据 —— 一个实例一份，互不可见；只读大件例外（多个实例读同一份，省几十 GB）。

## 2. 配置键（`wuwo/config/config.yaml`）

```yaml
# **键名带 `l_` 前缀 = Lugwit 自己的设置**（避免和各包内部的 data_dir / shared_home_dir 变量混淆）
l_data_dir: "{user}/.lugwit/main/{pkg}"    # 用户数据目录模板：{user} / {instance} / {pkg}
l_shared_home_dir: "E:/lugwit_rez/homes"   # 共享只读大件根；留空 = 不提供（包回落实例目录）
instance: "main"      # 留空 = 自动 inst-<md5(安装目录真路径)[:8]>（分支实例）
                      # 写名字 = 用原名；写 main 就是 ~/.lugwit/main/（主实例，人可读）
data_root: ""         # 留空 = ~/.lugwit/<实例目录名>；也可用 env LUGWIT_DATA_ROOT
port_offset: 0        # 本实例端口 = 卡片端口 + 该值（只给"要并行的那一份"加）
```

- **改名即不兼容**：`data_dir` → `l_data_dir`、`shared_home_dir` → `l_shared_home_dir`。旧名**被直接忽略**（无报错、无回退）。
- **两层配置**：程序层（树内 `wuwo/config/config.yaml`）给默认，用户实例层（`<数据根>/wuwo/config.yaml`）**只写覆盖项**。
- **用户实例层别再写 `l_data_dir`**：写死的旧值会**覆盖程序层**，把数据带回公共路径 ✗。
  本机 `~/.lugwit/main/wuwo/config.yaml` 里那份 stale 旧键**已删除**，只剩 `instance` / `data_root` / `port_offset` 之类的覆盖项。
- **`main` 只给一套安装写** ✗：两套都写会共享同一份数据根（wuwo 会警告、注册表记 `also_seen_at`、`wuwo doctor --instance` 会报）。
- **`port_offset` 给谁加**：想保持标准端口（8090/8475/8476/8470…）的那份留 `0`；要与之**同时运行**的那份加偏移（如 `1000`）。
- **整棵树搬家后想继续用旧数据**：在 `instance` 里写死一个稳定名字（否则 md5 变了就成了新实例）。
- **改 `instance` 要同步 `l_data_dir`**（模板里有硬编码的 `main`），否则 `wuwo doctor --instance` 报不一致。

**盘符不存在会自动回落的两个键**（换机 / 上公网服务器不崩）：

| 键 | 本机值 | 回落 |
|---|---|---|
| `packages.third_party` | `E:/lugwit_rez/rez-package-3rd` | `<trayapp>/rez-package-3rd` |
| `wowo_log_dir` | `E:/lugwit_rez/_logs_e` | `%LOCALAPPDATA%\Lugwit\logs\rez_pkg_log` |

环境变量（`wuwo/py_modules/wuwo_rez.py::_apply_wuwo_env` 启动时注入，临时覆盖用）：

```
LUGWIT_INSTANCE / LUGWIT_DATA_ROOT / LUGWIT_DATA_DIR_TEMPLATE / LUGWIT_SHARED_HOME
LUGWIT_LEGACY_ROOT / LUGWIT_PORT_OFFSET / WUWO_LOCK_DIR / WUWO_ROOT(=LUGWIT_ROOT) / WUWO_THIRD_PARTY_DIR
```

- `LUGWIT_DATA_ROOT` = 本实例数据根，**包只需拼 `/<包名>`**。
- `LUGWIT_DATA_DIR_TEMPLATE` = `l_data_dir` 已展开 `{user}` / `{instance}`，**仍留 `{pkg}` 给包填**。
- `LUGWIT_SHARED_HOME` = `l_shared_home_dir`，**盘符不存在就不注入** —— 包应借此回落到实例目录，而不是去没盘的地方建目录。
- `LUGWIT_LEGACY_ROOT` = `~/.lugwit`，即老路径兜底所在（§3 末档）。

## 3. 用户数据目录：一处实现（resolver），包别自己算

`wuwo/packages/l_app_ready/1.0.0/src/l_app_ready/paths.py` 的 **`pkg_data_dir(pkg)`**：

```python
from l_app_ready.paths import pkg_data_dir as _lugwit_pkg_root
```

解析顺序：

```
LUGWIT_DATA_ROOT           实例数据根（wuwo 环境里走这一档）
LUGWIT_DATA_DIR_TEMPLATE   代入 {pkg}（{user}/{instance} 已在 wuwo 侧展开）
树内 wuwo/config/config.yaml 的 l_data_dir 模板
兜底 ~/.lugwit/<pkg>       ← 老路径；**这一档就是"以前靠 junction 兼容"的替代品**
```

**为什么必须集中**：各包自己算就会漂移（"有的包落 `main`、有的落旧路径"）—— 这正是 2026-09-30 那次
**空目录事故**的根因。现全仓 **39 处** import（17 个包）已统一用它，包内本地实现**全部删除**。
`pkg_data_dir` 的兜底档保证老代码 / 老数据仍在 `~/.lugwit/<包>` 时解析到同一个路径
（已在公网服务器实机验证：那台机器的数据仍在 `~/.Lugwit/<包>`）。

> ⚠️ `l_app_ready` 是 **wuwo 自有包**（`cachable=False`），`paths.py` 是本地增量、不是上游的。
> 升级/重装 wuwo 覆盖该目录的现象是**全树** `ImportError: No module named 'l_app_ready.paths'`；
> 修法 = 把这个文件补回去（只补这一个文件，不用动任何包）。

**junction 已摘除**：NTFS-only 特性（复制/打包/同步要么跟进去重复拷、要么丢）、且会掩盖"谁在写哪个目录"，
所以不再作为兼容手段；保留它只在**迁移/统计**时才需要（`robocopy /XJ`、`os.scandir` + `is_junction()` 计数，
详见 `wuwo_多实例与迁移_手册.md` §5、§7）。货架侧（`rez-package-3rd` 的 pyside6 装配）仍有 junction，与用户数据无关、幂等自愈。

## 4. 包侧 home 阶梯（模型 / 大件包）

```
L_<包>_HOME              env，临时/单次覆盖
<包>/deploy_home.txt     部署级：一行绝对路径（大件常用；不入库、不部署到别的机器）
实例私有目录（存在才用）   <实例数据根>/<包>/
老位置（存在才用）        ~/.lugwit/<包>/      ← 历史数据兜底
实例私有目录（新建）      都没有时新建在这里
```

**"存在才用"是关键**：没写 `deploy_home.txt` 的包行为与以前一致；写了就指到指定盘。

**只读大件与实例隔离分开**：单份权重 18–32GB，按实例各存一份要浪费几十 GB ✗ → **只读**大件共享，
**可写**用户数据才实例隔离（Maya 式）。2026-10-01 已把 `~/.lugwit/*` 里的
`l_wanvideo`（31.85GB）、`l_ltxvideo`（26.48GB）、`l_indextts2`（18.20GB）**剪切**到
`E:/lugwit_rez/homes/<包>`（`l_hunyuanvideo` 本来就在那儿），各包用 `deploy_home.txt`（一行绝对路径）声明。

- 声明文件是**部署级**的：写的是本机盘符，**不入库**、换机器重新生成。
- `l_indextts2` 的独立 venv 剪切后仍可用（`pyvenv.cfg` 的 `home=` 指向 uv 的 base 解释器，与盘符无关）；
  只有 `Scripts\*.exe` shim 烤了旧绝对路径 —— 要装包时重跑一次它的 setup 即可。
- 只读大件**不在 `~/.lugwit` 里**，因此它们天然跨实例共享，且不参与实例迁移。

## 5. 迁移与体检命令

```bat
:: 实例数据分区迁移（默认 dry-run；--apply 真搬；--undo 回滚）
:: 注：wuwo.bat 的分发里已无 instances 子命令 → 直接跑脚本
python wuwo\py_modules\instances_migrate.py [--to <实例名>] [--apply|--undo]

wuwo doctor                                 :: 实例视图 + 路径审计（A 类/B 类）
wuwo doctor --instance                      :: 只报实例与冲突（同路径多键/抢名/同 offset/路径失效/模板不一致）
wuwo doctor --paths                         :: 只报路径审计（搬家前自检：A 类应为 0）
wuwo doctor --fix                           :: 只自动修 config.yaml 里"指回树内"的绝对路径（带 .bak）
wuwo doctor --json                          :: 机器可读
```

**迁移顺序（重要）**：① 停掉持有数据的服务 → ② 迁移（`--apply`，同盘 = 秒级 rename）→ ③ 重启服务。
**不需要**再补 junction（已摘除）——`--apply` 后老路径是空的属正常现象。
**服务不停就搬**：被占用的项会被跳过并报出来（不中断其它项），关掉后重跑即可。

## 6. 实测记录（2026-09-30 ~ 10-01，本机）

- **2026-09-30**：21/22 个私有目录搬入 `~/.lugwit/main/`（`l_notepad_client` 因 GUI 占用未搬，关闭后重跑即补）；
  14 个服务全部恢复在线，**零数据损失**；`wuwo doctor --instance` 显示 `[main] ← 本实例`，无冲突。
- **2026-09-30 夜**：**删除全部 junction**、**取消 `~/.lugwit/<包>` 公共层** → 数据严格按实例隔离。
  旧模型（公共层 + junction 兼容）就此作废，不要再按它操作。
- **2026-10-01**：只读大件从 `~/.lugwit/*` **剪切**到 `E:/lugwit_rez/homes/{l_wanvideo,l_ltxvideo,l_indextts2}`
  （共 **76.5GB**；`l_hunyuanvideo` 本来就在那儿），改由各包 `deploy_home.txt` 声明；
  `l_indextts2` 的独立 venv 搬完仍可用，只有 `Scripts\*.exe` 烤死了旧路径 —— 要装包时重跑它的 setup 即可。
- **2026-10-01 复核**：`~/.lugwit` 根下只剩 `instances.json` + `main/` + 工具文件（`run/`、`config/oriEnvVar.json`），
  **无 `<包>` 目录、无 junction**；注册表 `instances.json` 只有一条 `main`。
- 配置键改名落地：`data_dir` → `l_data_dir`、`shared_home_dir` → `l_shared_home_dir`（**不兼容旧名**）。

## 7. 常见问题

| 现象 | 原因/处理 |
|---|---|
| 第二份安装起不来 / 抢端口 | 两份都用了标准端口 → 给**要并行的那份**设 `port_offset`（1000 之类） |
| `wuwo svc` 操作到了"别人家"的服务 | 两份控制面端口相同且第二份没起主页 → 同上设 offset；或只跑其中一份 |
| 搬家后"个人数据不见了" | md5 变了 → 在 `instance` 写死旧名字；`instances.json` 能查到旧键对应的路径 |
| 包数据落到了 `~/.lugwit/<包>`（没进实例目录） | ① 还在写旧键 `data_dir`（已废弃、被忽略）→ 改 `l_data_dir`；② 包自己算了路径 → 改用 `pkg_data_dir()`（§3） |
| 用户实例层写了 `l_data_dir` 后数据回到老路径 | 用户层只写覆盖项；写死的旧值会覆盖程序层（§2） |
| `ImportError: No module named 'l_app_ready.paths'`（全树） | wuwo 升级覆盖了自有包目录 → 补回这个文件（§3） |
| 还想靠 junction 兼容老代码 | 已摘除（2026-09-30 夜，10-01 复核）：junction 会被复制/同步工具跟进去或直接丢，还掩盖"谁在写哪个目录"。兼容改由 `pkg_data_dir()` 的兜底档提供，改一行 import 即可（§3） |
| 换了实例名 / 数据根，旧数据没跟过来 | 预期行为：**"首次启动询问是否从其他实例复制数据"还没实现**（待办），只能手工搬 |
| `doctor` 报"实例名被多套安装抢用" | 两份 config 都写了同一个名字 → 只保留一份，另一份留空 |
| `doctor` 报 `l_data_dir` 与 `instance` 不一致 | 改了实例名没同步模板 → 两处一起改 |
| `doctor` A 类有命中 | 程序树里写了绝对路径 → 改相对或 `{TRAYAPP}`/`{USER_ROOT}`/`{WUWO}` 占位符 |
| 大件权重被搬走/复制了 | 不该发生：只读大件在共享根（`E:/lugwit_rez/homes`），不参与实例迁移 |
| 日志越堆越大 | `wowo_log_keep_days`（默认 14，每天最多清理一次）；日志根由 `wowo_log_dir` 指定 |
