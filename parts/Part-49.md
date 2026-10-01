# Part 49: Threat Hunting

> **ระดับ**: 🔴 Professional/Security | **เวลา**: ~5 ชั่วขนึ่ง

---

## 1. MITRE ATT&CK Hunting

```powershell
# ตรวจสอบตาม MITRE ATT&CK techniques

# T1059.001 - PowerShell execution (encoded commands)
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-PowerShell/Operational'; Id=4104} -MaxEvents 1000 -EA SilentlyContinue |
    ForEach-Object {
        if ($_.Message -match '-[Ee]nc|-[Ee]ncode|FromBase64') {
            [PSCustomObject]@{
                Time   = $_.TimeCreated
                Event  = 'T1059.001 Encoded PS'
                Detail = ($_.Message | Select-String '.*').Matches.Value | Select-Object -First 3 | Join-String
            }
        }
    }

# T1543.003 - New service installation
Get-WinEvent -FilterHashtable @{LogName='System'; Id=7045} -MaxEvents 100 -EA SilentlyContinue |
    ForEach-Object {
        $xml  = [xml]$_.ToXml()
        $data = $xml.Event.EventData.Data
        [PSCustomObject]@{
            Time    = $_.TimeCreated
            Event   = 'T1543.003 NewService'
            Service = ($data | Where-Object Name -eq 'ServiceName').'#text'
            Path    = ($data | Where-Object Name -eq 'ImagePath').'#text'
        }
    } | Format-Table -AutoSize

# T1053.005 - Scheduled task
Get-ScheduledTask | Where-Object {
    $_.Actions.Execute -match 'powershell|cmd|wscript|mshta|rundll32' -and
    $_.TaskPath -notlike '\Microsoft\*'
} | Select-Object TaskName, TaskPath,
    @{N='Execute';E={$_.Actions.Execute}},
    @{N='Arguments';E={$_.Actions.Arguments}} | Format-Table -AutoSize

# T1547.001 - Registry autorun
$startupKeys = @(
    'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run',
    'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run',
    'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce'
)
$startupKeys | ForEach-Object {
    $key = $_
    if (Test-Path $key) {
        $props = Get-ItemProperty $key
        $props | Get-Member -MemberType NoteProperty |
            Where-Object Name -notmatch '^PS' |
            ForEach-Object {
                $val = $props.$($_.Name)
                if ($val -match 'powershell|cmd|wscript|mshta|temp|appdata') {
                    Write-Host "[T1547] $key\$($_.Name) = $val" -ForegroundColor Yellow
                }
            }
    }
}
```

---

## 2. Living-off-the-Land (LOLBin) Detection

```powershell
# LOLBins = ตัวไฟ Windows ที่ถูก abuse
$lolbins = @(
    'mshta.exe', 'rundll32.exe', 'regsvr32.exe', 'wscript.exe',
    'cscript.exe', 'msiexec.exe', 'certutil.exe', 'bitsadmin.exe',
    'esentutl.exe', 'wmic.exe', 'regasm.exe', 'regsvcs.exe',
    'installutil.exe', 'ieexec.exe', 'mavinject.exe'
)

# Check process creation logs
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4688} -MaxEvents 5000 -EA SilentlyContinue |
    ForEach-Object {
        $xml  = [xml]$_.ToXml()
        $data = $xml.Event.EventData.Data
        $proc = ($data | Where-Object Name -eq 'NewProcessName').'#text'
        $cmd  = ($data | Where-Object Name -eq 'CommandLine').'#text'
        
        if ($lolbins | Where-Object { $proc -like "*$_" }) {
            [PSCustomObject]@{
                Time    = $_.TimeCreated
                Process = $proc
                CmdLine = $cmd
                Parent  = ($data | Where-Object Name -eq 'ParentProcessName').'#text'
            }
        }
    } | Format-Table -AutoSize

# certutil abuse patterns (download)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4688} -MaxEvents 5000 -EA SilentlyContinue |
    Where-Object { $_.Message -match 'certutil' -and $_.Message -match '-urlcache|-decode|-encode|-decodehex' } |
    Select-Object TimeCreated, Message | Format-List
```

---

## 3. Sysmon-based Hunting

```powershell
# Sysmon Event IDs:
# 1  = Process Create
# 3  = Network Connection
# 7  = Image Loaded
# 8  = CreateRemoteThread
# 11 = File Create
# 12/13 = Registry
# 22 = DNS Query
# 25 = Process Tampering

# Hunt for process injection (Event 8)
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=8} -MaxEvents 100 -EA SilentlyContinue |
    ForEach-Object {
        $xml  = [xml]$_.ToXml()
        $data = $xml.Event.EventData.Data
        [PSCustomObject]@{
            Time       = $_.TimeCreated
            SourceProc = ($data | Where-Object Name -eq 'SourceImage').'#text'
            TargetProc = ($data | Where-Object Name -eq 'TargetImage').'#text'
            StartAddr  = ($data | Where-Object Name -eq 'StartAddress').'#text'
        }
    } | Format-Table -AutoSize

# Suspicious DNS queries
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=22} -MaxEvents 500 -EA SilentlyContinue |
    ForEach-Object {
        $xml  = [xml]$_.ToXml()
        $data = $xml.Event.EventData.Data
        $query = ($data | Where-Object Name -eq 'QueryName').'#text'
        # หา DGA-like domains (long random strings)
        if ($query -match '[0-9a-z]{10,}\.[a-z]{2,4}$') {
            [PSCustomObject]@{
                Time    = $_.TimeCreated
                Query   = $query
                Process = ($data | Where-Object Name -eq 'Image').'#text'
            }
        }
    } | Format-Table -AutoSize
```

---

## 4. SIGMA Rule Implementation

```powershell
# SIGMA = standardized threat detection rules
# สร้าง PS function จาก SIGMA rule

# SIGMA rule example (YAML):
# title: Suspicious Encoded PowerShell
# logsource:
#   product: windows
#   service: powershell
# detection:
#   keywords:
#     - '-EncodedCommand'
#     - '-enc '
#   condition: keywords

function Invoke-SigmaRule {
    param(
        [hashtable]$Rule,
        [array]$Events
    )
    
    $keywords = $Rule.detection.keywords
    $matches  = $Events | Where-Object {
        $event = $_
        $keywords | Where-Object { $event.Message -match $_ }
    }
    
    [PSCustomObject]@{
        RuleName = $Rule.title
        Matches  = $matches.Count
        Events   = $matches
    }
}

$encPSRule = @{
    title     = 'Suspicious Encoded PowerShell'
    detection = @{ keywords = @('-EncodedCommand', '-enc ', 'FromBase64String') }
}

$events = Get-WinEvent -FilterHashtable @{
    LogName='Microsoft-Windows-PowerShell/Operational'; Id=4104
} -MaxEvents 1000 -EA SilentlyContinue

$result = Invoke-SigmaRule -Rule $encPSRule -Events $events
Write-Host "$($result.RuleName): $($result.Matches) matches"
```

---

**ก่อนหน้า ← [Part 48](Part-48.md) | ต่อไป → [Part 50: Red Team Automation](Part-50.md)**
