# HDA 资产模型 · 对照与借鉴（计划）

> 2026-10-19。用户口径：「houdini hda 的设计我认为是优秀的，可以学习么」→「写一个计划文档」。
> **这份文档只记"学哪几件、怎么落、判据是什么、哪些明确不学"**；HDA 的通用科普、我们的节点/图引擎
> 现状分别看 `Rez-Docs/Rez_pkg/l_node_engine.md` / `l_mindmap_mmd.md` / `工具契约_ToolSpec.md`。
> ⚠ 本文**只是计划**：P0 之前一行代码都还没动。

## 0. 一句话

我们的「图工具 / 子图 / 图级 `params` / `GetParam` 胶囊」拼起来**已经是 HDA 的骨架**（声明 + 实现 + 端口 +
提升），缺的是 HDA 那套**声明面的成熟件**：参数模板（范围/步进/分组）、条件显隐、资产版本、节点钩子，
以及把声明从 `.py` 挪进独立文件的能力。**学它 = 补声明面，不动执行模型。**

### 0.1 `.hda` 这个"容器文件"对应我们什么（2026-10-19 用户追问）

**一句话：我们没有和 `.hda` 对等的单文件。** `.hda`（老叫 `.otl`）是一个**封装容器**：一个文件里装着
类型属性（参数表 + 端口）+ 内部网络 + Python Module + 图标/帮助，扔进资产库就能被任何场景按类型引用。
它的等价物在我们这边是**一组文件**：

| `.hda` 里的一块 | 我们的对应物 | 落地位置 |
|---|---|---|
| 类型属性 · 参数表 | 图级 `graph["params"]`（图工具）／`ToolSpec.parameters`（python 工具，`tool_spec.py:105`） | 图 JSON / `ToolSpec` |
| 类型属性 · Inputs/Outputs | 图里的 `Output` 节点 + 端口现算（`/api/pygraphs/ports?kind=subgraph`） | 图 JSON |
| 内部网络（实现） | **图本体**：`<工程>/Tools/x.json` 的 `nodes` / `edges` | 图 JSON（或 python 类的 `execute`） |
| Python Module / 钩子 | 节点类所在 `.py` 模块（`param_labels.py` / 类文件）／图里的 `PythonCode`／**计划 P3 的 `hooks`** | `.py` / 图 |
| 图标 / 帮助 / 分类 | `ToolSpec.ui.icon` + `graph.note` / `tag`；类的 `DESCRIPTION` / `SEARCH_ALIASES` / `CATEGORY`（`icons.py`） | ToolSpec / 图 JSON / `.py` |
| 定义库 + 版本 | rez 包目录（`<名>/<版本>/`）+ 工程资产目录 + `plugins.json`；**计划 P4 的 `nodes/*.json` = 一个文件一个库（可装多个节点）** | 目录为主，单文件库还没做 |
| 二进制封装 / 加密 | **开发态明文、发布包可加密**：`.nlib`（AES-GCM，见 P4c）；**不学 HDA 的二进制定义格式**（见 §四） | 明文源 + 打包工具 |

所以：

- **最接近"一个文件 = 一个资产"的**：`<工程>/Tools/x.json`（图工具 = 一张普通图，带 `params` / `note` / `tag`）；
- **它当节点用时**：靠 `SubGraph` 内联跑那句图（`subgraph.py`），或当工具给 agent 调（`graph_tool_spec`）；
- **要带代码就别想单文件**：python 工具 = `ToolSpec{kind:"tool", source:{type:"python", module, class}}` + 那个模块，
  和 `.hda` 一样最后还是"描述 + 代码"两处，只是 `.hda` 把它们压进了一个压缩包；
- **计划里的 P4 `nodes/*.json`** 才是"一文件一节点定义"的正式对应物，而它**只定义节点**（不含内部网络），
  内部网络通过 `exec: {kind: pygraph, graph: pro/Tools/x}` **引用现成的图**。

⚠ **一处真实差异（P2 要处理）**：HDA 把**定义（库里的类型）**与**实例（场景里那个节点，只有类型 + 参数值）**
分得很干净；我们的图 JSON **既是定义又是实例**（被 `sub` 引用时是定义，打开时又是可编辑实例）。
所以才需要 P2 的 `asset_version`，否则"改工具图" = 改所有引用它的地方，没有版本边界。

### 0.2 格式定案 v1（2026-10-19，一页看完）

| 层 | 格式 | 谁写 / 谁读 | 说明 |
|---|---|---|---|
| **源码** | `.py`（节点类 + `param_labels.py`） | 引擎 `import` | 永远明文；diff / AI 改写 / 热重载都靠它 |
| **作者源**（唯一可编辑的节点文本） | **`nodes/*.json`** | 引擎（stdlib `json`） | 一文件一库、可装多个节点；单节点简写；整文件原子 |
| **库文件**（加载/分发形态） | **`*.sqlite`**（明文库） | 引擎 stdlib **`sqlite3`** | `meta` + `nodes(id,version,category,label,json,sha256)` + `assets(path,bytes)`；可 `UPDATE` 单行、可按 `category` 查；给同事/CI 传一整个库 |
| **外发分发** | **`.nlib`**（同一份 **SQLite 字节**加密 + 明文头 + 签名；`payload:"sqlite"`） | `l_nodelib` 解密 → `deserialize` | AES-256-GCM；key 走 `license.json`（L2）/ 指纹（L3）；**v1 只做 sqlite** |
| **不采用** | YAML 进引擎 / 自造"高效二进制" / 加密 `.py` / HMAC | — | YAML 只做转换器输入；`.hda` 式二进制定义格式明确不学 |

**加密依赖落点（定案）**：AES-GCM 需要第三方库（`cryptography`，仓里 `l_agent_chat` / `l_nginx` 等已在用），
而 `l_node_engine` 明写**只用标准库**（`package.py:8`）⇒ **不在引擎里加依赖**，改成：

- 新小包 **`l_nodelib`**（`requires = [python, cryptography, l_node_engine]`）：打包 / 查看 / 解密三件事都在它这；
- 引擎只做"**发现 `.nlib` 就懒加载 `l_nodelib`**"，没装就报"要打开加密库请装 `l_nodelib`"（**明文路径不受影响**，
  远端引擎宿主也不必为它装 crypto）；
- 要用加密库的宿主（如 `l_mindmap_mmd`）把 `l_nodelib` 写进自己的 `requires`。

### 0.3 v1 具体方案（⚠ 文件形态/加密粒度/验签顺序已被 §0.4 v2 提案取代；确认前仍以此为准，确认后以 §0.4 为准）

**A. 三种文件、三条入口，归一后只有一条注册路径**

| 文件 | 谁产生 | 引擎怎么读 |
|---|---|---|
| `<包>/nodes/*.json` 或 `<包数据目录>/nodes/*.json` | 人 / AI 写 | stdlib `json`（单节点简写也认） |
| `<包>/nodes/*.sqlite` | `nodelib_pack` | stdlib `sqlite3`（`?mode=ro&immutable=1`） |
| `*.nlib`（任意目录，按配置扫） | `nodelib_pack --encrypt` | `l_nodelib`：读明文头 → 找 key → 解密 → `deserialize` |

发现顺序：包内按**文件名**排序 → 再叠数据目录（**本地赢**）；同层同名 → 报错并点名两个文件。
插件开关沿用 `plugins.json`（给每个库一个 `enabled` + 面板「🔌 插件」重载）。

**B. SQLite schema（定稿，改结构就升 `meta.format` 版本）**

```sql
PRAGMA journal_mode = DELETE;          -- 发布库不要 WAL（单文件、可只读打开）
PRAGMA user_version = 1;               -- = meta.format 的数字版

CREATE TABLE meta(key TEXT PRIMARY KEY, value TEXT NOT NULL);
--   format "lugwit-nodelib/1" / library / version / author / copyright / license
--   keyslot / signed_by / signature / signed_at / engine_min / created

CREATE TABLE nodes(
  id TEXT NOT NULL, version INTEGER NOT NULL DEFAULT 1, category TEXT, label TEXT,
  json TEXT NOT NULL, sha256 TEXT NOT NULL, PRIMARY KEY(id, version));

CREATE TABLE assets(path TEXT PRIMARY KEY, bytes BLOB NOT NULL, sha256 TEXT NOT NULL);
```

**C. 工具（都只有一个用途，不做万能入口）**

```bat
:: 1) 打包：json 目录 → 明文库 → 可选加密发布包
nodelib_pack --src nodes\ --library tex_extra --out tex_extra.sqlite [--no-plain]
nodelib_pack --src nodes\ --library tex_extra --out tex_extra.nlib --encrypt --key-file license.json

:: 2) 查看（只读，不注册节点）
nodelib_inspect tex_extra.nlib              :: 明文头：库名/作者/许可/alg/keyslot/签名状态
nodelib_inspect tex_extra.nlib --with-key   :: 解密后列节点（id/label/中文参数/类型/默认值）
nodelib_inspect tex_extra.nlib --verify     :: 重算 sha256 + 验签 → verified / modified
nodelib_inspect a.nlib --diff b.nlib        :: 两版库的节点/参数差异
nodelib_inspect x.nlib --assets-dir out\    :: 导出图标/示例图（不覆盖已有文件）

:: 3) 查验/自证（引擎侧）
wuwor l_node_engine -- l_node_engine_selftest
```

`nodelib_pack` 内部顺序：读 json → 逐条校验（`id`/`exec` 格式/类型词表/端口）→ 单事务写库 →
逐行算 `sha256` → 写 `meta`（作者/许可）→ 签名（Ed25519，公钥进 `<包>/trusted_keys/`）→ 可选 AES-GCM 加密。

**D. 加载与校验（引擎侧，顺序定死）**

1. 读文件 → `format` / `user_version` 不认 → 整库拒，报"这个库要更新的引擎"；
2. 逐行校验 `nodes.sha256`；有签名则验签 → 置 `verified` / `modified` / `unsigned`；
3. `nodes_require_signed=1` 且非 `verified` → 整库不注册（说清缺谁的公钥）；
4. 逐行 `json` → 节点声明 → 结构校验（静态端口、`exec.kind` ∈ {callable, pygraph}）→ `register_node(NODE_SOURCE="file")`；
5. 任一行失败 → **整库不进**，错误列表「文件 : id : 原因」交给面板/日志；
6. `category` 用于节点库分组；`label` 用于显示（缺省回落 `id`）。

**E. 配置（服务端设置，落在 `plugins.json` / `settings` 同层）**

| 键 | 默认 | 作用 |
|---|---|---|
| `nodes_allow_file` | `1` | 允许文件节点（企业可整体关） |
| `nodes_require_signed` | `0` | 只收验签通过的库 |
| `nodes_trust_keys_dir` | `<l_node_engine>/trusted_keys` | 信任公钥目录 |
| `nodes_lib_dirs` | `[]` | 额外的库目录（默认只扫包内 + 数据目录） |

**F. 待拍板的 6 条 —— 建议默认值（不反对就按这个开工）**

