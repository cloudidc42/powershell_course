# Part 30: Registry และ Windows Configuration

> **ระดับ**: 🟠 Advanced | **เวลา**: ~3 ชั่วขนึ่ง

---

## 1. Registry Provider

```powershell
# Registry คือ PSDrive ใน PowerShell
Get-PSDrive | Where-Object Provider -like '*Registry*'
# HKLM = HKEY_LOCAL_MACHINE
# HKCU = HKEY_CURRENT_USER

# Browse registry เหมือน filesystem
Set-Location HKLM:
Get-ChildItem HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion
Get-Item 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion'

# อ่านค่า
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' -Name ProductName
$ver = (Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion').CurrentVersion

# เขียน/แก้ไขค่า
New-Item 'HKCU:\SOFTWARE\MyApp' -Force
New-ItemProperty 'HKCU:\SOFTWARE\MyApp' -Name 'Version' -Value '1.0' -PropertyType String
Set-ItemProperty 'HKCU:\SOFTWARE\MyApp' -Name 'Debug' -Value 1 -Type DWord

# ลบ
Remove-ItemProperty 'HKCU:\SOFTWARE\MyApp' -Name 'Debug'
Remove-Item 'HKCU:\SOFTWARE\MyApp' -Recurse
```

---

## 2. Registry Value Types

```powershell
$key = 'HKCU:\SOFTWARE\MyApp'
New-Item $key -Force

# String (REG_SZ)
New-ItemProperty $key -Name 'AppName' -Value 'MyApp' -PropertyType String

# DWORD (32-bit integer)
New-ItemProperty $key -Name 'MaxConn' -Value 100 -PropertyType DWord

# QWORD (64-bit integer)
New-ItemProperty $key -Name 'FileSize' -Value 9999999999 -PropertyType QWord

# Multi-String (REG_MULTI_SZ)
New-ItemProperty $key -Name 'Servers' -Value @('srv1','srv2','srv3') -PropertyType MultiString

# Expandable String (REG_EXPAND_SZ) - เขียน %APPDATA%
New-ItemProperty $key -Name 'DataPath' -Value '%APPDATA%\MyApp' -PropertyType ExpandString

# Binary
New-ItemProperty $key -Name 'SomeData' -Value ([byte[]](0x01,0x02,0x03)) -PropertyType Binary

# อ่าน และแสดง
Get-Item $key | Select-Object -ExpandProperty Property |
    ForEach-Object {
        $val = (Get-ItemProperty $key).$_
        [PSCustomObject]@{ Name=$_; Value=$val; Type=$val.GetType().Name }
    }
```

---

## 3. Autorun และ Startup

```powershell
# ดู startup locations
$startupKeys = @(
    'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run',
    'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce',
    'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run',
    'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce'
)

$startupKeys | ForEach-Object {
    $key = $_
    if (Test-Path $key) {
        Get-ItemProperty $key | 
            Get-Member -MemberType NoteProperty |
            Where-Object Name -notmatch '^PS' |
            ForEach-Object {
                [PSCustomObject]@{
                    Location = $key.Replace('HKLM:\','HKLM\').Replace('HKCU:\','HKCU\')
                    Name     = $_.Name
                    Value    = (Get-ItemProperty $key).$($_.Name)
                }
            }
    }
} | Format-Table -AutoSize

# เพิ่ม autorun
New-ItemProperty 'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run' `
    -Name 'MyApp' -Value 'C:\MyApp\myapp.exe' -PropertyType String

# ลบ autorun
Remove-ItemProperty 'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run' -Name 'MyApp'
```

---

## 4. Registry Backup และ Compare

```powershell
# Export key เป็น .reg file
reg.exe export 'HKCU\SOFTWARE\MyApp' 'C:\Backup\myapp.reg' /y

# Import
reg.exe import 'C:\Backup\myapp.reg'

# Snapshot key เป็น hashtable
function Get-RegistrySnapshot {
    param([string]$KeyPath)
    $snapshot = @{}
    if (Test-Path $KeyPath) {
        $props = Get-ItemProperty $KeyPath
        $props | Get-Member -MemberType NoteProperty |
            Where-Object Name -notmatch '^PS' |
            ForEach-Object { $snapshot[$_.Name] = $props.$($_.Name) }
        Get-ChildItem $KeyPath -ErrorAction SilentlyContinue |
            ForEach-Object { $snapshot["[SubKey:$($_.PSChildName)]"] = $_.SubKeyCount }
    }
    return $snapshot
}

# Compare before/after
$before = Get-RegistrySnapshot 'HKCU:\SOFTWARE\MyApp'
# ... เปลี่ยนแปลงบางอย่าง ...
$after  = Get-RegistrySnapshot 'HKCU:\SOFTWARE\MyApp'

$diff = Compare-Object ($before.Keys | Sort-Object) ($after.Keys | Sort-Object)
if ($diff) { Write-Host 'Keys changed:'; $diff }
```

---

## 5. System Configuration

```powershell
# Windows settings via registry

# เปิดใช้ Dark Mode
Set-ItemProperty 'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Themes\Personalize' `
    -Name AppsUseLightTheme -Value 0

# ซ่อน file extension
Set-ItemProperty 'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\Advanced' `
    -Name HideFileExt -Value 0

# ซ่อน hidden files
Set-ItemProperty 'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\Advanced' `
    -Name Hidden -Value 1

# UAC level
Set-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' `
    -Name ConsentPromptBehaviorAdmin -Value 5
# 0=no prompt, 1=creds on secure, 2=creds, 3=consent on secure, 4=consent, 5=default

# Remote Desktop enable
Set-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server' `
    -Name fDenyTSConnections -Value 0
# เปิด firewall rule
Enable-NetFirewallRule -DisplayGroup 'Remote Desktop'
```

---

**ก่อนหน้า ← [Part 29](Part-29.md) | ต่อไป → [Part 31: Processes & Services](Part-31.md)**
