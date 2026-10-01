# l_ltxvideo — 本机 LTX-Video 视频生成（纯 Rez）

> 目标：`wuwor l_ltxvideo -- l_ltxvideo_server` 起一个本机视频生成服务，`/healthz` 探活、
> `/generate` 出片；依赖由 **wuwo 装进共享货架**（`rez-package-3rd`），权重落在包 home。

---

## 1. 它是什么

| 层 | 内容 |
|---|---|
| Python 依赖 | `torch / torchvision / diffusers / transformers / accelerate / safetensors / imageio(_ffmpeg) / pillow`，全部写在 `package.py` 的 `requires`，wuwo 自动装 |
| 权重 | HF `Lightricks/LTX-Video`，**只拉 diffusers 根管线所需**（`allow_patterns`：`text_encoder/ transformer/ vae/ tokenizer/ scheduler/` + 小 json）≈ **27GB**（整仓 236GB，别整仓拉） |
| 落盘 | home：`L_LTXVIDEO_HOME` > `{包根}/deploy_home.txt` > `~/.lugwit/l_ltxvideo`；权重在 `<home>/weights`，产物在 `<home>/runtime/out` |
| 服务壳 | 本包 `src/l_ltxvideo/server.py`（Rez py3.12 跑），推理交 `worker.py` 子进程（同解释器），**单卡串行排队** |

> **home 阶梯末环的现状（2026-10-01）**：阶梯顺序不变，但本机这些权重**已不在** `~/.lugwit/` ——
> 已于 2026-10-01 剪切到 **`E:/lugwit_rez/homes/l_ltxvideo`**，由本包 `deploy_home.txt`（一行绝对路径）声明；
> 另可用 wuwo 注入的 `LUGWIT_SHARED_HOME`（= config 的 `l_shared_home_dir`，**盘符不存在时不注入**）。

HTTP 面：`GET /healthz`（轻量，不载模型）、`GET /state`、`POST /load`、`POST /generate` → `{task_id}`、`GET /tasks/{id}`、`GET /files/{name}`。

别名：`l_ltxvideo_server`（服务）/ `l_ltxvideo_setup`（拉权重+自检）/ `l_ltxvideo_doctor`（体检）/ `l_ltxvideo_demo`（试跑一段）。

---

## 2. 安装（从 l_model_hub 点一次「安装」）

`l_model_hub` 的「🖥 本机模型」页 → 卡片「⬇ 安装/更新」→ 后端依次做：
① 包不存在就先按模板生成 → ② wuwo 装依赖（共享货架）→ ③ 拉权重（带百分比）→ ④ 自检 + 注册服务卡片。

首装实测耗时：依赖（含 CUDA torch 3.4GB）+ 权重 27GB，视网速 20–60 分钟；**第二次装同族模型基本零成本**（torch/diffusers 复用）。

---

## 3. 踩过的坑（改 wuwo 前先读这里）

1. **Windows 的 torch 必须走 PyTorch 官方 CUDA 索引**：PyPI 与国内镜像上是 CPU 版（`torch-2.8.0-cp312-win_amd64.whl` 只有 241MB，`+cu128` 是 3.4GB）。
   → wuwo `auto_fetch_packages` 已内建：torch 系优先 `https://download.pytorch.org/whl/cu128`（`_CUDA_PKGS` / `_cuda_extra_index_args`）。
   → 但 **pip 的 CLI 索引参数会覆盖环境变量**，所以光设 `PIP_INDEX_URL` 不可靠；最终靠"预装 CUDA 版进货架"解决。
2. **rez-pip 处理不了 sympy 的相对路径 RECORD**：`[Errno 89] Don't know what to do with relative path in sympy-1.14.0.dist-info\RECORD`。一旦失败，wuwo 会换下一个索引 → **把 CUDA 轮子换成 CPU 轮子**。
   → 已在 wuwo 里让 **torch / torchvision / torchaudio 跳过 rez-pip**，直接走旧式 `pip install --target`。
3. **`requires` 里不要给 torch 写版本号**：wuwo 会把 rez 的 `name-x.y` 翻成 pip 的 `==x.y`，`torch==2.8` 是**不存在的精确版本**；而**连字符包名**（`imageio-ffmpeg`）会被按"名字-版本"拆成 `imageio==ffmpeg.*`。
   → 写法：`["torch", "torchvision", "imageio", "imageio_ffmpeg", …]`（不带版本、用下划线）。
4. **跨盘搬文件会 `WinError 17`**：pip 临时目录与货架不在同一个盘时，rez-pip 的搬迁直接失败（3.4GB 白下）。
   → 安装子进程的 `TEMP/TMP/TMPDIR` 指到**货架同盘**（hub 侧 `local_models._run_install` 已做，
     位置 = 货架父目录下的 `_temp/`，见 2026-09-30 迁盘）。
   → **装配侧（`auto_fetch_packages` 的 rez-pip / `pip --target`）还没做**，靠"torch 系跳过 rez-pip"
     ｜RECORD 清洗补丁绕开；补这段时要连 `TEMP/TMP/TMPDIR` 一起改。
