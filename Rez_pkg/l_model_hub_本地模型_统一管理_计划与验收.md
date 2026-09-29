# l_model_hub · 本地开源模型「统一管理 / 安装 / 监控 / 测试」计划与验收

> 目标：在 **l_model_hub 管理台**里点几下就能把开源模型（视频 / 图像 / 语音 / LLM）装起来、跑起来、测通、接成 hub 的引擎位。
> 包模型：**A 方案 —— 一个模型一个独立 Rez 包 + 一张服务卡片**，hub 只做控制面（不自己 spawn 进程、不塞业务数据）。

---

## 0. 为什么是 A 方案（约束来自本仓规则）

| 约束 | 出处 | 后果 |
|---|---|---|
| 服务卡片里的服务**不许直启**，一切走 `wuwo svc` | `AGENTS.md`「服务启动入口」 | hub 只能调卡片 API，不能自己起进程 |
| 业务数据不落本机，走 auth 中心存储 | `l_model_hub.md` §7 · `state_store.py:34-40` | 「装了什么/装到哪/许可同意」都写中心存储 |
| 重 AI 依赖（torch/diffusers）不能进 Rez `requires` | `Rez包创建和启动指导文档.md` | 必须独立 venv，子进程剥 `PYTHONPATH` |
| 子进程要剥 Rez 注入的环境 | `l_indextts2/config.py:155-175` | 否则 py3.10 撞 py3.12 stdlib（`SRE module mismatch`） |
| 不许硬编码外部路径绕依赖治理 | `AGENTS.md`「建包硬规则」 | 路径一律 `L_<PKG>_HOME` > `deploy_home.txt` > `~/.lugwit/<pkg>` |

**结论**：每个开源模型 = 一个包（如 `l_wanvideo`）+ 一张卡片；hub 新增一个 tab 做「目录 / 安装 / 监控 / 测试 / 接引擎位」。

---

## 1. 目标 / 非目标

**目标**
1. hub 管理台新增 **🖥 本机模型** 页：清单、状态、安装进度、监控、一键测试、设为引擎位
2. 安装 **全程可观测**：阶段 + 权重下载百分比 + 日志尾巴；失败给中文原因
3. 装完 **一条命令能自证**（内置参考提示词 → 产出小样 → 页面里直接预览）
4. 通过后 **自动接到引擎位**（keystore 写 `xxx_url` / `model`），hub 的 `/video` `/tts` `/image` 立即可选

**非目标（本轮不做）**
- 不做模型微调 / 训练
- 不做多机分布式推理
- 不做模型权重回传云端（只在本地留）

---

## 2. 目录与落盘约定

| 用途 | 路径 | 说明 |
|---|---|---|
| 模型包运行时根 | `L_<PKG>_HOME` > `{包根}/deploy_home.txt` > `~/.lugwit/<pkg>` | 照 `l_indextts2/config.py:75-80` |
| venv | `<home>/venv`（默认 python 3.10） | 独立于 Rez py3.12 |
| 权重 | `<home>/weights`（或仓库自带 `checkpoints/`） | 不入 git；`.gitignore` 兜底 |
| 输出小样 | `<home>/runtime/out` | 测试产物，可清理 |
| 卡片定义 | `l_homepage/config/services_builtin.json` + 用户覆盖 `~/.lugwit/l_homepage/runtime/services.json` | 照 `services_builtin.json:362-384` |
| hub 侧配置 | auth 中心存储：`local_models_dir`、`local_model_<id>_url`、`-enabled`、`-license_agreed_at` | 走 `keystore.get/set`（`keys.py:202/216`） |
| hub 侧清单 | 包内 `data/local_models.json`（只读目录） + 中心存储的用户覆盖 | 清单是"可安装目录"，不是业务数据 |

模型目录默认建议：`L_MODEL_HUB_MODELS` > 中心存储 `local_models_dir` > `D:\TD_Depot\Models`（本机已确认 E:/K: 有 3T+ 空间）。

---

## 3. hub 侧改动（很小，全是加法）

| 步骤 | 文件 | 改什么 |
|---|---|---|
| 1 | `server.py:249` `ADMIN_TABS` | 加 `{"id":"local","icon":"🖥","label":"本机模型","desc":"装/管/测开源模型"}`（自动进 `TAB_IDS` / `_tabs_html` / `__TABS__`） |
| 2 | `templates/admin.html` | 加 `<div class="section panel" data-tab="local">`（照 `admin.html:254`） |
| 3 | `admin.html:553 ensureTabData()` | 加 `else if(id==='local'){ await Promise.all([loadLocalModels()]); }` |
| 4 | `admin.html` 脚本段 | `loadLocalModels()` 渲染卡片/表格 + 轮询任务 |
| 5 | `server.py`（新段） | 6 个路由，见下表 |

