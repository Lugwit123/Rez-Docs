# l_indextts2 使用指南（本机 IndexTTS2 语音合成）

状态：**2026-09-28**。行号会漂，正文一律用**函数名 / 路由**定位（约定见 `../README.md`）。

## 1. 是什么

把 [IndexTTS2](https://github.com/index-tts/index-tts)（B 站开源、零样本音色克隆 + 情绪控制）
做成**本机常驻服务**，并接进 l_model_hub 的语音链路：

| 项 | 值 |
|---|---|
| 包名 / 版本 | `l_indextts2` / `999.0` |
| 服务卡 | 别名 `l_indextts2_server`，显示名「L IndexTTS2 本机语音」，端口 `8470` |
| 网关 | nginx `/indextts2/` → 剥前缀 → `127.0.0.1:8470`（页面/接口全用相对路径）|
| 模型版本 | IndexTTS-2.5（默认，`checkpoints`，多语言 + 语速）/ IndexTTS-2（`checkpoints_2`）|
| 推理环境 | **独立 uv venv（python 3.10）**，不是 Rez 环境（见 §2）|
| 数据 | 全在 `L_INDEXTTS2_HOME`（默认 `~/.lugwit/l_indextts2`）：venv / index-tts / 权重 / 音色 / runtime |
| 许可 | **bilibili Model Use License —— 非商用**；本包定位本机自用，商用需联系 indexspeech@bilibili.com |

## 2. 为什么是「独立 venv + 服务壳」而不是普通 Rez 依赖

IndexTTS2 的 `pyproject.toml` 写死 `requires-python = ">=3.10,<3.12"`，而 wuwo 只带
`py_312`（`wuwo\py_312`）—— **装不进 Rez 环境**。它的依赖也 pin 得很死
（`torch==2.8.*` / `transformers==4.52.1` / `keras==2.9.0` / `numba==0.63.0` …），
塞进 `l_model_hub` 这种被十几个服务共用的包环境必炸。

所以走 `l_comfyui` 的同一条路：**重 AI 依赖独立环境**。

- `l_indextts2`（py3.12，Rez 环境）＝ 服务壳：HTTP、音色档案、worker 进程管理；
- worker 进程＝ **venv 里的 python 3.10**，只跑推理，通过 stdin/stdout JSON 协议通信。

顺带的好处：显存 OOM / 模型库崩溃**带不走 HTTP 服务**，重启只需重拉 worker。

> 代价：torch 这套依赖不走 Rez 的 `requires` 治理，由 `tools/setup_runtime.py`
> 用 uv 装进 `L_INDEXTTS2_HOME/venv`。这是本项目「重 AI 依赖」的既有例外，改动前先读本节。

## 3. 安装（一次性，会下数 GB）

```cmd
wuwor l_indextts2 -- indextts2_setup --dry-run      :: 先看计划
wuwor l_indextts2 -- indextts2_setup                :: 克隆仓库 + uv venv(3.10) + 依赖 + 权重
```

分步/可选：

```cmd
wuwor l_indextts2 -- indextts2_setup --skip-weights        :: 先把环境跑通，权重回头再拉
wuwor l_indextts2 -- indextts2_setup --source modelscope   :: 权重换魔搭源（HF 慢时用）
wuwor l_indextts2 -- indextts2_setup --webui --seed-voices :: 额外装 gradio；把 examples 注册成音色
```

- torch 走 **CUDA 12.8 源**（`https://download.pytorch.org/whl/cu128`）：PyPI 上的 Windows
  wheel 是 **CPU 版**，直装会把 GPU 推理变成「跑得动但慢十倍」。
- 换运行时根目录：`--home D:\Lugwit\indextts2`（会写 `{包根}/deploy_home.txt`，
  部署级文件、**不入库**；也可用环境变量 `L_INDEXTTS2_HOME`）。

  > **home 阶梯末环的现状（2026-10-01）**：阶梯顺序不变（`L_INDEXTTS2_HOME` >
  > `{包根}/deploy_home.txt` > `~/.lugwit/l_indextts2`），但本机权重**已不在** `~/.lugwit/` ——
  > 已于 2026-10-01 剪切到 **`E:/lugwit_rez/homes/l_indextts2`**，由本包 `deploy_home.txt`
  > （一行绝对路径）声明；另可用 wuwo 注入的 `LUGWIT_SHARED_HOME`（= config 的
  > `l_shared_home_dir`，**盘符不存在时不注入**）。
- 权重目录：`<home>/checkpoints`（2.5）/ `<home>/checkpoints_2`（2.0）。
  **注意上游仓库里的 `hf_cache/` 是半成品**（`bigvgan` 只有 `config.json`），而官方代码
  「目录存在就跳过补装」→ 我们额外跑一次 `ensure_models_available` 把 bigvgan / w2v-bert /
  semantic_codec / campplus 补全（`fetch_weights.py` 内置，缺了会非零退出）。
  单独补：`wuwor l_indextts2 -- indextts2_fetch_weights`。
- 装完自检：`wuwor l_indextts2 -- indextts2_doctor`（venv / 仓库 / 权重 / 端口），
  或 `<venv>\Scripts\python.exe -m l_indextts2.worker --check`（看 torch / CUDA / 能否 import 引擎）。
- 只是**想听一句**验证装好了：`wuwor l_indextts2 -- indextts2_demo`
  （注册示例音色 → 合成 → 直接播放，首次等模型载入）。

## 4. 启停

**走服务卡（`wuwo svc`）**，别直启（直启会绕过重启锁与 watchdog）：

```cmd
wuwo svc start|restart l_indextts2     :: 起/重启（改 .py 后）
wuwo svc status l_indextts2            :: 状态（在线 = 端口 + HTTP 都通）
wuwo svc log l_indextts2 -n 100 -f     :: 服务日志（worker 的 stderr 也在这里）
wuwo svc open l_indextts2              :: 开页面
```

- 服务**默认不预热**：进程秒起，第一次合成才载模型（**30~60s**，README 实测冷启动）。
  想启动即载：启动前设 `L_INDEXTTS2_WARMUP=1`，或起来后 `curl -X POST .../load`。
- 单卡串行：请求按一把互斥锁排队（`engine.EngineWorker._lock`），
  `/healthz` 的 `busy_requests` 看排队深度。

## 5. 接口

| 路由 | 说明 |
|---|---|
| `POST /v1/audio/speech` | **OpenAI 兼容**：`{model:"indextts2", input, voice, response_format:"wav", speed}` + 本机扩展 `lang / emotion / emo_alpha / emo_vector / emo_text / use_emo_text / duration_factor`。`speed` 与 `duration_factor` 是倒数关系（`speed=0.5` → 时长 ×2）|
| `POST /tts` | l_model_hub 的既有形状（`text / voice / emotion` + 同样扩展字段）→ 返回 `audio_base64`（wav）|
| `GET /speakers` · `POST /speakers` · `DELETE /speakers?id=` | 音色档案：查 / 注册（`{name, audio_base64}`，wav 或 mp3，≤30MB）/ 删 |
| `POST /speakers/seed` | 把 index-tts 的 `examples/*.wav` 注册成音色（幂等）|
| `GET /healthz` · `POST /load` · `POST /worker/restart` | 状态（**不载模型**）/ 预热 / 重启 worker |
| `GET /` | 极简页面：试听、注册音色、看运行时状态 |
| `GET/POST /__dev__/src_watch` | 统一热重载开关（**只有 `.dev_mod` 启动才可能开**）|

`voice` 的语义（与云端 TTS 不同，重点）：

1. 音色档案 **id**（`GET /speakers` 里的 `id`）→ 首选；
2. 档案**显示名**；
3. 一段音频的**文件路径**（绝对/相对都行）；
4. 空 → 取第一个档案（所以至少要有一个档案，否则报「先注册参考音频」）。

## 6. 从 l_model_hub 调用

`l_model_hub` 侧已接好，两条路：

```cmd
:: ① 既有 /tts 形状，engine 选本机
curl -X POST http://127.0.0.1:8462/tts -H "Authorization: Bearer <sk-lmh-…>" ^
     -H "Content-Type: application/json" ^
     -d "{\"text\":\"你好\",\"engine\":\"indextts2\",\"voice\":\"voice_01\",\"emotion\":\"happy\"}"

:: ② OpenAI 兼容入口（客户端只认这一个）
curl -X POST http://127.0.0.1:8462/v1/audio/speech -H "Authorization: Bearer <sk-lmh-…>" ^
     -H "Content-Type: application/json" ^
     -d "{\"model\":\"indextts2\",\"input\":\"你好\",\"voice\":\"voice_01\"}" -o out.wav
```

- `/v1/audio/speech` 的 `model` 还能写 **`<provider>/<model>`**：那时 l_model_hub 会转发到
  该厂商（含 BYO 自定义提供商）的 OpenAI 兼容 `/audio/speech` —— 社区版
  `index-tts-vllm` 之类就是这种接法，把它按 BYO provider 填进管理台即可，不用改代码。
- 调用统计：本地合成按「**按次记、token 为 0**」进管理台的模型维度，键为 `indextts2/indextts2`。
- 相关内容：`L_INDEXTTS2_URL` / `L_INDEXTTS2_TIMEOUT`（l_model_hub 侧，见 `l_model_hub.md` §8）。

## 7. 环境变量

l_indextts2 服务侧：

| 变量 | 默认 | 说明 |
|---|---|---|
| `L_SRC_WATCH_PORT` | `8470` | 端口（所有服务共用这一个名）|
| `L_INDEXTTS2_HOME` | `~/.lugwit/l_indextts2` | 运行时根目录（venv / 仓库 / 权重 / 音色）|
| `L_INDEXTTS2_VERSION` | `2.5` | 模型版本（`2.5` / `2`）；也可写 `<home>/version.txt` |
| `L_INDEXTTS2_RUNTIME` | `<home>/runtime` | pid / 热重载状态 / 输出 wav |
| `L_INDEXTTS2_WARMUP` | 关 | `1` = 启动即载模型 |
| `L_INDEXTTS2_QWEN_EMO` | 关 | `1` = 初始化时带 `use_qwen_emo=True`（2.5 用 `use_emo_text` 必须开，代价是多占显存）|
| `L_INDEXTTS2_LOAD_TIMEOUT` / `_SYNTH_TIMEOUT` | `900` / `600` | 载模型 / 单次合成超时（秒）|
| `L_INDEXTTS2_START_TIMEOUT` | `120` | worker 启动握手超时 |

l_model_hub 侧：`L_INDEXTTS2_URL`（默认 `http://127.0.0.1:8470`）、`L_INDEXTTS2_TIMEOUT`（默认 `300`）。

## 8. 排错速查

| 症状 | 先看 |
|---|---|
| 合成返回 503「运行时缺失」 | 没跑过 `indextts2_setup`；或 `L_INDEXTTS2_HOME` 指到了别处（`indextts2_doctor` 一眼看出）|
| 合成返回 502「IndexTTS2 合成失败」 | `wuwo svc log l_indextts2 -n 100`：worker 的 stderr 在这里；`/healthz` 的 `last_error` 是最后一条原因 |
| 首次调用等很久 | 正常：冷启动 30~60s 载模型。要避免就 `L_INDEXTTS2_WARMUP=1` |
| 显存不足 / CUDA OOM | 2.5 默认 bf16、2.0 默认 fp16 已省显存；还不行就关掉别的占卡程序，或降到 `L_INDEXTTS2_VERSION=2` |
| 报「未知音色」 | 还没注册音色档案：`POST /speakers`，或 `indextts2_setup --seed-voices` |
| `speed` 方向反了 | `speed` 与 `duration_factor` 是倒数：`speed=0.5` 表示**变慢一倍**（时长 ×2）|
| 服务 /healthz 通但 worker 反复重启 | 看 `last_error`：多为权重半截、CUDA 版本不匹配、venv 被删 |
| 报缺 `hf_cache/bigvgan/bigvgan_generator.pt` | 上游 `hf_cache` 半成品，官方代码不补 → `wuwor l_indextts2 -- indextts2_fetch_weights` |
| 装 sdist 报 `SRE module mismatch` | Rez 的 py3.12 `PYTHONPATH` 泄漏给 3.10 子进程 → 子进程一律走 `config.child_env()`（已修，别再往子进程塞 `os.environ` 的 PYTHONPATH）|
| 页面在 `/indextts2/` 打不开接口 | 页面用相对路径，必须带**尾斜杠**访问（`/indextts2/`）；nginx 已配 `location = /indextts2` 302 补斜杠 |

## 9. 关键源文件

| 文件 | 职责 |
|---|---|
| `src/l_indextts2/server.py` | FastAPI 路由（`/v1/audio/speech`、`/tts`、`/speakers`、`/healthz`）+ 极简页面 + 热重载接入 |
| `src/l_indextts2/engine.py` | worker 进程管理：拉起 / 保活 / 串行锁 / 超时兜底 / `status()` |
| `src/l_indextts2/worker.py` | **venv(3.10) 里的真推理**：版本选择、infer kwargs 过滤、情绪→8 维向量 |
| `src/l_indextts2/speakers.py` | 音色档案（参考音频注册/解析，零样本克隆的 `spk_audio_prompt`）|
| `src/l_indextts2/config.py` | 路径/端口/版本解析（`deploy_home.txt` 约定同 l_model_hub 的 `deploy_auth_url.txt`）|
| `tools/setup_runtime.py` · `tools/fetch_weights.py` · `tools/start_service.py` | 安装 / 拉权重 / 启动入口 |
| `tests/smoke.py` | 无 GPU 自检（路径 / 音色 / 情绪映射 / kwargs 过滤 / 未就绪报错）|

l_model_hub 侧改动：`client.py`（`indextts2_tts` / `openai_speech` / `indextts2_url`）、
`server.py`（`TTSBody` 扩展 + `/tts` 的 indextts2 分支 + `/v1/audio/speech`）、
`tests/test_tts_indextts2.py`。

## 10. 其他

- 想用官方 WebUI（`:7860`，gradio）：`indextts2_setup --webui`，然后
  `<home>\venv\Scripts\python.exe <home>\index-tts\webui.py`（WebUI 与本服务**互斥**占卡，
  别同时跑，显存不够）。
- 想上 vLLM 加速：走社区 `index-tts-vllm`，按 BYO provider 填进 l_model_hub 管理台
  （`/v1/audio/speech` 已支持 `<provider>/<model>` 转发），本包不用改。