5. **货架路径是 `config.yaml` 单一来源，别自己推算**（2026-09-30）：第三方货架已从 D:（机械盘，
   冷读 67-73MB/s）迁到 **E: NVMe**（Predator GM7000，写 2.0GB/s）——`packages.third_party`。
   历史上 `wuwo_rez` / `auto_fetch_packages` / `pywin32_bootstrap` 共 8 处写死
   `source_dir.parent / "rez-package-3rd"`，换盘后就是「装到老地方、解析在新地方」→ 反复重装。
   新写法：`afp.resolve_third_party_dir()`（env > config > 老默认值）。
   模型包（`l_wanvideo` 这种）**不单开货架**：它就是「源码包」形态，放仓库源码货架
   （`rez-package-source`）最自然、还能跟 git 走。
6. **任务状态不收敛**：wuwo 是多层启动器（`wuwo_rez.py` → `rez.exe` → python），孙进程会攥着 stdout 管道，读循环永远等不到 EOF。
   → hub 侧加了"看门线程"：`proc.wait()` 一返回就收尾并关管道（`_run_install._watch`）。
7. 显存：RTX 4080 16GB 上默认档 `768×512 / 97 帧 / 24 步`（720p 会 OOM）；`enable_model_cpu_offload()` 已开。
8. **`tiktoken` / `sentencepiece` 必须显式写进 requires**：T5 词表要用它们。以前这台机器是**从基础 Python 的 site-packages 蹭**（那里被人装过一堆杂包）；一旦清理基础环境就报
   `ValueError: tiktoken is required to read a tiktoken file`，或更具误导性的
   `ValueError: Error parsing line b'\x0e' in ...spiece.model`（看着像权重损坏，其实是缺库 —— 实测权重与 HF 逐字节一致）。
9. **transformers 5.x 用不了这个 T5 词表**：`transformers 5.17.0` 下无论 fast/slow 路径都报
   `ValueError: not enough values to unpack (expected 2, got 1)` → 落到 tiktoken 兜底 → 同一个"解析失败"。
   LTX 的 diffusers 集成是按 **transformers 4.x** 写的，装 4.x 即可。
10. `/healthz` 与"生成"**不能共用一把锁**：早期实现把整段生成放在同一个锁里，探针请求会全部挂死
    （端口在监听但 HTTP 超时 → 看起来像"服务没起"）。现在生成用独立 `_gen_lock`，`state()` 不加锁。
11. `callback_on_step_end` **必须返回 `callback_kwargs`**：返回 `None` 会让 diffusers 在 `None` 上
    `.pop(...)` → `AttributeError: 'NoneType' object has no attribute 'pop'`（进度回调就是干这个的，实测踩到）。
12. **帧类型不统一 → 画质静默降级**：LTX 的 `res.frames[0]` 是 PIL 图，**Wan 的却是 torch tensor**（GPU/bf16）。
    `_to_np()` 里直接 `np.asarray(tensor)` 抛 `OSError` → 被 `_encode()` 的兜底 `except` 吃掉 →
    退回 diffusers `export_to_video` 的**低码率默认档**；日志只有一句 `ffmpeg 直编失败(OSError)，回退默认编码`，
    任务还报 `succeeded`，很容易当成"生成成功只是模型不行"。
    → 实测 5s/832×480/81 帧：回退档 **0.37MB**（≈610kbps）vs crf14 **1.43MB**（≈2.4Mbps）。
    → 修法：`_to_np()` 先 `detach().float().cpu().numpy()`，再把 0..1 浮点乘 255 转 uint8。
    → 排查手法：同参数 dummy 帧单跑编码器能过 → 问题在输入帧类型，不在 ffmpeg。

---

## 4. 已知可用的依赖组合（2026-09-30 实测）

**别随便升级其中一个** —— 这几家会互相打架，实测踩了三轮：

| 包 | 版本 | 为什么是这个 |
|---|---|---|
| torch / torchvision | `2.8.0+cu128` / `0.23.0+cu128` | Windows 的 CUDA 轮子只在 PyTorch 官方索引；走 `pip install --target` 手装（见坑 1/2） |
| diffusers | **0.35.2** | 0.40 需要 `huggingface_hub>=1.0`（它 import `get_cached_repo_tree`），与 transformers 4.x 冲突；LTX 管线自 0.32 就有 |
| transformers | **4.57.6** | 5.17 解不了这个（合法的）T5 `spiece.model`；4.57 要求 `huggingface_hub<1.0`、`tokenizers<=0.23.0` |
| huggingface_hub | **0.36.2** | 满足 diffusers 0.35 与 transformers 4.57 的交集（`<1.0`） |
| tokenizers | **0.22.2** | PyPI 上没有 0.23.0（4.57 要 `>=0.22,<=0.23.0`） |
| sentencepiece / tiktoken | 最新 | T5 词表解析必需（否则报"spiece.model 解析失败"，看着像权重坏了） |

