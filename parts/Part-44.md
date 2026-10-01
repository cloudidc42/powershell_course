# Part 44: Penetration Testing Lab Setup

> **ระดับ**: 🔴 Professional/Security | **เวลา**: ~5 ชั่วขนึ่ง

> ⚠️ **คำเตือน**: เนื้อหานี้ใช้สำหรับ **สภาพแวดล้อมที่ควบคุม** (โฮมแลป, VMs ส่วนตัว, CTF) **เท่านั้น** ห้ามใช้กับระบบจริงโดยไม่ได้รัปอนุญาตเป็นลายลักษณ์อักษร

---

## 1. Lab Environment Setup

```powershell
# ติดตั้ง lab VMs ด้วย Hyper-V

# Enable Hyper-V
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All -NoRestart

# สร้าง virtual switch
New-VMSwitch -Name 'LabInternal' -SwitchType Internal
New-VMSwitch -Name 'LabExternal' -NetAdapterName 'Ethernet' -AllowManagementOS $true

# NAT สำหรับ lab network
New-NetNat -Name 'LabNAT' -InternalIPInterfaceAddressPrefix '192.168.100.0/24'

# สร้าง VM
function New-LabVM {
    param(
        [string]$Name,
        [string]$ISO,
        [int]   $MemGB = 2,
        [int]   $DiskGB = 60,
        [string]$Switch = 'LabInternal'
    )
    
    $vmPath = "C:\VMs\$Name"
    New-Item $vmPath -ItemType Directory -Force
    
    $vm = New-VM -Name $Name `
        -Generation 2 `
        -MemoryStartupBytes ($MemGB * 1GB) `
        -Path $vmPath `
        -SwitchName $Switch
    
    $vhd = "$vmPath\$Name.vhdx"
    New-VHD -Path $vhd -SizeBytes ($DiskGB * 1GB) -Dynamic
    Add-VMHardDiskDrive -VM $vm -Path $vhd
    
    if ($ISO) {
        $dvd = Add-VMDvdDrive -VM $vm -Path $ISO
        Set-VMFirmware -VM $vm -FirstBootDevice $dvd
    }
    
    Set-VMProcessor $vm -Count 2
    Enable-VMIntegrationService $vm -Name 'Guest Service Interface'
    Start-VM $vm
    Write-Host "VM '$Name' created and started"
}

# สร้าง lab VMs
New-LabVM -Name 'DC01' -ISO 'C:\ISOs\Windows_Server_2022.iso' -MemGB 4
New-LabVM -Name 'WS01' -ISO 'C:\ISOs\Windows_10.iso' -MemGB 2
New-LabVM -Name 'Kali' -ISO 'C:\ISOs\kali-linux-2024.iso' -MemGB 4
```

---

## 2. Reconnaissance (ข้อมูลระบบ)

```powershell
# System enumeration (authorized targets only)
function Get-SystemInfo {
    [CmdletBinding()]
    param([string]$ComputerName = $env:COMPUTERNAME)
    
    $os   = Get-CimInstance Win32_OperatingSystem -ComputerName $ComputerName
    $cs   = Get-CimInstance Win32_ComputerSystem  -ComputerName $ComputerName
    $net  = Get-NetIPAddress -AddressFamily IPv4 | Where-Object { $_.IPAddress -notlike '169.*' -and $_.IPAddress -ne '127.0.0.1' }
    
    [PSCustomObject]@{
        Hostname       = $env:COMPUTERNAME
        Domain         = $env:USERDOMAIN
        OS             = "$($os.Caption) $($os.Version)"
        Architecture   = $os.OSArchitecture
        LastBoot       = $os.LastBootUpTime
        CurrentUser    = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name
        IsAdmin        = ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
        IPs            = $net.IPAddress -join ', '
        Antivirus      = (Get-CimInstance -Namespace root/SecurityCenter2 -ClassName AntiVirusProduct -ErrorAction SilentlyContinue).displayName
        PS_Version     = $PSVersionTable.PSVersion
        CLM_Mode       = $ExecutionContext.SessionState.LanguageMode
    }
}

Get-SystemInfo | Format-List

# Enumerate local users and groups
Get-LocalUser | Select-Object Name, Enabled, LastLogon, PasswordExpires
Get-LocalGroupMember 'Administrators'

# Running services
Get-Service | Where-Object Status -eq Running | 
    Select-Object Name, DisplayName, StartType |
    Sort-Object DisplayName

# Scheduled tasks (suspicious)
Get-ScheduledTask | Where-Object { 
    $_.TaskPath -notlike '\Microsoft\*' 
} | Select-Object TaskName, TaskPath, State | Format-Table -AutoSize
```

---

## 3. Network Enumeration

```powershell
# Host discovery (authorized lab network)
function Find-LiveHosts {
    param([string]$Subnet = '192.168.100')  # /24
    
    $alive = 1..254 | ForEach-Object -Parallel {
        $ip = "$($using:Subnet).$_"
        $ping = Test-Connection $ip -Count 1 -TimeoutSeconds 1 -ErrorAction SilentlyContinue
        if ($ping) { $ip }
    } -ThrottleLimit 50
    
    return $alive | Sort-Object { [System.Version]$_ }
}

$hosts = Find-LiveHosts '192.168.100'
Write-Host "Found $($hosts.Count) live hosts: $($hosts -join ', ')"

# Enumerate shares
function Get-NetworkShares {
    param([string[]]$Hosts)
    $Hosts | ForEach-Object {
        $h = $_
        try {
            Get-SmbShare -CimSession $h -ErrorAction Stop |
                Select-Object @{N='Host';E={$h}}, Name, Path, Description
        } catch { }
    }
}

# Banner grabbing
function Get-ServiceBanner {
    param([string]$Host, [int]$Port, [int]$Timeout = 1000)
    try {
        $tcp    = [System.Net.Sockets.TcpClient]::new()
        $ar     = $tcp.BeginConnect($Host, $Port, $null, $null)
        $ok     = $ar.AsyncWaitHandle.WaitOne($Timeout)
        if ($ok) {
            $stream = $tcp.GetStream()
            $buf    = New-Object byte[] 1024
            if ($stream.DataAvailable) {
                $n = $stream.Read($buf, 0, 1024)
                [System.Text.Encoding]::ASCII.GetString($buf, 0, $n).Trim()
            }
        }
        $tcp.Close()
    } catch { }
}
```

---

## 4. PowerShell Remoting for Lab

```powershell
# Enable remoting ใน lab VM
Enable-PSRemoting -Force
Set-Item WSMan:\localhost\Client\TrustedHosts -Value '192.168.100.*' -Force

# Test เชื่อมต่อ
$cred = Get-Credential
$session = New-PSSession -ComputerName '192.168.100.10' -Credential $cred -ErrorAction Stop
Invoke-Command $session { whoami; hostname; Get-Process | Select-Object -First 5 }
Remove-PSSession $session

# SSH remoting (PS 7+)
$session = New-PSSession -HostName '192.168.100.10' -UserName 'labuser' -SSHTransport
```

---

**ก่อนหน้า ← [Part 43](Part-43.md) | ต่อไป → [Part 45: AD Security](Part-45.md)**
