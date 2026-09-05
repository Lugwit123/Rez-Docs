# 变体哈希与 wuwo 的处理方法

> 主题：rez-pip 装包时"变体目录名"踩到 Windows 非法字符，导致包**看起来装上了、实际上是个空壳**。
> 本文以 `reflex` 包为完整案例，讲清来龙去脉、官方解法、wuwo 的历史取舍、本次的条件回退修复，以及排查手册。
>
> 关键代码位置（以函数名为准，行号截至 2026-09）：
>
> | 位置 | 作用 |
> |---|---|
> | `wuwo/py_modules/auto_fetch_packages.py:1630` `_rez_req_to_pip_spec` | rez 需求串 → pip 安装说明符 |
> | `wuwo/py_modules/auto_fetch_packages.py:1661` `_exact_ver_from_pip_spec` | 从 pip 说明符取精确版本（复用判断用） |
> | `wuwo/py_modules/auto_fetch_packages.py:2072` `_WIN_PATH_FORBIDDEN_RE` | Windows 路径非法字符表 |
> | `wuwo/py_modules/auto_fetch_packages.py:2075` `_pkg_variants_need_hash` | 判定"这个包的变体名能不能当目录名" |
> | `wuwo/py_modules/auto_fetch_packages.py:2091` `_apply_rez_pip_plain_variant_patch` | 可读变体目录 monkeypatch（含本次修改） |
> | `wuwo/py_modules/auto_fetch_packages.py:2273` `_install_pip_via_rez_pip` | 调官方 rez-pip 落地到 rez-package-3rd |
> | `wuwo/py_312/Lib/site-packages/rez/pip.py:415,427-432` | rez-pip 写 variants、强制 hashed_variants |
> | `wuwo/py_312/Lib/site-packages/rez/utils/pip.py:470-490` | "带 marker 的依赖进变体"规则 |

---

## 〇、先给结论（太长不看）

1. **不是**"wuwo 改成用哈希装包了"。是**只有当变体名字里有 Windows 不许用的字符时，才对那一个包退回哈希**；其余包照旧用可读目录名。
2. 触发条件是：某个 pip 包的依赖**同时**满足两条——带环境标记（environment marker）、且版本区间里含 `<` `>` 等符号。`reflex` 恰好中招。
3. 症状极具迷惑性：`wuwor <包>` 能解析成功、注册表里有这个包、`package.py` 也在，但 `import` 报 `ModuleNotFoundError`。因为**房子是空的**。
4. 修法有两处，都在 wuwo 层，不动 rez 本体：变体哈希条件回退 + 短版本 `==1` 改 `==1.*`。

---

## 一、把 rez 仓库想成一条街道

要理解这件事，先得理解 rez 的"变体（variant）"。用街道打比方：

```
rez-package-3rd/                     ← 这条街
  reflex/                            ← 一栋楼，门牌写楼名"reflex"
    0.9.10.post1/                    ← 楼里的一个单元，"0.9.10.post1"
      platform-windows/              ← 这一户的户型：Windows 座
        arch-AMD64/                  ← 再细分：AMD64 翼
          python-3.12/               ← 再细分：3.12 层
            psutil-7.0.0+<8.0/       ← 再细分：装 psutil 7.0.0 到 8.0 之间的那间
              python/                ← 屋里真正放家具的地方（site-packages）
                reflex/
              bin/
```

- **楼名** = 包家族（package family），rez 认目录名。
- **单元号** = 版本号。
- **户型/翼/层/间** = 变体标签（variant label）。同一个包可以同时存在"3.11 层"和"3.12 层"两间房，内容不一样（比如 C 扩展按 Python 版本各编译一份）。
- **`{root}`** = 当 rez 解析到某个具体变体时，`{root}` 指向**最里面那间房的门口**，所以 `package.py` 里写 `env.PYTHONPATH.append('{root}/python')` 能找到家具。

 rez 的规矩很直白：**变体标签直接当目录名刻在门牌上**。这一条就是全部麻烦的源头。

---

## 二、为什么 `psutil` 会变成门牌上的一个字

看 `reflex` 自己的依赖声明（wheel 元数据原文）：

```
Requires-Dist: psutil<8.0,>=7.0.0; sys_platform == 'win32'
```

