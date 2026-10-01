# Part 15: Modules และ Script Organization

> **ระดับ**: 🟡 Intermediate | **เวลา**: ~3 ชั่วโมง

---

## 1. Script Modules (.psm1)

```powershell
# MyUtils.psm1

function Get-Greeting {
    [CmdletBinding()]
    param([string]$Name = 'World')
    "Hello, $Name!"
}

function Get-RandomPassword {
    [CmdletBinding()]
    param(
        [int]$Length = 16,
        [switch]$IncludeSymbols
    )
    
    $chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789'
    if ($IncludeSymbols) { $chars += '!@#$%^&*()_+-=[]{}|;:,.<>?' }
    
    $rng = [System.Security.Cryptography.RandomNumberGenerator]::Create()
    $bytes = New-Object byte[] $Length
    $rng.GetBytes($bytes)
    
    $password = -join ($bytes | ForEach-Object { $chars[$_ % $chars.Length] })
    return $password
}

# Private function (not exported)
function Private-Helper {
    param($x)
    $x * 2
}

# Export only public functions
Export-ModuleMember -Function 'Get-Greeting', 'Get-RandomPassword'
```

---

## 2. Module Manifest (.psd1)

```powershell
# MyUtils.psd1
@{
    # Module version
    ModuleVersion   = '1.0.0'
    GUID            = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
    Author          = 'Your Name'
    CompanyName     = 'Your Company'
    Description     = 'PowerShell utility functions'
    
    # PowerShell version requirement
    PowerShellVersion = '5.1'
    
    # Root module
    RootModule      = 'MyUtils.psm1'
    
    # Exported functions (can use wildcards)
    FunctionsToExport = @('Get-Greeting', 'Get-RandomPassword')
    CmdletsToExport   = @()
    VariablesToExport = @()
    AliasesToExport   = @()
    
    # Dependencies
    RequiredModules = @(
        @{ ModuleName='PSReadLine'; ModuleVersion='2.0' }
    )
    
    # Private data / metadata
    PrivateData = @{
        PSData = @{
            Tags        = @('Utility', 'Security', 'Tools')
            ProjectUri  = 'https://github.com/username/MyUtils'
            LicenseUri  = 'https://opensource.org/licenses/MIT'
            ReleaseNotes = 'Initial release'
        }
    }
}
```

---

## 3. Module Directory Structure

```
MyUtils/
├── MyUtils.psd1          # manifest
├── MyUtils.psm1          # root module
├── Public/
│   ├── Get-Greeting.ps1
│   ├── Get-RandomPassword.ps1
│   └── Invoke-Task.ps1
├── Private/
│   ├── Helper.ps1
│   └── Validator.ps1
├── Classes/
│   └── Config.ps1
├── Tests/
│   └── MyUtils.Tests.ps1
└── en-US/
    └── MyUtils-help.xml
```

```powershell
# MyUtils.psm1 - auto-load all files
$Public  = @(Get-ChildItem -Path $PSScriptRoot\Public\*.ps1)
$Private = @(Get-ChildItem -Path $PSScriptRoot\Private\*.ps1)

# โหลดทุก file
foreach ($file in @($Public + $Private)) {
    try {
        . $file.FullName
    } catch {
        Write-Error "Failed to import $($file.FullName): $_"
    }
}

# Export public functions
Export-ModuleMember -Function $Public.BaseName
```

---

## 4. ติดตั้ง/ใช้ Module

```powershell
# ติดตั้งจาก PSGallery
Find-Module PSReadLine
Find-Module -Tag 'Security'
Find-Module -Name 'Pester' -AllVersions

Install-Module PSReadLine -Scope CurrentUser
Install-Module PSReadLine -Scope AllUsers   # ต้อง admin
Install-Module PSReadLine -AllowPrerelease
Install-Module PSReadLine -RequiredVersion 2.2.6

# Update
Update-Module PSReadLine
Update-Module  # อัพเดตทุกอย่าง

# ทดสอบ
Get-Module                          # loaded modules
Get-Module -ListAvailable           # installed modules
Get-Module -ListAvailable PSReadLine

# โหลด module
Import-Module MyUtils
Import-Module MyUtils -Force        # reload
Import-Module MyUtils -Verbose

# ไม่โหลด
Remove-Module MyUtils

# Module path
$env:PSModulePath -split ';'        # Windows
$env:PSModulePath -split ':'        # Linux/Mac
```

---

## 5. Script Scoping และ Dot-Sourcing

```powershell
# Dot-sourcing: เรียกใช้ใน scope ปัจจุบัน
. '.\helpers.ps1'       # load into current scope
. '.\config.ps1'        # load variables and functions

# & (call operator): เรียกใน child scope
& '.\script.ps1'        # child scope - vars don't leak
& '.\script.ps1' -Param1 'value'

# ความแตกต่าง:
# helpers.ps1 มี: function Foo { ... }
. '.\helpers.ps1'
Foo   # ใช้ได้เพราะ dot-sourced

& '.\helpers.ps1'
# Foo  # Error! ไม่มี function Foo ใน scope นี้

# Scoped variables
$Global:AppName = 'MyApp'   # global
$Script:Counter = 0         # script-level
$Local:Temp = 'value'       # current scope (default)
$Private:Secret = 'hidden'  # can't be seen in child
```

---

## 6. Publish ไป PSGallery

```powershell
# สร้าง manifest
New-ModuleManifest -Path '.\MyUtils.psd1' \
    -RootModule 'MyUtils.psm1' \
    -ModuleVersion '1.0.0' \
    -Author 'Your Name' \
    -Description 'My utility module'

# Test module
Test-ModuleManifest '.\MyUtils.psd1'

# Analyze ด้วย PSScriptAnalyzer
Invoke-ScriptAnalyzer '.\MyUtils.psm1'
Invoke-ScriptAnalyzer '.\' -Recurse

# Publish (need API key from powershellgallery.com)
# $apiKey = 'your-api-key'
# Publish-Module -Path '.\MyUtils' -NuGetApiKey $apiKey
# Publish-Module -Name 'MyUtils' -NuGetApiKey $apiKey

# Publish ไป local repository
Register-PSRepository -Name 'LocalRepo' \
    -SourceLocation 'C:\LocalModules' \
    -InstallationPolicy Trusted

Publish-Module -Path '.\MyUtils' -Repository 'LocalRepo'
Install-Module 'MyUtils' -Repository 'LocalRepo'
```

---

## 7. Module Best Practices

```powershell
# 1. ใช้ Verb-Noun naming
function Get-Data    { }
function Set-Config  { }
function New-Report  { }

# 2. ใช้ approved verbs
Get-Verb | Sort-Object Verb  # list approved verbs

# 3. เช็คสอบ module ก่อน import
function Import-RequiredModules {
    param([string[]]$Modules)
    
    foreach ($module in $Modules) {
        if (!(Get-Module -Name $module -ErrorAction SilentlyContinue)) {
            if (Get-Module -ListAvailable -Name $module) {
                Import-Module $module
            } else {
                Write-Warning "Module '$module' not found. Installing..."
                Install-Module $module -Scope CurrentUser -Force
                Import-Module $module
            }
        }
    }
}

Import-RequiredModules @('PSReadLine', 'Pester', 'ImportExcel')

# 4. ใช้ #Requires ที่ต้น script
#Requires -Version 5.1
#Requires -Modules @{ ModuleName='PSReadLine'; ModuleVersion='2.0' }
#Requires -RunAsAdministrator
```

---

**ก่อนหน้า ← [Part 14](Part-14.md) | ต่อไป → [Part 16: Classes & OOP](Part-16.md)**
