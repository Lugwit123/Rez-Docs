# l_model_hub 使用指南（模型中心 / OpenAI 兼容网关）

状态：**2026-09-28**。行号会漂，正文一律用**函数名 / 路由**定位（约定见 `../README.md`）。

## 1. 是什么

`l_model_hub` 干两件事：

1. **模型注册中心**：各厂商模型清单（`models.json` + 动态刷新 + 手动补充）、密钥（权威在
   `lugwit_auth` 中心密钥存储）、优先级与档位；
2. **OpenAI 兼容网关**：`POST /v1/chat/completions` 单入口，按档位链**多厂商自动降级**
   （同一厂商先轮 key 再换厂商），支持流式；另有画图 / TTS / 视频等厂商直调接口。
   语音合成除了云端两家（MiniMax / 豆包），还接了**本机 IndexTTS2**（包 `l_indextts2`，
   `:8470`，零样本克隆 + 音色档案）—— 见 `l_indextts2.md`。

它是**服务卡片**里的常驻服务（别名 `l_model_hub_server`，显示名「L Model Hub 模型中心」，端口 `8462`）。

| 属性 | 值 |
|---|---|
| 包名 / 版本 | `l_model_hub` / `999.0` |
| 入口 | `l_model_hub.server:app`（FastAPI） |
| 端口 | `8462`（env `L_SRC_WATCH_PORT` 可改，见 `src_hot_reload` 文档） |
| 运行数据 | 只在 `~/.lugwit/l_model_hub/runtime`（pid/socket，`L_MODEL_HUB_RUNTIME` 可改）；**业务数据全在 auth 中心存储，包内不写文件** |
| 外部前缀 | nginx `/model_hub/` → 剥前缀；靠 `X-Forwarded-Prefix` / `L_MODEL_HUB_ROOT_PATH` 回填 |

## 2. 启停与页面

**一律走 `wuwo svc`**（卡片里的服务不许直启）：

```cmd
wuwo svc list                       :: 找别名 l_model_hub_server
wuwo svc status l_model_hub
wuwo svc restart l_model_hub        :: 改 .py 后（热重载也会自己重启）
wuwo svc log l_model_hub -n 100     :: 排错先看这里
wuwo svc open l_model_hub
```

| 页面 | 路径 | 说明 |
|---|---|---|
| 管理台 | `/admin/<tab>` | 标签：`overview`（厂商状态 · 网关运行 · 调用统计）/ `keys` / `models` / `routing` / **`compare`（多模型并答对比）** / `test`；`/index` 是旧入口（前端改写为 `/admin/overview`）。**数据按标签懒加载**（进哪个标签才拉哪页的接口，见 `ensureTabData`），顶栏「刷新」= 全量重拉 |
| 接入文档 | `/ide-docs` | IDE 插件怎么填 Base URL / API Key（含 401 排错） |
| Swagger | `/docs` | 自实现标题栏 + 相对前缀图标 |

## 3. 接入（IDE 插件 / 脚本）

| 项 | 值 |
|---|---|
| Base URL | 经 nginx：`http://<host>:8080/model_hub/v1`；直连：`http://127.0.0.1:8462/v1` |
| API Key | **接入密钥 `sk-lmh-<24 hex>`**（首选）；也接受 `lugwit_token`（登录 JWT，572 字符，**不推荐**填进插件） |
| 模型名 | 清单里的 `provider/model`，或虚拟 **`auto`**（按档位链自己挑） |

接入密钥（`access_keys.py`）：

```cmd
curl -X POST http://127.0.0.1:8462/v1/access-keys -H "Authorization: Bearer <lugwit_token>" ^
     -H "Content-Type: application/json" -d "{\"name\":\"vscode\",\"note\":\"\",\"expires_days\":365}"
:: → {"key": {"key": "sk-lmh-…"}}  明文只在这一条响应里出现一次
curl http://127.0.0.1:8462/v1/access-keys          :: 列表（不含明文）
curl -X DELETE "http://127.0.0.1:8462/v1/access-keys?id=<id>"   :: 吊销
```

