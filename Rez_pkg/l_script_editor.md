# l_script_editor 使用文档

> 独立可复用的 Python 脚本编辑器组件库：提供代码编辑、自动补全、语法高亮、会话管理，以及**HTTP 远程接口**（远程执行代码、上传/下载文件、暴露 agent 工具，全部走脚本编辑器所在进程的 Bottle 后台线程）。

---

## 1. 快速开始

### 1.1 启动 HTTP 服务

在 `ScriptEditorTab` 实例上启动（监听 `127.0.0.1`，默认端口 `8764`）：

```python
from l_script_editor import ScriptEditorTab

editor_tab = ScriptEditorTab(parent=..., main_window=...)
editor_tab.start_http_server(host="127.0.0.1", port=8764)  # 后台守护线程，不阻塞 UI
```

- 若编辑器自带 UI，点击 **HTTP** 按钮即可开/关服务（按钮变绿 `HTTP ●` 表示已运行）。
- 服务状态会持久化：上次运行则下次自动启动，端口同样记忆。
- 关闭：`editor_tab.stop_http_server()`。

### 1.2 验证服务

```bash
curl.exe http://127.0.0.1:8764/status
# {"status": "running", "server_id": "a1b2c3d4", "editor_available": true}
```

---

## 2. 端口配置

| 项 | 值 |
|----|----|
| 默认端口 | `8764` |
| 环境变量 | `SCRIPT_EDITOR_HTTP_PORT`（如 `set SCRIPT_EDITOR_HTTP_PORT=8764`） |
| 合法区间 | `[8700, 8764]`；缺省 / 非法 / 越界时回退默认端口 |
| 用途 | 并行多实例时各配一个端口，避免绑定冲突 |

> 端口被占用时服务会自动重试，绑定成功后把实际端口同步回 UI（`resolve_http_port` / `http_port_env_override` 负责解析）。

---

## 3. HTTP 接口

| 端点 | 方法 | 用途 |
|------|:---:|------|
| `/execute` | GET / POST | 远程**同步**执行 Python 代码或 `.py` 文件（阻塞等待结果） |
| `/execute_async` | POST | 远程**异步**提交代码执行，立即返回 `request_id`（不阻塞） |
| `/execute_async/result/<id>` | GET | 查询异步执行任务的结果（非阻塞） |
| `/upload` | POST | 把单个文件内容写入远程路径（文本或二进制） |
| `/upload_folder` | POST | 递归上传本地文件夹到远程目录（文件树，跨机器） |
| `/download` | POST | 从远程路径读取文件内容（文本或二进制） |
| `/status` | GET | 健康检查 |
| `/tools` | GET | 发现 agent 工具清单 |
| `/docs` | GET | 交互式 API 文档页（HTML） |
| `/chat` | GET / POST | 代理 `l_agent_chat` 聊天网页与同源 API |

| 方式 | 传参 | 适用场景 |
|------|------|------|
| `GET /execute` | URL 查询参数 `?path=脚本.py&timeout=300` | **最简调用**：路径直接放 URL，空格/中文自动编码解码 |
| `POST /execute` | body `{"file_path": "...", "timeout": 300}` | AI 工作流：先写 `.py` 文件，再 `curl` 执行；可带 `content` 自动上传（见 3.3.1） |
| `POST /execute` | body `{"code": "...", "timeout": 300}` | 一行内联代码 |
| `POST /execute_async` | body `{"code": "...", "timeout": 300}` | **异步非阻塞**：立即返回 `request_id`，不等待执行完（见 3.4.1） |

### 3.1 `GET /status` — 健康检查

```bash
curl.exe http://127.0.0.1:8764/status
```

响应：

```json
{"status": "running", "server_id": "a1b2c3d4", "editor_available": true}
```

### 3.2 `GET /execute` — URL 参数最简调用（推荐）