| # | 项 | 建议默认 |
|---|---|---|
| 1 | P0 参数模板字段 | `min` / `max` / `step` / `unit` / `group` / `order`（直接进 `opts`） |
| 2 | P1 DSL 边界 | 等值 / 属于集合 / `any`·`all` 组合；**先不做数值比较** |
| 3 | P2 版本粒度 | 挂**节点**（`asset_version`，`MyBlur#2`）；包版本只当发布单位 |
| 4 | P4 落点 | 包内 `nodes/` + 数据目录 `nodes/` **两处都扫**（本地赢） |
| 5 | 命名空间 | `id` 全局平表（不做 `library:id` 前缀） |
| 6 | 库内多版本 | `nodes(id, version)` 允许并列；图里用 `asset_version` 选 |

**G. 里程碑（谁先做、验收看什么）**

| 阶段 | 产出 | 验收 |
|---|---|---|
| P4a | schema + 读取 + `nodelib_pack`（json→sqlite）+ `NODE_SOURCE="file"` | selftest：坏行整库拒 / 同名冲突 / 一库 3 节点全注册；探针：节点库出现 → 落卡 → 真跑 |
| P6 | 逐行 sha256 + `meta` 签名 + 三档状态 + `nodes_require_signed` | selftest：验签通过 / 改一行 → `modified` 可注册（除非开关）/ 缺公钥 → `unsigned` |
| P4c | `l_nodelib` + `.nlib` + `nodelib_inspect` | selftest 六条（见 P4c）+ 查看器无副作用 |
| 之后 | P0 / P1 / P5 / P2 / P3 / P4b | 各按 §六 判据 |




### 0.4 格式优化提案 v2（2026-10-19 用户：「关于文件格式再优化一下，给出方案」· **待确认**，确认后取代 §0.3 的 A/B/C 与 P4c/P4d 里与之冲突处）

**一句话**：把"明文 `.sqlite` + 加密 `.nlib`（整文件包一层）"两种文件合成**一种** —— **单一 `.nlib`，本身就是合法 SQLite 库，
加密发生在行内（只加密 `body` 与资源 BLOB）**，目录列、元数据、签名永远明文。

#### 0.4.1 要解决 v1 方案的 6 个毛病

| # | v1（整文件加密）的毛病 | v2 怎么解 |
|---|---|---|
| 1 | **两种文件**（`.sqlite` / `.nlib`）、**三套入口**（json / sqlite / nlib），工具与文档要讲三遍 | **只有一种库文件 `.nlib`**；`meta.encrypted = 0 \| 1` 决定 body 是明文还是密文；作者源仍是 json 目录 |
| 2 | 没 key 就**看不到任何东西**（查看器只剩明文头），节点库面板没法显示"这库有哪些节点（需解锁）" | 目录列（`id/version/category/label/search`）永远明文 ⇒ **无 key 也能列清单**、也能搜；解密只对"真要用的那几个" |
| 3 | 验签必须**先解密**（先签后加密），查看器 `--verify` 也要 key | 改成**先加密后签**：签的是**存储字节**（密文）的 sha256 ⇒ **无 key 即可验签** |
| 4 | 整份 SQLite 字节解密进内存（`deserialize`），库一大就整份占内存；且不能只读打开 | 磁盘上就是 SQLite：`file:…?mode=ro&immutable=1` 直读，**按行**解密、可 LRU、可懒注册 |
| 5 | 想做"**免费节点 + 付费节点同库**"做不到（整库一把锁） | 行级 `enc` 标志：同一个库里有的节点明文、有的密文 |
| 6 | 自造了一层容器（magic + 头 + nonce + payload），还要自己写读头工具 | 用 SQLite **自带的识别位**：`PRAGMA application_id = 0x4C4E4C42 ('LNLB')`（文件头偏移 68 的 4 字节，`file`/十六进制一眼可认）+ `user_version`；**不再有自造 magic** |

#### 0.4.2 文件形态（定稿草案）

```
作者源（人/AI 写，进 git）              库文件（产物，二进制）
tex_extra/                              tex_extra.nlib   ← 唯一的库文件，合法 SQLite
├─ library.json    # meta                  ├─ meta      (明文)
├─ nodes/*.json    # 一文件一节点或数组     ├─ nodes     (目录列明文 + body 明文或密文)
└─ assets/…        # 图标/示例图           ├─ blobs     (按 sha256 去重的资源，明文或密文)
                                          └─ assets    (path → blob)
        ──nodelib_pack──►  tex_extra.nlib  [--encrypt]
```

- **作者源拆成"一文件一节点"**：`nodes/MyBlur.json`（`git diff` 看得清、AI 改一个节点不碰别的、不会合并冲突）；
  数组形式（一文件多节点）仍认，作为 §0.2 已定的"单文件多节点"；
- 扩展名只有 **`.nlib`**；是否加密看 `meta.encrypted`，**不再用扩展名区分**（旧计划里的 `.sqlite` 取消）。

#### 0.4.3 SQLite schema（v2）

```sql
PRAGMA application_id = 0x4C4E4C42;   -- 'LNLB'
PRAGMA user_version   = 2;            -- 格式版本（迁移靠它）
PRAGMA journal_mode   = DELETE;       -- 发布库不用 WAL（单文件、可只读打开）
PRAGMA page_size      = 4096;         -- 固定，保证同输入同字节（可复现构建）

CREATE TABLE meta(key TEXT PRIMARY KEY, value TEXT NOT NULL);
--  明文：format / library / version / author / copyright / license / engine_min
--        encrypted(0|1) / keyslot / kdf_salt(hex) / requires(JSON 数组，exec 目标所在的包或模块)
--        signed_by / sig_alg / signature / signed_at / created

CREATE TABLE nodes(
  id        TEXT    NOT NULL,
  version   INTEGER NOT NULL DEFAULT 1,
  category  TEXT,                      -- 明文：节点库分组
  label     TEXT,                      -- 明文：显示名
  search    TEXT,                      -- 明文：搜索别名/关键词（SEARCH_ALIASES）
  enc       INTEGER NOT NULL DEFAULT 0,-- 0 明文 / 1 密文（行级）
  body      BLOB    NOT NULL,          -- enc=0：UTF-8 JSON；enc=1：nonce(12) || AES-GCM 密文+tag(16)
  sha256    TEXT    NOT NULL,          -- **存储字节**的 sha256（明文行=明文；密文行=密文）→ 无 key 可验
  PRIMARY KEY(id, version)
) WITHOUT ROWID;

CREATE TABLE blobs (sha256 TEXT PRIMARY KEY, enc INTEGER NOT NULL DEFAULT 0, bytes BLOB NOT NULL) WITHOUT ROWID;
CREATE TABLE assets(path TEXT PRIMARY KEY, sha256 TEXT NOT NULL REFERENCES blobs(sha256)) WITHOUT ROWID;
```

要点：

- `WITHOUT ROWID` + 复合主键：目录查询走主键、少一层间接，库小也省空间；
- `blobs`/`assets` 拆开 = **内容寻址去重**（同一张图标被 10 个节点引用只存一份）；
- `body` 明文用 **规范 JSON**（`sort_keys` + `separators=(",",":")` + UTF-8）⇒ 同输入同字节 ⇒ 可复现 + 签名稳定
  （这是**打包时**规范化一次；作者源里 JSON 想怎么缩进都行，源码文本不进签名）；
- 迁移：`user_version` 往上走，`nodelib_pack --upgrade` 做 `ALTER`/重建；加载器读到更高版本 → **整库拒收并说"要更新引擎"**。

#### 0.4.4 加密 / 签名 / 版权（行级方案）

| 项 | 做法 |
|---|---|
| 算法 | AES-256-GCM（`cryptography`，只在 `l_nodelib`，引擎保持纯标准库） |
| 密钥派生 | `data_key = HKDF-SHA256(master, salt=meta.kdf_salt, info=b"lnlib/rows/1")`；`master` 来自 `keyslot`（L2 `license.json` / L3 +机器指纹 / L4 联网） |
| nonce | 每行 / 每个 blob **随机 12 字节**，前缀存在密文里 |
| **AAD（防换行）** | `f"{library}\|{id}\|{version}\|body"`（blob 用 `sha256`）⇒ 把 A 行的密文塞进 B 行，**GCM 校验失败** |
| 签名 | Ed25519 签「`meta` 中**签名白名单字段**的排序串 + 所有 `nodes`/`blobs` 的 `(kind,id,version,sha256)` 排序串」；因为 `sha256` 是**存储字节**的，**无 key 即可验签** |
| 版权 | `meta.author/copyright/license` **明文**并进签名；再**每个节点 `body` 的 JSON 里也冗余一份 `author/license`**（加密后攻击者抠单行出去仍带署名） |
| 三档 | `verified`（签名 + 全部 `sha256` 对）/ `modified` / `unsigned`；`nodes_require_signed=1` 才拒载 |
| 混合库 | 行级 `enc`：免费节点明文、付费节点密文并存；库级 `keyslot` 先只做**一把**（行级多 key 作为 v2.1，别急） |

**这版和 §0.3 的差别（诚实说明）**：之前我把"应用层字段加密"评为"可读可查全废 → 不采"；现在结论改了 ——
**只加密 `body` 与资源，目录列留明文**，反而是**更好的折中**：清单可见、可搜、可验签、可混合锁；
损失只有一条：**节点 id / 显示名 / 分类对外可见**（和 HDA 的"类型名可见"一致，不是机密）；真要连目录都隐藏，
作为 **可选的"封印模式"**（`--seal`，仍整文件 AES-GCM 外包一层，回到 v1 的 `.nlib` 形态）留到有人要时再做，**v2 不做**。

#### 0.4.5 读取路径与性能（原型实测）

加载器：`sqlite3.connect("file:…?mode=ro&immutable=1", uri=True)` →
`SELECT id,version,category,label,search FROM nodes`（目录，无 key）→ 需要时按行 `SELECT body` →
`enc=1` 就 `AESGCM.decrypt(nonce, ct, aad)` → `json.loads` → 注册。

**原型实测**（2026-10-19，本机 py3.12 + `cryptography`，300 个节点，每节点 8 个参数）：

| 指标 | 结果 |
|---|---|
| 库体积 | **268 KB**（密文行，未压缩） |
| 构建（建库 + 全部加密 + VACUUM） | **74 ms** |
| **无 key 读目录**（300 行） | **1.1 ms** |
| **全部解密 + `json.loads`**（300 行） | **4.5 ms** |
| 文件头 `application_id` | `0x4C4E4C42`（偏移 68，确认可直接识别） |
| 签名（Ed25519）验签 | 通过 |
| **换行攻击**（用 `Node001` 的密文冒充 `Node000`） | **被拒**（AAD 不符） |
| **错 key** | **被拒** |

⇒ 效率上**不需要任何额外优化**（整库 5ms 级）；该花力气的仍是 P4c「效率评估」里说的 `import` 与 `INPUT_TYPES()`。

#### 0.4.6 工具与命令（更新）

