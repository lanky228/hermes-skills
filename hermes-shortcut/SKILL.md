---
name: hermes-shortcut
description: 仅在用户要求创建 Hermes Agent 桌面快捷方式时使用。在常见安装位置搜索 Hermes 可执行文件，并在桌面上创建 .lnk 快捷方式。
---

# Hermes 桌面快捷方式

为 Hermes Agent CLI 创建 Windows 桌面快捷方式（.lnk）。

## 查找 Hermes 可执行文件

按以下顺序搜索，使用 `Test-Path -LiteralPath <路径>` 验证每个候选，找到第一个即停止。

1. **虚拟环境**（最常见 — `pip install -e` 或 `uv sync` 安装）：
   ```
   <hermes仓库>\venv\Scripts\hermes.exe
   ```
   仓库通常位于 `%LOCALAPPDATA%\hermes\hermes-agent`，因此完整路径为：
   ```
   %LOCALAPPDATA%\hermes\venv\Scripts\hermes.exe
   ```

2. **Python 用户目录 Scripts**（`pip install --user` 安装）：
   ```
   %APPDATA%\Python\Python3XX\Scripts\hermes.exe
   ```

3. **系统级 Python Scripts**：
   ```
   C:\Program Files\Python3XX\Scripts\hermes.exe
   ```

4. **广度搜索**（最后手段）：
   ```powershell
   Get-ChildItem -LiteralPath "$env:LOCALAPPDATA\hermes" -Recurse -Filter "hermes.exe" -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty FullName
   ```

5. **回退方案** — 找不到 `.exe` 时：使用仓库中的 Python 入口脚本。
   ```
   <hermes仓库>\hermes
   ```
   此时用 `python.exe` 作为 `TargetPath`，脚本路径作为 `Arguments`（见下方回退模板）。

## 创建快捷方式

使用 `WScript.Shell` 创建 `.lnk`。`WorkingDirectory` 必须设为仓库根目录 — CLI 依赖它来解析 skills、plugins 和 config 的相对路径。

### 主方案：找到 hermes.exe

```powershell
$Desktop = [Environment]::GetFolderPath("Desktop")
$WshShell = New-Object -ComObject WScript.Shell
$Shortcut = $WshShell.CreateShortcut("$Desktop\Hermes.lnk")
$Shortcut.TargetPath = "<hermes.exe的完整路径>"
$Shortcut.WorkingDirectory = "<hermes仓库根目录>"
$Shortcut.Description = "Hermes Agent"
$Shortcut.Save()
```

### 回退方案：Python + 脚本

```powershell
$Desktop = [Environment]::GetFolderPath("Desktop")
$WshShell = New-Object -ComObject WScript.Shell
$Shortcut = $WshShell.CreateShortcut("$Desktop\Hermes.lnk")
$Shortcut.TargetPath = "<python.exe的完整路径>"
$Shortcut.Arguments = "<hermes仓库根目录>\hermes"
$Shortcut.WorkingDirectory = "<hermes仓库根目录>"
$Shortcut.Description = "Hermes Agent"
$Shortcut.Save()
```

### 字段说明

| 字段 | 是否必填 | 说明 |
|------|----------|------|
| `TargetPath` | 是 | `hermes.exe` 的完整路径（回退方案中为 `python.exe`） |
| `Arguments` | 仅回退方案 | hermes 脚本的完整路径 |
| `WorkingDirectory` | 是 | hermes 仓库根目录 |
| `Description` | 否 | `"Hermes Agent"` |
| `IconLocation` | 否 | 不设置则使用默认控制台图标 |

## 验证

```powershell
$Desktop = [Environment]::GetFolderPath("Desktop")
$Shortcut = (New-Object -ComObject WScript.Shell).CreateShortcut("$Desktop\Hermes.lnk")
$Shortcut.TargetPath; $Shortcut.Arguments; $Shortcut.WorkingDirectory
```

## 注意事项

- 如果快捷方式已存在，`CreateShortcut` 会静默覆盖。
- `[Environment]::GetFolderPath("Desktop")` 能正确处理 OneDrive 重定向和自定义桌面路径。
- `WorkingDirectory` 至关重要 — CLI 依赖仓库根目录来解析 skills、plugins 和 config 的相对路径。