**路径直接放 URL 查询参数**，空格 / 中文等特殊字符由客户端自动 percent-编码、服务端自动解码，**无需手动转义**：

```bash
# curl（--data-urlencode 自动编码路径中的空格/中文）
curl.exe --get "http://127.0.0.1:8764/execute" ^
  --data-urlencode "path=D:/my dir/中文脚本.py" ^
  --data-urlencode "timeout=300"

# python requests（params 自动编码）
import requests
resp = requests.get("http://127.0.0.1:8764/execute",
                    params={"path": "D:/my dir/中文脚本.py", "timeout": 300})
print(resp.json())

# 浏览器地址栏直接粘（自动编码）
# http://127.0.0.1:8764/execute?path=D:/TD_Depot/.../my_debug.py&timeout=300
```

| 参数 | 必填 | 说明 |
|------|:---:|------|
| `path` | ✅ | 要执行的 `.py` 文件绝对路径；不存在返回 404，非 `.py` 返回 400 |
| `timeout` | ❌ | 执行超时秒数，默认 `300`，合法范围 `[1, 3600]` |

### 3.3 `POST /execute` — body JSON 形式

> **💡 提前说明（跨机器重点）**：`POST /execute` 的 `file_path` 模式支持"**一次调用完成
> 上传 + 执行**"——在请求体里额外带 `content`（本地文件内容），服务端会自动把文件同步到
> `file_path` 后再执行，**无需先调 `/upload`**。详见下文 3.3.1。目标为远端服务器 IP 时尤其常用。

> **💡 提前说明（异步）**：以上 `/execute` 都是**同步阻塞**（等执行完才返回）。若需**不阻塞、
> 提交后继续做别的**，用 `POST /execute_async` 立即拿到 `request_id`、之后轮询结果，详见 3.4.1。

```bash
# 文件路径模式（推荐）
curl.exe -X POST http://127.0.0.1:8764/execute ^
  -H "Content-Type: application/json" ^
  --data-binary "@req.json"
```

```json
// req.json — 文件路径模式
{"file_path": "D:/TD_Depot/Wuzu_dev/.../my_debug.py", "timeout": 300}

// 或一行内联代码模式
{"code": "lprint('hello')", "timeout": 30}
```

> `file_path` **优先于** `code`；两者都空返回 400。底层最终都走同一个 `bridge.submit(code)`。

### 3.3.1 自动上传 / 同步（跨机器，目标为服务器 IP 时）

`POST /execute` 的 `file_path` 模式支持"**一次调用完成 上传 + 执行**"：
在请求体里额外带 `content`（该文件的本地内容），服务端会自动把文件同步到
`file_path` 后再读取执行 —— 无需先调 `/upload`。

| 字段 | 必填 | 说明 |
|------|:---:|------|
| `content` | 视情况 | 本地文件内容（文本）；`is_binary=True` 时视为 base64 |
| `is_binary` | ❌ | `content` 是否为 base64，默认 `false` |
| `mtime` | ❌ | 本地文件修改时间（epoch 秒），用于"日期不一致"判断 |

**同步规则（解决"文件已存在但内容 / 日期不一致"）**：

| 场景 | 行为 |
|------|------|
| `file_path` 在服务器不存在 | 自动 `mkdir + 写盘` 再执行 |
| 服务器已存在、未带 `mtime` | 总是用本次 `content` **覆盖**（确保执行的是交付版本，避免陈旧代码） |
| 服务器已存在、带 `mtime` | 仅当本地 `mtime` **大于**服务器文件 mtime 时覆盖；否则沿用服务器现有文件 |
| 未带 `content` | 不写盘，直接读取服务器现有文件（原行为） |

```json
// 自动上传：服务器缺失或内容不一致都会被同步后再执行
{"file_path": "D:/remote_app/my_debug.py", "content": "print('hello')", "timeout": 300}

// 带 mtime：仅在本地更新时才覆盖服务器旧文件
{"file_path": "D:/remote_app/my_debug.py", "content": "print('v2')", "mtime": 1730000000, "timeout": 300}
```