后半截 `; sys_platform == 'win32'` 叫**环境标记**。它的意思是：

> 购物清单上这条写着——"只有在下雨天才买"。

对 rez 来说，"看天气才买的东西"**不能**和其他常备品混在一张清单里，因为不同天气（不同平台）下清单内容不同。于是 rez-pip 的规则是：**带 marker 的依赖，进变体，不进 requires**。

代码在 `rez/utils/pip.py:470-490`：

```python
to_variant = False

if req.marker:
    marker_reqs = get_marker_sys_requirements(str(req.marker))

    if marker_reqs:
        sys_requires.update(marker_reqs)
        to_variant = True          # ← 带 marker，标记为"要分家"
...
rez_req = str(packaging_req_to_rez_req(req))

if to_variant:
    result_variant_requires.append(rez_req)   # ← 进变体标签
else:
    result_requires.append(rez_req)           # ← 进普通 requires
```

而 `rez_req` 是 pip 写法翻译成 rez 写法的结果：

```
psutil<8.0,>=7.0.0   →   psutil-7.0.0+<8.0
```

**注意这个翻译后的字符串里，`<` 被原样保留了。** 它接下来要被刻到门牌上。

顺带解释一下门牌上那几个符号的含义，免得误会：`7.0.0+` 表示"从 7.0.0 起（含）"，`<8.0` 表示"到 8.0 之前"。这是 rez 的版本区间写法，`+` 和 `<` 都是它语法的一部分，合法且必要。

---

## 三、事故现场：刻字师傅没有"小于号"这个字模

Windows 对文件/目录名有一条硬规定，这九个字符不许出现：

```
<   >   :   "   /   \   |   ?   *
```

（这是 NTFS/Win32 命名规则的的历史遗留，跟 Linux/macOS 不同——那边只禁 `/` 和 NUL。）

于是：

1. rez-pip 拿着 `psutil-7.0.0+<8.0` 去建目录；
2. 系统回答 `WinError 123 文件名、目录名或卷标语法不正确`；
3. 这间房**根本没建出来**；
4. 而它前面的 `platform-windows/arch-AMD64/python-3.12/` 已经建好了，是个**空壳**；
5. `package.py` 倒是写完了（写文件用的是另一条路径，没受影响）。

实际磁盘上看到的就是这样（修复前）：

```
rez-package-3rd\reflex\0.9.10.post1\platform-windows\arch-AMD64\python-3.12\   ← 空
rez-package-3rd\reflex\0.9.10.post1\package.py                                  ← 在
（整个 reflex 家族目录里文件总数：1）
```

比喻说全一点：

> 登记簿上写着"reflex 楼 0.9.10.post1 单元，Windows 座 AMD64 翼 3.12 层，还有一间 psutil 房"。
> 木匠照着刻门牌，刻到最后一间发现图纸上有个"小于号"，他的刀没有这个字模，刻不出来。
> 但他没吭声，把前四级的门牌挂好就收工了。
> 你在登记簿上一查——"有房"。推门进去——**空的，家具一件没搬进来**。

这就是最迷惑的地方。三层假象叠在一起：

| 你检查什么 | 看到什么 | 真相 |
|---|---|---|
| `wuwor reflex` 解析 | `resolved=44 requested=6 missing=0` | 登记簿上有名字而已 |
| 目录是否存在 | `reflex\0.9.10.post1\...` 存在 | 只有空壳 |
| `package.py` | 内容齐全，`requires` 正确 | 元数据写成功了 |
| `import reflex` | **ModuleNotFoundError** | `{root}/python` 里没有家具 |

wuwo 的日志里其实给了线索，只是不显眼：

```
Skipped [reflex-0.9.10.post1] ...\reflex\0.9.10.post1\package.py
    (platform-windows\arch-AMD64\python-3.12\psutil-7.0.0+<8.0)
```

`Skipped` 是因为它按变体名去找那间房，找不到匹配的（那间房没能建出来）。

---

## 四、rez 官方的解法：把户型名换成档案编号

rez-pip 的作者**完全知道**这件事。他们在代码里留了原话（`rez/pip.py:427-432`）：

```python
# Make the package use hashed variants. This is required because we
# can't control what ends up in its variants, and that can easily
# include problematic chars (>, +, ! etc).
# TODO: #672
#
pkg.hashed_variants = True
```

