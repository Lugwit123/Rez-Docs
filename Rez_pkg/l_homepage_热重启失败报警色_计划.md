# 计划：热重启失败 → 卡片报警色（含根因修复）

状态：**阶段 0、1 已落地，阶段 2 待做**　编写日期：2026-09-21
关联：[l_homepage.md](l_homepage.md)、[src_hot_reload_源码热重载与主页常驻.md](../src_hot_reload_源码热重载与主页常驻.md)

---

## 一、背景与现象

以「L Agent Chat」卡片为例：改 `.py` 触发源码热加载重启时，出现过"热重启失败"——
卡片状态仍显示"运行中/未启动"，界面上看不出任何异常；服务也可能长时间起不来。

实测确认：**热重载机制本身是好的**。touch `l_agent_chat/999.0/src/l_agent_chat/config.py`
后进程 `48204 → 120736`、`/health` 正常、`boot_id` 变化，重启成功。

失败是"重启过渡期被第二路人插手"，`~/.lugwit/l_homepage/runtime/restart_history.jsonl`
里 L Agent Chat 的失败记录：

| 时间 | 记录 | 含义 |
|---|---|---|
| 2026-09-21 23:32:29 | `start/watchdog`：`旧进程未退出：端口 1250 仍被 PID 120736 占用，本次start已取消` | 常驻守护在热重载重启窗口里抢着启动，撞车被取消 |
| 2026-09-21 17:30:14 | `start/watchdog`：`taskkill 均失败: PID 102128` | 1250 上有 taskkill 杀不掉的残留监听者 |
| 2026-09-21 15:07 | `⚠ 热重载熔断：180s 内已重启 4 次，暂停自动重启 60s` | 短时间多次存 `.py` 触发熔断，之后改代码"没反应" |

## 二、根因

`l_app_ready/hotreload_service.py::SrcWatchService.restart_self()`：
在 `Popen` 拉起新实例后**立刻在 `finally` 里释放重启锁**，但新实例还要几秒才 bind 端口。

主页 watchdog（`homepage_cli.py::_watchdog_tick` → `_restart_in_progress`）每 60s 只在
tick 开头查一次锁；一旦落在"锁已释放、端口还没起来"的空窗，就判定服务掉线并去
`_svc_manage(name,"start")` 启动竞争实例 → `旧进程未退出 / taskkill 均失败` 并取消。

L Agent Chat 启动慢（rez 解析 + uvicorn 约 5–20s），空窗更宽，所以最易中招。

> 结论：**热重启"失败"不是重启本身失败，而是重启空窗里 watchdog/卡片动作抢跑，两路各起一个实例。**

## 三、阶段 0（已完成）

目标：热重启失败在卡片上一眼可见（红色报警色）。

1. `homepage_cli.py::latest_restart_map()`：每张卡最近一次重启记录补 `ok / error / age`
   （原来只给 ts/op/trigger/pid，失败信息被丢弃）。
2. `templates/home.html`：
   - 新增 `.card.alarm` 样式（红色描边 + 呼吸红光 + 红色渐变边框）。
   - `paint()`：最近一次重启 `ok===false` → 加 `alarm`；**服务没起来长期报警；已恢复则失败后
     5 分钟内保留后自动褪去**；`title` 显示失败原因与时间。
   - `svcManage()`：手动 重启/热更新/热启动 失败立即报警（后端历史要等补齐新 PID 才落盘，
     用本地 `_alarmPending` 占位顶上，最长 60s）。
   - `paintBusy()`：进入过渡态先撤报警，结果出来再由 `paint` 决定。

验证：`http://127.0.0.1:8090/`，L Agent Chat 卡片 class = `card alarm`，tooltip 含失败原因，
无 JS 报错，其余卡片不受影响。

## 四、阶段 1：修根因（重启锁释放时机）【已落地】

**问题**：锁在新实例"拉起"就释放，而非"可用"才释放，留下抢跑空窗。

**实现（方案 A）**：`hotreload_service.py`
- 新增 `SrcWatchService._wait_new_listener_then_release(trigger, old_pids)`：
  等出现"不在旧监听列表里的新监听 PID"才 `release()`；
