# 网盘版本库（Depot）设计与实现

涉及包：

- `lugwit_baidu_netdisk/999.0` — blob 仓 + 元数据 + Web 页面
- `lugwit_netdisk_client/999.0` — PC 客户端（PySide6 + QWebEngineView）

## 1. 背景

跨地区传文件，公网带宽太贵。改用百度网盘做中转：
上传方推到网盘，下载方从网盘拉，走网盘自己的 CDN，不吃自建带宽。

在此之上加一层 **简化版 P4 版本管理**：改名/移动/回滚不产生任何流量。

## 2. 核心设计：内容寻址 blob 仓 + 数据库唯一权威

### 2.1 网盘只存 blob，只追加

```text
/apps/Lugwit/.depot/blob/<md5[:2]>/<md5>
/apps/Lugwit/.depot/manifest/<cl//1000:04d>/<cl:08d>.json
```

- blob 用 md5 命名 → 同内容全库只存一份
- 网盘里**没有目录结构**，路径语义全在数据库里
- 因此：重命名 = 改一行数据库；回滚 = 新版本指向旧 blob。**零字节流量**

### 2.2 manifest 兜底

每次提交额外写一个几 KB 的 JSON（CL 号、提交人、说明、文件列表 + md5）。
数据库炸了，按 CL 顺序重放 manifest 即可完整重建元数据。

### 2.3 为什么不选另外两种

| 方案 | 否决理由 |
|------|----------|
| 网盘按真实目录结构存 | 改名/移动要真搬文件，跨地区重传 |
| 全量元数据只放网盘 JSON | 并发提交无法加锁，列目录要拉一堆小文件 |

## 3. 数据库表（落在 `chatroom` 库）

`depot_store.py`，asyncpg 裸连接池（复刻 `account_service.py` 的 URL 处理）。

| 表 | 作用 | P4 对应 |
|----|------|---------|
| `depot_blob(md5 PK, size, remote_path)` | 内容 → 网盘路径 | — |
| `depot_changelist(id, owner, description, status)` | 变更列表，`status`=pending/submitted | CL |
| `depot_pending_file(cl_id, path, action, blob_md5, src_path, base_rev)` | 待提交区条目 | opened files |
| `depot_file_rev(path, rev, action, blob_md5, cl_id, owner)` | 文件某个版本 | revision |
| `depot_have(owner, path, rev) PK(owner,path)` | 某人本地是哪个版本 | have 表 |
| `depot_lock(path PK, owner)` | 独占签出 | `p4 edit` |

唯一索引：

- `ux_depot_file_rev ON depot_file_rev(path, rev)`
- `ux_depot_pending_path ON depot_pending_file(owner, path)` — 一人对一路径只能有一个待提交动作
- `ux_depot_cl_default ON depot_changelist(owner) WHERE status='pending' AND description=''` — 每人一个默认待提交列表

老库升级靠 `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`，不用手动迁移。

### 3.1 两步工作流（和 P4 一致）

```
p4 edit   → checkout()        加锁 + 进 depot_pending_file
p4 add    → mark_add()        内容先进 blob 仓，条目进待提交区（还没有版本号）
p4 delete → mark_delete()     标记待删
p4 move   → mark_move()       待提交区记一条（path=目标, src_path=源）
p4 revert → revert_pending()  从待提交区移除 + 解锁，不产生任何版本
p4 submit → submit_pending()  整批原子落库
```

关键点：**submit 之前 `depot_file_rev` 里什么都不写**。所以撤销是真撤销，不留垃圾版本。
blob 已上传但无人引用 → 孤儿，靠 GC 回收（尚未实现）。

`submit_pending()` 一个事务内：

1. `SELECT ... FOR UPDATE` 锁 CL 行，校验 status / owner
2. 展开 pending 条目 —— **`move` 一条展开成两条**（目标 `move` + 源 `delete`）
3. 逐路径校验 `depot_lock`，被别人锁住 → `DepotLocked` → HTTP 409
4. CL 改 `status='submitted'` + 写 `submitted_at`
5. 每条算 `max(rev)+1` 写 `depot_file_rev`，推进 `depot_have`，清自己的锁
6. 清空该 CL 的 `depot_pending_file`

### 3.2 一步直提仍然保留