翻译成人话：

> 我们管不了变体里会塞进什么字符，那些字符（`>` `+` `!` 之类）很可能出格。
> 所以干脆：**别把原名刻门牌了，改成档案编号。**

`hashed_variants = True` 的效果：把变体标签整串做 SHA1，拿 40 位十六进制当目录名。

```
platform-windows / arch-AMD64 / python-3.12 / psutil-7.0.0+<8.0
        ↓ SHA1
6e08c88658c81059153a46d293af5b04c806aaaa
```

十六进制的字符集只有 `0-9 a-f`，**永远不可能**撞上 Windows 的九个禁字。这是"从根上不出格"的解法。

代价也明显：人眼看到 `6e08c886...` 完全不知道是哪一户。

**但要点在于：档案编号只是门牌，登记簿上仍写原名。** 修复后 `reflex/0.9.10.post1/package.py` 里是这样的：

```python
variants = [['platform-windows', 'arch-AMD64', 'python-3.12', 'psutil-7.0.0+<8.0']]

def commands():
    env.PYTHONPATH.append('{root}/python')

hashed_variants = True
```

`variants` 里的语义一个字没改，rez 解析时会用同一套 SHA1 算法把标签换算成编号去对门牌。**语义零损失，只是目录难看。**

> 比喻：门牌上刻"档案号 6e08c886…"，但房产证、户籍登记、快递单上都还是"幸福路 12 号 3 单元"。查的人认编号，住的人记原名，两不耽误。

---

## 五、wuwo 原本的选择：嫌编号难认，吩咐"一律刻原名"

wuwo 的 `_apply_rez_pip_plain_variant_patch`（`auto_fetch_packages.py:2091`）做的事情，就是**把官方那句 `hashed_variants = True` 吞掉**。

它的文档字符串写得很清楚：

```python
def _apply_rez_pip_plain_variant_patch():
    """Idempotent monkeypatch: ignore rez-pip's forced hashed_variants=True.

    rez-pip hardcodes hashed_variants so variant payload dirs are 40-hex SHA1
    names (see rez.pip TODO #672). Skipping that assignment keeps the plain
    readable layout instead (e.g. python-3.12, windows/AMD64/3.12), which rez
    resolves natively. Hashed and plain packages coexist fine in one repo.
    """
```

### 5.1 为什么要吞

工程上的实在好处：

- **可排查**：`dir /b httpcore\1.0.9` 一眼看到 `python-3.12`，不用查表换算。
- **脚本可按名字匹配**：wuwo 里多处逻辑是按目录名找东西的，比如 `cleanup_legacy_py_variant_dirs`（清理老式 `*-py3.12` 目录）、`_mirror_copy_3rd_package`（公盘镜像拷贝）、`_normalize_rez_pip_family_casing`（家族名大小写校正）。
- **绝大多数包只有 `python-3.12` 这一级变体**，本来就合法，没必要变编号。

### 5.2 怎么吞的（Proxy 吞属性赋值）

这段实现值得单独讲，因为它不改 rez 源码、不写补丁文件，而是**运行时套一层壳**：

```python
class _PlainVariantProxy:
    def __init__(self, cm, real_pkg):
        object.__setattr__(self, "_cm", cm)
        object.__setattr__(self, "_pkg", real_pkg)

    def __getattr__(self, name):
        return getattr(object.__getattribute__(self, "_pkg"), name)

    def __setattr__(self, name, value):
        if name == "hashed_variants":
            return          # ← 关键：这一句赋值被丢掉
        setattr(object.__getattribute__(self, "_pkg"), name, value)

    def __enter__(self):
        return self

    def __exit__(self, *args):
        return object.__getattribute__(self, "_cm").__exit__(*args)
```

 rez-pip 写包时用的是 `with make_package(...) as pkg:`。wuwo 把 `rez.pip.make_package` 换成一个返回**代理对象**的版本：

- 代理把除 `hashed_variants` 之外的一切读写**原样转发**给真对象；
- 唯独 `pkg.hashed_variants = True` 这句被"没听见"。

> 比喻：管家不撕图纸，只是吩咐刻字师傅——"图纸上要是写'改用编号'，你就当没看见，照原名刻。"

