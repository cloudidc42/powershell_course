# Part 28: Jobs, Scheduling และ Background Tasks

> **ระดับ**: 🟠 Advanced | **เวลา**: ~4 ชั่วขนึง

---

## 1. Background Jobs

```powershell
# Start-Job: สร้าง PowerShell process ใหม่
$job = Start-Job -ScriptBlock {
    param($url)
    $result = Invoke-WebRequest $url
    return $result.StatusCode
} -ArgumentList 'https://example.com'

# ดูสถานะ
Get-Job
Get-Job $job.Id
$job.State    # Running, Completed, Failed

# รอให้เสร็จ
Wait-Job $job
Wait-Job $job -Timeout 30  # max 30 sec

# รับผลลัพธ์
$result = Receive-Job $job
Write-Host "Status: $result"

Receive-Job $job -Wait          # รอ + รับ
Receive-Job $job -AutoRemoveJob # remove เมื่อเสร็จ

# ลบ
$job | Remove-Job

# Parallel jobs
$servers = @('server1','server2','server3','server4','server5')
$jobs = $servers | ForEach-Object {
    Start-Job { param($s); Test-Connection $s -Count 1 -Quiet } -ArgumentList $_
}

$jobs | Wait-Job | ForEach-Object {
    $result = Receive-Job $_
    Write-Host "$($_.Name): $result"
}
$jobs | Remove-Job
```

---

## 2. Thread Jobs (PS7+)

```powershell
# Thread jobs: เบากว่ากว่า Start-Job (ไม่สร้าง process ใหม่)
Install-Module ThreadJob -Scope CurrentUser

$job = Start-ThreadJob {
    Start-Sleep 2
    return "Hello from thread!"
}

$result = $job | Wait-Job | Receive-Job
Write-Host $result

# Parallel dengan ForEach-Object -Parallel (PS7+)
$urls = @(
    'https://httpbin.org/delay/1'
    'https://httpbin.org/delay/2'
    'https://httpbin.org/delay/1'
)

$results = $urls | ForEach-Object -Parallel {
    $sw = [System.Diagnostics.Stopwatch]::StartNew()
    try {
        Invoke-WebRequest $_ -TimeoutSec 10 | Out-Null
        [PSCustomObject]@{ Url=$_; Status='OK'; Ms=$sw.ElapsedMilliseconds }
    } catch {
        [PSCustomObject]@{ Url=$_; Status='FAIL'; Ms=$sw.ElapsedMilliseconds }
    }
} -ThrottleLimit 3

$results | Format-Table -AutoSize
Write-Host "Total wall time: ~2s (parallel)" -ForegroundColor Green
```

---

## 3. Scheduled Jobs

```powershell
# Register-ScheduledJob: PowerShell-aware Task Scheduler
$trigger = New-JobTrigger -Daily -At '3:00 AM'

$options = New-ScheduledJobOption \
    -RunElevated \
    -MultipleInstancePolicy StopExisting

$job = Register-ScheduledJob \
    -Name 'DailyBackup' \
    -Trigger $trigger \
    -ScheduledJobOption $options \
    -ScriptBlock {
        $date = Get-Date -Format 'yyyy-MM-dd'
        $dest = "C:\Backups\backup_$date.zip"
        
        # compress and backup
        Compress-Archive -Path 'C:\Data' -DestinationPath $dest -CompressionLevel Optimal
        
        # log
        "[$date] Backup complete: $dest" | Add-Content 'C:\Logs\backup.log'
    }

# ดู registered jobs
Get-ScheduledJob
Get-ScheduledJob 'DailyBackup'

# เรียกใช้ทันที
$trigger2 = New-JobTrigger -Once -At (Get-Date).AddMinutes(1)
Register-ScheduledJob -Name 'OneTime' -Trigger $trigger2 -ScriptBlock { 'Done!' }

# เอาผลลัพธ์
Get-Job -Name 'DailyBackup' | Receive-Job

# ลบ
Unregister-ScheduledJob 'OneTime'
```

---

## 4. Windows Task Scheduler

```powershell
# จัดการ Windows Scheduled Tasks
$action  = New-ScheduledTaskAction -Execute 'pwsh.exe' \
    -Argument '-NonInteractive -WindowStyle Hidden -File "C:\Scripts\backup.ps1"'

$trigger = New-ScheduledTaskTrigger -Daily -At '2:30 AM'

$settings = New-ScheduledTaskSettingsSet \
    -ExecutionTimeLimit (New-TimeSpan -Hours 2) \
    -RunOnlyIfNetworkAvailable:$false \
    -WakeToRun:$false

$principal = New-ScheduledTaskPrincipal \
    -UserId 'SYSTEM' \
    -LogonType ServiceAccount \
    -RunLevel Highest

$task = New-ScheduledTask \
    -Action $action \
    -Trigger $trigger \
    -Settings $settings \
    -Principal $principal \
    -Description 'Daily backup script'

Register-ScheduledTask -TaskName 'DailyPSBackup' -TaskPath '\Custom\' -InputObject $task

# จัดการ
Get-ScheduledTask -TaskPath '\Custom\'
Start-ScheduledTask -TaskName 'DailyPSBackup'  # run now
Stop-ScheduledTask  -TaskName 'DailyPSBackup'
Enable-ScheduledTask  -TaskName 'DailyPSBackup'
Disable-ScheduledTask -TaskName 'DailyPSBackup'
Unregister-ScheduledTask -TaskName 'DailyPSBackup' -Confirm:$false
```

---

## 5. Runspaces (Ultimate Performance)

```powershell
# Runspaces: เร็วที่สุด (ใช้ thread pool)
function Invoke-Parallel {
    param(
        [array]$Items,
        [scriptblock]$ScriptBlock,
        [int]$ThrottleLimit = 10,
        [hashtable]$Variables = @{}
    )
    
    $pool = [System.Management.Automation.Runspaces.RunspacePool]::CreateRunspacePool(
        1, $ThrottleLimit
    )
    $pool.Open()
    
    $jobs = $Items | ForEach-Object {
        $ps = [System.Management.Automation.PowerShell]::Create()
        $ps.RunspacePool = $pool
        $ps.AddScript($ScriptBlock) | Out-Null
        $ps.AddArgument($_) | Out-Null
        
        foreach ($var in $Variables.GetEnumerator()) {
            $ps.AddVariable($var.Key, $var.Value) | Out-Null
        }
        
        @{ PS = $ps; Handle = $ps.BeginInvoke() }
    }
    
    $results = $jobs | ForEach-Object {
        $_.PS.EndInvoke($_.Handle)
        $_.PS.Dispose()
    }
    
    $pool.Close()
    $pool.Dispose()
    
    return $results
}

# Usage: scan 254 hosts in parallel
$sw = [System.Diagnostics.Stopwatch]::StartNew()

$online = Invoke-Parallel (1..254) {
    param($i)
    $ip = "192.168.1.$i"
    if (Test-Connection $ip -Count 1 -Quiet -TimeoutSeconds 1) {
        [PSCustomObject]@{ IP=$ip; Status='Online' }
    }
} -ThrottleLimit 50

$sw.Stop()
Write-Host "Scanned 254 hosts in $($sw.Elapsed.TotalSeconds)s"
$online | Format-Table
```

---

**ก่อนหน้า ← [Part 27](Part-27.md) | ต่อไป → [Part 29: Active Directory](Part-29.md)**
