# 本机服务发现与 IPC（脚本调用方免端口寻址）

> 2026-09-17 新增。细则见 `l_script_editor.md` §3B；本文是**跨包总览**，回答"本机怎么找到某个服务、怎么不写死端口调它"。

## 1. 为什么要它

本机有多个"带 HTTP 服务的宿主"（脚本编辑器、托盘、网盘客户端…）。历史上调用方把端口写死，而端口会**漂移**（首选端口被占就向上找空闲），于是：

- 端口一漂，写死的调用方全失效
- 多实例/多应用互相抢端口（实测出现 8764/8766/8768/8769 一路占过去）
- 固定端口是权宜之计：能治"漂移"，但治不了"谁该占哪个"

**另一个硬约束**：浏览器（含 QtWebEngine 承载的网页）**只能 HTTP over TCP**，连不了命名管道 —— 所以最终形态是**双栈**：页面走 TCP（固定端口 + 发现文件），本机脚本/AI 走 IPC（零端口依赖）。不是替代关系。

## 2. 服务发现（半 IPC）

服务启动后发布一个约定文件，调用方读它，不读端口：

```
~/.Lugwit/run/<service>.json          # 目录可用 LUGWIT_RUN_DIR 改
{"service":"netdisk_client","url":"http://127.0.0.1:8769","scheme":"http",
 "host":"127.0.0.1","port":8769,"listen":"0.0.0.0","pid":12345,
 "started_at":"2026-09-17T17:00:00","has_token":false,
 "ipc":"\\\\.\\pipe\\lugwit-netdisk_client-8769-<user>"}
```

- 服务名 = `SCRIPT_EDITOR_SERVICE_NAME`（默认 `script_editor`）
- 停止时删除；读取时按 pid 判僵尸并自动清理
- ⚠️ 监听通配地址（`0.0.0.0`/`::`）时 `url`/`host` **归一为 `127.0.0.1`**（可连地址），真实监听面记在 `listen` —— `0.0.0.0` 当目标去连在 Windows 上会直接失败
- 读法：

```cmd
python -m l_script_editor.service_registry                    :: 列全部服务
python -m l_script_editor.service_registry netdisk_client     :: 打印该服务 url
```

## 3. IPC（命名管道 / UDS）

- Windows：`\\.\pipe\lugwit-<service>-<port>-<user>`；其他平台：`<run_dir>/<service>-<port>.sock`
- 名字**带端口** → 同名服务多实例并存不抢管道（早期不带端口撞过 `[WinError 5] 拒绝访问`）
- 实现：stdlib `multiprocessing.connection`（AF_PIPE，**不需要 pywin32**）+ **authkey 鉴权**（默认 `lugwit-<service>`，可用 `SCRIPT_EDITOR_PIPE_KEY` 覆盖）
- **接口与 HTTP 完全一致**（`/status`、`/execute`、`/upload`…），只换传输层；`SCRIPT_EDITOR_PIPE=0` 关闭
- 调用方寻址规则：**先读发现文件的 `ipc` 字段**，没有发现文件时按「服务名 + 端口」推导

```cmd
python -m l_script_editor.pipe_bridge list
python -m l_script_editor.pipe_bridge --service netdisk_client --path /status
python -m l_script_editor.pipe_bridge --service netdisk_client --path /execute --json "{\"code\":\"print(1)\"}"
```

```python
from l_script_editor import pipe_bridge, service_registry
status, headers, body = pipe_bridge.request("netdisk_client", "POST", "/execute",
                                            body=b'{"code":"print(1)"}',
                                            headers={"Content-Type": "application/json"})
base = service_registry.resolve_url("netdisk_client")   # 需要走 HTTP 时用它（含回退）
```

## 4. 当前端口分配（固定端口 + 严格模式）

| 用途 | 端口 | 服务名 | 由谁设定 |
|---|---|---|---|
| 独立脚本编辑器服务（托盘菜单拉起 `l_script_editor_server`） | **8768** | `tray` | `l_tray/package.py` 设变量，`Tray.py` 经 `start_rez_package(env_overrides=…)` **透传给子进程** |
| 网盘客户端（内嵌服务） | **8769** | `netdisk_client` | `lugwit_netdisk_client/package.py` |
| 其他/调试实例 | 自定 | 建议专属名（如 `debug`） | 启动前设 `SCRIPT_EDITOR_HTTP_PORT` / `_STRICT` / `SERVICE_NAME` |

严格模式（`SCRIPT_EDITOR_HTTP_PORT_STRICT=1`）：被占时**不漂移** —— 控制台列出占用进程并询问 `[1] 结束 / [2] 放弃`，5 秒无输入即放弃并打印详情（`SCRIPT_EDITOR_HTTP_PORT_PROMPT_TIMEOUT` 可调）；无交互终端直接按"放弃"处理。

> ⚠️ 子进程要点：托盘的脚本服务是**子进程**，其环境来自 `l_script_editor` 而非 `l_tray` —— 只在 `l_tray/package.py` 里设变量是**不生效的**，必须显式透传。这是本次踩过的坑。

## 5. 什么时候用哪个

| 调用方 | 用哪个 | 说明 |
|---|---|---|
| 本机脚本 / AI / 自动化 / 运维工具 | **IPC** | 零端口依赖，最省心；管道名与地址从发现文件拿 |
| 浏览器 / WebEngine 页面 / 跨机工具 | **HTTP over TCP** | 只能用 TCP；地址从发现文件拿（或走 nginx 网关 `https://121.196.144.88/script_editor`） |
| 需要 token 的实例 | 两者都要带凭据 | 回环访问通常免 token；跨机必须带 `X-Editor-Token` |
