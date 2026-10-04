# Rez-Docs 文档索引

`rez-package-source` 各 Rez 包与 wuwo 启动器的设计/使用/排错文档。本索引区分**现行主文档**、**计划台账**与**历史归档页**；历史页只作追溯，不能覆盖主文档。状态口径截至 2026-10-04。

## 推荐阅读顺序

1. 先读 **Rez/wuwo 机制** 前兩篇(建包 + 装包排错),理解包从何而来、如何启动;
2. 再按需读 **单实例/热更新** 三篇(运维安全相关);
3. **包使用文档** 与 **架构设计** 按当前任务查阅,无先后依赖。
4. ⚠️ **卡片里的服务一律走 `wuwo svc list|status|start|stop|restart|reload|log`**
   （不要 `wuwor <包> -- <别名>` 直启）：先看 [Rez包创建和启动指导文档.md](Rez包创建和启动指导文档.md) §17.5，
   完整手册在规则文件 `.cursor/rules/service-lifecycle.mdc`。

---

## 一、Rez / wuwo 机制

| 文档 | 一句话摘要 |
|------|-----------|
| [Rez包创建和启动指导文档.md](Rez包创建和启动指导文档.md) | Rez 包目录结构、package.py 写法、alias 定义与 wuwor 启动方式,新建/维护包的入门 |
| [Rez_pkg/变体哈希与wuwo的处理方法.md](Rez_pkg/变体哈希与wuwo的处理方法.md) | rez-pip 变体目录名含 Windows 非法字符导致"空壳包"的问题,以 reflex 为完整案例 + 排查手册 |
| [solo_单实例守卫模式.md](solo_单实例守卫模式.md) | `.solo` 单实例守卫实现链路、双实例抢端口事故复盘、server 侧端口自检加固 |
| [src_hot_reload_源码热重载与主页常驻.md](src_hot_reload_源码热重载与主页常驻.md) | **现行主文档**：`SrcHotReload` / `L_SRC_WATCH`、服务重启、主页常驻；`l_notepad_server` 改 `.py` 须手动重启。**2026-09-23 增补**：常驻故障复盘 + 4 点加固（源码体检/端口释放重试/spawn 重试+限并发/退避）、热重载日志采集修复、`watchdog`/`hotreload` 诊断接口。**2026-09-27 增补**：主页**自重启健壮性**（等端口释放 + 最多 3 次重试 + 75s 就绪超时 + 清陈旧 guard + guard 让位）、`trigger=cli` 来源、服务操作统一入口 `wuwo svc`（见 `Rez_pkg/服务托管与统一启动入口_计划.md`） |
| [Rez_pkg/wuwo_gui.md](Rez_pkg/wuwo_gui.md) | **wuwo 图形管理界面**：doc_pkg / 按需拉包 / 包同步 / 环境解析 / 货架浏览 / 解释器清理 / 配置查看收进一个 PySide6 窗口；界面只拼参数，逻辑全在 `wuwo/py_modules/*.py` |
| [Rez_pkg/ayon.md](Rez_pkg/ayon.md) | **AYON Server 本地部署**：改造过的官方 compose（postgres + redis + ynput/ayon）+ 启停命令，跑在 podman/WSL；给 `wuwo_gui`「AYON」页提供只读数据源 |
| [Rez_pkg/wuwo_实例与数据分区.md](Rez_pkg/wuwo_实例与数据分区.md) | **多份安装分区概念（2026-10-01）**：程序区（不绑用户）/ 用户数据区（绑用户、`~/.lugwit/<实例>`）/ 共享只读大件区（`E:/lugwit_rez/homes/`）三区各绑什么；只讲目录归属，整树搬家操作见本文 §3（2026-10-04 已与原《多实例与迁移手册》合并为一篇） |

## 二、包使用文档(Rez_pkg/)

