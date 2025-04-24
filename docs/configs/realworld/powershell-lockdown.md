# PowerShell Lockdown Script

Dette script overvåger DDoS-aktivitet mod en FiveM-server og aktiverer automatisk lockdown ved mistænkelig aktivitet. Det bruger Windows Firewall og Discord-webhooks til notifikation.

```powershell
<#
.SYNOPSIS
  High-performance DDoS watcher for FiveM port, with Discord alerts.

.DESCRIPTION
  - Tracks unique new sources (SYN + new established TCP/UDP) over a sliding window.
  - On >= $AlertThresh → enter lockdown: disable static allow rules, block TCP+UDP, whitelist clients, notify Discord.
  - If in lockdown and count stays < $AlertThresh for $ReleaseWindow s → exit lockdown: restore static rules, remove lockdown rules, notify Discord.
  - Logs enter/exit events to C:\Scripts\DDoS-Watcher.log.
  - Console uses a single updating status line for minimal overhead.
#>

[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

$Port           = 30121
$WindowSec      = 15
$AlertThresh    = 50
$ReleaseWindow  = 120
$CheckInterval  = 1
$LogFile        = "C:\Scripts\DDoS-Watcher.log"
$DiscordWebhook = "" # Insert Discord webhook URL here

$seen         = @{}    # IP -> last-seen datetime
$allClients   = @{}    # IP -> $true
$lockdown     = $false
$releaseStart = $null  # when count first fell below threshold

function Log($msg) {
    "$((Get-Date).ToString('yyyy-MM-dd HH:mm:ss'))  $msg" |
      Out-File $LogFile -Append
}

function Notify-Discord($text) {
    $payload = @{ content = $text } | ConvertTo-Json -Compress
    try {
        Invoke-RestMethod -Uri $DiscordWebhook -Method Post `
          -ContentType 'application/json' -Body $payload -ErrorAction Stop
    } catch {
        Log "Discord webhook failed: $_"
    }
}

$static = Get-NetFirewallRule -Direction Inbound -Action Allow |
  Where-Object {
    (Get-NetFirewallPortFilter -AssociatedNetFirewallRule $_).LocalPort `
      -contains $Port
  }

Get-NetFirewallRule -Name "DDoS-Lockdown-TCP","DDoS-Lockdown-UDP","AllowClient_*" `
  -ErrorAction SilentlyContinue |
    Remove-NetFirewallRule -Confirm:$false -ErrorAction SilentlyContinue

function Get-CurrentClients {
    $list = @()
    $t = Get-NetTCPConnection -State Established -LocalPort $Port -ErrorAction SilentlyContinue
    if ($t) { $list += $t.RemoteAddress }
    $u = Get-NetUDPEndpoint -LocalPort $Port -ErrorAction SilentlyContinue
    if ($u) {
        $list += ($u | Where-Object { $_.PSObject.Properties.Match('RemoteAddress') } |
                   Select-Object -ExpandProperty RemoteAddress)
    }
    return $list | Sort-Object -Unique
}

function Get-NewSources {
    $new = @()
    $syn = Get-NetTCPConnection -State SynReceived -LocalPort $Port -ErrorAction SilentlyContinue
    if ($syn) { $new += $syn.RemoteAddress }

    foreach ($ip in Get-CurrentClients) {
        if (-not $allClients.ContainsKey($ip)) {
            $new += $ip
        }
        $allClients[$ip] = $true
    }
    return $new | Sort-Object -Unique
}

function Update-Window {
    param($ips)
    $now    = Get-Date
    $cutoff = $now.AddSeconds(-$WindowSec)

    foreach ($ip in $ips) { $seen[$ip] = $now }
    foreach ($old in $seen.Keys | Where-Object { $seen[$_] -lt $cutoff }) {
        $seen.Remove($old) | Out-Null
    }
}

function Enter-Lockdown {
    Log ">>> ENTER LOCKDOWN (>= $AlertThresh new sources)"
    Notify-Discord "# :warning: **DDoS observeret:** LOCKDOWN enabled :lock: "

    $script:lockdown     = $true
    $script:releaseStart = $null

    $static | ForEach-Object {
        Set-NetFirewallRule -Name $_.Name -Enabled False -ErrorAction SilentlyContinue
    }

    New-NetFirewallRule -Name "DDoS-Lockdown-TCP" `
      -DisplayName "DDoS Lockdown TCP" -Direction Inbound -Action Block `
      -Protocol TCP -LocalPort $Port -RemoteAddress Any -Profile Any `
      -ErrorAction SilentlyContinue

    New-NetFirewallRule -Name "DDoS-Lockdown-UDP" `
      -DisplayName "DDoS Lockdown UDP" -Direction Inbound -Action Block `
      -Protocol UDP -LocalPort $Port -RemoteAddress Any -Profile Any `
      -ErrorAction SilentlyContinue

    Get-CurrentClients | ForEach-Object {
        $safe = $_.Replace('.', '_')
        New-NetFirewallRule -Name "AllowClient_${safe}_TCP" `
          -DisplayName "Allow $_ TCP" -Direction Inbound -Action Allow `
          -Protocol TCP -LocalPort $Port -RemoteAddress $_ -Profile Any `
          -ErrorAction SilentlyContinue
        New-NetFirewallRule -Name "AllowClient_${safe}_UDP" `
          -DisplayName "Allow $_ UDP" -Direction Inbound -Action Allow `
          -Protocol UDP -LocalPort $Port -RemoteAddress $_ -Profile Any `
          -ErrorAction SilentlyContinue
    }
}

function Exit-Lockdown {
    Log "<<< EXIT LOCKDOWN (< $AlertThresh for $ReleaseWindow s)"
    Notify-Discord "# :white_check_mark: **Anti-DDoS:** LOCKDOWN ophævet :unlock: "

    $script:lockdown     = $false
    $script:releaseStart = $null

    Get-NetFirewallRule |
      Where-Object { $_.Name -like "DDoS-Lockdown-*" -or $_.Name -like "AllowClient_*" } |
      Remove-NetFirewallRule -Confirm:$false -ErrorAction SilentlyContinue

    $static | ForEach-Object {
        Set-NetFirewallRule -Name $_.Name -Enabled True -ErrorAction SilentlyContinue
    }
}

Log "=== DDoS-Watcher started on port $Port ==="

while ($true) {
    $newIPs = Get-NewSources
    Update-Window -ips $newIPs
    $count  = $seen.Count

    $ts    = (Get-Date).ToString('HH:mm:ss')
    $state = if ($lockdown) { 'LOCKDOWN' } else { 'NORMAL  ' }
    Write-Progress -Id 1 `
      -Activity "[$ts] Count=$count Thresh=$AlertThresh State=$state" `
      -PercentComplete ([math]::Min(100, ($count / $AlertThresh * 100)))

    if (-not $lockdown -and $count -ge $AlertThresh) {
        Enter-Lockdown
    }
    elseif ($lockdown) {
        if ($count -lt $AlertThresh) {
            if (-not $releaseStart) {
                $releaseStart = Get-Date
            }
            elseif ((Get-Date).Subtract($releaseStart).TotalSeconds -ge $ReleaseWindow) {
                Exit-Lockdown
            }
        }
        else {
            $releaseStart = $null
        }
    }

    Start-Sleep -Seconds $CheckInterval
}

```