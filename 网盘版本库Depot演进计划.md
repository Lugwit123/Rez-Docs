# 网盘版本库 Depot 演进计划（唯一计划台账）

状态：**计划台账，截至 2026-09-17**。已实现设计见[网盘版本库Depot设计.md](网盘版本库Depot设计.md)；百度 md5 实测事实以[Rez_pkg/百度云接口元数据实测.md](Rez_pkg/百度云接口元数据实测.md)为准。

> **2026-09-26 提示**：本台账只管 T1–T6 这条**存储/上传**主线，不含当天的**页面与本机侧**改动。
> 那些已落地的内容在别处：`Rez_pkg/lugwit_baidu_netdisk.md` §6/§6.2/§17（三栏 + 标签可拖动/跨面板 +
> 条末 `＋`、预览/编辑迁到右栏、中栏↔右栏可拖宽、刷新后恢复、工作区本地树右键）与 §5.6（工作区接口全表，
> 页面已直连服务端不再经托盘）、`Rez_pkg/l_tray.md` §2（托盘 `depot_local_*` 6 个网页动作：
> 目录树 / 变更序号 / 在资源管理器打开 / 新建目录·文件 / 删除到回收站）。

对象包：`rez-package-source/lugwit_baidu_netdisk/999.0`（999.0 源码即环境）。
相关文档：`Rez-Docs/Depot库与工作区方案.md`（待评审方案）、`Rez-Docs/网盘版本库Depot设计.md`（已实现设计）、`Rez-Docs/Rez_pkg/百度云接口元数据实测.md`（md5 实测事实源）。

---

## 0. 实施状态（2026-09-13）

| 任务 | 状态 | 交付物 / 证据 |
|---|---|---|
| T1 md5 权威化 | **实测后改判并落地** | `remote_meta()` / `meta_by_path()`；`ensure_blob` 上传后校验 size（+trusted md5）。**百度 md5 是混淆指纹，不能当权威**（见 T1 实测表） |
| T2 上传字节不过服务器 | **部分已实现，待端到端核实** | WChat 侧已实现「相册直传 + 登录闸门（`/login` 页）+ `static/upload_direct.js?v=4` 拦截层 + 懒回源」；通用 Depot 客户端直传、真实手机/公网流量验收与凭据边界仍待核实。详见本节；细节见 `Rez_pkg/lugwit_baidu_netdisk.md` §14 与 §7。<br>**2026-09-26 补**：① **下行**已全走百度 CDN 直链（缩略图 `max-age=604800` / 原图 dlink `max-age=259200`，含视频，PC 浏览器实测），服务器只发元数据；② **上行**直传仍只在 App 内（浏览器无原生层 → 回退服务端中转）；③ `upload_direct.js` 增加 `group` 转发（相册目标）故版本号 v=3 → **v=4**；④ 服务端中转那步不再整包进内存（`upload_file()` 已流式分片，峰值 ~4MB，见 §18）。 |
| T3 l_wchat 纳管 | **工具已实现并验证** | `tools/depot_import_tree.py`；实测源 686 文件/66.8MB；已导入 motherhood 9 个（CL #241/#242） |
| T4 manifest 优化 | **工具已实现并验证** | `tools/depot_manifest_gc.py`（保留策略 + `--verify` 抽样校验，实测 `CL #242: manifest=4 DB=4 缺=0 多=0 OK`）；`depot_manifest_verify.py` / `depot_manifest_replay.py` 2026-09-20 补齐（auth P7 灾备前置，回环实测：178 份清单 / 868 条版本 → 重建 SQL 覆盖全部、带 setval、apply 幂等；并检出库里既有漂移：**清单引用到的 blob 登记缺 71 条**） |
| T5 blob GC | **工具已实现并验证** | `tools/depot_blob_gc.py`（dry-run 实测：登记行 16 / 无引用 0 / 盘面孤儿 0） |
| T6 dir 快照保留 | **工具已实现** | `tools/depot_dir_snapshot_gc.py`（含"blob 兜底缺失则跳过"强校验；dir 根 `/notes` 的 `.versions` 已有快照） |