| 文档 | 一句话摘要 |
|------|-----------|
| [l_script_editor.md](Rez_pkg/l_script_editor.md) | 脚本编辑器组件库:代码编辑/补全/会话管理 + 8764 HTTP 远程执行服务 + `/ui/*` Qt UI 自动化端点 + **端口固定 / 服务发现(`~/.Lugwit/run/<service>.json`) / IPC 命名管道**。**2026-10-04 拆分**：逐端点细节已移入 `l_script_editor_端点全表.md`，本文聚焦使用指南 + 接口总览 |
| [Rez_pkg/l_script_editor_端点全表.md](Rez_pkg/l_script_editor_端点全表.md) | **端点全表（2026-10-04 从主篇拆出，各包"接口全表"的引用目标）**：HTTP `/status`·`/execute`·`/execute_async`·`/upload*`（含分块上传）·`/download`·`/upload_folder` + `/ui/*` Qt UI 自动化（tree / locate / action / wait / screenshot）+ Agent 工具系统 `/tools`。章节号不变（§3.1–§3.9、§5），主篇留接口总览 |
| [服务发现与IPC.md](Rez_pkg/服务发现与IPC.md) | **本机怎么找到并调用服务**:发现文件格式与 CLI、命名管道(带端口+authkey)、当前端口分配表、页面走 TCP/脚本走 IPC 的双栈理由 |
| [l_tray.md](Rez_pkg/l_tray.md) | **托盘（本机能力中枢）**：`19527` ExecServer 的 `/health`·`/run`·`/register`、`action_registry` 与 `web_actions` 白名单（当前 22 个，含 `depot_local_*` 6 个：目录树/变更序号/在资源管理器打开/新建目录·文件/删除到回收站）、托盘登录态与 Origin 限制、`requires`（watchdog / winshell）。**2026-09-27 增补**：`watchdog` vs `watchfiles` 实测选型表（§2）、登录态只在启动时恢复 → `restore_session` 补法（§3）、`reregister` 的 `reload` 是顶层键 + 动态 `/run` 自省（§5） |
| [l_model_hub.md](Rez_pkg/l_model_hub.md) | **模型中心 / OpenAI 兼容网关（8462）**：厂商密钥（权威在 auth `/api/v1/secrets`，`scope='model_hub'`）· 多厂商失败自动切换与档位（eco/std/max）· 统一登录闸门（凭据优先级 Bearer > 插件头 > cookie `lugwit_token` > env）· **接入密钥 `sk-lmh-…`**（`/v1/access-keys`，只存 sha256、365 天可吊销，给 IDE 插件替代 572 字符长 JWT）· **手动补充模型 id**（`/v1/models/manual`）· **调用统计落盘上云**（`usage_store`：**厂商 + 模型两维度**、按天分桶 + 中心存储 + depot `usage.json`，管理台带**模型柱形图**，`L_MODEL_HUB_USAGE_KEEP_DAYS` 默认 30 天；**路由页可搜可筛 + 拖拽排序 + 按厂商批量设档 + 只提交改动**；`POST /v1/chat/completions/multi` **多模型并答**（同一输入框并发多家、钉死不降级）；**模型可用性**逐模型探测并落库（`/v1/models/health`，不可用模型不进选择列表）） |
| [l_model_hub_本地模型_统一管理_计划与验收.md](Rez_pkg/l_model_hub_本地模型_统一管理_计划与验收.md) | **计划与验收**：在 l_model_hub 管理台里把开源模型（视频/图像/语音/LLM）装起来、跑起来、测通、接成 hub 引擎位；**A 方案 = 一个模型一个独立 Rez 包 + 一张服务卡片**，hub 只做控制面 |
| [公网部署与增量同步.md](Rez_pkg/公网部署与增量同步.md) | **公网部署（2026-10-01 实机走通）**：走 `l_repo_sync_gui` 直推通道（`http://121.196.144.88:8764` = 服务器上的 `l_script_editor.standalone_server`；`GET /status` 探活，`editor_available:true` 才可用；本机 IP 在白名单内免 token）· **增量靠远端 sha256 逐文件比对**（`upload_files(skip_unchanged=True)`：内容一致才跳过，比"同大小+同日期"更严；实测 `wuwo`+21 包共 12,092 文件里只传 1,238，回扫全部"需传 0"）· **批量 runner** `l_repo_sync_gui/999.0/deploy_to_public.py`（默认只探测、`--only <包>` 单包推进、**从不删除远端文件**、**从不重启远端服务**）· **排除清单**（`logs_cache/` 单文件几 MB 会撞 **413 单请求上限**、`deploy_home.txt` 部署级不入库、`task_files.json`/`setting.yaml` 本机专用、服务器的 `wuwo/config/config.yaml`、`l_mindmap_fasthtml` 私有数据）· **服务器环境坑**：**没有 E 盘**（`third_party`/`wowo_log_dir` 盘符缺失自动回落、`LUGWIT_SHARED_HOME` 不注入）、**没有 `Lib/` 目录**（树根探测必须 `wuwo` + (`Lib` 或 `rez-package-source`)，87 个文件受影响）、数据根是旧布局 `C:\Users\<用户>\.Lugwit\<包>` · **首次对接数据根 = 零搬移**（不覆盖它的 config；只追加 `instance`/`data_root`/`l_data_dir` 三行 + `.bak_deploy` 备份 + `pkg_data_dir()` 只读验证）· **8764 与托盘互不影响**（两条独立进程链，实测共同祖先为空；但别关整个 ConEmu 窗口）· 起 GUI 进程用参数列表 + `CREATE_NEW_CONSOLE`（别在 `cmd /c "..."` 里手拼引号）· **`ChatRoom` 不走文件通道**（它与 `Lib/ChatRoom` 是两棵树，应走 git）|
| [l_indextts2.md](Rez_pkg/l_indextts2.md) | **本机 IndexTTS2 语音合成（8470，卡片「L IndexTTS2 本机语音」）**：零样本音色克隆 + 情绪/语速控制 · **推理在独立 uv venv(python 3.10)**（IndexTTS2 的 `requires-python` 是 `>=3.10,<3.12`，装不进 wuwo 的 py_312；同 `l_comfyui` 路线）· 音色档案（参考音频注册 → `voice` 传档案 id）· OpenAI `/v1/audio/speech` + l_model_hub `/tts` 的 `engine=indextts2` · 安装 `indextts2_setup`、体检 `indextts2_doctor` · **许可 = bilibili 非商用** |
| [l_ltxvideo.md](Rez_pkg/l_ltxvideo.md) | **本机视频生成同壳族（l_ltxvideo / l_wanvideo，2026-10-02 合并）**：纯 Rez 依赖 + worker 子进程 + 单卡串行 + 自带单页前端；两模型对比（LTX-Video 2B @8475 vs Wan2.2-TI2V-5B @8476，**Wan 是 16GB 上能跑的最新最强**）· 安装走 l_model_hub「本机模型」页 · **踩坑 13 条**（torch CUDA 索引 / rez-pip sympy RECORD / `requires` 别写版本号 / 跨盘 WinError17 / 货架路径单一来源 / 帧类型 PIL vs tensor 画质静默降级 / **Wan 必须 `-Diffusers` 后缀仓库**）· 依赖组合表（torch 2.8.0+cu128 / diffusers 0.35.2 / transformers 4.57.6）· 阶段 ETA / 对外入口 nginx 两处 |
| [流程图智能体_图驱动控制流.md](Rez_pkg/流程图智能体_图驱动控制流.md) | **默认智能体的控制流由一张图决定（2026-10-04 落地）**：三档开关（`flow_enabled` / `flow_full` / `flow_name`）· 图在哪 & 编辑器入口 · `GET :1250/api/flow/status` 权威状态 · **运行轨迹**（`_traces/*.jsonl` + 编辑器 `📈 轨迹` 热力/回放）· **策略包**（图+观感+档位一键切换，可拖入导入）· 硬约束（`has_calls`/`steps_exhausted`/always 兜底/谓词表/guard 名/sub 防环）与保存前权威校验（`POST /api/flow/validate`）· **策略诊断** `flow_spec.diagnose` + 每轮 `🔧 策略建议` · 25 谓词与 state 清单 · 排错表 · 验收记录 |
| [l_mindmap_mmd.md](Rez_pkg/l_mindmap_mmd.md) | **脑图 / 流程图编辑器（8110，mm.js 迁移 · React Flow，2026-10-04 新建）**：`l_agent_chat` 默认智能体流程图（`default_intelligent_agent.json`，2026-10-04 由 `default.json` 改名）的可视化编辑入口，与运行端同源解析同一份 flows 目录（实例数据根）· 三种线型（贝塞尔/直角/直线）+ 回边虚线流动箭头 + **直出直入手柄权重 / 柄长系数 / 手柄偏向目标** + 连线智能避让（碰撞边界膨胀 + 局部软推 + 采样点可视）· 「整理布局」5 种（`flow` 默认，复刻手工布局）· 侧边设置面板（**四组可折叠**、占位列、实时预览、数值可手输、**设置存 `~/.lugwit/l_mindmap_mmd/settings.yaml`**，前端零默认值、模板补缺）· 「下游跟随」开关存 flow `_ui.follow_move` · 画布右上角 FPS · `/api/settings` GET/PUT/reset |
| [l_notepad_server.md](Rez_pkg/l_notepad_server.md) | L Notepad 服务端(8765):Web UI、REST API、多知识库(`/web/kb/{name}`),认证经 lugwit_auth |
| [Rez_pkg/知识库归档删除同步与索引清理_计划.md](Rez_pkg/知识库归档删除同步与索引清理_计划.md) | **索引侧已实施（2026-10-04）**：实测更正 —— 归档**列举本身就排除已删文件**、已删文件 `depot/file` 回 **410**，所以 `live_only`/`is_deleted_state` 是**冗余保险**；**7 篇已用 `/api/depot/delete` 显式标删**。**清旧行补齐**：以前只清词法侧，`_drop` 现在**两侧同删**，另补 `purge_orphan_index` 兜底（已删库的词法行 + 无词法行的向量残留，挂 kb tick/启动预热）。**不做**：同步器传播删除（多机各有工作区，"本地没有" ≠ "该删共享归档"）。含护栏（空扫描绝不清理等） |
| [Rez_pkg/笔记元数据与notepad工具改造_计划.md](Rez_pkg/笔记元数据与notepad工具改造_计划.md) | **已实施（2026-10-04）**：笔记**顶部元数据固有格式**（`<!-- lugwit-note … -->`：`updated`/`updated_by`/`note` + 最多 5 条 `lugwit-note-history`，读时**剥离**进 `meta`/`meta_lines`）；`notepad_read` 加**归档兜底**（标 `source`；已删文件 `depot/file` 回 410 故读不到）；新增 **`notepad_modify`**（`note` 必填、自动维护元数据与历史、`target=auto\|workspace\|archive`、`expect_rev` 写归档时校验）；并已加权限规则 `notepad_modify → ask`。**剩余**：kb 路由的显式删除入口 |
| [l_notepad_client.md](Rez_pkg/l_notepad_client.md) | **桌面客户端(排错向)**：启动方式(含 alias detach 坑)、日志位置、静默崩溃分层排查(Python 异常 vs Qt 原生崩溃)、事件查看器/WER dump 抓现场；**2026-09-23 修复**两处 Python 异常 + faulthandler 句柄 |
| [l_notepad_搜索接口使用文档.md](Rez_pkg/l_notepad_搜索接口使用文档.md) | **搜索接口怎么用**:`/api/search` 与 `/api/kb/{kb}/search` 参数/返回字段/打分公式、查询语法(引号短语/多字 OR 召回)、`lex/hybrid/sem` 三模式、向量语义(模型切换/阈值/重嵌)、索引维护与权限模型、已知坑。**2026-10-04 增补**：含 `/api/search/route` 快速选库（`depth` 分档 / 硬规则 / 验收标准），默认 `mode=auto`（分数判据回退） |
| [Rez_pkg/l_notepad_搜索改造史.md](Rez_pkg/l_notepad_搜索改造史.md) | **搜索改造时间线（2026-09-24 ~ 09-28，2026-10-04 由 4 篇合并）**：route/auto/全局搜索页/代码库索引 → 功能增强 → 症状索引方案 → 一期体检整改（重排喂错料 / RRF / 块长归一 / 导航块降权 / 语料 17k→2.5k / 重排默认关）→ 二期（症状串库 / 未索引可见 / 文件名召回 / `mode` 可观测 / confidence 校准 / 评测集扩到 40 条，真实 top-1 75.7%）。**§7 是订正口径清单（24 条）**：DB 并未变小、硬预算 3000ms 且非墙钟、重排"不劣且快"等 —— 看这篇能避开已被推翻的旧结论 |
| [l_homepage.md](Rez_pkg/l_homepage.md) | 主页开发笔记：`/homepage/deps` 依赖拓扑、卡片/局部刷新、日志窗口。**2026-09-23 增补**：常驻/热更新诊断页、故障率（按触发来源）、日志查看器「加载更早 + 历史日期」、兜底页三态、卡片 `window.open`、语法体检 CLI。**2026-09-24 增补**：卡片 Git 同步按钮（⬇ 拉取 / ⬆ 推送，含冲突时强制拉取）、复制命令按钮移到「编辑」旁、热启动即打开日志窗口。**2026-09-27 增补**：卡片 `name`(机械标识)/`label`(显示名) 拆分 + `migrate_cards`、`GET /api/v1/services/hosted` 索引端点、`status?target=` 单卡探测、`{name}/{op}?trigger=cli`、兜底页热更倒计时 + 服务自身日志面板 |
| [l_homepage_热重启失败报警色_计划.md](Rez_pkg/l_homepage_热重启失败报警色_计划.md) | ⚠️ **小计划，多半已可归档**：热重启失败时卡片报警色 + 根因修复；阶段 0、1 已落地，阶段 2 待做 —— 但 `src_hot_reload_源码热重载与主页常驻.md` 与 `l_homepage.md` 已描述该报警色机制。**确认阶段 2 无残留待办后**：要点并进 `l_homepage.md`，本篇移入「五、历史归档」 |
| [lugwit_baidu_netdisk.md](Rez_pkg/lugwit_baidu_netdisk.md) | **使用手册**：Depot 与网盘页面、接口、操作语义；实现模型与计划分别链接主文档/计划台账。**2026-09-26 增补**：页面三栏 + 标签可拖动/可跨面板/条末 `＋`（§6.2、§17）、预览/编辑从底栏挪进右栏、中栏↔右栏可拖宽度、**§5.6 工作区与库接口全表**（页面直连，不再经托盘）、工作区本地树右键（打开/新建/删除，见 §6.2 + `Rez_pkg/l_tray.md`）、**§20 越权与内存加固（P0）**（lock/unlock force/changes·change·tree 的 P6 补判、两个上传端点改流式、托盘 realpath/空 token/open 白名单）。**2026-09-27 增补**：`GET /api/depot/local_token`（HttpOnly cookie 下页面自取 token 调托盘，§5.1/§20）、§6.2 两模式刷新差异（浏览器 2.5s 轮询 vs **客户端快照不自动刷新**）与客户端桥 `treeDir` 无 `local_root` 限制 |
| [lugwit_baidu_netdisk.md §14](Rez_pkg/lugwit_baidu_netdisk.md) | **客户端直传 / 安卓壳**：`POST /api/upload/prepare|finish`、原生 HTTP 通道、登录 + HTTPS 闸门 |
| [lugwit_baidu_netdisk.md §16](Rez_pkg/lugwit_baidu_netdisk.md) | **blob 去重必须先验存**：登记行还在、网盘文件没了 → 提交只涨 rev 不写 blob，重传永远修不好（2026-09-22 修复） |
| [百度云接口元数据实测.md](Rez_pkg/百度云接口元数据实测.md) | **百度云接口字段实测唯一事实源（2026-09-17）**：md5 / 接口字段实测结论；设计文档、使用手册与计划只链接本文，不复制实验结论 |
| [l_agent_chat_技能市场与运行时接线_已完成与未完成.md](Rez_pkg/l_agent_chat_技能市场与运行时接线_已完成与未完成.md) | **已完成**：`l_agent_market` 库包（`gh:`/`http(s)`/本地三种源、Claude Code `marketplace.json` 归一、装到 `~/.lugwit/l_agent_chat/skills\|plugins`、`.market.json` 卸载保护）+ `l_agent_chat` 的 `/api/market*` 三端点与设置页两个市场面板。**未完成（本轮范围外）**：技能进 system 索引 / `skill` 工具 / `/技能名` 斜杠命令 —— 含锚点、方案、验收判据、3 个待拍板项与第三方技能正文的安全提示 |

