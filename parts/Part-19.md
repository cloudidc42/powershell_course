# Part 19: PowerShell Remoting

> **ระดับ**: 🟡 Intermediate | **เวลา**: ~4 ชั่วโมง

---

## 1. เปิดใช้ PSRemoting

```powershell
# ผู้ดูแล: เปิด WinRM
Enable-PSRemoting -Force

# ตรวจสอบสถานะ
Get-WSManInstance WSMan/Config/Listener  # ดู listeners
Get-Service WinRM
Test-WSMan -ComputerName 'Server01'

# ใน workgroup / cross-domain
Set-Item WSMan:\localhost\Client\TrustedHosts -Value 'Server01,Server02'
Set-Item WSMan:\localhost\Client\TrustedHosts -Value '*'  # all (ผลักพัน - lab only)

# Linux (PS Core) - ใช้ SSH remoting
# Install-Module -Name Microsoft.PowerShell.RemotingTools
# Enable-SSHRemoting
```

---

## 2. Invoke-Command

```powershell
# เรียก command บน remote server
Invoke-Command -ComputerName 'Server01' -ScriptBlock {
    Get-Process | Where-Object CPU -gt 10
}

# ส่งตัวแปรด้วย $using:
$threshold = 10
Invoke-Command -ComputerName 'Server01' -ScriptBlock {
    Get-Process | Where-Object { $_.CPU -gt $using:threshold }
}

# หลาย server พร้อมกัน
$servers = @('Server01', 'Server02', 'Server03')
Invoke-Command -ComputerName $servers -ScriptBlock {
    [PSCustomObject]@{
        Server  = $env:COMPUTERNAME
        CPU     = (Get-CimInstance Win32_Processor).LoadPercentage
        FreeMem = [math]::Round((Get-CimInstance Win32_OS).FreePhysicalMemory/1MB,1)
    }
} | Sort-Object Server | Format-Table -AutoSize

# ส่ง credential
$cred = Get-Credential
Invoke-Command -ComputerName 'Server01' -Credential $cred -ScriptBlock {
    whoami
}

# เรียก script file
Invoke-Command -ComputerName $servers -FilePath '.\Deploy.ps1'
```

---

## 3. PSSession

```powershell
# สร้าง persistent session
$session = New-PSSession -ComputerName 'Server01'

# ส่ง command หลายครั้ง
Invoke-Command -Session $session -ScriptBlock {
    $global:counter = 0  # คงอยู่ใน session
}

Invoke-Command -Session $session -ScriptBlock {
    $global:counter++
    "Counter: $global:counter"
}

# เข้าไป interactive
Enter-PSSession -Session $session
# [Server01]: PS C:\> ...
# [Server01]: PS C:\> exit
Exit-PSSession

# หรือเข้าโดยตรง
Enter-PSSession -ComputerName 'Server01'

# Multiple sessions
$sessions = New-PSSession -ComputerName $servers
Invoke-Command -Session $sessions -ScriptBlock { hostname }
$sessions | Remove-PSSession

# Session configuration
Get-PSSessionConfiguration    # ดู registered configurations

# Custom session config (จำกัดสิทธิ์)
# Register-PSSessionConfiguration -Name 'ReadOnly' -RunAsCredential $cred
```

---

## 4. SSH Remoting (PS7, Cross-platform)

```powershell
# ตั้งค่า SSH (ต้อง configure sshd_config ก่อน)
# เพิ่มใน sshd_config:
# Subsystem powershell /usr/bin/pwsh -sshs -NoLogo

# เชื่อมต่อผ่าน SSH
$session = New-PSSession -HostName 'linux-server' -UserName 'admin' -SSHTransport

Invoke-Command -Session $session -ScriptBlock {
    uname -a
    Get-Process | Sort-Object CPU -Descending | Select-Object -First 5
}

# SSH + key auth
$session = New-PSSession -HostName 'linux-server' \
    -UserName 'admin' \
    -KeyFilePath '~/.ssh/id_rsa' \
    -SSHTransport

$session | Remove-PSSession
```

---

## 5. Copy-Item Remoting

```powershell
$session = New-PSSession -ComputerName 'Server01'

# ส่งไฟล์ไป
Copy-Item '.\config.json' -Destination 'C:\App\' -ToSession $session
Copy-Item '.\Deploy\' -Destination 'C:\Deploy\' -ToSession $session -Recurse

# ดึงไฟล์มา
Copy-Item 'C:\App\logs\app.log' -Destination '.' -FromSession $session
Copy-Item 'C:\App\logs\' -Destination '.\RemoteLogs\' -FromSession $session -Recurse

$session | Remove-PSSession
```

---

## 6. JEA (Just Enough Administration)

```powershell
# JEA: ให้สิทธิ์เฉพาะที่จำเป็น
New-PSRoleCapabilityFile -Path 'OpsTeam.psrc' -VisibleCmdlets @{
    Name = 'Restart-Service'
    Parameters = @{ Name = 'Name'; ValidateSet = 'Spooler','W32Time' }
} -VisibleFunctions 'Get-ServiceStatus'

# Session configuration
New-PSSessionConfigurationFile -Path 'OpsEndpoint.pssc' \
    -SessionType RestrictedRemoteServer \
    -RoleDefinitions @{
        'DOMAIN\OpsTeam' = @{ RoleCapabilityFiles = 'OpsTeam.psrc' }
    } \
    -RunAsVirtualAccount

# Register
Register-PSSessionConfiguration -Name 'OpsEndpoint' \
    -Path 'OpsEndpoint.pssc' \
    -Force

# Test as user
Enter-PSSession -ComputerName 'Server01' -ConfigurationName 'OpsEndpoint'
# Only OpsTeam cmdlets available!
```

---

## 7. Remote Diagnostics Script

```powershell
function Get-ServerDiagnostics {
    param(
        [Parameter(Mandatory)]
        [string[]]$Servers,
        
        [PSCredential]$Credential
    )
    
    $params = @{ ComputerName = $Servers; ErrorAction = 'SilentlyContinue' }
    if ($Credential) { $params.Credential = $Credential }
    
    $sessions = New-PSSession @params
    
    $results = Invoke-Command -Session $sessions -ScriptBlock {
        $os   = Get-CimInstance Win32_OperatingSystem
        $cpu  = Get-CimInstance Win32_Processor
        $disk = Get-CimInstance Win32_LogicalDisk -Filter "DeviceID='C:'"
        
        [PSCustomObject]@{
            Server    = $env:COMPUTERNAME
            OS        = $os.Caption
            Uptime    = ((Get-Date) - $os.LastBootUpTime).Days
            CPU_Pct   = $cpu.LoadPercentage
            RAM_Free  = [math]::Round($os.FreePhysicalMemory/1MB, 1)
            Disk_Free = [math]::Round($disk.FreeSpace/1GB, 1)
            Status    = 'OK'
        }
    }
    
    $sessions | Remove-PSSession
    return $results
}

$diagnostics = Get-ServerDiagnostics -Servers @('Server01','Server02')
$diagnostics | Sort-Object CPU_Pct -Descending | Format-Table -AutoSize
```

---

**ก่อนหน้า ← [Part 18](Part-18.md) | ต่อไป → [Part 20: Web Development Intro](Part-20.md)**