顺带修复：`copy_remote` 的正确参数形态（8 种试出来的）、`depot_blob` 按 `(lib_root, md5)` 隔离、
`purge_test_libs` 按 (库, md5) 成对删行、`--clean-orphans` 清旧布局残留 63 个文件。

**待办**：T2 客户端直传；purge 测试库时顺带删掉对应 CL 的 manifest（现在会留下空壳清单）。

**前置（早前已完成）**：blob 按库隔离 —— `depot_blob` 主键改成 `(lib_root, md5)`，
同一 md5 在每个库各有一份物理文件，删某库不影响别的库（跨库复制靠服务端 `filemanager opera=copy`，字节仍过服务器出口；秒传当前不可用，2026-09-16 实测）。T3/T5 都建立在这项隔离语义上。

---

## 1. 目标 / 非目标

**目标**
1. md5 的权威来源改为**百度云返回值**（服务端本地算的只作比对）。
2. **上传字节不经过本服务器**；服务端只做"登记 + 校验"。
3. 存量把 `/apps/Lugwit/l_wchat`（裸目录，6 个业务子目录）**纳管进 depot 库 `/l_wchat`（blob 模式）**。
4. `manifest` 增长可控、可校验、可回放。
5. blob 实体做**引用计数 GC**。
6. dir 模式（库 `/notes`）的 `.versions/vNNN` 快照做保留策略。

**非目标**
- 不改 P4 语义（工作区/have/pending/锁不变）。
- 不做跨账号、不做多租户隔离。
- 不追求"服务端完全无文件流"：`submit`（服务端本机路径）这条便利通道保留，但不再是唯一通道。

---

## 2. 任务分解

每个任务统一给：现状 → 方案 → 改动点 → 验收 → 风险。

### T1 md5 权威化（**已解决：上报值是「可逆的加密 md5」，能还原成真值**）

**实测（2026-09-13/14，本机 OAuth 账号）**

| 项 | 结果 |
|---|---|
| `filemetas` 返回什么 | 32 位字符串，形如 `d208646ebo3157c8d1db9b5f1558ed9a`（含 `o` 等非 hex 字符） |
| 是不是 md5 | **不是裸 md5，但可逆**：算法见 AList `DecryptMd5`，我们已移植 |
| 还原正确率 | `blob/l_wchat` 里 **279/280** 样本：`DecryptMd5(上报) == 真 md5`；`EncryptMd5(真) == 上报` 同样 279/280 |
| 已知内容交叉验证 | 空串的上报值还原后 = `d41d8cd98f00b204e9800998ecf8427e`（空串 md5）✓ |

算法（`baidu_netdisk_api.py` 的 `decrypt_baidu_md5` / `encrypt_baidu_md5`）：
先把 md5 的 4 个 8 位块按 `b1+b0+b3+b2` 重排，再逐位 `nibble ^ (i & 15)`；
**第 9 位特殊**：`chr(n + 'g')` —— 这就是看到 `o/m/p/j/g` 的原因。

**结论**：读侧可以直接拿远端值当**真 md5**用（不必为了算 md5 去下载文件）。

**已落地**
1. `baidu_netdisk_api.py`：新增 `decrypt_baidu_md5()` / `encrypt_baidu_md5()`；
   `remote_meta()` 返回 `{size, md5(上报), md5_real(还原), md5_trusted}`，`md5_trusted` 现在恒真。
2. `meta_by_path()`：上传接口不回 fs_id，按父目录列名匹配后取 meta。
3. `ensure_blob` 上传后校验：**size 一致 + 还原出的真 md5 与本地一致**（不一致抛错不登记）。
4. `tools/depot_import_tree.py --no-download`：**零下载存量导入** —— 从列表还原 md5 →
   云端 `copy` 搬到 `blob/<库>/<md5[:2]>/<md5>` → 登记（源文件一个字节都不经过本机）。
