# l_WChat 视频剪辑页 · 非 AI 功能 自动化测试报告

- 被测对象：`rez-package-source/l_WChat/999.0/src/l_WChat/templates/video_editor.html` + `static/album_editor.js|album_video.js|album_beats.js`
- 被测实例：本机 `http://127.0.0.1:1234/video-editor`（`l_wchat_backend` 直连，未走 nginx 前缀）
- 测试方式：**无头浏览器自动化**（Playwright MCP，`--headless` + `--viewport-size=1440,1000`，桌面鼠标语义）
- 测试时间：2026-09-30
- 范围：**排除全部 AI 功能**（自动字幕 / 图文成片 / AI 视频 / AI 图片 / AI 配音 / 从 AI 素材选 / `/api/album/ai/*`）
- 工程基线：页面自动恢复的本机工程（13 段主轨 / 4 字幕 / 2 配乐 / 2 贴纸 / 总时长 0:41.0）

---

## 1. 结论速览

| 项 | 结果 |
|---|---|
| 内置自检 `AlbumEditor.selftest()` | **22/22 通过**（前提：`吸附拍点` 打开，见 §4.2） |
| 黑盒 UI 用例 | **41 通过 / 0 应用缺陷失败**（3 条首轮失败均为测试脚本自身选择器/落点问题，已复测通过） |
| 阻塞性故障 | **无**。页面加载、编辑、预览、渲染导出、下载成片全链路在无头 Chrome 下跑通 |
| 需关注问题 | 5 条，全部为「非阻塞 / 噪音级」或「测试自检质量」问题，见 §4 |

> **修复状态**：§4 的 5 条问题已于 2026-09-30 修复并逐条复验，见 §8。

**能用、好用**：除 AI 之外的功能在自动化下全部可操作且行为符合预期，包括 41s 工程的真渲染导出（另用 2s 最小工程计时）。

---

## 2. 环境与前置校验

| 校验点 | 结果 |
|---|---|
| 页面加载 | `GET /video-editor` 200，标题 `视频剪辑 - WChat`，无登录闸门（页面与 `/api/album/*` 均公开） |
| WebCodecs 门禁 | `typeof VideoEncoder === 'function'`；`isConfigSupported('avc1.42001f')` = **true**，`vp09.00.10.08` = **true** → 编辑器正常打开 |
| 分辨率档次探测 | 720p / 1080p 可选；**2160p(4K) 灰掉**（文案「本机不支持」，低端机/无硬件 4K 编码面） |
| 帧率 | 30fps / 60fps 均可用 |
| 工程恢复 | localStorage 自动恢复上次工程，13 段 / 0:41.0 一致 |
| 视口 | 1440×1000（桌面展开态：工具行无 `⋮` 收纳） |

> 说明：无头 Chrome 的 `VideoEncoder` 具备 H.264 软编能力，本页未触发 `webcodecsBlockReason()` 拒绝路径。

---

## 3. 用例矩阵与结果

### 3.1 内置自检（`await window.AlbumEditor.selftest()`）

22 条断言，全部离线（无网络/AI 调用）。

| 断言 | 结果 |
|---|---|
| 吸附：容差内命中拍点 / 容差外不吸附 / 目标含拍点+边缘+播放头+0 | 通过 |
| 分组：整组平移不改组内相对时间 | 通过 |
| 蒙版反向：中心/边角互补 | 通过 |
| 特效：强度 0 像素零变化 / >0 画出元素 / 同一 t 结果确定（预览=导出） | 通过 |
| 调色：identity 不触发像素路径 / LUT 单调且端点正确 / S 曲线压缩阴影提亮高光 | 通过 |
| 转场：库内 ≥8 种（实测 10 种，非直切 9）/ 中间帧同时含 A、B 内容 | 通过 |
| 导出队列：连点两次计数 1/2、2/2 | 通过 |
| 图文成片切句 ≥4 段（纯字符串函数，未触 AI 端点） | 通过（AI 功能本体未测） |
| P2-2：2160p 像素与码率（3840×2160 @216Mbps）/ 码率档 high>auto>low | 通过 |
| 缩略图三级兜底 / 视频不拿原图当背景 / 无 thumb+stored 时 0 候选 | 通过 |
| 清理：素材收集覆盖各容器、跳过文字卡/字幕 | 通过 |

### 3.2 黑盒 UI 用例（无头浏览器真实点击/输入/拖拽）

**A. 头部设置行**

| 用例 | 结果 |
|---|---|
| 片头标题输入框 | 通过 |
| 分辨率下拉（→1080p） | 通过 |
| 比例下拉（→9:16 竖屏） | 通过 |
| 帧率下拉（→60fps） | 通过 |
| 码率档（→高） | 通过 |
| 音频闪避 checkbox 勾选 | 通过 |
| 模糊背景 checkbox 勾选 | 通过 |
| 吸附拍点 checkbox 取消 | 通过 |
| 生成后上传版本库 checkbox 取消 | 通过 |
| 撤销/重做按钮禁用态渲染 | 通过（redo 初始禁用，有操作后 undo 启用） |

**B. 工具行**

| 用例 | 结果 | 关键证据 |
|---|---|---|
| + 文字卡 新增 / 撤销 / 重做 / 再撤销 | 通过 | clips 13 → 14 → 13 → 14 → 13 全对 |
| + 照片 来源弹窗 | 通过 | `☁️ 从相册里选` / `📁 本机文件` / `取消`，点取消正确关闭 |
| + 视频 来源弹窗 | 通过 | 同上 |
| + 配乐 来源弹窗 | 通过 | `用这个路径` / `从催眠混音台选` / `🔎 免费曲库搜索（IA / Freesound / Jamendo / Openverse）` / `取消` |
| + 贴纸 来源弹窗 | 通过 | `从相册选图` / `从 AI 素材选`(AI，未点) / `用这个路径` / `取消` |
| 📐 模板 面板 | 通过 | 卡点自动（最近 8/12/20 张）、3 个现成工程、6 个结构模板、发布到社区、已有工程「套用/删」 |
| + 轨道 菜单 | 通过 | 5 类：视频轨 / 贴纸轨 / 字幕轨 / 音频轨 / 特效轨 |
| + 字幕 新增 → 撤销 | 通过 | subs 4 → 5 → 4 |
| ✂️ 裁剪片段（先选中视频片段） | 通过 | 弹窗含 `源 2.0s · 选中 0.3–1.8s`、起点/终点输入、`▶ 试看选中段`/`■ 停`/`起点=预览处`/`终点=预览处`/`用满素材`/`完成` |
| 🎵 分析拍点 | 通过 | `拍点 77 个 · ~193.2 BPM`（纯前端 FFT/自相关，非 AI） |
| 🧹 清理损坏素材 | 通过 | 执行完成，无弹窗阻塞、无异常 |
| + 轨道 → 新增「✨特效轨」 | 通过 | 轨道行 5 → 6 → 撤销 5 |

**C. 预览 / 时间轴**

| 用例 | 结果 | 关键证据 |
|---|---|---|
| 缩放 −/+/全长 | 通过 | 46→64→46；全长 →26 |
| 参数按钮跳检查器 | 通过 | `#aveInsp` 存在 |
| 检查器 · 改时长 | 通过 | 滑块 2.5→1.5 → 总时长 `0:41.0 → 0:40.0`，撤销回 0:41.0 |
| 检查器 · 转场改「叠化」 | 通过 | `fade → dissolve` |
| 检查器 · 推近 Ken Burns 切换 | 通过 | true→false→还原 |
| 检查器 · 删除这个片段 → 撤销 | 通过 | clips 13 → 12 → 13 |
| ▶ 预览播放 | 通过 | `0:00.0 → 0:02.7`，按钮变 `⏸ 暂停`；再点停住 |
| 播放进度条 seek | 通过 | 点 60% 处 → `0:24.6 / 0:41.0` |
| 时间轴拖拽片段 | 通过 | 按 `data-id` 追踪，x 679 → 646（left 504.5px → 471.8px，吸附生效） |
| 双击「空隙」合上 | 通过 | 总时长 `0:41.0 → 0:33.2`，撤销回 0:41.0 |
| 轨道属性弹窗（点视频轨图标） | 通过 | 透明度 / 混合模式（14 种）/ 移除这一轨 / 完成 |
| 多选（Shift+点）→ 批量检查器 | 通过 | 「已选 2 个」+ 批量改时长 / 统一时长 / 批量套动画 / 批量套转场 |
| 时间轴空白框选 | 通过 | 拖拽中出现 `.ave-selbox`，释放后「已选 2 个」 |
| ⛶ 全屏 / ⤢ 退出全屏 | 通过 | 按钮文案来回切换 |
| 视频片段专属参数 | 通过 | 检查器含 变速 / 倒放 / 视频音量 / 分离音频到音频轨 / ✂️ 裁剪素材 |

