# l_WChat 视频剪辑 · 对标剪映：差距盘点 / 分期计划 / 验收标准

> **本文职责** = 目标 / 差距盘点 / 分期计划 / 验收标准 / 逐次改动记录（含"为什么这么改"的现场根因）。
> 测试本身（测什么 / 怎么测 / 通过率 / 复现方式）不在这里，见
> [Rez_pkg/l_WChat_视频剪辑页_非AI功能_自动化测试报告.md](Rez_pkg/l_WChat_视频剪辑页_非AI功能_自动化测试报告.md)。

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
  > 空白区 / 标尺 / 轨道名仍是 `pan-y`。原「长按弹目标轨列表」方案已删除。详见本文「八、验收记录」§8.2。
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

---

## 八、验收记录（逐次 UI 改动流水账要点）

> 本节 = "每次 UI 改动后按本文验收标准自测"的压缩流水账，来源为原《l_WChat 视频剪辑页 · 非 AI 功能 自动化测试报告》§9–§10
> （那段约占该报告后 2/3，已是变更日志而非测试报告，故裁为要点并入本节；测试报告本身只留"测什么 / 怎么测 / 结论"。
> 测试用例矩阵、环境、复现方式见
> [Rez_pkg/l_WChat_视频剪辑页_非AI功能_自动化测试报告.md](Rez_pkg/l_WChat_视频剪辑页_非AI功能_自动化测试报告.md)）。
>
> 每条保留 **现象 / 根因 / 修法** 三要素，宁愿把根因写成一行短句也不丢；实测数字在括注里。

### 8.1 手机端换轨入口 —— 长按弹目标轨列表（2026-09-30，原报告 §9）

- **现象**：屏幕提示写「上下拖=换轨（图片→贴纸/画中画、文字→字幕）」，但代码里触摸的竖向拖被显式排除（`dropHighlight` / `pointerup` 两处 `ev.pointerType !== "touch"`）→ 手机上换轨既没手势、也没替代入口。
- **根因**：卡片的 `touch-action` 是 `pan-y`，该值在 `touchstart` 就锁定；手指第一次竖移即被浏览器当滚动接管并发出 `pointercancel`，拖拽状态当场清空 —— 连非 passive 的 `touchmove` + `preventDefault()` 也拦不住。
- **修法**：改成「按住片段不动约 0.3s → 页面给提示 + 弹出目标轨列表」`openTrackChooser(c)`（文字 → `💬 字幕轨`；图片/视频 → `🖼 贴纸/画中画`，再补 `layerAccepts()` 通过的叠加轨），全程零位移，与竖划滚动零冲突。坑：长按计时器必须挂在**主 `pointerdown` 之后注册的第二个监听器**里（否则读不到刚建好的 `drag`）；弹窗 `z-index:4000` + `scrollIntoView({block:"center"})`（`.ave-picker` 绝对定位在编辑卡内，页面滚下去后会长在视口上方 `y=-110` 完全看不见）。缓存版本 `album_editor.js?v=20260930aa`。
- 遗留（按需求未做）：手机端其余 3 项 —— 按钮 22–32px 偏小、8 个入口收进 ⋮、用法说明被 `html.ave-immersive` 的 `!important` 藏死。

### 8.2 换轨改「长按抓起 → 直接拖到目标轨」（2026-10-01，原报告 §9.1）

- **现象/诉求**：用户要"按住片段并拖动就能换轨"，不接受"长按 → 弹目标轨列表 → 再点一下"。
- **根因**：§8.1 里「拦不住」的结论只对 `pan-y` 成立（`touch-action` 在 `touchstart` 锁定，浏览器接管后必发 `pointercancel`）；换成 `none` 后浏览器根本不碰这个手势。
- **修法**：卡片 `touch-action:none`（`.ave-clip,.ave-aclip,.ave-sub,.ave-ov,.ave-layit`），**`.ave-rows` / 标尺 / 轨道名仍是 `pan-y`**（空白区/标尺/gutter 的竖划照旧原生滚动）；卡片上的竖划改由 JS 代滚（`vertScroller()` 就近找纵向可滚祖先、`pageScrollBy()`、`flingPage()` 惯性 0.94 衰减，双指竖向 `two.axis==="y"` 同路）；长按 0.3s 不再弹列表，改 `drag.grab=true` + 卡片描边 + 状态条「已抓起：上下拖到别的轨道，左右拖改时间」，`openTrackChooser()` 删除；竖向判定由 `pointerType !== "touch"` 改成 `pointerType !== "touch" || drag.grab`（触摸必须先抓起才认竖向位移）；适用卡片扩到 `GRAB_KINDS = ["move","audio-move","sub-move","ov-move","lay-move"]`。缓存版本 `?v=20261001aa`。
- **已知取舍**：卡片起手竖划由 JS 代滚，只有 0.94 衰减的简易惯性，观感比原生滚动略"短"；换来的是长按抓起后竖拖不再被浏览器抢走手势。

