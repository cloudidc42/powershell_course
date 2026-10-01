# Part 28: Scheduled Tasks และ Automation

> **ระดับ**: 🟠 Advanced | **เวลา**: ~3 ชั่วขนึ่ง

---

## 1. Task Scheduler

```powershell
# สร้าง Scheduled Task
$action = New-ScheduledTaskAction \
    -Execute 'pwsh.exe' \
    -Argument '-NonInteractive -File C:\Scripts\backup.ps1' \
    -WorkingDirectory 'C:\Scripts'

$trigger = New-ScheduledTaskTrigger \
    -Daily -At '02:00AM'

$settings = New-ScheduledTaskSettingsSet \
    -ExecutionTimeLimit (New-TimeSpan -Hours 1) \
    -RestartCount 3 \
    -RestartInterval (New-TimeSpan -Minutes 1) \
    -RunOnlyIfNetworkAvailable \
    -WakeToRun

$principal = New-ScheduledTaskPrincipal \
    -UserId 'NT AUTHORITY\SYSTEM' \
    -RunLevel Highest

Register-ScheduledTask \
    -TaskName 'DailyBackup' \
    -TaskPath '\MyTasks\' \
    -Action $action \
    -Trigger $trigger \
    -Settings $settings \
    -Principal $principal \
    -Description 'Daily backup at 2 AM'

# Manage tasks
Get-ScheduledTask -TaskPath '\MyTasks\'
Get-ScheduledTask -TaskName 'DailyBackup' | Get-ScheduledTaskInfo

Start-ScheduledTask  -TaskName 'DailyBackup'
Stop-ScheduledTask   -TaskName 'DailyBackup'
Enable-ScheduledTask -TaskName 'DailyBackup'
Disable-ScheduledTask -TaskName 'DailyBackup'
Unregister-ScheduledTask -TaskName 'DailyBackup' -Confirm:$false
```

---

## 2. เพิ่มเติม Triggers

```powershell
# Trigger หลายส์
# Daily
$daily = New-ScheduledTaskTrigger -Daily -At '06:00'

# Weekly
$weekly = New-ScheduledTaskTrigger -Weekly -DaysOfWeek Monday,Wednesday,Friday -At '08:00'

# Monthly
$monthly = New-ScheduledTaskTrigger -Monthly -DaysOfMonth 1 -At '00:00'

# At startup
$startup = New-ScheduledTaskTrigger -AtStartup

# At logon
$logon = New-ScheduledTaskTrigger -AtLogOn -User 'DOMAIN\Alice'

# Once
$once = New-ScheduledTaskTrigger -Once -At (Get-Date).AddHours(1)

# Repeat within trigger
$repeat = New-ScheduledTaskTrigger -Daily -At '00:00'
$repeat.Repetition = (
    New-Object Microsoft.Management.Infrastructure.CimInstance 'MSFT_TaskRepetitionPattern',
    'Root/Microsoft/Windows/TaskScheduler'
)
# เรียกทุก 15 นาที
# Set-ScheduledTask แบบเต็มถ้วย COM interface

# Event trigger
$eventTrigger = New-ScheduledTaskTrigger `
    -OnEvent `
    -Log 'Application' `
    -Source 'Application Error' `
    -EventId 1000
```

---

## 3. Automation Scripts