**D. 云草稿 / 模板 / 持久化**

| 用例 | 结果 | 关键证据 |
|---|---|---|
| ☁ 存云端 | 通过 | `PUT /api/album/video-project/<名>` **200** |
| ☁ 云草稿 列表 | 通过 | `☁ 云草稿（1） · 13 段 · 09-30 11:07` + 删除这份云草稿 / 关闭 |
| 模板套用「暖阳回忆（现成）」 | 通过 | 二次确认后 clips 13→5、总时长 0:41.0→0:12.1 |
| localStorage 自动保存 | 通过 | 工程 JSON 11761 B，含 title/res/ratio/clips 等 |
| 刷新恢复 | 通过 | 重载后 clips/total/设置项逐项一致 |

**E. 导出渲染（真渲染，非 mock）**

| 用例 | 结果 | 关键证据 |
|---|---|---|
| 🎬 生成并上传（`生成后上传版本库` 置 OFF 分支） | 通过 | 进度 `导出 1/1：编码 0% · 0:00 / 0:02（10%）` → `✅ 已生成（未上传）· 0:02 · 720p · 30fps · 130.1 KB · 无配乐 · ⬇ 下载成片` |
| 渲染耗时 | 通过 | 2s / 720p30 / 2 段（文字卡 + fade）工程 **≈5s** 完成（无头 Chrome 软编） |
| ⬇ 下载成片 | 通过 | 实际触发下载，落盘文件 `视频相册_110816.mp4` |

> 导出用例为不污染版本库，使用**最小 2s 工程 + 关闭上传**；原始 41s 工程未做整片渲染。

---

## 4. 发现的问题（均非阻塞）

### 4.1 【P3 · 噪音】缩略图代理 404 且被反复重试

**现象**：页面加载即报 3 条 404，且在**每次时间轴重绘**时重复请求同一批 URL。实测 5 秒内同一个 mp4 的 404 出现 8 次。

```
404 /api/album/thumb/3cb765bf86aa405b8902e685afd8d1bb.mp4   ← 视频片段（plain_video.mp4，depot_only）
404 /api/album/thumb/faf5802e7bf94f61b7e943c1236274b2.jpg   ← depot_only 照片
404 /api/album/thumb/61768cd286e04e518891e8f957161afa.jpg   ← depot_only 照片
```

**原因（代码已自述）**：
- `api/routes.py:7536-7559` — `GET /album/thumb/{stored}` 只在「索引里存在 **且** 云盘能给出 `thumb_url`」时返回；否则 404（`这张没有可用缩略图（可能还没同步到云盘）`）。本机这 3 个素材是 `depot_only: true`，没本地回源过，云盘侧没缩略图挂点。
- `album_editor.js:786-802` `thumbCands()` — 候选链 `CDN thumb_url → /api/album/thumb/ → /api/album/file/`（视频禁用第三级，避免白下 mp4）。**功能上有兜底，用户看不到故障**。
- `album_editor.js:790` — 命中结果缓存在 `o._thumbOk`（**运行时字段，不落工程 JSON**）；而时间轴重建时片段对象由工程 JSON 重新生成 → 缓存失效 → **每次重绘把 404 重打一遍**。

**影响**：无用户可见故障（视频卡最终用 base64 海报兜底，已实测 `background-image: url(data:image/jpeg;base64,…)`；文字卡本来无图）。仅控制台噪音 + 冗余请求。

**建议**：把 `_thumbOk` 的「负结果」缓存提升为按 `stored` 的模块级 Map 并按时间失效（或落进工程 JSON）；服务端对已知无缩略图的素材返回 204 而非 404，避免控制台刷红。

### 4.2 【P3 · 测试自检质量】`selftest()` 的吸附断言不独立于用户设置

**现象**：当「吸附拍点」checkbox 为**关闭**时，`selftest()` 22 条里必挂 1 条：

```
FAIL 吸附：容差内命中拍点 | pxPerSec=320 容差=0.031s
```

把「吸附拍点」打开后 **22/22 全过**。已验证与缩放级别无关（26 / 46 / 120 / 320 px/s 四档，关闭时全挂、打开时全过）。

**影响**：`selftest()` 是控制台/CI 冒烟入口，用户合法的「关掉吸附」会让自检报红 → 误报。

**建议**：断言前临时把 `P.snap` 置 true（用例本身已临时改 `P.beats`，同一套写法），或把该断言标注为「仅在启用吸附时适用」并跳过。

### 4.3 【P4 · 无用开销】剪辑页仍加载并初始化宝宝爬屏特效

`video_editor.html:31` 只用 CSS 隐藏 `#babyCrawlWrap / #babyBubble`，但 `base.html` 仍把 `baby-crawl.js` + Three.js 拉起来并执行初始化。控制台可见：

```
[LOG] Checking THREE: object
[LOG] Three.js is ready
[LOG] setBabyStyle called with: svg
```

**影响**：剪辑页白做功（3D 上下文 + 初始化逻辑），仅性能/日志噪音。

**建议**：page 模式下直接不加载该脚本（输出 skip 标志）。

### 4.4 【P4 · 预期噪音】无用户手势时 `requestFullscreen` 被拒

```
[WARNING] Failed to execute 'requestFullscreen' on 'Element': API can only be initiated by a user gesture.
```

`video_editor.html:50-60` 已用 `p.catch()` 吞掉拒绝并保留「沉浸样式」，行为正确（真全屏退化为首次点击后升级）。属浏览器策略的可预期告警。

### 4.5 【P4 · 交互习惯】pick 弹窗不响应 Esc

所有 `.ave-picker` 弹窗（加照片/视频/配乐/贴纸/模板/轨道/裁剪）**只能点「取消 / 关闭 / 完成」关闭**，按 `Esc` 无效（实测连开 20+ 个弹窗全部堆叠不关）。功能上无碍，但桌面用户按 Esc 会有「没反应」的观感。可选改进。

---

## 5. 未覆盖 / 未验证（诚实边界）

| 项 | 原因 |
|---|---|
| 全部 AI 功能（自动字幕 / 图文成片 / AI 视频 / AI 图片 / AI 配音 / 从 AI 素材选） | 按需求排除；相关端点 `/api/album/ai/*` 未调用 |
| 免费曲库搜索（IA / Freesound / Jamendo / Openverse）、催眠混音台选曲 | 依赖外部服务与网络，本轮未打 |
| 「生成后上传版本库」**上传分支** | 避免污染版本库；只验证了 `未上传 + ⬇ 下载成片` 分支 |
| 41s 原始工程整片渲染耗时 | 导出链路已用最小工程验证；未测长片性能 |
| 手机 / 触摸分支（`pointerType==='touch'` 的横向平移、双指缩放、`⋮` 收纳菜单） | 本轮固定桌面视口与鼠标语义 |
| 关键帧动画编辑器内部、调色曲线弹窗内部、贴纸选择器内部、批量动作的实际套用 | 只验证入口打开与控件齐全 |
| HEIC / HEIF 素材（浏览器解不了） | 本机索引无此类 |
| 宽银幕/超窄屏断点（`innerWidth <= 768`） | 只用 1440×1000 |

---

## 6. 复现方式（可重跑）

```powershell
# 1) 确认后端在跑（l_wchat_backend，1234）
powershell -Command "(Invoke-WebRequest http://127.0.0.1:1234/api/album/views -UseBasicParsing).StatusCode"   # 期望 200

# 2) 无头 MCP 浏览器由 ~/.codemaker/mcps.json 的 playwright-mcp-server 提供
#    本次已加 ："--headless"、"--viewport-size=1440,1000"
```