- 存储：**auth 中心存储 prefs**，`scope='model_hub_access_keys'`，**只存 sha256 + 展示前缀**；
  默认 365 天（`expires_days=0` = 不过期），每次校验成功后按 `TOUCH_INTERVAL_S` 节流更新 `last_used`。
- 落点红线见 `../密钥落点与红线.md` §2 第 13 行。

## 4. 统一登录闸门（`gate.py`）

除公开白名单外**每个请求都验登录**。凭据来源优先级（`request_token()`）：

```
Authorization: Bearer <token>            ← 首选（模型调用走这条）
  > 插件头 api-key / x-api-key / x-auth-token / x-lugwit-token
  > cookie lugwit_token                  ← 浏览器页面
  > env LUGWIT_ACCESS_TOKEN              ← 本机部署
```

- `sk-lmh-…` 开头的 token 走**接入密钥分支**（`access_keys.verify`，命中即合成
  `{"sub": "access-key:<name>", "typ": "access_key"}`），其余走 `POST /api/v1/auth/verify`。
- **公开白名单**（`PUBLIC_PATHS`）：`/healthz`、`/__dev__/src_watch`、`/favicon.*`、`/static`、
  `/docs`、`/redoc`、`/openapi.json`、`/login`、`/v1/keys`（**匿名但 handler 内再核回环**，跨机 403）。
- 页面路径（`PAGE_PATHS` / `PAGE_PREFIXES`）未登录 → 跳登录页；其它路径 → **401**。
- **401 怎么查**：响应日志里有一行 `[gate] 401 … 凭据头=[…] token_len=… sha1=…`（只记头名与长度，
  不记明文）—— 先看这行判断是"没带凭据 / 带了但验不过 / 头名不对"。
  IDE 插件里长 JWT 被截断、或误填成 `Bearer` 之外的头，都在这行露馅。

## 5. 主要接口

### 模型清单

| 路由 | 说明 |
|---|---|
| `GET /models` | 清单（内置 + 自定义 BYO + 手动补充；可用 `provider` / `tag` 过滤） |
| `GET /models/default?provider=&tag=` | 某厂商默认模型 |
| `GET /find?model_id=` | 反查某模型 id 属于哪些厂商 |
| `POST /refresh` | 拉取单厂商模型列表（写入动态缓存） |
| `POST /refresh/all` | 并发刷新全部厂商（含 AK/SK 签名的火山） |
| `GET /v1/models` | OpenAI 形状清单（客户端用这个） |
| `GET/POST /v1/models/manual` | **手动补充模型 id**（刷新拉不到的：套餐档 / 灰度 / 私有接入点）。条目并进清单 → 网关**立刻**按 `provider/model` 路由，不再被"按档位兜底"静默换掉 |
| `GET /v1/models/health` · `POST /v1/models/health` | **模型可用性**：读 / 探一遍逐模型可用性并**存中心存储**（见 §5.2）。`POST` 可带 `provider`（只探一家）或 `only:[厂商/模型]`（只重探几个）；结果**并入**而非覆盖 |

### 网关

