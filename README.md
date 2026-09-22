# Rez-Docs 文档索引

`rez-package-source` 各 Rez 包与 wuwo 启动器的设计/使用/排错文档。本索引区分**现行主文档**、**计划台账**与**历史归档页**；历史页只作追溯，不能覆盖主文档。状态口径截至 2026-09-17。

## 推荐阅读顺序

1. 先读 **Rez/wuwo 机制** 前兩篇(建包 + 装包排错),理解包从何而来、如何启动;
2. 再按需读 **单实例/热更新** 三篇(运维安全相关);
3. **包使用文档** 与 **架构设计** 按当前任务查阅,无先后依赖。

---

## 一、Rez / wuwo 机制

| 文档 | 一句话摘要 |
|------|-----------|
| [Rez包创建和启动指导文档.md](Rez包创建和启动指导文档.md) | Rez 包目录结构、package.py 写法、alias 定义与 wuwor 启动方式,新建/维护包的入门 |
| [Rez_pkg/变体哈希与wuwo的处理方法.md](Rez_pkg/变体哈希与wuwo的处理方法.md) | rez-pip 变体目录名含 Windows 非法字符导致"空壳包"的问题,以 reflex 为完整案例 + 排查手册 |
| [solo_单实例守卫模式.md](solo_单实例守卫模式.md) | `.solo` 单实例守卫实现链路、双实例抢端口事故复盘、server 侧端口自检加固 |
| [dev_mod_热更新机制_fa50f01f.md](dev_mod_热更新机制_fa50f01f.md) | **历史说明**：保留 `.dev_mod` → `L_DEV_MOD=1` 与 wuwo `ENV_MODIFIERS` 门控；现行热重载不再用 uvicorn `--reload` |
| [src_hot_reload_源码热重载与主页常驻.md](src_hot_reload_源码热重载与主页常驻.md) | **现行主文档**：`SrcHotReload` / `L_SRC_WATCH`、服务重启、主页常驻；`l_notepad_server` 改 `.py` 须手动重启 |

## 二、包使用文档(Rez_pkg/)

| 文档 | 一句话摘要 |
|------|-----------|
| [l_script_editor.md](Rez_pkg/l_script_editor.md) | 脚本编辑器组件库:代码编辑/补全/会话管理 + 8764 HTTP 远程执行服务 + `/ui/*` Qt UI 自动化端点 + **端口固定 / 服务发现(`~/.Lugwit/run/<service>.json`) / IPC 命名管道** |
| [服务发现与IPC.md](Rez_pkg/服务发现与IPC.md) | **本机怎么找到并调用服务**:发现文件格式与 CLI、命名管道(带端口+authkey)、当前端口分配表、页面走 TCP/脚本走 IPC 的双栈理由 |
| [l_notepad_server.md](Rez_pkg/l_notepad_server.md) | L Notepad 服务端(8765):Web UI、REST API、多知识库(`/web/kb/{name}`),认证经 lugwit_auth |
| [l_notepad_搜索接口使用文档.md](Rez_pkg/l_notepad_搜索接口使用文档.md) | **搜索接口怎么用**:`/api/search` 与 `/api/kb/{kb}/search` 参数/返回字段/打分公式、查询语法(引号短语/多字 OR 召回)、`lex/hybrid/sem` 三模式、向量语义(模型切换/阈值/重嵌)、索引维护与权限模型、已知坑 |
| [l_homepage.md](Rez_pkg/l_homepage.md) | 主页 `/homepage/deps` 依赖拓扑页开发笔记:连线特效、拖拽建边、右键启停、卡片编辑 |
| [lugwit_baidu_netdisk.md](Rez_pkg/lugwit_baidu_netdisk.md) | **使用手册**：Depot 与网盘页面、接口、操作语义；实现模型与计划分别链接主文档/计划台账 |
| [lugwit_baidu_netdisk.md §14](Rez_pkg/lugwit_baidu_netdisk.md) | **客户端直传 / 安卓壳**：`POST /api/upload/prepare|finish`、原生 HTTP 通道、登录 + HTTPS 闸门 |
| [lugwit_baidu_netdisk.md §16](Rez_pkg/lugwit_baidu_netdisk.md) | **blob 去重必须先验存**：登记行还在、网盘文件没了 → 提交只涨 rev 不写 blob，重传永远修不好（2026-09-22 修复） |