## 三、工具使用指南

| 文档 | 一句话摘要 |
|------|-----------|
| [l_agent_chat使用指南.md](l_agent_chat使用指南.md) | 本地 AI 编码 Agent 聊天服务:FastAPI Web UI + SSE 流式对话,OpenAI 兼容接口(默认 DeepSeek)；含**输入框 `/` 命令与 `@` 文件补全**（两版 UI 同源，附新版实现三坑）；**2026-09-24 增补**：三档**权限模式**（default/allow_all/autopilot，与对话模式正交）、**终端沙盒化**（仅 Windows，AppContainer）、**提示词优化**（独立小模型，默认 `glm-4-flash`）；**2026-09-26 增补**：**MCP 市场**（环境页 → MCP 服务，默认源官方 MCP Registry，HTTP 连接复用 + 磁盘缓存/后台刷新）、热重载规则订正（单进程，改 `.py` 需手动重启）；**2026-09-28 增补**：`/tools` 弹窗重写（`/api/tools` + 分组 + 搜索 + portal 底部对齐）、`/rez` 改插 `rez_pkg <包>`、**思考按真实位置分段 + 工具/思考耗时 chip**、**逐行英中对照翻译 + 每行 🔊**、**语音走微软 Edge-TTS**、**源码弹窗 `/api/source`**、`persist:false` 无状态调用；**2026-10-02 增补**：`restate_question`（先复述用户问题的合成元工具，harness 保证 —— 只认第 0 步第一个调用、空或漏则重试一次再本地兜底；答复第一段恒为「**问题复述**：…」，工具卡显示复述全文）；**2026-10-04 增补**：**默认智能体由流程图驱动**（`flow_engine.run_flow`，图 `~/.lugwit/l_agent_chat/flows/default.json` 用户目录优先、`FLOW_ENABLED` 开关、失败回退硬编码护栏级联；可视化编辑走 `l_mindmap_mmd` :8110） |
| [l_agent_tool使用指南.md](l_agent_tool使用指南.md) | Agent 工具库:默认工具集(文件/Git/HTTP/远程执行等)与自定义注册,供脚本编辑器等复用。**2026-09-28 增补**：新增 `wait`（上限 60s，超出截断并提示改用 `run_background`）、`read_file` 返回 `line_start`/`line_end` |