| 路由 | 说明 |
|---|---|
| `POST /v1/chat/completions` | 统一入口（含 `stream`）；失败自动降级，错误按 **OpenAI 形状** `{"error":{"message","type","code"},"detail":…}` 返回 |
| `POST /v1/chat/completions/multi` | **多模型并答**：同一输入框一次提问，多家模型**并行**各答一份（每条**钉死**点名的厂商/模型，**不降级**；单家失败只标它自己）。见 §5.1 |
| `POST /v1/chat/completions/multi/stream` | 上面的**流式**版（SSE）：各家边生成边回传，含**思考内容**（`reasoning` 帧）；事件 `start`/`reasoning`/`delta`/`done`/`all_done` 都带 `request` 认领归属。管理台「对比」页用的就是这条 |
| `GET /v1/gateway/status` | 各厂商健康/停用 + 模型名 + `provider_chain` + 档位 + 状态存储自检 |
| `POST /v1/providers/health` | 并发自检全部厂商（按 `checks` 分免费/付费探针；每条带 `tokens`，供自检弹窗显示**这次花了多少 token**） |
| `POST /v1/gateway/check` | 单厂商自检（`client.check_provider`） |
| `POST /v1/gateway/disable` · `/v1/gateway/enable` | 手动停用/启用厂商（持久在 auth 中心存储） |
| `GET /v1/chain?model=` | **降级链预览**（同模型 → 同档 → 相邻档 → 代表模型）；排障先看它，别猜 |
| `GET /v1/gateway/usage?range=` | 调用统计：**厂商维度** `providers` + **模型维度** `models`（键 = `厂商/模型 id`；口径 = 本服务发出的**所有**模型调用），`range=all|today|7d|30d`（见 §7） |
| `GET /v1/gateway/usage/flush` | 手动落一次盘（+推 depot） |

> 网关用 litellm `Router`，进程级 `litellm.drop_params = True`：某厂商不认的参数（如
> `reasoning_effort`）**丢掉这个参数而不是整条链失败** —— 不设时一次 400 会让整个降级链白跑。

### 5.1 多模型并答（`/v1/chat/completions/multi`）

**同一个输入框，多家模型一起答同样的问题**：一次请求并发打多家，并排对比谁快、谁答得怎么样。

```jsonc
POST /v1/chat/completions/multi
{ "prompt": "用一句话介绍你自己",           // 或 messages（优先）
  "models": ["volcengine/deepseek-v4-1-flash",  // ① 精确：厂商/模型 id（= 统计里的键）
             "deepseek-v4-1-flash",             // ② 纯模型 id：按全局优先级取第一个可用厂商
             "auto"],                           // ③ auto：优先级第一个可用厂商的代表模型
  "history": {"volcengine/deepseek-v4-1-flash": [{"role":"user","content":"上一轮问题"},
                                                 {"role":"assistant","content":"它自己的上一轮答案"}]},
  "max_tokens": 512, "temperature": 0.7, "timeout": 120 }
```

- **钉死不降级**：每条都只打点名的那个厂商/模型 —— 一旦走降级链，就变成"同一个模型被问了好几遍"，
  对比失去意义（这是它与 `/v1/chat/completions` 的根本区别）。
- **并行 + 互不影响**：`asyncio.gather`；单条失败只体现在它自己的 `ok:false` + `error`，
  **整体始终 200**（一家挂了不拖垮其余）。
- 返回：`{count, ok_count, results:[{request, provider, model, ok, content, usage, latency_ms, key_index, error}]}`
  （顺序 = 请求顺序）；每条也计入调用统计（同 §7 口径）。
- 上限 `MULTI_MAX_MODELS = 8`；`models` 为空 / 输入为空 → 400。
- **流式 + 思考内容**：管理台走 `/v1/chat/completions/multi/stream`（SSE）—— 各家边生成边回传，
  思考型模型（deepseek / volc）的 `reasoning_content` 作为 `reasoning` 帧单独给出，前端单独一块显示。
  事件：`start`（厂商/模型）→ `reasoning` / `delta`（可交错）→ `done`（`ok`/`usage`/`latency_ms`/`error`）
  → `all_done`（`count`/`ok_count`）→ `data: [DONE]`；每帧带 `request`（= 你传的 `models` 那一项）。
  服务端用 `asyncio.Queue` 把 N 路汇成一条流，并带 `X-Accel-Buffering: no`（不关 nginx 缓冲 SSE 会被攒住）。
  厂商不认 `stream_options` 时**去掉该参数重试同一把 key**，而不是把"能用的模型"记成失败。
  非流式那条接口保留（脚本/一次性调用更方便）。
- **多轮**（`history`）：`{spec: messages}` **逐家**传各自的历史 —— 每个模型带着**自己上一轮的答案**
  继续；共用一份 messages 会把 A 的回答喂给 B，那不是对比。前端「⚖ 对比」页就是这么存的
  （`CMP.threads[模型]`，失败的那家不追加半截答案，免得下轮错位）。

