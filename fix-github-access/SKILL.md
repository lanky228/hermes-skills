---
name: fix-github-access
description: 仅在用户要求修复 GitHub 访问、GitHub 打不开、GitHub 被屏蔽、GitHub 无法访问时使用。提供自收敛迭代流程，通过修改 hosts 文件、禁用浏览器 DNS-over-HTTPS、验证连通性来恢复 GitHub 访问。
---

# 修复 GitHub 访问

本技能编码了在 DNS 或 SNI 层面被阻断时恢复 GitHub 访问的可行方法。流程设计为自校正：每一步都验证，失败则触发重试或替代方案，直到收敛。

## 核心流程（迭代至收敛）

### 阶段 1：诊断

1. 测试 `github.com:443` 的 TCP 连通性：
   ```powershell
   Test-NetConnection -ComputerName github.com -Port 443 -WarningAction SilentlyContinue -InformationLevel Quiet
   ```
2. 若 TCP 通，测试 HTTP 访问：
   ```powershell
   curl.exe -s -o NUL -w "%{http_code}" --connect-timeout 10 https://github.com
   ```
3. 若 HTTP 返回 200，问题不在网络层——检查浏览器设置（DoH、代理、扩展）。
4. 若 TCP 不通或 HTTP 超时，进入阶段 2。

### 阶段 2：获取最新可用 IP

**首选源** — GitHub520 项目，每日更新已验证 IP：

```powershell
$hosts = Invoke-WebRequest -Uri "https://raw.hellogithub.com/hosts" -UseBasicParsing -TimeoutSec 15 | Select-Object -ExpandProperty Content
```

**备选源 1** — 若首选源不可达，尝试 GitLab 镜像：

```powershell
$hosts = Invoke-WebRequest -Uri "https://gitlab.com/ineo6/hosts/-/raw/master/next-hosts" -UseBasicParsing -TimeoutSec 15 | Select-Object -ExpandProperty Content
```

**备选源 2** — 用已知可用 IP 直连获取：

```powershell
curl.exe -sL --connect-timeout 15 --resolve "raw.githubusercontent.com:443:185.199.111.133" "https://raw.githubusercontent.com/521xueweihan/GitHub520/main/hosts"
```

**兜底方案** — 使用内置的已知稳定 IP 列表（阶段 3 中的 IP 作为最后手段）。

### 阶段 3：验证 IP 连通性

解析 hosts 内容，逐条测试每个 IP:端口。对每行 `IP 域名` 执行：

```powershell
$ips = @(
    @{IP="140.82.112.3"; Domain="github.com"},
    @{IP="140.82.112.5"; Domain="api.github.com"},
    @{IP="140.82.113.4"; Domain="gist.github.com"},
    @{IP="140.82.112.9"; Domain="codeload.github.com"},
    @{IP="185.199.111.153"; Domain="github.io"},
    @{IP="151.101.1.194"; Domain="github.global.ssl.fastly.net"},
    @{IP="185.199.111.215"; Domain="github.githubassets.com"},
    @{IP="185.199.111.133"; Domain="raw.githubusercontent.com"},
    @{IP="185.199.110.133"; Domain="user-images.githubusercontent.com"},
    @{IP="185.199.109.133"; Domain="avatars.githubusercontent.com"},
    @{IP="185.199.108.133"; Domain="media.githubusercontent.com"},
    @{IP="140.82.114.22"; Domain="central.github.com"},
    @{IP="140.82.114.25"; Domain="alive.github.com"},
    @{IP="20.85.130.105"; Domain="copilot-proxy.githubusercontent.com"}
)
foreach ($entry in $ips) {
    $result = Test-NetConnection -ComputerName $entry.IP -Port 443 -WarningAction SilentlyContinue -InformationLevel Quiet
    "$($entry.IP) -> $($entry.Domain) : $result"
}
```

