# l_WChat 视频剪辑 · 对标剪映：差距盘点 / 分期计划 / 验收标准

> 对象：`l_WChat` 亲子相册里的视频剪辑（独立路由 `/video-editor`，编辑器本体 `static/album_editor.js`）。
> 定位不是"复刻剪映"，而是**家庭场景够用**：手机上一屏能编、成片直进版本库、手机端能看。
> 每次改动后按本文「验收标准」逐条自测，能自动化的写进验收命令。

---

## 一、现状（已经具备）

| 能力 | 说明 |
|---|---|
| 主轨叙事 | 照片 / 视频 / 文字卡首尾相接；空隙（自由挪时间）、换序、转场（硬切/淡入） |
| 多轨 | 主轨 + **可任意新增**：视频轨 / 贴纸轨 / 字幕轨 / 音频轨；每条轨可整轨 透明度 / 混合模式（16 种）/ 滤镜（11 预设）/ 蒙版（圆/圆角矩形/线性/暗角+羽化）/ 音量 / 静音 |
| 同类型互拖 | 轨内素材可在**同类型轨之间**互拖（视频轨↔视频轨、贴纸↔贴纸、字幕↔字幕、音频↔音频），拖回主轨/贴纸/配乐轨也支持；跨类型给明确提示 |
| 素材来源 | 相册 / 本机文件（自动上传，重开还在）/ 电脑本机路径引用 / AI 图片 · AI 视频 · AI 配音（经 l_model_hub） |
| 音频 | 配乐（本机文件 / 混音台 / 免费曲库 4 源搜索）/ 每段音量·淡入淡出·源内偏移 / 整轨音量·静音 / **视频自带声音可单独调音量、可一键分离到音频轨** |
| 卡点 | 拍点分析（Web Worker 后台跑，带进度与预计剩余，屏幕底部显示）/ 拍点线只读且**跟随所属音频移动** / 吸附拍点 / 卡点自动模板 |
| 画面 | Ken Burns 关键帧（两点式）+ **动画曲线编辑器**（三次贝塞尔，把手可拖，含回弹过冲）/ 亮度·对比·饱和 / 变速 / 倒放 / 模糊背景（contain + 模糊底）/ 比例 16:9·9:16·1:1·4:5 / 720p·1080p |
| 片段精修 | **源时间轴裁剪**（20s 素材挑 12–17s，带帧预览/试看）/ 时间轴拖右边缘改时长 / 左边缘改源内起点 |
| 工程与产出 | 本地自动存（双备份槽）/ 撤销重做 80 步 / 云草稿（跨设备）/ 模板（结构模板 + 现成工程 + 卡点自动）/ 预览=成片同一份渲染 / WebCodecs 编码 MP4 → 直传版本库 → 手机可看 |
| 交互 | 手机上默认全屏（沉浸 + 首次触摸升级真全屏）/ 按钮太多时收进 **⋮**（主操作常驻）/ 全按钮悬停提示 / 时间轴左侧图标轨标 / 双击片段或标尺缩放 |
| App | Android 包内置（安全上下文 https + 自签 CA）/ 启动页 `/l_wchat` / 内置「更新软件」 |

---

## 二、距离剪映的差距（分模块）

图例：✅ 已有 ｜ 🟡 部分 ｜ ❌ 缺

| 模块 | 剪映能力 | 我们 | 差距要点 |
|---|---|---|---|
| 时间轴 | 磁吸 + 自由双模、多轨混排、批量选择、片段组、吸附线提示 | 🟡 | 无框选/多选、无片段分组、吸附无视觉提示 |
| 转场/特效 | 100+ 转场、粒子/光效/故障等特效 | ❌ | 只有硬切/淡入；无特效层 |
| 关键帧 | 任意属性、任意数量、曲线编辑、批量粘贴 | 🟡 | 两点式（起→终）+ 曲线；无多点曲线、无粘贴 |
| 蒙版/合成 | 线性/镜面/圆形/矩形/爱心，可反向、可羽化、可动画 | 🟡 | 4 种形状 + 羽化，不可反向/不可动画 |
| 调色 | 曲线、色轮、HSL、LUT、示波器 | 🟡 | 亮度/对比/饱和 + 11 滤镜预设；无曲线/HSL/LUT |
| 音频 | 降噪、变声、闪避（音乐让位人声）、节拍识别、波形编辑 | 🟡 | 音量/淡入淡出/分离/拍点；无降噪、无闪避、无波形手编 |
| 字幕 | 自动识别（ASR）、花字、字体库、字幕样式模板、逐字动画 | 🟡 | 有字幕轨 + AI 配音（TTS）+ MiniMax 试听；无 ASR 自动字幕、无花字/字体库 |
| 语音 | 文本朗读（多音色）、音频转文本 | 🟡 | TTS 已有；ASR 缺（依赖模型/服务） |
| 画中画 | 多视频层、混合模式、抠像（人体/色度） | 🟡 | 多视频层 + 混合模式已有；抠像缺（Chroma Key 可浏览器本地做） |
| 导出 | 4K/60fps、码率可控、硬件加速、后台导出、模板包 | 🟡 | 720p/1080p；后台导出＝浏览器必须有前台页面；无 4K/硬件加速开关 |
| 协作/云 | 云草稿、模板社区、团队协作 | 🟡 | 云草稿 + 版本库已有；无模板社区/协作 |
| 智能 | 智能踩点、智能字幕、智能抠像、图文成片、AI 剪辑 | 🟡 | AI 图片/视频/配音 + 拍点已有；图文成片/智能字幕/智能抠像缺 |

