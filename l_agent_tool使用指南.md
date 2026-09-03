# l_agent_tool 使用指南

## 概述

`l_agent_tool` 是 Agent 工具库，提供可复用的 agent 能力，供脚本编辑器等模块复用。

## 包信息

| 属性 | 值 |
|------|-----|
| 包名 | `l_agent_tool` |
| 版本 | `999.0` |
| 作者 | Lugwit Team |
| 依赖 | `l_git`, `python-3.12+<3.13` |

## 主要功能

### 默认工具集

`l_agent_tool` 提供以下默认工具（不依赖编辑器实例）：

#### 文件操作

| 工具名 | 功能 | 参数 |
|--------|------|------|
| `read_file` | 读取文本文件内容（utf-8 优先，失败回退 gb18030） | `path`（必填）, `max_chars`（可选，默认 6000） |
| `write_file` | 写入文本文件（自动创建父目录） | `path`（必填）, `content`（必填） |
| `list_dir` | 列出目录内容 | `path`（必填）, `limit`（可选，默认 120） |
| `find_files` | 按 glob 模式递归查找文件 | `pattern`（必填）, `path`（可选） |
| `search_text` | 按正则表达式在文件内容中搜索 | `pattern`（必填）, `path`（可选）, `include`（可选） |

#### 命令执行

| 工具名 | 功能 | 参数 |
|--------|------|------|
| `run_command` | 在 PowerShell 中执行命令（危险命令自动阻断） | `command`（必填）, `timeout`（可选，默认 60） |
| `kill_port` | 结束占用指定端口的进程树（taskkill /T + 僵尸 socket 后代清理 + 独占绑定验证，返回含 `port_free`） | `port`（必填） |

#### HTTP 请求

| 工具名 | 功能 | 参数 |
|--------|------|------|
| `http_get` | 发起 HTTP GET 请求 | `url`（必填）, `timeout`（可选，默认 10） |
| `http_post` | 发起 HTTP POST 请求（JSON body） | `url`（必填）, `payload`（可选）, `timeout`（可选，默认 10） |
| `fetch_url` | 抓取网页内容（纯文本） | `url`（必填）, `max_chars`（可选，默认 20000） |

#### Git 操作

| 工具名 | 功能 | 参数 |
|--------|------|------|
| `git_available` | 检查本机 git 是否可用 | 无 |
| `git_execute` | 执行 git 子命令 | `args`（必填）, `cwd`（可选）, `timeout`（可选，默认 120） |
| `git_ls_remote` | 查询远端仓库引用（ls-remote） | `repo_url`（必填）, `ref`（可选）, `timeout_sec`（可选，默认 30） |
| `git_clone` | 克隆远端仓库到本地路径 | `url`（必填）, `path`（必填）, `timeout`（可选，默认 300） |
| `git_init` | 在指定目录初始化 git 仓库 | `path`（必填）, `branch`（可选，默认 main） |
| `git_commit` | 提交当前仓库暂存区变更 | `path`（必填）, `message`（必填） |
| `git_push` | 推送当前分支到远端 | `path`（必填）, `branch`（必填）, `remote`（可选，默认 origin）, `timeout`（可选，默认 300） |

#### 其他工具

| 工具名 | 功能 | 参数 |
|--------|------|------|
| `upload_file` | 上传本地文件到远程脚本编辑器 HTTP 服务，支持文本和二进制 | `local_path`（必填）, `remote_url`（必填）, `remote_path`（必填）, `is_binary`（可选，默认 False） |
| `download_file` | 从远程脚本编辑器 HTTP 服务下载文件到本地，支持文本和二进制 | `remote_url`（必填）, `remote_path`（必填）, `local_path`（必填）, `is_binary`（可选，默认 False） |
| `now` | 返回当前时间（ISO 格式） | 无 |
| `get_env_var` | 读取环境变量 | `name`（必填） |
| `echo` | 原样返回传入的参数（调试用） | `args`（可选） |