### 8.3 抓起动画 + 实时跟手（2026-10-01，原报告 §10.3）

**现象/诉求**：抓起要有动画示意、片段要跟手。**根因/坑（逐条）**：

- 只抬卡片，卡片下移会被后面的兄弟行盖住 → 要给**整行**加 `.ave-grabrow{z-index:6}`。
- 只跟竖向：横向已被「改时间」（`setCardStart` 改 `left`）带着走，再 `translateY` 就是双倍位移。
- **最关键**：浮起的卡片正压在手指下，不关掉 `pointer-events`，`elementFromPoint` 永远命中自己那一行 → 换轨判定全落空，故 `.grabbing{pointer-events:none}`。
- 跟手阶段留过渡，卡片会追不上手指 → 手指一移动就换 `transition:none`。
- 松手回位用 FLIP：松手时卡片已被 `renderAll()` 重建，动画做在旧节点上等于没做 → 记下位移、把**新**节点放回松手位再 `.18s` 滑回本行。
- 落轨示意用 `flashDrop(to)` 让目标行亮一下：换轨后 `sel = null`、节点重建，指不了具体卡片，亮行最稳。

缓存版本 `?v=20261001ac`。

### 8.4 时间轴手感：裁剪把手可见性 + 空隙块可选带参数（2026-10-01，原报告 §10.4）

- **① 两端裁剪把手（现象）**：原来是一条 `8px`、白 18% 的中缝，压在深色卡面/照片上基本看不见。**修法**：改 `11px`（手机 14px）圆角竖条 + 白 26→46% 渐变 + 1px 白 50% 内描边，`::after` 画两根亮握痕，悬停变主题粉；拖动时统一加 `.trimming`（覆盖 `dur / *-dur / trimin / lay-dur`，`layit` 分支自己 `return` 所以单独标），松手/`pointercancel` 由 `clearTrim()` 收。现状：只有视频片段两端都有把手（左=改入点），照片/文字卡只有右把手，`.ave-aclip`（配乐）没有把手。
- **② 空隙块可选 + 参数 + 三种拖拽**（用户要求：空隙也要能选、有参数、勾「固定长度」后拖前边缘带着后面走）：
  - 选中 `sel = {type:"gap", id:<所属片段 id>}`（空隙没有自己的 id，挂在前一段之后的那个片段上）；**只就地改 `.sel` class + 刷面板、不整树重建** —— 重建会把"双击合上"的两次点击拆到两个节点上。
  - 参数：`空隙时长(s)`（数字输入，改 `c.gap`，>0 时强制 `transition = "cut"`）、`固定长度`（新字段 `c.gapFixed`，已进 `projectJson()` 白名单）、`合上空隙`按钮（= 双击）。
  - 拖拽① 不勾：直接改空隙时长（上一段不动，块只变宽，后面节点平移）。
  - 拖拽② 勾 + 左边缘：拉伸/回缩**上一段素材出点**（视频 `trimOut` 夹在 `trimIn+0.2 … srcDur`；图片/文字 `dur` 夹在 `0.4 … 30`），空隙时长锁死。
  - 拖拽③ 勾 + 中/右：**整块后移**（Δ 记到"上一段之前的空隙"`prev.gap += Δ`，夹 ≥0）。空隙起点粘在上一段末尾，不让上一段让位就推不动它 —— **是模型决定的，不是取巧**。
  - 首空隙（`ci === 0`）：前面没有片段可让位 → 勾了固定长度也按①处理，面板里给出说明。
  - 拖拽期间走**局部样式平移**（整树重建会每帧重算音频波形 → 卡）；松手才 `saveProject() + renderAll()`；只点一下（没位移）不重排。

缓存版本 `?v=20261001ad`（当时尚未部署）。

### 8.5 抬起/跟手推广到"任何一次拖动"（2026-10-01，原报告 §10.5）