> 适用：把本地开发好的 `.py` 直接交到远端脚本编辑器所在机器执行，无需先 `/upload`，
> 且能保证执行的是当前最新版本。`GET /execute` 无法携带 body，不支持自动上传。

### 3.4 响应结构

```json
{
  "success": true,
  "stdout": "hello\n",
  "stderr": "",
  "result": null,
  "error": null,
  "request_id": "a1b2c3d4e5f6"
}
```

| 字段 | 说明 |
|------|------|
| `success` | 是否执行成功 |
| `stdout` / `stderr` | 标准输出 / 错误输出 |
| `result` | `execute_code_from_api` 的返回值 |
| `error` | 失败原因；超时返回 `"执行超时 (N秒)"` |

### 3.4.1 `POST /execute_async` — 异步提交代码执行（不阻塞）

与 `POST /execute` 的**同步阻塞**不同，`/execute_async` 提交代码后**立即返回** `request_id`，
代码在服务器后台线程执行，调用方不必等待执行完成，可继续做其他事，之后再轮询结果。

```bash
# 提交（立即返回，不阻塞）
curl.exe -X POST http://127.0.0.1:8764/execute_async ^
  -H "Content-Type: application/json" ^
  -d "{\"code\":\"print('hello')\",\"timeout\":300}"

# -> {"success": true, "request_id": "a7f72c78-05e", "status": "running"}
```

| 参数 | 必填 | 说明 |
|------|:---:|------|
| `code` | ✅ | 要执行的 Python 代码 |
| `timeout` | ❌ | 执行超时秒数，默认 `300`，范围 `[1, 3600]` |

**查询结果**：`GET /execute_async/result/<request_id>`（非阻塞，立即返回当前状态）：

```bash
curl.exe http://127.0.0.1:8764/execute_async/result/a7f72c78-05e
```

- 执行中：`{"request_id": "...", "status": "running"}`
- 已完成：
  ```json
  {
    "request_id": "a7f72c78-05e",
    "status": "done",
    "success": true,
    "stdout": "hello\n",
    "stderr": "",
    "result": null,
    "error": null
  }
  ```

> 未知 `request_id` 返回 404；任务过多（默认上限 256）返回 503。配合 l_agent_tool 的
> `execute_sync` 工具使用（见 5.1.2），或直接 `http_get` 轮询。

### 3.5 编码与文件读取

- `.py` 文件读取编码自动回退：**utf-8 → gb18030**，中文注释/Windows 旧编码文件也能执行。
- 只允许 `.py` 后缀（大小写不敏感），否则 400。

### 3.6 `POST /upload` — 上传文件到远程路径

把本地文件内容发送到远程脚本编辑器，写入到指定路径。**自动创建父目录**。

| 参数 | 必填 | 类型 | 说明 |
|------|:---:|:---:|------|
| `remote_path` | ✅ | string | 远程保存路径（绝对路径） |
| `content` | ✅ | string | 文件内容；`is_binary=true` 时为 base64 字符串 |
| `is_binary` | ❌ | bool | 是否为二进制，默认 `false` |

```bash
# 文本上传
curl.exe -X POST http://127.0.0.1:8764/upload ^
  -H "Content-Type: application/json" ^
  -d "{\"remote_path\":\"D:/project/x.py\",\"content\":\"print('hello')\"}"

# 二进制上传（base64）
curl.exe -X POST http://127.0.0.1:8764/upload ^
  -H "Content-Type: application/json" ^
  -d "{\"remote_path\":\"D:/assets/img.png\",\"content\":\"<BASE64>\",\"is_binary\":true}"
```

