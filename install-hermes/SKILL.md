---
name: install-hermes
description: 在中国大陆安装 Hermes Agent（Nous Research 的 AI 编程助手）。仅在用户明确要求安装 Hermes、Hermes Agent 或 hermes-agent 时使用。核心策略：所有下载必须走国内镜像，并行测试多个源选最快，避免卡死。
---

# 在中国大陆安装 Hermes Agent

Hermes Agent 是 Nous Research 开发的 AI 编程助手，支持终端交互、多平台消息接入、技能系统等。

## 核心原则

GitHub 在国内下载极慢或直接阻断。**每一步下载都必须走国内镜像，并行测试多个源，选最快的用，绝不卡在一个源上等超时。**

## 已验证的镜像源

| 镜像 | 速度 | 用途 | 代理 URL 格式 |
|------|------|------|------|
| `ghfast.top` | 最快 | GitHub 文件代理 | `https://ghfast.top/<完整 GitHub URL>` |
| `pypi.tuna.tsinghua.edu.cn` | 快 | PyPI 包代理 | `--index-url https://pypi.tuna.tsinghua.edu.cn/simple` |
| `kkgithub.com` | 一般 | GitHub 文件代理 | `https://kkgithub.com/<org>/<repo>/releases/download/...` |
| `mirror.ghproxy.com` | 慢/容易超时 | GitHub 文件代理 | `https://mirror.ghproxy.com/<完整 GitHub URL>` |

**每次下载都要并行测试多个源，先用完的那个就是最快的。**

## 安装步骤

### 第 1 步：安装 uv（Python 包管理器）

并行测试多个镜像下载 uv，竞速选最快的解压：

```powershell
$ErrorActionPreference = 'Stop'
$ProgressPreference = 'SilentlyContinue'

$uvDir = "$env:LOCALAPPDATA\hermes\bin"
New-Item -ItemType Directory -Path $uvDir -Force | Out-Null

$urls = @(
    'https://ghfast.top/https://github.com/astral-sh/uv/releases/latest/download/uv-x86_64-pc-windows-msvc.zip',
    'https://kkgithub.com/astral-sh/uv/releases/latest/download/uv-x86_64-pc-windows-msvc.zip',
    'https://mirror.ghproxy.com/https://github.com/astral-sh/uv/releases/latest/download/uv-x86_64-pc-windows-msvc.zip'
)

$zip = "$env:TEMP\uv.zip"
$tmpFiles = @()
$jobMap = @{}
$jobs = $urls | ForEach-Object {
    $tmpFile = "$env:TEMP\uv_$(Get-Random).zip"
    $tmpFiles += $tmpFile
    $job = Start-Job -ScriptBlock {
        param($url, $out)
        try {
            Invoke-WebRequest -Uri $url -OutFile $out -UseBasicParsing -TimeoutSec 60
            $true
        } catch { $false }
    } -ArgumentList $_, $tmpFile
    $jobMap[$job.Id] = $tmpFile
    $job
}

$success = $false
while (@($jobs | Where-Object State -eq 'Running').Count -gt 0) {
    $completed = $jobs | Where-Object State -eq 'Completed'
    foreach ($job in $completed) {
        $result = Receive-Job -Job $job
        if ($result) {
            Copy-Item -LiteralPath $jobMap[$job.Id] -Destination $zip -Force
            $success = $true
        }
        Remove-Job -Job $job -ErrorAction SilentlyContinue
        $jobs = $jobs | Where-Object { $_ -ne $job }
        if ($success) { break }
    }
    if ($success) { break }
    Start-Sleep -Milliseconds 500
}
$jobs | Remove-Job -Force -ErrorAction SilentlyContinue
$tmpFiles | ForEach-Object { Remove-Item -LiteralPath $_ -Force -ErrorAction SilentlyContinue }
if (-not $success) { throw "无法下载 uv：所有镜像源均失败" }

Expand-Archive -Path $zip -DestinationPath $uvDir -Force
Remove-Item -LiteralPath $zip -Force
if ($env:Path -split ';' -notcontains $uvDir) {
    $env:Path = "$uvDir;$env:Path"
}
& "$uvDir\uv.exe" --version
```

### 第 2 步：安装 Python 3.11（通过 uv + 镜像）

这是最容易卡住的步骤。uv 默认从 GitHub 下载 Python，必须设置 `UV_PYTHON_INSTALL_MIRROR` 环境变量。
**注意：镜像 URL 必须是完整基础路径（包含 `https://github.com/...` 部分），不能只写域名。**

```powershell
$ErrorActionPreference = 'Stop'
$ProgressPreference = 'SilentlyContinue'

$uvExe = "$env:LOCALAPPDATA\hermes\bin\uv.exe"
$env:UV_PYTHON_INSTALL_MIRROR = "https://ghfast.top/https://github.com/astral-sh/python-build-standalone/releases/download"
if ($env:Path -split ';' -notcontains "$env:LOCALAPPDATA\hermes\bin") {
    $env:Path = "$env:LOCALAPPDATA\hermes\bin;$env:Path"
}
& $uvExe python install 3.11
if ($LASTEXITCODE -ne 0) { throw "Python 3.11 安装失败" }
```