- **现象**：§8.3 只把抬起/跟手做在**长按抓起**那条路（`drag.grab`），用户反馈手机上普通拖拽没跟上。
- **修法**：任何一张卡被拖动都抬起 + 跟手（`liftCard(card, strong)`：`.ave-lift` 白 40% 描边 + `0 12px 26px` 投影 + `scale(1.06)`，`strong` 再补粉圈 `.ave-grab`），长按抓起只是"多一圈粉边 + 允许竖拖换轨"；触发点在 `dropHighlight()`（该函数只在 `move / ov-move / sub-move / lay-move / audio-move` 里调，裁剪把手的 `*-dur / trimin` 不抬 —— 拉边缘时卡片不该飘）；`.ave-lift` 自带 `pointer-events:none`；收场仍由 `unGrab()` 统一收，FLIP 滑回不变。
- **iOS 坑**：长按 callout / 选择放大镜会直接吞掉手势（"手机上不跟手"常见元凶）→ 卡片加 `-webkit-touch-callout:none;-webkit-user-select:none`。

缓存版本 `?v=20261001ae`。

### 8.6 工具条压缩：按钮变小 + 去 emoji + 行距归零 + 行尾留白（2026-10-01，原报告 §10.6 / §10.6.1）

- **诉求**（用户两轮，附截图）：按钮再小一点、不要按钮上的表情图标、每行不要留空隙。
- **第一轮（行距）**：`.ave-btn` 桌面 `padding:7px 11px→3px 7px`、`font-size:12→11px`、`border-radius:9→6px`；手机 `min-height:32→24px`、`padding:5px 8px→3px 6px`、`font-size:11→10.5px`；行尾 `⋮`（`.ave-more`）`34→24px`（手机 22px）；工具条 9 个按钮 + 播放/全屏**动态文案**去 emoji（保留 `+`/`↶ ↷`/`⋮`，对话框/选择器里的图标与标题未动）；`.ave-head/.ave-bar/.ave-tools/.ave-tl` 的 `margin-bottom` 桌面 `10→0`、手机 `6/5→0`，行内 `gap` 同步收紧。**连带修**：按旧标签索引的说明字典 `TIPS`（`tipAll()` 用按钮文字查表）会失配 → 键名同步 + 补漏 `+ 视频（相册里的）`。
- **第二轮（"还是存在很多空隙"，圈的是行尾空白）**：首轮误判 —— 第一句"每行不要留空隙"我理解成"行距"，只清了 margin；第二句带截图才看清是**每行右侧没被按钮占满的空白** → 按钮 `flex:1 1 auto;min-width:0` 铺满整行（选择器一律 `:not(.ave-more)` —— `.ave-tools .ave-btn` 权重大于 `.ave-more{flex:0 0 auto}`，不排除的话行尾 ⋮ 会被撑宽）；`#aveTitle` 由 `0 0 84px` 改 `1 1 84px`；时间轴 `.ave-zoombar{flex:1 1 auto}` 让滑块吸收剩余宽度。两轮都保留（行距 0 + 按钮铺满，两个方向都不留白）。
- **关键坑（fitRow 测量口径）**：按钮改成 flex 铺满后，行内宽度**恒等于行宽**，原 `usedW()` 一路判成"放不下"→ 会把按钮全收进 ⋮。改成测量前先把可见子项临时 `style.flex="0 0 auto"` 量自然宽，量完还原。

缓存版本 `?v=20261001af`。

### 8.7 叠加轨卡片贴满 + 剪辑页设置路由（全屏策略）（2026-10-01，原报告 §10.7）

- **① 叠加轨上下空隙（现象）**：自建视频轨上的片段上下有空隙、跟主轨不一致。**根因（740 宽实测）**：主轨 `.ave-clip` 是 `top:0;height:42px`（卡片高 46px、距行顶 0、行底 −2px），叠加轨 `.ave-layit` 却是 `top:calc(7px*rowk);height:calc(28px*rowk)` 细条 → 上下各留一圈。**修法**：`.ave-layit` 统一成 `top:0;height:calc(42px*rowk)`，移动端那组高度规则加上它（`.ave-clip,.ave-aclip,.ave-sub,.ave-ov,.ave-layit{height:calc(46px*rowk)}`），改后与主轨逐项一致；`gapblk`（空隙块）/`fxbar`（特效条）保持细条（它们本来就是"标记"，不是素材卡）。
- **② 设置路由 + 全屏策略（诉求）**：要"是否全屏"可设置，入口放最上一行的 ⋮ 里。`api/pages.py` 新增 `GET /video-editor/settings` → 新模板 `templates/video_editor_settings.html`（三档 `auto/full/off` + 每值一句说明 + "← 回剪辑页"走 `__wurl` 带 nginx 前缀）；存 `localStorage["ave_fullscreen_policy"]`（**故意不放服务端**：这是"这台设备上看着舒不舒服"的偏好，跨设备同步会互相打扰），读写都包 try（隐私模式不炸）；入口按钮放不下时被 fitRow 自动收进该行 ⋮。
- **部署坑**：`deploy_l_wchat.py` 的文件清单是**手写**的，新模板必须补进去，否则远端没这个文件、`/video-editor/settings` 直接 500（已加，清单 45 个文件）。

