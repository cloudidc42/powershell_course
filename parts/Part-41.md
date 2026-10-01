# Part 41: Windows Security Architecture

> **ระดับ**: 🔴 Professional/Security | **เวลา**: ~5 ชั่วขนึ่ง

> ⚠️ **คำเตือน**: เนื้อหานี้ใช้เพื่อการศึกษา, การทดสอบ CTF, การตรวจสอบความปลอดภัยที่ได้รับอนุญาต, และ Blue Team/Defensive Security **เท่านั้น** ห้ามใช้โดยไม่ได้รับอนุญาต

---

## 1. Windows Security Layers

```
Windows Security Architecture:

+----------------------------------+
|  User Applications               |
+----------------------------------+
|  Win32 API / .NET CLR            |
+----------------------------------+
|  Windows Subsystem (ntdll.dll)   |
+----------------------------------+
|  Kernel (ntoskrnl.exe)          |
|  - Security Reference Monitor    |
|  - Object Manager                |
|  - I/O Manager                   |
+----------------------------------+
|  Hardware (CPU rings 0-3)        |
+----------------------------------+

Security Components:
- LSA (Local Security Authority)     - lsass.exe
- SAM (Security Account Manager)     - credentials
- AMSI (Antimalware Scan Interface)  - script scanning
- PPL (Protected Process Light)      - process isolation
- Credential Guard (VBS)             - LSASS isolation
- ETW (Event Tracing for Windows)    - telemetry
```

---

## 2. Token และ Privilege

```powershell
# ดู token ของตัวเอง
function Get-CurrentTokenInfo {
    $id = [System.Security.Principal.WindowsIdentity]::GetCurrent()
    $principal = [System.Security.Principal.WindowsPrincipal]::new($id)
    
    [PSCustomObject]@{
        Name           = $id.Name
        SID            = $id.User.Value
        AuthType       = $id.AuthenticationType
        IsAdmin        = $principal.IsInRole([System.Security.Principal.WindowsBuiltInRole]::Administrator)
        IsSystem       = $id.IsSystem
        Groups         = ($id.Groups | Get-SIDFriendlyName)
        ImpersonationLevel = $id.ImpersonationLevel
    }
}

function Get-SIDFriendlyName {
    param([System.Security.Principal.SecurityIdentifier[]]$SIDs)
    $SIDs | ForEach-Object {
        try { $_.Translate([System.Security.Principal.NTAccount]).Value }
        catch { $_.Value }
    }
}

Get-CurrentTokenInfo

# Check privileges ด้วย whoami
& whoami /priv
& whoami /groups

# whoami ด้วย PS
[System.Security.Principal.WindowsIdentity]::GetCurrent() | Format-List
```

---

## 3. Access Control (ACL)

```powershell
# ดู NTFS permissions
$acl = Get-Acl 'C:\Sensitive\data.txt'
$acl.Access | Format-Table FileSystemRights, IdentityReference, AccessControlType

# เพิ่ม permission
$acl = Get-Acl 'C:\App'
$rule = [System.Security.AccessControl.FileSystemAccessRule]::new(
    'DOMAIN\AppUser',
    'ReadAndExecute',
    'ContainerInherit,ObjectInherit',
    'None',
    'Allow'
)
$acl.AddAccessRule($rule)
Set-Acl 'C:\App' -AclObject $acl

# Take ownership
$acl = Get-Acl 'C:\Lockout'
$acl.SetOwner([System.Security.Principal.NTAccount]'BUILTIN\Administrators')
Set-Acl 'C:\Lockout' -AclObject $acl

# ตรวจสอป misconfigured permissions (audit)
Get-ChildItem 'C:\Program Files' -Recurse -ErrorAction SilentlyContinue |
    Get-Acl -ErrorAction SilentlyContinue |
    ForEach-Object {
        $path = $_.Path
        $_.Access | Where-Object {
            $_.IdentityReference -like '*Users*' -and
            $_.FileSystemRights -match 'Write|Modify|FullControl'
        } | ForEach-Object {
            [PSCustomObject]@{
                Path   = $path
                Rights = $_.FileSystemRights
                User   = $_.IdentityReference
            }
        }
    } | Format-Table -AutoSize
```

---

## 4. โครงสร้าง Authentication

```powershell
# NTLM vs Kerberos
# NTLM: challenge-response, ใช้ LSASS, susceptible to Pass-the-Hash
# Kerberos: ticket-based, ใช้ KDC, ดีกว่า NTLM

# ดู Kerberos tickets
& klist.exe
& klist tgt

# PowerShell credentials
$cred = Get-Credential  # ป้อน username/password

# SecureString พื้นฐาน
$secure = ConvertTo-SecureString 'P@ssw0rd' -AsPlainText -Force
$plain  = [System.Net.NetworkCredential]::new('', $secure).Password

# Windows Credential Manager
$cred = [System.Net.CredentialCache]::DefaultCredentials

# เสริม SPN (Service Principal Name) ดู
& setspn.exe -L 'computer_name'
```

---

## 5. Audit Policy

```powershell
# ดู audit policies
& auditpol.exe /get /category:*

# ตั้งค่า audit
& auditpol.exe /set /subcategory:"Logon" /success:enable /failure:enable
& auditpol.exe /set /subcategory:"Process Creation" /success:enable

# ค้นหาใน Security Event Log
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -MaxEvents 20 |
    ForEach-Object {
        $xml = [xml]$_.ToXml()
        $data = $xml.Event.EventData.Data
        [PSCustomObject]@{
            Time          = $_.TimeCreated
            AccountName   = ($data | Where-Object Name -eq 'TargetUserName').'#text'
            LogonType     = ($data | Where-Object Name -eq 'LogonType').'#text'
            WorkStation   = ($data | Where-Object Name -eq 'WorkstationName').'#text'
            SourceIP      = ($data | Where-Object Name -eq 'IpAddress').'#text'
        }
    } | Format-Table -AutoSize

# ID 4625 = Failed logon (brute-force detection)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 100 |
    Group-Object { ([xml]$_.ToXml()).Event.EventData.Data | Where-Object Name -eq 'IpAddress' | Select-Object -Expand '#text' } |
    Where-Object Count -gt 5 |
    Sort-Object Count -Descending
```

---

**ก่อนหน้า ← [Part 40](Part-40.md) | ต่อไป → [Part 42: AMSI & Script Security](Part-42.md)**