**一句话结论**：**"家庭记录片"这条线已经够用**（叙事 + 多轨 + 卡点 + 蒙版 + 音频分离 + 版本库分发），
距离剪映的差距集中在四块：**① 特效/转场库 ② 智能（ASR 自动字幕、智能抠像、图文成片）③ 调色深度 ④ 时间轴交互（多选/分组/吸附提示）**。

---

## 三、分期计划

### P0（每项 0.5–1 天，价值/成本比最高，建议先做）

| # | 内容 | 验收标准（可测） |
|---|---|---|
| P0-1 | **时间轴吸附视觉提示**：拖动片段靠近拍点/邻段/播放头时显示竖线 + 片段边缘"咔"反馈 | 拖到距拍点 <8px 时出现高亮吸附线；松手位置与拍点误差 ≤40ms；`吸附拍点` 关掉后不再出现 |
| P0-2 | **多选与批量操作**：框选（空白处拖框）/ Shift 点选；批量改时长、批量删除、批量套动画 | 框选 3 个片段后检查器显示"已选 3"；批量改时长后 3 个片段时长一致（±0.05s）；批量删除后可 ↶ 撤销 |
| P0-3 | **片段分组**：把多个片段合成一组，整组移动/复制 | 成组后拖动一个=整组平移；组内片段相对时间不变（偏差 ≤0.02s）；可解组 |
| P0-4 | **色度抠像（绿幕）**：每个视频/图片层加"抠像"开关（色相 + 容差 + 边缘羽化） | 纯绿背景素材开启后，画面中绿色区域变为透明（采样点 alpha→0）；容差滑杆能改变保留范围；导出与预览一致 |
| P0-5 | **蒙版反向 + 蒙版动画**：蒙版加"反向"开关；形状/位置可在关键帧上动 | 反向开关生效（保留区与未反向互补，采样点像素互换）；蒙版位置在片段起终两端不同（导出抽帧可见差异） |

### P1（每项 1–3 天）

| # | 内容 | 验收标准 |
|---|---|---|
| P1-1 | **转场库**：至少 8 种（推/滑/擦除/缩放/黑场/白闪/叠化/旋转） | 相邻片段切换后，中间帧同时含两段内容；每种转场可单独设时长；导出无黑帧 |
| P1-2 | **特效层**：光斑/星光/雨雪/故障 4 类贴图特效，可调强度/速度 | 特效叠加后画面出现预期元素；强度 0 时与未加完全一致（像素差 <2/255） |
| P1-3 | **ASR 自动字幕**：从视频/音频提取文本 → 生成字幕轨（走 l_model_hub） | 选一段 10s 中文语音，生成 ≥1 条字幕，文本与实听一致率 ≥80%；时间点误差 ≤1s；可整条编辑 |
| P1-4 | **调色增强**：HSL 三通道 + 曲线（至少 RGB 曲线） | 拖动曲线控制点画面随之变化；HSL 单通道调整可只影响该色相范围 |
| P1-5 | **音频闪避**：人声/配乐并存时自动压低配乐 | 开启后，有语音处配乐电平下降 ≥6dB；无语音处恢复 |
| P1-6 | **图文成片**：贴一段文案 → 自动配图 + TTS + 字幕生成完整工程 | 输入 100 字，产出：≥4 个画面片段 + 配音 + 字幕；整片时长与配音长度差 ≤1s；可继续手工编辑 |
| P1-7 | **后台导出队列**：多个成片排队导出，进度在底部作业条 | 连续点两次生成，队列显示"1/2、2/2"；两个文件都进版本库且可播放 |

### P2（长期/按需）

| # | 内容 | 验收标准 |
|---|---|---|
| P2-1 | 模板社区（上传/下载模板，带素材包） | 别人账号下载模板后可一键用我的素材重建，导出成功 |
| P2-2 | 4K/60fps 与码率档位（浏览器能力允许时） | 导出 4K 成片，分辨率与码率符合所选档位；低端机自动禁用并给提示 |
| P2-3 | 智能抠像（人体/自动主体） | 单人素材抠像后边缘无明显绿边；动作中不出现大面积漏抠 |
| P2-4 | 音效库（转场音效/环境音）接入免费源 | 搜索"whoosh"能出结果并一键加入音频轨 |

---

## 四、验收标准的通用要求（每个功能都要过）

1. **预览=成片**：同一时间点，舞台像素与导出帧像素平均差 ≤ 3/255。
2. **可撤销**：破坏性操作（删除/移除轨/分离音频）后 ↶ 能恢复原状。
3. **可持久化**：刷新页面后工程结构、轨道属性、曲线、蒙版、音量全部还原（localStorage 主槽或云草稿）。
4. **手机可用**：390×844 下不出现横向滚动条；按钮放不下时进 ⋮；关键操作点击区 ≥32px。
5. **不卡**：拖动过程只改当前元素样式（不整树重建）；10 秒内不出现 >200ms 的卡顿（Performance 面板抽检）。
6. **失败要说清**：素材读不到/无音轨/编码不支持时，状态栏给中文原因，且不破坏当前工程。
7. **回滚**：每期改动独立可回滚（部署脚本按文件清单推，出问题重推上一版即可）。

---

## 五、风险与依赖

