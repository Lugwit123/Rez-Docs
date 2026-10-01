# l_wanvideo — 本机 Wan2.2-TI2V-5B 视频生成（纯 Rez）

> 与 `l_ltxvideo` 同一套壳（纯 Rez 依赖 + worker 子进程 + 单卡串行 + 自带前端页），
> **包本体由 l_model_hub 的「本机模型」生成器按模板产出**（`runtime = rez_diffusers_video`）。

## 为什么选 5B

Wan-AI 最新的一批（`Wan2.2-Animate-2-14B` / `Wan-Dancer-14B` / `T2V-A14B` / `I2V-A14B`）都是
**14B MoE**，官方要 **24–48GB 显存**；本机 RTX 4080（16GB）跑不动。
**`Wan2.2-TI2V-5B` 是 16GB 上能跑的最新版**，质量明显强于同机位的 `l_ltxvideo`（LTX 2B）。

| 项 | 值 |
|---|---|
| 端口 / 卡片 | `127.0.0.1:8476` / `l_wanvideo_server` |
| 对外入口 | `http://127.0.0.1:8080/wanvideo/`（nginx upstream `lugwit_wanvideo_backend` + `location /wanvideo/`） |
| 权重 | HF **`Wan-AI/Wan2.2-TI2V-5B-Diffusers`**（≈32GB） |
| home | `L_WANVIDEO_HOME` > `{包根}/deploy_home.txt` > `~/.lugwit/l_wanvideo` |
| 16GB 档位 | **832×480 / 81 帧 / 16fps ≈ 5s**（720p 的 1280×704 需要 24GB+，本机不建议） |
| 别名 | `l_wanvideo_server` / `l_wanvideo_setup` / `l_wanvideo_doctor` / `l_wanvideo_demo` |

> **home 阶梯末环的现状（2026-10-01）**：阶梯顺序不变，但本机这些权重**已不在** `~/.lugwit/` ——
> 已于 2026-10-01 剪切到 **`E:/lugwit_rez/homes/l_wanvideo`**，由本包 `deploy_home.txt`（一行绝对路径）声明；
> 另可用 wuwo 注入的 `LUGWIT_SHARED_HOME`（= config 的 `l_shared_home_dir`，**盘符不存在时不注入**）。

## 仓库格式的坑（必读）

**必须用 `-Diffusers` 后缀的仓库**。同名不带后缀的 `Wan-AI/Wan2.2-TI2V-5B` 是 **Wan 原生格式**：
根目录三块 `diffusion_pytorch_model-0000X-of-00003.safetensors` + `models_t5_umt5-xxl-enc-bf16.pth`
+ `Wan2.2_VAE.pth`（≈33GB）——`WanPipeline.from_pretrained()` **读不了**。

- 踩过的现象：`allow_patterns` 只匹配到 `*.json` 与 `google/**`（0.02GB），selfcheck 报
  `⚠ 权重缺：model_index.json, text_encoder, transformer, vae, tokenizer`
- diffusers 版布局（与 `l_ltxvideo` 的 `allow_patterns` 天然匹配）：
  `model_index.json` + `scheduler/` + `text_encoder/`(3 分片) + `tokenizer/` + `transformer/`(5 分片) + `vae/`

## 画质静默降级的坑（已修，务必别再踩）

**Wan 的 `res.frames[0]` 是 torch tensor（GPU/bf16），不是 PIL 图**（LTX 才是 PIL）。
`worker._to_np()` 里若直接 `np.asarray(tensor)` → 抛 `OSError` → 被 `_encode()` 的兜底
`except` 吃掉 → **回退到 diffusers `export_to_video` 的默认低码率档**，日志里只有一句
`ffmpeg 直编失败(OSError)，回退默认编码`，看起来"生成成功"但画面很糊。

- 实测：5s / 832×480 / 81 帧 —— 回退档 **0.37MB**（≈610kbps）vs crf 14 **1.43MB**（≈2.4Mbps），肉眼差别明显
- 修法：`_to_np()` 先 `detach().float().cpu().numpy()`，再把 0..1 的浮点乘 255 转 uint8
- 排查手法：单独跑一次编码器（同参数 dummy 帧）能过 → 说明问题在**输入帧类型**，不在 ffmpeg 本身

## 阶段与"预计剩余时间"

页面（和 `/tasks`）每条进度都带 `stage` + `eta_s`，**每个阶段给自己的剩余时间**——
进度百分比在阶段边界上会长时间不动（读 32GB 权重几分钟、VAE 解码几十秒、编码十几秒），
只显示"已用 250s"用户没法判断还要等多久。

| 阶段 `stage` | 剩余时间怎么来 |
|---|---|
| `queue` | 前面还有任务 → **不给 ETA**（拿加载耗时冒充排队耗时就是骗人） |
| `load` | 上次实测加载耗时 − 已用（每 15s 一条心跳时更新） |
| `diffuse` | 本步实测均耗时 × 剩余步数（自校准，不用基线） |
| `decode` | 历史中位（最后一步回调里就切成"解码中"——那之后到出帧还有几十秒） |
| `encode` | 历史中位 |

实测值落在 `<home>/runtime/stage_eta.json`（`eta.py`，按 stage 存最近 5 次、读时取中位数）。
**没有历史就返回 `None`**，页面显示"剩余：暂无参考（首次运行）"，不猜；
文件坏了/写不进去只影响提示，绝不影响生成。`/state.stage_eta` 给出各阶段历史参考。

## 环境与依赖

与 `l_ltxvideo` **共用同一套共享货架依赖**（torch 2.8.0+cu128 / diffusers 0.35.2 / transformers 4.57.6 /
tokenizers 0.22.2 / huggingface_hub 0.36.2 / sentencepiece / tiktoken…）。
版本组合与踩坑见 `l_ltxvideo.md` 的「已知可用的依赖组合」与坑表 —— **那些坑对 Wan 一样适用**。

## 验证

```cmd
wuwor l_wanvideo -- l_wanvideo_doctor     :: 依赖/权重/GPU
wuwor l_wanvideo -- l_wanvideo_demo       :: 试跑一段（480p）
curl http://127.0.0.1:8476/healthz
:: 或页面：http://127.0.0.1:8080/wanvideo/
```
