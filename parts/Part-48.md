# Part 48: Digital Forensics

> **ระดับ**: 🔴 Professional/Security | **เวลา**: ~5 ชั่วขนึ่ง

---

## 1. File Forensics

```powershell
# คำนวณ hash เพื่อ verify integrity
function Get-FileForensics {
    param([string]$Path)
    
    $file = Get-Item $Path
    [PSCustomObject]@{
        Path        = $file.FullName
        Size        = $file.Length
        Created     = $file.CreationTime
        Modified    = $file.LastWriteTime
        Accessed    = $file.LastAccessTime
        Owner       = (Get-Acl $Path).Owner
        MD5         = (Get-FileHash $Path -Algorithm MD5).Hash
        SHA1        = (Get-FileHash $Path -Algorithm SHA1).Hash
        SHA256      = (Get-FileHash $Path -Algorithm SHA256).Hash
        Signature   = (Get-AuthenticodeSignature $Path).Status
        Attributes  = $file.Attributes
        Extension   = $file.Extension
    }
}

Get-FileForensics 'C:\Windows\System32\notepad.exe' | Format-List

# Zone Identifier (ไฟล์จาก internet)
function Get-ZoneIdentifier {
    param([string]$Path)
    $zone = Get-Content "$Path:Zone.Identifier" -ErrorAction SilentlyContinue
    if ($zone) {
        $zone | ConvertFrom-StringData
    } else {
        'No Zone.Identifier (not from internet)'
    }
}

Get-ZoneIdentifier 'C:\Downloads\file.exe'

# ค้นหาไฟล์ที่ถูกดัดแปลงนามสกุล
$suspExts = @('.exe','.dll','.ps1','.bat','.cmd','.vbs','.js')
Get-ChildItem $env:TEMP, $env:TMP, 'C:\Windows\Temp', "$env:APPDATA" -Recurse -ErrorAction SilentlyContinue |
    Where-Object { $_.Extension -in $suspExts } |
    Select-Object FullName, Length, CreationTime, LastWriteTime |
    Sort-Object CreationTime -Descending
```

---

## 2. NTFS Artifacts

```powershell
# MFT (Master File Table) ด้วย PowerShell
# ต้องการ admin + raw disk access

# Alternate Data Streams
function Find-AlternateDataStreams {
    param([string]$Path = '.')
    Get-ChildItem $Path -Recurse -ErrorAction SilentlyContinue |
        ForEach-Object {
            $streams = Get-Item $_.FullName -Stream * -ErrorAction SilentlyContinue |
                Where-Object Stream -ne ':$DATA'
            if ($streams) {
                $file = $_
                $streams | ForEach-Object {
                    [PSCustomObject]@{
                        File   = $file.FullName
                        Stream = $_.Stream
                        Length = $_.Length
                    }
                }
            }
        }
}

Find-AlternateDataStreams 'C:\Users' | Format-Table

# อ่าน ADS
Get-Content 'C:\file.txt:hidden_stream'

# USN Journal (file change history)
fsutil.exe usn readjournal C: csv | ConvertFrom-Csv | Select-Object -First 50

# ตรวจสอป Recycle Bin
$recyclePath = 'C:\$Recycle.Bin'
if (Test-Path $recyclePath) {
    Get-ChildItem $recyclePath -Recurse -Force -ErrorAction SilentlyContinue |
        Where-Object { $_.Name -like '$I*' } |
        ForEach-Object {
            [PSCustomObject]@{
                RecyclePath = $_.FullName
                Created     = $_.CreationTime
                Size        = $_.Length
            }
        }
}
```

---

## 3. Registry Forensics

```powershell
# UserAssist - โปรแกรมที่ถูกเรียกใช้
$userassist = Get-ItemProperty 'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\UserAssist\{CEBFF5CD-ACE2-4F4F-9178-9926F41749EA}\Count' -ErrorAction SilentlyContinue
$userassist | Get-Member -MemberType NoteProperty | Where-Object Name -notmatch '^PS' | ForEach-Object {
    [PSCustomObject]@{
        Program = [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String((($_.Name -split '').Cast([char]) | ForEach-Object { [char]([byte][char]$_ -bxor 0x0d) }) -join ''))
        Data    = $userassist.$($_.Name)
    }
}

# MRU Lists (recently used files/commands)
$cmdMRU = Get-ItemProperty 'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\RunMRU' -ErrorAction SilentlyContinue
$cmdMRU | Get-Member -MemberType NoteProperty | Where-Object Name -match '^[a-z]$' | ForEach-Object {
    [PSCustomObject]@{ Cmd = $cmdMRU.$($_.Name) }
}

# BAM (Background Activity Monitor) - ประวัติการเรียกใช้โปรแกรม
Get-ChildItem 'HKLM:\SYSTEM\CurrentControlSet\Services\bam\State\UserSettings' -ErrorAction SilentlyContinue |
    ForEach-Object {
        $key = $_
        Get-ItemProperty $key.PSPath |
            Get-Member -MemberType NoteProperty |
            Where-Object { $_.Name -like '\Device\*' } |
            ForEach-Object {
                $raw = (Get-ItemProperty $key.PSPath).$($_.Name)
                $ts  = [BitConverter]::ToInt64($raw, 0)
                $dt  = [datetime]::FromFileTime($ts)
                [PSCustomObject]@{ Program=$_.Name; LastRun=$dt; User=$key.PSChildName }
            }
    } | Sort-Object LastRun -Descending
```

---

## 4. Timeline Analysis

```powershell
# สร้าง super timeline จากหลายแหล่ง
function Get-ForensicTimeline {
    param([datetime]$StartTime, [datetime]$EndTime)
    
    $events = @()
    
    # File system timestamps
    Get-ChildItem C:\Windows\Temp, $env:TEMP -Recurse -ErrorAction SilentlyContinue |
        Where-Object { $_.LastWriteTime -ge $StartTime -and $_.LastWriteTime -le $EndTime } |
        ForEach-Object {
            $events += [PSCustomObject]@{
                Time   = $_.LastWriteTime; Source = 'FileSystem'
                Type   = 'FileModified'; Detail = $_.FullName
            }
        }
    
    # Event log
    Get-WinEvent -FilterHashtable @{
        LogName   = 'Security'
        StartTime = $StartTime
        EndTime   = $EndTime
        Id        = @(4624,4625,4688,4657,7045)
    } -ErrorAction SilentlyContinue | ForEach-Object {
        $events += [PSCustomObject]@{
            Time   = $_.TimeCreated; Source = 'EventLog'
            Type   = "ID-$($_.Id)"; Detail = $_.Message.Substring(0, [Math]::Min(100, $_.Message.Length))
        }
    }
    
    $events | Sort-Object Time
}

Get-ForensicTimeline -StartTime (Get-Date).AddHours(-6) -EndTime (Get-Date) |
    Format-Table Time, Source, Type, Detail -AutoSize
```

---

**ก่อนหน้า ← [Part 47](Part-47.md) | ต่อไป → [Part 49: Threat Hunting](Part-49.md)**