**管理台「⚖ 对比」标签页**（独立标签，不在测试页里）：
- 右上「**组数**」下拉（2–6 组）= 本轮同时问几家，也决定结果区列数（`--cmp-n`，每轮渲染时**冻结**自己的列数，改组数不会把历史轮次重排）；
- 下面**每组一个可搜索下拉**（候选来自清单，**排除了 embedding / rerank / ASR / 图像 / 视频** —— 清单里 `tags` 太粗，230 个都标 `text`，所以按 id 特征再做一层排除；各家默认模型排前面；空位自动补一个**不同厂商**的模型，各组默认不撞车）。**每组各有一个搜索框**：输入即筛（按模型 id / 厂商 id / 厂商显示名，如 `glm`、`volcengine`），`↑↓` 选、`Enter` 确定、`Esc` 关；**失焦时输入框若不是有效模型名就还原成当前已选**（避免"搜到一半点走 = 选择丢了"）。组数与每组选择都记 `localStorage`；
- **输入框固定在视口底部**（`position:fixed` 居中，宽度对齐内容栏；结果区留 104px 底边距）—— 边看结果边追问，不用滚回顶部；`Ctrl/⌘ + Enter` 发送；
- 结果**一轮一块**（`第 N 轮 · 问题` + 该轮各组卡片并排，卡头 `厂商 + 模型 + 延迟 + ↑↓ token`，失败卡红框显上游错误），往下追加成对话；「清空对话」重置各组历史（模型选择保留）。
- **确定不可用的模型不进选择列表**（见 §5.2）：标题右侧写「已隐藏 N 个不可用模型」；被隐藏后原来选中的组会自动换一个**不同厂商**的模型（不会退化成两组同一个模型），失效/重复的选择也会被纠正并回写 `localStorage`。

### 5.2 模型可用性（逐模型探测 + 落库）

**为什么要逐模型探**：厂商健康（`client.check_provider`）只探**一个代表模型** —— 厂商活着不代表某个
模型能用。实测：硅基流动账号余额不足时 `deepseek-ai/DeepSeek-V3.2` 必挂（HTTP 402），而别家模型正常；
欠费 / 未开通 / 无权限都是按**模型 + 账号**定的。

三态（`model_health._classify`），**只有 `unavailable` 参与隐藏**：

| kind | 触发 | 界面行为 |
|---|---|---|
| `ok` | HTTP 200 | 正常列出 |
| `unavailable` | 400 / 401 / 402 / 403 / 404（参数或模型被拒、无授权、欠费、无权限、模型不存在）—— 重试也不会好 | **从「对比」页选择列表隐藏**；模型清单打「不可用」徽章 |
| `transient` | 网络异常 / 429 / 5xx —— 厂商侧一时的事 | 只标状态**不隐藏**（一次抖动就把整库模型藏起来是误伤） |

- 探测点：`POST /v1/models/health`（`model_health.probe`）—— 并发 `workers`（默认 8）逐模型发
  `max_tokens=1`，只探**静态清单里的可聊模型**（动态拉来的不进调用链，不探）；每条也计入调用统计（§7）。
- **落库**：`state_store.doc(scope="model_hub_model_health")` —— auth 中心存储（PG）里的
  `prefs:model_hub_model_health`，本机不落文件；结构 `{models: {"厂商/模型": {...}}, meta: {last_run, updated_at}}`。
  结果**并入**已存结论（这次只探一家时别家的记录要留着），`model_health.reset()` 可清空重来。
- 管理台：「概览 → 🏢 厂商 / 网关状态」标题右侧 **🔄 探测模型可用性**（旁边显示
  `已探 N 个 · 可用 X · 不可用 Y · 时间`）；每家厂商卡上多一个「**可用 N/M**」徽章（没探过显示
  「模型未探」）；「模型」页的模型卡上给不可用的打「不可用」徽章。

### 语音合成（TTS）