还做了幂等保护（`_wuwo_plain_variant_patched` 标记），避免重复打补丁。

### 5.3 盲区

管家的吩咐漏了一件事：**原名本身可能是刻不出来的。**

`python-3.12` 永远合法，所以这补丁跑了很久都没事。直到 `reflex` 出现——它的变体里带 `psutil-7.0.0+<8.0`，`<` 是禁字。管家说"照原名刻"，师傅刻不出来，房子就空了。

换句话说：**这个补丁把 rez 官方用来兜底的安全网剪掉了，却没补上第二层。**

---

## 六、本次修复：条件回退（三分支决策）

修的思路不是"改回全量哈希"，也不是"给非法字符做替换"，而是：

> **平时仍用可读原名；一旦发现这个包的原名里有 Windows 禁字，就只对这一个包放行官方的哈希方案。**

### 6.1 非法字符表

```python
# Windows 路径非法字符：变体标签含这些字符时，可读变体目录无法创建。
# 典型来源是 rez-pip 把 requirements 的版本区间塞进 variant_requires，
# 如 psutil<8.0,>=7.0.0 -> "psutil-7.0.0+<8.0"（含 "<"）。
_WIN_PATH_FORBIDDEN_RE = re.compile(r'[<>:"/\\|?*]')
```

九个字符，与 Win32 命名规则一一对应。

> 注意：第四节引的 rez 原话把 `+`、`!` 也列为 problematic，那是**跨平台**的顾虑（`+` 在 URL 里会被解释成空格，`!` 在部分 shell 里是历史展开符）。单就"Windows 能不能建这个目录"而言，`+` 和 `!` 是合法的，所以本表不收——否则会把大量本来能正常工作的包（例如带 local version 的 `1.0+cu121`）无谓地推成哈希目录，白白丢掉可读性。判据要卡在"真的建不出来"上。

### 6.2 判定函数

```python
def _pkg_variants_need_hash(pkg) -> bool:
    """True if any variant label of the package can't be used as a dir name.

    ``pkg.variants`` at this point is a list of variant-requires lists
    (e.g. ``[['platform-windows', 'arch-AMD64', 'python-3.12',
    'psutil-7.0.0+<8.0']]``).
    """
    variants = getattr(pkg, "variants", None) or []
    for variant in variants:
        labels = variant if isinstance(variant, (list, tuple)) else [variant]
        for label in labels:
            if _WIN_PATH_FORBIDDEN_RE.search(str(label)):
                return True
    return False
```

注意它兼容两种形态（单个 list，或直接一串标签），因为 rez 内部 `variants` 是"变体列表的列表"。

### 6.3 在 Proxy 里改成"看一眼再决定"

```python
def __setattr__(self, name, value):
    if name == "hashed_variants" and not _pkg_variants_need_hash(
        object.__getattribute__(self, "_pkg")
    ):
        return  # keep the default (plain dirs) instead of forced True
    setattr(object.__getattribute__(self, "_pkg"), name, value)
```

逻辑：

- `hashed_variants` 赋值来临时，**先看这个包的变体名有没有出格字符**；
- 没出格 → 照旧吞掉（保持可读，行为与从前完全一致）；
- 出格 → **放行**，让 rez 用哈希目录（安全网补回来了）。

### 6.4 为什么这个时机能拿到 variants

顺序是刚好的。`rez/pip.py` 里：

```python
# line 414-415
if variant_requires:
    pkg.variants = [variant_requires]      # ← 先写变体

# line 427-432
# ...comment...
pkg.hashed_variants = True                 # ← 后写哈希开关
```

`variants` 在 `hashed_variants` **之前**赋值，所以代理在后者被赋值时已经能读到前者。若 rez 哪天调换顺序，这个判定会退化成"读不到 variants → 认为不需要哈希"，也就是回到修复前的行为——**不会更糟**，但值得知道。

### 6.5 决策表

| 包的变体标签 | 需要哈希？ | `hashed_variants` | 磁盘目录 | 与修复前差异 |
|---|---|---|---|---|
| `['python-3.12']` | 否 | 被吞 | `python-3.12/` | 无 |
| `['platform-windows','arch-AMD64','python-3.12']` | 否 | 被吞 | 三级可读 | 无 |
| `[..., 'psutil-7.0.0+<8.0']` | **是** | **放行 True** | `6e08c886…/` | **修好了** |

