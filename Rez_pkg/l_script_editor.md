# l_script_editor 使用文档

> 独立可复用的 Python 脚本编辑器组件库：提供代码编辑、自动补全、语法高亮、会话管理，以及**HTTP 远程接口**（远程执行代码、上传/下载文件、暴露 agent 工具、**UI 自动化**——Qt 版 Playwright，全部走脚本编辑器所在进程的 Bottle 后台线程）。

| 项 | 值 |
|----|----|
| 包路径 | `rez-package-source/l_script_editor/999.0` |
| 依赖 | `python-3.12` / `pyside6` / `Lugwit_Module` / `l_qt_wgt_lib` / `l_agent_tool` |
| 启动 | `wuwor l_script_editor -- l_script_editor_demo`（组件演示）<br>`wuwor l_script_editor -- l_script_editor_server`（无 UI 常驻服务）<br>`wuwor l_script_editor -- l_script_editor_test`（运行测试） |
| 测试 | `python src\run_lse_tests.py`（63 用例，offscreen 无头） |

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
# {"status": "running", "server_id": "a1b2c3d4", "editor_available": true, "protocol_version": "1.1", ...}
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
| `/status` | GET | 健康检查（含执行观测：executing / inflight / async_tasks） |
| `/tools` | GET | 发现 agent 工具清单 |
| `/docs` | GET | 交互式 API 文档页（HTML） |
| `/chat` | GET / POST | 代理 `l_agent_chat` 聊天网页与同源 API |
| `/ui/tree` | GET | **UI 自动化**：导出控件树 |
| `/ui/locate` | POST | **UI 自动化**：解析定位器，返回控件信息 |
| `/ui/action` | POST | **UI 自动化**：click / set_text / press_key / select 等动作 |
| `/ui/wait` | POST | **UI 自动化**：auto-wait 等待控件状态 |
| `/ui/screenshot` | GET | **UI 自动化**：控件截图（PNG，JSON base64 或 raw） |

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
{
  "status": "running",
  "server_id": "a1b2c3d4",
  "editor_available": true,
  "protocol_version": "1.1",
  "auth_required": false,
  "file_jail": false,
  "ui_automation": true,
  "executing": false,
  "exec_elapsed_s": null,
  "inflight_available": 8,
  "async_tasks": 0
}
```

`executing` / `exec_elapsed_s` 用于观测「Qt 主线程当前是否被远程代码占用、已占用多久」。

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

> 未知 `request_id` 返回 404；任务过多（默认上限 256）返回 503；已完成任务保留
> `SCRIPT_EDITOR_ASYNC_TTL`（默认 86400s）后自动清理。配合 l_agent_tool 的
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

### 3.9 `/ui/*` — UI 自动化端点（Qt 版 Playwright）

允许外部客户端像 Playwright 驱动网页一样驱动脚本编辑器所在的整个 Qt 窗口：
定位控件、执行动作、等待状态、截图。所有操作在 Qt 主线程执行（复用执行桥接），
天然线程安全；模态对话框嵌套事件循环期间也能响应（queued 信号照常投递）。

#### 3.9.1 定位器（Locator）

所有 `/ui/*` 端点用同一个 locator JSON 描述目标控件：

| `by` | 匹配规则 | 示例 |
|------|----------|------|
| `objectName` | `setObjectName` 设置的名字 | `{"by":"objectName","value":"http_server_btn"}` |
| `text` | 按钮/标签/分组框的可见文字 | `{"by":"text","value":"执行"}` |
| `placeholder` | QLineEdit 占位符 | `{"by":"placeholder","value":"搜索..."}` |
| `accessibleName` | 无障碍名称 | `{"by":"accessibleName","value":"关闭"}` |
| `class` | 元对象类名（精确匹配） | `{"by":"class","value":"QPushButton"}` |
| `type` | Qt 类型（isinstance，支持子类） | `{"by":"type","value":"QComboBox"}` |
| `indexPath` | 自 root 的子控件索引路径 | `{"by":"indexPath","value":"0/2/1"}` |

- 可选 `nth`（默认 0）：多个匹配时取第 N 个，负数从后往前
- 可选 `root`（默认 `"window"`）：`"window"` = 编辑器所在顶层窗口；`"self"` = 编辑器自身；或传入另一个 locator dict 作为搜索范围

#### 3.9.2 `GET /ui/tree` — 控件树

```bash
curl.exe "http://127.0.0.1:8764/ui/tree?root=window&max_depth=12"
```

返回递归控件树（class / object_name / text / value / placeholder / visible /
enabled / checked / rect / children），每层默认截断 200 个子控件。

#### 3.9.3 `POST /ui/locate` — 解析定位器

```bash
curl.exe -X POST http://127.0.0.1:8764/ui/locate ^
  -H "Content-Type: application/json" ^
  -d "{\"by\":\"text\",\"value\":\"HTTP\",\"timeout\":5}"
```

定位超时返回 `{"success": false, "error": "定位超时 ..."}`（HTTP 200）；locator 参数非法返回 400。

#### 3.9.4 `POST /ui/action` — 执行动作

```bash
curl.exe -X POST http://127.0.0.1:8764/ui/action ^
  -H "Content-Type: application/json" ^
  -d "{\"by\":\"objectName\",\"value\":\"name_edit\",\"action\":\"set_text\",\"params\":{\"text\":\"hello\"}}"
```

| action | params | 说明 |
|--------|--------|------|
| `click` / `double_click` / `right_click` | — | 合成鼠标事件（按钮类走 `click()` 保证 toggle） |
| `hover` / `focus` | — | 移入 / 聚焦 |
| `check` / `uncheck` | — | 勾选框状态设置 |
| `set_text` | `{"text": "..."}` | 直接设值（QLineEdit / 文本框 / 下拉框 / 标签） |
| `type_text` | `{"text": "abc"}` | 真实键盘事件逐字输入 |
| `press_key` | `{"key": "Return"}` | 按键，支持 `ctrl+s` / `alt+F4` 组合 |
| `select` | `{"value": "B"}` 或 `{"index": 1}` | 下拉框 / 列表 / Tab 选择 |
| `clear` | — | 清空输入框 |
| `scroll` | `{"dx": 0, "dy": -3}` | 滚轮 |

动作要求控件可见且可用，否则 `success=false`。

#### 3.9.5 `POST /ui/wait` — auto-wait

```bash
curl.exe -X POST http://127.0.0.1:8764/ui/wait ^
  -H "Content-Type: application/json" ^
  -d "{\"by\":\"objectName\",\"value\":\"status_label\",\"state\":\"text\",\"expected\":\"done\",\"timeout\":10}"
```

| state | 说明 |
|-------|------|
| `exists` / `gone` | 出现 / 消失（隐藏或销毁均算 gone） |
| `visible` / `hidden` | 可见性 |
| `enabled` / `disabled` | 可用性 |
| `checked` / `unchecked` | 勾选状态 |
| `text` / `text_contains` | 文本等于 / 包含（需 `expected`） |

轮询期间驱动 Qt 事件循环，异步 UI 变化（QTimer / 信号触发）也能等到。

#### 3.9.6 `GET /ui/screenshot` — 截图

```bash
# JSON base64（默认）
curl.exe "http://127.0.0.1:8764/ui/screenshot?by=objectName&value=http_server_btn"

# 浏览器直接看图
curl.exe -o btn.png "http://127.0.0.1:8764/ui/screenshot?by=objectName&value=http_server_btn&raw=1"

# 整窗截图（不带 locator）
curl.exe -o win.png "http://127.0.0.1:8764/ui/screenshot?raw=1"
```

#### 3.9.7 Playwright 式工作流示例

```python
import requests

base = "http://127.0.0.1:8764"

# 1) 找到执行按钮并点击
requests.post(f"{base}/ui/action", json={
    "by": "objectName", "value": "run_btn", "action": "click"})

# 2) 等输出标签出现"完成"（auto-wait，无需手写 sleep）
requests.post(f"{base}/ui/wait", json={
    "by": "objectName", "value": "status_label",
    "state": "text", "expected": "完成", "timeout": 30})

# 3) 截图存档
img = requests.get(f"{base}/ui/screenshot?by=objectName&value=status_label").json()["image"]
```

---

## 3A. 安全

本服务具备**任意代码执行**能力，安全模型分三层：

### 3A.1 监听地址（默认仅本机）

```python
editor_tab.start_http_server()                        # 127.0.0.1（默认，推荐）
editor_tab.start_http_server(host="127.0.0.1")        # 显式本机
```

### 3A.2 非回环监听守卫

监听局域网（`host="0.0.0.0"` 等）且未配置令牌时，**服务拒绝启动**并打印原因。
跨机器使用必须配置令牌（见下），或显式自担风险：

| 环境变量 | 说明 |
|----------|------|
| `SCRIPT_EDITOR_TOKEN` | 访问令牌。设置后除 `/status` `/docs` 外所有端点要求携带：`X-Editor-Token: <token>` 或 `Authorization: Bearer <token>` 或 `?token=<token>` |
| `SCRIPT_EDITOR_ALLOW_INSECURE` | 设为 `1` 显式跳过非回环守卫（自担风险） |
| `SCRIPT_EDITOR_FILE_ROOTS` | 文件白名单目录（os.pathsep 分隔，Windows 用 `;`）。设置后 `/upload` `/download` `/upload_folder` `/execute` 的文件路径必须落在其中，越界返回 403；未设置则不限制 |

```powershell
# 跨机器安全用法（示例）
set SCRIPT_EDITOR_TOKEN=一串随机长令牌
set SCRIPT_EDITOR_FILE_ROOTS=D:\remote_app;D:\downloads
wuwor l_script_editor -- l_script_editor_server
# 另一台机器：
curl.exe -X POST http://192.168.1.100:8764/execute -H "X-Editor-Token: 一串随机长令牌" ...
```

### 3A.3 其他安全事实

- `/status` 与 `/docs` 免鉴权（仅暴露健康状态与文档，无敏感信息）
- 令牌校验用 `hmac.compare_digest`，防时序攻击
- `/upload_folder` 的 `rel_path` 始终有路径穿越防护（拒绝 `..` / 绝对路径 / 盘符）

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

> 完整 schema 描述以 `/tools` 端点实时返回为准；详见 `Rez-Docs/l_agent_tool使用指南.md`。

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
- **执行超时只是放弃等待**：Qt 主线程上的代码仍在运行，后续请求会排队；`/status` 的 `executing` / `exec_elapsed_s` 可观测。
- **不要在远程代码里同步回调 `/execute`**（同线程自等待会死锁）——桥接会拒绝来自 Qt 主线程的直接提交；请改用后台线程 + 轮询，或 `/execute_async`。
- 异步任务结果保留 `SCRIPT_EDITOR_ASYNC_TTL`（默认 86400s）后自动清理。

---

## 7. 测试

测试为包内 unittest（HTTP 层 + UI 自动化 + 安全，offscreen 无头运行，不弹窗）：

```powershell
# wuwor 环境（依赖齐全）
wuwor l_script_editor -- l_script_editor_test

# 任意装有 PySide6 的 Python 3.12（自动挂载兄弟包源码 / 自动 stub 兜底）
python src\run_lse_tests.py

# 端到端冒烟（真实 ScriptEditorTab + HTTP 服务 + /ui/* 全链路）
python src\smoke_e2e.py
```

覆盖范围（63 用例）：`/execute`（POST/GET/自动上传/异常）、`/execute_async`
（提交/轮询/TTL 清扫）、桥接重入保护、`/ui/*`（tree/locate 7 种 by/action
12 种动作/wait 10 种 state/screenshot）、令牌鉴权（header/Bearer/query）、
文件白名单 jail、非回环绑定守卫。

---

## 8. 组件 API 速览

| 类 / 函数 | 说明 |
|-----------|------|
| `ScriptEditorTab` | 主编辑组件；`start_http_server(host, port)` / `stop_http_server()` / `execute_code_from_api(code)` / `agent` |
| `EditorAgent` | agent 工具调用器（注入执行环境为 `agent` / `tools`）；`list_tools()` / `call(name, ...)` / `register_tool(...)` |
| `ToolRegistry` / `AgentTool` | 工具注册表 / 工具定义 |
| `register_default_tool` / `DEFAULT_TOOLS` | 模块级默认工具注册 |
| `ScriptEditorHttpServer` | HTTP 服务管理器（后台线程；默认 `127.0.0.1`，非回环需令牌） |
| `check_bind_guard` / `_read_token` / `_read_file_roots` | 安全守卫与环境变量解析（`http_server`） |
| `ui_automation` | UI 自动化引擎：`parse_locator` / `resolve_widget` / `dump_widget_tree` / `perform_action` / `wait_for` / `grab_png_b64` |
| `CodeEditorWithCompletion` / `CodeCompleter` | 编辑器 + 自动补全 |
| `PythonHighlighter` / `SyntaxHighlightColors` | 语法高亮 |
| `SessionManager` | 会话管理 |
| `reload_mod()` | 深度重载整个库（含子模块），返回重载后的 `ScriptEditorTab` 类 |