缓存版本 `?v=20261001ag`。

### 8.8 「更多操作」里可指定哪些按钮常显（2026-10-01，原报告 §10.8）

- **诉求**：哪些按钮留行里、哪些收 ⋮ 原来由 fitRow 自动算，用户要自己定。
- **修法**：`openRowMenu()` 从"只列藏起来的"改成**列出这一行所有项**（跳过 `.ave-more` 自己与 `.ave-pri` 主操作），每行 = 左边原按钮/输入副本 + 右边「常显」开关；存 `localStorage["ave_rowPins"]`，**按行记**（行的第一个类名作 key：`ave-head-cfg`/`ave-head-acts`/`ave-tools`）；项 key 用元素 `id` 或内含控件的 `id`，**不能用按钮文字**（"关闭"会变"← 返回相册"）；fitRow 先收**没打勾**的（从后往前），实在放不下才动打勾的，`.ave-pri`（预览 / 生成并上传）照旧永不收。

缓存版本 `?v=20261001ah`。

### 8.9 反代前缀被补了两次：设置页"回剪辑页"跳到 /l_wchat/l_wchat/...（2026-10-01，原报告 §10.9）

- **现象**：`http://127.0.0.1:8080/l_wchat/video-editor/settings` 点"回剪辑页"落到 `.../l_wchat/l_wchat/video-editor`。
- **根因（两处叠加）**：① `base.html` 末尾"渲染后的绝对路径统一补前缀"里 `fixAttr()` 是**无条件补**（`if (v.charAt(0)==="/" && v.charAt(1)!=="/") el.setAttribute(attr, p+v)`），不看是否已带前缀（同文件 MutationObserver 用的 `maybeFix()` 就带 `v.indexOf(p)!==0`）；② 写设置页时又手动 `vsBack.href = window.__wurl("/video-editor")` 先补了一次 → 两遍。**直接 1234 端口看不出来**：那时 `__WCHAT_PREFIX = ""`，`fixAttr` 一开头就 `return`，只有经 nginx `/l_wchat/` 才暴露。
- **定位过程（值得记）**：远端 `__wurl('/video-editor')` 单独调用返回是对的 `/l_wchat/video-editor`，但 `getAttribute('href')` 是双前缀 → 在页面脚本执行前注入钩子包住 `window.__wurl`，记下每次调用的参数/返回/当时 pathname，证明属性是**被后续遍历改的**，顺线找到 `fixAttr`。
- **修法**：① 设置页去掉那次多余的 `__wurl`（`href="/video-editor"` 交给 base.html 的遍历只补一次）；② `fixAttr()` 改幂等（加 `v.indexOf(p) !== 0`，与 `maybeFix` 一致），这类"模板自己补过前缀"的坑不会再犯。

### 8.10 「常显」开关改成反映真实状态（2026-10-01，原报告 §10.10）

- **现象**：用户截图 —— `↶ ↷ 存云端 云草稿 全屏` 明明都在行里，面板里却全显示未勾的「常显」。
- **根因**：开关显示的是**我打过勾没**（pin 标志），不是**这一项现在到底在不在行里**；§8.8 的语义（"打勾=固定留行"）跟用户的期待（"开关＝它现在是不是展开了"）不是一回事。
- **修法**：开关状态读 DOM 真实显隐（`src.style.display !== "none"`）；文案按"点了会发生什么"写（在行里 →「收起」，在更多里 →「常显」）；存储改**两集合** `ave_rowPins = {rowId:{show:[…],hide:[…]}}`（`show`=强制留行、`hide`=强制收起），老格式（数组）读成 `show` 兼容；fitRow 顺序 = ① 先收 `hide` 点名的 ② 还不够就从后往前收没打标的（跳过 `.ave-pri` 与 `show`）③ 打了 `show` 也放得下才留，实在不行兜底收 ④ **收完还有空位就回填**；点完必须 `syncPins()` 重刷整排（否则又变成"不反映真实状态"）。

缓存版本 `?v=20261001ao`。

### 8.11 「常显」第三版：文案固定 + 高亮表示开关（2026-10-01，原报告 §10.11）

