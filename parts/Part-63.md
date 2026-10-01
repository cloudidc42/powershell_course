# Part 63: Network Security Automation

> **ระดับ**: 🔴 Advanced | **เวลา**: ~5 ชั่วโมง
>
> ⚠️ **คำเตือน**: เนื้อหานี้ใช้สำหรับการทดสอบที่ได้รับอนุญาต, CTF, Blue Team และการเรียนรู้เท่านั้น — ห้ามใช้โดยไม่ได้รับอนุญาต

---

## 1. Firewall Rule Management

```powershell
# Windows Firewall automation
function Get-FirewallRules {
    param([string]$Direction = 'Inbound', [switch]$Enabled)
    $rules = Get-NetFirewallRule -Direction $Direction
    if ($Enabled) { $rules = $rules | Where-Object { $_.Enabled -eq 'True' } }
    $rules | Select-Object Name, DisplayName, Action, Profile, Enabled |
        Sort-Object Action, Name
}

function Add-FirewallRule {
    param(
        [string]$Name,
        [string]$Protocol    = 'TCP',
        [int[]]$LocalPort,
        [string]$RemoteAddress = 'Any',
        [string]$Action        = 'Allow',
        [string]$Profile       = 'Domain,Private'
    )
    $params = @{
        DisplayName   = $Name
        Name          = $Name
        Protocol      = $Protocol
        Direction     = 'Inbound'
        Action        = $Action
        Profile       = $Profile
        Enabled       = 'True'
    }
    if ($LocalPort)      { $params['LocalPort']     = $LocalPort }
    if ($RemoteAddress -ne 'Any') { $params['RemoteAddress'] = $RemoteAddress }
    
    New-NetFirewallRule @params
    Write-Host "Added firewall rule: $Name" -ForegroundColor Green
}

# Allow only known IPs to RDP
Add-FirewallRule -Name 'RDP-Restricted' -LocalPort 3389 -RemoteAddress '10.0.0.0/8'

# Block outbound to known malicious IPs (blocklist)
function Add-IPBlocklist {
    param([string[]]$IPs, [string]$RuleName = 'BlockMalicious')
    New-NetFirewallRule `
        -DisplayName $RuleName `
        -Name        $RuleName `
        -Direction   Outbound `
        -Action      Block `
        -RemoteAddress $IPs `
        -Enabled     True
    Write-Host "Blocked $($IPs.Count) IPs" -ForegroundColor Yellow
}

# Download and apply threat intel blocklist
$blocklist = Invoke-RestMethod 'https://raw.githubusercontent.com/firehol/blocklist-ipsets/master/firehol_level1.netset'
$ips       = $blocklist -split "`n" | Where-Object { $_ -match '^[0-9]' -and $_ -notmatch '#' } | Select-Object -First 100
Add-IPBlocklist -IPs $ips
```

---

## 2. Network Traffic Analysis

```powershell
# Monitor active connections
function Watch-NetworkConnections {
    param([int]$Interval = 5, [int]$Count = 12)
    
    $baseline = @{}
    
    for ($i = 0; $i -lt $Count; $i++) {
        $current = Get-NetTCPConnection -State Established | Group-Object RemoteAddress | 
            Sort-Object Count -Descending
        
        foreach ($grp in $current) {
            if (-not $baseline[$grp.Name]) {
                Write-Host "[+] New connection source: $($grp.Name) ($($grp.Count) connections)" -ForegroundColor Yellow
            }
            $baseline[$grp.Name] = $grp.Count
        }
        
        Start-Sleep $Interval
    }
}

# Suspicious connection detector
function Find-SuspiciousConnections {
    $suspicious = @()
    
    # Connections on unusual ports
    $unusual = Get-NetTCPConnection -State Established |
        Where-Object { $_.RemotePort -notin @(80,443,22,3389,5985,5986) }
    
    foreach ($conn in $unusual) {
        $proc = Get-Process -Id $conn.OwningProcess -ErrorAction SilentlyContinue
        $suspicious += [PSCustomObject]@{
            Process      = $proc.Name
            PID          = $conn.OwningProcess
            LocalPort    = $conn.LocalPort
            RemoteAddress= $conn.RemoteAddress
            RemotePort   = $conn.RemotePort
            Type         = 'UnusualPort'
        }
    }
    
    # Processes making many connections (potential C2 beacon)
    $beacons = Get-NetTCPConnection -State Established |
        Group-Object OwningProcess | Where-Object { $_.Count -gt 20 }
    
    foreach ($b in $beacons) {
        $proc = Get-Process -Id $b.Name -ErrorAction SilentlyContinue
        $suspicious += [PSCustomObject]@{
            Process      = $proc.Name
            PID          = $b.Name
            Connections  = $b.Count
            Type         = 'ExcessiveConnections'
        }
    }
    
    return $suspicious
}

$threats = Find-SuspiciousConnections
if ($threats) {
    Write-Warning "Found $($threats.Count) suspicious connections!"
    $threats | Format-Table
}
```

---

## 3. SSL/TLS Certificate Auditing

```powershell
function Test-TLSCertificate {
    param([string]$Hostname, [int]$Port = 443)
    
    try {
        $tcp    = [System.Net.Sockets.TcpClient]::new($Hostname, $Port)
        $ssl    = [System.Net.Security.SslStream]::new($tcp.GetStream(), $false,
            { $true }  # accept all certs for inspection
        )
        $ssl.AuthenticateAsClient($Hostname)
        $cert   = [System.Security.Cryptography.X509Certificates.X509Certificate2]$ssl.RemoteCertificate
        
        $daysLeft = ([datetime]$cert.NotAfter - [datetime]::Now).Days
        
        [PSCustomObject]@{
            Hostname  = $Hostname
            Subject   = $cert.Subject
            Issuer    = $cert.Issuer
            NotBefore = $cert.NotBefore
            NotAfter  = $cert.NotAfter
            DaysLeft  = $daysLeft
            Expired   = $daysLeft -le 0
            Expiring  = $daysLeft -le 30
            Protocol  = $ssl.SslProtocol
            KeySize   = $cert.PublicKey.Key.KeySize
        }
    } catch {
        [PSCustomObject]@{ Hostname=$Hostname; Error=$_.Exception.Message }
    } finally {
        if ($ssl)  { $ssl.Dispose() }
        if ($tcp)  { $tcp.Dispose() }
    }
}

# Audit multiple hosts
$hosts = @('www.example.com','api.example.com','portal.example.com')
$results = $hosts | ForEach-Object -Parallel { Test-TLSCertificate $_ } -ThrottleLimit 5

$results | Format-Table Hostname, DaysLeft, Protocol, KeySize, Expiring
$expiring = $results | Where-Object { $_.DaysLeft -le 30 -and -not $_.Error }
if ($expiring) {
    Write-Warning "$($expiring.Count) certificates expiring within 30 days!"
    $expiring | Select-Object Hostname, NotAfter, DaysLeft | Format-Table
}
```

---

**ก่อนหน้า ← [Part 62](Part-62.md) | ต่อไป → [Part 64: Web App Security](Part-64.md)**