- 页面：`http://127.0.0.1:1234/video-editor`（无登录；带素材可 `/video-editor?add=<stored,stored>`）
- 自检入口：控制台 `await AlbumEditor.selftest()`（**先确保「吸附拍点」勾上**）
- 关键 DOM 锚点：`#videoEditorRoot`（页模式挂载）、`#aveInsp`（检查器）、`.ave-picker`（弹窗）、`.ave-clip/.ave-sub/.ave-aclip/.ave-ov/.ave-layit`（时间轴卡片）、`#aveTotal`（总时长）、`#aveLoad`（导出进度）

---

## 7. 本次测试对环境的改动（已回收）

| 改动 | 状态 |
|---|---|
| `~/.codemaker/mcps.json`：playwright-mcp-server 增加 `--headless`、`--viewport-size=1440,1000` | **保留**（用户要求，浏览器窗口不再前置显示） |
| localStorage 工程 `albumVideoProject` 的 title/分辨率/比例/帧率/码率/音频闪避/模糊背景/吸附拍点/生成后上传 | **已还原**为测试前原值（title 空、720p、16:9、30fps、码率自动、闪避 off、模糊 off、吸附 on、生成后上传 on），刷新后逐项复核一致 |
| 云端草稿（测试产生 1 份） | **已删除**，`GET /api/album/video-project` 返回空列表 |
| localStorage 临时备份键 `__origProjectBackup` | **已清理**（无 `__*` 残留键） |
| 导出渲染产物 | 落到 playwright 下载目录 `视频相册_110816.mp4`（未改仓库） |

---

## 8. 缺陷修复记录（2026-09-30）

### 8.1 修复清单

| # | 问题 | 改动 | 文件 |
|---|---|---|---|
| 1 | 缩略图 404 每次重绘重打 | 新增**负结果缓存**：按 URL 记「确认取不到」，10 分钟 TTL；`setCardThumb` 候选循环改为 `while` + 跳过已知坏 URL；`onerror` 时记一笔 | `static/album_editor.js`（`thumbCands` 之后新增 `thumbBadAt/thumbUrlKnownBad`；`setCardThumb`） |
| 2 | 自检吸附断言随用户开关误报 | `selftest()` 第 1 组先临时 `P.snap = true`，断言完连同 `P.beats` 一起还原 | `static/album_editor.js` `selftest()` |
| 3 | 剪辑页白加载宝宝爬屏特效 | `base.html` 把 three.js + baby-crawl.js 那段包成 `{% block baby_crawl %}`；`video_editor.html` 覆盖为空块 | `templates/base.html`、`templates/video_editor.html` |
| 4 | 无手势时 `requestFullscreen` 被拒并报 warning | `tryRealFull` 先用 `navigator.userActivation.isActive` 判断；不是真手势就**不消耗监听**，留给下一次；成功后摘掉监听（去掉 `once:true`） | `templates/video_editor.html` |
| 5 | 弹窗不响应 Esc | 新增 document 级 keydown：Esc 关掉栈顶 `.ave-picker` 并 `stopPropagation`（不干扰 `.ave-cmask` 曲线编辑器） | `static/album_editor.js` `build()` |

配套：`album.html` / `video_editor.html` 的 `album_editor.js?v=` 由 `20260930w` 逐次提到（最终 **`20260930aa`**，含 §9 的手机换轨改动），否则浏览器吃旧缓存。

### 8.2 复验结果（无头浏览器，同一实例）

| 复验项 | 结果 |
|---|---|
| 缩略图请求量 | 开局加载：3 次探测（3 个素材各一次，不可免）；**连续 10 轮时间轴重绘后：0 次 404**（修复前同一 mp4 在 5 秒内被打 8 次） |
| 缩略图表现未回退 | 13 段里仍只有 1 段无背景（文字卡，本就无图）；视频卡正常显示封面 |
| 自检健壮性 | `吸附拍点` = OFF → **22/22**；= ON → **22/22**；自检不残留改动（跑完 checkbox 仍是 OFF） |
| Esc 关弹窗 | 叠开 3 层 → Esc → 2 → 1 → 0；多按一次无副作用、不误关编辑器 |
| 剪辑页特效 | 控制台不再出现 `Checking THREE` / `Three.js is ready` / `setBabyStyle`；页面无 three.js 与 baby-crawl.js 请求 |
| 相册页回归 | `typeof THREE === 'object'`；`baby-crawl.js` 仍加载并渲染出 canvas（默认块未受影响） |
| 浮层编辑器回归 | 相册页 `AlbumEditor.open({page:false})` 正常：13 段 / 0:41.0 / selftest 22/22 / Esc 关弹窗生效 |
| 语法检查 | `node --check album_editor.js` → OK |

### 8.3 未改动 / 仍存在的已知噪音

- 首次加载时 3 个 `depot_only` 素材仍会产生 3 条 `404 /api/album/thumb/<stored>`（探测本身无法消除；已不再重复）。若要连这 3 条也消掉，需服务端对「已知无缩略图」返回 **204** 而非 404 —— 属 API 语义变更，本轮未做。
- 无头环境仍会在真手势判定上退化为「保持沉浸样式」（不自升级真全屏），这是浏览器策略，非缺陷。

### 8.4 生效方式

本轮改动为 Jinja 模板 + 静态 JS：模板即时生效（已实测），静态资源靠 `?v=` 破缓存，**未重启服务**。

---

## 9. 手机端换轨入口（2026-09-30 增补）

**问题**：屏幕提示写「上下拖=换轨（图片→贴纸/画中画、文字→字幕）」，但代码里触摸的竖向拖被显式排除
（`dropHighlight` / `pointerup` 两处 `ev.pointerType !== "touch"`）→ 手机上换轨既没手势、也没替代入口。

**为什么「长按后竖拖」这条路走不通**（390×844 + 真触摸，实测事件流）：

```
pointerdown(touch, ave-clip) → pointermove(touch) → pointercancel(0,0)
```

卡片的 `touch-action` 是 `pan-y`，该值在 `touchstart` 就锁定；手指第一次竖移即被浏览器当滚动接管并发出
`pointercancel`，拖拽状态当场清空 —— 连非 passive 的 `touchmove` + `preventDefault()` 也拦不住。

**做法**：改成「按住片段不动约 0.3s → 页面给提示 + 弹出目标轨列表」，全程零位移，与竖划滚动零冲突。

- 新增 `openTrackChooser(c)`：按素材类型给目标 —— 文字 → `💬 字幕轨`；图片/视频 → `🖼 贴纸 / 画中画`；
  再补上 `layerAccepts()` 通过的叠加轨。点选后走既有 `moveAcross("clips", to, id, 0)`。
- 长按计时器挂在**主 `pointerdown` 之后注册**的第二个监听器里（这样才能读到刚建好的 `drag`）；
  手指先横划（>8px）→ 取消计时、照旧「改时间」；先竖划（>8px 且竖向为主）→ 让位给页面滚动。
- `pointerup` / `pointercancel` 里清计时器，免得抬手后才弹窗。
- 弹窗 `z-index: 4000` + `scrollIntoView({block:"center"})`：`.ave-picker` 是绝对定位在编辑卡内的，
  在时间轴上长按时页面常已滚下去，不滚回来会落在视口上方（实测 `y=-110`，完全看不见）。
- 提示文案同步改为「上下拖=换轨（手机上按住片段→选目标轨）」。
- 缓存版本 `album_editor.js?v=` 提到 **`20260930aa`**。

**验证**（390×844，CDP 真触摸）：

| 用例 | 结果 |
|---|---|
| 长按图片片段 | 弹「换到哪条轨？「1000064928.jpg」」+ `🖼 贴纸 / 画中画`、`取消` |
| 点「贴纸 / 画中画」 | 主轨 13→12、贴纸轨 2→3，状态条「已挪到「贴纸·画中画」轨…」，撤销可用 |
| 长按文字卡 | 弹 `💬 字幕轨`（文字不会给贴纸轨） |
| 快速竖划（不等长按） | 不弹窗，竖划仍归页面滚动 |
| 快速横拖（不等长按） | 不弹窗，片段照旧改时间 |
| 页面已滚 243px 时长按 | 弹窗自动滚回视野、完整可见（y=41） |
| 桌面回归 | 1440×1000 自检 22/22；鼠标竖拖换轨的原逻辑未动 |

> 遗留：手机端其余 3 项（按钮 22–32px 偏小、8 个入口收进 ⋮、用法说明被 `html.ave-immersive` 的
> `!important` 藏死）按需求**未优化**。

