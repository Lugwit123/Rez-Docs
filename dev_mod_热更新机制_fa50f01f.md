# dev_mod 特殊包名与热更新机制（历史说明）

状态：**历史机制说明；现行主文档见[src_hot_reload_源码热重载与主页常驻.md](src_hot_reload_源码热重载与主页常驻.md)**。截至 2026-09-17，本文不再作为服务热重载的执行手册。

## 保留的有效机制

- wuwo 的 `ENV_MODIFIERS` 仍识别 `.dev_mod`，并向目标进程注入 `L_DEV_MOD=1`。
- `L_DEV_MOD` 仍是开发态源码监视的门控：多数接入 `SrcWatchService` 的服务只有带 `.dev_mod` 才可开启 `L_SRC_WATCH`。
- `.solo`、`.soloignore` 的单实例语义独立维护，见[solo_单实例守卫模式.md](solo_单实例守卫模式.md)。

## 已迁移 / 已废弃

早期以 uvicorn `--reload`、服务专属 `*_RELOAD` 变量、`--reload` 别名为中心的正文已废弃：其 reloader worker 可逃过 `.solo` 命令行匹配并导致端口竞争，且 watcher 在本环境有失效记录。

当前使用 `l_app_ready.hotreload.SrcHotReload` / `SrcWatchService`：模板通过 Jinja `auto_reload` 刷新生效，代码变更由服务自重启链处理。服务清单、接口路径、门控、并发锁与熔断见现行主文档。

**例外警示**：截至 2026-09-17，`l_notepad_server` 的 `L_SRC_WATCH` 对 `.py` 改动实测不可靠；Python 改动后必须手动重启。
