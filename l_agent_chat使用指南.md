# l_agent_chat 使用指南

## 概述

`l_agent_chat` 是本地 AI 编码 Agent 聊天服务：FastAPI 提供 Web UI + SSE 流式对话，
可调用本地工具集（文件/命令/Git/HTTP）辅助编码，支持对话存储、远程工具服务、
上下文压缩与 token 统计。模型走 OpenAI 兼容 Chat Completions 接口（默认 DeepSeek）。

## 包信息

| 属性 | 值 |
|------|-----|
| 包名 | `l_agent_chat` |
| 版本 | `999.0` |
| 作者 | Lugwit Team |
| 依赖 | `python-3.12+<3.13`, `fastapi`, `uvicorn`, `jinja2`, `requests`, `psutil`, `l_agent_tool` |
| 默认端口 | `1250` |
| 默认模型 | `deepseek-v4-flash` |

## 启动

进入 rez 环境并启动服务：

```bat
wuwo rez env l_agent_chat -- l_agent_chat
```

或调用包 alias（相当于上面的命令）：

```bat
wuwo rez env l_agent_chat -- l_agent_chat -y
```

`-y`：端口被占用时自动结束占用进程，不询问。

带重载（推荐开发用）：`l_agent_chat` 的启动器 **默认已开启 uvicorn 热重载**
（`reload=True`，监视 `src/l_agent_chat` 源码目录），改源码保存即自动重启生效，
**无需手动重启服务**。因此 `.dev_mod` 修饰符对该包是多余的——它只对
「默认关、带 `.dev_mod` 才开」的其它后端（auth/netdisk/note/chat）有意义。

若仍显式带 `.dev_mod`，语法必须跟在包名后：

```bat
wuwo rez env l_agent_chat .dev_mod -- l_agent_chat
```

访问：浏览器打开 `http://127.0.0.1:1250`。

## 主要功能

### 聊天（SSE 流式）

`POST /api/chat`，请求体：

```json
{
  "messages": [{"role": "user", "content": "帮我写一个函数..."}],
  "model": "deepseek-v4-flash",
  "temperature": 0.1,
  "stream": true,
  "max_tool_steps": 3,
  "thinking": true,
  "reasoning_effort": "high",
  "show_reasoning": true
}
```

SSE 事件类型：

| 事件 | 说明 |
|------|------|
| `tool_start` | 开始执行工具，附 `tool`/`path`/`command` |
| `tool_result` | 工具结果文本 |
| `tool_error` | 工具执行异常 |
| `tool_rejected` | 用户拒绝了需人工审批的工具 |
| `approval_required` | 请求人工审批，附 `approval_id`/`tool`/`args`/`diff`（写文件时含 diff 预览） |
| `reasoning` | 思考过程（仅 `show_reasoning=true` 时透出） |
| `delta` | 回复增量 |
| `done` | 结束，附完整 `reply` |
| `error` | 出错信息 |

人工审批：工具执行前对 `APPROVAL_TOOLS` 中的工具发 `approval_required` 事件，
前端用户确认后回调 `POST /api/tool-approval`（`{approval_id, approved}`）。

### 工具调用

`l_agent_chat` 复用 `l_agent_tool` 的工具集（文件/命令/Git/HTTP），流程分两种：

- **快速路径**：`run_fast_tools` 直接解析用户消息命中简单工具（读文件/列目录等），
  不触发危险工具。
- **Planner 路径**：`_planner_action` 让模型逐步规划，最多 `max_tool_steps` 步；
  命中 `APPROVAL_TOOLS`（write_file/run_command/kill_port/git_push）时先审批。

### 上下文压缩

当估算 token 超过 `CONTEXT_LIMIT_TOKENS * COMPRESS_THRESHOLD` 时，把早期对话历史
交给 LLM 压缩成中文摘要，保留最近 `COMPRESS_KEEP_RECENT` 轮，避免上下文超限。

### Token 统计

每次 LLM 调用的 `usage`（prompt/completion/total tokens）累计到当前会话 `stats`
字段，可通过 `/api/session/current` 读取，含每次调用明细（`calls`）。