- 等待期间每 5s `RestartGuard.renew()` 续期，避免 `LOCK_TTL`(30s) 到期失效；
- 超时（默认 25s）仍释放并限频告警，交回 watchdog/用户接管；
- 参数 `L_SRC_WATCH_READY_TIMEOUT`（默认 25s，`0` = 退回旧行为），沿用统一前缀；
- `restart_self()` 不再用 `finally` 立刻释放，改为拉起成功后调用上述等待。

**验证**：touch `l_agent_chat/config.py` 实测——
`t≈6.8s` 抢锁（旧进程仍在监听）→ `t≈10s` 旧进程已停、锁**保持** → `t≈23.1s` 新 PID 监听且锁释放；
`t=10~23s` 整个 down 窗口锁都在，watchdog 会被 `restart_in_progress` 挡开。
`test_restart_lock.py` PASS=3 FAIL=0。

**遗留**：`LOCK_TTL` 仍 30s（靠 renew 续期）；`l_homepage` 自身的 `_spawn_self_restart` 是独立
实现，未走此路径（主页不 watchdog 自己，影响小）。

## 五、阶段 2：覆盖"服务自热重载失败（熔断）"

**现状缺口**：服务自己的源码热加载重启（`src-watch`）不写进主页 `restart_history.jsonl`，
所以熔断/自重启失败在卡片上仍无提示；只有"服务掉线 → watchdog 失败"才会报警。

**方案（递增）**：

1. **复用现有状态接口**：主页在 `_gather_status` 里对每张服务卡并行探测其
   `/__dev__/src_watch`（各服务路径不同：`/__dev__/src_watch`、`/api/src_watch`、
   `/api/v1/auth/src_watch`…），读 `breaker.remaining>0 / restart_lock` → 卡片加"熔断中/重启中"
   态。需维护"卡片名 → 接口路径"映射，代价中等。
2. **服务回报主页**：服务自重启成功后回调主页登记一条事件（需新增内部接口与鉴权），
   数据最准，改动最大。
3. **保持现状**：只依赖阶段 0 的"掉线后失败报警"。最省，但熔断空窗仍不可见。

**建议**：先做阶段 1（根因），阶段 2 视需要选 1。

## 六、验证与回归

- 单实例/端口：热重载后 `Get-NetTCPConnection -LocalPort <port>` 只有一个 LISTEN。
- 报警色：
  - 故意让服务起不来（如临时写坏 `app.py`）→ 卡片红色且长期保持；
  - 修好并起来 → 5 分钟内自动褪色；
  - 正常卡片不受影响（只 `L Agent Chat` 变红）。
- 手动链路：卡片「重启 / 热更新 / 热启动」失败 → toast 报错 + 卡片立即变红。
- 主页自身：改 `homepage_cli.py` 触发 `restart-self`，确认报警不误伤主页卡。
- 前端：浏览器 console 无报错；暗/亮主题下红色可见。

## 七、风险与回滚

- 阶段 0 纯前端展示 + 只读字段，回滚 = 还原两处文件即可，无数据影响。
- 阶段 1 改共享组件 `l_app_ready`，影响全部接入服务；回滚 = 还原 `restart_self()` 的
  锁释放逻辑。上线前在 1~2 个服务（如 `l_WChat`、`l_agent_chat`）灰度验证。

## 八、关键代码位置

- `wuwo/packages/l_app_ready/1.0.0/src/l_app_ready/hotreload_service.py`
  - `SrcWatchService.restart_self`（锁释放时机，阶段 1）
  - `RestartGuard.acquire/release/in_progress`、`RESTART_WINDOW/RESTART_MAX/BREAKER_*`（熔断）
- `l_homepage/999.0/src/l_homepage/homepage_cli.py`
  - `latest_restart_map`（阶段 0 已改）、`_watchdog_tick`、`_restart_in_progress`、
    `_svc_manage_impl`、`_record_restart*`
- `l_homepage/999.0/src/l_homepage/templates/home.html`
  - `.card.alarm` 样式、`paint()`、`paintBusy()`、`svcManage()`、`_alarmPending`
- 运行期状态：`~/.lugwit/l_homepage/runtime/{restart_history.jsonl,restart_guard.json,watchdog.log}`；
  服务侧 `~/.lugwit/<pkg>/runtime/{restart_guard.json,src_watch.json}`；
  锁 `%TEMP%/lugwit_hotreload/<alias>.lock`