5. `tools/baidu_md5_probe.py`：样本化回归（还原/加密一致率、字符集、接口一致性）。
6. 新增 `tools/baidu_fp_known.py`：已知内容对照实验（`""`/`"abc"`/… 上传后取回值比对）。

**仍保留的自算路径**：`scan_file()` 继续作为兜底与上传侧依据（`precreate` 的秒传依旧用
我们自己算的分片 md5），还原值只在读侧使用；某天百度改了混淆算法，代码会还原失败而不是静默出错。

**验收（已达）**
- 上传后远端还原 md5 == 本地 md5；size 亦然（不符即报错）。
- 零下载导入的 429 个音频文件全部落到 `blob/l_wchat/` 并与库内 md5 对账一致。

---

### T2 上传字节不过服务器（客户端直传）

**统一状态（截至 2026-09-17）**：WChat 侧已实现「相册直传 + 登录闸门（`/login` 页）+ `static/upload_direct.js?v=3` 拦截层 + 懒回源」，并有服务端 / 客户端 / 前端 JS 三套测试（见下）；但这些未构成通用 Depot 客户端直传已完成的证据，也未提供真实手机端到端流量验收。因此 T2 记为“部分已实现，待端到端核实”。保留服务端代理作为回退路径；不得据此断言所有上传字节已绕过服务器。

**现状**：`/api/depot/submit_stream`（`web_server.py:1615-1644` 附近）把整个 body 读进内存；
`mark_add_stream`（`web_server.py:1810-1835` 附近）流式落临时文件；
两者都由服务端再分片上传到百度。没有下发 `access_token`、没有下发 `dlink`。

**已落地（WChat 侧，2026-09-16/17）**：相册直传 + 登录闸门（`/login` 页）+ 前端
`static/upload_direct.js?v=3` 拦截层 + 懒回源；闸门 = 「`lugwit_auth` 登录 + HTTPS（本机回环例外）」。
测试：`tests/test_upload_direct_server_side.py`（服务端自证链路）、
`tests/test_upload_direct_client.py`（外部客户端全链路）、
`l_WChat/999.0/tests/test_direct_upload_local.mjs`（前端 JS 本机验证）。
**仍待**：通用 Depot 客户端直传、真实手机端公网流量验收、凭据边界核实。

**方案（分两档，先 A 后 B）**

**A 档：云端已有内容（零字节）**
- 新端点 `POST /api/depot/register`：body `{path, md5, size, remote_path?, cl_id?}`，
  服务端只做：`file_metas` 校验该 md5/size 在目标库路径存在 → `blob_put` → 进 pending。
  适合"先在网盘里有内容，再登记进 depot"（也正好是 T3 的登记口）。

**B 档：客户端直传百度**
1. 服务端下发上传凭证（**现行端点**：1028 的 `POST /api/upload/prepare` / `POST /api/upload/finish`；
   l_WChat 侧 `POST /api/upload/album/prepare|finish`）返回
   `{access_token（账号级）, upload_host（locateupload 结果）, target_dir, rtype, 分片大小}`。
   复用 `locate_upload_host()`（`baidu_netdisk_api.py:252-282`）与 `precreate()`（`:283-330`）。
   闸门 = 「`lugwit_auth` 登录 + HTTPS（本机回环例外）」。
2. 客户端按 4MB 分片直传 `superfile2` → `create`，全程不经过本服务。
3. 客户端回传 `{path, md5, size, fs_id}` → 服务端 `file_metas` 复核 → 登记 → 进 pending。
4. **秒传探测前移**：登记前先用 `md5+size` 问百度（同库同 md5）→ 命中则直接标 `rapid`，
   客户端连传都不用传。**（2026-09-16 实测不命中，`rapid` 字段仅保留、当前恒为假）**

**改动点**
- `web_server.py`：新增 `register` 与直传端点（现行 `/api/upload/prepare` / `/api/upload/finish`）；`submit_stream` / `mark_add_stream`
  保留为兼容路径（服务端代理），标注"会过服务器"。