- **现象/用户原话**："点击没有反应，希望按钮永远显示常显，用高亮表示是否打开"。
- **根因（§8.10 的副作用）**：① 窄屏下把某项标上后若挤不下，fitRow 的兜底会立刻把它收起 → 看起来"点了没反应"；② 文案在「收起 / 常显」之间跳，用户读不出"这是开关"。
- **修法**：文案**恒为「常显」**（不再变「收起」）；`.ave-pin.on` = 已常显（主题紫渐变 + 亮边 + 白字加粗），关着 = 灰底灰字 —— 只靠高亮区分；开 = 写 `show`（固定留行，挤掉没常显的），关 = 写 `hide`（收进「更多」）；点一下**立刻**高亮变化（不再依赖 DOM 显隐）+ `fitRow(row)` 立即重排（行被面板挡着，点「完成」看结果）；挤不下时标记仍保持高亮（那是"你的意图"），但 title 会写「已常显，但这一行位置不够：先取消别的常显，或屏幕宽一点就会出现」；标记只在窄屏（`innerWidth ≤ 768`）参与 fitRow（桌面本来就全展开、没有 ⋮，面板也进不去）。

缓存版本 `?v=20261001ap`。

### 8.12 时间轴总长 + 当前轨道 + 新素材落点 + 主轨插入位置（2026-10-02，原报告 §10.12）

- **需求**：① 时间线长度别再"由最后一段结尾决定"，要能设；② 轨道前的图标可选中高亮；③ 新插入/导入素材落到**选中轨道**的**播放头**处；④ 轨道与素材不匹配时弹窗问放哪条轨。
- **用户拍板**：**总长 = 标尺长度 + 成片时长**（内容更长导出前再确认并截断；更短则尾部留黑/静音）；**主轨插入时播放头在段中间 → 弹窗问**（切开插入 / 插到该段之后 / 插到该段之前）。
- **实现**：新字段 `P.duration`（秒，0=跟随内容，进 `projectJson()` 白名单）；`totalDur()` = 标尺范围 = `max(内容, 设定值)`；`renderDur()` = 导出多长 = `设定值 || 内容`（`generate()/runOneExport()` 用它）；`renderTimeline` 标尺 `span` 也把设定值算进去（否则标尺不跟着长）；按钮在时间轴行 `时长 自动/0:30:00`，点开对话框（数字输入 + 0/15/30/45/60/90/120 快捷档）。`curTrack`（`""|clips|audio|ov|sub|lay:<id>`）记进视图槽（`albumVideoProject.view.cur`，刷新还在）；`gcell()` 给单元格挂 `data-track`，**单击 = 选中**（`.ave-glabel.cur` 粉边发光）、**双击 = 原来的整轨属性弹窗**、再点取消；`+ 轨道` 新建后自动选中。`resolveTarget(kind, cont)`：选中轨能收就用它，不能收就弹窗列**能收这类素材的轨**（候选顺序主轨/配乐轨/贴纸轨/字幕轨 + 各叠加轨，判定复用拖拽换轨那套 `layerAccepts`），没选轨 → 各流程原来的默认去处；已接线照片、视频、文字卡、字幕、配乐、贴纸（含四条来源）。主轨首尾相接没有绝对位置 → `insertMainClip()`：播放头在内容之后 = 接末尾（带空档）、在某段起点或空隙里 = 直接插在那段之前（剩下空档留给后一段）、**严格在某段中间 → 弹窗**；选"切开"用 `splitClipAt()`（视频按 `trimIn/trimOut` 切、图片/文字卡按 `dur` 切）。
- **还没接当前轨道的流程**（当时未做）：模板、图文成片、AI 图片/视频、自动字幕 —— 它们是"整工程/整轨"级操作，本来就会自己建轨或覆盖时间轴（AI 图片/视频出来的素材若想落到选中轨，下一轮再接）。

缓存版本 `?v=20261002aw`。

### 8.13 去掉"打开空时间轴就自动弹相册选择器"（2026-10-03，原报告 §10.13）

- **现象**：打开 `/video-editor` 时时间轴上一个素材都没有，就自动弹出素材选择组件。
- **根因**：`album_editor.js` 的 `open()` 里有一句 `if (!P.clips.length) addSelectedPhotos();`，而 `addSelectedPhotos()` 在"相册页带过来的勾选为空"时会 `openPhotoPicker()` → 空工程一打开就弹选择器。
- **修法**：删掉那一句调用 + 删掉已成死代码的 `addSelectedPhotos()`（全仓只剩注释里提到它），并在原处留一行注释说明"这是按用户要求去掉的、要恢复就把这两段加回来"。**不受影响**：相册页勾选照片后跳剪辑页走的是 `opts.add`（原样保留）；空时间轴加素材用工具栏的「+ 照片 / + 视频 / …」（会落到 §8.12 的当前轨道 + 播放头处）。