```python
# python requests
import requests, base64

# 文本
requests.post("http://127.0.0.1:8764/upload", json={
    "remote_path": "D:/project/x.py",
    "content": open("D:/x.py", encoding="utf-8").read(),
})

# 二进制
requests.post("http://127.0.0.1:8764/upload", json={
    "remote_path": "D:/assets/img.png",
    "content": base64.b64encode(open("D:/img.png", "rb").read()).decode("ascii"),
    "is_binary": True,
})
```

响应：

```json
{"success": true, "remote_path": "D:/project/x.py", "bytes": 1234}
```

### 3.7 `POST /download` — 从远程路径下载文件

| 参数 | 必填 | 类型 | 说明 |
|------|:---:|:---:|------|
| `remote_path` | ✅ | string | 远程文件绝对路径 |
| `is_binary` | ❌ | bool | 是否按二进制下载，默认 `false` |

```bash
# 文本下载
curl.exe -X POST http://127.0.0.1:8764/download ^
  -H "Content-Type: application/json" ^
  -d "{\"remote_path\":\"D:/project/x.py\"}"
```

```python
# python requests
import requests, base64

# 文本
resp = requests.post("http://127.0.0.1:8764/download",
                     json={"remote_path": "D:/project/x.py"})
print(resp.json()["content"])

# 二进制
resp = requests.post("http://127.0.0.1:8764/download",
                     json={"remote_path": "D:/assets/img.png", "is_binary": True})
data = base64.b64decode(resp.json()["content"])
open("D:/img.png", "wb").write(data)
```

响应（文本模式）：

```json
{"success": true, "remote_path": "D:/project/x.py", "bytes": 1234, "is_binary": false, "content": "..."}
```

响应（二进制模式）：

```json
{"success": true, "remote_path": "D:/assets/img.png", "bytes": 1234, "is_binary": true, "content": "<BASE64>"}
```

文件不存在返回 404，非文件返回 400，编码自动回退 utf-8 → gb18030。

### 3.8 `POST /upload_folder` — 上传本地文件夹（文件树）

把**整个本地文件夹**递归上传到服务器，在 `remote_dir` 下重建目录结构。与单文件 `/upload`
一致走 JSON（文本 utf-8 / 二进制 base64），因此**跨机器可用**。

| 参数 | 必填 | 类型 | 说明 |
|------|:---:|:---:|------|
| `remote_dir` | ✅ | string | 服务器端目标目录（绝对路径） |
| `files` | ✅ | array | 文件列表，每项 `{"rel_path","content","is_binary"}` |
| `overwrite` | ❌ | bool | 是否覆盖已存在文件，默认 `false`（跳过） |

- `rel_path`：相对 `remote_dir` 的路径，会做**路径穿越防护**（拒绝 `..`、绝对路径、盘符），必须落在 `remote_dir` 内
- `is_binary=false`（默认）：`content` 为文本，utf-8 写入
- `is_binary=true`：`content` 为 base64，解码后写二进制

```json
// 请求体：本地文件夹 src/（含 sub/b.txt 文本与 data.bin 二进制）
{
  "remote_dir": "D:/server/scripts",
  "files": [
    {"rel_path": "sub/b.txt", "content": "hello"},
    {"rel_path": "data.bin", "content": "<BASE64>", "is_binary": true}
  ]
}
```

```bash
curl.exe -X POST http://192.168.1.100:8764/upload_folder ^
  -H "Content-Type: application/json" ^
  --data-binary "@req_folder.json"
```

响应（逐文件汇报，含跳过的与失败的）：

```json
{
  "success": true,
  "remote_dir": "D:/server/scripts",
  "files": 2,
  "bytes": 1234,
  "written": ["sub/b.txt", "data.bin"],
  "skipped": [],
  "errors": []
}
```

> l_agent_tool 提供配套客户端函数 `upload_folder_to_server(base_url, local_dir, remote_dir, ...)`，
> 自动遍历本地文件夹并按此协议提交。

---

## 4. 远程调试工作流

