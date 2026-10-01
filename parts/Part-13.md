# Part 13: File System Operations

> **ระดับ**: 🟡 Intermediate | **เวลา**: ~3 ชั่วโมง

---

## 1. Paths และ Navigation

```powershell
# Path operations
$path = 'C:\Users\Alice\Documents\report.pdf'

[System.IO.Path]::GetFileName($path)      # report.pdf
[System.IO.Path]::GetFileNameWithoutExtension($path)  # report
[System.IO.Path]::GetExtension($path)     # .pdf
[System.IO.Path]::GetDirectoryName($path) # C:\Users\Alice\Documents
[System.IO.Path]::GetFullPath('.\foo')    # full path
[System.IO.Path]::Combine('C:\tmp', 'a', 'b.txt')  # C:\tmp\a\b.txt
[System.IO.Path]::GetTempFileName()       # temp file
[System.IO.Path]::GetTempPath()           # temp directory

# PowerShell equivalents
Split-Path $path                   # C:\Users\Alice\Documents
Split-Path $path -Leaf             # report.pdf
Split-Path $path -Parent           # same as no qualifier
Join-Path 'C:\tmp' 'a' 'b.txt'    # C:\tmp\a\b.txt
Resolve-Path '.'

# Test existence
Test-Path 'C:\Windows'             # True
Test-Path 'C:\Windows' -PathType Container  # dir
Test-Path 'C:\Windows\notepad.exe' -PathType Leaf  # file
```

---

## 2. Get-ChildItem แบบ Advanced

```powershell
# สำรวจไฟล์
Get-ChildItem 'C:\Windows' -Filter '*.dll'
Get-ChildItem 'C:\Windows' -Include '*.dll','*.exe' -Recurse
Get-ChildItem 'C:\Windows' -Exclude '*.log' -Recurse

# เฉพาะ files หรือ directories
Get-ChildItem 'C:\Windows' -File             # files only
Get-ChildItem 'C:\Windows' -Directory        # dirs only
Get-ChildItem 'C:\Windows' -Recurse -File    # all files, recursive

# เอาไฟล์ hidden และ system
Get-ChildItem 'C:\' -Force    # show hidden
Get-ChildItem 'C:\' -Hidden   # only hidden

# File properties
Get-ChildItem 'C:\Windows\*.dll' | 
    Select-Object Name, 
        Length, 
        @{N='Size_KB';E={[math]::Round($_.Length/1KB,1)}},
        LastWriteTime,
        Attributes

# เรียงเรียงตามขนาด
Get-ChildItem 'C:\Windows' -File -Recurse -ErrorAction SilentlyContinue |
    Sort-Object Length -Descending |
    Select-Object -First 10 |
    Format-Table Name, @{N='MB';E={"{0:N1}" -f ($_.Length/1MB)}} -AutoSize
```

---

## 3. อ่าน/เขียนไฟล์

```powershell
# อ่าน text
$content = Get-Content 'file.txt'
$content = Get-Content 'file.txt' -Raw        # one string
$lines = Get-Content 'file.txt' -TotalCount 10  # first 10 lines
$tail = Get-Content 'file.txt' -Tail 20       # last 20 lines

# อ่านแบบ stream (ไฟล์ใหญ่)
$reader = [System.IO.StreamReader]::new('large.txt')
try {
    while (!$reader.EndOfStream) {
        $line = $reader.ReadLine()
        # process $line
    }
} finally {
    $reader.Dispose()
}

# เขียน text
Set-Content 'output.txt' 'Hello, World!'
Add-Content 'output.txt' 'Second line'
"Third line" | Out-File 'output.txt' -Append

# Here-string to file
@'
Line 1
Line 2
Line 3
'@ | Set-Content 'multiline.txt'

# Encoding
Get-Content 'file.txt' -Encoding UTF8
Set-Content 'out.txt' $content -Encoding UTF8
[System.IO.File]::WriteAllText('out.txt', $text, [System.Text.Encoding]::UTF8)

# อ่าน binary
$bytes = [System.IO.File]::ReadAllBytes('image.png')
[System.IO.File]::WriteAllBytes('copy.png', $bytes)
```

---

## 4. Copy, Move, Delete

```powershell
# Copy
Copy-Item 'source.txt' 'dest.txt'
Copy-Item 'C:\Source\' 'C:\Dest\' -Recurse           # copy folder
Copy-Item 'C:\Source\' 'C:\Dest\' -Recurse -Force    # overwrite
Copy-Item '*.ps1' 'C:\Scripts\'                       # multiple

# Move
Move-Item 'old.txt' 'new.txt'
Move-Item 'C:\old\' 'C:\new\'

# Rename
Rename-Item 'old.txt' 'new.txt'
Get-ChildItem '*.txt' | Rename-Item -NewName { $_.Name -replace '\.txt$', '.md' }

# Delete
Remove-Item 'file.txt'
Remove-Item 'folder' -Recurse          # delete folder and contents
Remove-Item 'folder' -Recurse -Force   # force delete

# ตรวจสอบก่อนลบ
if (Test-Path 'temp') {
    Remove-Item 'temp' -Recurse -Confirm:$false
}

# WhatIf - simulate
Remove-Item 'folder' -Recurse -WhatIf   # don't actually delete
```

---

## 5. New-Item และ Directories

