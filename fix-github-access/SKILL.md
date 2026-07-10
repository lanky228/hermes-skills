---
name: fix-github-access
description: 仅在用户要求修复 GitHub 访问、GitHub 打不开、GitHub 被屏蔽、GitHub 无法访问、GitHub 不稳定时使用。提供自收敛迭代流程，通过修改 hosts 文件、禁用浏览器 DNS-over-HTTPS、验证连通性来恢复 GitHub 访问。若用户追问"为什么不稳定"，解释 GFW SNI 阻断 + CDN IP 轮换的猫鼠游戏本质。
---

# 修复 GitHub 访问

## 原理说明（为什么不稳定）

GitHub 在中国大陆的访问问题本质是 **GFW 的 SNI 深度包检测（DPI）阻断**：

1. **DNS 层面**：默认 DNS 可能返回被污染的 IP，或直接无法解析
2. **SNI 阻断**：GFW 检查 TLS 握手中 ClientHello 的 SNI 字段（明文包含 `github.com`），命中则发送 RST 包掐断连接。这导致 **TCP 能通（ping 通），HTTPS 不通**
3. **猫鼠游戏**：GitHub 使用 Fastly CDN，IP 地址定期轮换。GitHub520 项目每天扫描新 IP，但 GFW 也在不断封禁新 IP——**一个 IP 通常只能撑几小时到一两天**

**hosts 方案本质是"打地鼠"**：每次 IP 失效就必须重新拉取、重新验证、重新写入。要稳定访问，唯一可靠方案是走 VPN/代理隧道，绕过 GFW 的 DPI 检测。本技能提供的是"快速续命"方案，不保证长期稳定。

## 核心流程（迭代至收敛）

### 阶段 1：诊断

1. 测试 TCP 连通性（绕过 hosts，直接测试 IP 的可达性）：
   ```powershell
   Test-NetConnection -ComputerName github.com -Port 443 -WarningAction SilentlyContinue -InformationLevel Quiet
   ```
2. 测试 HTTP 访问：
   ```powershell
   curl.exe -s -o NUL -w "%{http_code}" --connect-timeout 10 https://github.com
   ```
3. 若 HTTP 返回 200：问题不在网络层，检查浏览器设置（DoH、代理、扩展）。
4. 若 TCP 通但 HTTP 超时：典型的 SNI 阻断 → 当前 hosts IP 已失效，进入阶段 2。
5. 若 TCP 不通：DNS 或 IP 被封，进入阶段 2。

### 阶段 2：获取最新可用 IP

**首选源** — GitHub520 项目，每日更新已验证 IP：

```powershell
$hosts = Invoke-WebRequest -Uri "https://raw.hellogithub.com/hosts" -UseBasicParsing -TimeoutSec 15 | Select-Object -ExpandProperty Content
```

**备选源 1** — 若首选源不可达，尝试 GitLab 镜像：

```powershell
$hosts = Invoke-WebRequest -Uri "https://gitlab.com/ineo6/hosts/-/raw/master/next-hosts" -UseBasicParsing -TimeoutSec 15 | Select-Object -ExpandProperty Content
```

**备选源 2** — 用已知可用 IP 直连获取（绕过 DNS）：

```powershell
curl.exe -sL --connect-timeout 15 --resolve "raw.githubusercontent.com:443:185.199.111.133" "https://raw.githubusercontent.com/521xueweihan/GitHub520/main/hosts"
```

**兜底方案** — 以上全部不可达时，使用阶段 3 中的内置 IP 列表逐个测试。

### 阶段 3：验证 IP 连通性（关键步骤）

**必须逐个测试，只使用返回 `True` 的 IP。** 新旧 IP 都要测，因为新 IP 可能不通但旧 IP 仍可用。

```powershell
$ips = @(
    # github.com 候选
    @{IP="20.205.243.166"; Domain="github.com"},
    @{IP="140.82.112.3"; Domain="github.com-old"},
    # api.github.com 候选
    @{IP="20.205.243.168"; Domain="api.github.com"},
    @{IP="140.82.112.5"; Domain="api.github.com-old"},
    # gist.github.com 候选
    @{IP="159.24.3.173"; Domain="gist.github.com"},
    @{IP="140.82.113.4"; Domain="gist.github.com-old"},
    # codeload.github.com 候选
    @{IP="20.205.243.165"; Domain="codeload.github.com"},
    @{IP="140.82.112.9"; Domain="codeload.github.com-old"},
    # github.global.ssl.fastly.net 候选
    @{IP="31.13.83.34"; Domain="github.global.ssl.fastly.net"},
    @{IP="151.101.1.194"; Domain="github.global.ssl.fastly.net-old"},
    # 其他核心域名
    @{IP="185.199.111.153"; Domain="github.io"},
    @{IP="185.199.109.215"; Domain="github.githubassets.com"},
    @{IP="185.199.109.133"; Domain="raw.githubusercontent.com"},
    @{IP="185.199.109.133"; Domain="avatars"},
    @{IP="185.199.109.133"; Domain="media"},
    @{IP="185.199.109.133"; Domain="objects"},
    @{IP="140.82.114.22"; Domain="central.github.com"},
    @{IP="140.82.112.25"; Domain="alive.github.com"},
    @{IP="140.82.113.17"; Domain="github.community"},
    @{IP="140.82.113.21"; Domain="education.github.com"},
    @{IP="20.85.130.105"; Domain="copilot-proxy"}
)
foreach ($entry in $ips) {
    $result = Test-NetConnection -ComputerName $entry.IP -Port 443 -WarningAction SilentlyContinue -InformationLevel Quiet
    Write-Output "$($entry.IP) -> $($entry.Domain) : $result"
}
```

