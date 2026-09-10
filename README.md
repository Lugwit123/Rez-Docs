# Rez-Docs 文档索引

`rez-package-source` 各 Rez 包与 wuwo 启动器的设计/使用/排错文档。每篇均为独立成品文档,本索引只做导航。

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
| [dev_mod_热更新机制_fa50f01f.md](dev_mod_热更新机制_fa50f01f.md) | `.dev_mod` → `L_DEV_MOD=1` 按需启用 uvicorn 热更新,四个后端服务(auth/netdisk/chat/note)接入方式 |
| [src_hot_reload_源码热重载与主页常驻.md](src_hot_reload_源码热重载与主页常驻.md) | `SrcHotReload` 替代 uvicorn `--reload`:模板即时生效 + 代码变更自重启,根治 reload 孤儿 worker 盲区 |

## 二、包使用文档(Rez_pkg/)

| 文档 | 一句话摘要 |
|------|-----------|
| [l_script_editor.md](Rez_pkg/l_script_editor.md) | 脚本编辑器组件库:代码编辑/补全/会话管理 + 8764 HTTP 远程执行服务 + `/ui/*` Qt UI 自动化端点 |
| [l_notepad_server.md](Rez_pkg/l_notepad_server.md) | L Notepad 服务端(8765):Web UI、REST API、多知识库(`/web/kb/{name}`),认证经 lugwit_auth |
| [l_homepage.md](Rez_pkg/l_homepage.md) | 主页 `/homepage/deps` 依赖拓扑页开发笔记:连线特效、拖拽建边、右键启停、卡片编辑 |
| [lugwit_baidu_netdisk.md](Rez_pkg/lugwit_baidu_netdisk.md) | 百度网盘当内容寻址 blob 仓 + 简版 Perforce:提交/版本/回滚/签出,改名移动零流量 |

## 三、工具使用指南

| 文档 | 一句话摘要 |
|------|-----------|
| [l_agent_chat使用指南.md](l_agent_chat使用指南.md) | 本地 AI 编码 Agent 聊天服务:FastAPI Web UI + SSE 流式对话,OpenAI 兼容接口(默认 DeepSeek) |
| [l_agent_tool使用指南.md](l_agent_tool使用指南.md) | Agent 工具库:默认工具集(文件/Git/HTTP/远程执行等)与自定义注册,供脚本编辑器等复用 |

## 四、架构与设计

| 文档 | 一句话摘要 |
|------|-----------|
| [Nginx反向代理机制.md](Nginx反向代理机制.md) | nginx 8080 统一入口:路由规则表、WebSocket/大文件上传支持、各后端端口只监听本机 |
| [登录统一走Nginx代理.md](登录统一走Nginx代理.md) | 客户端登录/认证/账号/收藏 API 全部收敛到 8080 前缀路由,不直连后端端口 |
| [网盘版本库Depot设计.md](网盘版本库Depot设计.md) | 内容寻址 blob 仓 + Postgres 元数据唯一权威:md5 秒传、CL 版本链、零流量改名回滚 |
| [标题栏提供的服务.md](标题栏提供的服务.md) | 无边框标题栏(`L_FramelessMainWindow`)下沉通用服务:登录、服务器配置、脚本编辑器、帮助文档 |
| [宝妈笔记App架构与发布.md](宝妈笔记App架构与发布.md) | l_WChat Android App(Capacitor + WebView)架构、云服务器对接与发布流程 |
| [ComfyUI调试经验.md](ComfyUI调试经验.md) | ComfyUI 前端调试案例集:节点 flag 图标不显示等 DOM/CSS 排查过程与修复 |

---

## 维护约定

- 包使用文档放 `Rez_pkg/` 子目录,文件名与包名一致;机制/架构类放根目录。
- 文档内引用代码位置以**函数名**为准(行号会漂移),并注明"行号截至日期"。
- 新增文档后请在本索引对应分组补一行。
