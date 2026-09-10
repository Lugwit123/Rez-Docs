# Electron 程序启动崩溃排查记录（VS Code / 0xC0000005）

> 记录时间：2026-09-10 11:19
> 机器：`wb.fengqingqing` 工作机
> 现象：VS Code 一启动就崩（进程退出码 `0xC0000005`，访问冲突）

---

## 一、一句话结论

**VS Code 本身没坏。** 真正的原因是它的"上一级程序"在启动时往环境里塞了两个变量：

| 变量 | 值 | 危害 |
|---|---|---|
| `ELECTRON_DISABLE_SANDBOX` | `1` | 关掉 Chromium 沙箱 → 启动即 `0xC0000005` 崩溃 |
| `ELECTRON_RUN_AS_NODE` | `1` | 把 Electron 降级成无窗口的纯 Node → 起不来界面 |

把这两个变量从**用户级环境变量**里清掉后，双击 VS Code 恢复正常。

---

## 二、根因分析

### 2.1 变量从哪来

这两个变量**不在注册表**（排查时注册表已是干净的），而是 **Wuzu 客户端在运行期动态注入**给它自己派生的子进程。

**实证**：排查时所在的那个终端本身就是 Wuzu 的子进程，其中依然能看到：

```
ELECTRON_DISABLE_SANDBOX = [1]
ELECTRON_RUN_AS_NODE     = [1]
ELECTRON_DISABLE_SECURITY_WARNINGS = [true]
```

→ 证明它们是**按进程树继承**的注入变量，不落注册表。

### 2.2 为什么"双击能开、从 Wuzu 里开就崩"

| 启动方式 | 环境变量来源 | 结果 |
|---|---|---|
| 资源管理器 / 开始菜单 双击 | 注册表（用户级 + 系统级） | ✅ 正常（变量已清干净） |
| 桌面 `启动VSCode.bat` | 脚本内先 `set "X="` 清空 | ✅ 正常（万能兜底） |
| Wuzu 客户端内部 / Wuzu 派生的终端里敲 `code` | Wuzu 进程注入 | ❌ 仍会崩 |

关键机制：

```
资源管理器双击   → 继承注册表环境（干净）→ 变量不存在 → VS Code 正常启动
Wuzu 派生终端    → 继承 Wuzu 内存环境（脏）→ 变量存在  → 崩溃
```

---

## 三、实际做过的改动（共 2 处）

### ① 删除注册表里的两个变量 ← **真正生效的修复**

从 `HKCU\Environment`（用户级环境变量，双击程序继承的正是这一层）移除：

- `ELECTRON_DISABLE_SANDBOX`
- `ELECTRON_RUN_AS_NODE`

**实证**：该注册表键最后写入时间为 **2026-09-10 11:13:00**（删除动作会刷新键写入时间），且当前键值列表中已无任何 `ELECTRON*` 条目。

> 删除命令参考（如需复现）：
>
> ```powershell
> Remove-ItemProperty -Path 'HKCU:\Environment' -Name 'ELECTRON_DISABLE_SANDBOX' -ErrorAction SilentlyContinue
> Remove-ItemProperty -Path 'HKCU:\Environment' -Name 'ELECTRON_RUN_AS_NODE'     -ErrorAction SilentlyContinue
> ```

### ② 桌面创建 `启动VSCode.bat`（保险，非主因）

路径：`C:\Users\wb.fengqingqing\Desktop\启动VSCode.bat`

作用：启动前先把变量清空，**即使父进程是脏的也能启动**。

```bat
@echo off
chcp 65001 >nul
rem VS Code 一键启动（清除被注入的 Electron 变量）
set "ELECTRON_DISABLE_SANDBOX="
set "ELECTRON_RUN_AS_NODE="
set "ELECTRON_DISABLE_SECURITY_WARNINGS="
start "" "<VS Code 安装路径>\Code.exe"
```

---

## 四、排除的干扰项（当时差点误判）

| 嫌疑对象 | 结论 |
|---|---|
| `QzhddrGuard64.dll`（EDR 杀软注入） | ❌ 无关 |
| `ai_agent.dll` | ❌ 无关 |

这两个当时都被列进过怀疑清单，实测与本次崩溃无关，**不要**把它们当成元凶去处理。

---

## 五、排查用到的检查命令（备查）

```powershell
# 1) 用户级环境变量（双击程序继承这一层）
Get-ItemProperty 'HKCU:\Environment'

# 2) 系统级环境变量
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Environment'

# 3) 当前进程（验证子进程是否被注入）
Get-ChildItem Env: | Where-Object Name -like 'ELECTRON*'

# 4) 注册表键最后写入时间（判断是否被改动过）
#    读取 HKCU\Environment 的 LastWriteTime
```

---

## 六、后续注意事项

1. **日常使用**：从桌面 / 开始菜单正常双击 VS Code 即可，现在是好的。
2. **唯一要避开的入口**：不要从 Wuzu 客户端内部或它派生的终端里打开 VS Code。
3. **哪天又崩了**：八成是某个工具（安装器 / 某次"修复"）把变量写回了注册表。
   - 应急：双击桌面 `启动VSCode.bat`（万能兜底）
   - 根治：重新检查 `HKCU:\Environment` 有无 `ELECTRON*` 残留
4. **同类问题推广**：任何 Electron 应用（VS Code / Cursor / Trae / Qoder 等）出现"启动即崩"，优先查 `ELECTRON_DISABLE_SANDBOX` 与 `ELECTRON_RUN_AS_NODE` 这两个变量。

---

## 七、附：本次会话相关的输入文件

- `.cowork-temp/vscode-verify-4/User/globalStorage/vscode.git`
- `.cowork-temp/vscode-verify-4/CachedConfigurations/defaults/__default__profile__-configurationDefaultsOverrides/configuration.json`
- `.cowork-temp/claude/.../tasks/bkpjcbx5p.output`
- `.cowork-temp/claude/.../tasks/bki3mou7o.output`