缓存版本 `?v=20261003ax`。

### 8.14 项目文件夹 + 自动存服务器 + 定时转存百度云版本库（2026-10-03，原报告 §10.14）

- **用户模型（口述定）**：**一个项目 = 服务器上一个文件夹**（`project_folder/<项目名>/`），这个文件夹就是百度云版本库的工作区；**素材本体永不上传**，工程 JSON 只存素材链接；新建项目必须在顶部输入项目名。节奏：**存服务器跟手（改动停 ~4s）、版本库节流（≥10 分钟一版）+ 后台定时补提**。
- **实现**：新增 `services/video_projects.py`（`ROOT = <USER_DATA_DIR>/project_folder`；`depot_path(name) = /l_wchat/剪辑工程/<名字>/project.json`；`save_project()` 写工作副本 + 按节流决定是否顺手提版、`submit_version()` **复用成长记录的 `_depot_submit`**（靠 md5 去重不堆版本）、`sweep()` + 守护线程 `_start_sweeper()` 每 10 分钟把"有改动但没提版"的补提一版 = 用户要的"定时转存"、`history()`/`read_version()`）；路由 `GET/PUT/DELETE /api/album/video-projects[/<name>]` + `POST …/<name>/backup` + `GET …/<name>/history[/<rev>]`（PUT 有 2MB 上限与"素材只存 URL"的报错文案，阻断式 depot 调用都走 `run_in_threadpool`）；前端顶栏（`.ave-head-cfg` 标题之后）加「项目名」输入 + 「项目」按钮，`saveProject()` 末尾挂 `scheduleServerPush()`（4s 防抖 PUT），项目名记在视图槽（刷新还在）；项目弹窗 = 列表（名字·段数·KB·更新时间·最近 rev）+ 每项 [打开][历史版本][删服务器副本] + [新建/另存为][备份当前项目一版到版本库]；历史版本点一版 → 先自动存一版当前（可反悔）→ 用 `restoreSnap()` 换成那一版。
- **例外**：工具栏「+ 照片 → 本机文件 / 本机路径」这条来源仍要把文件传到服务器才有 URL（否则刷新即失效），已在文档标注。
- **实测（curl 全链路）**：`PUT` → `{ok:true, bytes:494, submitted:true, rev:1}`，本地出现工作副本；`history` 显示版本真的进了**百度云版本库**（每项目一个目录）；内容没变再 PUT → `submitted:false, dirty:false`（不堆版本）；改成 2 段后 PUT → `dirty:true, submitted:false`（被 10 分钟节流挡住，**不是丢**）；`POST …/backup` → `rev:2`；`GET …/history/1` → 能取回那一版。
- **部署坑**：新模块必须进手写清单（漏了那些路由会 `ImportError` 500）—— 已加，清单 48 个文件。
- 旧的「云草稿」（`/album/video-project`）**保留未动**，与新项目机制并存。

缓存版本 `?v=20261003ay`。

### 8.15 「没勾固定长度，拖空隙却把上一段推走」排查（2026-10-03，原报告 §10.15）

- **现象**：用户反馈空隙没勾「固定长度」，拖动却像锁了长度、还把前面的片段推走。
- **结论：不是 bug** —— 是那个空隙身上 `gapFixed = true`。这个标记**按每个空隙存在工程里**（`gapblk` 的 `data-gapid` → `P.clips[i].gapFixed`，进 `projectJson()` 白名单），勾过一次就一直跟着，改天再拖它就是"固定长度"的那套行为。（同拖拽量 +46px：`gapFixed:false` → 上一段 `left/gap` 全 0、块只变宽；`gapFixed:true` → `c1.gap 0→1`、`c1.left 0→46px`。）
- **顺手做的两处"别再猜"**：① 拖动时状态栏把走的哪一支说明白（不勾 =「（只改时长，上一段不动）」；勾着 = 末尾补「（这个空隙勾着「固定长度」：想只改空隙时长就把它取消）」）；② 固定态的块原来是"实线但同色"不好认 → 改成**紫实线**（`.ave-gapblk.fix{border-color:#b79cf0}` + 淡紫斜纹），块 title 写清拖法。
- **辨识口径**：**虚线 = 不固定，紫实线 = 固定**；选中块 → 参数面板「固定长度」勾选框就是真实值；点「合上空隙」会顺带把它清掉。

缓存版本 `?v=20261003az`。

### 8.16 全屏策略改由包内 `.env` 决定（2026-10-03，原报告 §10.16）