## 四、架构与设计

| 文档 | 一句话摘要 |
|------|-----------|
| [Nginx反向代理机制.md](Nginx反向代理机制.md) | **网络与路由主文档**：统一入口、路由表、前缀剥离、WebSocket、排错 |
| [密钥落点与红线.md](密钥落点与红线.md) | **密钥治理速查（2026-09-26 实测）**：三条铁律 / 落点一览（哪是 PG 密文、哪是 DPAPI 包裹、哪是派生 token）/ 中心密钥存储读写链路（消费方 → hub `/v1/keys` → auth `/api/v1/secrets` → PG + KEK）/ 优先级口径 / 禁止清单与允许例外 / 自检排错命令 / 已知欠账（dev 库、5432、l_WChat 双份 key、depot 无密钥扫描…） |
| [lugwit_auth统一用户授权服务设计.md](lugwit_auth统一用户授权服务设计.md) | **统一用户授权（SSO）主文档**：lugwit_auth(1027) JWT + 浏览器 SSO（PKCE 授权码）+ 统一 `lugwit_token` cookie；**§9.8 现状矩阵** = 各包登录统一台账（8 个包已统一：l_tray / l_qframelesswindow / l_WChat / l_agent_chat / l_notepad_server / l_model_hub / l_homepage / ChatRoom；l_scheduler 登录源指 1027；netdisk / 匿名模型服务 / script_editor 有意保留）；2026-10-02 起 1027 签 **RS256**（JWKS 发公钥），消费方经 `create_jwt_service()`（RS256 验签 + HS256 存量兼容）或 `/api/v1/auth/me` 网络校验；含登录/登出路由、密钥落点、各包接入样板（**§9.8 各包接入现状已拆为独立台账** → `Rez_pkg/各包登录接入台账.md`）（`sso/start` + `sso/callback` PKCE 样板见 l_WChat / l_notepad_server） |
| [Rez_pkg/各包登录接入台账.md](Rez_pkg/各包登录接入台账.md) | **各包登录接入台账（2026-10-04 从设计文档 §9.8 拆出）**：各包登录方式 / cookie 名 / SSO 一致性差距矩阵 —— 随各包接入变化频繁，故独立成篇；**设计与协议**见 `lugwit_auth统一用户授权服务设计.md` |
| [授权登录代码归属与公共包抽取_计划.md](授权登录代码归属与公共包抽取_计划.md) | **计划（2026-10-03；D1–D5 已定；P0 完成、P1 原语完成 + 7 处收敛 + 核心包摘重依赖 + l_tray 收口）**：授权登录代码三分 —— 服务端归 `lugwit_auth` / 客户端逻辑抽新包 `lugwit_auth_client` / 其余业务自留。根因 = `lugwit_auth` 一个 Rez 包同时装服务端与客户端 SDK 且 `requires` 重，各包被迫三选一（吃重依赖 / **整包抄 SDK** `l_qframelesswindow/_auth_client` / 手写 HTTP）。含 17 包现状表、S1–S8 服务端归属、C1–C9 公共包清单、P0–P3 分期与验收、风险红线。**无过渡期**（D1–D5 已定，唯一例外 D3 离线/网络验签都留） |
| [Rez_pkg/HTTPS证书与域名申请总结.md](Rez_pkg/HTTPS证书与域名申请总结.md) | **HTTPS 主文档**：IP 自签为主（App 内置 CA）；DuckDNS / Let's Encrypt 路线已停用；443 SNI、续期与自愈 |
| [网盘版本库Depot设计.md](网盘版本库Depot设计.md) | **已实现设计主文档**：按库隔离 blob、元数据与 Depot 工作流 |
| [网盘版本库Depot演进计划.md](网盘版本库Depot演进计划.md) | **唯一计划台账**：T1–T6、T2 直传部分已实现且待端到端核实（登录闸门 + `/login` 页 + `upload_direct.js` 拦截层 + 懒回源） |
| [Depot库与工作区方案.md](Depot库与工作区方案.md) | **方案文档（部分落地）**：P4 Client View 与工作区候选设计 —— "工作区 + 页面直连"已于 2026-09-26 落地（`depot_workspace` 接口见 Depot设计 §6.2）；`ws_id` 数据库维度等另一半未实施 |
| [标题栏提供的服务.md](标题栏提供的服务.md) | 标题栏登录入口、服务器设置、脚本编辑器接入；工具细节链接独立指南 |
| [相册真源与本地缓存方案.md](相册真源与本地缓存方案.md) | **需求确认稿**：相册**唯一真源应是版本库**，本机 `_index.json` 只当缓存（可丢可重建）——现状把它当真源，导致两台机器"同一个库、内容不一样"；含数据归属表、P0–P5 分阶段方案、待拍板 6 问、验收标准与风险。**§10（2026-09-27）**：现状更新 + 与本文冲突点（R1 目前是目标态）、P1 的 7 个硬缺口（`stored` 不可派生居首）、**只读对账 `GET /api/album/reconcile` 已实现**、Q1–Q6 意见 |
| [相册功能与数据模型.md](相册功能与数据模型.md) | **相册主文档（2026-09-26）**：相册=容器（手动建、不再自动命名）/ 视图=数据模型（`_views.json` v2 + 权限位，`time·age·album·ext·size·tag·trash` + and/or/not，内置 `月龄`/`回收站` 均可编辑且要密码） / **删相册→照片挪进保留相册「其他」**（自动新建、不能删）/ 回收站（云盘 + depot `opera=move`）/ 视频（≤500MB 直传、>500MB 客户端实时压缩带进度、网格内联播放）/ 百度直链与浏览器缓存管理 / 三处落点与索引字段 / **AI 打标（百度智能云识图，手动批量；AK/SK 走 auth 中心密钥存储）** + **标签可见/可手动增删（§7.1）** + **标签随版本进 depot、版本库页面可看（§7.2）** / 已知取舍与排错速查 |
| [l_WChat_视频剪辑_对标剪映_计划与验收.md](l_WChat_视频剪辑_对标剪映_计划与验收.md) | **计划与验收**（2026-10-04 增补「八、验收记录」= 逐次 UI 改动要点，含根因）：`l_WChat` 视频剪辑（独立路由 `/video-editor`）对标剪映 —— 现状能力、差距盘点、分期计划、每次改动后逐条自测的验收标准（定位是"家庭场景够用"而非复刻剪映） |
| [Rez_pkg/l_WChat_视频剪辑页_非AI功能_自动化测试报告.md](Rez_pkg/l_WChat_视频剪辑页_非AI功能_自动化测试报告.md) | **自动化测试报告（2026-09-30；2026-10-04 裁剪 952→294 行）**：无头浏览器（Playwright）对 `video_editor.html` 的非 AI 功能回归 —— 41 条黑盒用例矩阵 + 22 条自检断言 + 环境/WebCodecs 门禁 + 5 条缺陷。**原 §9–§10 的逐次 UI 改动流水账已移入计划篇「八、验收记录」**，本篇只留"测什么/怎么测/结论" |
| [l_agent_chat_改造记录.md](l_agent_chat_改造记录.md) | **改造记录（2026-10-04 合并两篇交接）**：会话存云（P4 式 depot ↔ workspace、状态比对、SSE 进度）+ `.code-workspace` 工作区 + 历史交接（主循环 / 知识库 / 部署通道的已修已验 + 三条"别重复踩"陷阱）。已实现与待办分区；`D:` 老路径逐处标注"当时环境，已失效"（§24 汇总） |
| [l_log会话上云计划.md](l_log会话上云计划.md) | **已实施（核实 2026-10-04）**：此前索引写"未实施"与正文矛盾，经代码核实为**已落地** —— 会话落 `<根>/ai_chats/<sha1(logId)[:16]>.json`，随该根库 `/l_log` 同步（`chat_sync` / `depot_ledger` / `workspace_status` 的 `chats` scope / `/api/ai/chat/{push,pull,status,history,revert}`）；`agent_proxy` 带 `persist:false` 免 l_agent_chat 侧重复落盘。§阶段 1–8 留作实现史。**建议改名 `l_log会话上云_实现记录.md`**，但该文件名被 10 处代码注释引用，改名需一并处理 |
| [Rez_pkg/服务托管与统一启动入口_计划.md](Rez_pkg/服务托管与统一启动入口_计划.md) | **现行方案（P0–P3 已落地并实机验收 2026-09-27）**：服务生命周期收敛到主页卡片这一个控制面；`wuwo svc ...`（不依赖 rez 环境、薄 HTTP 客户端，`list/status/start/stop/restart/reload/hotstart/log [-f]/open`）是 AI 与脚本的统一入口；卡片数据**实时向主页要**（`GET /api/v1/services/hosted`，可选 `?status=1` / `?target=` / `?log=N`，另有 `status?target=` 单卡探测、`{name}/{op}?trigger=cli`）；卡片拆 **`name`(机械标识)/`label`(显示名)**；对"绕过入口直启"**只提示不阻断**（`L_HOSTED_BY` + `.unmanaged`/`L_SVC_DIRECT` 静音）；含自重启健壮性修复与误杀回归验收 |

