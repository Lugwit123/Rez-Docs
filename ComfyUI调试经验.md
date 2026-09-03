# ComfyUI 调试经验

## 1. Node Flag 图标不显示

### 现象
节点标题栏右侧的 flag 按钮（⏭️🐛📌⭐🔒）图标不可见，按钮看起来是空的。

### 根因排查过程

| 步骤 | 排查内容 | 发现 |
|------|---------|------|
| 1 | 检查 DOM 是否有 SVG | SVG 存在但 `computed width = 0px` |
| 2 | 检查按钮尺寸 | `w-3.5 h-4` = 14×16px |
| 3 | 检查 CSS | user-agent 默认 `padding: 6px`，14px 按钮内实际内容区 = 0px |

### 修复

1. **按钮加 `p-0`** 去除默认内边距，改 `w-4 h-4`（16×16）
2. **SVG 尺寸** 从 12×12 调大至 14×14，`stroke-width="2.5"`
3. **改用 emoji** 替代 SVG，资源更省、DOM 更简：

```html
<!-- 修复前 -->
<button class="flex w-3.5 h-4 ... border">  <!-- 14×16, 默认padding 6px → 内容区0px -->
  <svg width="12" height="12" stroke-width="2">...</svg>  <!-- 实际线条仅1px -->
</button>

<!-- 修复后 -->
<button class="flex w-4 h-4 ... border p-0">  <!-- 16×16, 无padding -->
  <span :style="{ filter: active ? 'none' : 'brightness(0.5) saturate(0.3)' }">⏭️</span>
</button>
```

### Emoji 状态视觉规范

| 状态 | filter | 边框 | 背景 |
|------|--------|------|------|
| 激活 | `none` | 原色 | 原色填充 |
| 未激活 | `brightness(0.5) saturate(0.3)` | `dimColor()` 暗色 | `transparent` |

```typescript
/** 降饱和度/亮度（RGB 各通道与灰色 45:55 混合） */
function dimColor(hex: string): string {
  const r = parseInt(hex.slice(1, 3), 16)
  const g = parseInt(hex.slice(3, 5), 16)
  const b = parseInt(hex.slice(5, 7), 16)
  const mix = (c: number) => Math.round(c * 0.45 + 128 * 0.55)
  return `#${mix(r).toString(16).padStart(2, '0')}${mix(g).toString(16).padStart(2, '0')}${mix(b).toString(16).padStart(2, '0')}`
}
```

### 教训
- **button 默认有 padding**，小按钮必须加 `p-0`
- **SVG stroke 在小尺寸下几乎不可见**：`stroke-width / viewBox * renderSize` 计算实际线宽
- **Emoji 比 SVG 更省资源**：单个 `<span>` vs `<svg>` + 多个 `<path>`
- Emoji 颜色不受 CSS 控制，用 `filter` 降饱和/亮度实现灰暗态

---

## 2. GetInstance 节点输入框不是下拉框

### 现象
Get Instance of Class 节点的 `class_path` 输入框是文本输入框（STRING widget），不是带搜索的下拉框（COMBO widget）。

### 根因
类列表扫描是异步的（`threading.Thread`），`INPUT_TYPES` 是同步调用的类方法。首次调用时扫描尚未完成，代码 fallback 到 STRING widget。

```python
# 旧代码：扫描未完 → STRING
available = _discover_classes()  # 0.5s 超时，未完返回 []
if not available:
    return { "class_path": ("STRING", {...}) }  # 文本输入
```

### 修复

1. **模块导入时立即启动扫描**（不再等 `INPUT_TYPES` 调用）：

```python
# 模块底部
_start_async_scan()
```

2. **`_discover_classes()` 等待 5 秒**（而非 0.5 秒立即返回）：

```python
def _discover_classes() -> List[str]:
    if _SCAN_THREAD and _SCAN_THREAD.is_alive():
        _SCAN_THREAD.join(timeout=5.0)  # 等 5 秒
    ...
```

3. **始终返回 COMBO 格式**，不再有 STRING 降级：

```python
@classmethod
def INPUT_TYPES(cls):
    available = _discover_classes()
    return {
        "required": {
            "class_path": (available or [""], {"default": available[0] if available else ""}),
        }
    }