| 路由 | 说明 |
|---|---|
| `POST /tts` | 引擎选择式：`engine = minimax \| doubao \| indextts2`，返回 `audio_base64`（**多带一个 `format`**：云端两家 `mp3`，本机 IndexTTS2 是 `wav`）；未知 engine → 400 |
| `POST /v1/audio/speech` | **OpenAI 兼容**：`model=indextts2`（默认）→ 本机服务；`model=<provider>/<model>` → 转发到该厂商（含 BYO）的 `/audio/speech`；未知模型 → 404 并列出可用清单。成功直接回音频字节（`audio/wav`，本机路径带 `X-L-Indextts2-*` 头）|

本机 IndexTTS2 的调用细节（`voice` 是**音色档案 id** 而不是音色名、首次调用要等
30~60s 载模型 → 超时默认 300s）见 `l_indextts2.md`。`/v1/audio/speech` 的
`speed` 与 IndexTTS2 的 `duration_factor` 是倒数关系（`speed=0.5` → 时长 ×2）。

### 密钥与积分

| 路由 | 说明 |
|---|---|
| `GET /keys` · `GET /keys/{provider}` | 掩码摘要 / 存储自检（需登录） |
| `POST /keys` | 写密钥（网页调用方登录态；**多把 key 一行一把**，降级时先轮 key 再换厂商） |
| `GET /v1/keys[/{provider}]` | 取**明文**（**只允许回环**，见 §4 白名单） |
| `GET /v1/credits/quota` · `GET /v1/credits/log` | 剩余积分 / 最近扣费记录（wuzu 个人积分、AIGW 个人积分） |

## 6. 路由：档位与优先级

- **档位**（`levels.py`）：`eco` / `std` / `max`，各自一份候选模型；`GET /v1/gateway/status` 里
  `levels.order` / `labels` / `default`。
- **优先级**（`priority.py`）：厂商顺序，调 `/v1/chain` 能直接看到某请求模型的实际顺序。
- **自定义提供商（BYO）**：管理台「密钥」页添加，与内置厂商并列；`no_temperature` 标记透传给调用方
  （`provider_omits_temperature`）。

### 6.1 管理台「路由」页怎么用（2026-09-28 重排）

两块都是**左窄右宽两栏（左列 sticky）+ 右列可搜可筛**，理由是原来"左列表格 + 右列 45 个模型竖排"
一页 5000px、左边一片空白：

| 功能 | 说明 |
|---|---|
| 全局厂商顺序 | 左列，9 家一间一行，**整行可拖拽排序**（HTML5 DnD，抓手 `⠿` 提示），也可 ↑/↓ 换位（到顶/到底的按钮禁用，点了不算改动） |
| 同模型多厂商顺序 | 右列，一行一个模型 + 各厂商顺序 chips（**chips 也能拖**，只在本组内换位 —— 跨组没有语义）；顶部搜索框按**模型 id 或厂商名**过滤，标题带 `命中/总数` |
| 档位清单 | **不在主界面**：点标题右侧「🎚 档位编辑」在弹窗里改 —— 一行一档（编号 + 显示名 + id + ↑/↓/✕），底部一行加新档（id + 显示名 + ＋加一档）+「保存档位清单 / 取消（只丢清单改动）」 |
| 模型档位 | **卡片 / 列表**下拉切换（默认**卡片**）= 卡片模式下一张卡就是**一家厂商**（grid `minmax(330px,1fr)` 自动排布），卡头 `厂商名 + pid + N 个 + 折叠箭头 + 全设为…`，卡内每个模型一条（名字 + 档位**按钮组**，点一下即换）；**卡片高度固定 250px**（各家模型数差很多，不定高就是参差不齐的一堆长条 —— 定高后卡与卡对齐，模型多于卡高时在**卡内**滚动，折叠态自动高度）。列表模式是"厂商分组条 + 一行一个 + 下拉"，密度高。两种模式共用同一套「只提交改动」判定 |
| 搜索 / 筛选 | 关键词（模型 id 或厂商名）+ 厂商下拉 + **只看已改**（只列被手动设过档的模型） |
| 分组折叠 | **默认展开**（点卡头/组头 ▾ 可收起）；折叠态记 localStorage（键 `l_model_hub.lv.collapsed.v2`，界面偏好不上服务端） |
| 未保存提示 | 两块标题右侧 `● N 处改动未保存` + **右下角常驻保存条**（页面很长，顶栏按钮会滚出视野） |
| 只提交改动 | 保存只 POST 真正改过的那几项（原来 45 个模型 = 45 次 POST）；`↺` 记的是**清覆盖**（发空 `level`），不是再写一条等于默认值的覆盖 |