```powershell
# backup.ps1 - สําหรับใช้เป็น scheduled task
#Requires -Version 5.1

param(
    [string]$SourcePath  = 'C:\Data',
    [string]$BackupRoot  = 'D:\Backups',
    [int]   $KeepDays    = 30,
    [string]$LogFile     = 'C:\Logs\backup.log'
)

function Write-Log {
    param([string]$Message, [string]$Level = 'INFO')
    $ts   = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $line = "[$ts] [$Level] $Message"
    $line | Add-Content $LogFile
    Write-Host $line
}

try {
    $datestamp = Get-Date -Format 'yyyy-MM-dd_HHmmss'
    $dest      = Join-Path $BackupRoot $datestamp
    
    Write-Log "Starting backup: $SourcePath -> $dest"
    New-Item $dest -ItemType Directory -Force | Out-Null
    
    $sw = [System.Diagnostics.Stopwatch]::StartNew()
    Copy-Item $SourcePath -Destination $dest -Recurse -Force
    $sw.Stop()
    
    $size = (Get-ChildItem $dest -Recurse | Measure-Object Length -Sum).Sum
    Write-Log "Backup complete: $([math]::Round($size/1MB,1))MB in $($sw.Elapsed.TotalSeconds)s"
    
    # Cleanup old backups
    $cutoff = (Get-Date).AddDays(-$KeepDays)
    $old = Get-ChildItem $BackupRoot -Directory | Where-Object { $_.CreationTime -lt $cutoff }
    foreach ($dir in $old) {
        Write-Log "Removing old backup: $($dir.Name)"
        Remove-Item $dir.FullName -Recurse -Force
    }
    
    Write-Log "Done. Removed $($old.Count) old backups."
    exit 0
} catch {
    Write-Log "BACKUP FAILED: $_" 'ERROR'
    exit 1
}
```

---

## 4. Self-Healing Script

```powershell
# monitor.ps1 - รันทุก 5 นาที ตรวจสอบ service

$services = @('W3SVC', 'MSSQLSERVER', 'WSearch')

function Ensure-ServiceRunning {
    param([string]$ServiceName)
    
    $svc = Get-Service $ServiceName -ErrorAction SilentlyContinue
    if (!$svc) {
        Write-EventLog -LogName Application -Source 'ServiceMonitor' \
            -EventId 9001 -EntryType Error \
            -Message "Service '$ServiceName' not found"
        return
    }
    
    if ($svc.Status -ne 'Running') {
        Write-Host "$ServiceName is $($svc.Status), restarting..."
        Start-Service $ServiceName
        Start-Sleep 3
        
        $svc.Refresh()
        if ($svc.Status -eq 'Running') {
            Write-Host "$ServiceName restarted OK"
        } else {
            Write-Error "$ServiceName failed to restart!"
            # ส่งแจ้ง email/Teams/Slack
        }
    }
}

# Register event source ถ้ายังไม่มี
if (![System.Diagnostics.EventLog]::SourceExists('ServiceMonitor')) {
    New-EventLog -LogName Application -Source 'ServiceMonitor'
}

$services | ForEach-Object { Ensure-ServiceRunning $_ }
```

---

## 5. Windows Service (สร้าง service เอง)

```powershell
# ติดตั้ง NSSM (Non-Sucking Service Manager) หรือ
# ใช้ New-Service cmdlet

# PowerShell script เป็น Windows Service
# sc.exe create MyPSService binpath="pwsh -NonInteractive -File C:\Services\myservice.ps1"

# สร้างด้วย New-Service
New-Service `
    -Name 'MyMonitorService' `
    -BinaryPathName 'pwsh -NonInteractive -ExecutionPolicy Bypass -File C:\Services\monitor.ps1' `
    -DisplayName 'My Monitor Service' `
    -Description 'PowerShell monitoring service' `
    -StartupType Automatic

# จัดการ
 Start-Service MyMonitorService
Get-Service MyMonitorService
Stop-Service MyMonitorService
Remove-Service MyMonitorService  # PS 6.0+

# myservice.ps1 - ลูปเป็น service
while ($true) {
    try {
        # ทำงาน
        Get-Process | Where-Object { $_.CPU -gt 90 } | ForEach-Object {
            Write-EventLog -LogName Application -Source 'MyService' \
                -EventId 1001 -EntryType Warning \
                -Message "High CPU: $($_.Name) = $($_.CPU)%"
        }
    } catch { }
    Start-Sleep 60
}
```

---

**ก่อนหน้า ← [Part 27](Part-27.md) | ต่อไป → [Part 29: Active Directory](Part-29.md)**
