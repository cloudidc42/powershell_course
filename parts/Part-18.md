# Part 18: WMI / CIM - สอบถาม Windows

> **ระดับ**: 🟡 Intermediate | **เวลา**: ~3 ชั่วโมง

---

## 1. CIM vs WMI

```powershell
# WMI (เก่า) - ยังใช้ได้
# Get-WmiObject -Class Win32_ComputerSystem

# CIM (ใหม่, แนะนำ) - รองรับ PS Core ด้วย
$cs = Get-CimInstance -ClassName Win32_ComputerSystem
$cs.Name         # computer name
$cs.Manufacturer # Dell, HP, etc.
$cs.Model        # model name
$cs.TotalPhysicalMemory / 1GB  # RAM in GB
$cs.NumberOfProcessors
$cs.NumberOfLogicalProcessors

# OS Info
$os = Get-CimInstance Win32_OperatingSystem
$os.Caption         # Windows 11 Pro
$os.Version         # 10.0.22621
$os.BuildNumber     # 22621
$os.InstallDate
$os.LastBootUpTime

(Get-Date) - $os.LastBootUpTime  # uptime

# CPU
$cpu = Get-CimInstance Win32_Processor
$cpu.Name            # Intel Core i7...
$cpu.MaxClockSpeed   # MHz
$cpu.NumberOfCores
$cpu.LoadPercentage
```

---

## 2. Storage และ Memory

```powershell
# Disk
$disks = Get-CimInstance Win32_LogicalDisk -Filter "DriveType=3"  # local
foreach ($disk in $disks) {
    $total = $disk.Size / 1GB
    $free  = $disk.FreeSpace / 1GB
    $used  = $total - $free
    $pct   = [math]::Round($free / $total * 100)
    
    Write-Host "$($disk.DeviceID) Total:$([math]::Round($total,1))GB Free:$([math]::Round($free,1))GB ($pct% free)"
}

# Physical disk
Get-CimInstance Win32_DiskDrive | Select-Object Model, InterfaceType, MediaType,
    @{N='GB';E={[math]::Round($_.Size/1GB,1)}}

# Memory
$mem = Get-CimInstance Win32_PhysicalMemory
$mem | Select-Object BankLabel, Capacity, Speed, Manufacturer,
    @{N='GB';E={$_.Capacity/1GB}}

($mem | Measure-Object Capacity -Sum).Sum / 1GB  # total RAM

# Memory usage
$os = Get-CimInstance Win32_OperatingSystem
$totalMem = $os.TotalVisibleMemorySize / 1MB
$freeMem  = $os.FreePhysicalMemory / 1MB
$usedMem  = $totalMem - $freeMem
Write-Host "RAM: $([math]::Round($usedMem,1))GB / $([math]::Round($totalMem,1))GB"
```

---

## 3. Processes และ Services

```powershell
# Processes via CIM
$procs = Get-CimInstance Win32_Process |
    Select-Object ProcessId, Name, 
        @{N='CPU';    E={$_.KernelModeTime + $_.UserModeTime}},
        @{N='RAM_MB'; E={[math]::Round($_.WorkingSetSize/1MB,1)}},
        CommandLine |
    Sort-Object RAM_MB -Descending

# หา process พร้อม parent
$proc = Get-CimInstance Win32_Process -Filter "Name='powershell.exe'"
foreach ($p in $proc) {
    $parent = Get-CimInstance Win32_Process -Filter "ProcessId=$($p.ParentProcessId)"
    Write-Host "$($p.Name) -> parent: $($parent.Name)"
}

# Services
$services = Get-CimInstance Win32_Service | 
    Where-Object { $_.StartMode -eq 'Auto' } |
    Select-Object Name, DisplayName, State, StartMode, PathName

# เริ่ม/หยุด service
Get-CimInstance Win32_Service -Filter "Name='Spooler'" | 
    Invoke-CimMethod -MethodName StopService
Get-CimInstance Win32_Service -Filter "Name='Spooler'" | 
    Invoke-CimMethod -MethodName StartService
```

---

## 4. Network และ Hardware

```powershell
# Network adapters
Get-CimInstance Win32_NetworkAdapterConfiguration -Filter 'IPEnabled=True' | 
    Select-Object Description, IPAddress, MACAddress, DefaultIPGateway

# Network stats
Get-CimInstance Win32_PerfFormattedData_Tcpip_NetworkInterface |
    Where-Object { $_.BytesTotalPersec -gt 0 } |
    Select-Object Name, 
        @{N='RecvKB/s'; E={[math]::Round($_.BytesReceivedPersec/1KB,1)}},
        @{N='SentKB/s'; E={[math]::Round($_.BytesSentPersec/1KB,1)}}

# GPU
Get-CimInstance Win32_VideoController |
    Select-Object Caption, AdapterRAM, DriverVersion,
        @{N='RAM_GB';E={[math]::Round($_.AdapterRAM/1GB,1)}}

# BIOS
$bios = Get-CimInstance Win32_BIOS
$bios.Manufacturer
$bios.SMBIOSBIOSVersion
$bios.ReleaseDate

# Motherboard
$board = Get-CimInstance Win32_BaseBoard
$board.Manufacturer
$board.Product
$board.SerialNumber
```

---

## 5. Remote CIM

```powershell
# Remote สอบถาม (WS-Management/WSMAN)
$cimSession = New-CimSession -ComputerName 'Server01'

Get-CimInstance Win32_OperatingSystem -CimSession $cimSession |
    Select-Object Caption, Version, LastBootUpTime

Get-CimInstance Win32_Service -CimSession $cimSession |
    Where-Object { $_.State -eq 'Running' }

Remove-CimSession $cimSession

# Multiple servers
$servers = @('Server01', 'Server02', 'Server03')
$sessions = New-CimSession -ComputerName $servers

$diskInfo = Get-CimInstance Win32_LogicalDisk -CimSession $sessions |
    Where-Object DriveType -eq 3 |
    Select-Object PSComputerName, DeviceID,
        @{N='Total_GB'; E={[math]::Round($_.Size/1GB,1)}},
        @{N='Free_GB';  E={[math]::Round($_.FreeSpace/1GB,1)}},
        @{N='Pct_Free'; E={[math]::Round($_.FreeSpace/$_.Size*100)}}

$diskInfo | Format-Table
$sessions | Remove-CimSession
```

---

## 6. WMI Event Subscription

```powershell
# Monitor process creation
$query = "SELECT * FROM __InstanceCreationEvent WITHIN 1 WHERE TargetInstance ISA 'Win32_Process'"

$action = {
    $proc = $Event.SourceEventArgs.NewEvent.TargetInstance
    Write-Host "New Process: $($proc.Name) PID:$($proc.ProcessId)"
}

$subscription = Register-CimIndicationEvent \
    -Query $query \
    -SourceIdentifier 'ProcessMonitor' \
    -Action $action

Write-Host "Monitoring processes for 10 seconds..."
Start-Sleep 10
Unregister-Event 'ProcessMonitor'

# Monitor file creation
$fsQuery = @"
SELECT * FROM __InstanceCreationEvent WITHIN 2 
WHERE TargetInstance ISA 'CIM_DataFile' 
AND TargetInstance.Drive = 'C:' 
AND TargetInstance.Path = '\\Temp\\'
"@
```

---

**ก่อนหน้า ← [Part 17](Part-17.md) | ต่อไป → [Part 19: PowerShell Remoting](Part-19.md)**