### 6.6 为什么不选另外两条路

**（A）把非法字符替换掉**（`psutil-7.0.0+<8.0` → `psutil-7.0.0+_8.0`）

看着最"可读"，实际最危险：变体标签**不是装饰文字，是 rez 的语义**。`psutil-7.0.0+<8.0` 表示"psutil 版本 ∈ [7.0.0, 8.0)"。改成 `_8.0` 后，rez 会把它理解成家族 `psutil`、版本 `7.0.0+_8.0` 这种鬼东西，**再也匹配不上任何真实的 psutil**。

> 比喻：门牌刻不出来，你就把"幸福路"改成"幸福\_路"。师傅是好刻了，可邮局按"幸福路"投递，永远送不到。

要真走这条路，必须同时教 rez 在解析时做反向翻译，等于给变体名加一层编码协议，成本和风险都远超收益。

**（B）干脆全量用哈希**

安全，但把 wuwo 攒的可读性全丢了，而且会影响按目录名匹配的若干清理/镜像逻辑。为一个"少数派问题"让"多数派"付代价，不划算。

**（C）本次选择：只让中招的那一个包退回哈希**

- 影响面精确到"变体含禁字的包"，其他包一字不变；
- 用的是 rez 原生机制，不需要发明协议；
- 语义完整保留在 `package.py` 的 `variants` 字段里；
- rez 明确支持同一仓库里 hash 与 plain 包共存（补丁文档字符串里那句 "Hashed and plain packages coexist fine in one repo" 就是这个意思）。

---

## 七、顺带修的第二坑：`==1` 与 `==1.*`

修好 `reflex` 能 import 之后，紧接着撞上第二个坑，性质不同但同源——都是"翻译丢信息"。

### 7.1 现象

```
The following package conflicts occurred: (h11-0.16.0+<1 <--!--> h11==0.14.0)

Resolve paths starting from initial requests to conflict:
  reflex --> reflex-0.9.10.post1 --> httpx-0.28.1 --> httpcore-1.0.0 --> h11-0.14.0
```

### 7.2 链条

`httpx` 的 rez 包里写着：

```python
requires = ['httpcore-1', ...]
```

`httpcore-1` 在 rez 里是**前缀语义**，意思是"1 这一族，1.x 都行"。

wuwo 的 `_rez_req_to_pip_spec`（`auto_fetch_packages.py:1630`）负责把 rez 需求串翻译成 pip 说明符。原来的分支：

```python
if lo.endswith("+") and hi.startswith("<") and hi.endswith("_") and hi[1:-1] == lo[:-1]:
    return f"{name}=={lo[:-1]}"        # → "httpcore==1"
```

于是 pip 收到 `httpcore==1`。而 **PEP440 把 `==1` 规范化为 `==1.0.0`**——它不是"1 族任意"，而是"就要 1.0.0"。

pip 于是买了个绝版老型号回来：

```
httpcore 1.0.0  →  requires h11>=0.13,<0.15
wsproto 1.3.2   →  requires h11-0.16.0+<1
```

两个邻居对同一把锁的要求互斥（一个要 0.14 的锁，一个要 0.16 的锁），整个环境 resolve 失败。

> 比喻：你说"要 1 号电池"，采购员听成"要编号就叫『1』的那节电池"，翻出一节绝版 1.0.0 型。装上去发现电池盖要 0.16 规格，锁不上。

### 7.3 修法

```python
if lo.endswith("+") and hi.startswith("<") and hi.endswith("_") and hi[1:-1] == lo[:-1]:
    ver = lo[:-1]
    # rez 的短版本是前缀语义（httpx 的 'httpcore-1' 意为 1.*）。pip 侧必须
    # 写成 ==1.*；写成 ==1 会被 PEP440 规范化为 1.0.0，于是装上老版本并引发
    # 依赖冲突（httpcore 1.0.0 钉 h11<0.15，与 wsproto 要求的 h11>=0.16 互斥，
    # 整个 reflex 环境因此无法 resolve）。
    if "." not in ver:
        return f"{name}=={ver}.*"
    return f"{name}=={ver}"
```

判据很朴素：**版本号里不含 `.` 的，就是前缀写法，必须补 `.*`。**