---

## 五、历史归档（只作追溯）

> ⚠️ **本组文档是当时的现场记录，不能作为现状基线** —— 查现行行为请走上面各组的主文档。
> 它们的价值在于"当时为什么这么改 / 踩了什么坑"，只作追溯。

| 文档 | 一句话摘要 |
|------|-----------|
| [dev_mod_热更新机制_fa50f01f.md](dev_mod_热更新机制_fa50f01f.md) | **历史说明**：`.dev_mod` → `L_DEV_MOD=1` 与 wuwo `ENV_MODIFIERS` 门控；现行热重载已不用 uvicorn `--reload`（现行看 `src_hot_reload_源码热重载与主页常驻.md`） |
| [electron启动的坑.md](electron启动的坑.md) | **一次性排错记录（2026-09-10）**：VS Code 等 Electron 程序一启动就崩（`0xC0000005`）——根因是上一级程序往环境里塞了变量；排查过程与修法 |
| [ComfyUI调试经验.md](ComfyUI调试经验.md) | **一次性前端调试案例**：ComfyUI 节点 flag 图标不显示等 DOM/CSS 排查过程与修复 |
| [CodeMaker能力清单与移植评估.md](CodeMaker能力清单与移植评估.md) | **外部组件调研（快照 2026-09-20）**：从 `codemaker-26.9.4` 捆绑 Agent 挖出的治理层行为（规则注入/ignore/hooks/MCP/spec 解析）与 `l_agent_chat` 的差距对照。**自述：非本仓现状、非已批准计划** |
| [宝妈笔记App架构与发布.md](宝妈笔记App架构与发布.md) | **App 架构 / 发布历史**：自述含早期 http / cleartext / `--reload`，**不能当部署基线**（现行发布与 HTTPS 见 `Rez_pkg/HTTPS证书与域名申请总结.md`） |

**待归档候选**（确认无残留待办后再移入本组）：`Rez_pkg/l_homepage_热重启失败报警色_计划.md`。`l_log会话上云计划.md` 的矛盾已解除（核实为已实施，见上），可选归档。

---

## 维护约定

- 包使用文档放 `Rez_pkg/` 子目录,文件名与包名一致;机制/架构类放根目录。
- 文档内引用代码位置以**函数名**为准(行号会漂移),并注明"行号截至日期"。
- 新增文档后请在本索引对应分组补一行。
- **OpenSpec change 落地后**（`openspec/changes/archive/`），把「怎么实现的 / 为什么这么选」
  合并进本目录对应主文档的新小节（如 `§13`），并在本节登记该文档；`openspec/specs/` 只放
  需求契约（SHALL + 场景），不写实现细节 —— 两边别互相复制整段。