```bat
:: 打包（作者源 → 库）。默认明文；加 --encrypt 则 body/blobs 行内加密。全程不落中间明文文件。
nodelib_pack  --src tex_extra\ --out tex_extra.nlib [--encrypt --key-file license.json] [--sign key.pem]

:: 查看（只读、不注册节点；无 key 也能用）
nodelib_inspect x.nlib                  :: 目录：库信息 + 节点清单（id/version/category/label）+ 签名状态
nodelib_inspect x.nlib --with-key       :: 解密 body：列参数/类型/默认值/exec 目标（只显示不导入）
nodelib_inspect x.nlib --verify         :: 无 key 验签：逐行 sha256 + Ed25519 → verified/modified
nodelib_inspect a.nlib --diff b.nlib    :: 目录级差异（无 key 即可）；加 --with-key 才比参数
nodelib_inspect x.nlib --sql "SELECT …" :: 只读 SQL（仅 SELECT；密文列显示 <enc:N bytes>）

:: 改一个节点（开发期）
nodelib_pack --patch x.nlib --node nodes\MyBlur.json    :: UPDATE 单行并重算 sha256/签名（需私钥）
```

可复现构建：`--source-date-epoch` 固定 `created`；`page_size` 固定；`VACUUM`；规范 JSON ⇒ **同输入同输出字节**，CI 里可对拍。

#### 0.4.7 兼容与迁移（相对已写的 v1 计划）

| 项 | v1（已写） | v2（本提案） | 动作 |
|---|---|---|---|
| 库文件 | `*.sqlite`（明文）+ `*.nlib`（整文件加密） | **只有 `*.nlib`** | 删掉 `.sqlite` 作为分发形态；明文 `.nlib` 就是明文库 |
| 加密粒度 | 整文件 | **行级**（`body` / `blobs`）+ 可选 `--seal`（未来） | 重写 P4c |
| 验签 | 解密后 | **无 key 可验**（先加密后签） | 重写 P6 的验签顺序 |
| 目录可见性 | 加密库不可见 | 目录列明文 | 查看器有"无 key 也列清单" |
| 资源 | `assets(path, bytes)` | `blobs` 去重 + `assets` 引用 | 改 schema |
| 作者源 | `nodes/*.json`（可多节点数组） | **一文件一节点为推荐**，数组仍认 | 工具/文档同步 |
| 容器 magic | 自造 `LNLIB\x01` | **`application_id` + `user_version`** | 取消自造 magic |

#### 0.4.8 风险与未决（需要你点头的）

1. **目录明文可接受吗？**（`id/label/category/search` 对外可见。要完全隐藏就得走 `--seal`，代价是回到整文件包装、失去"无 key 清单/验签/混合锁"。）
2. **混合库要不要进 v1？**（行级 `enc` 已天然支持，只是 UI 上要显示"🔒 需许可证"。）
3. **`search` 列放哪些词？**（建议只放 `SEARCH_ALIASES` 与中文别名；别放参数名。）
4. **私钥与公钥分发**：`trusted_keys/` 随引擎包走；私钥只放发布机，不进仓库（P6 已写）。
5. 与 **P2 节点版本**的关系：`nodes(id, version)` 主键不变；目录里 `latest` 由 `MAX(version)` 算，图里 `asset_version` 照旧。

**建议**：采纳 v2。理由一句话：**单一文件、无 key 可看可验、行级可混合、同一份 SQLite 工具链全能用，而且原型实测整库 5 ms 级**；
代价只是目录列对外可见，且这个损失与 HDA 的"类型名可见"同量级。

### 0.5 实施进度（P4a 最小闭环**已落地**，2026-10-19，AIVideo 试跑）

| 件 | 位置 | 状态 |
|---|---|---|
| 引擎读/写库 | `l_node_engine/src/l_node_engine/nodelib.py` | ✅ 纯标准库（`sqlite3`/`json`/`hashlib`）：`pack` / `read_catalog` / `load_library` / `unload_library` / `node_versions` / `get_versioned` / `discover` |
| schema | 同上 `_SCHEMA_SQL` | ✅ 按 §0.4.3：`application_id=0x4C4E4C42`、`user_version=2`、`meta` + `nodes(id,version,category,label,search,enc,body,sha256)` + `blobs` + `assets`；`page_size=4096`、`journal_mode=DELETE`、打包 `VACUUM` |
| 注册 | 生成 `_FileNode` 子类 → `register_node(..., NODE_SOURCE="file")`；`get_node_info` 多带 `versions/latest/library` | ✅ 只注册 `latest`；旧版在 `nodelib.VERSIONS` 里按需取 |
| 原子性 | `load_library()` 先建完全部类再动注册表 | ✅ 一行坏 = 整库拒（exec 目标缺失 / sha256 不符 / 密文行 → 都点名） |
| 工具 | `l_node_engine/tools/nodelib_cli.py` + `package.py` 三个别名 | ✅ `l_node_engine_nodelib_pack` / `_inspect` / `_export` |
| 样例（AIVideo） | `l_node_engine/nodelib_samples/ai_video/`（作者源）+ `ai_video.nlib`（产物） | ✅ `nodes/AIVideoFile.json` 由 `nodelib_export AIVideo --id AIVideoFile` 生成（19 入参、中文 label、`exec.kind=class → l_node_engine.ai_nodes:AIVideo`） |
| 宿主接线 | `l_mindmap_mmd/server._node_engine_impl` | ✅ 启动时 `l_node_engine.nodelib.discover()`（扫 `L_NODE_ENGINE_NODELIBS` + `<包数据目录>/nodes`），坏库只记一条 |
| **签名 / 验签（P6 v1）** | `nodelib.verify` / `pack(sign_key=…, sign_key_name=…)` / CLI `inspect --verify` | ✅ **两种算法**：`ed25519`（推荐，私钥留发布机、公钥随包，要 `cryptography`——2026-10-19 用户口径「可以按需添加依赖」已加进 `requires`）与 `hmac-sha256`（纯标准库，team 内用）；三档 `verified` / `modified` / `unsigned`；密钥走 `L_NODE_ENGINE_SIGN_KEY=<名>:<hex>` 或 `<包>/trusted_keys/<名>.key`；`L_NODE_ENGINE_REQUIRE_SIGNED=1` 时才拒载。`meta.sig_alg` 决定用哪套 |
| **节点版本（执行 + UI，P2）** | `nodelib.resolve_class` + `graph._execute_node` + `validate` + `PyNode` 徽标下拉 + `PyGraphEditor.infoFor` + `pySchema` 的 `asset_version` 白名单（`toStorage` / `normalizeGraph` / `toPrompt`） | ✅ 图里 `asset_version` 按版本取类、未知版本报错不回落；卡片头**版本下拉**（非最新高亮）、切换走 `mutate({step:true})` ⇒ Ctrl+Z；**每版端口/控件各自重算**（`/api/pygraphs/node_info?name=&version=`）；剩：增量指纹含版本 |
| **行级加密（P4c 核心）** | `nodelib.pack(encrypt=True, keyslot=…, lib_key=…)` + `_read_rows` 解密 | ✅ AES-256-GCM（`cryptography`）；只加密 `body`/blob，**目录列仍明文**（无 key 也能列清单）；AAD 绑 `库\\|id\\|版本`（换行即失败）；key 走 `L_NODE_ENGINE_LIB_KEY` 或 `<包数据目录>/license.json`；**`l_nodelib` 这个新包不再必要** |
| 自证 | `l_node_engine_selftest` 第 39 组 | ✅ 可复现打包 / 无 key 读目录 / 与内置 `AIVideo` 入参面完全一致（19）/ 借执行体真跑（桩掉网络）/ 多版本只挂 latest + 未知版本报错 / 坏库整库拒 / 冲突与卸载 |

**真机证据**（2026-10-19 23:35，`:8110` 重载后查 `/api/pygraphs/object_info`）：

```
has_node: True
inputs: 19 versions: [1] latest: 1 source: file display: AIVideoFile library: ai_video_sample
category: l_node_engine/ai outputs: ['VIDEO']
same_ports_as_builtin: True
```

