# ayon 本地部署文档

> 把 **AYON Server** 装成本地 rez 独立包：一份改造过的官方 compose（`postgres` + `redis` + `ynput/ayon`），加上一组启停命令，跑在 **podman** 的 WSL 机器里。配套给 `wuwo_gui` 的「AYON」页提供只读数据源。
>
> 只解决「本地把服务器跑起来 + 被本机工具访问」，**不碰依赖解析** —— 依赖仍归 rez / 货架管，AYON 只是被查询的外部系统。

| 项 | 值 |
|----|----|
| 包路径 | `wuwo/packages/ayon/1.0.0` |
| 依赖 | `python-3.12+<3.13`（仅此一项；容器引擎走外部命令，不引 docker-py） |
| 启动 | `wuwor ayon -- ayon_start` |
| 停止 / 清空 | `wuwor ayon -- ayon_stop` / `wuwor ayon -- ayon_down -v` |
| 状态 | `wuwor ayon -- ayon_status`（容器列表 + `/api/info` 探活） |
| 日志 | `wuwor ayon -- ayon_logs [server\|postgres\|redis]` |
| 部署目录 | 包根（`{root}`），可用环境变量 `AYON_DEPLOY_DIR` 改 |
| 服务地址 | `http://127.0.0.1:5000` |
| 已验版本 | AYON `1.16.6+202609071331` / postgres `17` / redis `alpine` |
| 自动展开 | **不参与**（家族目录带 `.wuwo_no_auto`，见 §5.3） |

---

## 1. 快速开始

```bash
# 起服务（首次会拉镜像；ready 后打印地址）
wuwor ayon -- ayon_start

# 看状态：容器 + /api/info 探活
wuwor ayon -- ayon_status

# 打开浏览器建管理员账号（首次）
start http://127.0.0.1:5000
```

`ayon_start` 是**幂等**的：容器已在跑就直接进入就绪等待，不会重复创建。首次启动会经历
`TimeoutError` → `HTTP 503` → 就绪三个阶段（server 起来后要先连上 postgres/redis 才肯回
200），这是正常的，不是错误。

---

## 2. 命令与实现

| 命令 | 作用 | 实现位置 |
|------|------|----------|
| `ayon_start` | `compose up -d` + 等 `/api/info` 就绪 | `cmd_start` → `main.py` |
| `ayon_stop` | `compose stop`（保留数据） | `cmd_stop` |
| `ayon_down [-v]` | `compose down`；`-v` 连 postgres 命名卷一起删 | `cmd_down` |
| `ayon_logs [service] [-n N]` | 跟随日志，可指定 server/postgres/redis | `cmd_logs` |
| `ayon_status` | `compose ps` + 探活（退出码 0=就绪） | `cmd_status` |
| `ayon_where` | 打印全部路径与引擎，排障用 | `cmd_where` |

`package.py` 的 `commands()` 只做三件事：设 `AYON_*` 环境变量、注册上面六个 alias、给出
`{root}`。**六个 alias 都直接用绝对路径调 `main.py`**，所以不需要 `PYTHONPATH`（原因见 §6.2）。

---

## 3. 目录

```
wuwo/packages/ayon/
├── .wuwo_no_auto                     # 不参与核心包自动展开
└── 1.0.0/
    ├── package.py                    # requires / variants 说明 / commands() / 六个 alias
    ├── docker-compose.ayon.yml       # 运行时生成（首次 start 时从 src 复制；见 §6.5）
    ├── .env                          # 运行时生成
    ├── addons/                       # bind mount → /addons（AYON 下载 addon 的地方）
    ├── storage/                      # bind mount → /storage（项目数据）
    ├── logs/
    └── src/ayon_launcher/
        ├── docker-compose.yml        # 模板（真源，改这个）
        ├── main.py                   # 入口：六个命令 + 端口转发 + 就绪探测
        └── __init__.py
```

> 运行时生成物（compose 副本 / `.env` / `addons` / `storage` / `logs`）由 `ensure_deploy()`
> 铺出来，都带 `.gitignore` 忽略。**改 compose 要改 `src/ayon_launcher/docker-compose.yml`**，
> 副本会在下次 `ayon_start` 时按内容比对自动覆盖。

---

## 4. 相对官方 compose 的三处改动