**关键规则**：只使用返回 `True` 的 IP。若某 IP 不通，从 hosts 列表中测试其他备选 IP。若所有 IP 均不通，说明网络层面被阻断——建议使用 VPN/代理，停止迭代。

### 阶段 4：更新 hosts 文件（需管理员权限）

从已验证通过的 IP 构建 hosts 块。将脚本写入临时文件并以管理员权限执行：

```powershell
$script = @'
$hostsPath = "$env:SystemRoot\System32\drivers\etc\hosts"
$githubBlock = @"

# GitHub Hosts Start - Updated $(Get-Date -Format 'yyyy-MM-dd')
<已验证的IP>     github.com
<已验证的IP>     api.github.com
...（所有已验证的 IP-域名 对应项）
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
Start-Process powershell -ArgumentList "-NoProfile", "-ExecutionPolicy", "Bypass", "-File", "`"$scriptPath`"" -Verb RunAs -Wait
```

**权限被拒时**：使用 `Start-Process -Verb RunAs` 以管理员身份运行脚本。这是标准做法，必须能工作。若仍失败，需要用户手动以管理员身份运行。

### 阶段 5：禁用浏览器 DNS-over-HTTPS（需管理员权限）

浏览器（Chrome/Edge）使用自己的 DNS 解析器（DoH），会绕过 hosts 文件。通过注册表关闭：

```powershell
$regScript = @'
$chromePath = "HKLM:\Software\Policies\Google\Chrome"
$edgePath = "HKLM:\Software\Policies\Microsoft\Edge"

if (-not (Test-Path $chromePath)) { New-Item -Path $chromePath -Force }
Set-ItemProperty -Path $chromePath -Name "DnsOverHttpsMode" -Value "off" -Force

if (-not (Test-Path $edgePath)) { New-Item -Path $edgePath -Force }
Set-ItemProperty -Path $edgePath -Name "DnsOverHttpsMode" -Value "off" -Force
'@
$scriptPath = Join-Path $env:TEMP "disable_doh.ps1"
Set-Content -Path $scriptPath -Value $regScript -Encoding UTF8
Start-Process powershell -ArgumentList "-NoProfile", "-ExecutionPolicy", "Bypass", "-File", "`"$scriptPath`"" -Verb RunAs -Wait
```

### 阶段 6：验证修复结果

1. **刷新 DNS 缓存**：
   ```powershell
   ipconfig /flushdns
   ```

2. **验证 DNS 解析使用 hosts 文件**：
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

若任一验证步骤失败：

| 失败情况 | 处理方式 |
|---------|---------|
| hosts 文件未更新 | 重新执行阶段 4（管理员权限）；确认文件包含 GitHub 块 |
| DNS 仍解析到旧 IP | 执行 `ipconfig /flushdns`；检查 hosts 文件编码（必须 ASCII） |
| curl 返回非 200 | 重新逐个测试 IP（阶段 3）；尝试 hosts 列表中的备选 IP |
| 浏览器仍打不开 | 先杀掉所有浏览器进程；访问 `chrome://net-internals/#dns` 清除缓存；验证 DoH 注册表项已设置 |
| 所有 IP 均不可达 | 网络层面阻断——建议使用 VPN 或代理；停止迭代 |

## 必须映射的关键域名

以下域名是 GitHub 完整功能所需，确保全部在 hosts 块中：

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

- hosts 文件必须使用 ASCII 编码（不能用 UTF-8 with BOM）
- 每次修改 hosts 文件后必须执行 `ipconfig /flushdns`
- 修改 hosts 后浏览器必须完全重启（不能仅刷新页面）
- GitHub520 的 IP 每日更新；若后续访问再次失败，重新执行此流程
- `nslookup` 会绕过 hosts 文件——不要用它验证；用 `ping` 或 `Resolve-DnsName`
- 若用户开了 VPN，建议先断开——部分 VPN 有独立 DNS，会绕过 hosts