**最典型用法：先写 `.py` 文件，再一键远程执行验证**（改代码无需冷重启）：

```powershell
# 1. 写好脚本 D:/TD_Depot/.../my_debug.py（自动同步到开发板，无需手动部署）
# 2. 远程执行
curl.exe --get "http://127.0.0.1:8764/execute" `
  --data-urlencode "path=D:/TD_Depot/.../my_debug.py" `
  --data-urlencode "timeout=600"
```

执行环境特性：

- `main_window`、`lprint` **已自动注入**，脚本里可直接用，无需 import。
- 代码在**脚本编辑器所在进程**内执行，可访问进程内对象（如 MuseHelper、各类单例），适合排查运行时状态。
- 深度重载组件：`from l_script_editor import reload_mod; ReloadedTab = reload_mod()` 后重建实例即可热更新整个库。

### 4.1 跨机器文件传输工作流

把本地开发好的脚本/资源推送到远端脚本编辑器所在的机器（同一 HTTP 服务即可，无需额外部署）：

```python
from l_agent_tool import EditorAgent

tools = EditorAgent()
remote = "http://192.168.1.100:8764"   # 远端脚本编辑器 HTTP 地址

# 1) 上传脚本到远端
tools.upload_file(
    local_path="D:/repo/myservice/server.py",
    remote_url=remote,
    remote_path="D:/remote_app/server.py",
)

# 2) 触发远端执行刚上传的脚本
tools.call("run_command", "cd /d D:/remote_app && python server.py", timeout=300)

# 3) 把远端生成的结果拉回本地分析
tools.download_file(
    remote_url=remote,
    remote_path="D:/remote_app/logs/run.log",
    local_path="D:/downloads/run.log",
)
```

或直接在命令行调用：

```powershell
# 上传
curl.exe -X POST http://192.168.1.100:8764/upload ^
  -H "Content-Type: application/json" ^
  --data-binary "@req_upload.json"

# req_upload.json
# {"remote_path":"D:/remote_app/server.py","content":"<脚本内容>"}

# 下载
curl.exe -X POST http://192.168.1.100:8764/download ^
  -H "Content-Type: application/json" ^
  -d "{\"remote_path\":\"D:/remote_app/logs/run.log\"}"
```

---

## 5. Agent 工具系统

脚本编辑器内置 agent 能力：执行环境中注入 **`agent` / `tools`** 命名空间（指向同一个 `EditorAgent`），提供一组**默认工具**并支持自定义注册，供脚本或外部 agent 调用。

### 5.1 默认工具一览

| 工具 | 说明 |
|------|------|
| `list_tools` / `describe_tool` / `has_tool` / `call_tool` | 工具发现与调用 |
| `read_file` / `write_file` | 文本文件读写（utf-8 优先，回退 gb18030） |
| `list_dir` / `find_files` / `search_text` | 目录 / glob / 正则搜索 |
| `run_command` | PowerShell 执行命令（危险命令自动阻断） |
| `kill_port` | 结束占用指定端口的进程 |
| `http_get` / `http_post` / `fetch_url` | HTTP 请求与网页抓取 |
| `upload_file` / `download_file` | 跨机器文件互传（基于 `/upload` / `/download`） |
| `execute_sync` | 远程**异步**提交代码执行（基于 `/execute_async`），立即返回 `request_id`，不阻塞 |
| `upload_folder_to_server` | 把本地文件夹递归上传（基于 `/upload_folder`） |
| `git_available` / `git_execute` / `git_ls_remote` / `git_clone` / `git_init` / `git_commit` / `git_push` | Git 操作 |
| `get_env_var` / `now` / `echo` | 环境变量 / 时间 / 调试 |
| `get_editor_state` / `get_current_code` / `get_all_codes` / `set_code` | 编辑器控制 |
| `run_code` / `inject_vars` / `get_http_port` | 代码执行 / 变量注入 / 端口查询 |

