# lugwit_baidu_netdisk · Depot blob 按库隔离与上传链路

适用范围：`rez-package-source/lugwit_baidu_netdisk/999.0`（源码即环境）。
本文记录 **blob 存储的隔离模型** 与 **md5 / 字节的上传链路**：先写清改造前的真实行为，
再写本次改造（已落地）的设计，最后列出还没做的第二阶段。

---

## 1. 起因：两个被问到的点

1. **`version_depot/blob/` 下不同库的文件应该隔离**，两个库可以各自有相同 md5 的文件
   （不该出现 `blob/<两位十六进制>/` 这种目录，第一层只能是库名）。
2. **md5 应该以百度云返回的为准**；用户从客户端上传文件到百度云时，**字节不该经过本服务器**。

改造前的实际情况与这两条预期都不一致，下面第 2 节是事实，第 3 节是改造后的模型。

---

## 2. 改造前的事实（已核对源码）

### 2.1 物理路径看着隔离，登记层没有隔离

- `blob_path(apps, md5, library)` 会拼库段 → `{apps}/version_depot/blob/<库>/<md5[:2]>/<md5>`
  （`depot_service.py:35-49`），同一 md5 在不同库算出的路径不同。
- 但 `depot_blob` 主键是 **`md5 TEXT PRIMARY KEY`**（旧 `depot_store.py:34`），一个 md5 **只有一行**。
- `blob_put` 的 ON CONFLICT 规则是「已有非空 `lib_root` 就保留先到那行的 `remote_path`」，
  等价于 **先到先得**：第二个库引用同内容时命中 `blob_get` 去重，**不再复制**，下载仍去
  先到那个库的文件夹取。
- 后果：清掉 A 库文件夹会让 B 库同 md5 的下载断掉；代码里也没有任何地方校验
  `lib_root` 与引用者所在库是否一致。

> 当时 `depot_migrate_blob_lib.py` 头注释写「库之间物理隔离」，与 PK=md5 的去重语义自相矛盾。

### 2.2 md5 全由服务端算，百度返回的 md5 没被采用

- `scan_file()` 读一次文件流同时得到 `size + 整文件 md5 + 各分片 md5`
  （`baidu_netdisk_api.py:556-571`）；`blob_put` 的 md5/size 全部来自它
  （`depot_service.py:176`、`532`）。
- 百度返回的 md5 只被用来断言分片响应里有该字段（`baidu_netdisk_api.py:538-539`、`631-632`），
  **不比对值**；`upload_file_stream` 返回的 `content_md5` 是本地算完塞回去的
  （`:612`、`:641`），调用方也不用。上传后**没有回读 `filemetas` 校验**。
- 秒传靠百度 `precreate.return_type == 2`（`:515-521`、`:606-615`），不是在服务端拿
  md5+size 先探测；此外还有一层服务端库内去重 `blob_get(md5)`。

> ⚠️ **更正（2026-09-14）**：当时判断"百度不给真 md5、只能自己算"是**错的**。它给的是
> **可逆的加密 md5**（AList `DecryptMd5` 算法），我们已移植到
> `baidu_netdisk_api.decrypt_baidu_md5()` / `encrypt_baidu_md5()`，实测 **279/280** 还原正确。
> 现在：读侧可以直接拿远端值当**真 md5**用（`depot_import_tree.py --no-download` 零下载导入）；
> 上传后 `ensure_blob` 会比对还原出的真 md5；上传与秒传仍用自己算的 md5。
> 详见《Rez_pkg/百度云接口元数据实测.md》§2。

### 2.3 字节全程过服务器，没有客户端直传

- `/api/depot/submit`：请求体是**服务端本地路径** → 服务端读盘上传。
- `/api/depot/submit_stream`：`request.body()` 全量进内存 → 写临时文件 → 上传
  （`web_server.py:1615-1644`）。
- `/api/depot/mark_add_stream`：流式落临时文件 → 上传（`web_server.py:1810-1835`）。
- `/api/files/upload_stream` 同理（`web_server.py:1152`）。上传实现是 4MB 分片、服务端
  直连 xpan：precreate → locateupload → superfile2 → create。
- **没有**任何「客户端直传百度、服务端只收 md5」的路径：不下发 `access_token`，也不下发
  `dlink`（`dlink` 由服务端代理转发：`_open_upstream` / `download_dlink_to_path`）。
- 但**缝合点已经留好**：`POST /api/depot/reconcile`（`web_server.py:2116-2141` →
  `reconcile_items`，`depot_service.py:824-872`）的语义正是「执行机自己把内容传进 blob 仓，
  服务端只收 md5/size/action」——今天执行机仍复用服务端上传代码，所以字节还是过服务器。

---

## 3. 改造（已落地）：blob 按库隔离

### 3.1 数据模型

```sql
depot_blob (
  lib_root    TEXT NOT NULL DEFAULT '',   -- 该副本属于哪个库；'' 仅用于迁移前全局池遗留
  md5         TEXT NOT NULL,
  size        BIGINT NOT NULL,
  remote_path TEXT NOT NULL,              -- 该库文件夹里的那份文件
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (lib_root, md5)             -- ← 本次改造：从 (md5) 换成两列
)
```

不变式：**`version_depot/blob/` 的第一层目录只能是库名**；同一 md5 在 N 个库就有 N 行、
N 份物理文件，互不牵连。

### 3.2 读写语义