| 风险 | 影响 | 应对 |
|---|---|---|
| 火山等外部模型 key 失效 | ASR / AI 视频不可用 | 功能入口保留 + 明确提示；ASR 可换本机 whisper 兜底 |
| 浏览器 WebCodecs 受限（部分 WebView） | 不能出片 | 已有检测与提示；必要时回退服务端 ffmpeg |
| 手机内存（多轨 4K 素材） | 崩溃 | 素材统一抽稀到编码分辨率再合成；限制同时解码轨数 |
| 版本库/网盘配额 | 上传失败 | 失败后重试队列 + 本地保留成片 |

---

## 六、当前迭代（本次）

### 20260930（P0 / P1 / P2 一次性落地）

**P0（时间轴交互与渲染管线）**

- ✅ 吸附视觉提示：吸附目标从"只有拍点"扩到 **拍点 / 邻段边缘 / 播放头 / 0**，容差 10px，命中时画青/黄/红提示线 + 轻微震动
- ✅ 多选与批量操作：`Shift` 加选、空白处框选；批量改时长 / 套动画 / 套转场 / 删除 / 成组 / 解组 / 复制组
- ✅ 片段分组：整组平移（只改组内第一段 `gap`，组内相对时间零偏差），组色条
- ✅ 色度抠像（绿幕）：色相/容差/柔边 + 画面吸管取色
- ✅ 蒙版反向 + 蒙版动画（`maskInvert` / `maskAnim`，配合位移缩放的 `mf={x,y,scale}`）

**P1（库 / 智能 / 调色 / 音频 / 导出）**

- ✅ 转场库 10 种：cut / fade / dissolve / push / slide / wipe / zoom / black / white / rotate（可调时长）
- ✅ 特效层：新增 `fx` 轨道类型，5 类程序化粒子（光斑/星光/雨/雪/故障），确定性伪随机（预览=导出），强度 0 像素零变化，画在最上层
- ✅ 调色增强：RGB 主曲线 4 控制点（单调三次 Hermite LUT）+ HSL 8 色相区（色相/饱和/明度），曲线可视化编辑器；无调色时不走像素路径
- ✅ ASR 自动字幕：新增 `services/asr.py`（l_model_hub ASR → 本机 whisper 兜底）+ `POST /api/album/ai/asr` + 工具栏「🎤 自动字幕」
- ✅ 音频闪避：`mixAudio()` 增 `opts.duck`，人声侧链（视频原声段 + 有字字幕段 + 配音段）平滑压低配乐（attack/release），头部开关（导出时生效）
- ✅ 后台导出队列：连点生成 → 串行排队，底部作业条显示 `1/2、2/2`，按钮不再长时间禁用
- ✅ 图文成片：文案按标点切句（不足 4 段按长度均分）→ 每句 AI 配图 + AI 配音 + 字幕 → 组装工程（可撤销）

**P2**

- ✅ 模板社区：`GET /api/album/video-templates`（列表，轻量）/ `GET .../{id}`（全文，套用时拉）/ `POST`（发布）/ `DELETE .../{id}`（存 `USER_DATA_DIR/video_templates/*.json`）+ 模板面板「☁️ 社区模板」区（发布 / 套用 / 删除）
- ✅ 4K / 60fps / 码率档：`presetFor()` 增 2160p，新增 `P.fps∈{30,60}`、`P.rate∈{auto,high,mid,low}`；`encodeMovie()` 的帧率与码率真正下发（原先 `preset.bitrate` 只用于体积预检）；`VideoEncoder.isConfigSupported` 探测，低端机自动禁用并提示
- ✅ 音效库：配乐面板加 8 个音效预设关键词按钮（whoosh / swoosh / 环境音 / 掌声 / 笑声 / UI 点击 / 快门 / 铃声），复用免费曲库 4 源
- ⏸ 智能抠像（人体分割）**暂缓**：需引入 MediaPipe 模型与 wasm（体积 + 许可待确认），建议先用色度抠像替代

**收尾**

- ✅ `window.AlbumEditor.selftest()`：返回断言数组，覆盖吸附容差 / 分组相对时间 / 蒙版反向互补 / 特效强度 0 像素差 / 特效确定性 / 曲线 LUT 单调 / 转场中间帧双内容 / 队列计数 / 切句 / 4K 与码率档
- ✅ 空时间轴也能预览各层：去掉 `drawStage()` 在"无片段"时的提前 return（原先只画「先加片段」占位就返回），现在占位文字之下仍会渲染叠加轨/贴纸/字幕/特效层 —— 只放一条特效轨时即可看到粒子
- ✅ 时间轴缩略图彻底修好（原问题："有的素材无法在时间线查看缩略图"）。实测三个根因：① CDN `thumb_url` 带 `expires=8h`，存进工程后过期；② `/api/album/thumb/` 是依赖进程内缓存的尽力代理，实测 **404**；③ 老工程里的视频片段**压根没存 `thumb`**。修法：`thumbCands()` 逐级候选 **CDN → 缩略图代理 → 原图 `/api/album/file/` → 向相册要一次新鲜链接**（整表 5 分钟缓存 + 并发合并），每级 6 秒超时（CDN 有时既不 load 也不 error 会挂住链条），视频再加**从文件抽一帧**的终极兜底；叠加轨/贴纸轨同时修掉漏加反代前缀 `WU()` 的老问题
- ✅ 素材本身有问题时不再"静默空卡"，而是在时间轴上明确标记：卡片打红斜纹 + ⚠ + 悬停说明（"无法解码 code 4，文件可能只同步了一半" / "素材取不到，可能已删除或未同步"）——区分"功能坏了"和"素材坏了"
- ✅ 新增「🧹 清理损坏素材」一键清理：逐个**真实探测**时间轴上的素材（图片能否解码 / 视频音频能否读出元数据，各带 6~9s 超时，并发跑），把损坏的（如只同步了一半的 4KB 视频）和取不到的（已从相册删除/未同步）一次性移除，列出"哪个容器 + 素材名 + 原因"确认后执行，**可 ↶ 一次撤销**；覆盖主轨 / 贴纸·画中画 / 叠加轨 / 配乐轨四类容器，文字卡与字幕不算素材
  - 两道安全阀防误删：① 相册列表拿不到（服务/网络异常）→ 直接中止不删；② **全部素材都探测失败**时判定为整体异常 → 中止（绝不把工程删空）