```

### 教训
- **ComfyUI 的 COMBO widget** 接收 `(list, options_dict)` 格式
- **异步初始化与同步 API 的矛盾**：`INPUT_TYPES` 是同步的，必须在调用前完成数据准备
- **模块导入时机**：利用 Python 模块级别的代码在 `import` 时执行，提前启动耗时操作

---

## 3. Badge 图标用 CSS mask-image 不可靠

### 现象
底部 badge 的 `icon-[lucide--package]` 图标不显示。

### 根因
Tailwind CSS v4 + `@tailwindcss/vite` 的 JIT 编译器只扫描源码中的**静态** class 名。动态绑定的 `icon-[lucide--package]` 类名不会被编译，CSS 规则未生成。

即使添加 hidden div safelist 也无效，因为 Tailwind v4 的扫描机制不同。

### 修复
放弃 CSS mask-image，改用**内联 SVG**（`v-html`）：

```html
<!-- 修复前（不生效） -->
<i :class="'icon-[lucide--package] size-3.5'" />

<!-- 修复后（可靠） -->
<svg width="10" height="10" viewBox="0 0 24 24"
     fill="none" stroke="currentColor" stroke-width="2"
     v-html="iconSvg" />
```

### 教训
- **Tailwind CSS Icons 不支持动态类名**，必须静态出现在源码中
- **内联 SVG** 是最可靠的小图标方案，不依赖 CSS 编译
- `v-html` 直接注入 SVG path 数据，无需构建工具配合

---

## 4. 自动生成节点函数必须显式 return

### 现象
节点执行成功但下游节点收到 `None`，`SaveText` 节点 `preview_text` 为空。

### 根因
ComfyUI 的 `auto_node_scanner` 将普通 Python 函数包装为节点，函数的返回值直接传递给下游。如果函数只用 `print()` 输出、没有 `return`，下游收到 `None`。

### 修复
```python
# 修复前
def list_files(directory):
    result_str = ...
    print(result_str)  # ❌ print 不会传递给下游

# 修复后
def list_files(directory):
    result_str = ...
    print(result_str)
    return result_str  # ✅ 必须显式 return
```

---

## 5. lprint 日志被 loop.is_running() 阻断

### 现象
ComfyUI 前端控制台日志面板为空，但后端日志文件有内容。

### 根因
`lprint` 函数中 `if not loop.is_running(): return` 在事件循环已创建但尚未开始时返回 `False`，导致所有日志广播被跳过。

### 修复
```python
# 修复前
if not loop.is_running():  # 过于严格
    return

# 修复后
if loop is None or loop.is_closed():  # 仅检查不存在/已关闭
    return
```

---

## 6. bottle 中 url_root 不可用

### 现象
使用 `request.url_root` 获取根 URL 时行为异常。

### 根因
bottle 框架的 `url_root` 属性不可靠，应使用 `request.urlparts`。

### 修复
```python
# 错误
root = request.url_root

# 正确
from urllib.parse import urlunparse
parts = request.urlparts
root = urlunparse((parts.scheme, parts.netloc, '', '', '', ''))
```

---

## 7. Windows 热重载端口冲突

### 现象
热重载时端口报 `OSError 10048`，但控制台无错误输出。

### 根因
端口冲突错误被写入独立日志文件，未输出到主控制台。

### 修复
检查热重载进程的独立日志文件，确认是否为端口 `TIME_WAIT` 残留。

---

## 8. 前端调试工具链

### 浏览器开发者工具

```javascript
// 检查 DOM 中元素的实际渲染尺寸
const el = document.querySelector('.your-class')
const cs = getComputedStyle(el)
console.log(cs.width, cs.height, cs.display, cs.visibility)

// 检查 SVG 是否渲染
const svg = document.querySelector('svg')
const rect = svg.getBoundingClientRect()
console.log(rect.width, rect.height, svg.innerHTML.slice(0, 100))

// 检查 mask-image 是否生效
console.log(getComputedStyle(el).maskImage)
```

### 常用验证步骤

1. **DOM 存在？** `querySelector` 确认元素在 DOM 中
2. **尺寸正常？** `getBoundingClientRect()` 确认宽高 > 0
3. **可见性？** `getComputedStyle` 检查 `display/visibility/opacity`
4. **CSS 生效？** 检查 computed style 是否有预期值
5. **硬刷新** `Ctrl+Shift+R` 排除浏览器缓存

---

## 9. 后端 API 调试123

### 验证节点注册

```bash
# 检查 object_info 中是否有节点
curl -s http://localhost:8701/object_info | python -c "import sys,json; d=json.load(sys.stdin); print([k for k in d if 'GetInstance' in k])"
```

### 验证类扫描

```python
# 在 Python 中直接测试扫描
from comfyui_backend.get_instance_node import _do_discover
classes = _do_discover()
print(f"发现 {len(classes)} 个类: {classes[:5]}")
```

### 查看后端日志

后端日志输出到 `day5_20_20260719.md` 文件（路径见 `project_environment_configuration` 记忆）。