- `baidu_netdisk_api.py`：导出 `locate_upload_host` / `precreate` / `create_file` 给 HTTP 层用
  （现在只在内部用）。
- 客户端（`l_notepad_client` / 调度器 / depot 页面）：实现直传 + 回执；页面直传可先做
  （浏览器 → 百度 CORS 限制待实测，若不通则退回 A 档：先上传到网盘再登记）。
- 鉴权（**原"凭证最小权限、短有效期"结论错误，2026-09-17 更正**）：下发的是**服务器自己的账号级 `access_token`（同一把，约 30 天，无法单独撤销）**；`uploadid`/`upload_host` 只是会话票据；客户端不接触授权码 `code` 与 `refresh_token`；`LUGWIT_UPLOAD_TICKET_TOKEN` 开关只能**停止继续下发**，追不回已发出的，真收回只能撤销应用授权（服务端也需重新授权）。

**验收**
- 客户端直传一个 20MB 文件：服务端进程网络出口流量 ≈ 0（用 `netstat`/计数验证），
  depot 版本、md5、size 与百度 `filemetas` 一致。
- 服务端只收到 1 个登记请求（无文件体）。

**风险**
- 百度 CORS / 凭证泄露 / 分片重试与秒传判定差异 → 保留服务端代理兜底。
- `reconcile` 已有同类语义（`web_server.py:2116-2141`），T2 只是把上传动作也移出服务端。

---

### T3 存量迁移：`/apps/Lugwit/l_wchat` → depot 库 `/l_wchat`（blob 模式）

**现状**（实测）
```
/apps/Lugwit/l_wchat/           ← 裸目录，L_WChat 业务数据，未纳管
├─ album/  diet_img/  motherhood/  公众号每日推送文章/  成长记录/  音频/
/apps/Lugwit/version_depot/blob/l_wchat/  ← depot 库，blob 模式，当前只有 4 个 md5 实体
```
`/l_wchat` 库已存在（blob 模式），所以这是**把裸目录内容导入同一个库**，不是新建库。

**实测（2026-09-13）**
- `filemanager opera=copy` **可用**，但参数形态极挑（试了 8 种）：
  `filelist` 必须是**对象数组、每项自带 `dest`**（可带 `newname`）、**不能带 `async`**、
  同一次调用里不能重复同一个 src、目标目录必须先存在。成功返回 `info[i].to_path`/`to_fs_id`。
- 因此"用百度 md5 直接算目标路径"这条路**不通**（百度 md5 是混淆指纹，见 T1）：
  要真 md5 必须自己读一遍字节。
- 源目录规模：**686 个文件 / 66.8MB**
  （album 7/7.8MB、diet_img 165/36.8MB、motherhood 9/7.2KB、
  公众号每日推送文章 70/472.6KB、成长记录 14/9.3KB、音频 421/21.7MB）

**方案（下载一次算 md5 + 云端复制搬运，零上行）**
1. `copy_remote(access_token, items[{src,dest,newname}])`：已加入 `baidu_netdisk_api.py`。
2. 每个文件：`dlink` 下载到临时文件 → `scan_file()` 得 `(size, md5)` → `blob_get(md5, "/l_wchat")`：
   - 命中（同库已有）→ 只登记逻辑路径，**不复制、不占空间**；
   - 未命中 → `mkdir_p` 目标 `blob/<库>/<md5[:2]>/` → `copy` 原文件过去、用 `newname=<md5>` 改名
     → `blob_put(md5, size, remote_path, "/l_wchat")`。**零上行**（只有 1 次下载用于算 md5）。
3. 整批一个 changelist：`store.submit(owner, "导入存量: <批>", entries, ws_id)`
   （rev 递增 + have 推进 + manifest 一把过）。
4. 幂等：`--skip-existing` 默认开——目标库已有该逻辑路径的 rev 就跳过（连下载都省）。
   分批 `--batch <顶层子目录>`，失败可单独重跑。
5. 源目录**不删**：导入完成、抽检下载正常后，再由人工决定重命名/删除（另一步、另行确认）。