老的 `submit_files()` 走 `store.submit()`，不经待提交区，一次调用直接产生版本。
`/api/depot/submit_stream` 和自动化脚本还在用。两条路并存，互不干扰。

`DepotLocked` 带 `.path` / `.owner`，Web 层映射成 HTTP 409。

### 3.3 其他查询

- `list_dir(prefix)`：`SELECT DISTINCT ON (path) ... ORDER BY path, rev DESC`，
  折叠子路径成目录，跳过 `action='delete'`
- `sub_paths(prefix)`：递归取存活文件，整目录移动用
- `sync_plan(owner, prefix)`：CTE `latest` LEFT JOIN `depot_have`，
  条件 `h.rev IS DISTINCT FROM l.rev` → 待同步清单
- `pending_status(owner, prefix)`：给文件列表标状态图标用

## 4. 上传层

`baidu_netdisk_api.py`。

### 4.1 秒传（之前是错的）

`precreate` 返回 `return_type == 2` 就是**秒传命中，零上行**。
旧代码把返回的空 `block_list` 当成"所有分片都要传"，白传一遍。现已在
`upload_file` 和 `upload_file_stream` 两处都拦截。

### 4.2 流式分片（10GB 文件不进内存）

```python
scan_file(path)  -> (总字节, 整文件 md5, 各分片 md5)   # 单次遍历
read_chunk(path, i)                                    # 按需 seek
```

`precreate` 返回的 `block_list` = **还缺哪些分片** → 天然断点续传。

## 5. 提交流程

`depot_service.submit_files()`：

```text
scan_file 得 md5
  → store.blob_get(md5) 命中？ 零网络，直接复用      (dedup)
  → 否则 upload_file_stream
       → precreate return_type==2 ?  秒传，零上行     (rapid)
       → 否则只传缺失分片                             (uploaded)
  → store.submit() 事务写元数据
  → write_manifest()   失败只记日志，不影响提交
```

返回统计：`{"uploaded", "rapid", "dedup", "bytes"}`。

其他操作：

- `revert_to(path, rev)` — 新建一版指向旧 blob，零流量
- `delete_files(paths)` — 写 `action='delete'`，blob 保留（历史仍可取）

## 6. Web 接口

`web_server.py`，页面 `/depot`，API 前缀 `/api/depot/`：

```text
status  list  history  changes  change/{cl_id}
submit  submit_stream  revert  delete
lock  unlock  locks  sync_plan  download
```

约定：

- 身份取 JWT `sub` 字段（`_owner()`），缺失 → 401
- `_depot_err()` 统一映射：`DepotLocked`→409，`ValueError`→400，其余→500
- `_depot_ctx()` 一次拿齐 `(store, access_token, apps_root)`，库不可用 → 503
- `/api/depot/download` 解析 blob 的 `fs_id` → `dlink` → `StreamingResponse`，
  带 `X-Depot-Rev` 响应头

页面 `web_depot.html` 标题栏抄 `l_notepad_server/templates/web_edit.html` 的
`.topbar` / `.topbar-meta` 样式；两栏布局 `minmax(0,1fr) 400px`，左侧文件表 + CL 视图，
右侧版本历史面板。

## 7. 路径安全

`normalize_depot_path()` 拒绝：空路径、含 `..`、首段为 `.depot`。

## 8. 客户端

见《Rez包创建指导文档》第 12 章（QtWebEngine 两个坑 + QWebChannel 桥）。

```bat
wuwor lugwit_netdisk_client -- netdisk_client_doctor   :: 体检
wuwor lugwit_netdisk_client -- netdisk_client          :: 启动
```

## 9. 已验证

真实网盘 + 真实 `chatroom` 库跑通：提交 / 去重 / 秒传 / 历史 / 回滚 / 加锁 /
锁冲突 409 / sync_plan / blob 路径 / manifest 路径。测试数据已清理（errno 0）。

## 10. 已知未做

- errno 31034 限流退避
- 分片固定 4MB（VIP 账号可用 32MB）
- 并行 Range 下载
- 孤儿 blob GC
- 客户端 `fileMd5` 尚未接进提交流程（本地探秒传）
- 传输 daemon（SQLite 任务表 + 断点续传）

## 11. 明确不做

LAN 直连 / WireGuard 打洞 / Headscale / `Transport` 抽象层。
只做网盘一条路。过早抽象比不抽象更糟。
