# Part 02: Installation & Environment Setup

> **ระดับ**: 🟢 Beginner  
> **เวลาที่ใช้**: ~2 ชั่วโมง  
> **เป้าหมาย**: ติดตั้ง PowerShell 7 และตั้งค่า development environment ที่สมบูรณ์

---

## 📌 สารบัญ

1. [Windows PowerShell vs PowerShell 7](#1-windows-powershell-vs-powershell-7)
2. [ติดตั้ง PowerShell 7 บน Windows](#2-ติดตั้ง-powershell-7-บน-windows)
3. [ติดตั้งบน Linux](#3-ติดตั้งบน-linux)
4. [ติดตั้งบน macOS](#4-ติดตั้งบน-macos)
5. [VS Code Setup](#5-vs-code-setup)
6. [Execution Policy](#6-execution-policy)
7. [PowerShell Profile](#7-powershell-profile)
8. [PowerShell Gallery & Modules](#8-powershell-gallery--modules)
9. [Environment Variables](#9-environment-variables)
10. [Exercises](#10-exercises)

---

## 1. Windows PowerShell vs PowerShell 7

### ความแตกต่างที่ต้องรู้

```powershell
# Windows PowerShell (built-in)
# Path: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
# Version: 5.1 (ไม่มีการอัพเดทใหม่)
# .NET: .NET Framework 4.x

# PowerShell 7+ (ต้องติดตั้งเอง)
# Path: C:\Program Files\PowerShell\7\pwsh.exe
# Version: 7.x (active development)
# .NET: .NET 6/7/8
```

### ทำไมต้องใช้ PowerShell 7?

| Feature | Windows PS 5.1 | PowerShell 7 |
|---------|---------------|-------------|
| Cross-platform | ❌ | ✅ |
| Ternary operator | ❌ | ✅ |
| Null coalescing (??) | ❌ | ✅ |
| Pipeline chain (&&, \|\|) | ❌ | ✅ |
| ForEach-Object -Parallel | ❌ | ✅ |
| SSH Remoting | ❌ | ✅ |
| Simplified Where-Object | Limited | ✅ |
| Updated .NET | .NET 4.x | .NET 8 |
| Performance | Baseline | **Faster** |

```powershell
# ตรวจสอบว่าใช้ version ไหน
$PSVersionTable.PSVersion
# หรือ
pwsh --version   # PowerShell 7
powershell --version  # Windows PowerShell
```

---

## 2. ติดตั้ง PowerShell 7 บน Windows

### วิธีที่ 1: Microsoft Store (ง่ายที่สุด)

```
1. เปิด Microsoft Store
2. ค้นหา "PowerShell"
3. คลิก Install
4. รอจนติดตั้งเสร็จ
```

### วิธีที่ 2: winget (แนะนำ)

```powershell
# เปิด CMD หรือ PowerShell แล้วรัน:
winget install Microsoft.PowerShell

# อัพเดท
winget upgrade Microsoft.PowerShell

# ดูเวอร์ชัน preview
winget install Microsoft.PowerShell.Preview
```

### วิธีที่ 3: Chocolatey

```powershell
# ต้องติดตั้ง Chocolatey ก่อน
choco install powershell-core
choco upgrade powershell-core
```

### วิธีที่ 4: Manual Download

```powershell
# ดาวน์โหลดจาก GitHub releases:
# https://github.com/PowerShell/PowerShell/releases
# เลือก PowerShell-7.x.x-win-x64.msi

# หรือใช้ script อัตโนมัติ
iex "& { $(irm https://aka.ms/install-powershell.ps1) } -UseMSI"
```

### วิธีที่ 5: Scoop

```powershell
# ติดตั้ง scoop ก่อน (ถ้ายังไม่มี)
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression

# ติดตั้ง PowerShell
scoop install pwsh
```

### ตรวจสอบการติดตั้ง

```powershell
# เปิด PowerShell 7 (pwsh)
pwsh

# ตรวจสอบ
$PSVersionTable
# PSVersion ควรเป็น 7.x.x

# ดู path
(Get-Command pwsh).Source
# C:\Program Files\PowerShell\7\pwsh.exe
```

---

## 3. ติดตั้งบน Linux

### Ubuntu/Debian

```bash
# Ubuntu 22.04
sudo apt-get update
sudo apt-get install -y wget apt-transport-https software-properties-common

# นำเข้า GPG key
wget -q "https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb"
sudo dpkg -i packages-microsoft-prod.deb

# ติดตั้ง
sudo apt-get update
sudo apt-get install -y powershell

# เริ่ม PowerShell
pwsh
```

```bash
# Ubuntu/Debian - วิธีที่ 2 (snap)
sudo snap install powershell --classic
```

### CentOS/RHEL/Fedora

```bash
# RHEL/CentOS 8
sudo rpm -Uvh https://packages.microsoft.com/config/rhel/8/packages-microsoft-prod.rpm
sudo dnf install -y powershell

# Fedora 37+
sudo dnf install -y powershell
```

### Alpine Linux

```bash
sudo apk add powershell
```

### Universal Method (Snap)

```bash
sudo snap install powershell --classic
pwsh
```

---

## 4. ติดตั้งบน macOS

### Homebrew (แนะนำ)

```bash
# ติดตั้ง Homebrew ก่อน (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง PowerShell
brew install --cask powershell

# อัพเดท
brew upgrade --cask powershell

# เริ่มใช้
pwsh
```

### Direct Download

```
1. ไปที่ https://github.com/PowerShell/PowerShell/releases
2. ดาวน์โหลด .pkg file สำหรับ macOS
3. ดับเบิ้ลคลิกติดตั้ง
```

---

## 5. VS Code Setup

### ติดตั้ง VS Code

```powershell
# Windows
winget install Microsoft.VisualStudioCode

# หรือดาวน์โหลดจาก https://code.visualstudio.com/
```

### ติดตั้ง PowerShell Extension

```
1. เปิด VS Code
2. กด Ctrl+Shift+X (Extensions)
3. ค้นหา "PowerShell"
4. ติดตั้ง "PowerShell" by Microsoft
5. Reload VS Code
```

```powershell
# หรือใช้ command line
code --install-extension ms-vscode.PowerShell
```

### VS Code Settings สำหรับ PowerShell

```json
// settings.json (Ctrl+Shift+P → "Open Settings (JSON)")
{
    "[powershell]": {
        "editor.tabSize": 4,
        "editor.renderWhitespace": "all",
        "files.encoding": "utf8bom"
    },
    "powershell.codeFormatting.preset": "Stroustrup",
    "powershell.scriptAnalysis.enable": true,
    "powershell.integratedConsole.showOnStartup": false,
    "powershell.powerShellDefaultVersion": "PowerShell (x64)",
    "editor.formatOnSave": true,
    "files.autoSave": "afterDelay",
    "terminal.integrated.defaultProfile.windows": "PowerShell"
}
```

### Keyboard Shortcuts สำคัญ

| Shortcut | Action |
|----------|--------|
| `F5` | Run current script |
| `F8` | Run selected code |
| `Ctrl+F5` | Run without debugging |
| `F9` | Toggle breakpoint |
| `F10` | Step over |
| `F11` | Step into |
| `Shift+F11` | Step out |
| `Ctrl+Space` | Autocomplete |
| `Ctrl+Shift+P` | Command Palette |

### Extensions เพิ่มเติมที่แนะนำ

```
- GitLens (git integration)
- indent-rainbow (indent visualization)
- Bracket Pair Colorizer 2
- Error Lens
- Thunder Client (REST API testing)
```

---

## 6. Execution Policy

**Execution Policy** ควบคุมว่า script ใดที่สามารถ run ได้

### Policy Levels

```powershell
# ดู policy ปัจจุบัน
Get-ExecutionPolicy
Get-ExecutionPolicy -List  # ดูทุก scope
```

| Policy | คำอธิบาย |
|--------|----------|
| `Restricted` | ไม่ให้ run scripts เลย (default บน client Windows) |
| `AllSigned` | ต้อง sign ทุก script |
| `RemoteSigned` | Scripts ที่ดาวน์โหลดต้อง sign, local ไม่ต้อง |
| `Unrestricted` | Run ได้ทั้งหมด (เตือนสำหรับ internet scripts) |
| `Bypass` | ไม่มีการ block เลย |
| `Undefined` | ใช้ policy ของ scope บน |

```powershell
# ดู policy ของแต่ละ scope
Get-ExecutionPolicy -List
# Scope           ExecutionPolicy
# -----           ---------------
# MachinePolicy   Undefined
# UserPolicy      Undefined
# Process         Undefined
# CurrentUser     Undefined
# LocalMachine    Restricted

# ตั้งค่า (แนะนำสำหรับ dev)
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser

# สำหรับ LocalMachine (ต้อง Admin)
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope LocalMachine

# Bypass เฉพาะ current session
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process

# Run script แบบ bypass ชั่วคราว
pwsh -ExecutionPolicy Bypass -File myscript.ps1
```

### Unblock Script ที่ดาวน์โหลดมา

```powershell
# ดู security zone ของไฟล์
Get-Item 'downloaded-script.ps1' | Get-ItemProperty -Name Zone.Identifier

# Unblock ไฟล์
Unblock-File 'downloaded-script.ps1'

# Unblock ทั้ง folder
Get-ChildItem 'C:\Scripts' -Recurse | Unblock-File
```

---

## 7. PowerShell Profile

**Profile** คือ script ที่ run อัตโนมัติทุกครั้งที่เปิด PowerShell

### ดู Profile Paths

```powershell
# ดู profile paths ทั้งหมด
$PROFILE | Get-Member -MemberType NoteProperty | 
    Select-Object Name, @{N='Path';E={$PROFILE.$($_.Name)}}

# Profiles:
# AllUsersAllHosts       - C:\Program Files\PowerShell\7\profile.ps1
# AllUsersCurrentHost    - C:\Program Files\PowerShell\7\Microsoft.PowerShell_profile.ps1
# CurrentUserAllHosts    - ~\Documents\PowerShell\profile.ps1
# CurrentUserCurrentHost - ~\Documents\PowerShell\Microsoft.PowerShell_profile.ps1

# Profile ที่ใช้บ่อยที่สุด
$PROFILE  # CurrentUserCurrentHost
```

### สร้างและแก้ไข Profile

```powershell
# ตรวจสอบว่ามี profile หรือยัง
Test-Path $PROFILE

# สร้าง profile ถ้ายังไม่มี
if (!(Test-Path $PROFILE)) {
    New-Item -Path $PROFILE -ItemType File -Force
}

# เปิดแก้ไขใน VS Code
code $PROFILE

# หรือใน notepad
notepad $PROFILE
```

### ตัวอย่าง Profile ที่มีประโยชน์

```powershell
# ===== PowerShell Profile =====
# บันทึกใน: $PROFILE

# --- Prompt Customization ---
function prompt {
    $path = $executionContext.SessionState.Path.CurrentLocation.Path
    $host.UI.RawUI.WindowTitle = "PS: $path"
    
    Write-Host "[" -NoNewline -ForegroundColor DarkGray
    Write-Host (Get-Date -Format 'HH:mm:ss') -NoNewline -ForegroundColor DarkCyan
    Write-Host "]" -NoNewline -ForegroundColor DarkGray
    Write-Host " $path" -NoNewline -ForegroundColor Cyan
    Write-Host " >" -NoNewline -ForegroundColor Yellow
    return " "
}

# --- Aliases ---
Set-Alias -Name g -Value git
Set-Alias -Name k -Value kubectl
Set-Alias -Name tf -Value terraform
Set-Alias -Name np -Value notepad

# --- Useful Functions ---
function which ($cmd) { (Get-Command $cmd).Source }
function touch ($file) { New-Item -ItemType File -Path $file -Force }
function ll { Get-ChildItem -Force @args }
function la { Get-ChildItem -Force -Hidden @args }

# Go to common directories
function docs { Set-Location "$env:USERPROFILE\Documents" }
function desk { Set-Location "$env:USERPROFILE\Desktop" }
function down { Set-Location "$env:USERPROFILE\Downloads" }

# Git shortcuts
function gs { git status }
function ga { git add @args }
function gc { git commit -m @args }
function gp { git push @args }
function gl { git log --oneline -20 }
function gd { git diff @args }

# --- Auto-complete improvements ---
Set-PSReadLineKeyHandler -Key Tab -Function MenuComplete
Set-PSReadLineOption -PredictionSource History
Set-PSReadLineOption -PredictionViewStyle ListView

# --- Import modules ---
# Import-Module posh-git         # git prompt (ถ้าติดตั้ง)
# Import-Module oh-my-posh       # fancy prompt
# Import-Module PSReadLine       # already built-in PS 7

# --- Environment ---
$env:EDITOR = 'code'

# --- Welcome message ---
Write-Host "PowerShell $($PSVersionTable.PSVersion) | $(Get-Date -Format 'yyyy-MM-dd')" -ForegroundColor DarkCyan
```

### โหลด Profile ใหม่

```powershell
# โหลด profile โดยไม่ต้องเปิด terminal ใหม่
. $PROFILE

# หรือใช้ dot-source
. "$env:USERPROFILE\Documents\PowerShell\Microsoft.PowerShell_profile.ps1"
```

---

## 8. PowerShell Gallery & Modules

### PowerShell Gallery คืออะไร?

PowerShell Gallery ([PowerShellGallery.com](https://www.powershellgallery.com)) คือ central repository สำหรับ PowerShell modules และ scripts ทั้งจาก Microsoft และ community

### Module Management

```powershell
# ค้นหา module
Find-Module *azure*
Find-Module -Tag 'Security'
Find-Module PowerCLI  # VMware module

# ดูรายละเอียด module
Find-Module ImportExcel | Select-Object *

# ติดตั้ง module
Install-Module ImportExcel                  # ยืนยัน
Install-Module ImportExcel -Force           # บังคับ
Install-Module ImportExcel -Scope CurrentUser  # ติดตั้งเฉพาะ user ปัจจุบัน
Install-Module ImportExcel -AllowPrerelease    # รวม pre-release

# ดู modules ที่ติดตั้งแล้ว
Get-Module -ListAvailable
Get-Module -ListAvailable | Where-Object Name -like '*import*'
Get-InstalledModule

# โหลด module
Import-Module ImportExcel
Import-Module ImportExcel -Force  # reload

# ดู module ที่ loaded แล้ว
Get-Module

# อัพเดท module
Update-Module ImportExcel
Update-Module  # อัพเดททั้งหมด

# ถอนการติดตั้ง
Uninstall-Module ImportExcel

# ดู commands ใน module
Get-Command -Module ImportExcel
```

### Modules ที่แนะนำให้ติดตั้ง

```powershell
# สำหรับ development
Install-Module -Name PSReadLine -Force -AllowPrerelease  # better autocomplete
Install-Module -Name posh-git                            # git integration
Install-Module -Name oh-my-posh                          # fancy prompt
Install-Module -Name Terminal-Icons                      # file icons in terminal

# สำหรับการทำงาน
Install-Module -Name ImportExcel          # Excel without Excel
Install-Module -Name dbatools            # SQL Server management
Install-Module -Name PSWindowsUpdate     # Windows Update management

# สำหรับ Security
Install-Module -Name PowerSploit         # Security testing (authorized)
Install-Module -Name Pester              # Unit testing
Install-Module -Name PSScriptAnalyzer    # Code quality

# สำหรับ Cloud
Install-Module -Name Az                  # Azure
Install-Module -Name AWSPowerShell       # AWS
Install-Module -Name GoogleCloud         # GCP
```

### ตั้งค่า PSGallery

```powershell
# ตรวจสอบ repository
Get-PSRepository

# ตั้งให้ PSGallery trusted
Set-PSRepository -Name PSGallery -InstallationPolicy Trusted

# เพิ่ม custom repository
Register-PSRepository -Name 'MyRepo' `
    -SourceLocation 'https://myserver/nuget/powershell' `
    -InstallationPolicy Trusted
```

---

## 9. Environment Variables

### อ่าน Environment Variables

```powershell
# ดู env vars ทั้งหมด
Get-ChildItem Env:
ls Env:          # alias

# ดูค่าเฉพาะตัว
$env:PATH
$env:USERPROFILE
$env:USERNAME
$env:COMPUTERNAME
$env:SystemRoot
$env:TEMP
$env:ProgramFiles
$env:APPDATA

# ค้นหา
$env:PATH -split ';'   # แสดง PATH entries แยก
Get-ChildItem Env: | Where-Object Name -like '*user*'
```

### ตั้งค่า Environment Variables

```powershell
# ตั้งค่าชั่วคราว (เฉพาะ session นี้)
$env:MY_VAR = "Hello"
$env:API_KEY = "secret123"

# ตั้งค่าถาวร (User level)
[System.Environment]::SetEnvironmentVariable('MY_VAR', 'Hello', 'User')

# ตั้งค่าถาวร (Machine level - ต้อง Admin)
[System.Environment]::SetEnvironmentVariable('MY_VAR', 'Hello', 'Machine')

# อ่านค่าถาวร
[System.Environment]::GetEnvironmentVariable('MY_VAR', 'User')
[System.Environment]::GetEnvironmentVariable('MY_VAR', 'Machine')

# ลบ env var
Remove-Item Env:MY_VAR
[System.Environment]::SetEnvironmentVariable('MY_VAR', $null, 'User')
```

### PATH Management

```powershell
# ดู PATH
$env:PATH -split ';'

# เพิ่ม path ชั่วคราว
$env:PATH += ";C:\MyTools"

# เพิ่ม path ถาวร
$userPath = [System.Environment]::GetEnvironmentVariable('PATH', 'User')
$newPath  = $userPath + ";C:\MyTools"
[System.Environment]::SetEnvironmentVariable('PATH', $newPath, 'User')

# ฟังก์ชัน helper สำหรับเพิ่ม PATH
function Add-ToPath {
    param([string]$NewPath, [string]$Scope = 'User')
    $current = [System.Environment]::GetEnvironmentVariable('PATH', $Scope)
    if ($current -notlike "*$NewPath*") {
        [System.Environment]::SetEnvironmentVariable('PATH', "$current;$NewPath", $Scope)
        Write-Host "Added '$NewPath' to PATH ($Scope)" -ForegroundColor Green
    } else {
        Write-Host "'$NewPath' already in PATH" -ForegroundColor Yellow
    }
}

Add-ToPath 'C:\MyTools'
Add-ToPath 'C:\Scripts'
```

---

## 10. Exercises

### Exercise 1: Version Check

```powershell
# ตรวจสอบ PowerShell environment
Write-Host "=== PowerShell Environment ==="
Write-Host "Version: $($PSVersionTable.PSVersion)"
Write-Host "Edition: $($PSVersionTable.PSEdition)"
Write-Host "OS: $($PSVersionTable.OS)"
Write-Host "Platform: $($PSVersionTable.Platform)"
Write-Host "Path: $((Get-Command pwsh -ErrorAction SilentlyContinue)?.Source)"
Write-Host ""
Write-Host "=== Execution Policy ==="
Get-ExecutionPolicy -List
```

### Exercise 2: Setup Profile

```powershell
# สร้าง profile ถ้ายังไม่มี
if (!(Test-Path $PROFILE)) {
    New-Item -Path $PROFILE -ItemType File -Force
    Write-Host "Created profile at: $PROFILE" -ForegroundColor Green
} else {
    Write-Host "Profile exists at: $PROFILE" -ForegroundColor Yellow
}

# เปิดใน VS Code
code $PROFILE

# เพิ่ม aliases พื้นฐาน
$profileContent = @'
# Custom Aliases
function which ($cmd) { (Get-Command $cmd -ErrorAction SilentlyContinue)?.Source }
function touch ($file) { New-Item -ItemType File -Path $file -Force | Out-Null }
function .. { Set-Location .. }
function ... { Set-Location ../.. }

# Improve autocomplete
Set-PSReadLineKeyHandler -Key Tab -Function MenuComplete
'@

Add-Content $PROFILE $profileContent
Write-Host "Added functions to profile" -ForegroundColor Green

# โหลด profile
. $PROFILE
```

### Exercise 3: Install Useful Modules

```powershell
# ตรวจสอบ modules ที่มีอยู่
Get-InstalledModule | Select-Object Name, Version | Format-Table -AutoSize

# ติดตั้ง modules ที่แนะนำ
$modules = @('PSReadLine', 'Pester', 'PSScriptAnalyzer')

foreach ($mod in $modules) {
    if (!(Get-InstalledModule $mod -ErrorAction SilentlyContinue)) {
        Write-Host "Installing $mod..." -ForegroundColor Cyan
        Install-Module $mod -Force -AllowClobber -Scope CurrentUser
        Write-Host "Installed $mod" -ForegroundColor Green
    } else {
        Write-Host "$mod already installed" -ForegroundColor Yellow
    }
}
```

### Exercise 4: Environment Variables

```powershell
# สร้าง environment variables สำหรับ development

# Dev paths
$devPaths = @{
    'PROJECTS_HOME' = "$env:USERPROFILE\Projects"
    'SCRIPTS_HOME'  = "$env:USERPROFILE\Scripts"
    'LOGS_HOME'     = "$env:USERPROFILE\Logs"
}

foreach ($var in $devPaths.GetEnumerator()) {
    # ตั้ง env var
    [System.Environment]::SetEnvironmentVariable($var.Key, $var.Value, 'User')
    
    # สร้าง folder ถ้ายังไม่มี
    if (!(Test-Path $var.Value)) {
        New-Item -Path $var.Value -ItemType Directory -Force | Out-Null
    }
    
    Write-Host "Set $($var.Key) = $($var.Value)" -ForegroundColor Green
}

# โหลด env vars ใหม่
$env:PROJECTS_HOME = [System.Environment]::GetEnvironmentVariable('PROJECTS_HOME', 'User')
Write-Host "PROJECTS_HOME = $env:PROJECTS_HOME"
```

### Exercise 5: Complete Setup Script

```powershell
# setup.ps1 - สคริปต์ตั้งค่า development environment อัตโนมัติ

#Requires -Version 7.0
[CmdletBinding()]
param()

function Write-Step {
    param([string]$Message, [string]$Status = 'Info')
    $colors = @{ Info = 'Cyan'; Success = 'Green'; Warning = 'Yellow'; Error = 'Red' }
    Write-Host "[$Status] $Message" -ForegroundColor $colors[$Status]
}

Write-Step "Starting PowerShell Development Environment Setup" 'Info'
Write-Host ("=" * 60)

# 1. Check PowerShell version
if ($PSVersionTable.PSVersion.Major -lt 7) {
    Write-Step "PowerShell 7+ required. Current: $($PSVersionTable.PSVersion)" 'Error'
    exit 1
}
Write-Step "PowerShell $($PSVersionTable.PSVersion) detected" 'Success'

# 2. Set Execution Policy
try {
    Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser -Force
    Write-Step "Execution Policy set to RemoteSigned" 'Success'
} catch {
    Write-Step "Could not set Execution Policy: $_" 'Warning'
}

# 3. Set PSGallery as trusted
Set-PSRepository -Name PSGallery -InstallationPolicy Trusted
Write-Step "PSGallery set as trusted" 'Success'

# 4. Install essential modules
$essentialModules = @(
    'PSReadLine'
    'Pester'
    'PSScriptAnalyzer'
    'ImportExcel'
)

foreach ($mod in $essentialModules) {
    try {
        if (!(Get-InstalledModule $mod -ErrorAction SilentlyContinue)) {
            Install-Module $mod -Force -AllowClobber -Scope CurrentUser -ErrorAction Stop
            Write-Step "Installed: $mod" 'Success'
        } else {
            Write-Step "Already installed: $mod" 'Info'
        }
    } catch {
        Write-Step "Failed to install $mod: $_" 'Warning'
    }
}

# 5. Create directory structure
$dirs = @(
    "$env:USERPROFILE\Projects"
    "$env:USERPROFILE\Scripts"
    "$env:USERPROFILE\Logs"
    "$env:USERPROFILE\Tools"
)

foreach ($dir in $dirs) {
    if (!(Test-Path $dir)) {
        New-Item -Path $dir -ItemType Directory -Force | Out-Null
        Write-Step "Created directory: $dir" 'Success'
    }
}

# 6. Create/Update Profile
if (!(Test-Path $PROFILE)) {
    New-Item -Path $PROFILE -ItemType File -Force | Out-Null
    Write-Step "Created profile: $PROFILE" 'Success'
} else {
    Write-Step "Profile exists: $PROFILE" 'Info'
}

Write-Host ("=" * 60)
Write-Step "Setup Complete! Restart your terminal." 'Success'
```

---

## 📝 สรุป Part 02

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Windows PS vs PS 7 | 5.1 legacy, 7.x active |
| ติดตั้ง | winget, chocolatey, snap, direct |
| VS Code | Extension + settings |
| Execution Policy | RemoteSigned แนะนำสำหรับ dev |
| Profile | รัน auto ทุกครั้งที่เปิด PS |
| Modules | Install จาก PSGallery |
| Env Vars | $env:, SetEnvironmentVariable |

---

**ก่อนหน้า ← [Part 01: Introduction](Part-01.md)**  
**ต่อไป → [Part 03: Variables & Data Types](Part-03.md)**