样例已按"用户安装"的方式放进 `<包数据目录>/nodes/ai_video.nlib`
（`C:\Users\<用户>\.lugwit\main\l_mindmap_mmd\nodes\`），
所以编辑器节点库里今天就能看到 `AIVideoFile`（分类 `l_node_engine/ai`）。

**属性面板「定义」行**（2026-10-19 用户口径）：文件节点显示 `.nlib` 路径 + `exec` 实现（`kind → target`），
手写节点显示 `源码文件:行号`；`NodeInspector`（查看全部字段）与右侧属性面板两处都加了。

### 0.6 完成度总表（2026-10-20 核对；**不是全实现了**）

用户 2026-10-20 问「全部实现了么」→ **没有**。逐项核对结果如下（"证据"列能直接点开核）：

| 项 | 状态 | 证据 / 缺什么 |
|---|---|---|
| **P4 库文件**（`.nlib` = SQLite、一库多节点、整库原子、冲突规则） | ✅ | `nodelib.py`；`selftest` 第 39 组 |
| **P4c 行级加密**（AES-256-GCM，目录列明文） | ✅ | `pack(encrypt=…)` + `_read_rows`；CLI 实测 `加密：是（keyslot=demo）` |
| **P4c 查看器（头 / 目录）** | ✅ | `nodelib_inspect`（无 key 也能列清单、`--verify`） |
| **P6 签名**（Ed25519 / HMAC 两档、三档信任、`REQUIRE_SIGNED`） | ✅ | `selftest` 第 39 组 ⑧⑨；`trusted_keys/README.md` |
| **P2 节点版本**（库内多版本、执行/校验按版本、卡片下拉切换、按版本端口） | ✅ | `resolve_class` / `validate` / `node_info` 端点 / `PyNode` 下拉；浏览器实测 v1→20 行、v2→18 行 |
| **属性面板「定义」行** | ✅ | `NodeInspector` + 右侧属性面板 |
| **P0 的 `min/max/step`** | ✅（前端早就会用） | `PyWidgetInput` 夹取 + 传 `step/min/max` |
| **P0 的 `unit / group / order`** | ✅ | 卡片：`group` 归堆 + 组名小标题 + 组内 `order` 排序、`unit` 跟类型后显示；属性面板：`order` 排序 + `unit` 显示（**两处都有组标题**）；浏览器实测 `自由 / ##形状(半径,模式) / ##输出(甲,乙)`、`半径 FLOAT · px` |
| **全部节点的帮助**（动态生成，2026-10-20） | ✅ | 不写 150+ 篇静态文档：`GET /api/pygraphs/help/node/<注册名或类名>` 按 `get_node_info()` **现生成**一页（说明 / 信息（分类·来源·定义文件:行号·exec·别名）/ 参数表（中文 label·端口键·类型·控件·默认·tooltip）/ 输出）；`GET /api/pygraphs/help/nodes` 是**全部节点目录**（按分类分组 + 搜索框，点进去即上面那页）；`HELP` 缺省也发 `help: "node/<类名>"` ⇒ 编辑器「❓ 节点帮助」对**每个节点**都有「📖 打开帮助」；实测：目录 **247 个节点 / 32 个分类**（含插件节点）、`AIVideo` 9.9KB / `TexBlur` 4.7KB / `2d_view`（按类名 `View2D` 解析）4.3KB；⚠ `helpDocUrl` 要**逐段编码**（`node/AIVideo` 不能把 `/` 编成 `%2F`，否则落到 `/help/{name}` 404） |
| **「❓ 节点帮助」标签页**（2026-10-20） | ✅ | `pyLayout` 新标签 `nodehelp`（home=right，默认右栏排在「属性」后）；组件 `PyNodeHelp.jsx`：选中节点 → 名称/class_type/版本徽标 + 说明 + **信息**（分类/来源/定义文件与行号/`exec`/搜索别名）+ **参数表**（中文 label·端口键 / 类型 / 控件 / 默认值 / tooltip，可选行标 `?`）+ **输出端口**；顶部两个按钮：`📖 打开帮助`（节点声明了 `help` 时）/`📚 帮助中心`；节点元数据新增 `help` 字段（类写 `HELP="<名字>"`，库声明写 `"help": "<名字>"`，`nodelib_export` 会带出）；真机实测：tab 在、选 `AIVideo` → 参数 19 行 + 定义路径 + 两个按钮（`📖 打开帮助` / `📚 帮助中心`） |
| **帮助页用真实画布组件**（2026-10-20） | ✅ | 新页面 `/uihelp`（服端复用 `flow.html` 壳，`main.jsx` 按 pathname 分流到 `help/HelpDemo.jsx`）：**直接 import 画布的 `PyNode`/`PyNote`**，同一个 `@xyflow/react` + 同一份 `style.css` + 引擎真元数据（只换值），因此与画布**像素级同源**；演示 AIVideo 表单卡 / 普通卡 / 注释卡三张，控件真能改（`onWidgetChange` 回显）；帮助首页与两篇文档都加了入口；**文档里可内嵌真组件**：`/uihelp?only=form|normal|note&embed=1`（`embed=1` 去掉标题/页脚/导航，只留卡片）；⚠ 三个坑：① `flow.html` 的 `<base>` 归一化正则要带上 `uihelp`（漏了 = 资源 404）；② 帮助页里的链接/iframe 必须写**相对路径**，并由服务端注入 `<base href="<前缀>/">`（写死 `/uihelp` 会被当站点根、绕过反篇前缀）；③ `embed=1` 时不画 `NavMenu`（否则盖在卡片上） |
| **帮助中心 + 注释指向帮助**（2026-10-20） | ✅ | 文档在包里 `l_mindmap_mmd/help/*.html`（**可交互 + 动画**，改文件即更新）；端点 `GET /api/pygraphs/help`（首页）/ `/help/list`（清单）/ `/help/<名字>`（固定网址）；注释节点写一行 `help:<名字>` 就直接渲染那篇帮助（卡片 iframe / 🪟 同一固定网址）；编辑器顶栏加 **❓ 帮助**；实测：首页 3977B、`node_library` 10683B、`note_window&node=5` 返回同一篇帮助；帮助页点「版本」页签 + v2 → 行 5→7、主题切换生效 |
| **HTML 注释 + 独立窗口**（案例演示） | ✅ | `Note.text` 以标签开头 → **sandbox iframe**（`allow-scripts`，不给 same-origin）渲染；**主题跟卡片底色走**（纸白→亮、其余→暗；注入 `color-scheme` + `--note-*` 变量，未写颜色的 HTML 自动跟随）；标题栏新增 **🪟 独立窗口查看**，且**同一注释节点网址固定**：`/api/pygraphs/note_window?graph=<图>&node=<id>&theme=<light|dark>`（`window.open(url, "note-<图>-<节点>")` 复用同一标签页；拿不到图地址才退 Blob）；演示图 `<工程>/demo/demo_nodelib_features.json`（**图文并茂**：暗/亮两张 HTML 注释用**内联 SVG + HTML/CSS 示意图**讲解 —— 含「🖥 在网页里大概长这样」的**浏览器窗口 / 画布节点卡 / 属性面板**示意；HTML 注释卡自动加宽到 560px；🪟 用 **Blob 单标签**打开，不再多留 about:blank）；浏览器实测 2 注释 / 1 iframe / 2 个 🪟、点击后 `window.open` 收到 2926 字符文档（含演示脚本） |
| **内置 `AIVideo` 退休（由节点库提供）** | ✅ | `ai_nodes.NODE_CLASS_MAPPINGS` 去掉 `AIVideo`（**类保留** = 执行体/导出源）；样例节点改名 `AIVideoFile → AIVideo`，发布到 `<工程>/nodes/ai_video.nlib`；真机 `source=file · library=ai_video_sample · versions=[1] · 19 入参 · exec class → l_node_engine.ai_nodes:AIVideo`，`AIVideoFile` 已不存在；`demo_ai_video` 照常渲染（AIVideo 卡 20 行、零警告）；⚠ **删了节点库就没有 AIVideo 节点**（预期） |
| **工程级节点库**（随工程走） | ✅ | 宿主扫 `<工程根>/nodes/`（`registered_projects`）与包数据目录**并列**；样例已发布到 `lugwit_agent_a/nodes/ai_video.nlib`（作者源仍在 `l_node_engine/nodelib_samples`），用户目录那份已删；真机 `AIVideoFile=True` 
| **批量切版本** | ✅（多选未逐一视觉自证） | 右键节点「🏷 版本 ▸（N 个同类节点）」：选中的同类节点一起切 / 跟随最新；整批一次 `mutate({step:true})` ⇒ 一次 Ctrl+Z；单节点实测切 v1 后 `select=1 / is-pinned` 
| **卡片页签栏** | ✅ | ≤3 页画可点页签、>3 页退下拉（`pyg-card-tabs`）；浏览器实测 `基本/高级/默认` 切换 → 行 `备注` ↔ `种子,步数` |
| **参数页签**（`opts.tab`） | ✅ | 面板：`opts.tab` 分页 + 可点页签栏（换节点回到第一页）；卡片：可点页签栏（≤3 页）/ 下拉（>3 页），见下面「卡片页签栏」；浏览器实测 `基本 / 高级 / 默认` 三页切换、卡片 `##基本 / ##高级`；⚠ 页签清单必须算在 `selectedPorts` **之后**（写在前面 = TDZ 白屏） |
| **P1 条件显隐**（`visible_when` / `disabled_when`） | ✅ | 引擎 `conditions.py`（DSL：等值/集合/all-any；**fail-open**）+ 前端 `pyConditions.js` 镜像；卡片行、表单卡、属性面板都过滤；`disabled_when` 在面板置灰不隐藏；selftest 第 40 组；浏览器实测 `mode` 切 a→b、`flag` 勾上后按 `all` 规则逐级显隐 |
| **P3 节点钩子**（`on_create` / `on_param_change` / `on_loaded`） | ✅ | 引擎 `hooks.py`（只读、白名单、异常降级、返回值只认 opts/ui/warn）+ `get_node_info` 发 `hooks` 摘要（**不带 call**）+ 宿主 `POST /api/pygraphs/node_hook`（本机写接口要 Bearer）+ 前端「改参数→问钩子→补控件 opts」；selftest 第 41 组；`.nlib`/作者源声明里也能写 `hooks`（打包时校验钩子名与 `call`）；selftest ⑤e |
| **P4b `exec.kind="pygraph"`** | ✅ | `EXEC_KINDS` 加 `pygraph`；`exec.graph` = 资产地址（`pro/…` / 工程名/… / 绝对路径）→ 宿主注入 `set_graph_resolver`（`_pygraph_path` 那套）；**加载时解析**（地址坏了整库拒）+ 执行走 `run_pygraph`（实参按图入参名注入、输出按声明顺序取）；selftest ⑤d；真机实测 `pg_demo.nlib` 里 `PgDemo` 注册成功（解析器**必须在 discover 之前**注册，晚注册=整库被拒） |
| **直接加载作者源目录**（`library.json` + `nodes/*.json`） | ✅ | `nodelib.load_source_dir()`（与 `.nlib` **同一份** normalize/build/register）+ `discover()` 同时扫 `*.nlib` 与含 `library.json` 的目录；开发期改 json 刷新即生效；作者源**未签名** ⇒ `REQUIRE_SIGNED=1` 时拒收；selftest ⑪ + 真机 `DevDemo=True`（`<包数据目录>/nodes/devlib/`）；⚠ 别把 `.nlib` 放进作者源目录（同 id 会被扫两次） |
| **增量指纹含版本**（P2 尾巴） | ✅ | `graph._node_fingerprint` 带 `@vN`；selftest ⑤c 断言同结构不同版本指纹不同 |
| **查看器扩展** | ✅ | `--diff`（目录级差异，无 key 可跑）/ `--assets-dir`（导出资源、密文用库 key 解、不覆盖、挡越界）/ `--sql`（**只读**：单条 SELECT/WITH，密文列显示 `<N bytes>`）/ `--with-key`（**密文行用库 key 解**，错 key 明确「解密失败」）；四条都实测 |
| **P5 参数提升**（一键 params + `GetParam` + dock 胶囊） | ✅ | 属性面板「提升到接口」下拉（`promoteParam`）：补 `graph.params` 一行 + 起 `GetParam` + `S.attachEndpointChip`，`mutate({step:true})` ⇒ Ctrl+Z；浏览器实测 chip 0→1、接口签名多出 `seconds: number = 5`、Ctrl+Z 退回 |
| **§0.4 的 `--seal`**（整文件封印，可选） | ❌（本就不进 v1） | — |
| **§八 4 条待拍板**（P0 字段名 / P1 DSL 边界 / P4 落点 / 命名空间） | 仍开着 | 落点事实上已按"包内 + 数据目录都扫"实现 |

**一句话**：**"库 + 加密 + 签名 + 版本 + 参数提升"已通且自证；**HDA 对照表全部落地**（§0.6 已无 ❌ 项）。。**



## 一、HDA 到底是什么（六件，2026-10-19 口述核对）

| # | HDA 的东西 | 语义 |
|---|---|---|
| 1 | Type Properties · **Parameters** 页 | 资产面板上的参数表：`name` / `label` / `type` / `default` / `range` / `menu` / `disable when` / 分组与页签 |
| 2 | Type Properties · **Inputs/Outputs** 页 | 对外端口（名字/类型/必填/默认），与内部某端口绑定 |
| 3 | **内部网络** | 资产展开后的子网 —— 资产的"实现" |
| 4 | **Python Module / 事件钩子** | OnCreated / OnInputChanged / OnLoaded 等，资产自己带代码 |
| 5 | **Definition 与版本** | 同名资产可多版本共存（definition 库），实例按版本解析 |
| 6 | **参数提升（promote）** | 把内层节点参数提升到资产面板（等价"对外接口"） |

## 二、对照：HDA ↔ 我们