- **需求**：在 `l_WChat/999.0/src/l_WChat/` 下加一个 `.env` 决定视频编辑页是否全屏。
- **实现**：`.env`（新增，包根）`VIDEO_EDITOR_FULLSCREEN=auto`（可选 `auto`/`full`/`off`）；`paths.py` 新增 `env_of(key, default)`（纯标准库解析 `KEY=VALUE`，# 注释、引号都认）与 `video_editor_fullscreen()`（**每次都现读文件** → 改完不用重启；非法值回落 `auto`）；`api/pages.py` 两个路由把 `fs_policy` 放进模板 context；生效 = `pol = (localStorage 里显式选过 full/off) ? 它 : SERVER_POLICY`，两者都没有才 `auto` —— 即**服务端 `.env` 是整机默认，浏览器「设置」里显式选过的覆盖它**（选「自动」= 跟服务端）；设置页说明会把当前服务端默认值念出来。
- **部署**：**故意不进部署清单** —— 通道本来就把 `.env` 排除（`FULL_MODE_EXTRA`，防密钥覆盖远端），远端保留自己那一份；远端现在没有该文件 = 走 `auto` 兜底。想让每次部署同步本地值，就把它加进 `deploy_l_wchat.py` 的 files。
- **约束**：`.env` 只放这种**无密**开关，密钥/口令仍走 `<user_data>/l_WChat/config.json`（源码树会被镜像/提交）。

### 8.17 「全屏」语义改成"收掉底部标签栏"，不再请求系统全屏（2026-10-03，原报告 §10.17）

- **现象/澄清**：用户说他说的全屏**只是让手机底部那排标签切换按钮不显示**，不是浏览器系统全屏。
- **修法**：**删掉**整段"第一次真手势顺势要系统全屏"的逻辑（`tryRealFull` / `armFull` / 四个手势监听 / `fullscreenchange` 依赖）—— 模板里 `requestFullscreen` 一个都没有了；策略只决定要不要给 `<html>` 加 `ave-immersive`（`full` 一律收、`off` 一律保留、`auto` 窄屏/触屏收、宽屏保留）；编辑器右上角的「⛶ 全屏」**保持原样**（那是用户主动点的系统全屏，与这个策略无关）；设置页三档说明 + 包内 `.env` 注释同步改口径。
- **顺手修的一个坑**：设置页新增的说明串里一开始用了 ASCII 双引号包中文（`"这是"编辑器整页观感"，…"`）→ 会把 JS 字符串截断、整页脚本崩；已改成中文书名号「」。

### 8.18 再改：策略只加 `ave-hidenav`（真·只藏底部标签栏）（2026-10-04，原报告 §10.18）

- **现象/反馈**："还是会导致这部分消失" —— §8.17 虽然不再请求系统全屏，但仍把页面切成 `ave-immersive` 那套"整页观感"（除底部标签栏外还收掉页头说明、卡片留白、全局返回钮、快捷菜单），而用户要的**只是底部那一排消失**。
- **修法**：新增专用规则 `html.ave-hidenav .bottom-nav,html.ave-hidenav #bottomNav{display:none !important;}`；`ave-immersive` 那套**保留不动**（它现在只服务于编辑器右上角「⛶ 全屏」按钮）；页面策略改成 `document.documentElement.classList.toggle("ave-hidenav", !!wantHide)` —— 不再加 `ave-immersive`，所以页头/留白/返回钮/快捷菜单都回来了；三档行为 `full` = 一律藏底部标签栏、`off` = 一律保留、`auto` = 窄屏/触屏藏、宽屏不动（远端 `.env` 仍是 `auto`）；设置页三档说明 + `.env` 注释改成"**只**管底部那一排（页头/留白照旧）"。

缓存版本 `?v=20261004ba`。

### 8.19 UI 再紧凑一轮（2026-10-04，原报告 §10.19）