**已实现**：`tools/depot_import_tree.py`（默认 dry-run 只盘点；`--yes` 执行；
`--limit N` 小批量验证；`--batch` 分批；`--ws` 指定工作区，默认新建 `<owner>-import`）

**验收**
- `GET /api/depot/list?dir=/l_wchat` 能列出原目录树；随机抽 5 个文件
  `download` 内容与源一致（md5+size 比对）。
- `depot_blob` 里 `/l_wchat` 行数 = 导入的不同内容数；`blob/l_wchat/` 下只有库内分片目录。
- 源目录仍在（未删）。

**风险**
- `filemanager opera=copy` 是否支持跨目录复制、是否有单目录条目上限（`_apps` 下 `专辑/成长记录`
  可能条目很多）→ 待实测；失败降级为下载重传。
- 同名不同内容（同一路径多个历史版本）在裸目录里不存在（裸目录只有当前态），
  所以导入只产生**一个初始版本**，历史从导入点开始。

---

### T4 manifest 优化

**现状**：`manifest_path()` = `.depot/manifest/<cl_id//1000 四位>/<cl_id 八位>.json`
（`depot_service.py:66-67`），每次提交写一份（`:206-240`、`:324-342`、`:583-599`），
失败仅告警。**没有任何清理逻辑**；实测 `.depot/manifest/0000/` 已约 140 份（cl_id 5→240）。

**方案**
1. **保留策略**：`keep_recent_buckets`（默认 1 个桶 = 最近 1000 个 CL）+ `keep_min`（默认 200 份）
   取并集，其余按桶删除；提供 `tools/depot_manifest_gc.py --dry-run/--yes`。✅ 已实现
2. **一致性校验**：`tools/depot_manifest_verify.py` 抽样比对
   `manifest.json` ↔ `depot_file_rev`/`depot_blob`（路径集合、md5、size），输出差异清单。
   ✅ **已实现**（2026-09-20，供 auth 灾备 P7 用）：递归读清单目录 → `(path,rev)` 唯一性与
   rev 递增检查 → 与库里 `depot_file_rev`（action/md5/size）逐条比对 → 「清单有库里没有」/
   「库里有清单没有」/「字段不一致」/「blob 登记缺失」四类漂移 + `--json` 报告 + 退出码
   （0 一致 / 1 漂移 / 2 清单不可解析）。
3. **回放工具**：`tools/depot_manifest_replay.py` 在空库上按序重建
   （DB 灾难恢复用）；先只做 dry-run + 生成 SQL/CSV，不直接写库。
   ✅ **已实现**（默认 dry-run 出 SQL、`--sql-out` 落文件、`--apply` 单事务写库）：
   重建 `depot_changelist` + `depot_file_rev`（+ `depot_blob` 登记，`--blob-mode ensure|only-existing|none`），
   并补 `setval` 对齐自增序列（否则回放后新建 CL 撞主键）。`--from-cl/--to-cl` 可分段。
   ⚠️ `blob-mode=ensure` 只补"登记"，不校验网盘上物理文件是否存在。
4. 写 manifest 改成**幂等 upsert**（现在是覆盖写文件），并对失败加一次重试，
   避免"提交成功、兜底清单缺失"。⏳ 未做

**验收**
- GC dry-run 输出"将删 N 份 / 保留 M 份"，确认后执行，`.depot/manifest` 只剩保留集；
- verify 对现有数据报 0 差异（或列出已知差异）；
- replay 在临时 schema/临时表上能重建出与现库一致的行数与 md5 分布。

**风险**：manifest 是 DB 丢失时唯一的重建依据 → GC 必须保留最近桶 + 最少份数，且删除前
先做一次 `tools/depot_dump_tables.py` 全量备份。

---

### T5 blob 引用计数 GC

**现状**：设计是"删版本/删库都不动内容"，所以 `blob/<库>/<2位>/<md5>` 只增不减；
孤儿只能靠 `tools/depot_migrate_blob_lib.py --clean-orphans`（判据：md5 不在任何
`action<>'delete'` 的 rev 里）手动清。今天是手工清了 63 个。