- ✅ 修手机端「参数面板展开后划不到顶部」（用户反馈）。实测两个堵点：① `.ave-wrap` 是 `overflow:auto` 且 `overscroll-behavior:contain`——它自己并不需要滚动，却把滚动链**掐断**在它这里，页面根本收不到手势；② 时间轴区域 `.ave-rows`/标尺/轨道名是 `touch-action:pan-x`、卡片是 `none`，手指落在时间轴上竖划一律不动（面板一展开，屏幕绝大部分正是时间轴+面板）。修法：手机端把 `.ave-wrap` 的 `overscroll-behavior` 放回 `auto`，时间轴整片（rows / 标尺 / 轨道名 / 各类卡片 / 贴纸轨项）统一 `touch-action:pan-y` —— **竖向还给页面滚动，横向仍由 JS 处理**；空白处触摸改为自定义横向平移时间轴（`drag.kind="pan"`，实测 `scrollLeft 0→100`），鼠标/触控笔原样保留框选；触摸竖直拖不再当"换轨"落点（否则想滚页面会误把片段挪到别的轨道）

  > 2026-10-01 再调整：**卡片**由 `pan-y` 改回 `none`（长按 0.3s 抓起后要能接着竖拖换轨，`pan-y` 会中途
  > 被浏览器接管发 `pointercancel`）；卡片起手的竖划改由 JS 代滚（`pagescroll` + 简易惯性），
  > 空白区 / 标尺 / 轨道名仍是 `pan-y`。原「长按弹目标轨列表」方案已删除。详见测试报告 §9.1。
- 版本：`album_video.js?v=20260930a`、`album_editor.js?v=20260930n`
- 验收：浏览器内 `await AlbumEditor.selftest()` → **22/22 通过**（吸附 3 项 / 分组 1 / 蒙版反向 1 / 特效 3 / 调色 3 / 转场 2 / 导出队列 1 / 图文成片 2 / 4K 与码率档 2 / 缩略图候选链 3 / 清理素材收集 1）
- 📱 移动端验收基准视口已固化到项目根 `AGENTS.md`：真机 **1080×1920 / DPR 3 → 测 360×640**（Playwright 预设 `Galaxy S5`，自带 mobile+touch+3x）；iPhone 13 的 390×664 是 CSS 视口（物理 1170×1992，不低），但比例 19.5:9 比真机 16:9 宽，窄屏问题只在 360×640 测得出来。移动端改动按 `AGENTS.md` 的清单跑
  - 另做真浏览器端到端：空工程 + 只放一条「故障 90%」特效轨，画布像素从 `sum=11,777,930 / nz=7,456`（仅占位字）变为 `sum=288,075,732 / nz=218,190`，特效确实渲染；控制台无 error
  - 另做真浏览器端到端（**用相册全部 20 个真实素材 + 2 个不存在的素材**建工程，图片故意给"已过期的 CDN 链接"、视频给"老工程那样没有 thumb"）：22 张卡片 **22/22 都有明确结果** —— 原图兜底 14 / 新鲜 CDN 3 / 视频抽帧 2 / 标记素材问题 3，**零张静默空卡**；其中 `…_132550.mp4` 实测为**损坏文件**（4096 字节、无 `ftyp` box、浏览器 `DEMUXER_ERROR_COULD_NOT_OPEN`），故按"素材坏了"标记而非缩略图 bug

---

## 七、实现方案（P0 / P1 / P2 详细设计）

> 本节是"先出方案再改"的落地设计：逐项给出**数据模型 / 改动点（文件·函数·行号）/ 影响文件 / 验收钩子**。
> 代码基线：`static/album_editor.js`（约 4985 行）、`static/album_video.js`、`api/routes.py`、`services/l_model_hub.py`。
> 全部新增字段都同步进 `projectJson()` / `parseProjectJson()`，保证老工程能读、能持久化。

### 7.0 通用约定（每项都适用）

| 事项 | 做法 |
|---|---|
| 版本号 | 每次改动同步 `templates/video_editor.html` 的三个 `?v=` 与 `album_editor.js` 头注释；本次建议 `20260930a`（注：当前 HTML 里是 `20260929t`，文档旧值 `…s` 已过期） |
| 可撤销 | 破坏性操作（批量删除/成组/解组/套模板/图文成片）**前后显式 `pushHistory()`**，不依赖 600ms 防抖，保证"一步一撤" |
| 可持久化 | 新增字段全部经 `projectJson()`（约 L70）写出、`parseProjectJson()`（约 L186）补默认值 |
| 手机可用 | 新控件优先放入 `details` 折叠块或 `⋮`；点击区 ≥32px |
| 不卡 | 拖动只改当前元素样式；像素级处理（抠像/调色/智能抠像）只在开启时对目标层/段生效 |
| 失败要说清 | 所有新流程异常走 `setStatus(msg, true)` 给中文原因，不破坏当前工程 |
| 验收命令 | 新增 `window.AlbumEditor.selftest()`：返回断言数组（`{name, pass, detail}`），控制台可跑；覆盖吸附容差、分组相对时间、蒙版反向互补采样、转场中间帧双内容、特效强度 0 像素差、队列计数 |