### 9.1 改为「长按抓起 → 直接拖到目标轨」（2026-10-01，用户要求：不要弹菜单）

**诉求**：手机上是「按住片段并拖动就能换轨」，不接受「长按 → 弹目标轨列表 → 再点一下」。

**关键改动**：把卡片的 `touch-action` 从 `pan-y` 改成 **`none`**。
§9 里「拦不住」的结论只对 `pan-y` 成立（`touch-action` 在 `touchstart` 就锁定，浏览器接管后必发
`pointercancel`）；换成 `none` 后浏览器根本不会碰这个手势，长按后继续竖拖的 `pointermove` 一切正常。

- 手机端 CSS：`.ave-clip,.ave-aclip,.ave-sub,.ave-ov,.ave-layit{touch-action:none;}`；
  **`.ave-rows` / 标尺 / 轨道名仍是 `pan-y`**（空白区、标尺、gutter 上的竖划照旧是原生滚动）。
- 卡片上的竖划由 JS 代滚：新增 `vertScroller()`（就近找纵向可滚动祖先，找不到就用 `document.scrollingElement`）、
  `pageScrollBy()`、`flingPage()`（松手惯性，0.94 衰减，<0.03 px/ms 停）。起点落在卡片上的竖划
  （`drag.kind = "pagescroll"`）1:1 跟手，双指竖向（`two.axis === "y"`）也走同一条路。
- 长按 0.3s **不再弹列表**，改为把 `drag.grab = true` + 卡片描边抬起 + 状态条
  「已抓起：上下拖到别的轨道，左右拖改时间」；`openTrackChooser()` 已删除。
- `dropHighlight()` / `pointerup` 的竖向判定由 `ev.pointerType !== "touch"` 改成
  `pointerType !== "touch" || drag.grab`：触摸**必须先抓起**才认竖向位移（长按前竖划 = 滚页面）。
- 抓起的适用卡片扩大到 `GRAB_KINDS = ["move", "audio-move", "sub-move", "ov-move", "lay-move"]`
  （与鼠标端本来就能跨轨的种类对齐）；`audio-move` 补了 `y0/movedY` 并开始调 `dropHighlight()`。
- 提示文案：「上下拖=换轨（手机上按住片段再上下拖）」；缓存版本 `album_editor.js?v=20261001aa`
  （`album.html` / `video_editor.html` 两处）。

**验证**（`node --check` OK；无头 Chromium 1000×1000 打开 `/video-editor`，用合成 PointerEvent
`pointerType:"touch"` 走完整链路 —— 逻辑全在 pointer 事件里，不在 touch 事件里，所以这套能测）：

| 用例 | 结果 |
|---|---|
| 长按 380ms 后卡片 | 内联描边 `0 0 0 2px rgb(255,143,168)` + 状态条「已抓起：上下拖到别的轨道，左右拖改时间」 |
| 抓起后竖拖到「贴纸·画中画」行 | 拖动中目标行高亮 `dropt = ["L1"]`；松手后片段数 2→1、贴纸轨项 0→1，状态条「已挪到「贴纸·画中画」（贴纸轨）」 |
| 快速竖划（不等长按，40px） | 不走换轨：`.ave-wrap` 的 `scrollTop` 40→0（自己代滚的页面滚动生效） |
| 快速横拖 40px | 卡片 `left` 0px→40px（照旧改时间），滚动量 0 |
| 加载期控制台 | 无新增报错（仅 4 条既有 `depot_only` 缩略图 404，见 §8.3） |

**已知取舍**：卡片起手竖划由 JS 代滚，只有 0.94 衰减的简易惯性，观感比原生滚动略"短"；
换来的是长按抓起后竖拖不再被浏览器抢走手势。真机（390×844 触摸）手感仍建议顺手过一遍。

---

## 10. 悬浮小窗预览「不生效」定位与修复（2026-09-30 增补）

**现象**：主预览滚出屏幕后，应该浮出的「小窗预览」看不见。

**排查**：机制本身是好的 —— `#aveCanvas` 用 `IntersectionObserver`（阈值 0.15/0.35/0.6/1）盯着，
`fltRatio < 0.35` 就浮。实测（390×844）可见率 0.229 → `.ave-float` `display:block` 且画布已绘制
（`getImageData` 非零）。所以"没浮出来"不成立。

**真正原因**：小窗位置持久化在 `localStorage.aveFloatPos`，但**恢复时按原值直接写 `left/top`，不按当前视口夹取**。
宽屏那次拖到右下角会存成 `[920,481]`；换到 390 宽的手机上算出来 `left:920px` →
小窗**整块落在屏幕外**，看着就像"没生效"。实测同一份 `localStorage`：

| | 修复前 | 修复后 |
|---|---|---|
| `aveFloatPos` | `[920,481]` | `[920,481]`（未改存储） |
| 小窗 rect（390 宽视口） | `left=920` → 不可见 | `left=228` → **完整可见** |

**修复**（`static/album_editor.js`）：
- `ensureFloat()` 恢复位置时先按当前视口夹一次（宽用 `FLT_CSS_W`，高用视口高兜底）。
- 新增 `clampFloatPos()`：用**真实尺寸**（`offsetWidth/offsetHeight`）再夹一次；
  在 `syncFloat()` 里 `paintFloat()` 之后调用（画完才有真实高度）。
- 监听 `resize` / `orientationchange` → 转屏、拉窗口后也会把小窗拉回屏幕内。
- 缓存版本 `album_editor.js?v=` 提到 **`20260930ab`**。

**验证**：

| 用例 | 结果 |
|---|---|
| 坏坐标 `[920,481]` + 390 宽视口，滚到预览出屏 | 小窗 `left 920 → 228`，完整可见 |
| 拖小窗到右下角 | 位置夹在视口内（x 上限 228 = 390−162），存成 `[228,721]` |
| 滚回顶部（预览可见） | 小窗 `display:none`（正确收起） |
| 视口 844 → 420 高（转屏/拉窗口） | 小窗自动 `top 721 → 260`，完整可见（resize 夹取生效） |
| 相册页浮层编辑器（`page:false`，滚的是 `.ave-wrap`） | 可见率 0 → 小窗浮出且在屏内；可见率 0.769 → 不浮（阈值正确） |

**生效方式**：静态 JS + Jinja 模板，运行中实例**已直接生效**（实测 `/video-editor` 返回
`?v=20260930ab`、`/static/album_editor.js` 含新函数），无需重启。

### 10.1 部署到远端生产机（2026-09-30 已执行）

部署代码文件是**现成的**：`rez-package-source/l_repo_sync_gui/999.0/deploy_l_wchat.py`
（`l_repo_sync_gui` 的"直推部署"，不走 git：base64 上传 → sha256 全量校验 → 按映射杀端口重拉）。
本次改动的 4 个文件**本来就在它的显式清单里**，无需改清单：

| 改动文件 | 清单位置 |
|---|---|
| `999.0/src/l_WChat/static/album_editor.js` | `deploy_l_wchat.py:66` |
| `999.0/src/l_WChat/templates/video_editor.html` | `deploy_l_wchat.py:57` |
| `999.0/src/l_WChat/templates/album.html` | `deploy_l_wchat.py:37` |
| `999.0/src/l_WChat/templates/base.html` | `deploy_l_wchat.py:38` |

```cmd
:: 只推 l_WChat（相册线）→ 重启 1234 l_wchat_backend；带 --with-netdisk 才连带 1028
"<trayapp>\wuwo\wuwor.bat" l_repo_sync_gui -- python "<trayapp>\rez-package-source\l_repo_sync_gui\999.0\deploy_l_wchat.py"
```

**执行结果**：

```
[deploy] 探活 OK server_id=dc3c2df6 auth_required=True inflight=8
[deploy] ==== l_WChat: 上传 44 个文件 → <远端>/rez-package-source/l_WChat
[deploy] l_WChat: 上传 pushed=44 skipped=0 errs=0
[deploy] l_WChat: 校验 checked=44 mismatch=0
[deploy] l_WChat: 重启（:1234 l_wchat_backend）…
[deploy] l_WChat: up=True killed=[] detail= | CARD_RESTART l_wchat_backend started=True
        log=<远端>D:\Temp\Log\rez_pkg_log\l_WChat\l_WChat_l_wchat_backend_20260930.log
[deploy] 部署结束 rc=0
```