### API 设计

| 路由 | 方法 | 作用 | 复用 |
|---|---|---|---|
| `/v1/local-models` | GET | 清单 + 已装状态 + 服务状态（在线/端口/healthz）+ 是否已接引擎位 | 读包内 json + keystore |
| `/v1/local-models/install` | POST | 起安装任务 → `{task_id}`；body `{id, accept_license:true, skip_weights?:bool}` | 照 `server.py:1007` 任务骨架 |
| `/v1/local-models/tasks/{task_id}` | GET | `{stage, percent, log_tail, error}` | 同上 |
| `/v1/local-models/tasks/{task_id}/cancel` | POST | 取消（杀子进程 + 清理半成品可选） | — |
| `/v1/local-models/{id}/{start\|stop\|restart}` | POST | **只经卡片 API**：`POST /api/v1/services/<card>/<op>?trigger=cli` | `AGENTS.md` 服务规则 |
| `/v1/local-models/{id}/test` | POST | 跑内置参考提示词 → 返回小样 URL + 耗时 | 引擎自带 `/tts` `/generate` 等 |
| `/v1/local-models/{id}/logs?n=200` | GET | 日志尾巴（卡片的 `wuwo svc log` 等价物） | — |

**监控数据**：服务状态取端口 + `/healthz`；GPU 取 `nvidia-smi --query-gpu=name,memory.used,memory.total,utilization.gpu,temperature.gpu --format=csv,noheader`（RTX 4080 16GB 已实测可用）；任务进度取上表 `tasks/{id}`。

---

## 4. 模型清单 schema（`local_models.json`）

```jsonc
{
  "id": "wan-ti2v-5b",
  "name": "Wan2.2 TI2V-5B（文/图生视频）",
  "kind": "video",                 // video|image|tts|asr|llm
  "pkg": "l_wanvideo",             // 对应 Rez 包 + 卡片
  "card": "l_wanvideo_server",     // 卡片 name（命令行标识）
  "url_env": "L_WANVIDEO_URL",     // 引擎地址环境变量（照 L_INDEXTTS2_URL）
  "default_url": "http://127.0.0.1:8475",
  "repo": "https://github.com/Wan-Video/Wan2.2",
  "weights": { "source": ["modelscope", "hf"], "approx_gb": 20, "files": ["..."] },
  "python": "3.10",
  "cuda": "cu128",
  "vram_gb": 16,                   // 显存需求（预检用）
  "license": { "name": "Apache-2.0", "commercial": true, "note": "" },
  "health": "/healthz",
  "test": { "prompt": "无人机航拍雪山日出，缓慢右移", "expect": "mp4", "max_wait_s": 600 },
  "engine_slot": { "field": "local_video_url", "model_field": "local_video_model" }
}
```

许可字段是**硬闸口**：`commercial:false` 的模型在 UI 上标红，安装前必须勾选同意（记 `local_model_<id>_license_agreed_at` = 时间 + 用户）。

---

## 5. 第一个参考包（P0 打通链路）

建议 **`l_wanvideo`**（Wan2.2-TI2V-5B 打底，Apache-2.0 商用友好，16GB 可跑 720p 短片）。

安装脚本五步（照 `l_indextts2/tools/setup_runtime.py` 结构）：
1. `git clone --depth 1 <repo> <home>/wan`
2. `uv venv --python 3.10 <home>/venv`
3. `uv pip install torch torchaudio --index-url https://download.pytorch.org/whl/cu128`（**必须显式 cu128**，否则 PyPI 给 CPU 版）
4. `uv pip install -e <home>/wan --index-url <aliyun>`
5. 取权重（`--source modelscope|hf`，**权重下载要报百分比**）→ 自检（torch/cuda/device/模型可加载）

推理侧（worker，照 `l_indextts2`）：
- 子进程 + stdin/stdout JSON 协议；**单卡串行互斥锁**（`l_indextts2/engine.py:45,217`）
- HTTP 面：`GET /healthz`（轻，不载模型）、`POST /load`（预热）、`POST /generate`（任务）、`GET /tasks/{id}`
- 超时：冷启动载模型给足（照 `L_INDEXTTS2_START_TIMEOUT=120`、`_LOAD_TIMEOUT=900`）