| HDA | 我们已有的（证据） | 差的 |
|---|---|---|
| Inputs / Outputs | 图级 `graph["params"]`（`l_node_engine/graph.py`，`/pygraph` 的「⇄ 接口」可编辑）；`Output` 节点；工具参数由图端口现算（`server.py` 的 `ports` 端点） | 缺"接口页签"式集中视图（现在散在图级 params + 图里 `GetParam` 胶囊） |
| Parameters | `param_labels.py`（**中文 label**，2026-10-19 新加）+ `param_spec.py`（`name`/`type`/`default`/`optional`）+ 控件容器词表（`ToolSpec` §5.4） | **范围/步进/单位、条件显隐、分组与页签、参数模板** |
| 内部网络 = 实现 | `SubGraph`（`subgraph.py`，另一张图当一步跑）；**图工具 = 一张普通图**（`<工程>/Tools/`） | 图文件在"工程资产"里，不随节点定义走；没有"资产自带网络"的打包口径 |
| Python Module / 事件 | `PythonCode` 节点（同权限）、`HostStep`（宿主步骤）、`Gate`（决策门） | **节点级钩子**（创建 / 参数变化 / 载入）没有 |
| Definition 多版本 | rez 包版本；工程资产目录 | **同一节点多版本共存、按图选版本**（`MyBlur#2`）没有 |
| 参数提升 promote | `GetParam` / `SetParam`（`param_nodes.py`）+ 端口旁胶囊（`dock`，`pySchema.attachEndpointChip`） | **"一键提升/收回"的编辑动作**（现在手放 `GetParam` 再连线） |
| 错误上端口 | `errvals` 面板 + 运行轨迹 + 卡片 `⚠` | 基本对齐 |
| 分类/命名空间 | 节点 `CATEGORY`（`a/b/c`）；工程名派生（`.lproject` stem） | 分类**多级/自定义分组**够用即可 |

## 三、学什么（P0–P5，按风险从低到高）

### P0 参数模板（**推荐先做**，纯声明面）

目标：参数不只剩 `default`，还有**范围/步进/单位/分组/顺序**，控件不飘。

字段草案（全部落在 `INPUT_TYPES` 的 `opts`，与现有 `tooltip`/`container`/`std_type` 同一层）：

```python
"radius": ("INT", {"default": 8, "min": 1, "max": 256, "step": 1, "unit": "px",
                   "group": "模糊", "order": 10, "label": "半径"}),
```

- 判据：`/api/pygraphs/object_info` 里这些键**原样带出**；前端数字框吃 `min/max/step`、
  分组按 `group`+`order` 归堆；`l_node_engine_selftest` 加一组"opts 直通 + 不覆盖未知键"。
- 不给默认值的节点**不报错**（都缺省 = 今天的行为）。

### P1 条件显隐 `visible_when` / `disabled_when`

目标：一个参数是否显示/可编辑，由**其它参数的值**决定（HDA 最实用的一样）。

```python
"image_tail": (..., {"visible_when": {"multi_shot": True}}),
"quality":     (..., {"disabled_when": {"provider": ["agnes", "vidu"]}}),
```

- **受限 DSL，不用 python `eval`**：只允许 `{参数: 字面量}` 或 `{参数: [允许值…]}`，
  以及外层 `any/all` 两个组合词；求值在引擎（`guard` 白名单同一层），前端做**同样的浅判定**只为即时显隐。
- `disabled_when` 只改"能不能编"，**不改执行**：隐藏/置灰的参数照旧参与执行（值还在）。
- 判据：selftest 一组（`any/all` / 嵌套参数不存在 / 值类型不匹配 → 当"显示"即 fail-open）；
  画布探针一条（切 `multi_shot` → `image_tail` 出现/消失，`data-hidden` 可查）。

### P2 节点版本（用户口径 2026-10-19：「节点直接有版本，可以随时在资产图里切换某个节点的版本」）

**定义侧（库）：同一 `id` 多行，`(id, version)` 是主键**

```sql
nodes(id, version, category, label, json, sha256, PRIMARY KEY(id, version))
```

- `version` 是**节点自己的版本**（不是库版本、不是 rez 包版本）；库 `meta.version` 只记"这一版库"；
- 注册：`id` **只注册最新版**（`latest = max(version)`）进节点库，避免同一种节点刷屏；
  旧版**照样可被图引用**（`REGISTRY_VERSIONS[(id, req)]` → 那份声明），`get_node_info(name, version=)` 带版本取；
- `/object_info` 每个节点多两个字段：`"versions": [1,2,3]`、`"latest": 3`（前端版本菜单就靠它）。

**图侧（资产图里切换）**

- 节点上多一个字段 `asset_version`（整数；**缺省 / 缺字段 = 跟随 `latest`**）；
- 切换 = 改这个字段：`mutate({step:true})` ⇒ **Ctrl+Z 能退**；同一 `id` 的多个节点可各选不同版本；
- UI：卡片标题旁一枚 `v2` 徽标（**非最新**才高亮，最新低调）；属性面板 / 右键菜单「版本 ▸」列出 `versions`
  （来自 `/object_info`），末尾一项「跟随最新」= 清掉 `asset_version`；
- 切换后**端口可能变**：立刻按新版本重算端口与控件（`GET /api/pygraphs/ports?kind=node&id=…&version=…` 或本地
  `object_info` 的版本切片）；参数值**按名字保留**，类型不兼容的那个回落新默认值 + 卡片 `⚠` 提示；
  旧端口上的连线**保留但悬空**（与"端口不存在"同一套处理：`errvals` 报、不静默执行）。

**执行侧（解析顺序，定死）**

1. 取节点 `class_type = id`；
2. `asset_version` 有值 → 解析 `(id, 该版本)`；**不存在 = 明确报错**（`未知节点版本 MyBlur#9`），
   **绝不回落 latest**（否则"改了却没生效"）；
3. 无值 → 用注册表里该 `id` 的 `latest`（图打开时可提示"该节点有新版本 v3"）。

**缓存（必须动，否则切版本复用旧结果）**

- 增量指纹里必须含 **`asset_version`（解析后的具体数字）**：`(id, 2)` 与 `(id, 3)` 是不同指纹；
- `ALWAYS_RUN` / 子图 / 图工具三处的口径不变，只是"版本"进了 key。

**判据**

- 库里 `MyBlur` 有 `v1`（输出 1）与 `v2`（输出 2）→ 同一张图放两个节点，分别 pin `1` / `2` → 跑出两种结果；
- 图里 pin `9`（不存在）→ `/validate` 与跑图**都明确报错**，报错文案带 `id#9`；
- 切换版本后**指纹变化**（第二次跑不复用第一次的缓存）；
- 切换后端口变化 → 卡片 `⚠`、`errvals` 有记录、连线不静默丢弃；
- 旧版不进节点库面板（只 latest 出现），但「版本 ▸」菜单列得出来；
- 自证：`l_node_engine_selftest` 新增一组（多版本解析 / 未知版本报错 / 指纹含版本）+ 探针一条
  （切换菜单 → 端口与执行结果都跟着变）。


### P3 节点钩子 `hooks`

目标：节点被创建 / 参数变化 / 图载入时跑一小段代码（HDA 的 OnCreated / OnInputChanged / OnLoaded）。

```python
HOOKS = {"on_param_change": {"param": "provider", "call": "mypkg.hooks:refresh_models"}}
```

- **必须过 `guard.check_import_allowed` 白名单**（与 PythonCode 同一条）；`dry_run` 期间钩子**只读**；
- 钩子返回值**只允许**改 `opts`（候选值 / 显隐）或 `ui`，**不许改图结构**；
- 失败策略：钩子异常 → 记一条 `⚠` 到节点 `errvals`，**不阻断执行**（同 HDA 的 cook warning 不致命）；
- 判据：selftest 三组（白名单拒 / 非白名单拒 / 异常降级成警告）；探针一条（改参数 → 候选值真变）。

### P4 声明文件格式（与上一问「单独文件格式存节点」合并）

目标：一份文件描述**一到多个**节点（用户 2026-10-19 追加口径：「我也希望单文件可以封装多个节点」），
代码只当**执行引用**。一个文件 = 一个**资产库**，等价于 `.hda` 包一整个库而不是单个类型。

**格式已定（2026-10-19）：「JSON 为规范格式，YAML 不进引擎」**

- 引擎包**只用标准库**（`package.py:8` 的硬口径 + `requires` 只有 python）⇒ 解析器只能是 `json`；
- 所以规范文件是 **`nodes/*.json`**，引擎自己读、自己校验、自证里也读得动；
- 想写 YAML 的人（手写更舒服）走**可选转换器** `tools/nodelib_convert.py`（`pyyaml` 只在那个工具的环境里，
  **不进 `l_node_engine/package.py` 的 requires**）：`x.yaml → x.json`，落盘后**只有一条加载路径**；
- 为什么不是"两个都支持"：两套解析 = 两份校验 + 两种错误文案 + 自证要跑两遍，而收益只是"少按一次转换"。

```json
// nodes/tex_extra.json —— 一份文件 = 一个库（library 只用于报错/排序，不当命名空间前缀）
{
  "library": "tex_extra",
  "version": 1,
  "defaults": { "category": "tex/extra" },
  "nodes": [
    {
      "id": "MyBlur",
      "version": 1,
      "category": "tex/filter",
      "label": "我的模糊",
      "inputs": [
        { "key": "image", "type": "IMAGE", "label": "图像" },
        { "key": "radius", "type": "INT", "default": 8, "min": 1, "max": 256, "label": "半径", "container": "number" }
      ],
      "outputs": [{ "name": "image", "type": "IMAGE" }],
      "exec": { "kind": "callable", "target": "mypkg.filters:blur" }
    },
    {
      "id": "MyTile",
      "label": "我的平铺",
      "inputs": [
        { "key": "image", "type": "IMAGE", "label": "图像" },
        { "key": "n", "type": "INT", "default": 2, "label": "份数" }
      ],
      "outputs": [{ "name": "image", "type": "IMAGE" }],
      "exec": { "kind": "pygraph", "graph": "pro/Tools/tile" }
    }
  ]
}
```

**单节点简写也认**：文件顶层直接写 `id`（没有 `nodes` 数组）= 一个仅含一个节点的库，
解析时归一成 `nodes: [那份]` —— 两种写法在加载器里**只有一条路径**。

**加载与冲突（定死，别在实现时再议）**

- 位置：`<包>/nodes/*.json`（作者源）+ `<包>/nodes/*.sqlite`（库产物，见 P4d）+ `<包数据目录>/nodes/*`（用户自己加）；
- 顺序：包内按**文件名**排序 → 再叠数据目录（**本地赢**：同 `id` 覆盖，等价 `register_node` 的后写覆盖）；
- **同层重复 = 报错**（两个文件都点名）—— 不许静默后写赢，那会让"我加的节点没生效"变成谜；
- **整文件原子**：任一条目校验失败（缺 `id`/`exec` 不认识/端口类型不在词表）→ **该文件整份不进**，
  报错写「文件 : 条目 : 原因」；不允许"一半节点进了注册表"的诡状态（HDA 也是整份拒收）；
- 关掉/重载走既有插件的口子（`plugins.json` 里给这个库一个开关 + 面板「🔌 插件」里的重载按钮）；
- **产物链**：作者写 `nodes/*.json` → `nodelib_pack` 生成 `*.sqlite`（明文库）→ 需要外发再加密成 `*.nlib`。
  加载器三种都认，最终都归一到每节点一条声明再注册（见 P4d 的 schema）。

**仍与 `.py` 节点共用一条出口**：`get_node_info()` 只多认一类 `NODE_SOURCE="file"`（今天已有
`class` / `comfy` / 函数派生三类先例）→ 画布 `PyNode`、canvas 后端、端口 API、增量缓存**都不用改**。