**远端回验**（公网入口是 443 HTTPS + 自签证书，要 `-k`；`http://` 与 `:8080` 都会 404）：

| 请求 | 结果 |
|---|---|
| `curl -k https://121.196.144.88/l_wchat/video-editor` | 引用 `album_editor.js?v=20260930ab` |
| `curl -k https://121.196.144.88/l_wchat/static/album_editor.js?v=20260930ab` | 含 `clampFloatPos` / `openTrackChooser`，379620 字节 |
| `curl -k https://121.196.144.88/l_wchat/api/album/views` | **200** |

> 说明：本次首次部署时该脚本还是"全量重传 + 没有 `--dry`"，44 个文件（含 `apk/baoma.apk`、
> `mediabunny.mjs` 等大件）全部重传。同日已补齐：`--dry` / `--no-restart` / `--force-restart` /
> `skip_unchanged` 内容探测 / 无变更不重启 / 重启后业务端点探活 —— 见 `l_repo_sync_gui/README.md`。

### 10.2 本轮部署（2026-10-01，§9.1 的换轨改动）

**踩到的坑（务必先看）**：`deploy_l_wchat.py` 的远端树根是**照本机 `__file__` 推导**的
（"按标记目录运行期推导，不写死盘符"）。本机开发树在 `E:\lugwit\trayapp`，而**远端那棵树在
`D:\TD_Depot\Software\Lugwit_syncPlug\lugwit_insapp\trayapp`** —— 远端连 E: 盘都没有。
照默认跑 `--dry` 会报 **44 个全 missing**（指向远端不存在的 `E:\...`），真推就是往错的树写文件。

**修法**：`deploy_l_wchat.py` 顶部加了两个可覆盖入口（默认仍是推导值，不改变原语义）：

```cmd
set DEPLOY_REMOTE_ROOT=D:\TD_Depot\Software\Lugwit_syncPlug\lugwit_insapp\trayapp\rez-package-source
set DEPLOY_WUWO_DIR=D:\TD_Depot\Software\Lugwit_syncPlug\lugwit_insapp\trayapp\wuwo
"<trayapp>\wuwo\wuwor.bat" l_repo_sync_gui -- python "<trayapp>\rez-package-source\l_repo_sync_gui\999.0\deploy_l_wchat.py"
```

**执行结果**（先 `--dry` 确认再推；本轮只动 3 个文件：`static/album_editor.js`、
`templates/album.html`、`templates/video_editor.html`）：

```
[deploy] --dry：远端已一致 41 / 需要上传 3 —— mismatch=3（正是那三个文件）
[deploy] l_WChat: 上传 written=3 skipped=41 errs=0
[deploy] l_WChat: 校验 checked=44 mismatch=0
[deploy] l_WChat: up=True killed=['14740'] | CARD_RESTART l_wchat_backend started=True
[deploy] l_WChat: 远端 :1234/api/album/views → HTTP 200
[deploy] 部署结束 rc=0
```

**远端回验**（远端本机 `127.0.0.1:1234`，走 `/execute` 读回）：页面 200 且引用
`album_editor.js?v=20261001aa`；JS 200 且含 `pagescroll` / `unGrab` /
`GRAB_KINDS = ["move", "audio-move"…` / `touch-action:none` / 「已抓起：上下拖到别的轨道」，
**不含** `openTrackChooser`（弹列表方案已删干净）。

### 10.3 抓起动画 + 实时跟手（2026-10-01，用户要求：要有动画示意、片段跟手）

同一份 `album_editor.js`，缓存版本 `?v=20261001ac`。三件事：

| 目的 | 做法 | 坑 |
|---|---|---|
| 抬起 | 抓起时加 `.grabbing`（描边 + 大投影 + `scale(1.06)`，`.14s` 过渡） | 同时要给**整行**加 `.ave-grabrow{z-index:6}`，只抬卡片的话卡片下移会被后面的兄弟行盖住 |
| 跟手 | `dropHighlight()` 里 `transform: translateY(手指位移) scale(1.04)` | 只跟竖向：横向已被「改时间」（`setCardStart` 改 `left`）带着走，再 translate 就是双倍位移 |
| 摸到目标轨 | `.grabbing{pointer-events:none}` | **最关键**：浮起的卡片正压在手指下，不关掉它 `elementFromPoint` 永远命中自己那一行 → 换轨判定全落空 |
| 不滞后 | 抬起那下给过渡，手指一移动就换 `transition:none` | 跟手阶段留过渡，卡片会追不上手指 |
| 松手回位 | FLIP：记下位移，`renderAll()` 重建后把**新**节点放回松手位再滑回本行（`.18s`） | 松手时卡片已被重建，动画做在旧节点上等于没做；换轨成功的节点已不在原轨，`cardNode` 找不到 → 自然不跑 |
| 落轨示意 | `flashDrop(to)` 让目标行亮一下（`.26s`） | 换轨后 `sel = null`、节点重建，指不了具体卡片，亮行最稳 |

**实测**（无头 1000×1000 + 合成 `pointerType:"touch"` 指针事件）：

| 用例 | 结果 |
|---|---|
| 按住 380ms | `grabbing` + `scale(1.06)` + 行 `ave-grabrow` + 计算样式 `pointer-events:none` |
| 竖拖 4 步 | `translateY(12.25/24.5/36.75/49px)` 逐帧跟上手指；>24px 后目标行 `dropt` |
| 松手落贴纸轨 | 主轨 2→1、贴纸轨 0→1；行闪一下后 `dropt` 归零 |
| 抓起后原地松手 | 无位移、无残留 class/transform |
| 抓起竖拖后不换轨松手 | 新节点先停松手位（`transition:none`）→ `.18s` 滑回本行 → 清理干净 |
| 快速横拖（不等长按） | 不抬起、无 transform（照旧只改时间） |

**部署**（2026-10-01，`deploy_l_wchat.py` + 上节的 `DEPLOY_REMOTE_ROOT` 覆盖）：
`written=3`（JS + 两个模板）→ `checked=44 mismatch=0` → 重启 :1234（killed=[12468]）→
`/api/album/views` 200，rc=0。远端回验：页面引用 `?v=20261001ac`，公网 JS 与本地**sha256 全等**。
> 注：同文件里另一个会话的 `fmtT` 时间码改动经用户确认后随本次一起上线。

### 10.4 时间轴手感：裁剪把手可见性 + 空隙块（虚线框）可选中带参数（2026-10-01）

缓存版本 `?v=20261001ad`（**尚未部署**）。

**① 两端裁剪把手**：原来是一条 `8px`、白 18% 的中缝，压在深色卡面/照片上基本看不见。改为
`11px`（手机 14px）圆角竖条 + 白 26→46% 渐变 + 1px 白 50% 内描边，`::after` 画两根亮握痕；
悬停变主题粉；按住拖动时 `pointerdown` 统一加 `.trimming`（覆盖 `dur / *-dur / trimin / lay-dur`，
`layit` 那条分支自己 `return` 所以单独标），松手/`pointercancel` 由 `clearTrim()` 收。
> 现状：只有视频片段两端都有把手（左=改入点）；照片/文字卡只有右把手；`.ave-aclip`（配乐）没有把手。

**② 空隙块可选 + 参数 + 三种拖拽**（用户要求：空隙也要能选、有参数、勾「固定长度」后拖前边缘带着后面走）

| 项 | 实现 |
|---|---|
| 选中 | `sel = {type:"gap", id:<所属片段 id>}`（空隙没有自己的 id，挂在前一段之后的那个片段上）；**只就地改 `.sel` class + 刷面板，不整树重建** —— 重建会把"双击合上"的两次点击拆到两个节点上 |
| 参数 | `空隙时长(s)`（数字输入，改 `c.gap`，>0 时强制 `transition = "cut"`）、`固定长度`（勾选，新字段 `c.gapFixed`，已进 `projectJson()` 白名单）、`合上空隙`按钮（= 双击）；另附"上一段能拉到多少秒"的提示 |
| 拖拽① 不勾固定长度 | 拖 = 直接改空隙时长（上一段不动，块只变宽，后面节点平移） |
| 拖拽② 勾了 + 左边缘 | 拉伸/回缩**上一段素材的出点**（视频 `trimOut` 夹在 `trimIn+0.2 … srcDur`；图片/文字 `dur` 夹在 `0.4 … 30`），空隙时长锁死，空隙块及其后所有节点跟着 Δ 平移 |
| 拖拽③ 勾了 + 中/右 | **整块后移**：Δ 记到"上一段之前的空隙"里（`prev.gap += Δ`，夹 ≥0），上一段及其后全部一起搬；本空隙时长不变。空隙起点粘在上一段末尾，不让上一段让位就推不动它 —— 这是模型决定的，不是取巧 |
| 首空隙（`ci === 0`） | 前面没有片段可让位 → 勾了固定长度也按①处理，面板里给出说明 |
| 拖拽期间 | 跟其它拖拽一样走**局部样式平移**（整树重建会每帧重算音频波形 → 卡）；松手才 `saveProject() + renderAll()`；只点一下（没位移）不重排 |