### 7.1 P0-1 时间轴吸附视觉提示

- **数据模型**：无新字段。吸附目标扩展为：拍点 `P.beats`、相邻段边缘（前段 `end` / 后段 `start`）、播放头 `playT`、`0`。
- **改动点**（[album_editor.js](file:///d:/TD_Depot/Software/Lugwit_syncPlug/lugwit_insapp/trayapp/rez-package-source/l_WChat/999.0/src/l_WChat/static/album_editor.js)）：
  - 新增 `snapTimeEx(t)` → `{t, hit:{kind:'beat'|'edge'|'head'|'zero', at}}`；容差 `10/pxPerSec`（约 8px 起高亮，≤40ms 误差天然满足，因为落点直接取目标值）。旧 [`snapTime()`](file:///d:/TD_Depot/Software/Lugwit_syncPlug/lugwit_insapp/trayapp/rez-package-source/l_WChat/999.0/src/l_WChat/static/album_editor.js#L286-L295) 改为 `return snapTimeEx(t).t`（兼容 L1884/L2954/L3822… 全部调用点）。
  - [`bindTimeline()`](file:///d:/TD_Depot/Software/Lugwit_syncPlug/lugwit_insapp/trayapp/rez-package-source/l_WChat/999.0/src/l_WChat/static/album_editor.js#L3699-L3916) `pointermove`：`move/dur/trimin/ov-move/lay-move/sub-move/audio-move` 各分支算完吸附后调用 `showSnapLine(hit)`（写入 `drag.snapHit`）；`pointerup` 调用 `hideSnapLine()`。
  - [`renderTimeline()`](file:///d:/TD_Depot/Software/Lugwit_syncPlug/lugwit_insapp/trayapp/rez-package-source/l_WChat/999.0/src/l_WChat/static/album_editor.js#L1589) 末尾创建 `d.snapline`（`.ave-snapline`，`position:absolute;left:Xpx;top:0;bottom:0`），拖动期间不重建时间轴，元素可稳定复用。
  - CSS 新增 `.ave-snapline`（高亮竖线，按 `hit.kind` 换色：拍点=青、边缘=黄、播放头=红）。
  - 触感：命中时 `navigator.vibrate?.(8)`（手机"咔"感）。
- **关闭开关**：`P.snap=false` → `snapTimeEx` 返回 `hit:null` → 不显示线、不吸附。
- **影响文件**：`album_editor.js`（唯一）。
- **验收钩子**：`selftest()` 断言 `snapTimeEx(beat±0.05).hit.kind==='beat'`；视觉项人工确认。

### 7.2 P0-2 多选与批量操作

- **数据模型**：模块级新增 `var selSet = []`（元素同 `sel` 结构 `{type,id,layer?}`）；`sel` 保留为"主选中"（兼容现有代码与检查器）。
- **改动点**：
  - [`bindTimeline()`](file:///d:/TD_Depot/Software/Lugwit_syncPlug/lugwit_insapp/trayapp/rez-package-source/l_WChat/999.0/src/l_WChat/static/album_editor.js#L3704-L3784) `pointerdown`：点卡片时 `ev.shiftKey` → 切换 `selSet` 成员；普通点 → `selSet=[{…}]`。
  - 空白处（非卡片/标尺/空隙）：`drag={kind:"box", x0,y0}`；`pointermove` 画 `.ave-selbox`；`pointerup` 用矩形与 `.ave-clip/.ave-aclip/.ave-sub/.ave-ov/.ave-layit` 的交集填 `selSet`。
  - [`renderTimeline()`](file:///d:/TD_Depot/Software/Lugwit_syncPlug/lugwit_insapp/trayapp/rez-package-source/l_WChat/999.0/src/l_WChat/static/album_editor.js#L1589) 的 `.sel` 判定：`selSet` 含该 id 即加 `.sel`（L1700/L1725/L1752/L1767/L1782 五处）。
  - [`renderInspector()`](file:///d:/TD_Depot/Software/Lugwit_syncPlug/lugwit_insapp/trayapp/rez-package-source/l_WChat/999.0/src/l_WChat/static/album_editor.js#L1848)：`selSet.length>1` 时优先渲染"批量面板"（已选 N / 批量改时长 / 批量删除 / 批量套动画 / 批量套转场 / 成组）。
  - 批量改时长：主轨片段写 `dur`（视频写 `trimOut`）、音频/贴纸写 `dur`——统一成同一值 → 一致性误差 0。
  - 批量删除：`pushHistory()` → 从各数组 filter 掉 → `saveProject()` → `renderAll()`。
- **影响文件**：`album_editor.js`；CSS `.ave-selbox`。
- **验收钩子**：`selftest()` 断言"批量改时长后 3 段 `clipDur` 一致（±0.05s）"。

### 7.3 P0-3 片段分组

- **数据模型**：clip 新增 `grp`（组 id，字符串；空=未分组）。组只作用于**主轨**（跨轨拖动时会提示并跳过整组）。
- **改动点**：
  - `projectJson()`（L79-90）与 `parseProjectJson()`（L191/L204）补 `grp`。
  - `renderInspector()` 批量面板加"成组/解组/复制组"。
  - **整组平移**（关键）：`pointerup` 的 `move` 分支命中组内片段时，把 `dt` 只加到**组内第一段的 `gap`**（其余段 `gap` 不变）→ 相对时间零偏差；同时跳过 `applyClipDrag` 的 reorder 分支以免打乱组内顺序。
  - `renderTimeline()` 给组内卡片加 `.grp` 类 + 组色条。
- **影响文件**：`album_editor.js`；CSS `.grp`。
- **验收钩子**：`selftest()` 断言"整组平移后组内相对时间偏差 ≤0.02s"。

### 7.4 P0-4 色度抠像（绿幕）

- **数据模型**：层 `L.key = {on:false, hue:120, tol:25, soft:0.15}`（`hue` 目标色相角度、`tol` 容差角度、`soft` 边缘过渡）。
- **改动点**：
  - [renderFrame](file:///d:/TD_Depot/Software/Lugwit_syncPlug/lugwit_insapp/trayapp/rez-package-source/l_WChat/999.0/src/l_WChat/static/album_editor.js#L598-L636) 叠加轨分支：当 `L.key.on` 时，素材先画到离屏 canvas → `chromaKeyCanvas()` 逐像素转 HSV，把靠近 `hue`（且饱和度够）的像素 alpha 归零、`soft` 区间做 alpha 渐变 → 再 `drawImage` 到 `mctx`。
  - 新函数 `chromaKeyCanvas(cv, w, h, key)`（复用一块离屏画布；视频逐帧无缓存）。
  - [`openLayerDlg()`](file:///d:/TD_Depot/Software/Lugwit_syncPlug/lugwit_insapp/trayapp/rez-package-source/l_WChat/999.0/src/l_WChat/static/album_editor.js#L3325) 视频/贴纸层加"抠像"开关 + 色相/容差/羽化滑杆 + **"吸取背景色"**（点舞台取像素写回 `hue`）。
  - `newLayer()`（L1326）补 `key` 默认值。
- **影响文件**：`album_editor.js`（唯一）。
- **验收钩子**：`selftest()` 用纯绿测试图断言"采样点 alpha→0"；容差变化改变保留范围；导出=预览（同 `renderFrame`）。

### 7.5 P0-5 蒙版反向 + 蒙版动画

- **数据模型**：层新增 `L.maskInvert:false`、`L.maskAnim:{s:{x,y,scale}, e:{x,y,scale}, cb?}`（两点式，与片段 `anim` 同构）。
- **改动点**：
  - [`applyLayerMask()`](file:///d:/TD_Depot/Software/Lugwit_syncPlug/lugwit_insapp/trayapp/rez-package-source/l_WChat/999.0/src/l_WChat/static/album_editor.js#L1368-L1399) 重写：几何参数从写死的"居中/固定半径"改为接受 `mf={x,y,scale}`（归一化偏心 + 尺寸系数）；
    - 正向：现有 `destination-in` 抠形；
    - 反向：先 `fillRect` 全白铺满，再 `globalCompositeOperation='destination-out'` 画同形状 → 保留形状外（与未反向互补）。
  - `renderFrame` 计算 `p`（图层激活区间内进度）→ `mf = lerp(maskAnim.s, maskAnim.e, easeP)`（复用现有 `cubicY/easeP`），传给 `applyLayerMask`。
  - [`openLayerDlg()`](file:///d:/TD_Depot/Software/Lugwit_syncPlug/lugwit_insapp/trayapp/rez-package-source/l_WChat/999.0/src/l_WChat/static/album_editor.js#L3371-L3377) 蒙版区加"反向"复选框 + "蒙版动画（起/终 x,y,缩放 + 曲线预设）"。
  - `newLayer()` 补 `maskInvert/maskAnim` 默认值。
- **影响文件**：`album_editor.js`（唯一）。
- **验收钩子**：`selftest()` 断言"反向蒙版与正向在采样点 alpha 互补"。

### 7.6 P1-1 转场库（≥8 种）

- **数据模型**：`clip.transition` 扩展枚举 `cut|fade|push|slide|wipe|zoom|black|white|dissolve|rotate`；新增 `clip.xfer`（转场时长，默认 0.4s）。
- **改动点**：
  - [renderFrame](file:///d:/TD_Depot/Software/Lugwit_syncPlug/lugwit_insapp/trayapp/rez-package-source/l_WChat/999.0/src/l_WChat/static/album_editor.js#L530-L540) 替换现有"fade 压暗"：在段首 `xfer` 窗口内，先渲染**前段末帧**到主画布，再把当前段渲到离屏 `transCv`，按类型合成（位移/擦除/缩放/纯色/叠化/旋转）。
  - 新函数 `renderTransition(ctx, t, w, h, segA, segB, type, fd)`；`timeline()` 保留 fade 的 overlap 逻辑不动，其它转场用"画中过渡"（不改总时长）。
  - 片段检查器：转场下拉（[现有 `transition` UI](file:///d:/TD_Depot/Software/Lugwit_syncPlug/lugwit_insapp/trayapp/rez-package-source/l_WChat/999.0/src/l_WChat/static/album_editor.js#L1848)）+ 时长滑杆。
- **影响文件**：`album_editor.js`。
- **验收钩子**：`selftest()` 断言"转场中间帧同时包含两段内容（两段特征色都出现）"，并抽帧确认无黑帧。

### 7.7 P1-2 特效层

- **数据模型**：新层类型 `fx`（`LAYER_KIND.fx`，图标 ✨），层属性 `L.fxType ∈ {sparkle,stars,rain,snow,glitch}`、`L.fxStrength(0~1)`、`L.fxSpeed(0.2~3)`。
- **改动点**：
  - `LAYER_KIND`（L1334）增 `fx`；`newLayer()` 补默认；"＋轨道"菜单增"特效轨"。
  - `renderFrame` 增 fx 分支 → `drawFx(ctx, type, w, h, t, strength, speed)`：用"种子 + t"的确定性伪随机程序化绘制粒子（不依赖外部素材）。
  - **强度 0 = 完全不画**（保证与未加时像素差 0 < 2/255）。
  - `openLayerDlg()` 增 fx 控件（类型/强度/速度）。
- **影响文件**：`album_editor.js`。
- **验收钩子**：`selftest()` 断言"强度 0 时前后帧像素差 <2/255"。

### 7.8 P1-3 ASR 自动字幕

- **后端**（新增）：
  - `services/asr.py`：`transcribe(path) -> [{start,dur,text}]`，按优先级探测：① `l_model_hub` 的 ASR（`POST /asr` 或 `/v1/audio/transcriptions`，404/不可用则跳过）→ ② 本机 whisper CLI（`whisper` / `faster-whisper` / `whisper.cpp` main，探测 PATH）→ ③ 都没有则抛明确中文错误。
  - `api/routes.py` 新增 `POST /album/ai/asr`（输入服务端音频/视频 URL 或本地素材名；返回 `[{start,dur,text}]`）。
- **前端**：新按钮"🎤 自动字幕"→ 取选中音频/视频（或整轨）→ 调 `/api/album/ai/asr` → 生成 `P.subs`（或字幕轨 items），每条可继续编辑。
- **依赖/风险**：当前仓库与 `l_model_hub` **均无 ASR**（已确认）；本机需装 whisper 或 hub 提供 ASR，否则入口保留 + 明确提示（与第五节风险表一致）。
- **影响文件**：新增 `services/asr.py`；`api/routes.py`；`album_editor.js`；`config.py`（可选配置项）。
- **验收**：10s 中文语音 → ≥1 条字幕、一致率 ≥80%、时间误差 ≤1s、可整条编辑。

### 7.9 P1-4 调色增强（HSL + RGB 曲线）

- **数据模型**：`clip.grade = { curve:{rgb:[4 控制点], r/g/b?}, hsl:{8 色相区:{h,s,l}} }`。
- **改动点**：
  - `renderFrame` 主轨片段分支：若 `c.grade` 有值 → 片段先画到离屏 → `applyColorGrade(cv, c.grade)`（RGB 曲线 → 256 级 LUT；HSL 按色相区间只作用该色相）→ `drawImage`。
  - 片段检查器增"调色"折叠块：RGB 曲线编辑器（复用 `openCurveEditor` 交互/`easePath` 绘制）+ HSL 8 色环滑杆。
  - 无 `grade` 时不走像素路径（不影响性能）。
- **影响文件**：`album_editor.js`。
- **验收**：拖曲线控制点画面变化；HSL 单通道只影响该色相范围。

### 7.10 P1-5 音频闪避（Ducking）

- **数据模型**：`P.duck = {on:false, level:0.35, attack:0.2, release:0.3}`（`level≈-9dB`）。
- **改动点**：
  - [`mixAudio()`](file:///d:/TD_Depot/Software/Lugwit_syncPlug/lugwit_insapp/trayapp/rez-package-source/l_WChat/999.0/src/l_WChat/static/album_video.js#L649-L685) 增第 4 参 `opts.duck`：先用"人声侧链区间"（视频自带音轨段 + ASR 命中区间；无 ASR 时退化为"非配乐音轨区间"）算出 sidechain 包络，再对**配乐轨**的 `gain` 叠加 `setValueAtTime + linearRampToValueAtTime` 的压低/恢复斜坡。
  - [`generate()`](file:///d:/TD_Depot/Software/Lugwit_syncPlug/lugwit_insapp/trayapp/rez-package-source/l_WChat/999.0/src/l_WChat/static/album_editor.js#L4825) 传入 `duck`；头部新增"音频闪避"开关。
- **影响文件**：`album_video.js`、`album_editor.js`。
- **验收**：开启后语音段配乐电平下降 ≥6dB、无语音处恢复。

### 7.11 P1-6 图文成片

- **流程**（前端 `textToVideoFlow()` + 按钮"📝 图文成片"）：
  1. 文案按标点切句（每句一段，控制段数 ≥4）；
  2. 每句：`POST /api/album/ai/image`（句子作 prompt + 可选风格前缀）→ 配图；`POST /api/album/ai/tts` → 配音；
  3. 组装工程：`P.clips`（照片，`dur=配音时长`）+ `P.subs`（字幕=句子）+ `P.audio`（配音）→ `P.res/ratio` 沿用当前；总时长 ≈ 配音总长（差 ≤1s，因为段长直接取自配音）。
  - 全程复用底部作业条显示进度；完成后可继续手工编辑。
- **后端**：无需新接口（复用 §「AI 服务」既有 `ai/image`、`ai/tts`）。
- **影响文件**：`album_editor.js`。
- **验收**：100 字 → ≥4 片段 + 配音 + 字幕；总时长与配音差 ≤1s；可继续编辑。

### 7.12 P1-7 后台导出队列

- **数据模型**：模块级 `exportQueue=[]`、`exportRunning=false`。
- **改动点**：
  - 把现 [`generate()`](file:///d:/TD_Depot/Software/Lugwit_syncPlug/lugwit_insapp/trayapp/rez-package-source/l_WChat/999.0/src/l_WChat/static/album_editor.js#L4825-L4915) 抽为 `runOneExport(task)`（保留原逻辑），外面加 `enqueueExport()` + `runQueue()` 串行执行。
  - 进度贴到底部作业条（复用 [jobStart/jobProg/jobEnd](file:///d:/TD_Depot/Software/Lugwit_syncPlug/lugwit_insapp/trayapp/rez-package-source/l_WChat/999.0/src/l_WChat/static/album_editor.js#L3559-L3588) 与 `d.jobBar`），并显示 `1/2、2/2`。
  - 每完成一个即 `uploadBlob` 进版本库；按钮不再长时间禁用（排队即可）。
- **影响文件**：`album_editor.js`。
- **验收**：连点两次 → 队列 1/2、2/2；两个文件都进版本库且可播放。

### 7.13 P2-1 模板社区

- **后端**（新增）：`GET/POST/DELETE /api/album/video-templates[/{id}]`，存 `USER_DATA_DIR/video_templates/*.json`（工程 JSON + 元信息 + 可选素材包清单/打包文件）。
- **前端**：模板面板加"社区"页：列表 / 上传当前工程（含素材）/ 下载并套用（素材换成本机或服务端副本，走现有 `applyCloudDraft` 同款路径）。
- **影响文件**：`api/routes.py`、`album_editor.js`。
- **验收**：别人账号下载模板后可一键用我的素材重建，导出成功。

### 7.14 P2-2 4K/60fps 与码率档位

- **数据模型**：`P.res` 增 `2160p`；新增 `P.fps ∈ {30,60}`；新增 `P.rate ∈ {auto,high,mid,low}`。
- **改动点**：
  - [`presetFor()`](file:///d:/TD_Depot/Software/Lugwit_syncPlug/lugwit_insapp/trayapp/rez-package-source/l_WChat/999.0/src/l_WChat/static/album_editor.js#L56-L67) 增 2160p（3840×2160）与码率档位映射；头部下拉增 2160p / 30·60fps / 码率。
  - [`encodeMovie()`](file:///d:/TD_Depot/Software/Lugwit_syncPlug/lugwit_insapp/trayapp/rez-package-source/l_WChat/999.0/src/l_WChat/static/album_video.js#L689-L753)：`FPS` 常量改读 `opts.fps`；码率档给定时用 `{codec:"avc", bitrate:preset.bitrate}` 替代 `Quality`（mediabunny 支持 `bitrate`，见 `validateVideoEncodingConfig`）——当前 `preset.bitrate` 只用于体积预检、并未真正下发，本项顺手修掉。
  - 低端机自动禁用：用 `VideoEncoder.isConfigSupported`（4K/60）探测，不支持则禁用选项 + 中文提示。
- **影响文件**：`album_editor.js`、`album_video.js`、`video_editor.html`（版本号）。
- **验收**：导出 4K 成片分辨率/码率符合所选；低端机禁用并提示。

### 7.15 P2-3 智能抠像（人体/自动主体）

- **方案**：浏览器端人体分割用 **MediaPipe ImageSegmenter（Selfie Segmenter）**，模型与 wasm 本地化到 `static/mediapipe/…`，首次使用时懒加载。
- **改动点**：开启"智能抠像"的层，每帧把素材画到离屏 → `segment()` 得 mask → 作为 alpha 通道（复用蒙版管线）→ 叠回。加载失败/无模型 → 中文提示并建议改用绿幕抠像（P0-4）。
- **依赖/风险**：需引入第三方模型与 wasm 体积（许可与包体需确认）；性能开销大，仅按需开启。
- **影响文件**：`static/` 新增模型与 wasm；`album_editor.js`（分支 + UI）。
- **验收**：单人素材抠像边缘无明显绿边、动作中无大面积漏抠。

### 7.16 P2-4 音效库

- **方案**：复用免费曲库 4 源（`/api/hypnosis/sources/search` + `/download`），加音效预设关键词按钮（whoosh / 转场音效 / 环境音 / 掌声 / 笑声），结果一键 `addAudioClip` 进音频轨。
- **影响文件**：`album_editor.js`（音效面板/快捷入口）；可选 `api/hypnosis_sources.py` 增音效源。
- **验收**：搜 "whoosh" 出结果并一键加入音频轨。

### 7.17 影响文件总览与实施顺序

| 期 | 项 | 主要文件 |
|---|---|---|
| P0 | 吸附提示 / 多选 / 分组 / 抠像 / 蒙版反向+动画 | `album_editor.js` |
| P1 | 转场库 / 特效层 / 调色增强 / 图文成片 / 导出队列 | `album_editor.js` |
| P1 | 音频闪避 | `album_video.js` + `album_editor.js` |
| P1 | ASR 自动字幕 | 新增 `services/asr.py` + `api/routes.py` + `album_editor.js` |
| P2 | 模板社区 | `api/routes.py` + `album_editor.js` |
| P2 | 4K/60fps/码率 | `album_editor.js` + `album_video.js` |
| P2 | 智能抠像 | `static/mediapipe/*` + `album_editor.js` |
| P2 | 音效库 | `album_editor.js`（可选 `api/hypnosis_sources.py`） |

- **实施顺序**：P0-1 → P0-5 → P1-1…P1-7 → P2-1…P2-4；每期结束 bump 版本号 + 更新第六节"当前迭代"。
- **需确认的依赖**：① 本机是否有 whisper（P1-3）；② 是否允许引入 MediaPipe 模型/wasm（P2-3）；③ 模板社区的存储位置与上传权限（P2-1）。