- **诉求**："请让 ui 紧凑一些"。§8.18 之后页面外壳（页头/卡片留白）回来了，观感比之前松，这一轮把**外壳 + 编辑器自身的间距**统一压一档（间距/字号，不动功能与布局口径）。
- **改动（改前 → 改后）**：页外壳（`video_editor.html` 内联样式）`.ve-title` 加 `margin:0 0 6px`（原来没写，吃默认 1em）、`.ve-help` `13px/1.7/mb:14px` → `12.5px/1.55/mb:8px`、说明按钮 20→18px、手机档标题 `17→16px` / `mb:8px→4px`；`.ave-wrap`（编辑器外框）`padding:14px 16px 18px` → `8px 10px 10px`（手机 `8px 8px 96px` → `5px 6px 78px`）；`.ave-stage`（画布区）`padding:8px;margin-bottom:4px` → `5px;3px`；`.ave-tl` / `.ave-tl-head`（时间轴块）`padding:8px` → `5px`、块头 `12px/mb:2px` → `11.5px/1px`（手机 `padding:6px` → `4px`）；`.ave-insp` / `.ave-irow`（参数面板）`gap:10px 18px;padding-top:8px` → `6px 10px;6px`、行内 `gap:8px` → `6px`（手机 `padding-bottom:8px` → `5px`）；`.ave-status`（状态栏）`margin-top:10px;font-size:12px;line-height:1.7` → `5px;11.5px;1.45`（手机 `mt:6px;pb:4px` → `4px;2px`）。
- **还能更紧的两个杠杆**（这轮没动，要就说）：① 时间轴行高（`--ave-row-h` 44→40 + 卡片 42→38，手机保持 44/46 的比例），代价是手机点按面积再小一点；② 头部工具行/时间轴块头再降 1px 字号。

缓存版本 `?v=20261004bb`。

### 8.20 部署通道记录与远端根坑（原报告 §10.1 / §10.2）

- **2026-09-30（§10.1）**：用现成的 `rez-package-source/l_repo_sync_gui/999.0/deploy_l_wchat.py`（`l_repo_sync_gui` 的"直推部署"，不走 git：base64 上传 → sha256 全量校验 → 按映射杀端口重拉）；本次 4 个文件（`static/album_editor.js`、`templates/video_editor.html`、`templates/album.html`、`templates/base.html`）本来就在它的显式清单里，无需改清单。执行 = 上传 44 文件 → `checked=44 mismatch=0` → 重启 `:1234 l_wchat_backend`。远端回验注意：公网入口是 **443 HTTPS + 自签证书，要 `-k`**（`http://` 与 `:8080` 都会 404）。
- **2026-10-01（§10.2）坑**：`deploy_l_wchat.py` 的远端树根是**照本机 `__file__` 推导**的（"按标记目录运行期推导，不写死盘符"）；本机开发树在 `E:\lugwit\trayapp`，而远端那棵树在 `D:\TD_Depot\Software\Lugwit_syncPlug\lugwit_insapp\trayapp` —— **远端连 E: 盘都没有**。照默认跑 `--dry` 会报 **44 个全 missing**（指向远端不存在的 `E:\…`），真推就是往错的树写文件。**修法**：脚本顶部加两个可覆盖入口（默认仍是推导值，不改变原语义）：`set DEPLOY_REMOTE_ROOT=D:\TD_Depot\…\trayapp\rez-package-source` / `set DEPLOY_WUWO_DIR=D:\TD_Depot\…\trayapp\wuwo`。
- 此后每轮部署都先 `--dry` 确认再推，并做**远端回验**（页面引用的 `?v=`、JS 与本地 `sha256` 全等、业务端点探活 200）；远端 `/video-editor*` 后来开始要登录（302 到 `/login`），部分轮次只能靠"部署校验逐字节一致 + 静态断言"代替匿名渲染校验。

### 8.21 悬浮小窗预览"不生效"定位与修复（2026-09-30，原报告 §10）

- **现象**：主预览滚出屏幕后，应该浮出的「小窗预览」看不见。
- **排查/根因**：机制本身是好的 —— `#aveCanvas` 用 `IntersectionObserver`（阈值 0.15/0.35/0.6/1）盯着，`fltRatio < 0.35` 就浮；实测（390×844）可见率 0.229 → `.ave-float` `display:block` 且画布已绘制（`getImageData` 非零），所以"没浮出来"不成立。**真正原因**：小窗位置持久化在 `localStorage.aveFloatPos`，但**恢复时按原值直接写 `left/top`、不按当前视口夹取** —— 宽屏那次拖到右下角存成 `[920,481]`，换到 390 宽手机上算出来 `left:920px` → 小窗**整块落在屏幕外**，看着就像"没生效"（同一份 `localStorage`，修复前 `left=920` 不可见、修复后 `left=228` 完整可见）。
- **修法**（`static/album_editor.js`）：`ensureFloat()` 恢复位置时先按当前视口夹一次（宽用 `FLT_CSS_W`、高用视口高兜底）；新增 `clampFloatPos()` 用**真实尺寸**（`offsetWidth/offsetHeight`）再夹一次，在 `syncFloat()` 里 `paintFloat()` 之后调用（画完才有真实高度）；监听 `resize` / `orientationchange` → 转屏、拉窗口后也会把小窗拉回屏幕内。缓存版本 `?v=20260930ab`。