> 完整 schema 描述以 `/tools` 端点实时返回为准；详见 `Doc/l_agent_tool使用指南.md`。

### 5.1.1 跨机器文件传输（`upload_file` / `download_file`）

`upload_file` 把本地文件内容通过 `POST /upload` 推到远程脚本编辑器；`download_file` 通过 `POST /download` 把远程文件拉回本地。文本与二进制都支持。

| 工具 | 参数 |
|------|------|
| `upload_file` | `local_path`（必填）、`remote_url`（必填）、`remote_path`（必填）、`is_binary`（可选，默认 false） |
| `download_file` | `remote_url`（必填）、`remote_path`（必填）、`local_path`（必填）、`is_binary`（可选，默认 false） |

```python
from l_agent_tool import EditorAgent

tools = EditorAgent()
remote = "http://192.168.1.100:8764"

# 上传本地脚本到远程项目
tools.upload_file(
    local_path="D:/my_script.py",
    remote_url=remote,
    remote_path="D:/project/my_script.py",
)
# -> {"success": True, "remote_path": "D:/project/my_script.py", "bytes": 1234}

# 二进制上传（图片、模型等）
tools.upload_file(
    local_path="D:/assets/image.png",
    remote_url=remote,
    remote_path="D:/project/assets/image.png",
    is_binary=True,
)

# 把远程文件拉回本地
tools.download_file(
    remote_url=remote,
    remote_path="D:/project/output.txt",
    local_path="D:/downloads/output.txt",
)
# -> {"success": True, "local_path": "D:/downloads/output.txt", "bytes": 1234}
```

### 5.1.2 远程异步执行（`execute_sync`）与文件夹上传（`upload_folder_to_server`）

**`execute_sync`** 把代码异步提交到远端脚本编辑器执行，**立即返回 `request_id`，不阻塞**。
执行在远端后台线程进行，之后用 `http_get` 轮询 `GET /execute_async/result/<request_id>` 取结果
（或直接对服务器发起同款 HTTP 请求）。

```python
from l_agent_tool import EditorAgent
import time, urllib.request, json

tools = EditorAgent()
remote = "http://192.168.1.100:8764"

# 1) 异步提交（立即返回，不阻塞）
r = tools.execute_sync(remote_url=remote, code="import time; time.sleep(2); print('done')")
# -> {"success": True, "request_id": "a7f72c78-05e", "status": "running"}
request_id = r["request_id"]

# 2) 轮询结果（status 变为 done 即完成）
for _ in range(60):
    with urllib.request.urlopen(f"{remote}/execute_async/result/{request_id}") as resp:
        res = json.loads(resp.read().decode())
    if res.get("status") == "done":
        print(res)   # {"status": "done", "stdout": "done\n", ...}
        break
    time.sleep(0.2)
```

| 工具 | 参数 |
|------|------|
| `execute_sync` | `remote_url`（必填）、`code`（必填）、`timeout`（可选，默认 300） |

**`upload_folder_to_server`** 把整个本地文件夹递归上传到远端 `remote_dir`（基于 `/upload_folder`）：

```python
tools.upload_folder_to_server(
    base_url=remote,          # 服务器地址
    local_dir="D:/my_app",    # 本地文件夹
    remote_dir="D:/server/my_app",  # 远端目标目录
)
# -> {"success": True, "files": n, "written": [...], "skipped": [...], "errors": [...]}
```

| 工具 | 参数 |
|------|------|
| `upload_folder_to_server` | `base_url`（必填）、`local_dir`（必填）、`remote_dir`（必填）、`overwrite`（可选，默认 false） |

### 5.2 在脚本中调用

```python
# 列出所有工具
tools.list_tools()

# 按名称调用（agent 主入口）
tools.call("read_file", "D:/x.py")
tools.call("run_command", "dir")

# 属性式调用（等价于 call）
tools.read_file("D:/x.py")
```

### 5.3 注册自定义工具

