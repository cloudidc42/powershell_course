# Part 56: SQL Server DBA ด้วย PowerShell

> **ระดับ**: 🟡 Professional | **เวลา**: ~4 ชั่วขนึ่ง

---

## 1. dbatools Module

```powershell
# dbatools = community SQL Server module
Install-Module dbatools -Force

# Connect
$sql = Connect-DbaInstance -SqlInstance 'SQL01' -TrustServerCertificate

# Database info
Get-DbaDatabase -SqlInstance 'SQL01' | 
    Select-Object Name, Status, Size, Owner, CreateDate | 
    Format-Table -AutoSize

# Server info
Get-DbaInstanceProperty -SqlInstance 'SQL01' | Select-Object Name, Value
Get-DbaMaxMemory -SqlInstance 'SQL01'

# Jobs
Get-DbaAgentJob -SqlInstance 'SQL01' | Select-Object Name, Enabled, LastRunDate, LastRunOutcome
Start-DbaAgentJob -SqlInstance 'SQL01' -Job 'Nightly Backup'
```

---

## 2. Backup และ Restore

```powershell
# Backup databases
Backup-DbaDatabase `
    -SqlInstance 'SQL01' `
    -Database 'MyDB' `
    -BackupDirectory 'D:\Backups\SQL' `
    -Type Full `
    -CompressBackup

# Backup all user databases
Get-DbaDatabase -SqlInstance 'SQL01' -ExcludeSystem |
    Backup-DbaDatabase -BackupDirectory 'D:\Backups\SQL' -Type Full

# Verify backup
Test-DbaLastBackup -SqlInstance 'SQL01' -Database 'MyDB' | 
    Select-Object Database, Destination, Success

# Restore
Restore-DbaDatabase `
    -SqlInstance 'SQL02' `
    -Path 'D:\Backups\SQL\MyDB_FULL_*.bak' `
    -DatabaseName 'MyDB_Restored' `
    -WithReplace

# Automated backup script
$instances = @('SQL01','SQL02','SQL03')
foreach ($inst in $instances) {
    try {
        $result = Get-DbaDatabase -SqlInstance $inst -ExcludeSystem |
            Backup-DbaDatabase -BackupDirectory 'D:\Backups\SQL' -Type Full -CompressBackup
        Write-Host "$inst: $($result.Count) databases backed up" -ForegroundColor Green
    } catch {
        Write-Warning "$inst FAILED: $_"
    }
}
```

---

## 3. Query และ Maintenance

```powershell
# Run queries
Invoke-DbaQuery -SqlInstance 'SQL01' -Database 'MyDB' `
    -Query 'SELECT TOP 10 * FROM Orders ORDER BY OrderDate DESC'

# Index maintenance
Get-DbaIndex -SqlInstance 'SQL01' -Database 'MyDB' | 
    Where-Object { $_.FragmentationPercent -gt 30 } |
    ForEach-Object {
        Write-Host "Rebuilding $($_.Name)..."
        $_.Rebuild()
    }

# Statistics update
Update-DbaStatistic -SqlInstance 'SQL01' -Database 'MyDB'

# Check blocking
Get-DbaProcess -SqlInstance 'SQL01' -Blocked |
    Select-Object Spid, BlockingSpid, Database, Command, Status

# Kill long-running queries
Get-DbaProcess -SqlInstance 'SQL01' |
    Where-Object { $_.ElapsedTime -gt 3600 -and $_.Command -eq 'SELECT' } |
    Stop-DbaProcess

# Disk usage
Get-DbaDiskSpace -ComputerName 'SQL01' | 
    Select-Object Name, Free, Total, PercentFree |
    Format-Table -AutoSize
```

---

## 4. ติดตาม SQL Server Health

```powershell
# Health check dashboard
function Get-SqlHealthReport {
    param([string[]]$Instances)
    
    $Instances | ForEach-Object {
        $inst = $_
        try {
            $svr = Connect-DbaInstance -SqlInstance $inst -TrustServerCertificate
            
            # CPU ใช้ scheduler ring buffer
            $cpu = Invoke-DbaQuery -SqlInstance $inst -Query @"
SELECT TOP 1
    record.value('(./Record/SchedulerMonitorEvent/SystemHealth/ProcessUtilization)[1]', 'int') AS sql_cpu,
    record.value('(./Record/SchedulerMonitorEvent/SystemHealth/SystemIdle)[1]', 'int') AS sys_idle
FROM (
    SELECT CONVERT(XML, record) AS record
    FROM sys.dm_os_ring_buffers
    WHERE ring_buffer_type = N'RING_BUFFER_SCHEDULER_MONITOR'
) AS x
"@
            
            [PSCustomObject]@{
                Instance   = $inst
                Status     = 'Online'
                SQLMemGB   = [math]::Round($svr.MemoryUsageMB/1024, 1)
                UserCount  = (Get-DbaDatabase -SqlInstance $inst).Count
                SQLCpu     = "$($cpu.sql_cpu)%"
                Connections= (Get-DbaProcess -SqlInstance $inst | Where-Object IsSystem -eq $false).Count
            }
        } catch {
            [PSCustomObject]@{ Instance=$inst; Status="OFFLINE: $_" }
        }
    }
}

Get-SqlHealthReport -Instances @('SQL01','SQL02') | Format-Table -AutoSize
```

---

**ก่อนหน้า ← [Part 55](Part-55.md) | ต่อไป → [Part 57: Exchange & M365](Part-57.md)**