### 第 3 步：安装 Git（PortableGit，免管理员权限）

```powershell
$ErrorActionPreference = 'Stop'
$ProgressPreference = 'SilentlyContinue'

$gitDir = "$env:LOCALAPPDATA\hermes\git"
New-Item -ItemType Directory -Path $gitDir -Force | Out-Null

$file = "$env:TEMP\PortableGit.7z.exe"
$urls = @(
    'https://ghfast.top/https://github.com/git-for-windows/git/releases/download/v2.54.0.windows.1/PortableGit-2.54.0-64-bit.7z.exe',
    'https://kkgithub.com/git-for-windows/git/releases/download/v2.54.0.windows.1/PortableGit-2.54.0-64-bit.7z.exe',
    'https://mirror.ghproxy.com/https://github.com/git-for-windows/git/releases/download/v2.54.0.windows.1/PortableGit-2.54.0-64-bit.7z.exe'
)

$downloaded = $false
foreach ($url in $urls) {
    try {
        Invoke-WebRequest -Uri $url -OutFile $file -UseBasicParsing -TimeoutSec 120
        $downloaded = $true; break
    } catch { continue }
}
if (-not $downloaded) { throw "无法下载 PortableGit：所有镜像源均失败" }

Start-Process -FilePath $file -ArgumentList "-o`"$gitDir`"", "-y" -NoNewWindow -Wait
$env:Path = "$gitDir\cmd;$gitDir\bin;$gitDir\usr\bin;$env:Path"
& "$gitDir\cmd\git.exe" --version
if ($LASTEXITCODE -ne 0) { throw "Git 安装失败" }

Remove-Item -LiteralPath $file -Force

[Environment]::SetEnvironmentVariable("HERMES_GIT_BASH_PATH", "$gitDir\bin\bash.exe", "User")

# 将 Git 目录持久化到 User PATH
$userPath = [Environment]::GetEnvironmentVariable("Path", "User")
$gitPaths = @("$gitDir\cmd", "$gitDir\bin", "$gitDir\usr\bin")
$pathEntries = $userPath -split ';'
foreach ($gp in $gitPaths) {
    if ($pathEntries -notcontains $gp) {
        $userPath = "$userPath;$gp"
    }
}
[Environment]::SetEnvironmentVariable("Path", $userPath, "User")
```

### 第 4 步：克隆 Hermes Agent 仓库

```powershell
$ErrorActionPreference = 'Stop'
$ProgressPreference = 'SilentlyContinue'

$repoUrl = "https://ghfast.top/https://github.com/NousResearch/hermes-agent.git"
$repoDir = "$env:LOCALAPPDATA\hermes\hermes-agent"
$gitExe = "$env:LOCALAPPDATA\hermes\git\cmd\git.exe"
if (-not (Test-Path $gitExe)) { throw "Git 未安装，请先执行第 3 步" }

if (Test-Path $repoDir) {
    $backupDir = "$repoDir.bak.$(Get-Date -Format 'yyyyMMddHHmmss')"
    Rename-Item -LiteralPath $repoDir -NewName (Split-Path $backupDir -Leaf)
}
& $gitExe clone --depth 1 $repoUrl $repoDir
if ($LASTEXITCODE -ne 0) { throw "克隆仓库失败" }
```

### 第 5 步：创建虚拟环境并安装依赖（使用清华 PyPI 镜像）

```powershell
$ErrorActionPreference = 'Stop'
$ProgressPreference = 'SilentlyContinue'

$repoDir = "$env:LOCALAPPDATA\hermes\hermes-agent"
$uvExe = "$env:LOCALAPPDATA\hermes\bin\uv.exe"
if ($env:Path -split ';' -notcontains "$env:LOCALAPPDATA\hermes\bin") {
    $env:Path = "$env:LOCALAPPDATA\hermes\bin;$env:Path"
}

& $uvExe venv "$env:LOCALAPPDATA\hermes\venv" --python 3.11
if ($LASTEXITCODE -ne 0) { throw "创建虚拟环境失败" }

& $uvExe pip install -e "$($repoDir)[all]" `
    --index-url https://pypi.tuna.tsinghua.edu.cn/simple `
    --python "$env:LOCALAPPDATA\hermes\venv\Scripts\python.exe"