**实测**（无头 1000×1000，合成鼠标指针事件，`pxPerSec = 46`）：

| 用例 | 结果 |
|---|---|
| 点虚线框 | `.sel` 上身；面板出 `空隙时长=1.50` / `固定长度` 未勾 / `合上空隙` 按钮 |
| 输入 2.5 秒 | `c2.gap 1.5→2.5`，块宽 69→115px，位置不动 |
| 勾固定长度 | `c2.gapFixed=true`，块加 `.fix`（虚线转实线） |
| 勾了 + 中/右拖 +1s | `c1.gap 0→1`、`c2.gap` 保持 2.5、总时长 +1s，状态「已整块后移 1.00s，空隙保持 2.50s」 |
| 勾了 + 左边缘拖 +1s | `c1.dur 3→4`（图片被拉长，卡片 138→184px 且左边不动）、`c2.gap` 保持 2.5、块随之右移 46px，状态「已拉伸上一段素材到 4.00s，空隙保持 2.50s」 |
| 不勾 + 拖 −1s | 空隙时长 2.5→1.5（按①） |
| 刷新页面 | `gapFixed` 落盘并在 `parseProjectJson` 后仍生效（块保持 `.fix`）；控制台无新增报错 |

### 10.5 抬起/跟手推广到"任何一次拖动"（2026-10-01，用户复述需求：手机上拖拽也要跟手）

§10.3 只把抬起/跟手做在**长按抓起**那条路上（`drag.grab`），用户反馈手机上普通拖拽没跟上。本轮改成：
**任何一张卡被拖动都抬起 + 跟手**，长按抓起只是"多一圈粉边 + 允许竖拖换轨"。

| 项 | 实现 |
|---|---|
| 抬起 | `liftCard(card, strong)`：加 `.ave-lift`（白 40% 描边 + `0 12px 26px` 投影 + `scale(1.06)`，`.14s` 过渡），`strong` 时再补 `.ave-grab`（粉圈，长按抓起用）；顺手把整行 z-index 抬起（`.ave-grabrow`） |
| 触发点 | `dropHighlight()` 里 `!drag.lifted` 就抬（该函数只在 `move / ov-move / sub-move / lay-move / audio-move` 里调，裁剪把手的 `*-dur / trimin` 不抬 —— 拉边缘时卡片不该飘） |
| 跟手 | 同一处 `translateY(movedY) scale(1.04)`；`.ave-lift` 自带 `pointer-events:none`（浮起后 `elementFromPoint` 才能摸到下面的行，否则换轨判定全落空） |
| 收场 | 仍由 `unGrab()` 统一收（`.ave-lift` / `.ave-grab` / 行 `.ave-grabrow` / inline transform），FLIP 滑回逻辑不变 |
| iOS 长按被系统吃掉 | 卡片上加 `-webkit-touch-callout:none;-webkit-user-select:none`（长按 callout / 选择放大镜会直接吞掉手势，是"手机上不跟手"的常见元凶） |

**实测**（无头 1000×1000，合成指针事件；`?v=20261001ae`）：

| 用例 | 结果 |
|---|---|
| 鼠标拖（斜向 12/9 px 四步） | 每步 `ave-lift` + `translateY(9/18/27/36px) scale(1.04)`、`left 12/24/36/48px`、行 `ave-grabrow`；目标行 `dropt=L1`；松手移轨成功、class/transform 全清 |
| 触摸普通横拖（不等长按） | `ave-lift`（无 `.ave-grab`）、`translateY(0) scale(1.04)`、`left` 跟手；松手清理干净 |
| 触摸快竖划 | **不抬**；`.ave-wrap` 滚动 80→32→0（自己代滚 + 惯性），与"抬起"互不干扰 |
| 触摸长按 400ms 后斜拖 | `ave-lift ave-grab` + `scale(1.06)`；拖动中 `translateY(24px) scale(1.04)`、`left` 47.84→71.84；松手 `.ave-lift`/`.ave-grabrow` 归零 |

**部署**：同 §10.2 通道（`DEPLOY_REMOTE_ROOT` 覆盖），`written=3` → `checked=44 mismatch=0` → 重启 :1234 → 探活 200，rc=0；
公网 `?v=20261001ae` 页面 200 且 JS 与本地 **sha256 全等**（含本轮把手 / 空隙参数 / 抬起跟手三件事）。

### 10.6 工具条压缩：按钮变小 + 去掉 emoji + 行距归零（2026-10-01）

用户反馈（附截图）：按钮再小一点、不要按钮上的表情图标、每行不要留空隙。`?v=20261001af`。

| 项 | 改动 |
|---|---|
| 按钮尺寸 | `.ave-btn` 桌面 `padding:7px 11px→3px 7px`、`font-size:12→11px`、`border-radius:9→6px`；手机 `min-height:32→24px`、`padding:5px 8px→3px 6px`、`font-size:11→10.5px`；行尾 `⋮`（`.ave-more`）`34→24px`（手机 22px） |
| 去 emoji | 工具条 9 个按钮：`☁ 存云端`/`☁ 云草稿`/`▶ 预览`/`🎬 生成并上传`/`⛶ 全屏`/`📐 模板`/`🎤 自动字幕`/`📝 图文成片`/`✂️ 裁剪片段`/`🎬 AI 视频`/`🖼 AI 图片`/`🎵 分析拍点`/`🧹 清理损坏素材` → 只留文字；播放/全屏按钮的**动态文案**（`d.play`/`d.full.textContent`）同步去图标。保留 `+`（新增语义）、`↶ ↷`（撤销/重做）、`⋮`（更多）；**对话框 / 选择器里的图标未动**，标题 `✂️ 视频相册编辑器` 也未动 |
| 行距归零 | `.ave-head` / `.ave-bar` / `.ave-tools` / `.ave-tl` 的 `margin-bottom` 桌面 `10→0`、手机 `6/5→0`；行内 `gap` 桌面 `10/8→6/4`、手机 `3px`；`.ave-stage{margin-bottom:8→4}`（手机 2） |
| 连带修 | 按旧标签索引的说明字典 `TIPS`（`tipAll()` 用按钮文字查表）会失配 → 键名同步成新标签，并补 `+ 视频（相册里的）`（本来就漏） |

**实测**（`Network.setCacheDisabled` 强制取新 JS，无头 + CDP 设备仿真）：

| 档位 | 结果 |
|---|---|
| 390×844（手机档，mobile:true） | 按钮高 **24px**；`.ave-head/.ave-head-acts/.ave-tools` 的 `margin-bottom` 全 **0px**、`gap` **3px**；`documentElement.scrollWidth = 390`（无横向溢出）；工具行可见 `+ 轨道 / + 照片（用已勾选的）/ + 视频（相册里的）/ ⋮`，其余自动收进 ⋮（⋮ 的右边界 312 < 390，在屏内） |
| 740×900 | fitRow 收放正常：可见到 `裁剪片段`(right=656) + ⋮(right=681)、`clientWidth=722` |
| 文案/提示 | 工具条 25 个按钮文字无 emoji；`存云端/云草稿/预览/生成并上传/模板/裁剪片段/分析拍点` 的 `title` 都从新键名取到（不再退化成"自己当提示"） |

**部署**：`written=3` → `checked=44 mismatch=0` → 重启 :1234（killed=[19448]）→ 探活 200，rc=0；
公网 `?v=20261001af` 页面 200、JS 与本地 **sha256 全等**，且断言 `id="aveGen">生成并上传` 等新标签都在。