`GET /v1/levels` 给三个不同的映射，别混：

| 字段 | 含义 | 用途 |
|---|---|---|
| `assignments` | **生效**档位（含运行时覆盖） | 每行的下拉初值 |
| `overrides` | **手动设过**的档位 | 行标黄（"已改"）、「只看已改」筛选 |
| `manifest` | **清单（models.json）默认**，忽略覆盖 | `↺` 回到这个值 |

写入仍走 `POST /v1/levels/order`（档位清单）与 `POST /v1/levels/model`（`level: ""` = 清除覆盖）。

**管理台 `/admin/routing`（2026-09-28 重排）**：优先级那块是「左列 sticky + 右列主区」的两栏（左边**全局厂商顺序**短、跟随滚动）；
档位那块**单栏**（档位清单收进「🎚 档位编辑」弹窗，主界面只留逐模型明细）。

| 交互 | 说明 |
|---|---|
| 优先级 · 全局厂商顺序 | 每行 `编号 + ⠿ + 厂商名`，**拖拽排序**（原生 HTML5 DnD，不引拖拽库）或 ↑/↓ 调整（首末位按钮置灰，点了不算改动） |
| 优先级 · 同模型多厂商 | 一行一个模型（模型 id + 候选厂商 chips，chips 可**组内**拖拽 + ↑/↓），带**搜索**（模型 id 或厂商名），计数 `命中/总数` |
| 档位 · 模型档位 | **卡片 / 列表**两种展示（顶栏下拉切换，默认**卡片**）：**卡片 = 一家厂商一张卡**（卡头 `厂商名 + pid + N 个 + 折叠 + 全设为…`，卡内每个模型一条：名字 + 档位按钮组点一下即换）；列表 = 厂商分组条 + 名字 + 档位下拉 + `↺`，密度高。带**搜索 / 厂商下拉 / 只看已改**三个筛子；分组**默认展开**（手动折叠记 `localStorage`，键 `.lv.collapsed.v2`）、卡片/列表选择也记 `localStorage`。**档位清单本身**（增删 / 调高低）在标题右侧「🎚 档位编辑」弹窗里，不占主界面 |
| 已改标记 | 黄名字 = 与线上值不同；「只看已改」一键筛出自己配过的（上百个模型里找手动档） |
| ↺ 回到清单默认 | 按 `manifest` 显示默认档；`levels.py` 的 `/v1/levels` 现在同时给 `assignments`（生效值）/ `manifest`（清单默认）/ `overrides`（手动设过的），所以「回默认」是**清覆盖**（POST `level=""`），不是再写一条等于默认值的覆盖 |
| 保存 | **只提交改动项**（原来 45 个模型 = 45 次 POST），提交项数写进 toast；顶栏按钮旁有「● N 处改动未保存」，档位页很长 → 另有**右下角常驻保存条** |

数据按标签**懒加载**（`ensureTabData`）：进入 `/admin/routing` 才拉优先级与档位，不再首屏把密钥/模型清单/自检一起拉。

## 7. 调用统计与时间维护（2026-09-28）

计数**不再重启清零**，实现在 `usage_store.py`；**两个维度同时记**：厂商（`providers`）与
**模型**（`models`）。

**口径 = 本服务发出的所有模型调用**（2026-09-28 扩口径，之前只算网关那一条路）：