## 三、工具使用指南

| 文档 | 一句话摘要 |
|------|-----------|
| [l_agent_chat使用指南.md](l_agent_chat使用指南.md) | 本地 AI 编码 Agent 聊天服务:FastAPI Web UI + SSE 流式对话,OpenAI 兼容接口(默认 DeepSeek)；含**输入框 `/` 命令与 `@` 文件补全**（两版 UI 同源，附新版实现三坑） |
| [l_agent_tool使用指南.md](l_agent_tool使用指南.md) | Agent 工具库:默认工具集(文件/Git/HTTP/远程执行等)与自定义注册,供脚本编辑器等复用 |

## 四、架构与设计

| 文档 | 一句话摘要 |
|------|-----------|
| [Nginx反向代理机制.md](Nginx反向代理机制.md) | **网络与路由主文档**：统一入口、路由表、前缀剥离、WebSocket、排错 |
| [Rez_pkg/HTTPS证书与域名申请总结.md](Rez_pkg/HTTPS证书与域名申请总结.md) | **HTTPS 主文档**：IP 自签为主（App 内置 CA）；DuckDNS / Let's Encrypt 路线已停用；443 SNI、续期与自愈 |
| [网盘版本库Depot设计.md](网盘版本库Depot设计.md) | **已实现设计主文档**：按库隔离 blob、元数据与 Depot 工作流 |
| [网盘版本库Depot演进计划.md](网盘版本库Depot演进计划.md) | **唯一计划台账**：T1–T6、T2 直传部分已实现且待端到端核实（登录闸门 + `/login` 页 + `upload_direct.js` 拦截层 + 懒回源） |
| [Depot库与工作区方案.md](Depot库与工作区方案.md) | **未实施/待评审方案**：P4 Client View 与工作区候选设计，不能当现状 |
| [标题栏提供的服务.md](标题栏提供的服务.md) | 标题栏登录入口、服务器设置、脚本编辑器接入；工具细节链接独立指南 |
| [宝妈笔记App架构与发布.md](宝妈笔记App架构与发布.md) | App 架构/发布历史记录；其中 HTTP、cleartext、`--reload` 内容需核实，按现行链接执行 |
| [ComfyUI调试经验.md](ComfyUI调试经验.md) | ComfyUI 前端调试案例集:节点 flag 图标不显示等 DOM/CSS 排查过程与修复 |
| [CodeMaker能力清单与移植评估.md](CodeMaker能力清单与移植评估.md) | **外部组件调研 + 移植候选**：从 `codemaker-26.9.4` 捆绑 Agent 挖出的治理层行为（规则注入/ignore/hooks/MCP/spec 解析）与 `l_agent_chat` 的差距对照；**非现状、非已批准计划** |
| [l_agent_chat会话存云与工作区.md](l_agent_chat会话存云与工作区.md) | **未完成改造的交接文档**：会话改为云为真源（P4 式 depot ↔ workspace）、不一致用状态显示；已完成部分 + 确切下一步 + 服务端 P4 API 全表；**§13 云端会话在侧栏可见 + 按需拉取（已实现）**、**§14 `⚠ 云端内容缺失` 重传修不好的服务端去重缺陷（已修）** |
| [l_log会话上云计划.md](l_log会话上云计划.md) | **计划文档（未实施）**：l_log 的 AI 会话如何上云 —— 落点 `<根>/ai_chats/` → 库 `/l_log`；照抄 l_agent_chat 会话存储的 7 条硬事实；分 8 步实施与验收 |

---

## 维护约定

- 包使用文档放 `Rez_pkg/` 子目录,文件名与包名一致;机制/架构类放根目录。
- 文档内引用代码位置以**函数名**为准(行号会漂移),并注明"行号截至日期"。
- 新增文档后请在本索引对应分组补一行。
- **OpenSpec change 落地后**（`openspec/changes/archive/`），把「怎么实现的 / 为什么这么选」
  合并进本目录对应主文档的新小节（如 `§13`），并在本节登记该文档；`openspec/specs/` 只放
  需求契约（SHALL + 场景），不写实现细节 —— 两边别互相复制整段。
