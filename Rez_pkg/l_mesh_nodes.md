# `l_mesh_nodes` —— 程序化 3D 建模（`l_node_engine` 的**引擎外插件**）

> 2026-10-21 从 `l_node_engine` 核心**拆出来**成独立包（照 `l_tex_nodes` 的分层），
> 引擎只保留接口：注册表 / 插件壳 / 值契约。**插件形态 B**：宿主 `requires` 里没有它，
> 启用时按 rez 包名解析路径再 import。

## 1. 包结构

```
l_mesh_nodes/999.0/
  package.py                      requires = [python, numpy, numba, l_node_engine]
    alias l_mesh_nodes_selftest            几何算子自证（python -m l_mesh_nodes.mesh3d）
    alias l_mesh_nodes_plugin_selftest     插件层自证（48 节点注册 / 真跑图 / 上限 / 停用摘干净）
    alias l_mesh_nodes_test                回归网（tests/test_mesh3d.py，147 条）
    alias l_mesh_nodes_gpu_bench           numpy / numba / torch 基准
  src/l_mesh_nodes/
    mesh3d.py            纯 numpy 几何算子（图元/细分/置换/雕刻/倒角/UV/绑定/散布/重拓扑…）
    mesh3d_plugin.py     48 个节点类 + `Mesh3DPlugin`（**导入即注册**）
    mesh3d_gpu.py        GPU 可选后端（torch 优先 / cupy 备选；插件「依赖目录」里有才用）
    mesh3d_njit.py       CPU JIT（numba `@njit`：taubin + remesh 栅格化）
    plugin_selftest.py   插件层自证
  tests/test_mesh3d.py   147 条回归网
  tools/mesh3d_gpu_bench.py
```

**依赖纪律**：`numpy` / `numba` 写在 `requires`（本包是"插件可以重依赖"的那一侧）；
`manifold3d`、`torch`、`cupy` 一律走插件面板「依赖目录」（`pip --target`），宿主 `requires` 里永不出现。

## 2. 引擎侧接口（核心只剩这些）

| 接口 | 位置 | 用途 |
|---|---|---|
| `register_node` / `unregister_source` | `l_node_engine.registry` | 节点进表 / 按 `NODE_SOURCE` 摘干净 |
| `Plugin` / `register_plugin` | `l_node_engine.plugins` | 插件壳（字段/状态/`apply`） |
| `plugins.load_external()` | 同上 | **引擎外插件加载**：读声明 → rez 解析 → `import` |
| `plugins_rez.resolve(pkg)` | `l_node_engine.plugins_rez` | `wuwor <pkg> -- python …` 取 `PYTHONPATH`（进程内 + 磁盘缓存） |
| 值契约 | — | `MESH` 端口上是**普通 dict**（`{kind, format, vertices, faces, attrs}`），引擎只透传 |

## 3. 接入方式（形态 B）

`<插件配置目录>/external_plugins.json`：

```json
[{"id": "mesh3d", "module": "l_mesh_nodes.mesh3d_plugin", "rez": "l_mesh_nodes",
  "deps": ["C:\\Users\\<你>\\.lugwit\\main\\l_node_engine\\mesh_deps"]}]
```

加载链路：
1. 引擎进程启动时 `plugins._prefetch_external()` **后台线程**预热（有声明才起）；
2. `load_external()`：`plugins_rez.resolve("l_mesh_nodes")` → 跑一次
   `wuwor l_mesh_nodes -- python <打印 sys.path>` → 挑出本包 payload → 追加 `sys.path` 末尾；
3. `import l_mesh_nodes.mesh3d_plugin` → 模块末尾 `register_plugin(Mesh3DPlugin())`；
4. 宿主照旧 `bootstrap()` / 面板「重载」→ **48 个节点**进节点库。

缓存与失败：
- 进程内缓存 + 磁盘缓存 `<配置目录>/_rez_paths.json`（7 天；路径不存在就重解析）；
- 解析/import 失败**不抛**：原因进 `EXTERNAL_STATE["errors"]`（面板/`l_node_engine_selftest` 能看到），宿主照常跑、只是少一块节点；
- 首次启动 `_rez_paths.json` 为空时要跑一次 wuwor（实测约 3 s；之后秒开）。
  ⚠ 首个「插件列表」请求在**冷启动**时仍会等预热完成（实测 12–15 s，其中大头是 comfy 插件那 65 个节点），
  第二次 4 ms —— 这是既有行为，不是本次拆包引入的。

## 4. 自证

```cmd
wuwor l_mesh_nodes -- l_mesh_nodes_selftest          :: 几何（mesh3d: OK）
wuwor l_mesh_nodes -- l_mesh_nodes_plugin_selftest   :: 插件层（48 节点/真跑图/上限/停用）
cd rez-package-source\l_mesh_nodes\999.0 && wuwor l_mesh_nodes -- python -m unittest tests.test_mesh3d
wuwor l_node_engine -- l_node_engine_selftest        :: 引擎侧（已不含 mesh3d，只验外部插件接口）
```

## 5. 算法 / 节点 / GPU / 踩坑

细节仍在两处，别在这里重复：
- **机制与字段（权威）**：`Rez-Docs/Rez_pkg/l_node_engine.md` §3.0d″（插件框架 + 40+ 节点的端口/口径）
- **现状 / 待办 / 实测数字**：`Rez-Docs/Rez_pkg/3D程序化建模_交接文档.md`（§1.6 GPU、§1.6b numba、§5 TODO、§3 十条坑）
