# Part 47: Incident Response (IR)

> **ระดับ**: 🔴 Professional/Security | **เวลา**: ~5 ชั่วขนึ่ง

---

## 1. IR Triage Script

```powershell
#Requires -RunAsAdministrator
# Incident Response Triage - เก็บข้อมูลตอนต้น

$triageDir = "C:\IR_Triage_$(Get-Date -Format yyyyMMdd_HHmmss)"
New-Item $triageDir -ItemType Directory -Force | Out-Null

function Save-TriageData {
    param([string]$Name, [scriptblock]$Collector)
    $path = "$triageDir\$Name"
    try {
        $data = & $Collector
        if ($data) {
            $data | ConvertTo-Json -Depth 5 | Set-Content "$path.json"
            $data | Export-Csv "$path.csv" -NoTypeInformation -ErrorAction SilentlyContinue
        }
    } catch {
        "Error: $_" | Set-Content "$path.error.txt"
    }
}

# Collect system state
Save-TriageData 'running_processes' {
    Get-Process | Select-Object Id, Name, Path, CPU, WorkingSet64, StartTime,
        @{N='CommandLine';E={(Get-CimInstance Win32_Process -Filter "ProcessId=$($_.Id)").CommandLine}}
}

Save-TriageData 'network_connections' {
    Get-NetTCPConnection | Where-Object { $_.State -ne 'Closed' } |
        Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort, State, OwningProcess,
            @{N='Process';E={(Get-Process $_.OwningProcess -EA SilentlyContinue).Name}}
}

Save-TriageData 'services' {
    Get-Service | Select-Object Name, DisplayName, Status, StartType
}

Save-TriageData 'scheduled_tasks' {
    Get-ScheduledTask | Get-ScheduledTaskInfo |
        Select-Object TaskName, TaskPath, LastRunTime, NextRunTime, LastTaskResult
}

Save-TriageData 'autorun_reg' {
    $keys = @(
        'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run',
        'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run'
    )
    $keys | ForEach-Object {
        $k = $_
        if (Test-Path $k) {
            Get-ItemProperty $k | Get-Member -MemberType NoteProperty |
                Where-Object Name -notmatch '^PS' |
                Select-Object @{N='Key';E={$k}}, @{N='Name';E={$_.Name}},
                    @{N='Value';E={(Get-ItemProperty $k).$($_.Name)}}
        }
    }
}

Save-TriageData 'local_users' {
    Get-LocalUser | Select-Object Name, Enabled, LastLogon, SID, PasswordLastSet
}

Save-TriageData 'dns_cache' {
    Get-DnsClientCache | Select-Object Entry, RecordName, RecordType, Data, TimeToLive
}

Save-TriageData 'prefetch' {
    if (Test-Path 'C:\Windows\Prefetch') {
        Get-ChildItem 'C:\Windows\Prefetch' *.pf |
            Select-Object Name, LastWriteTime, Length
    }
}

Write-Host "Triage data saved to: $triageDir" -ForegroundColor Green
```

---

## 2. Memory Dump และ Artifact Collection

```powershell
# เก็บ process memory dump
function Get-ProcessMemoryDump {
    param([int]$PID, [string]$OutputPath)
    
    # ต้องการ procdump.exe (Sysinternals) หรือ ProcDump from Sysinternal Suite
    if (Get-Command 'procdump.exe' -ErrorAction SilentlyContinue) {
        & procdump.exe -ma $PID $OutputPath
    } else {
        # Native .NET method
        Add-Type -TypeDefinition @'
            using System;
            using System.Runtime.InteropServices;
            public class MiniDump {
                [DllImport("dbghelp.dll")]
                public static extern bool MiniDumpWriteDump(IntPtr hProcess, int processId,
                    IntPtr hFile, int dumpType, IntPtr excParam, IntPtr userParam, IntPtr callParam);
            }
'@
        $proc = Get-Process -Id $PID
        $fs   = [IO.File]::Create($OutputPath)
        [MiniDump]::MiniDumpWriteDump($proc.Handle, $PID, $fs.SafeFileHandle.DangerousGetHandle(), 2, [IntPtr]::Zero, [IntPtr]::Zero, [IntPtr]::Zero)
        $fs.Close()
        Write-Host "Dump written to $OutputPath"
    }
}

# เก็บ Event Logs สำหรับ IR
function Export-IREventLogs {
    param([string]$OutputDir)
    
    $logs = @(
        'System', 'Application', 'Security',
        'Microsoft-Windows-PowerShell/Operational',
        'Microsoft-Windows-TaskScheduler/Operational',
        'Microsoft-Windows-Sysmon/Operational',
        'Microsoft-Windows-WMI-Activity/Operational'
    )
    
    foreach ($log in $logs) {
        $safe = $log -replace '[/\\:]', '_'
        $path = "$OutputDir\$safe.evtx"
        try {
            wevtutil.exe epl $log $path /ow:true
            Write-Host "Exported: $log"
        } catch { Write-Warning "Failed: $log" }
    }
}
```

---

## 3. IOC Detection

```powershell
# Check IOC list against running system
function Search-IOC {
    param(
        [string[]]$IpList     = @(),
        [string[]]$HashList   = @(),
        [string[]]$DomainList = @(),
        [string[]]$FileList   = @()
    )
    
    $hits = @()
    
    # IP addresses in network connections
    if ($IpList) {
        $conns = Get-NetTCPConnection -State Established
        foreach ($ip in $IpList) {
            $match = $conns | Where-Object RemoteAddress -eq $ip
            if ($match) { $hits += "[IP] $ip - PID $($match.OwningProcess)" }
        }
    }
    
    # File hashes
    if ($HashList) {
        $processes = Get-Process | Where-Object Path
        foreach ($proc in $processes) {
            try {
                $hash = (Get-FileHash $proc.Path -Algorithm MD5).Hash
                if ($HashList -contains $hash) { $hits += "[HASH] $hash - $($proc.Path)" }
            } catch { }
        }
    }
    
    # Domain names in DNS cache
    if ($DomainList) {
        $cache = Get-DnsClientCache
        foreach ($domain in $DomainList) {
            if ($cache | Where-Object Entry -like "*$domain*") { $hits += "[DNS] $domain in cache" }
        }
    }
    
    # Files
    if ($FileList) {
        foreach ($path in $FileList) {
            if (Test-Path $path) { $hits += "[FILE] Found: $path" }
        }
    }
    
    return $hits
}

# เรียกใช้
$iocs = Search-IOC `
    -IpList @('185.220.101.1', '198.51.100.5') `
    -HashList @('5f4dcc3b5aa765d61d8327deb882cf99') `
    -DomainList @('evil.example.com', 'malware.cc') `
    -FileList @('C:\Windows\Temp\payload.exe', 'C:\Users\Public\run.ps1')

if ($iocs) {
    Write-Host 'IOC MATCHES FOUND:' -ForegroundColor Red
    $iocs | ForEach-Object { Write-Host "  $_" -ForegroundColor Yellow }
} else {
    Write-Host 'No IOC matches' -ForegroundColor Green
}
```

---

**ก่อนหน้า ← [Part 46](Part-46.md) | ต่อไป → [Part 48: Digital Forensics](Part-48.md)**