#### 10.6.1 补：每行右侧留白（用户第二次反馈"还是存在很多空隙"，附截图圈的就是行尾空白）

上一条把"行距"清零了，但用户圈出来的是**每行右侧那块没被按钮占满的空白**。改成按钮铺满整行：

| 项 | 改动 |
|---|---|
| 按钮铺满 | `.ave-head-acts .ave-btn:not(.ave-more)` / `.ave-tools .ave-btn:not(.ave-more)` → `flex:1 1 auto;min-width:0`；`.ave-head-cfg` 里的 `input/select/按钮` 同样放开（`.ave-head-cfg *{flex:0 0 auto}` 之后加一条覆盖）；`#aveTitle` 由 `0 0 84px` 改 `1 1 84px`；时间轴的 `.ave-zoombar{flex:1 1 auto}` 让滑块吸收剩余宽度 |
| ⋮ 不参与拉伸 | 选择器都用 `:not(.ave-more)` —— `.ave-tools .ave-btn` 的权重大于 `.ave-more{flex:0 0 auto}`，不排除的话行尾 ⋮ 会被撑宽 |
| **fitRow 测量口径**（关键坑） | 按钮改成 flex 铺满后，行内宽度**恒等于行宽**，原 `usedW()` 一路判成"放不下"→ 会把按钮全收进 ⋮。改成测量前先把可见子项临时 `style.flex="0 0 auto"` 量自然宽，量完还原 |

**实测**（`Network.setCacheDisabled`，CDP 设备仿真；`tailGap = 行右边界 − 最右可见子项右边界`）：

| 档位 | 结果 |
|---|---|
| 390×844 | 五行（head / head-cfg / head-acts / tools / tl-head）**tailGap 全 0px**；`scrollWidth-window.innerWidth=0`（无横向溢出）；acts 8 个可见（↶31 ↷31 存云端54 云草稿54 预览43 生成并上传75 全屏43 + ⋮22 = 374 ≈ 行宽 372）、tools 4 个（68/142/131 + ⋮22）、隐藏 13 个进 ⋮ 菜单 |
| 740×900 | 同样 tailGap 全 0；acts 9 个、tools 11 个可见 |

（首轮误判说明：用户第一句"每行不要留空隙"我理解成"行距"，只清了 margin；第二句带截图才看清是**行尾留白**。
两轮都保留着 —— 行距 0 + 按钮铺满，两个方向都不留白。）

### 10.7 叠加轨卡片贴满 + 剪辑页设置路由（全屏策略）（2026-10-01）

`?v=20261001ag`。用户两件事：① 自建视频轨上的片段上下有空隙、跟主轨不一致；② 想要"是否全屏"可设置，
入口放在最上一行的 ⋮ 里，打开一个设置路由，里面有"全屏策略"下拉（自动 / 全屏 / 不全屏）。

**① 空隙的成因（实测数字，740 宽）**

| | 卡片高 | 距行顶 | 行底余量 | 行高 |
|---|---|---|---|---|
| 主轨片段 `.ave-clip` | 46px | 0 | −2px（比行还高 2px） | 44 |
| 叠加轨素材 `.ave-layit`（改前） | **28px** | **7px** | **9px** | 44 |

`.ave-layit` 原本是 `top:calc(7px*rowk); height:calc(28px*rowk)`（细条样式），主轨是 `top:0; height:42px`（移动端 46px）——
自建轨（视频/贴纸/字幕/音频轨都复用 `.ave-layit`）因此上下各留一圈。改法：`.ave-layit` 统一成
`top:0;height:calc(42px*rowk)`，并把移动端那组高度规则加上它：`.ave-clip,.ave-aclip,.ave-sub,.ave-ov,.ave-layit{height:calc(46px*rowk)}`。
改后实测与主轨**逐项一致**（46 / 0 / −2 / 44）。`gapblk`（空隙块）与 `fxbar`（特效条）保持细条样式（它们本来就是"标记"，不是素材卡）。

**② 设置路由 + 全屏策略**

| 件 | 内容 |
|---|---|
| 路由 | `api/pages.py` 新增 `GET /video-editor/settings` → 新模板 `templates/video_editor_settings.html` |
| 模板 | 一个卡片：`全屏策略` 下拉（`auto` 自动（推荐）/ `full` 全屏 / `off` 不全屏）+ 每个值一句说明 + "← 回剪辑页"（走 `__wurl` 带 nginx 前缀） |
| 存储 | `localStorage["ave_fullscreen_policy"]`（**故意不放服务端**：这是"这台设备上看着舒不舒服"的偏好，跨设备同步会互相打扰）；读写都包 try（隐私模式不炸） |
| 入口 | 剪辑页最上一行（`.ave-head-cfg`）里加一个「设置」按钮：**放不下时会被 fitRow 自动收进这一行的 ⋮**（实测 740 档该 ⋮ 菜单 = `["设置","完成"]`），桌面端（>768 宽不折叠）直接看得见（38px） |
| 生效 | `video_editor.html` 的 `boot()` 读策略：`auto` = 现在这样（先沉浸，第一次真手势顺势要系统全屏）；`full` = 一进来就立刻尝试系统全屏（被拒静默保持沉浸）；`off` = 不沉浸也不要全屏（普通网页样式）。`⛶ 全屏` 手动按钮不受影响 |

**实测**：策略 `off` → `<html>` 无 `ave-immersive`；`auto` / `full` → 有。设置页 200，下拉三项齐全，
改选写入 localStorage 且说明文字同步。`album.html`（相册页浮层编辑器）不受策略影响 —— 它本来就不进沉浸。

**部署（重要坑）**：`deploy_l_wchat.py` 的文件清单是**手写**的，新模板必须补进去，否则远端没有这个文件、
`/video-editor/settings` 直接 500。已加 `999.0/src/l_WChat/templates/video_editor_settings.html`，
清单一共 45 个文件。本轮 dry：`44 一致 / 需上传 5`（4 mismatch + 1 **missing** = 新模板）；
实推 `written=5` → `checked=45 mismatch=0` → 重启 :1234（killed=[15404]）→ 探活 200，rc=0。
公网复验：`/l_wchat/video-editor/settings` 200 且含三项下拉；`/l_wchat/video-editor` 引用 `?v=20261001ag` 且含 `ave_fullscreen_policy`；
JS 与本地 **sha256 全等**。

### 10.8 「更多操作」里可以指定哪些按钮常显（2026-10-01）

需求：哪些按钮留在行里、哪些收进 ⋮ 原来是 fitRow 自动算的，用户要能自己定。`?v=20261001ah`。

| 项 | 实现 |
|---|---|
| 面板 | `openRowMenu()` 从"只列藏起来的"改成**列出这一行的所有项**（跳过 `.ave-more` 自己与 `.ave-pri` 主操作），每行 `.ave-menurow` = 左边原来的按钮/输入副本 + 右边「常显」开关；面板顶部一句说明 |
| 开关 | `.ave-pin`（未打＝`常显`，已打＝`常显 ✓`，`.on` 类换主题色）；点一下写盘 + `fitRow(row)` 立刻重排（面板挡着行，点「完成」就能看到结果） |
| 存储 | `localStorage["ave_rowPins"] = { "ave-tools": ["aveAiVideo"], ... }`：**按行记**（行的第一个类名作 key：`ave-head-cfg` / `ave-head-acts` / `ave-tools`）。项 key 用元素 `id` 或内含控件的 `id`（`#aveRes`、`#aveDuck` 这种），**不能用按钮文字**——"关闭"会变"← 返回相册" |
| fitRow 规则 | 先收**没打勾**的（从后往前），实在放不下才动打了勾的（最后兜底，保证行永不溢出）；`.ave-pri`（预览/生成并上传）照旧永不收 |

**实测**（390×844，合成点击）：

| 用例 | 结果 |
|---|---|
| 打开工具行 ⋮ | 14 行（15 项里排除主操作 `+ 轨道`），每行都有「常显」；顶部说明在位；初始 0 个打勾 |
| 给「AI 视频」打勾 | `localStorage = {"ave-tools":["aveAiVideo"]}`；开关变「常显 ✓」；行内立刻变成 `+ 轨道 / + 照片（用已勾选的）/ + 视频（相册里的）/ AI 视频 / ⋮` |
| 刷新页面 | 打勾仍生效（AI 视频还在行里） |
| 取消打勾 | 存储回到 `{}`，AI 视频回到 ⋮ |
| 横向溢出 | `scrollWidth - innerWidth = 0`（收放后行仍不溢出） |
| 部署 | dry 3 mismatch → `written=3` → `checked=45 mismatch=0` → 重启 :1234（killed=[15716]）→ 探活 200；公网 JS 与本地 sha256 全等 |