**分两步**：**P4a** 只 `callable` + **静态端口** + 单/多节点两种文件；**P4b** 再补 `pygraph` 引用与动态端口。
**明确不做**的第一版：`hidden` 注入、`SINGLETON`、`register_hashed_inputs`（增量指纹）—— 这三样先只允许 `.py` 节点。

**判据**（selftest + 探针，缺一不算完）：

- 一文件 3 节点 → 注册表里 3 个都在、节点库按 `category` 归堆；
- 单节点简写与 `nodes` 数组**解析成同一份结构**；
- 其中 1 条坏 → **整份不进**，报错里点名文件 + 条目 + 原因（其余文件不受影响）；
- 同层两个文件写同一个 `id` → 报错；包内 + 数据目录同名 → 数据目录赢；
- `exec` 目标不存在 / 不在导入白名单 → 该节点**不注册**且卡片能说清（别让整张 `/object_info` 500，函数插件那次教训）；
- 探针一条：节点库出现这些节点 → 落画布 → 真跑一次 → 结果对。

**打包产物（P4a 的一部分，不再是"可选项"）—— SQLite 明文库**

作者写 `nodes/*.json`；`nodelib_pack` 把它导入成**一个 SQLite 库文件**（结构见 P4d）：

```
tex_extra.sqlite          # 明文库：meta + nodes + assets（BLOB）
tex_extra.nlib            # 需要外发时：同一份 sqlite 字节 AES-GCM 加密 + 明文头 + 签名
```

- 加载器认**目录**（`nodes/*.json`）、**`*.sqlite`**、**`*.nlib`** 三种，走**同一份**节点注册与校验；
- 校验/防篡改：`nodes.sha256` 逐行 + `meta` 的签名（rez 包本来就按 sha256 走增量部署）；
- ⚠ 大资源（图标/png）**不要塞进节点声明**：声明里只写 `"icon": "assets/blur.svg"`，字节进 `assets` 表。

### P4c 加密发布包 `.nlib`（⚠ 容器与加密粒度见 §0.4：v2 = 行级加密 + 单一 `.nlib`）（用户口径 2026-10-19「我还是希望做二进制加密」）

**边界先说清**：这是**发布包**的加密，**开发态永远是明文**（`.py` / `nodes/*.json`）—— 否则 diff、AI 改写、
热重载全废。加密只解决"外发之后别被随手改、别被随手转给同事、别明文躺在别人硬盘上"，
**不解决"防住有动机的人"**（key 在客户端，早晚能被捞出来）。

**容器格式（一个文件、二进制、只加密 payload）**

```
magic   : b"LNLIB\x01"            # 4+1 字节，加载器据此认"这是加密库"
u32     : header_len             # 小端
header  : JSON（**明文**，只有元数据，泄露面可控）
          {"format":"lugwit-nodelib/2","alg":"aes-256-gcm","kdf":"hkdf-sha256",
           "keyslot":"license:studio-2026",            # 用哪把 key（见下）
           "library":"tex_extra","version":1,
           "author":"…","copyright":"…","license":"…",
           "payload":"sqlite","payload_sha256":"…",
           "sig":{"alg":"ed25519","key":"studio-2026","sig":"…"}}
payload : AES-256-GCM(**SQLite 库文件的整份字节**)   # nonce(12B) + ciphertext + tag(16B)；见 P4d
```

- **先签名再加密**：`sig` 签的是 `header 归一化 + 明文 payload 的 sha256`，验签在解密后做 ——
  加密只防看，签名才防"冒名"。两件都做，缺一不可；
- 明文头故意留 `library/author/license/keyslot`：别人拿到文件时"这是谁的库、要什么许可"仍可辨
  （版权声明要能看见，不能连作者一起藏起来）。

**密钥从哪来（三档，按强度递增）**

| 档 | key 来源 | 能防什么 | 代价 |
|---|---|---|---|
| L1 | 发布包**内置**的 key（放在**包里**，不放在 `.nlib` 里） | 防止明文外流、防随手改 | 拿到包的人能提取 |
| L2 | `license.json`（许可证文件，DPAPI 加密存本机，同 `depot_bridge` 的 refresh token 那套） | 防"整个目录拷给别人" | 许可证要单独发一份 |
| L3 | `license + 机器指纹`（HKDF(license_key, hostname+用户名)） | 防跨机复制 | 换机 = 换证；重装系统可能失效 |
| L4 | **联网取 key**（服务端签发，带席位/到期） | 真正的商业控制（可吊销、可计量） | 离线不可用；要部署签发端 |

默认 **L2**；企业/售卖场景升 L3；L4 只在真有商业需求时做（它已经不是"文件格式"问题，是授权系统）。

**一个 `.nlib` = 一个库（可装多个节点）—— 这就是"类似 HDA 封装"的答案**

用户 2026-10-19 追问「`.nlib` 支持类似 hda 封装多个节点么」→ **支持，而且是它天生的形态**：

| | Houdini | 我们 |
|---|---|---|
| 单类型 | `.hda`（一个文件 = 一个数字资产） | `nodes/x.json`（`id` 在顶层）= 单节点库 |
| 资产库 | `.hdalc` / 资产库（多类型） | **`.nlib` / `*.sqlite`（一个文件 = 一个库，`nodes` 行装 N 个）** |

- payload 现在是 **SQLite 库文件**（2026-10-19 用户定案「直接使用 SQLite」，见 P4d）→ **整份字节 AES-GCM 加密**；
- **库级原子**：解密后先按 `nodes.sha256` + `meta` 签名整库校验 ⇒ 任一行对不上 = **整个库拒收**
  （防止有人只替换其中一个节点）；
- **一个 `.nlib` 一个 `keyslot`**（一副许可证）⇒ 要让不同节点走不同许可，就**拆成多个 `.nlib`**
  （别在一个库头里做多密钥，那会把许可逻辑塞进容器格式）；
- **多版本两种放法**（对应 §八 待拍板 6）：①同库内并列（`nodes(id, version)` 主键 = 1 / 2，头 `version` 是库版本）；
  ②同库不同文件（`tex_extra-1.nlib` / `tex_extra-2.nlib`）。①更省事、②更好审计 —— 实现时先按①，②作为发行约定；
- 加载：解密 → `conn.deserialize(bytes)` → `SELECT * FROM nodes` → 每行的 `json` 走与目录**完全相同**的解析
  （单节点简写也认）；库里任一节点注册失败 → **整库不进**（沿用"整文件原子"，口径不变）；
- `.nlib` 里也可以**只装一个节点**：单节点库就是"最像 `.hda`"的那一档，工具不为此分叉。

**工具与判据**

- `tools/nodelib_pack.py`：明文目录 → `.nlib`（读 `license.json` 取 key，签名后加密）；
- **查看器**（用户 2026-10-19 追问「是不是还多写一个 `.nlib` 的查看器」→ **要**，但"查看"分两级，
  **谁都不许注册节点、不许执行 `exec` 目标**）：

  | 级别 | 命令 | 没 key 时 | 有 key 时 |
  |---|---|---|---|
  | **头查看（必做）** | `nodelib_inspect <file>` | 打印**明文头**：格式版本 / 库名 / 作者 / 版权 / 许可 / `alg` / `keyslot` / 签名状态 | 同左（头本来就能读） |
  | **内容查看（必做）** | `nodelib_inspect <file> --with-key` | 明确说"这是加密库，需要 `<keyslot>` 的许可证（`license.json`）"——**不许**显示成"文件损坏" | 解密后只读列出：节点清单（`id` / label / 中文参数表 / 类型 / 默认值）、`exec` 目标（**只显示不导入**）、图标/示例图路径 |

  附加开关（都是只读）：`--json`（给 CI / AI 用）、`--verify`（重算 sha256 + 验签 → `verified` / `modified`）、
  `--diff other.nlib`（两版库的节点/参数差异，发行前对拍）、`--assets-dir <dir>`（把图标/示例图导出来看一眼，
  **不覆盖已有文件**）。GUI 那层（编辑器里「查看库」）晚做，先有 CLI 就够排查。
- 加载器：目录（`nodes/*.json`）/ `*.sqlite` / `.nlib` 三条入口 → **解密后落到同一份解析与校验**（整库原子照旧）；
- **共用一份实现**：key 解析（`license.json` / 指纹 / 环境变量）、头解析、解密、校验都在引擎里，
  查看器与加载器都调它 —— 不许查看器自己再写一套（否则"装得上但看不了"这种漂移必然出现）；
- selftest：①加密→解密→节点数/端口与明文一致；②**改一个字节 → GCM tag 失败 → 整份不进**并说"文件被改或 key 不对"；
  ③错 key 明确报"缺 `license.json` / 指纹不匹配"（**不许**静默跳过）；④密文库与明文库同名 → 按既有冲突规则报错；
  ⑤**查看器无副作用**：跑完 `--with-key` 后注册表节点数不变（`dry_run` 同一口径）、磁盘无新文件；
  ⑥**多节点库**：一个 `.nlib`（payload=sqlite）装 3 个节点 → 解密后 3 个全注册、`category` 各自归堆；
  **改库里一行的 `json` 却不更新 `sha256` / 签名** → 校验失败 → **整库拒收**并点名该 `id`
  （GCM 只挡密文改动，挡不住"用别人的 key 重加密"这种冒充 —— 那种由签名兜底）；
- ⚠ **key 不许硬编码在包里**（会被 `grep` 出来，也进 git）：只走 `license.json` / 凭证存储 / 环境变量；
- ⚠ `.nlib` **不进 git 二进制大文件习惯**：发布包走部署通道分发放，仓库里留明文源。
- ⚠ 查看器的 `--with-key` **不是**"解密泄露口"：它只做"给有许可证的人看自己买的东西"；
  明文头之外的任何内容一律不下发、不落临时明文文件（要落就落 `%TEMP%` 并在退出时删）。

**效率评估（2026-10-19 用户追问「这种格式高效么，有开源的高效二进制格式可以参考么」）**

结论：**够用，不必换**。理由是我们**不在热路径上**：

| 环节 | 实际开销（量级） | 判断 |
|---|---|---|
| 一个库的体积 | 声明 10~300 KB；图标/示例图按路径引用（BLOB 存库） | 小 |
| 读 SQLite：`SELECT` 全部 `nodes` 行 + `json.loads` 每行 | 毫秒级（C 实现；`sqlite3` 是标准库） | 不是瓶颈 |
| 明文库可直接 `SELECT`；加密库先解密再 `deserialize`（内存） | 毫秒级 | 不是瓶颈 |
| AES-256-GCM 解密 300 KB | **<1 ms**（AES-NI，几十 GB/s 级） | 不是瓶颈 |
| **真正的耗时** | Python `import` + 几百个节点类的 `INPUT_TYPES()` 现算 + 注册 | ← 要优化就优化这里（命中缓存 / 懒注册） |

所以优化优先级：①**加载结果缓存**（按 `(路径, size, mtime, sha256)` 命中，直接复用注册表快照）＞ ②**懒注册**
（节点库面板按分类按需取 `get_node_info`）＞ ③**资源按路径**（已定）＞ ④换序列化格式（收益 μs 级）。