### 编辑器专属工具

当 `EditorAgent` 绑定编辑器实例时，额外提供以下工具：

| 工具名 | 功能 | 参数 |
|--------|------|------|
| `get_editor_state` | 获取编辑器状态（HTTP 端口/运行状态/tab 数/当前代码长度） | 无 |
| `get_current_code` | 获取当前 tab 的代码内容 | 无 |
| `get_all_codes` | 获取所有 tab 的代码内容 | 无 |
| `set_code` | 替换当前 tab 的代码内容 | `code`（必填） |
| `run_code` | 在脚本编辑器执行环境中运行 Python 代码 | `code`（必填） |
| `inject_vars` | 注入变量到执行环境 | `var_dict`（必填） |
| `get_http_port` | 获取 HTTP 服务端口 | 无 |

### 元工具

| 工具名 | 功能 |
|--------|------|
| `list_tools` | 列出所有可用工具 |
| `describe_tool` | 查看某个工具的 schema |
| `has_tool` | 判断工具是否存在 |
| `call_tool` | 按名称调用工具 |

## 典型用法

### 在脚本编辑器中执行

```python
from l_agent_tool import EditorAgent, DEFAULT_TOOLS

# 列出所有可用工具
tools = EditorAgent()
names = [t["name"] for t in tools.list_tools()]
print(f"共 {len(names)} 个工具: {', '.join(names)}")

# 按名称调用工具
result = tools.call("read_file", "D:/x.py")
print(result)

# 直接属性式调用
result = tools.read_file("D:/x.py")

# 注册自定义工具
@tools.register_tool("double", description="翻倍", parameters=[{"name": "n", "type": "number", "required": True}])
def _double(n):
    return {"result": n * 2}

result = tools.call("double", 21)

# 上传本地文件到远程脚本编辑器 HTTP 服务
result = tools.upload_file(
    local_path="D:/my_script.py",
    remote_url="http://192.168.1.100:8764",
    remote_path="D:/remote_project/my_script.py",
)
print(result)
# 成功时: {"success": True, "remote_path": "D:/remote_project/my_script.py", "bytes": 1234}

# 二进制文件上传
result = tools.upload_file(
    local_path="D:/assets/image.png",
    remote_url="http://192.168.1.100:8764",
    remote_path="D:/remote_project/assets/image.png",
    is_binary=True,
)

# 从远程下载文件到本地
result = tools.download_file(
    remote_url="http://192.168.1.100:8764",
    remote_path="D:/remote_project/my_script.py",
    local_path="D:/downloads/my_script.py",
)
print(result)
# 成功时: {"success": True, "local_path": "D:/downloads/my_script.py", "bytes": 1234}

# 二进制文件下载
result = tools.download_file(
    remote_url="http://192.168.1.100:8764",
    remote_path="D:/remote_project/assets/image.png",
    local_path="D:/downloads/image.png",
    is_binary=True,
)
```

### 绑定编辑器实例

```python
from l_agent_tool import EditorAgent

# 传入编辑器实例以获取编辑器专属工具
tools = EditorAgent(editor=editor_instance)
tools.run_code("print('hello')")
```

## 依赖该包的包

以下包依赖 `l_agent_tool`：

| 包名 | 用途 |
|------|------|
| `l_script_editor` | Python 脚本编辑器组件库，使用 agent 工具实现代码执行和文件操作能力 |

## 环境变量

| 变量名 | 说明 |
|--------|------|
| `PYTHONPATH` | 自动添加 `{root}/src` |
| `L_AGENT_TOOL_ROOT` | 指向包根目录 |

## 别名

```bash
# 列出 l_agent_tool 默认工具
l_agent_tool_list
```

## 安全特性

- `run_command` 工具会自动阻断危险命令（如 `rm -rf`, `format` 等）
- 文件操作支持自动编码检测（utf-8 → gb18030 回退）
- 命令执行默认超时控制