### 会话管理

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/session/current` | GET | 当前会话 + 消息 + 会话列表 + token 统计 |
| `/api/session/new` | POST | 新建会话 |
| `/api/session/switch` | POST | 切换会话 |
| `/api/session/delete` | POST | 删除会话 |
| `/api/session/rename` | POST | 重命名会话 |

会话文件存储于 `<工作区>/.l_agent_ws/sessions/session_<id>.json`。

### 远程工具服务

`l_agent_chat` 可连接外部同类服务（如远程脚本编辑器 `121.196.144.88:8764`），
在设置页添加并激活后，Agent 走远程工具服务执行。

| 端点 | 说明 |
|------|------|
| `GET /api/tool-services` | 列出服务 |
| `POST /api/tool-services` | 创建服务 |
| `POST /api/tool-services/activate` | 激活服务 |
| `POST /api/tool-services/test` | 测试连通 |
| `POST /api/tool-services/invoke` | 调用远端服务的一个工具 |
| `POST/DELETE /api/tool-services/<sid>` | 更新/删除 |

### 其他端点

| 端点 | 说明 |
|------|------|
| `GET /` | 聊天网页 |
| `GET /health` | 健康检查 |
| `GET /api/models` | 模型列表 |
| `GET/POST /api/settings` | 读取/更新运行配置 |
| `GET /api/workspace` | 工作区信息 |
| `POST /api/workspace` | 设置工作区 |
| `GET/POST /api/workspace/root` | 工作区根目录管理 |
| `POST /api/workspace/root/activate` | 激活根目录 |
| `GET /api/browse` | 浏览目录 |
| `GET /api/browse_rez` | 多级浏览 rez 包仓库 |
| `POST /api/translate` | AI / 免费翻译 |

## 配置项

设置页写入的配置存于 `<工作区>/.l_agent_ws/settings.json`，重启后仍生效。
优先级：环境变量 > 设置文件 > 内置默认。

| 配置键 | 环境变量 | 默认 | 说明 |
|--------|----------|------|------|
| `api_url` | `AGENT_CHAT_API_URL` | `https://api.deepseek.com/chat/completions` | OpenAI 兼容接口 |
| `api_key` | `DEEPSEEK_API_KEY` | `""` | API 密钥（不硬编码内置） |
| `default_model` | `AGENT_CHAT_MODEL` | `deepseek-v4-flash` | 默认模型 |
| `host` | `AGENT_CHAT_HOST` | `127.0.0.1` | 监听地址（局域网设 `0.0.0.0`） |
| `port` | `AGENT_CHAT_PORT` | `1250` | 服务端口 |
| `temperature` | — | `0.1` | 采样温度 |
| `timeout` | — | `120` | 请求超时（秒） |
| `max_history_turns` | — | `20` | 携带历史轮数 |
| `max_tool_steps` | — | `3` | 工具规划最大步数 |
| `context_limit_tokens` | — | `6000` | 上下文压缩阈值（估算 token） |
| `compress_threshold` | — | `0.85` | 达到阈值比例触发压缩 |
| `compress_keep_recent` | — | `6` | 压缩时保留最近轮数 |
| `translator_backend` | `AGENT_CHAT_TRANSLATOR` | `baidu` | 翻译后端（baidu/mymemory/ai） |
| `rez_roots` | `AGENT_CHAT_REZ_ROOTS` | 源码+3rd 仓库 | rez 包仓库根（`;` 分隔） |

## 依赖该包的包

| 包名 | 用途 |
|------|------|
| `start_multi_app` | 多应用启动（引入 `l_agent_chat`） |

## 端口占用处理

启动器启动前会检查端口：被占用时打印 PID/进程名/exe 路径并询问是否结束；
`-y` 直接结束。会剔除 netstat/psutil 中的僵尸 LISTENING 残留，并清理
reload 模式的 supervisor + worker 双进程树。

Windows 独占绑定防共享（治本）：

- uvicorn 默认给监听 socket 设 `SO_REUSEADDR`，Windows 下允许多进程共享绑定
  同一端口——旧进程残留时请求会被路由到旧代码（"改了源码不生效"的元凶）。
- 启动器在调用 `uvicorn.run` 前打补丁（`_patch_uvicorn_exclusive_bind`），
  把监听 socket 改为 `SO_EXCLUSIVEADDRUSE` 独占绑定：端口被占用时直接报错，
  绝不静默共享。reload 模式 socket 由 supervisor 创建一次传给 worker，
  worker 重启复用同一 socket，无重新绑定竞态。
- 端口探测 `_port_bindable` 同样用独占绑定，避免 `SO_REUSEADDR` 误判"已释放"。
- 僵尸 socket（netstat 显示 LISTENING 但 PID 已死，句柄被后代继承持有）：
  自动找出死 PID 的存活后代并结束（`_clear_zombie_holders`）。
- 启动自检：启动时生成 `boot_id` 注入环境变量，后台线程轮询 `/health`
  核对回显的 `boot_id`，确认应答者确为本次启动的进程。

## 常见问题

- **改了源码不生效**：确认 `reload` 生效的条件——启动器需以 `reload=True` 且
  `reload_dirs` 指向 `src/l_agent_chat`（已默认）。独占绑定下旧进程残留会导致
  启动直接报错而非静默抢请求，用 `-y` 重启即可自动清理（含僵尸 socket）。
- **API key 报错**：通过 `DEEPSEEK_API_KEY` 环境变量或设置页配置，见 `/api/settings`。
- **启动报"端口仍不可用（僵尸 socket 或无权限）"**： zombie 持有者不在本用户
  进程树内（如系统服务占用），手动 `netstat -ano | findstr :1250` 排查或重启系统。