**参考过的开源二进制格式（都是对的，但对我们多是过度工程）**

| 格式 | 优点 | 为什么我们不用 |
|---|---|---|
| **SQLite** | **Python 内置 `sqlite3`（零依赖）**；单文件库；事务、索引、CRC、页级增量写 | **最值得留作后手**：真需要"单文件库 + 可查询 + 增量更新 / 只换一个节点"时它比"自造容器"省事得多（数据列可以 `id/version/category/json/blob`）。P4a 阶段不做，留成备选 |
| **CBOR**（RFC 8949） | 标准、紧凑、**确定性编码**（canonical） | 只在"要省那点字节"或"签名要跨语言稳定"时才值；我们有 `manifest sha256` 覆盖**原始字节**，不需要重排 |
| **MessagePack** | 最快最省事的二进制 JSON | 同上；且不带 schema，换过去等于"同一份 JSON 压紧"，收益不够 |
| **FlatBuffers / Cap'n Proto** | **零拷贝**、带 schema、适合热路径 / 游戏 | 需要 codegen + 双份维护（`.fbs`/`.capnp` ↔ 代码）；我们的热路径是**执行**不是**读定义** |
| **Protobuf** | 生态好、向后兼容、varint | 同 FlatBuffers：schema 编译期绑定，声明面频繁人工编辑时反而更痛 |
| **Avro** | schema 随数据走（header 内嵌），与"一文件一库"概念最像 | 为批处理流设计；我们没有流式需求 |
| **Parquet / Arrow** | 列式、分析快 | 我们不是"按列查询的表"，用不上 |
| **Zstandard** | 比 deflate 快且压得更好 | Python 3.12 要额外包（`zstandard`），而 zip 是内置；体积/耗时都不是瓶颈 |
| **Blosc / HDF5 / Zarr** | 数值/大数组压缩 | 只对**贴图/纹理缓存**有意义（另案：贴图结果缓存），不是节点声明 |

**两条与"高效"相关的铁律**（写进坑里）：① **签原始字节**（`nodes.sha256` 覆盖那一行 `json` 的原样；
容器头里的 `payload_sha256` 覆盖 SQLite 文件原样），不要"解析 → 再序列化 → 签"，否则键序/空白一变签名就假失效；
② `assets` 表里的 png/jpg **不要再压缩**（直接存 BLOB，压缩已压缩的图只会慢）。

### P4d `.nlib` vs SQLite（⚠ 结论已演进为 §0.4：`.nlib` 本身就是 SQLite）（2026-10-19 用户追问「直接使用 SQLite 格式可以么，能加版权和加密么」）

**能，而且都做得到**（不用 SQLCipher 也能加密）：

| 能力 | SQLite 怎么做 | 我们这边的结论 |
|---|---|---|
| 多节点 | `nodes` 表，一行一个节点（`id` / `version` / `category` / `label` / `json` / `sha256`） | 天然支持 |
| 版权 / 许可 | `meta(key,value)` 放 `library/version/author/copyright/license/keyslot`；**每行冗余** author+license（防只抠一行） | 可行 |
| 签名 | 对 `meta` + 每行 `json` 的 **规范化序列化** 做 Ed25519；**不签整文件**（页布局会变） | 与 P6 同口径 |
| 加密 | ①**整文件 AES-GCM 包一层** → 解密 → `conn.deserialize(bytes)`（**实测 py3.12 + sqlite 3.49.1 支持**，零临时文件）；② SQLCipher（`PRAGMA key`，另一个格式，要 `pysqlcipher3` 编译依赖，**不采**） | ①可行，和 `.nlib` 同一套 key |
| 增量 | `INSERT/UPDATE` 单行，不用重写整库；`VACUUM` 收碎片 | 强项 |
| 查询 | `WHERE category LIKE …`，不必把全库读进内存 | 强项（库大了才有意义） |
| 依赖 | **Python 内置 `sqlite3`**（`l_node_engine` 的"只用标准库"不受影响） | 零依赖 |
| 可 diff / 可手改 | 二进制，不行（与 `.nlib` 一样） | 都要靠"明文 JSON 源"那一层 |

**选型结论（2026-10-19 用户拍板：「那就直接使用 SQLite 吧」）**

- **库格式 = SQLite**：明文库 `*.sqlite`（引擎 `sqlite3` 直读，零依赖）；分发/外发 = `*.nlib`
  （容器头 + **整份 SQLite 字节** 走 AES-256-GCM）；
- **作者源仍是明文 JSON**（`nodes/*.json`，diff / grep / AI 改写靠它）→ `nodelib_pack` 导入成 `.sqlite` →
  需要外发再加密成 `.nlib`。**不要**让人直接手写 SQLite（二进制不可 diff，也说不清 schema 版本）；
- **加载器三条入口**：目录里的 `nodes/*.json`（开发）／`*.sqlite`（明文库）／`*.nlib`（加密库）→
  归一后走**同一份**节点注册与校验（整库原子不变）；
- schema（定稿，改动要升 `format` 版本）：

```sql
CREATE TABLE meta(key TEXT PRIMARY KEY, value TEXT);
--   library / version / format("lugwit-nodelib/1") / author / copyright / license
--   / keyslot / signed_by / signature / created / engine_min
CREATE TABLE nodes(
  id        TEXT NOT NULL,
  version   INTEGER NOT NULL DEFAULT 1,
  category  TEXT,
  label     TEXT,
  json      TEXT NOT NULL,          -- 就是 P4 那份节点声明（单节点结构）
  sha256    TEXT NOT NULL,          -- 该 json 的 sha256（签它，不签整文件）
  PRIMARY KEY(id, version)
);
CREATE TABLE assets(path TEXT PRIMARY KEY, bytes BLOB, sha256 TEXT);
-- 图标 / 示例图 / 预览：**按路径引用**，BLOB 存这里（单文件自包含）
```

- **签名**：Ed25519 签 `meta`（按 key 排序的规范化 JSON）+ 每行 `nodes.sha256` 的排序串；
  公钥在 `trusted_keys/`，三档 `verified` / `modified` / `unsigned` 与 P6 同口径；
- **加密**：整份 SQLite 字节 → AES-GCM（`l_nodelib`）→ `.nlib`；解密用 `conn.deserialize(bytes)`（实测可用），
  **不落临时明文文件**；
- **增量（本机）**：明文 `*.sqlite` 里 `UPDATE` 单行即可；**分发包仍整包重发**（重加密成本可忽略）——
  别把"行级增量下发"写进 v1 的承诺里；
- **迁移**：`.nlib` 头保留 `"payload": "sqlite"`（`"zip"` 仅作历史/备选），换格式不改容器；
- ⚠ **不要两套都实现**：v1 只做 `sqlite`。

**`*.sqlite` 本身能不能加密（2026-10-19 用户追问）**

能，但要认清三条路的代价 —— **`*.sqlite` 与 `.nlib` 的分工就是"明文的归明文、外发的归容器"**：

| 做法 | 结果 | 代价 / 判断 |
|---|---|---|
| ① **整文件 AES-GCM 包一层** | 已经不是 `.sqlite` 了，就是 `.nlib`（我们已定） | 0 额外依赖；**这就是"明文库加密"的答案**：加密后换扩展名 |
| ② **SQLCipher / SEE**（页级加密，`PRAGMA key`） | 文件仍能当数据库用，但**不是标准 SQLite 格式** | 要 `pysqlcipher3` / `sqlcipher3` 绑定（Windows 无现成 wheel，得编译）；与"引擎零依赖"冲突；SQLCipher 是独立分支，跨工具兼容差 → **不采** |
| ③ **应用层字段加密**（每行 `json` 先加密再存 BLOB） | 库能打开，但内容是密文；`SELECT … WHERE category` 只能对明文字段查 | key 管理同 ①，却把可读/可查/可 diff 全废 → **不采** |

**做法（推荐，已定）**：**不落明文**。作者源是 JSON；`nodelib_pack` 可以**直接输出 `.nlib`**（json → 内存里建库 →
加密落盘），全程不在磁盘留明文 `.sqlite`。`.sqlite` 只在"内部要 `sqlite3` CLI 查、要备份、要 CI 对拍"时生成，
放受控目录（或用后即删）。要"内部也不明文"，勾 `--no-plain` 即可（工具默认在数据目录留一份明文库作缓存，
可关）。









### P5 参数提升（编辑动作）

目标：选中内层节点的某个参数 → 「提升到接口」→ 自动生成图级 `params` 一行 + 端口旁 `GetParam` 胶囊（`dock`）；
反向「收回」= 删那行 + 摘胶囊（**线不动的前提**：胶囊是 `GetParam`，摘掉前先确认没有别处引用）。

- 判据：一次提升后图上节点数 +1（`GetParam`）、接口面板多一行、端点上出现胶囊；
  探针一条（提升 → 跑 → 改接口默认值 → 结果跟着变）。

### P6 完整性 / 署名 / 许可证（与 P4 配套；**加密在 P4c，这里只管验真与署名**）

用户 2026-10-19 追问：「如何防篡改 / 版权保护」。先把两件事分开 —— 它们的技术手段完全不同：

| 要保的 | 能不能真保住 | 做法 |
|---|---|---|
| **完整性**（没人偷偷改过） | **能**（前提：公钥可信） | `manifest.json` 里每份 `.json` 带 **sha256** + 整份 manifest 带**签名**（Ed25519，私钥在发布机，公钥随包内置 `<包>/trusted_keys/*.pub`） |
| **署名**（改了别冒充原作者） | **能** | 签名覆盖 `library` / `author` / `license` 字段；改任何字节 → 验签失败 → 标"已修改" |
| **保密**（别抄走算法） | **客户端做不到** | 只有一条靠谱路：**算法留在服务端**（走既有远端引擎契约 `/engine/run`，客户端只剩声明壳） |
| **阻止别人改自己的副本** | 做不到（也没必要） | 我们**允许**改：改了就降级成"本地未签名"，企业开关 `nodes_require_signed: true` 时才拒载 |

**三档信任状态**（`library.trust`，加载器算出来，不写死在文件里）：

⚠ **v1 已经落地的是 HMAC-SHA256**（`nodelib.verify`，纯标准库；见 §0.5 那行）。Ed25519 需要 `cryptography`，
等 `l_nodelib`（P4c）接上；`meta.sig_alg` 已留位，两套共用 `signature/signed_by` 字段，换算法不动 schema。

| 状态 | 条件 | UI |
|---|---|---|
| `verified` | 签名通过，且 sha256 全对 | 节点库/卡片上 `🔏 已验证`（悬停给 library / author / license） |
| `modified` | 有签名但哈希对不上，或声明被本地改过 | `⚠ 已修改`（照常能用 —— diff/手改/AI 改写是我们的正常用法） |
| `unsigned` | 没有签名 | `· 未签名`；`nodes_require_signed=1` 时**不注册**并说明原因 |

**字段**（都在库文件顶层，一起进签名）：

```json
{
  "library": "tex_extra",
  "author": "某某工作室",
  "copyright": "© 2026 某某工作室",
  "license": "CC-BY-4.0",
  "signature": { "alg": "ed25519", "key": "studio-2026", "sig": "base64…" }
}
```

