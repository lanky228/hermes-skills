---
name: hermes-shortcut
description: Use ONLY when the user asks to create a Hermes Agent desktop shortcut on Windows. Searches for the Hermes executable in common installation locations and creates a .lnk shortcut on the desktop.
---

# Hermes Desktop Shortcut

Create a Windows desktop shortcut (.lnk) for the Hermes Agent CLI.

## Finding the Hermes executable

Search these locations in order. Use `Test-Path -LiteralPath <path>` to verify each candidate. Stop at the first match.

1. **Virtual environment** (most common — `pip install -e` or `uv sync`):
   ```
   <hermes-repo>\venv\Scripts\hermes.exe
   ```
   The repo is typically at `%LOCALAPPDATA%\hermes\hermes-agent`, making the exe path:
   ```
   %LOCALAPPDATA%\hermes\venv\Scripts\hermes.exe
   ```

2. **Python user Scripts** (`pip install --user`):
   ```
   %APPDATA%\Python\Python3XX\Scripts\hermes.exe
   ```

3. **System-wide Python Scripts**:
   ```
   C:\Program Files\Python3XX\Scripts\hermes.exe
   ```

4. **Broad search** (last resort):
   ```
   Get-ChildItem -LiteralPath "$env:LOCALAPPDATA\hermes" -Recurse -Filter "hermes.exe" -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty FullName
   ```

5. **Fallback** — no `.exe` found: use the repo's Python entry point script.
   ```
   <hermes-repo>\hermes
   ```
   In this case, use `python.exe` as `TargetPath` and the script path as `Arguments` (see fallback template below).

## Creating the shortcut

Use `WScript.Shell` to create the `.lnk`. The `WorkingDirectory` must be the repo root — the CLI resolves skills, plugins, and config relative to it.

### Primary: hermes.exe found

```powershell
$Desktop = [Environment]::GetFolderPath("Desktop")
$WshShell = New-Object -ComObject WScript.Shell
$Shortcut = $WshShell.CreateShortcut("$Desktop\Hermes.lnk")
$Shortcut.TargetPath = "<path-to-hermes.exe>"
$Shortcut.WorkingDirectory = "<hermes-repo-root>"
$Shortcut.Description = "Hermes Agent"
$Shortcut.Save()
```

### Fallback: Python + script

```powershell
$Desktop = [Environment]::GetFolderPath("Desktop")
$WshShell = New-Object -ComObject WScript.Shell
$Shortcut = $WshShell.CreateShortcut("$Desktop\Hermes.lnk")
$Shortcut.TargetPath = "<path-to-python.exe>"
$Shortcut.Arguments = "<hermes-repo-root>\hermes"
$Shortcut.WorkingDirectory = "<hermes-repo-root>"
$Shortcut.Description = "Hermes Agent"
$Shortcut.Save()
```

### Fields

| Field | Required | Value |
|-------|----------|-------|
| `TargetPath` | Yes | Full path to `hermes.exe` (or `python.exe` for fallback) |
| `Arguments` | Fallback only | Full path to the hermes script |
| `WorkingDirectory` | Yes | Hermes agent repo root |
| `Description` | No | `"Hermes Agent"` |
| `IconLocation` | No | Leave unset for default console icon |

## Verification

```powershell
$Desktop = [Environment]::GetFolderPath("Desktop")
$Shortcut = (New-Object -ComObject WScript.Shell).CreateShortcut("$Desktop\Hermes.lnk")
$Shortcut.TargetPath; $Shortcut.Arguments; $Shortcut.WorkingDirectory
```

## Troubleshooting

- If the shortcut already exists, `CreateShortcut` overwrites it silently.
- `[Environment]::GetFolderPath("Desktop")` handles OneDrive-redirected and custom desktop paths correctly.
- `WorkingDirectory` is critical — the CLI resolves relative paths to skills, plugins, and config from the repo root.