加载耗时参考（RTX 4080 16GB，26GB 权重在 C: 盘）：首次 `LTXPipeline.from_pretrained` 数分钟（T5 约 9.5GB 先进内存）。

## 5. 前端与对外入口

- 服务**自带单页 UI**：`GET /`（提示词 / 分辨率 / 帧数 / 步数 / fps / 进度条 / 出片后页内播放+下载 / 最近产物 `/files`）。
  页面里所有请求都是**相对路径**，所以直连 `http://127.0.0.1:8475/` 与经反代 `http://127.0.0.1:8080/ltxvideo/` 都能用。
- 对外暴露要在 `l_nginx` 加两处（照 `/indextts2` 抄）：
  - `conf/lugwit.conf` → `upstream lugwit_ltxvideo_backend { server 127.0.0.1:8475; keepalive 16; }`
  - `conf/routes.conf` → `location = /ltxvideo { return 302 /ltxvideo/; }` + `location /ltxvideo/ { proxy_pass http://lugwit_ltxvideo_backend/; }`
  - 生效：`wuwor l_nginx -- nginx_reload`
- 没有这两处时，`/ltxvideo/` 会落到兜底（主页）→ 看到 `{"detail":"Not Found"}`（很容易误判成"服务没起"）。

### 5.1 阶段与"预计剩余时间"

页面（和 `/tasks`）每条进度都带 `stage` + `eta_s`，**每个阶段给自己的剩余时间**——
进度百分比在阶段边界上会长时间不动（读 27GB 权重几分钟 / VAE 解码几十秒 / 编码十几秒），
只显示"已用 250s"用户没法判断还要等多久。

| 阶段 `stage` | 剩余时间怎么来 |
|---|---|
| `queue` | 前面还有任务 → **不给 ETA**（拿加载耗时冒充排队耗时就是骗人） |
| `load` | 上次实测加载耗时 − 已用（每 15s 一条心跳时更新） |
| `diffuse` | 本步实测均耗时 × 剩余步数（自校准，不用基线） |
| `decode` | 历史中位（最后一步回调里就切成"解码中"——那之后到出帧还有几十秒） |
| `encode` | 历史中位 |

实测值落在 `<home>/runtime/stage_eta.json`（`eta.py`：按 stage 存最近 5 次、读时取中位数）。
**没有历史返回 `None`**，页面显示"剩余：暂无参考（首次运行）"，不猜；
文件坏了/写不进去只影响提示，绝不影响生成。`/state.stage_eta` 给各阶段历史参考，页面下方
"阶段参考（历史中位）"那一行就是它。

**编码/帧类型坑**：`res.frames[0]` 在 LTX 是 PIL 图、在 Wan 是 torch tensor。`worker._to_np()`
必须处理 tensor（`detach().float().cpu().numpy()` + 0..1 浮点转 uint8），否则
`np.asarray(tensor)` 抛 OSError → 被 `_encode()` 兜底吃掉 → **静默退回低码率档**
（实测 5s/832×480: 0.37MB vs crf14 1.43MB）。

## 6. 验证

```cmd
:: 体检（依赖/权重/GPU，不动文件）
wuwor l_ltxvideo -- l_ltxvideo_doctor

:: 试跑一段（产物路径会打印）
wuwor l_ltxvideo -- l_ltxvideo_demo

:: 服务
wuwor l_ltxvideo -- l_ltxvideo_server        :: 或从主页卡片启动（推荐）
curl http://127.0.0.1:8475/healthz
```

从 hub「本机模型」页：`▶ 启动` → 等 `引擎在线` → `🧪 测试`（内置提示词出小样，页面直接播）。

---

## 5. 现实状态（2026-09-30）

- ✅ 包结构 / 服务壳 / 引擎 / worker / 生成器模板 / hub 页面与 API 全部就位
- ✅ 权重已下完（`<home>/weights` 26.5GB）
- ⚠️ **CUDA torch 预装**目前是**手工步骤**（`pip --target <shelf>/torch/<ver>/python torch==2.8.0+cu128 -i …/whl/cu128` + 补 `package.py`），
  原因见坑 1/2；下一步要做成安装流程里的**自动预装**（复用 `rez_pip_installer` 的包生成逻辑，但用货架里的 python，而不是 `~/.local/bin/python3.12.exe` 那个坏 shim）
- ⏳ 端到端出片测试待 torch 落地后补