（`payload_sha256` 由构建脚本生成，**不手写**；`license` 也可以是机器可读的私有串，
如 `"commercial: seat=1; expires=2027-12-31"`。）

**判据**：selftest 三组（验签通过 → `verified`；改一个字节 → `modified` 但仍可注册；带签名但公钥缺失 →
`unsigned` + 说明）；`nodes_require_signed=1` 时未签名库**不注册**且原因写进日志/面板；
**`*.sqlite` 库、`.nlib` 库与目录源**走**同一份**校验（签名在 `meta`、逐行哈希在 `nodes`，不许两套）。

⚠ **别做的事**：① 用 **HMAC/对称密钥**当版权保护（key 就在客户端，等于没锁；要加密就用 P4c 的 AES-GCM +
许可证绑定）；② 把"防篡改"说成"防抄袭"；③ 给 **rez 包里的 `.py` 执行体**做加密/混淆（和热重载、diff、
AI 改写冲突，成本 > 收益）—— 要保护算法就把它留到服务端（远端引擎），P4c 加密的是**发布包**
（声明 + 资源），不是 rez 包里的源码；④ 明文头里别塞敏感内容（`.nlib` 头是**明文**，只放库名/作者/许可/alg）。

## 四、明确**不学**的

| 不学 | 为什么 |
|---|---|
| Hscript / `ch()` 表达式语言 | 我们已有"值 + 类型 + 执行器"模型；再塞一套表达式 = 两套语义打架。要"参数引用参数"用 `GetParam` 表达 |
| cook 依赖模型（SOP 上下文、时间依赖） | 我们是"拓扑执行 + 增量指纹"，没有 SOP/DOP 那套上下文与时间求值 |
| `.hda` 的**定义格式**（把节点定义压成二进制） | **不学它是格式**：声明仍是 JSON、代码仍在 `.py`（diff / 合并 / grep / AI 改写都要文本）。**但发布包可以加密成二进制**：`.nlib`（P4c），那是外发容器，不是定义格式 |
| 端口类型系统 | 已有唯一词表（`工具契约_ToolSpec.md` §5.4）；照搬 Houdini 的类型会再造一份 |

## 五、现状 vs 目标的差距（可核对）

| 项 | 今天 | 证据 |
|---|---|---|
| 参数中文名 | ✅ 每节点每参数都有中文 `label` | `param_labels.py`（引擎 + 贴图各一份）；`l_node_engine_selftest` 第 38 组 |
| 参数范围/步进 | ❌ 只有 `default` | `param_spec.py` / 各 `INPUT_TYPES` 里没有 `min/max/step` |
| 条件显隐 | ❌ | 前端只有"当前模型不认这个参数 → 置灰"（`videoCaps.js`），那是**能力**不是**参数依赖** |
| 资产版本 | ❌ | 图里没有 `asset_version` 字段（`l_mindmap_mmd/web/src/pySchema.js` 的节点形状里没有） |
| 节点钩子 | ❌ | `HostStep` 是"图里的宿主步骤"，不是"节点事件" |
| 独立声明文件 | ❌ | 节点全是 `.py` 类（`l_node_engine/__init__.py` 导入即注册）；**P4 的口径是一文件一个库（可装多个节点）**，且整文件原子 |
| 参数提升 | 半 | `GetParam`+胶囊机制齐了（`pySchema.unfoldEndpointChip`），缺"一键提升"的 UI |

## 六、判据总表（做的时候照着跑）

| 阶段 | 自证 |
|---|---|
| P0 | `wuwor l_node_engine -- l_node_engine_selftest`（新组：opts 直通）+ `l_mindmap_mmd_web_build` + 探针 `params`（数字框带着 `min/max` 属性） |
| P1 | selftest 新组（DSL 求值 + fail-open）+ 探针 `params`（切开关 → 参数显隐）；**两后端**（DOM `PyNode` 与 `canvas` `CanvasSurface`）各一遍 |
| P2 | selftest 新组（`#2` 命中 / `#9` 报错）+ 宿主 `/validate` 一条 |
| P3 | selftest 三组（白名单 / 拒 / 异常降级）+ 探针一条 |
| P4 | selftest（坏 JSON / 缺字段 / `exec` 白名单 / 与 `.py` 共存）+ 探针 `tools`（节点库→落卡→真跑） |
| P5 | selftest（提升后图结构 = params + GetParam + dock）+ 探针一条 |

## 七、坑与风险

1. **别在声明文件里引用可变全局**：文件节点的 `exec` 只允许 `module:callable`，参数一律走端口；
2. **`get_node_info()` 是唯一出口**——别在 `graph.py` 或宿主里再拼一份端口，否则两处漂（
   2026-10-17 `params_from_callable` 不复用的坑就是同类）；
3. **动态端口与声明文件互斥**（`AgentParams` / `PythonCode` / `CallFunction` 这类现算端口的节点**不许**搬进声明文件）；
4. **`dry_run`（演练模式）下钩子与文件节点都不许有副作用**（同 `agent_tools.set_dry_run` 口径）；
5. **条件显隐是显示层**：隐藏不等于不执行、不等于不是必填（必填判据仍看 `spec.required`）；
6. **版本不许"悄悄升"**：节点升版必须是显式动作，否则同一张图跑出两种结果；
7. **解析器只有 `json`（标准库）**：`l_node_engine` 的 `requires` 只有 python，不许为声明文件加 `pyyaml`；
   要 YAML 就写进 `tools/nodelib_convert.py`（那个工具自己的环境里装），**转出来的 `.json` 才是唯一加载路径**；
8. **图里落节点前先校验**：文件节点若 `exec` 目标不存在 → 卡片写"未注册/目标缺失"，别崩整张 `/object_info`（
   函数插件那次 500 的教训）；
9. **库文件是原子的**（2026-10-19 追加）：一文件多节点时，**任一条目坏 → 整份不进**并点名"文件 : 条目 : 原因"；
   别做"好的先注册、坏的跳过"—— 那会让"我明明写了三个节点怎么只出现两个"变成谜；
10. **同层同名不许静默覆盖**：包内两个文件写同一个 `id` = 报错（只有"包内 vs 用户数据目录"这一层允许覆盖，本地赢）；
11. **哈希/签名覆盖声明原文**：`nodes.sha256` 算的是那一行 `json` 的**原样字节**（不是"解析后再序列化"），
    否则键序 / 空白 / 行尾一变签名就假失效；
12. **别为"高效"换格式**：瓶颈在 `import` 与 `INPUT_TYPES()` 现算，不在 `SELECT` / `json.loads`；要提速先做
    "加载结果缓存 / 懒注册（按 `category` 查）"，`assets` 直接存 BLOB（别再压缩已压缩的图片）。

## 八、待拍板（做之前要定）

**已经拍完的（2026-10-19，别再翻）**：

| 已定 | 结论 |
|---|---|
| 作者源格式 | **JSON**（`nodes/*.json`，标准库 `json`）—— 引擎 `requires` 只有 python，不许为它加 `pyyaml`；YAML 只作可选转换器的输入 |
| **库格式** | **SQLite**（`*.sqlite`，标准库 `sqlite3`；`meta` + `nodes(id,version,category,label,json,sha256)` + `assets(BLOB)`）—— 2026-10-19 用户拍板「直接使用 SQLite」 |
| 二进制 | 定义格式不二进制（JSON + `.py`）；**发布包要加密** → `.nlib` 容器（AES-256-GCM，payload = 整份 SQLite 字节，见 P4c/P4d），开发态永远明文 |
| 一文件几节点 | **可装多个**（`nodes` 多行；作者源是 `nodes` 数组）；单节点简写也认，解析成同一结构 |
| 原子性 | **整库/整文件原子**（一条坏 = 整份不进，点名"文件 : 条目 : 原因"） |
| 冲突 | 包内按文件名排序 → 叠数据目录（**本地赢**）；**同层同名报错** |
| 加密 / HMAC | **发布包加密要做**（`.nlib`，key 走 license/凭证存储，**不硬编码**）；**不用 HMAC**；完整性/署名 = 逐行 sha256 + Ed25519（P6） |
| 加密实现落点 | 新包 **`l_nodelib`**（`cryptography` + `l_node_engine`）；**引擎仍是纯标准库**，发现 `.nlib` 才懒加载它，没装就明确报错 |
| payload | v1 = **sqlite**（整份 SQLite 库字节）；头写 `"payload":"sqlite"` + `payload_sha256`；容器/加密/签名/查看器不随 payload 变（见 P4d） |
| **节点版本** | **节点自带 `version`**；库内 `(id, version)` 多行并列；注册表 `id` 只挂 `latest`，旧版按需解析；图侧 `nodes[id].asset_version` **逐节点切换**（缺省/缺字段 = 跟随 latest，未知版本**报错不回落**），见 P2 |


仍然要定的：

1. **P0 的字段名**：`min/max/step/unit/group/order` 还是照 ComfyUI 的 `tooltip/range` 口径折中？
2. **P1 的 DSL 边界**：只允许"等值 / 属于集合 / any-all"，还是要 `数值比较`（`>=` 之类）？
3. **P4 的落点**：文件放 `<包>/nodes/` 还是只放 `<包数据目录>`（后者能改但不可分发）？
4. **P4 库文件的命名空间**：`id` 就是全局注册名（今天的 `register_node` 平表），还是允许 `library:id`
   带前缀（可共存两份同名节点，但图里的键要跟着变）？

（原来的"版本粒度"与"库内多版本"两条已在 2026-10-19 拍定：**节点自带版本 + 库内并列**，见上表与 P2。）

## 九、建议顺序与工作量

| 顺序 | 阶段 | 工作量 | 说明 |
|---|---|---|---|
| 1 | P0 | 小（半天） | 纯声明面，前后端各一处；风险最低，先把"参数像样"做掉 |
| 2 | P1 | 小~中 | 引擎求值 + 前端显隐；两后端要一起 |
| 3 | P5 | 中 | 只动编辑器（引擎零改动），用户体感最直接 |
| 4 | P2 | 中 | 牵涉图格式（`asset_version`）与 `/validate` |
| 5 | P4a | 中~大 | 新 `NODE_SOURCE="file"` + **SQLite 库**（schema + 读取 + 逐行原子 / 冲突规则）+ `nodelib_pack`（json→sqlite） |
| 6 | P6 | 小 | 与 P4 同批：签名校验 + 三档状态 + `nodes_require_signed` |
| 7 | P4c | 中 | 发布包加密：**新包 `l_nodelib`**（cryptography）+ `.nlib` 容器 + `nodelib_pack`（明文→密文）+ **`nodelib_inspect` 查看器**（头 / 有 key 看内容 / `--verify` / `--diff`，只读不注册）+ 解密入口（L2 key = `license.json`） |
| 8 | P3 | 大 | 事件语义最容易出"隐式副作用"，放最后 |
| 9 | P4b | 大 | `pygraph` 引用 + 动态端口 |

> 每一阶段的"完成"一律按 §六 的判据跑一遍再打勾；**不许只 build 就说做完了**。
