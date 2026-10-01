# Part 31: Processes, Services และ System Monitoring

> **ระดับ**: 🟠 Advanced | **เวลา**: ~3 ชั่วขนึ่ง

---

## 1. Process Management

```powershell
# ดู processes
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10
Get-Process -Name 'chrome' -ErrorAction SilentlyContinue
Get-Process | Where-Object { $_.WorkingSet64 -gt 500MB } | Select-Object Name, Id, @{N='MB';E={[math]::Round($_.WorkingSet64/1MB)}}

# Start process
$proc = Start-Process 'notepad.exe' -PassThru
Start-Process 'chrome.exe' -ArgumentList 'https://example.com' -WindowStyle Minimized
Start-Process 'pwsh.exe' -ArgumentList '-File C:\script.ps1' -NoNewWindow -Wait

# Stop process
Stop-Process -Id 1234 -Force
Stop-Process -Name 'notepad' -Force
Get-Process notepad | Stop-Process

# Wait for process
$proc = Start-Process 'setup.exe' -PassThru
$proc.WaitForExit(30000)  # timeout 30s
Write-Host "Exit code: $($proc.ExitCode)"

# Elevate (Run as Admin)
Start-Process 'pwsh.exe' -Verb RunAs -ArgumentList '-Command Get-Service'

# Process detail
$p = Get-Process -Name 'code' | Select-Object -First 1
$p | Select-Object Id, Name, CPU, WorkingSet64, StartTime, Path, MainWindowTitle
$p.Modules | Select-Object ModuleName, FileName | Format-Table -AutoSize
```

---

## 2. Service Management

```powershell
# Get services
Get-Service | Where-Object Status -eq Stopped | Select-Object Name, DisplayName, StartType
Get-Service -Name 'WinRM'
Get-Service | Group-Object Status

# Control services
Start-Service   'Spooler'
Stop-Service    'Spooler' -Force
Restart-Service 'Spooler'
Suspend-Service 'Spooler'  # pause
Resume-Service  'Spooler'

# Change startup type
Set-Service 'Spooler' -StartupType Automatic
Set-Service 'Spooler' -StartupType Manual
Set-Service 'Spooler' -StartupType Disabled

# Create service
New-Service `
    -Name 'MyService' `
    -BinaryPathName 'C:\Services\myapp.exe' `
    -DisplayName 'My Application Service' `
    -Description 'Custom service' `
    -StartupType Automatic

# Service recovery actions (via sc.exe)
& sc.exe failure 'MyService' reset=86400 actions=restart/60000/restart/120000/restart/0
# reset: days, actions: action/delay_ms triplets

# Remove service
Remove-Service 'MyService'  # PS 6.0+
# หรือ
& sc.exe delete 'MyService'
```

---

## 3. System Resource Monitoring

```powershell
# CPU Usage snapshot
$cpu = Get-Counter '\Processor(_Total)\% Processor Time' -SampleInterval 1 -MaxSamples 3
$avg = ($cpu.CounterSamples.CookedValue | Measure-Object -Average).Average
Write-Host "CPU: $([math]::Round($avg, 1))%"

# Memory
$os = Get-CimInstance Win32_OperatingSystem
$totalGB = [math]::Round($os.TotalVisibleMemorySize / 1MB, 1)
$freeGB  = [math]::Round($os.FreePhysicalMemory / 1MB, 1)
$usedGB  = $totalGB - $freeGB
$pct     = [math]::Round($usedGB / $totalGB * 100, 1)
Write-Host "Memory: $usedGB/$totalGB GB ($pct%)"

# Disk I/O
$disk = Get-Counter `
    '\PhysicalDisk(_Total)\Disk Read Bytes/sec', `
    '\PhysicalDisk(_Total)\Disk Write Bytes/sec' `
    -SampleInterval 1 -MaxSamples 1
$read  = [math]::Round($disk.CounterSamples[0].CookedValue / 1MB, 2)
$write = [math]::Round($disk.CounterSamples[1].CookedValue / 1MB, 2)
Write-Host "Disk I/O: Read=$read MB/s  Write=$write MB/s"

# Network
$net = Get-NetAdapterStatistics | Where-Object { $_.ReceivedBytes -gt 0 }
$net | Select-Object Name, ReceivedBytes, SentBytes
```

---

## 4. Real-time Monitor

```powershell
# หน้าติดตาม process แบบ real-time
function Watch-TopProcesses {
    param([int]$RefreshSec = 2, [int]$TopN = 10)
    while ($true) {
        Clear-Host
        $ts = Get-Date -Format 'HH:mm:ss'
        $os = Get-CimInstance Win32_OperatingSystem
        $cpu = (Get-Counter '\Processor(_Total)\% Processor Time' -MaxSamples 1).CounterSamples.CookedValue
        $memPct = [math]::Round(($os.TotalVisibleMemorySize - $os.FreePhysicalMemory) / $os.TotalVisibleMemorySize * 100, 1)
        
        Write-Host "=== Process Monitor  $ts === CPU: $([math]::Round($cpu,1))%  MEM: $memPct%" -ForegroundColor Cyan
        
        Get-Process | Sort-Object CPU -Descending | Select-Object -First $TopN |
            Format-Table @(
                @{N='PID';E={$_.Id};W=8},
                @{N='Name';E={$_.Name};W=20},
                @{N='CPU%';E={[math]::Round($_.CPU,1)};W=8},
                @{N='Mem(MB)';E={[math]::Round($_.WorkingSet64/1MB,1)};W=10},
                @{N='Threads';E={$_.Threads.Count};W=9}
            ) -AutoSize
        
        Start-Sleep $RefreshSec
    }
}

# เรียกใช้งาน
Watch-TopProcesses -RefreshSec 3 -TopN 15
```

---

## 5. Event Log

```powershell
# อ่าน Event Log
Get-EventLog -LogName System -Newest 20
Get-EventLog -LogName Application -EntryType Error -Newest 50
Get-EventLog -LogName Security -InstanceId 4624 -Newest 10  # Logon events

# WinEvent (modern)
Get-WinEvent -LogName System -MaxEvents 50 |
    Where-Object { $_.LevelDisplayName -eq 'Error' } |
    Select-Object TimeCreated, Id, Message | Format-Table -Wrap

# Filter by time
Get-WinEvent -FilterHashtable @{
    LogName   = 'System'
    Level     = 2  # Error
    StartTime = (Get-Date).AddHours(-24)
}

# เขียน event log
Write-EventLog -LogName Application -Source 'MyScript' -EventId 1000 -EntryType Information -Message 'Script started'
Write-EventLog -LogName Application -Source 'MyScript' -EventId 9999 -EntryType Error -Message 'Critical failure!'

# Clear log
Clear-EventLog -LogName Application  # ระวัง!
```

---

**ก่อนหน้า ← [Part 30](Part-30.md) | ต่อไป → [Part 32: Network & TCP/IP](Part-32.md)**