**关键规则**：
- 每个域名优先用新 IP，新 IP 不通则用旧 IP，都不通则尝试 GitHub520 列表中的其他备选 IP
- 只把返回 `True` 的 IP 写入 hosts
- 若所有核心 IP 均不通，说明网络层面被阻断——建议用户使用 VPN 或代理，停止迭代

### 阶段 4：更新 hosts 文件（需管理员权限）

**重要**：自动化提权（`Start-Process -Verb RunAs`）会弹出 UAC 对话框。如果用户不在电脑前或未点击，命令会超时。此时应**生成自提升脚本到桌面**，让用户手动运行。

#### 方案 A：自动化（优先尝试）

从已验证的 IP 构建 hosts 块，写入临时脚本，提权执行：

```powershell
$script = @'
$hostsPath = "$env:SystemRoot\System32\drivers\etc\hosts"
$githubBlock = @"

# GitHub Hosts Start - Updated $(Get-Date -Format 'yyyy-MM-dd')
<已验证的IP>     github.com
<已验证的IP>     api.github.com
...（所有已验证的 IP-域名 对应项，用阶段 3 的结果填充）
# GitHub Hosts End
"@

$currentContent = Get-Content -Path $hostsPath -Raw -Encoding Ascii
if ($currentContent -match "# GitHub Hosts Start") {
    $newContent = $currentContent -replace "(?s)# GitHub Hosts Start.*# GitHub Hosts End", $githubBlock.Trim()
} else {
    $newContent = $currentContent.TrimEnd() + "`r`n" + $githubBlock
}
Set-Content -Path $hostsPath -Value $newContent -Encoding Ascii -Force
ipconfig /flushdns
'@
$scriptPath = Join-Path $env:TEMP "update_github_hosts.ps1"
Set-Content -Path $scriptPath -Value $script -Encoding UTF8
Start-Process powershell -ArgumentList "-NoProfile", "-ExecutionPolicy", "Bypass", "-File", "`"$scriptPath`"" -Verb RunAs
```

**注意**：不要使用 `-Wait` 参数，UAC 弹窗会导致整个进程阻塞超时。执行后检查 hosts 文件是否实际更新。若未更新（UAC 未点击），进入方案 B。

#### 方案 B：生成自提升脚本（UAC 超时时的 fallback）

将完整的自提升 PowerShell 脚本写到用户桌面 `修复GitHub访问.ps1`，内容如下：

```powershell
# Self-elevating GitHub hosts fix script
if (-NOT ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator")) {
    $arguments = "-NoProfile -ExecutionPolicy Bypass -File `"" + $PSCommandPath + "`""
    Start-Process powershell -Verb RunAs -ArgumentList $arguments
    exit
}

$hostsPath = "$env:SystemRoot\System32\drivers\etc\hosts"
$backupPath = "$env:TEMP\hosts_backup"
Copy-Item $hostsPath $backupPath -Force

$githubBlock = @"
# GitHub Hosts Start - Updated $(Get-Date -Format 'yyyy-MM-dd')
<已验证IP>   github.com
<已验证IP>   api.github.com
...（用阶段 3 验证通过的 IP 填充）
# GitHub Hosts End
"@

$currentContent = Get-Content -Path $hostsPath -Raw -Encoding Ascii
if ($currentContent -match "# GitHub Hosts Start") {
    $newContent = $currentContent -replace "(?s)# GitHub Hosts Start.*# GitHub Hosts End", $githubBlock.Trim()
} else {
    $newContent = $currentContent.TrimEnd() + "`r`n" + $githubBlock
}
Set-Content -Path $hostsPath -Value $newContent -Encoding Ascii -Force
ipconfig /flushdns

Write-Host "GitHub hosts updated!" -ForegroundColor Green
Start-Sleep -Seconds 3
```

告知用户：右键点击桌面上的 `修复GitHub访问.ps1` → **"使用 PowerShell 运行"** → 在 UAC 弹窗中点击"是"。

### 阶段 5：禁用浏览器 DNS-over-HTTPS

浏览器（Chrome/Edge）使用自己的 DNS 解析器（DoH），会绕过 hosts 文件。必须在完成阶段 4 后执行：

```powershell
$script = @'
$chromePath = "HKLM:\Software\Policies\Google\Chrome"
$edgePath = "HKLM:\Software\Policies\Microsoft\Edge"
if (-not (Test-Path $chromePath)) { New-Item -Path $chromePath -Force }
Set-ItemProperty -Path $chromePath -Name "DnsOverHttpsMode" -Value "off" -Force
if (-not (Test-Path $edgePath)) { New-Item -Path $edgePath -Force }
Set-ItemProperty -Path $edgePath -Name "DnsOverHttpsMode" -Value "off" -Force
'@
$scriptPath = Join-Path $env:TEMP "disable_doh.ps1"
Set-Content -Path $scriptPath -Value $script -Encoding UTF8
Start-Process powershell -ArgumentList "-NoProfile", "-ExecutionPolicy", "Bypass", "-File", "`"$scriptPath`"" -Verb RunAs
```

### 阶段 6：验证修复结果

1. **刷新 DNS**：
   ```powershell
   ipconfig /flushdns
   ```

2. **验证 DNS 解析使用 hosts 文件**（不能用 `nslookup`，它会绕过 hosts）：
   ```powershell
   ping github.com -n 1
   # 应显示 hosts 中的 IP，而非 20.205.243.166
   ```

3. **验证 HTTP 访问**：
   ```powershell
   curl.exe -s -o NUL -w "%{http_code}" --connect-timeout 10 https://github.com
   # 必须返回 200
   ```

4. **验证 git 访问**（若已安装 git）：
   ```powershell
   git ls-remote https://github.com/521xueweihan/GitHub520.git HEAD
   ```

5. **浏览器测试**：先杀掉所有浏览器进程，再用禁用 DoH 和 QUIC 的参数启动：
   ```powershell
   Get-Process -Name "chrome", "msedge", "firefox" -ErrorAction SilentlyContinue | Stop-Process -Force -ErrorAction SilentlyContinue
   Start-Process "msedge" -ArgumentList "--disable-features=UseDnsOverHttps", "--disable-quic", "https://github.com"
   ```

### 阶段 7：重试 / 自校正

| 失败情况 | 处理方式 |
|---------|---------|
| 无法获取 GitHub520 hosts | 尝试备选源 1、2，最后用内置 IP 列表 |
| 新 IP 不通但旧 IP 通 | 保留旧 IP 继续使用；新 IP 可能尚未被 GFW 放行或已被封 |
| hosts 文件未更新 | 检查 UAC 是否被点击；若超时则改用方案 B 生成自提升脚本 |
| DNS 仍解析到旧 IP | 执行 `ipconfig /flushdns`；检查 hosts 文件编码（必须 ASCII） |
| curl 返回非 200 | 重新逐个测试 IP（阶段 3）；TCP 通但 HTTP 不通 = SNI 阻断，该 IP 不能用 |
| 浏览器仍打不开 | 完全杀掉浏览器进程；访问 `chrome://net-internals/#dns` 清除内部缓存；验证 DoH 注册表项已设置 |
| 所有 IP 均不可达 | 网络层面全阻断——建议用户使用 VPN 或代理；停止迭代 |

## 必须映射的关键域名

```
github.com, api.github.com, gist.github.com, codeload.github.com,
github.io, githubstatus.com, github.community,
github.global.ssl.fastly.net, github.githubassets.com,
raw.githubusercontent.com, raw.github.com,
objects.githubusercontent.com, user-images.githubusercontent.com,
private-user-images.githubusercontent.com,
avatars.githubusercontent.com, avatars0-5.githubusercontent.com,
favicons.githubusercontent.com, desktop.githubusercontent.com,
camo.githubusercontent.com, cloud.githubusercontent.com,
github.map.fastly.net, media.githubusercontent.com,
central.github.com, collector.github.com, education.github.com,
alive.github.com, live.github.com,
copilot-proxy.githubusercontent.com
```

## 重要注意事项

- **这是一场猫鼠游戏**：hosts 方案不是一劳永逸的，IP 随时可能失效。用户问"为什么不稳定"时，解释 GFW SNI 阻断 + CDN IP 轮换的机制
- 要长期稳定，唯一可靠方案是 VPN/代理隧道
- hosts 文件必须使用 ASCII 编码（不能用 UTF-8 with BOM）
- 每次修改 hosts 文件后必须执行 `ipconfig /flushdns`
- 修改 hosts 后浏览器必须完全重启（不能仅刷新页面）
- `nslookup` 会绕过 hosts 文件——不要用它验证；用 `ping` 或 `Resolve-DnsName`
- 若用户开了 VPN，建议先断开——部分 VPN 有独立 DNS，会绕过 hosts
- 自动化提权时**不要使用 `-Wait` 参数**，UAC 弹窗会导致进程阻塞超时