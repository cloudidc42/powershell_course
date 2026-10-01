# Part 43: AV/EDR Technology (Blue Team)

> **ระดับ**: 🔴 Professional/Security | **เวลา**: ~5 ชั่วขนึ่ง

> ⚠️ **สำหรับ**: นัก Blue Team, นักวิเคราะห์มัลแวร์, Security Research, Penetration Testing ที่ได้รับอนุญาต

---

## 1. AV/EDR Architecture

```
AV = Antivirus    - signature/heuristic scanning
EDR = Endpoint Detection & Response - behavioral, telemetry, threat hunting

Detection Methods:

1. Signature-based    - hash/pattern matching of known malware
2. Heuristic          - rules to catch suspicious code patterns
3. Behavioral         - monitor runtime actions (API calls, file I/O)
4. Machine Learning   - model-based classification
5. Sandboxing         - run in isolated environment
6. Memory scanning    - check process memory

EDR Telemetry Sources:
- Kernel callbacks    (process create, file I/O, registry)
- ETW providers       (Microsoft-Windows-Kernel-Process, etc.)
- Minifilter driver   (file system filter)
- User-mode hooks     (IAT hooking, inline hooking)
- AMSI               (script scanning)
- WMI subscriptions
```

---

## 2. Windows Defender via PowerShell

```powershell
# Windows Defender status
Get-MpComputerStatus | Select-Object `
    AMRunningMode, AMServiceEnabled, AntispywareEnabled, AntivirusEnabled,
    BehaviorMonitorEnabled, IoavProtectionEnabled, OnAccessProtectionEnabled,
    RealTimeProtectionEnabled, AntivirusSignatureLastUpdated

# Scan
Start-MpScan -ScanType QuickScan
Start-MpScan -ScanType FullScan
Start-MpScan -ScanType CustomScan -ScanPath 'C:\Suspicious'

# Threat history
Get-MpThreat | Select-Object ThreatName, SeverityID, CurrentStatus, DetectionTime
Get-MpThreatDetection | Format-Table -AutoSize

# Update signatures
Update-MpSignature

# Add exclusion (use with care!)
Add-MpPreference -ExclusionPath 'C:\DevTools'          # path exclusion
Add-MpPreference -ExclusionExtension '.ps1'             # extension
Add-MpPreference -ExclusionProcess 'CustomApp.exe'     # process

Get-MpPreference | Select-Object ExclusionPath, ExclusionExtension, ExclusionProcess

# Remove exclusion
Remove-MpPreference -ExclusionPath 'C:\DevTools'
```

---

## 3. EDR Telemetry (ETW)

```powershell
# ETW = Event Tracing for Windows— EDR products tap into this

# List ETW providers
logman.exe query providers | Select-String 'Microsoft-Windows-Kernel'

# ดู real-time ETW ด้วย PS
# (ต้องการ elevation)

# PowerShell-specific ETW providers
# Microsoft-Windows-PowerShell (GUID: A0C1853B-5C40-4B15-8766-3CF1C58F985A)
# Microsoft-Antimalware-AMFilter
# Microsoft-Antimalware-Engine

# Event sources AV/EDR monitors:
$monitoredEvents = @(
    @{Log='Microsoft-Windows-PowerShell/Operational'; Id=4104; Desc='Script Block Logging'},
    @{Log='Microsoft-Windows-PowerShell/Operational'; Id=4103; Desc='Module Logging'},
    @{Log='Security'; Id=4688; Desc='Process Creation (if audited)'},
    @{Log='Security'; Id=4663; Desc='Object Access'},
    @{Log='System';   Id=7045; Desc='New Service Installed'}
)
$monitoredEvents | Format-Table -AutoSize

# Monitor process creation (4688) - requires audit policy
Get-WinEvent -FilterHashtable @{
    LogName='Security'; Id=4688
} -MaxEvents 50 | ForEach-Object {
    $xml = [xml]$_.ToXml()
    $data = $xml.Event.EventData.Data
    [PSCustomObject]@{
        Time    = $_.TimeCreated
        Process = ($data | Where-Object Name -eq 'NewProcessName').'#text'
        CmdLine = ($data | Where-Object Name -eq 'CommandLine').'#text'
        Creator = ($data | Where-Object Name -eq 'ParentProcessName').'#text'
    }
} | Format-Table -AutoSize
```

---

## 4. Threat Hunting with PowerShell

```powershell
# Hunt หา suspicious processes
function Get-SuspiciousProcesses {
    $procs = Get-Process
    $suspicious = @()
    
    foreach ($p in $procs) {
        $flags = @()
        
        # Run from temp/download
        if ($p.Path -match 'Temp|Download|AppData\\Local|Users\\Public') {
            $flags += 'SuspiciousPath'
        }
        
        # No parent or orphan
        try {
            $parent = Get-Process -Id (Get-CimInstance Win32_Process -Filter "ProcessId=$($p.Id)").ParentProcessId -ErrorAction SilentlyContinue
            if (!$parent) { $flags += 'NoParent' }
        } catch {}
        
        # Network connection + not signed
        $netconn = Get-NetTCPConnection -OwningProcess $p.Id -ErrorAction SilentlyContinue
        if ($netconn) {
            $sig = Get-AuthenticodeSignature $p.Path -ErrorAction SilentlyContinue
            if ($sig.Status -ne 'Valid') { $flags += 'UnsignedWithNetwork' }
        }
        
        if ($flags.Count -gt 0) {
            $suspicious += [PSCustomObject]@{
                PID   = $p.Id
                Name  = $p.Name
                Path  = $p.Path
                Flags = $flags -join ', '
            }
        }
    }
    $suspicious
}

Get-SuspiciousProcesses | Format-Table -AutoSize
```

---

**ก่อนหน้า ← [Part 42](Part-42.md) | ต่อไป → [Part 44: Penetration Testing Lab](Part-44.md)**