| 场景 | 行为 |
|---|---|
| 上传判定 | `blob_get(md5, 本库)`：**只认本库那一行**。别的库有同 md5 **不算命中**，本库必须自己有一份 |
| 跨库同内容 | 目标库没有 → 正常走 `ensure_blob` → 百度 `precreate` 命中秒传 → **零上行**，只在目标库文件夹多一个文件对象 |
| 同库重复 | 命中本库行 → 完全跳过网络（零请求），计 `dedup` |
| 读取 | 先 `blob_get(md5, 路径所属库)`，再 `blob_get_any(md5)` 兜底（老数据），最后才是旧布局路径（`blob_path` / `legacy_blob_path` 候选链，`depot_service.py:640`、`web_server.py:2167`） |
| reconcile 校验 | `blob_get(md5, 库)` 必须有；否则 `400 内容尚未上传到 blob 仓: <路径> md5=... （库 /x）` |

### 3.3 改动清单

- `src/lugwit_baidu_netdisk/depot_store.py`
  - `_SCHEMA_SQL`：`depot_blob` 建表带 `PRIMARY KEY (lib_root, md5)`；补幂等迁移：
    `DO $$` 里检测主键列数，单列则 `DROP CONSTRAINT depot_blob_pkey` + `ADD PRIMARY KEY (lib_root, md5)`。
  - 新增 `_norm_lib()`：库标识统一成 `/libname`，空/根 → `''`。
  - `blob_get(md5, library="")`：传库 → 精确匹配；不传 → 任意库（优先遗留空库行）。
  - 新增 `blob_get_any(md5)`：读路径兜底专用。
  - `blob_put()`：`ON CONFLICT (lib_root, md5) DO UPDATE SET size, remote_path`（不再有"先到先得"覆盖规则）。
- `src/lugwit_baidu_netdisk/depot_service.py`
  - `ensure_blob` 文档更新：命中的必须是**本库**行。
  - `submit_files`：`blob_get(md5, root)`；`mark_add`：`blob_get(md5, library_of_path(dpath))` 共用同一 lib 变量。
  - dir 物化下载：`blob_get(md5, 库) or blob_get_any(md5)`。
  - `reconcile_items`：按 (库, md5) 校验，报错文案带库名。
  - `blob_path` 文档：写清「按库隔离 + 跨库复制走秒传」。
- `src/lugwit_baidu_netdisk/web_server.py`
  - 下载解析 blob 行：先本库、再任意库兜底。
- `tools/purge_test_libs.py`：删 blob 行改成按 `(lib_root, md5)` 成对删
  （否则会把真实库同 md5 的行一起删掉）。
- `tools/depot_migrate_blob_lib.py`：`--copy` 的登记改成「按目标库 upsert + 删掉旧的空库行」；
  头注释说明隔离语义。
- 帮助页 `web_help.html`：`depot_blob` 结构、隔离说明、迁移 SQL 同步。

### 3.4 迁移与回滚

- **不需要搬数据**：改造时库里 7 行（`/l_wchat` 4、`/rez_pkg` 3）已经在各自库文件夹里，只改键。
  （注：2026-09-16 起知识库不再各自成库，`/rez_pkg` 的内容已 move 到 `/notes/rez_pkg/`；本句记录的是当时状态。）
- 键变更由 `connect()` 自动完成（幂等），无需单独脚本。
- 回滚：`ALTER TABLE depot_blob DROP CONSTRAINT depot_blob_pkey; ADD PRIMARY KEY (md5);`
  然后恢复旧 `blob_put` 的 ON CONFLICT（会退回"先到先得"语义）。

### 3.5 验收

- `tests/test_depot_api.py` → RESULT PASS（54 项），其中「同内容零上传（dedup）」仍为 `dedup: 1`。
- `tests/test_depot_pending.py` → RESULT PASS。
- 主键核对：`pg_index.indisprimary AND indnkeyatts = 2`；`depot_blob` 分组只有真实库。
- 云端核对：`version_depot/blob/` 下只剩库文件夹（用 `tools/depot_scan_test_libs.py` 复查）。

---

## 4. 第二阶段（未做）：md5 权威来源与客户端直传

目标：**字节不经本服务器**，md5 以百度为准。

1. **md5 权威来源改为百度回读**：上传后调 `filemetas(fs_id)` 取 `md5` 与 `size`，与本地
   `scan_file` 结果比对，不一致就报错（现在只断言字段存在）。`depot_blob.size` 用远端值。
2. **客户端直传**（改造成本主要在客户端）：
   - 服务端不再接收文件体，只接收 `{path, md5, size, remote_path}` 登记（现有
     `reconcile` 已是这个形状）。
   - 上传动作交给执行机/客户端：要么由客户端自己拿用户 OAuth token 直连 xpan，
     要么服务端下发一次性的上传凭证（`locateupload` 域名 + `access_token`）。
   - 需要额外解决：CORS / 凭证最小权限与有效期 / 分片直传的重试与秒传判定（
     `precreate.return_type==2`）/ 服务端如何验证「这份内容确实在网盘上」（`filemetas` 查 md5+size）。
3. **秒传探测前移**：登记时若百度已有该 md5+size（同库），直接标 `rapid`，连客户端上传都省掉。

---

## 5. 相关文件

- 元数据存储：`src/lugwit_baidu_netdisk/depot_store.py`
- 业务编排 / 路径映射：`src/lugwit_baidu_netdisk/depot_service.py`
- HTTP 层：`src/lugwit_baidu_netdisk/web_server.py`
- 百度接口封装（分片上传 / 秒传 / filemetas）：`src/lugwit_baidu_netdisk/baidu_netdisk_api.py`
- 迁移与清理工具：`tools/depot_migrate_blob_lib.py`、`tools/depot_scan_test_libs.py`、`tools/purge_test_libs.py`
- 用户手册（帮助页）：`src/lugwit_baidu_netdisk/web_help.html`（路由 `GET /help`）