if ($LASTEXITCODE -ne 0) { throw "安装依赖失败" }
```

### 第 6 步：添加到 PATH 并验证

```powershell
$ErrorActionPreference = 'Stop'
$venvScripts = "$env:LOCALAPPDATA\hermes\venv\Scripts"
$userPath = [Environment]::GetEnvironmentVariable("Path", "User")
$pathEntries = $userPath -split ';'
if ($pathEntries -notcontains $venvScripts) {
    [Environment]::SetEnvironmentVariable("Path", "$userPath;$venvScripts", "User")
}
if ($env:Path -split ';' -notcontains $venvScripts) {
    $env:Path = "$venvScripts;$env:Path"
}
hermes --version
if ($LASTEXITCODE -ne 0) { throw "Hermes 安装验证失败" }
```

### 第 7 步：配置 API（可选，以阿里云百炼为例）

如果不确定自己的 API Key 属于哪种计费方式，请先查看阿里云百炼控制台，或依次尝试以下端点。

| 计费方式 | Base URL |
|----------|----------|
| Token Plan 团队版 | `https://token-plan.cn-beijing.maas.aliyuncs.com/apps/anthropic` |
| Coding Plan | `https://coding.dashscope.aliyuncs.com/apps/anthropic` |
| 按量计费 | `https://dashscope.aliyuncs.com/apps/anthropic` |

**安全提示：API Key 会写入 `~/.hermes/.env` 文件，不会出现在命令行历史中。**

```powershell
$ErrorActionPreference = 'Stop'

$baseUrl = Read-Host -Prompt "请输入 Base URL（从上表选择）"
$apiKey = Read-Host -Prompt "请输入 API Key" -AsSecureString
$apiKeyPlain = [System.Runtime.InteropServices.Marshal]::PtrToStringAuto(
    [System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($apiKey)
)

# 将 API Key 写入 ~/.hermes/.env（Hermes 的密钥存储文件）
$envFile = "$env:LOCALAPPDATA\hermes\.env"
$envContent = if (Test-Path $envFile) { Get-Content $envFile -Raw } else { "" }
$escapedKey = $apiKeyPlain.Replace('$', '$$')
if ($envContent -notmatch "MODEL_API_KEY=") {
    Add-Content -LiteralPath $envFile -Value "MODEL_API_KEY=$escapedKey" -Encoding UTF8
} else {
    (Get-Content $envFile) -replace 'MODEL_API_KEY=.*', "MODEL_API_KEY=$escapedKey" | Set-Content $envFile -Encoding UTF8
}
Remove-Variable apiKeyPlain

# 配置非敏感项
hermes config set model.provider custom
hermes config set model.base_url $baseUrl
hermes config set model.api_mode anthropic_messages
hermes config set model.default qwen3.7-max
```

验证配置（使用 `--dry-run` 或轻量请求，如果支持）：
```powershell
hermes chat -q "你好"
```
如果返回 403，说明端点与 Key 类型不匹配，请切换其他端点重试。

### 第 8 步：清理临时文件

安装完成后清理残留的临时文件（无论前面步骤是否成功，都建议执行）：

```powershell
Remove-Item -Path "$env:TEMP\uv*.zip" -Force -ErrorAction SilentlyContinue
Remove-Item -Path "$env:TEMP\PortableGit*.7z.exe" -Force -ErrorAction SilentlyContinue
Remove-Item -Path "$env:TEMP\python-*.exe" -Force -ErrorAction SilentlyContinue
```

## 常见问题排查

| 问题 | 原因 | 解决 |
|------|------|------|
| Python 下载超时 | 未设置 `UV_PYTHON_INSTALL_MIRROR` | 确认环境变量值为完整 GitHub releases 路径 |
| `hermes chat` 超时 | API 端点不匹配 | 逐个测试三种端点，用直接 curl 验证 |
| `git` 命令找不到 | 未安装 Git 或 PATH 未刷新 | 确认 PortableGit 解压成功，刷新 `$env:Path` |
| API 返回 403 `invalid api-key` | 端点与 Key 类型不匹配 | 切换其他端点重试 |
| `uv pip install` 报错 `[all]` 解析失败 | 使用了旧版技能文件 | 确认 `$repoDir` 后面使用的是 `$($repoDir)[all]` 格式 |

## 禁止事项

- **禁止使用 `winget` 安装 Python** — 走 python.org 下载慢，且会弹 UAC 窗口卡死
- **禁止直接运行官方 `install.ps1`** — 不走镜像，全部从 GitHub 下载，必定超时
- **禁止使用默认 PyPI 源** — 必须用 `--index-url https://pypi.tuna.tsinghua.edu.cn/simple`
- **禁止只测一个镜像源** — 必须并行测试多个，哪个快用哪个
- **禁止在命令行中直接传入 API Key** — 使用 `Read-Host -AsSecureString` 安全输入

## 安装位置汇总

所有文件安装在 `%LOCALAPPDATA%\hermes\` 下：

```
%LOCALAPPDATA%\hermes\
├── bin\          # uv.exe
├── git\          # PortableGit（含 bash.exe）
├── venv\         # Python 虚拟环境
├── hermes-agent\ # 源代码仓库
├── config.yaml   # 配置文件
└── .env          # 密钥和 Token
```