卡片记录（`services_builtin.json` 追加）：
```json
{ "name": "l_wanvideo_server", "label": "L WanVideo 本机视频", "url": "/wanvideo/",
  "port": 8475, "kind": "service", "auto_start": false,
  "packages": ["l_wanvideo", ".dev_mod", ".solo"],
  "run_args": ["l_wanvideo_server"],
  "run_cmd": "wuwor l_wanvideo .dev_mod .solo -- l_wanvideo_server" }
```

hub 侧接线：`client.local_video_url()`（照 `client.indextts2_url()`）+ `local_video_generate()`；`/video/task` 增加 `provider=local` 分支，与火山 Seedance 并列。

---

## 6. 安装任务状态机（断点续装）

```
queued → cloning → venv → deps → weights(0-100%) → selfcheck → card → done
                                          ↘ failed（中文原因 + 日志尾巴 + 重试按钮）
```
- 进度**真实**：每步上报；权重阶段按已下载字节 / 总字节算百分比（拿不到总大小就报"已下 X MB"）
- 状态落**中心存储**_task_<id>`），hub 重启后：`done` 保留结论；中间态标记"上次被中断"→ 允许"继续/重装"
- 取消：杀子进程 + 保留已下权重（续装可复用），不删已装好的 venv/权重

---

## 7. 验收标准

**功能验收**
1. UI 能列出清单 + 已装状态 + 卡片状态（在线/端口通/未启动），未装的可一键装
2. 安装过程能看到阶段 + 百分比 + 日志尾巴；失败给中文原因（不是堆栈）
3. 装完点「测试」能用内置提示词出小样，页面直接预览（视频可播放/图片可看/音频可听）
4. 测试通过后「设为引擎位」一键生效：hub `/video`（或 `/tts` `/image`）能选到它并跑通一次
5. 启停**只经卡片**（`wuwo svc` 等价 API）；直启会被规则拦（符合现有豁免说明）

**通用红线**
6. 卸载/取消**不留孤儿进程**、不删他人权重；端口不残留占用
7. 安装/推理在失败时**不影响网关**（其它模型照常可用）
8. 权重/venv **不进 git**；许可未同意的模型**不能安装**
9. 重启 hub 后：已装状态可恢复，中断的安装可继续/重装（不出现"卡在 40% 假装在装"）
10. 一键测试有**超时上限**（默认 600s，按模型配），超时给明确提示而不是无限转圈

**观测**
11. 页面能看到 GPU 显存/利用率/温度 + 当前任务阶段
12. 日志尾巴可跟随（增量拉取），报错行能定位到具体步骤

---

## 8. 分期

| 期 | 内容 | 交付判据 |
|---|---|---|
| **P0** | hub「本机模型」页骨架（清单/状态/日志/GPU）+ 状态机 + `l_wanvideo` 参考包（装+测通） | 验收 1–4、6、8 通过 |
| **P1** | 断点续装 + 取消 + 卸载 + 许可闸口 UI + 设为引擎位（写 keystore 并出现在 `/video` 引擎列表） | 验收 5、9、11、12 通过 |
| **P2** | 更多模型（图像 Qwen-Image / FLUX、视频 LTX-Video、LLM 本地化）；可选 ComfyUI 后端（`l_comfyui_backend` 8700）跑工作流 | 新增模型复用同一页面零改 UI |

---

## 9. 风险与回滚

| 风险 | 处置 |
|---|---|
| （16GB） | 清单里写 `vram_gb`，安装前预检；超了直接拦并给"要换哪个档"的建议 |
| 权重下载慢/断 | HF 走 `hf-mirror`，默认 ModelScope；支持续装 |
| 许可不许可商用 | 清单 `license.commercial` 硬闸口 + UI 标红 + 记录同意时间 |
| 长任务把 hub 卡住 | 安装一律子进程 + 线程池，网关路径零阻塞（照 `server.py:412 asyncio.to_thread`） |
| 和现有 `l_indextts2` 职责重叠 | 语音继续用现成包，新页面把 `l_indextts2` 也**作为清单项纳管**（统一入口，不重复实现） |
| 回滚 | 全是加法：`ADMIN_TABS` 去掉一项 + 删 `data-tab="local"` 面板即恢复现状；新包不装就不影响 |