**引用来源（必须全算进去，漏一个就会误删）**
1. `depot_file_rev.blob_md5`（`action <> 'delete'`）
2. `depot_pending_file.blob_md5`（待提交里引用的内容）
3. dir 模式快照 `.versions/vNNN`（历史版本要以 blob 实体为源，见 T6）
4. `depot_blob.remote_path` 自身（行还在就保留）
5. 迁移脚本/人工产物的白名单（如 `.depot/blob` 兜底副本，见 T6 决策）

**方案**：`tools/depot_blob_gc.py`
- 计算 `live = {(lib_root, md5)}`（上述来源并集）；
- 候选 = `depot_blob` 中不在 live 的行 → 其 `remote_path` 文件；以及库里盘面上存在但
  DB 无登记、且不在 live 的实体（扫描 `blob/<库>/<2位>/`）；
- **默认 dry-run**，输出：可回收条数/文件数/字节数、按库分组、抽样路径；
- `--yes` 执行：先删云上文件 → 再删 DB 行（与现有工具一致：云端先动，DB 后动，失败即中止）；
- 保护：`--keep-legacy-fallback`（默认开）保留 `.depot/blob` 下被引用 md5 的兜底副本；
- 每次执行前自动跑一次 `depot_dump_tables.py` 备份到 `~/.lugwit/depot_backup/<stamp>/`。

**验收**
- 在测试库上造"提交→删版本→GC"闭环：GC 后该 md5 的实体消失、别的库同 md5 的实体**仍在**
  （隔离回归点）、真实库下载仍 200；
- dry-run 与实际删除条目数一致；失败中途不留"DB 有行、云无文件"。

**风险**：误删 = 内容永久丢失（网盘回收站有时限）→ dry-run + 备份 + 隔离语义（每库一份）
是三重保护；先在测试库跑通再对真实库。

---

### T6 dir 模式快照保留

**现状**：dir 模式（库 `/notes`）每次提交既写活文件又留一份
`dir_mirror/<父目录>/.versions/<名>/vNNN/<名>`（`depot_service.py:78-92`），
**每版本一份**，是膨胀最快的部分；没有保留策略。

**方案**：`tools/depot_dir_snapshot_gc.py`
- 保留集 = 最近 K 版（默认 5，可配）+ 最新版 + 被 `depot_have` 引用的版本 +
  被 `depot_lock` 持有路径的当前版 + `--keep-rev` 显式指定；
- 回收其余 `.versions/vNNN` 目录；活文件不动；
- 删除只影响"快照副本"，历史版本内容仍可从 blob 实体重建
  （dir 模式的 `_dir_mode_fs_id` 已有"blob 仓兜底"逻辑，`web_server.py` 下载路径），
  因此该 GC **不丢历史**，只丢冗余快照 —— 这点必须在执行前用抽样下载验证。
- 默认 dry-run；执行前打印"将删 N 个快照版本 / 涉及 M 个文件"，并要求
  `--yes`；建议配 `LUGWIT_DEPOT_DIR_KEEP_VERSIONS` 作为默认策略。

**验收**
- 对某文件抽 3 个被回收的 rev 执行 `download` → 内容仍正确（说明走了 blob 兜底）；
- `dir_mirror` 文件数下降到保留集规模；`list`/`history` 结果不变。

**风险**：若某版本既无快照、blob 实体又被 GC 掉（T5 与 T6 交叉），历史就真丢了 →
**顺序要求：先跑 T4/T6，最后跑 T5；且 T5 的 live 集合必须包含 T6 保留集**。

---

## 3. 依赖与实施顺序

```
T1 (md5 权威) ──┬─> T2 (直传/登记)      [T1 是 T2 的校验基础]
                └─> T3 (l_wchat 纳管)   [T3 用 T1 的 md5 回读 + copy 搬运]

T4 (manifest) ─┐
T6 (dir 快照) ─┼─> T5 (blob GC)         [T5 的 live 集依赖 T4/T6 的保留集]
```