原始文件来自 [ynput/ayon-docker](https://github.com/ynput/ayon-docker/blob/main/docker-compose.yml)：

| # | 改动 | 原因 |
|---|------|------|
| 1 | 去掉 `/etc/localtime:/etc/localtime:ro` | 官方注释里就写了 Windows 要注掉（本机是 Windows + podman WSL 机器） |
| 2 | `addons` / `storage` 的 bind mount 改用 `${AYON_ADDONS_DIR}` / `${AYON_STORAGE_DIR}`，默认 `./addons` `./storage` | 部署根由 launcher 决定，不写死在 compose 里 |
| 3 | 不写死 `version`，沿用官方的三个 tag 变量 | 换版本不用改 compose |

> 官方 `.env.example` **不存在**，默认值都写在 compose 的 `:-` 兜底里，所以本机不需要 `.env`
> —— `.env` 是 launcher 生成给人工改 tag/端口用的。

---

## 5. 与 `wuwo_gui` 的对接

### 5.1 打开「AYON」页

`config.yaml` 的 `ayon.server_url` 填上地址即出现该页签，**留空则整个页签不显示**（不装 AYON
的机器上不会多一个只能报错的页）：

```yaml
ayon:
  server_url: "http://127.0.0.1:5000"
  api_key: ""            # 建完账号再生成 key 填这里
```

也可用环境变量临时覆盖：`WUWO_AYON_URL` / `WUWO_AYON_API_KEY`。

### 5.2 页面看到什么

三个只读 GET：`/api/info`、`/api/projects`、`/api/bundles`。端点表列「状态 / 耗时 / 响应结构 /
错误」，选任意行看原始响应；下方是项目表与 bundle 表（含 addon 数）。

失败分**两级**，别混：

| 现象 | 含义 |
|------|------|
| 状态列「（连不上）」 | 传输层没通（`ConnectionRefused` 等）—— 服务没跑或端口转发断了 |
| 有 HTTP 状态码 | 服务活着，只是这个端点不对（`404`）或没权限（`401`） |

**401 是预期**：本机 AYON 还没建管理员账号，`/api/projects`、`/api/bundles` 都要求鉴权，
只有 `/api/info` 匿名可读。建完账号 + 填 `api_key` 后两张表才有数据。

### 5.3 为什么它挂 `.wuwo_no_auto`

`wuwo/packages/<家族>/` 下的包会被 `wuwo_rez.py:_get_core_package_names` **无条件 prepend
到每次 `rez env` 的请求里**。AYON 是容器工具，不该进每个人的环境 —— 所以家族目录放一个空标记
文件 `.wuwo_no_auto`，扫描时跳过。删掉该文件即恢复自动注入。

---

## 6. 四个真坑（都实测复现过）

### 6.1 `variants` 不能写在 `requires` 里，且分支不能为空

- `requires` 是「**无版本要求、扁平字符串列表**」；变体必须写独立键 `variants`。
  写成 `requires = [[], ["ayon-core"]]` 会直接抛 `PackageMetadataError`（schema 只认 `[str]`）。
- 变体分支也**不能为空**：`variants = [[], [...]]` 里的 `[]` 同样非法。

### 6.2 纯工具包别用 `variants` —— `{root}` 会失效

变体包的 `{root}` 会被解析成**变体请求串**（本包实测为 `python-3.12+<3.13`），于是
`"{root}/src/..."` 拼成废路径，六个 alias 全废：

```
python.exe: can't open file '...\ayon\1.0.0\python-3.12+<3.13/src/ayon_launcher/main.py'
```

`variants` 的语义是「同一版本下按需求切互斥分支」，配合真实 build 流程使用；**不带源码、
不 build 的工具包用它是负收益**。本包最终退回扁平 `requires`。真要多装客户端，
把它加进 `requires` 或单独 `wuwor ayon-core`。

### 6.3 `commands()` 里不能把 `"{root}"` 赋给局部变量

```python
# ✗ 会 RecursionError: maximum recursion depth exceeded
def commands():
    root = "{root}"
    env.AYON_ROOT = root
    alias("ayon_x", 'python "' + root + '/main.py"')

# ✓ 只能内联
def commands():
    env.AYON_ROOT = "{root}"
    alias("ayon_x", 'python "{root}/main.py"')
```

原因：局部变量拿到的是 rez 的 FormatString 对象，再交给 `env.*` / `alias()` 时会被 rex
**反复格式化** —— `rex.py:format_field` 里 `value = self.format(value)` 递归下去。

> 顺带一条：**不要**在 `commands()` 里 prepend `PYTHONPATH`。本包入口全用绝对路径调，
> 不需要靠它找模块，而 `PYTHONPATH` + 自定义变量的组合也会踩上面这条。

### 6.4 `netsh portproxy` 依赖 IP Helper 服务

podman 的端口只发布在 **WSL 虚拟机内部**（`podman port` 显示的 `0.0.0.0:5000` 是容器网络里的
地址），Windows 宿主机并不监听。本机既有做法是 `netsh portproxy` 指到虚拟机 IP。

而 `portproxy` 依赖 **`iphlpsvc`（IP Helper）** 服务 —— 它本是 `Stopped`，规则看着在、
实际全不通（连带本机原有的 5173/5715/8000/27017 几条也一直没生效）：

```cmd
net start iphlpsvc            :: 要管理员
```

`main.py` 的 `ensure_port_forward()` 会在 `ayon_start` / `ayon_status` 探活失败时自动按
**当前**虚拟机 IP 重写转发规则（该 IP 每次 `podman machine start` 都可能变），并区分报出
「权限不足」与「规则写了但仍不通（多半是 iphlpsvc 没跑）」两种情况。

---

## 7. 环境改动（本机做过什么）

### 7.1 `.wslconfig`

`podman machine set --memory` **对 WSL 机器不支持**：

```
Error: changing memory not supported for WSL machines
```

只能在 `~/.wslconfig` 设（**所有** WSL 发行版共用的全局上限）：

```ini
[wsl2]
localhostForwarding=true
memory=8GB
vmIdleTimeout=-1
```

| 键 | 为什么 |
|----|--------|
| `memory=8GB` | postgres 是内存大头；官方推荐 16G，2G 是贴地板。本机 127GiB 有余。改完需 `wsl --shutdown` 才对已有机器生效 |
| `vmIdleTimeout=-1` | **禁用 WSL 空闲关停**。本机出现过 VM 被反复重启（`podman info` 的 `Uptime` 多次归零），容器随之被优雅停掉，表现就是 `127.0.0.1:5000` 时通时断 |

改完统一 `wsl --shutdown` 生效（**会停掉所有容器**，包括别家的）。

### 7.2 容器不会跟着 VM 自动回来

`restart: unless-stopped` 只管**容器进程崩了要重拉**，管不了整个 VM 重启 —— 容器是被正常停掉的
（`Exited (0)`，日志里 `received fast shutdown request` / `Server is shutting down`），不算崩溃。
**VM 重启后要跑一次 `ayon_start`。**

---

## 8. 排障

| 现象 | 原因 / 处置 |
|------|-------------|
| 浏览器 `ERR_CONNECTION_RESET`，`podman ps` 空 | VM 被重启过 → `ayon_start`（§7.2） |
| `127.0.0.1:5000` 不通但 `172.22.51.89:5000` 通（IP 见 §6.4） | 端口转发没建 → `net start iphlpsvc`，再 `ayon_start` 让它重写规则 |
| server 容器 `unhealthy`，API 一直 `503`；redis `redis-cli ping` 是 PONG、postgres 能查库 | server 卡在反复重连 redis → `podman restart ayon-server-1`（实测恢复，两地址都回 200） |
| `ayon_where` 报路径是 `python-3.12+<3.13/src/...` | 变体写法把 `{root}` 弄坏了 → §6.2，退回扁平 `requires` |
| `PackageCommandError ... maximum recursion depth exceeded` | `commands()` 用了局部变量 → §6.3 |

排障三件套：

```cmd
podman ps -a --format "{{.Names}} | {{.Status}}"
podman info --format "MemTotal={{.Host.MemTotal}} Uptime={{.Host.Uptime}}"
wuwor ayon -- ayon_where
```

---

## 9. 数据与备份

| 位置 | 内容 | 说明 |
|------|------|------|
| 命名卷 `ayon_db` | postgres 数据 | `ayon_down -v` 会删它 |
| `addons/` | AYON 运行时下载的 addon | bind mount，**不会被 `down -v` 删** |
| `storage/` | 项目数据 | 同上 |

**清空重来**：`ayon_down -v` → 手动删 `addons/`、`storage/` → `ayon_start`（会重新初始化，包括管理员账号）。

**换 tag / 端口**：改包根 `.env` 里的 `AYON_STACK_SERVER_TAG` / `AYON_STACK_SERVER_PORT`，再
`ayon_down && ayon_start`。端口也走 `AYON_SERVER_PORT` 环境变量。

---

## 10. 验证记录

| 项 | 结果 |
|----|------|
| 三容器 | `ayon-postgres-1` / `ayon-redis-1` / `ayon-server-1` 全部 `Up (healthy)` |
| `/api/info` | 200 → `{"version":"1.16.6+202609071331", ...}` |
| `ayon_status` | 报「就绪（HTTP 200）」，退出码 0 |
| `ayon_start` 幂等 | 重复执行正常，已在跑的容器不重建 |
| `ayon_where` | 六项路径 +项目名 + 地址全部正确 |
| MCP（playwright）访问 | `browser_navigate http://127.0.0.1:5000` → `Page Title: AYON`，前端渲染并跳 `/onboarding`；全页截图正常 |
| `wuwo_gui`「AYON」页 | `/api/info` 200 并解析出版本；`/api/projects`、`/api/bundles` 401（未建账号，符合预期） |

---

## 11. 参考资料

- [ynput/ayon-docker](https://github.com/ynput/ayon-docker) —— 官方 compose 来源
- [AYON Server 本地部署](https://help.ayon.app/articles/2293963-ayon-server-local-deployment)
- [AYON Docker 配置项](https://help.ayon.app/articles/4287499-ayon-docker-configuration-options)
- 变更记录见 `wuwo/doc/CHANGELOG.md`（2026-09-17 两条：新增 `ayon` 包 / VM 重启排查）