配套改 `_exact_ver_from_pip_spec`（`auto_fetch_packages.py:1661`），否则 `1.*` 会被当成精确版本参与"目录名前缀必须相等"的复用判断，导致每次都觉得"版本对不上"而反复重装：

```python
ver = spec.split("==", 1)[1].strip()
# '==1.*' 是前缀约束而非精确版本，不能参与"版本目录名前缀必须相等"的复用判断
if ver.endswith(".*"):
    return None
return ver or None
```

---

## 八、排查手册：怎么判断自己中没中招

### 8.1 三个典型特征

同时出现基本可以确诊：

1. `wuwor <包>` 解析显示 `missing=0`（登记簿上有名）；
2. 但 `import <包>` 报 `ModuleNotFoundError`；
3. `<包>\<版本>\` 下的变体目录存在，却是空的。

### 8.2 一眼看穿：数文件

在 `rez-package-3rd` 下数某个家族的真实文件数。**正常包至少几十上百个，空壳只有 1 个（package.py）**。

```bat
cd /d D:\TD_Depot\Software\Lugwit_syncPlug\lugwit_insapp\trayapp\rez-package-3rd
for /f %i in ('dir /s /b /a-d reflex 2^>nul ^| find /c /v ""') do @echo files=%i
```

### 8.3 看变体名有没有出格字符

```bat
findstr /I "variants" reflex\0.9.10.post1\package.py
```

若 `variants` 里出现 `<` `>` `:` `"` `|` `?` `*`，且 `hashed_variants` 不为 True，就是中招。

### 8.4 看变体目录链

```bat
dir /b /s /ad reflex
```

正常应到 `...\python-3.12\python`（里面有货）。若停在某一层且**该层为空**，就是那一级门牌没刻出来。

### 8.5 日志里的关键词

```bat
wuwor <包> -- python -c "import <包>" 2>&1 | findstr /I "Skipped WinError hashed Installed"
```

- `Skipped [<包>-<版本>] ... (<一长串变体>)` → 按变体名找不到房，多半是空壳；
- `WinError 123` → 确诊非法字符；
- `Installed [<包>-<版本>] ... (<40位十六进制>)` → 走了哈希，正常落地。

### 8.6 万一是"传递依赖"缺失，别手动删家族目录

踩过的坑：把 `httpcore` 整个移走想让它重装，结果

```
PackageFamilyNotFoundError: package family not found: httpcore, was required by: httpx
```

原因：wuwo 的 auto_fetch 是在**resolve 之前**收集"根请求"的闭包。根包 `reflex` 还在，它就认为一切正常，不会去补传递依赖的缺口。

正确做法：用家族名显式请求，让 auto_fetch 把它当根包处理。

```bat
wuwor httpcore -- python -c "import httpcore;print(httpcore.__version__)"
```

（注意 `wuwor httpcore-1.0.9` 这种带版本的请求，走的是"wanted_ver 精确匹配"，可能反而不触发安装；用裸家族名最省事。）

---

## 九、验证：reflex 案例前后对比

| 项 | 修复前 | 修复后 |
|---|---|---|
| `reflex` 家族文件数 | 1（只有 package.py） | 完整载荷 |
| 变体目录 | `platform-windows\arch-AMD64\python-3.12\`（空） | `6e08c88658c81059153a46d293af5b04c806aaaa\` |
| `import reflex` | ModuleNotFoundError | 成功，0.9.10.post1 |
| `reflex --version` | 不可用 | `0.9.10.post1` |
| httpcore | 1.0.0（钉 h11<0.15，冲突） | 1.0.9（h11>=0.16，通过） |
| 环境 resolve | 冲突失败 | 67 包全解析 |
| `mindmap_reflex_doctor` | — | ALL PASS (5/5) |

`package.py` 的落地形态（语义与原名都在，只有目录变编号）：

```python
name = 'reflex'
version = '0.9.10.post1'
requires = [...]
variants = [['platform-windows', 'arch-AMD64', 'python-3.12', 'psutil-7.0.0+<8.0']]

def commands():
    env.PYTHONPATH.append('{root}/python')
    env.PATH.append('{root}/bin')