> 注：`fitRow` 的循环只会"收"，不会把已收的再放回来 —— 所以给某项打勾后，是"把没打勾的收掉给它腾地方"，
> 不会顺便把别的没打勾的也塞回行里（观感上：打勾 = 抢占行内位置）。

### 10.9 反代前缀补了两次：设置页"回剪辑页"跳到 /l_wchat/l_wchat/...（2026-10-01）

用户报：`http://127.0.0.1:8080/l_wchat/video-editor/settings` 上点"回剪辑页"落到
`http://127.0.0.1:8080/l_wchat/l_wchat/video-editor`。

**根因（两处叠加）**
1. `base.html` 末尾那段"渲染后的绝对路径统一补前缀"里，`fixAttr()` 是**无条件补**的：
   `if (v.charAt(0) === "/" && v.charAt(1) !== "/") el.setAttribute(attr, p + v)` ——
   它不看是否已带前缀（同文件里 MutationObserver 用的 `maybeFix()` 就带 `v.indexOf(p) !== 0` 这道判断）。
2. 我写设置页时又手动 `vsBack.href = window.__wurl("/video-editor")` 先补了一次 → 两遍 → `/l_wchat/l_wchat/video-editor`。

**直接 1234 端口看不出来**：那时 `__WCHAT_PREFIX = ""`，`fixAttr` 一开头就 `return` 了 —— 所以只有经 nginx `/l_wchat/` 访问才暴露。

**定位过程**（值得记）：远端 `__wurl('/video-editor')` 单独调用返回的是**对的** `/l_wchat/video-editor`，
但 `getAttribute('href')` 是双前缀。于是在页面脚本执行前注入钩子包住 `window.__wurl`，把它每次调用的
参数/返回/当时 pathname 都记下来 —— 日志显示我那次调用返回的是单前缀 ✓，说明**属性是被后续的遍历改的**，
顺线找到 `base.html` 的 `fixAttr`。

**修法**：① 设置页去掉那次多余的 `__wurl`（`href="/video-editor"` 交给 base.html 的遍历补，只补一次）；
② 把 `fixAttr()` 改成幂等（加 `v.indexOf(p) !== 0`，与 `maybeFix` 一致），这类"模板自己补过前缀"的坑不会再犯。
两个文件都已推（本轮 47 文件清单，`written=6` / `checked=47 mismatch=0` / 重启 :1234 / 探活 200），
并用部署通道读回远端文件内容确认：`fixAttr` 幂等 ✓、旧写法已无 ✓、设置页无手动 `__wurl` ✓、`href="/video-editor"` ✓。

> 期间本机与远端 WChat 都开始要登录（`/video-editor*` 一律 302 到 `/login`），与本改动无关；
> 因此这次没法匿名点进去复验，请登录后硬刷新设置页确认"回剪辑页"落到 `.../l_wchat/video-editor`。

### 10.10 「常显」开关改成反映真实状态（2026-10-01，用户反馈"没有反应目前的真实状态"）

用户截图：`↶ ↷ 存云端 云草稿 全屏` 明明都在行里，面板里却全显示未勾的「常显」。

**问题**：开关显示的是**我打过勾没**（pin 标志），不是**这一项现在到底在不在行里**。§10.8 的语义（"打勾=固定留行"）
跟用户的期待（"开关＝它现在是不是展开了"）不是一回事。`?v=20261001ao`。

**改法**

| 项 | 现在 |
|---|---|
| 开关状态 | 读 DOM 真实显隐：`src.style.display !== "none"` → 在行里就是「收起 ✓」（`.on`），在更多里就是「常显」 |
| 文案 | 按"点了会发生什么"写：在行里 →「收起」（收进更多）；在更多里 →「常显」（固定留到行里） |
| 存储 | `ave_rowPins` 改成**两集合** `{rowId:{show:[…],hide:[…]}}`（`show`=强制留行，`hide`=强制收起）；老格式（数组）读成 `show` 兼容 |
| fitRow 顺序 | ① 先收起 `hide` 点名的；② 还不够就从后往前收没打标的（跳过 `.ave-pri` 与 `show`）；③ 打了 `show` 也放得下才留，实在不行兜底收；④ **收完还有空位就回填**（把没被 `hide` 点名的放回来，能放几个放几个） |
| 点完必须重刷整排 | `syncPins()`：点一项会连带挤进/挤出一个，所有开关都要按**新的真实显隐**重画（否则又变成"不反映真实状态"） |

**实测**（本机要登录，`/static/*` 免登录 → 在同源页面里搭最小宿主 `#videoEditorRoot` + 加载三个静态 js 来测，
这样 localStorage 也是真的）：

| 用例 | 结果 |
|---|---|
| 初始（无标记） | 行内 `↶ ↷ 存云端 云草稿 全屏`，面板里这几项 = 「收起 ✓」，`← 返回相册`（在更多里）= 「常显」；`预览/生成并上传` 不在面板里（主操作） |
| 点 全屏 的「收起」 | 落盘 `{"ave-head-acts":{"show":[],"hide":["aveFull"]}}`；行里换回 `← 返回相册`（回填）；开关重刷成 `全屏=常显`、`← 返回相册=收起` |
| 再点 全屏 的「常显」 | 落盘 `{"show":["aveFull"],"hide":[]}`；行里换回 `全屏`，`← 返回相册=常显` |
| 冷启动重放 | 预置 `{show:["aveFull"],hide:["aveCloudOpen"]}` → 行内 `云草稿` 被收起、`全屏` 留在行里；面板逐项与行内一致 |
| 部署 | `written=3` → `checked=47 mismatch=0` → 重启 :1234（killed=[1124]）→ 探活 200；公网 `?v=20261001ao` JS 与本地 sha256 全等 |

### 10.11 「常显」第三版：文案固定 + 高亮表示开关（2026-10-01，用户："点击没有反应，希望按钮永远显示常显，用高亮表示是否打开"）

`?v=20261001ap`。上一版（§10.10）把开关状态绑到"此刻在不在行里"，两个后果：
① 窄屏下把某项标上后若挤不下，fitRow 的兜底会立刻把它收起 → 看起来"点了没反应"；
② 文案在「收起 / 常显」之间跳，用户读不出"这是开关"。

**现在**：
| 项 | 现在 |
|---|---|
| 文案 | **恒为「常显」**（不再变「收起」） |
| 状态 | `.ave-pin.on` = 已常显。开着＝主题紫渐变 + 亮边 + 白字加粗，关着＝灰底灰字 —— 只靠高亮区分 |
| 语义 | 开 = 写进 `show`（固定留行，挤掉没常显的）；关 = 写进 `hide`（收进「更多」） |
| 反馈 | 点一下**立刻**高亮变化（不再依赖 DOM 显隐），`fitRow(row)` 立即重排（行被面板挡着，点「完成」看结果） |
| 挤不下时 | 标记仍保持高亮（那是"你的意图"），但 title 会写：「已常显，但这一行位置不够：先取消别的常显，或屏幕宽一点就会出现」 |
| 生效范围 | 标记只在窄屏（`innerWidth ≤ 768`）参与 fitRow —— 桌面本来就全展开、没有 ⋮，面板也进不去 |

**实测**：预置 `show:["aveFull"]` → 面板里「全屏」高亮、其余灰，title 分别为"已常显（点一下取消）"/"点一下常显：固定留在上面那一行"；
点「云草稿」的常显 → class 立刻变成 `on`（文案仍「常显」），落盘 `show:["aveFull","aveCloudOpen"]`；再点 → 取消高亮、落盘 `hide:["aveCloudOpen"]`。
部署 `written=3` → `checked=47 mismatch=0` → 重启 :1234（killed=[16248]）→ 探活 200；
公网 `?v=20261001ap` 的 JS 已含 `.ave-pin.on` 渐变、`refreshTips`、"位置不够"提示（sha 与本地不等的唯一原因是
**另一个会话在我推完后继续改了本地同一文件**，与本次改动无关）。