| 来源 | 记录点 |
|---|---|
| 网关 `/v1/chat/completions` | `gateway._record` |
| 直调厂商的 `/chat` `/chat/reasoning` `/chat/vision` `/chat/fallback` | `client._do_chat*` |
| **厂商自检探针**（`/v1/providers/health`、`/v1/gateway/check`） | `client._selfcheck_once` |
| 画图 / 语音 / 视频（火山 Seedream、ModelScope、MiniMax TTS、豆包 TTS、Seedance、**本机 IndexTTS2**） | `client._count*` —— **按次记、token 为 0**（界面显示 `—`）；本机 TTS 的模型键是 `indextts2/indextts2` |

**模型维度的键 = `厂商/模型 id`**（模型 id 自己可含 `/`，按**首个** `/` 拆）：同一模型换家厂商
**分开**统计 —— 价钱与成功率都不一样，统计里必须看得出是哪家在烧这个模型的钱；
管理台的「**同名模型合并**」勾选框（默认开）把同名模型并成一行、柱子里按厂商分段上色，
是**前端展示选项**，数据仍是分开存的。

| 环节 | 行为 |
|---|---|
| 内存 | 每次调用 `add(provider, ok, pt, ct, model="厂商/模型")`：**厂商 + 模型**各记一份累计，并在 `daily` / `daily_models` 当日桶里各记一份 |
| 落盘 | 后台线程每 `L_MODEL_HUB_USAGE_FLUSH_S`（默认 60s）`flush()`：写 auth 中心存储 `scope='model_hub_usage'`（读回当基线）；中心存储不可用 → 落本地 `~/.lugwit/l_model_hub/usage.json` 兜底 |
| 上云 | 同一次 flush 推一份 `usage.json` 到百度云版本库（ws=`l_model_hub`，库 `/l_model_hub`；工作区缺了会自动登记，见《网盘版本库Depot设计.md》§6.2） |
| 时间维护 | flush 前 `prune()`：丢掉早于保留期的日期桶（**厂商与模型两套桶一起清**；`L_MODEL_HUB_USAGE_KEEP_DAYS`，默认 30，`0`=不清理）；**累计那份不受影响** |
| 范围 | `GET /v1/gateway/usage?range=`：`all`（累计）/`today`/`7d`/`30d` —— 范围按**日期窗口**算（`now - N 天`），不是"最后 N 个桶" |

管理台「📊 调用统计」（概览页）：厂商表（调用/成功/失败/输入输出 token）+ **按模型柱形图**
（行名 = 厂商 + 模型，柱长按调用次数相对最大值归一，ok 蓝 / fail 红两段；勾「同名模型合并」时
同名模型并一行、段色区分厂商；只画前 12 行；悬停有完整明细）+ 范围下拉 + 「立即落盘」+
一行说明（范围 / 累计起点 / 按天保留 N 天 / 最近落盘 / 本次清掉多少旧桶 / 云推送失败原因）。

⚠️ 排障：调用明明发生了但表里没变 → 先 `GET /v1/gateway/usage` 看 `persist.last_error`
（中心存储写不进去时这里会有原因），再 `GET /v1/gateway/usage/flush` 手动落一次并**立刻回读**
中心存储 `prefs:model_hub_usage`（`models` / `daily_models` 是否真写进去 —— 只看到 `providers`
说明是改造前形状的老 payload 被写回来了）。

## 8. 环境变量

