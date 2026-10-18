---
alwaysApply: false
description: 故障排查记录（公网部署机）：l_WChat 的 AI 功能全挂、hub 报「所有可用厂商均失败」。根因是**远端系统代理指向没人监听的 127.0.0.1:7890**，与 hub 代码/部署无关。
---

# 公网部署机「AI 服务不可用」排查记录（l_model_hub / l_WChat）

**日期**：2026-10-14
**现象**：`https://121.196.144.88/l_wchat/postpartum/2026/10/14` 的 AI 功能不可用，怀疑远端 `l_model_hub` 挂了。
**机型**：公网部署机，trayapp 根 `D:\TD_Depot\Software\Lugwit_syncPlug\lugwit_insapp\trayapp`。

---

## 1. 结论（TL;DR）

**远端机器的系统级 HTTP 代理开着，但代理不在跑**：

| 项 | 值 |
|---|---|
| 注册表 `HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings` | `ProxyEnable=0x1` · `ProxyServer=127.0.0.1:7890` |
| `127.0.0.1:7890` | **没人监听**（`connect_ex`≠0；7891 / 7897 / 10808 / 10809 / 1080 / 8889 / 20171 也都不在听） |
| 进程 | 只有 `FlClashHelperService.exe`；**FlClash 主程序没在跑**（安装痕迹：`C:\Program Files\FlClash` + 桌面快捷方式） |

于是 Python 的 HTTP 栈（Windows 上会读注册表代理）每次出网都去连那个空端口 → 立即 `ProxyError` →
hub 「所有可用厂商均失败」→ WChat 的 AI 全挂。

**与 `l_model_hub` 的代码无关，与本次部署无关。**

---

## 2. 排查路径（每一步拿到了什么证据、排除了什么）

| # | 动作 | 观察到 | 排除/定位 |
|---|---|---|---|
| 1 | 读**远端 WChat 后端日志** `D:\Temp\Log\rez_pkg_log\l_WChat\l_WChat_l_wchat_backend_<启动日>.log` | `AI 调用失败 l_model_hub/auto: 模型中心 HTTP 500: 所有可用厂商均失败: litellm.InternalServerError: OpenAIException - Connection error.. Received Model Group=aliyun/qwen3.7-max …`；后续变成 `HTTP 502: 无可用厂商` | 请求**到了 hub** ⇒ 不是 WChat 自身的问题，是 hub 出网失败 |
| 2 | 远端本机 `GET :8462/healthz` | 200 `{"ok":true,"service":"l_model_hub","providers":10}`；启动日志「中心存储可用，18 家：…」 | **hub 活着、密钥在** ⇒ 排除"服务挂了 / 没密钥" |
| 3 | hub 管理台「概览」的厂商自检（用户在浏览器看到） | `ProxyError: HTTPSConnectionPool(host='api.deepseek.com', port=443): Max retries exceeded … (Caused by ProxyError('Unable to connect to proxy', NewConnectionError("HTTPSConnectionPool(host='127.0.0.1', port=7890): Failed to establish a new connection: [WinError 10061] 由于目标计算机积极拒绝")))` | **直接指到代理** |
| 4 | 远端取证（走 `l_repo_sync_gui` 的 `DeployChannel.run_remote`） | ① 服务进程 env 里**没有** `HTTP_PROXY/HTTPS_PROXY/ALL_PROXY`；② 注册表系统代理 `ProxyEnable=1 / 127.0.0.1:7890`；③ 7890 及常见代理端口全无监听；④ `tasklist` 只有 FlClashHelperService，FlClash 主程序缺失 | 不是 env、**是系统设置**；代理软件只在跑"helper 服务" |

> ⚠ **两条日志坑（下次直接绕开）**
> 1. **WChat 的日志文件按"启动日"命名**：远端 WChat-11 起一直没重启**，所以"今天的日志"其实是
>    `l_WChat_l_wchat_backend_20261011.log`（里面混着今天 22:26 的流量）。找日志要按 **mtime** 取，别按文件名日期。
> 2. **hub 的日志每次启动重写**：`l_model_hub_..._20261014.log` 只有 16KB，出问题那段会被"重启"覆盖掉
>    ⇒ 排障要**当场取**，别事后翻。

---

## 3. 为什么"没设 `*_PROXY` 环境变量"也会走代理（关键知识点）

**Windows 上 `urllib.request.getproxies()` 会读注册表的 Internet Settings（WinINET）**，`requests` /
`httpx`（`trust_env=True`）/ `openai` / `litellm` 默认都会用它。所以只要有
`ProxyEnable=1` + `ProxyServer=<host:port>`，**即使一个 `*_PROXY` 环境变量都没设**，也会去连那个代理。
排查时"env 干净"**不能**排除代理 —— 必须看注册表。

