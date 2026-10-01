# Part 37: PowerShell Providers และ PSDrives

> **ระดับ**: 🟠 Advanced | **เวลา**: ~3 ชั่วขนึ่ง

---

## 1. Built-in Providers

```powershell
# ดู providers ทั้งหมด
Get-PSProvider | Select-Object Name, Capabilities, Drives

# Providers:
# FileSystem  - C:, D: etc.
# Registry    - HKLM:, HKCU:
# Environment - Env:
# Variable    - Variable:
# Function    - Function:
# Alias       - Alias:
# Certificate - Cert:

# Browse Environment variables
Set-Location Env:
Get-ChildItem
$env:PATH
$env:COMPUTERNAME

# Browse Variables
Get-ChildItem Variable:     # all PS variables
(Get-Variable -Name 'PSVersionTable').Value

# Browse Functions
Get-ChildItem Function: | Select-Object Name | Sort-Object Name

# Browse Aliases
Get-ChildItem Alias: | Where-Object Definition -like 'Get-*' | Format-Table Name, Definition

# Certificates
Get-ChildItem Cert:\CurrentUser\My | Select-Object Subject, Thumbprint, NotAfter
Get-ChildItem Cert:\LocalMachine\Root | Where-Object { $_.NotAfter -lt (Get-Date).AddDays(30) }
```

---

## 2. Custom PSDrive

```powershell
# สร้าง PSDrive ใหม่
New-PSDrive -Name 'Projects' -PSProvider FileSystem -Root 'C:\Users\user\Projects'
Set-Location Projects:
Get-ChildItem

# Drive ชั่วคราว
# New-PSDrive ข้างต้นอยู่แค่ใน session
# -Persist เพื่อเพิ่มแสดงใน Explorer
New-PSDrive -Name 'Work' -PSProvider FileSystem -Root '\\server\share' -Credential (Get-Credential) -Persist

# ลบ drive
Remove-PSDrive -Name 'Projects'

# เพิ่ม drive ทุกครั้งที่เสด็จ ($PROFILE)
# New-PSDrive -Name 'Code' -PSProvider FileSystem -Root 'C:\Code'
```

---

## 3. Custom Provider

```powershell
# สร้าง Custom Provider (พื้นฐาน)
# Provider ต้องสร้างใน binary module (C# / .NET)
# ตัวอย่าง: สร้าง provider ไว้ใน memory แบบง่าย

$code = @'
using System.Collections.Generic;
using System.Management.Automation;
using System.Management.Automation.Provider;

[CmdletProvider("InMemory", ProviderCapabilities.None)]
public class InMemoryProvider : DriveCmdletProvider
{
    private static Dictionary<string, object> _store = new();
    
    protected override PSDriveInfo NewDrive(PSDriveInfo drive) => drive;
    protected override object NewDriveParameters()             => null;
}
'@

# Add-Type $code แล้วเรียกใช้ New-PSDrive -PSProvider InMemory

# แทนที่ ใช้ hashtable เลียนแบบ provider
$store = @{}
$store['config/app/host'] = 'localhost'
$store['config/app/port'] = 8080
$store['config/db/host']  = 'db.example.com'

# Navigation helper
function Get-StoreValue {
    param([string]$Path)
    $store[$Path]
}

function Set-StoreValue {
    param([string]$Path, $Value)
    $store[$Path] = $Value
}

function Get-StoreChildren {
    param([string]$Prefix = '')
    $store.Keys | Where-Object { $_ -like "$Prefix*" } |
        Group-Object { ($_ -replace "^$Prefix", '') -split '/' | Select-Object -First 1 } |
        Select-Object Name, Count
}

Get-StoreChildren 'config/'
Get-StoreValue 'config/app/port'
```

---

## 4. Certificate Management

```powershell
# Self-signed certificate
$cert = New-SelfSignedCertificate `
    -DnsName 'myapp.local', '*.myapp.local' `
    -CertStoreLocation 'Cert:\LocalMachine\My' `
    -NotAfter (Get-Date).AddYears(5) `
    -KeyLength 2048 `
    -KeyAlgorithm RSA

# Export PFX (with password)
$pw = ConvertTo-SecureString 'ExportPass123!' -AsPlainText -Force
Export-PfxCertificate -Cert $cert -FilePath 'C:\certs\myapp.pfx' -Password $pw

# Export CER (public only)
Export-Certificate -Cert $cert -FilePath 'C:\certs\myapp.cer' -Type CERT

# Import certificate
Import-PfxCertificate -FilePath 'C:\certs\myapp.pfx' `
    -CertStoreLocation 'Cert:\LocalMachine\My' `
    -Password $pw

# Find expiring certs
Get-ChildItem Cert:\LocalMachine\My |
    Where-Object { $_.NotAfter -lt (Get-Date).AddDays(90) } |
    Select-Object Subject, Thumbprint, NotAfter |
    Sort-Object NotAfter
```

---

**ก่อนหน้า ← [Part 36](Part-36.md) | ต่อไป → [Part 38: Binary Modules](Part-38.md)**