| 变量 | 默认 | 说明 |
|---|---|---|
| `L_SRC_WATCH_PORT` | `8462` | 端口（所有服务共用这一个名，见 `../src_hot_reload_源码热重载与主页常驻.md`） |
| `L_MODEL_HUB_KEY_TTL` | `60` | 密钥中心存储读缓存秒数（改密钥后最多滞后这么久） |
| `L_MODEL_HUB_USAGE_FLUSH_S` | `60` | 调用统计落盘/上云周期 |
| `L_MODEL_HUB_USAGE_KEEP_DAYS` | `30` | 按天桶保留天数，`0` = 不清理 |
| `L_MODEL_HUB_DEPOT_URL` / `_WS` / `_LIBRARY` | `http://127.0.0.1:8080/baidu` / `l_model_hub` / `/l_model_hub` | 上云目标（账号取 `LUGWIT_USER`/`LUGWIT_PASSWORD` 或 hub 密钥库 `depot_user`/`depot_password`） |
| `L_MODEL_HUB_CORS` | `*` | 允许的 Origin（逗号分隔） |
| `L_MODEL_HUB_ROOT_PATH` | 空 | 反代没有 `X-Forwarded-Prefix` 时手动指定外部前缀 |
| `L_MODEL_HUB_MAX_CONCURRENCY` | `0`（不限） | 网关并发上限 |
| `L_MODEL_HUB_RUNTIME` | `~/.lugwit/l_model_hub/runtime` | 热重载运行时目录（pid/socket） |
| `L_INDEXTTS2_URL` | `http://127.0.0.1:8470` | 本机 IndexTTS2 服务地址（见 `l_indextts2.md`）|
| `L_INDEXTTS2_TIMEOUT` | `300` | 本机 TTS 超时（秒）：首次要载模型，别用云端那套 30s |
| `LUGWIT_AUTH_USER` / `LUGWIT_AUTH_URL` | — | 访问 auth 中心密钥存储的服务凭据 / 地址 |
| `LUGWIT_ACCESS_TOKEN` | — | 本机免登录调用时的登录态 |

## 9. 排错速查

| 症状 | 先查 |
|---|---|
| 插件 401 | 日志 `[gate] 401 … 凭据头=[…] token_len=…`；Base URL 是否 `/model_hub/v1`；改用 `sk-lmh-…` 接入密钥 |
| 用的模型不是我要的那家 | `GET /v1/chain?model=<id>` 看实际链；模型不在清单 → `POST /v1/models/manual` 手动补 |
| 某厂商 400 但别家能用 | `litellm.drop_params` 是否生效；该模型是否在 `no_temperature` 名单 |
| 调用统计不动 | `GET /v1/gateway/usage` 的 `persist` 段：`last_store`/`last_error`/`last_depot`/`depot_error`；再 `GET /v1/gateway/usage/flush` 手动落一次 |
| depot 推送 400「请指定工作区」 | `usage_store.ensure_workspace()` 没建成（库名/账号不对）；直接调它看返回 |
| 改了模板页面没变 | `l_model_hub` 的 `_render_page()` 是**请求时现读**文件 → 刷新即变；若是别的服务（Jinja 缓存）要 `wuwo svc restart <包>`，详见 `../src_hot_reload_源码热重载与主页常驻.md` |

## 10. 关键源文件

| 文件 | 职责 |
|---|---|
| `server.py` | 全部路由 + 多标签管理台渲染 + 登录闸门/统计线程的启动 |
| `gateway.py` | litellm 网关：降级链、档位、调用统计 `_record`、错误整形 |
| `gate.py` | 统一登录闸门（凭据优先级 / 白名单 / 401 诊断） |
| `keys.py` · `auth_secrets.py` | 厂商密钥（env > 中心存储，TTL 缓存） |
| `access_keys.py` | 接入密钥 `sk-lmh-…`（签发/校验/吊销，只存 sha256） |
| `manual_models.py` | 手动补充模型 id（并进清单，改完自动重建网关 model_list） |
| `usage_store.py` | 调用统计：按天分桶、落中心存储、推 depot、保留期清理 |
| `registry.py` · `providers.py` · `custom_providers.py` | 模型清单解析 / 厂商派生 / BYO 提供商 |
| `levels.py` · `priority.py` | 档位 / 厂商优先级 |
| `credits.py` · `client.py` | 积分扣费 / 直调客户端（含自检探针、画图、TTS；TTS 三家 = MiniMax / 豆包 / **本机 IndexTTS2**，后者 `indextts2_tts`，BYO 转发 `openai_speech`） |
| `templates/admin.html` · `_header.html` · `ide_docs.html` | 管理台（多标签）· 共享标题栏 · 接入文档 |