---

## 4. 同一根因的其他表象（别当成独立故障）

- hub 全厂商失败 → **所有经 hub 的 AI 功能**（WChat 的写作/暖语/标题…）一起挂；
- `LiteLLM: Failed to fetch remote model cost map from https://raw.githubusercontent.com/… WinError 10061`
  —— 无害（有本地兜底），但**是同一个代理故障**的旁证；
- WChat：`⚠️ 相册直链解析失败：pan.baidu.com Max retries exceeded`、`百度网盘元数据查询失败 …: 500`
  —— 同源（出网被挡）。

---

## 5. 修法（三选一，按省事排序）

1. **在远端把 FlClash 打开**（它自启过，现在只剩 helper 服务）—— 恢复原状，最省事；
   若它本来就是为了 GitHub / 国外模型下载，这条最对。
2. **关掉系统代理**（`ProxyEnable=0`）：`api.deepseek.com` / 火山 / 硅基 / 魔搭 / 阿里 / MiniMax / 智谱
   这些**国内站不用代理**也通，AI 立刻恢复；代价是 `raw.githubusercontent.com` 类继续失败（有兜底）。
3. **只让 hub 不走代理**：给 `l_model_hub` 服务加 `NO_PROXY`（或按域名白名单），系统代理保持原样。
   属于**代码/配置改动**（`l_model_hub/package.py` 的 `commands()` 或卡片 env），维护成本在
   "以后若某家模型必须走代理，白名单要跟着维护" ⇒ **要拍板**。

---

## 6. 怎么自证修好了

| 判据 | 怎么看 |
|---|---|
| 厂商自检不再全是"失败" | hub 管理台「概览」→ 各厂商「可用 N/0 失败」 |
| 真打一次 | `POST :8462/v1/chat/completions {"model":"auto", …}`（要服务账号令牌）→ 200 且有正文 |
| 最快 | 修完在 WChat 页面重试一次 AI；或看 `l_WChat` 日志里不再出现 `AI 调用失败 l_model_hub/auto` |

⚠ **只看 `GET :8462/healthz` 200 是不够的** —— 它只说明"进程活着"，与"能不能出网"无关
（本次就是 healthz 200 + 全厂商失败）。

---

## 7. 教训

1. **"AI 全挂"先看厂商自检的错误原文**，别先看"服务在不在"。`healthz 200` 完全不代表能出网。
2. **代理是单点**：公网机上"系统代理 + 本机端口（FlClash/v2ray 的 7890）"这种组合，代理一停，
   **所有**走 HTTP 的程序一起挂（Python 的是因为读注册表，与 env 无关）。
3. 远端排障**走 `DeployChannel.run_remote` 在机器本机取证**（端口 / 注册表 / 进程 / env），比隔空猜快一个数量级：
   ```python
   from l_repo_sync_gui.deploy_channel import DeployChannel
   ch = DeployChannel(); ch.probe()
   print(ch.run_remote("import socket;s=socket.socket();print(s.connect_ex(('127.0.0.1',7890)))"))
   ```
4. 远端日志的**命名/覆盖规则**先搞清（本次两条都踩了）：WChat 按启动日命名、hub 每次启动重写。

---

## 8. 附：同一次会话里对远端的部署（与本次故障无关）

| 包 / 对象 | 结果 |
|---|---|
| `l_py_ui` | 11/11（**新包，必须先上**：另两个包 `requires` 它） |
| `l_agent_chat` | 100 传 / 323 校验 0 不符 → 重启后在线，`/health` 200 |
| `l_agent_tool` | 40 传（远端旧版缺 `AgentTool(deadline=…)`，导致远端启动 `TypeError`） |
| `l_node_engine` | 47 传 |
| `l_agent_market` | 已一致 |
| `rez-package-3rd` **13 个 family** | 5993 文件（远端**不能上网**，`pyflakes` / `python_lsp_server` / `jedi` 等本机 pip 包直接补） |
| `l_model_hub` | 16 传，`/healthz` 200 |
| `l_mindmap_mmd` | 同步中 |

**过程里踩到的坑（值得记）**：这次部署**一度把远端 `l_agent_chat` 打下来**（`rc=3`）。
不是本次改动的问题，而是**远端整体落后**（缺 69 个文件）：一并带过去的 `package.py`（早前加的 LSP 依赖）
+ `code_mode.py`（用 `AgentTool(deadline=…)`）撞上远端**旧的 `l_agent_tool`** 与**缺失的 pip 包**，
rez 解析和 import 各炸一次。补齐后恢复。
⇒ **推 `l_agent_chat` 建议连 `l_agent_tool` / `l_node_engine` / `rez-package-3rd` 一起推**，
否则就是"推完服务起不来"。