- 可并行：T4 与 T6 互不依赖；T2 的 A 档（register）可先做，B 档（直传）后做。
- 建议里程碑：
  - M1：T1 + T2-A + T3 dry-run（一周内可交付，收益：md5 可信 + 存量可见）
  - M2：T3 实迁 + T4（清理 + 可校验）
  - M3：T6 + T5（空间回收闭环）
  - M4：T2-B（客户端直传，含客户端改动与联调）

---

## 4. 通用约定（每个任务都要遵守）

1. **只读优先**：任何删/迁工具默认 `--dry-run`，`--yes` 才执行，输出"计划 → 执行 → 复核"三段。
2. **先云后库**：删除顺序一律"云端文件 → DB 行"，失败即中止，避免"DB 有行、云无文件"。
3. **备份**：执行前 `tools/depot_dump_tables.py` 全量 dump 到 `~/.lugwit/depot_backup/<stamp>/`。
4. **隔离不破坏**：任何回收都要验证"别的库同 md5 实体仍在"。
5. **守恒校验**：执行后跑 `tools/depot_scan_test_libs.py`（只读）+ 抽样 `download`。
6. **文档同步**：改动 `depot_blob`/路径形状/接口时，同步
   `web_help.html`（`GET /help`）与《网盘版本库Depot设计.md》。

---

## 5. 工具清单（本计划新增 / 改动）

| 文件 | 任务 | 说明 |
|---|---|---|
| `tools/depot_import_tree.py` | T3 | 裸目录 → depot 库（copy 搬运 + 初始版本），dry-run 优先 |
| `tools/depot_register_cli.py` | T2-A | 本地批量登记（md5/size 已在云端） |
| `tools/depot_manifest_gc.py` | T4 | manifest 保留策略 |
| `tools/depot_manifest_verify.py` | T4 | manifest ↔ DB 一致性校验 |
| `tools/depot_manifest_replay.py` | T4 | 空库回放重建（灾备） |
| `tools/depot_dir_snapshot_gc.py` | T6 | dir 模式 `.versions/vNNN` 保留策略 |
| `tools/depot_blob_gc.py` | T5 | blob 引用计数 GC |
| `baidu_netdisk_api.py` | T1/T3 | 新增 `remote_meta()`（取 md5）、`copy_remote()`（filemanager opera=copy） |
| `depot_service.py` | T1/T2 | 上传后回读比对；`register_items()`（只登记不传字节） |
| `web_server.py` | T2 | `/api/depot/register`、`/api/upload/prepare`、`/api/upload/finish`（l_WChat 侧 `/api/upload/album/prepare|finish`） |
| 客户端包 | T2-B | 直传实现 + 回执 |

---

## 6. 待实测确认（写进实现前必须先验证的假设）

1. `filemetas` 是否返回 `md5`（若否，T1 退化为本地算 + size 校验）。
2. `filemanager opera=copy` 是否支持跨目录复制、是否计入配额、是否有单目录条目上限。
3. ~~同账号内"秒传"是否真的不额外占用空间（影响 T3/T5 的空间收益估算）。~~ **已关闭（2026-09-16/17）：秒传不可用，该问题不适用。**
4. 浏览器直传百度是否受 CORS 限制（决定 T2-B 是"页面直传"还是"客户端直传"）。
5. `.versions/vNNN` 快照删除后，`history` + `download` 是否确实走 blob 兜底（T6 前置）。

---

## 7. 完成定义（DoD）

- T1：所有新登记行的 `md5/size` 来自百度回读，且与本地一致；不一致必报错。
- T2：存在一条"客户端直传百度 → 服务端只收登记"的通路，服务端无文件体流量。
- T3：`/l_wchat` 库能列出原裸目录全部文件，抽检内容一致，源目录未动。
- T4：manifest 有保留策略 + 校验 + 回放；目录数量停止无界增长。
- T5：blob 实体有 GC 闭环（dry-run/执行/复核），且不破坏别的库。
- T6：dir 快照有保留策略，历史版本仍可下载。