```powershell
# สร้างไฟล์
New-Item 'new.txt' -ItemType File
New-Item 'new.txt' -ItemType File -Value 'Hello!'

# สร้าง directory
New-Item 'NewFolder' -ItemType Directory
$null = New-Item -Path 'C:\deep\nested\folder' -ItemType Directory -Force

# PowerShell 5.1 shortcut
md 'NewFolder'          # mkdir alias

# Ensure directory exists
function Ensure-Directory {
    param([string]$Path)
    if (!(Test-Path $Path -PathType Container)) {
        $null = New-Item -Path $Path -ItemType Directory -Force
        Write-Verbose "Created: $Path"
    }
    return $Path
}

# Temp directory
$tempDir = Join-Path ([System.IO.Path]::GetTempPath()) ([System.IO.Path]::GetRandomFileName())
$null = New-Item $tempDir -ItemType Directory
try {
    # work in $tempDir
} finally {
    Remove-Item $tempDir -Recurse -Force
}
```

---

## 6. File Attributes และ ACLs

```powershell
# Attributes
$file = Get-Item 'C:\Windows\notepad.exe'
$file.Attributes        # Normal, ReadOnly, Hidden, System...

# Set/clear attributes
$file.Attributes = 'Hidden'
$file.Attributes = 'Normal'
$file.Attributes += [System.IO.FileAttributes]::ReadOnly
$file.Attributes -= [System.IO.FileAttributes]::ReadOnly

# Set-ItemProperty
Set-ItemProperty 'secret.txt' -Name Attributes -Value 'Hidden,Archive'

# ACL
$acl = Get-Acl 'C:\sensitive'
$acl.Access | Format-Table IdentityReference, FileSystemRights, AccessControlType -AutoSize

# เพิ่มสิทธิ์
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "DOMAIN\User", "Read", "Allow"
)
$acl.SetAccessRule($rule)
Set-Acl 'C:\shared' $acl
```

---

## 7. FileSystemWatcher (Monitor Changes)

```powershell
function Watch-Directory {
    param(
        [string]$Path = '.',
        [string]$Filter = '*.*',
        [switch]$Recurse
    )
    
    $watcher = [System.IO.FileSystemWatcher]::new($Path, $Filter)
    $watcher.IncludeSubdirectories = [bool]$Recurse
    $watcher.EnableRaisingEvents = $true
    
    # Event handlers
    $action = {
        $name = $Event.SourceEventArgs.Name
        $changeType = $Event.SourceEventArgs.ChangeType
        $time = $Event.TimeGenerated
        Write-Host "[$time] $changeType : $name"
    }
    
    Register-ObjectEvent $watcher 'Created' -Action $action | Out-Null
    Register-ObjectEvent $watcher 'Deleted' -Action $action | Out-Null
    Register-ObjectEvent $watcher 'Changed' -Action $action | Out-Null
    Register-ObjectEvent $watcher 'Renamed' -Action {
        $name = $Event.SourceEventArgs.Name
        $old  = $Event.SourceEventArgs.OldName
        Write-Host "[RENAMED] $old -> $name"
    } | Out-Null
    
    Write-Host "Watching '$Path' (Ctrl+C to stop)"
    
    try {
        while ($true) { Start-Sleep 1 }
    } finally {
        $watcher.Dispose()
        Get-EventSubscriber | Unregister-Event
    }
}

# Watch-Directory -Path 'C:\Projects' -Recurse
```

---

## 8. Practical: File Utilities

```powershell
# หาไฟล์ซ้ำ (duplicates)
function Find-DuplicateFiles {
    param([string]$Path = '.', [switch]$Recurse)
    
    $params = @{ File = $true; ErrorAction = 'SilentlyContinue' }
    if ($Recurse) { $params.Recurse = $true }
    
    Get-ChildItem $Path @params |
        Group-Object Length |
        Where-Object Count -gt 1 |
        ForEach-Object {
            $_.Group | Group-Object { 
                $hash = Get-FileHash $_.FullName -Algorithm MD5
                $hash.Hash
            }
        } |
        Where-Object Count -gt 1 |
        ForEach-Object {
            Write-Host "Duplicates ($($_.Count) files, Hash: $($_.Name)):"
            $_.Group | ForEach-Object { Write-Host "  $($_.FullName)" }
        }
}

# สร้าง directory tree report
function Get-DirectoryTree {
    param([string]$Path = '.', [int]$Depth = 0, [int]$MaxDepth = 5)
    
    if ($Depth -gt $MaxDepth) { return }
    
    $indent = '  ' * $Depth
    $dir = Get-Item $Path
    $size = (Get-ChildItem $Path -Recurse -File -ErrorAction SilentlyContinue | 
             Measure-Object Length -Sum).Sum
    
    Write-Host "${indent}[$(Split-Path $Path -Leaf)] $("{0:N1}MB" -f ($size/1MB))"
    
    Get-ChildItem $Path -Directory -ErrorAction SilentlyContinue | ForEach-Object {
        Get-DirectoryTree $_.FullName ($Depth + 1) $MaxDepth
    }
}

Get-DirectoryTree 'C:\Projects' -MaxDepth 3
```

---

**ก่อนหน้า ← [Part 12](Part-12.md) | ต่อไป → [Part 14: CSV/JSON/XML](Part-14.md)**