```python
@tools.register_tool("double", description="翻倍",
                     parameters=[{"name": "n", "type": "number", "required": True}])
def _double(n):
    return {"result": n * 2}

tools.call("double", 21)   # -> {"result": 42}
```

模块级 `register_default_tool` / `DEFAULT_TOOLS` 可注册全局默认工具（所有编辑器实例共享）。

### 5.4 HTTP 工具发现

```bash
curl.exe http://127.0.0.1:8764/tools
# {"tools": [{"name": "read_file", "description": "...", "parameters": [...]}, ...]}
```

外部 agent 可通过 `/tools` 发现能力，再通过 `/execute` 提交 `tools.call(...)` 完成调用。

---

## 6. 注意事项

- **PowerShell 的 `curl` 是 `Invoke-WebRequest` 别名**，必须用 `curl.exe`；内联 JSON 含特殊字符（`\n`、反斜杠）易解析失败，最稳妥是写入文件用 `--data-binary "@file.json"`。
- `timeout` 设太小时长任务会被提前终止（如扫描上千分类，建议 600s 以上）。
- 服务为后台守护线程运行（bottle + wsgiref 零依赖后端），不阻塞 UI。
- 端口冲突会自动重试绑定；多实例请用 `SCRIPT_EDITOR_HTTP_PORT` 分开端口。
- `/upload` 与 `/download` 走 JSON body，二进制内容必须 **base64 编码**传输，避免 JSON 转义与编码问题。
- `/upload` 会**自动创建父目录**；`/download` 远程文件不存在返回 404、非文件返回 400。
- `upload_file` / `download_file` 底层是同机 HTTP 请求，跨机器时只要远端 HTTP 服务可访问（防火墙放行端口）即可使用。
- `/execute` 为**同步阻塞**（等执行完才返回）；需要"不阻塞、提交后继续做别的"时用 `/execute_async` + 轮询 `/execute_async/result/<id>`。
- `/upload_folder` 的 `rel_path` 有**路径穿越防护**，目标必须落在 `remote_dir` 内；已存在文件在 `overwrite=false` 时默认跳过。

---

## 7. 组件 API 速览

| 类 / 函数 | 说明 |
|-----------|------|
| `ScriptEditorTab` | 主编辑组件；`start_http_server(host, port)` / `stop_http_server()` / `execute_code_from_api(code)` / `agent` |
| `EditorAgent` | agent 工具调用器（注入执行环境为 `agent` / `tools`）；`list_tools()` / `call(name, ...)` / `register_tool(...)` |
| `ToolRegistry` / `AgentTool` | 工具注册表 / 工具定义 |
| `register_default_tool` / `DEFAULT_TOOLS` | 模块级默认工具注册 |
| `ScriptEditorHttpServer` | HTTP 服务管理器（后台线程） |
| `CodeEditorWithCompletion` / `CodeCompleter` | 编辑器 + 自动补全 |
| `PythonHighlighter` / `SyntaxHighlightColors` | 语法高亮 |
| `SessionManager` | 会话管理 |
| `reload_mod()` | 深度重载整个库（含子模块），返回重载后的 `ScriptEditorTab` 类 |

---

## 8. 变更记录

### 2026-09-03

- **知识库工作区页面的「编辑」用的是笔记服务内置的 `<textarea>` 编辑器**（`l_notepad_server` 的
  `templates/web_kb.html`），**不是本组件库**——`.md` 笔记的在线编辑/保存走
  `PUT /api/kb/{kb}/workspace/file`，与本组件的脚本编辑/`/execute` 无耦合。
  本组件（l_script_editor / 8764）仍专用于 Python 脚本的编辑与远程执行。
- 依赖层面：本包 `requires` 未变（python-3.12 / pyside6 / Lugwit_Module / l_qt_wgt_lib / l_agent_tool），
  本次不涉及 py_flow hack 或 watchfiles 变更。