hashed_variants = True
```

---

## 十、影响面与回归风险

- **精确命中**：只有"变体标签含 Windows 禁字"的包行为改变。本仓库里 `reflex` 是唯一已知的；`reflex_base`、`httpcore`、`h11`、`wsproto`、`granian` 等全部仍是 `python-3.12` 这类可读目录，一字未变。
- **已装包不受影响**：monkeypatch 只在**安装时**起作用，不会去改已落地的目录。要重装某个包才需要触发（`force_reinstall`，或把该家族目录改名让位后 `wuwor <裸家族名>`）。
- **公盘镜像 / 清理脚本**：`_mirror_copy_3rd_package`、`cleanup_legacy_py_variant_dirs` 匹配的是**版本目录名**（`0.9.10.post1`、`*-py3.12`），哈希发生在版本目录**下一层**，不受影响。
- **`_exact_ver_from_pip_spec` 的改动**：只让 `==X.*` 返回 `None`（视作前缀约束）。原本返回精确版本的路径不变。
- **潜在退化点**：若 rez 未来把 `pkg.variants` 的赋值挪到 `hashed_variants` 之后，判定会读不到 variants 从而退回 plain 行为（即恢复修复前的毛病，不会新增崩溃）。届时可改为在 `__enter__` 后延迟决策或读 `variant_requires` 原始入参。

---

## 十一、给后来人的四条守则

1. **`import` 失败先怀疑"空壳"，别急着重装。** 数一下家族目录文件数只要 5 秒，能省掉半小时瞎猜。别一上来就 `pip install` 硬补——那会绕过 wuwo 的依赖治理。
2. **不要在 `package.py` 里写 `env.PYTHONPATH.append(r"E:\...")` 这类硬编码补依赖。** 缺什么就写进 `requires`，交给 wuwo 自动解析下载（见《Rez包创建和启动指导文档》"自动下载依赖"一节）。本次 reflex 的问题恰恰是"写进 requires 了但装坏了"，正解是修装配层，不是绕过它。
3. **凡是"把变体/标签当目录名"的地方，都要过一遍禁字表。** 这不只是 Windows 的事——Linux 上 `:` 会毁掉 `PATH`，空格会毁掉 shell 脚本。
4. **monkeypatch 剪掉官方安全网时，必须补上第二层。** `_apply_rez_pip_plain_variant_patch` 为了可读性吞掉 `hashed_variants`，本身是个合理取舍，但它默认"变体名总是干净的"，这个假设没写在任何地方，也没人兜底。以后再加类似 patch，请照本次的样子加一条"出格就退回官方行为"的分支。

---

## 附：本文涉及的两处改动原文

### A. `auto_fetch_packages.py` 变体哈希条件回退

```python
_WIN_PATH_FORBIDDEN_RE = re.compile(r'[<>:"/\\|?*]')


def _pkg_variants_need_hash(pkg) -> bool:
    variants = getattr(pkg, "variants", None) or []
    for variant in variants:
        labels = variant if isinstance(variant, (list, tuple)) else [variant]
        for label in labels:
            if _WIN_PATH_FORBIDDEN_RE.search(str(label)):
                return True
    return False
```

```python
    class _PlainVariantProxy:
        def __setattr__(self, name, value):
            if name == "hashed_variants" and not _pkg_variants_need_hash(
                object.__getattribute__(self, "_pkg")
            ):
                return  # keep the default (plain dirs) instead of forced True
            setattr(object.__getattribute__(self, "_pkg"), name, value)
```

### B. `auto_fetch_packages.py` 短版本前缀修正

```python
    if lo.endswith("+") and hi.startswith("<") and hi.endswith("_") and hi[1:-1] == lo[:-1]:
        ver = lo[:-1]
        if "." not in ver:
            return f"{name}=={ver}.*"
        return f"{name}=={ver}"
```

```python
    ver = spec.split("==", 1)[1].strip()
    if ver.endswith(".*"):
        return None
    return ver or None
```

---

## 变更记录

| 日期 | 内容 |
|---|---|
| 2026-09-05 | 首版。记录 reflex 变体目录名含 `<` 导致空壳的完整根因、rez 官方 hashed_variants 机制、wuwo plain variant patch 的取舍与盲区、本次条件回退方案，以及 `httpcore==1` → `==1.*` 的第二坑与排查手册。 |
