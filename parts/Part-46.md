# Part 46: Privilege Escalation (CTF/Lab)

> **ระดับ**: 🔴 Professional/Security | **เวลา**: ~5 ชั่วขนึ่ง

> ⚠️ **สำหรับ**: CTF และ Authorized Lab Only — ห้ามใช้กับระบบจริงโดยเด็ดขาด

---

## 1. สิ่งที่ต้องตรวจสอบ (Checklist)

```powershell
# Windows Privesc Checklist สำหรับ CTF

# 1. System Info
[PSCustomObject]@{
    OS        = (Get-CimInstance Win32_OperatingSystem).Caption
    Version   = (Get-CimInstance Win32_OperatingSystem).Version
    Arch      = [System.Environment]::Is64BitOperatingSystem
    IsAdmin   = ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole('Administrator')
    User      = $env:USERNAME
    Domain    = $env:USERDOMAIN
    CLM       = $ExecutionContext.SessionState.LanguageMode
} | Format-List

# 2. Privileges
& whoami /priv

# Key privileges for escalation:
# SeImpersonatePrivilege  --> Potato attacks
# SeDebugPrivilege        --> process injection
# SeLoadDriverPrivilege   --> load kernel driver
# SeTakeOwnershipPrivilege --> take file ownership
# SeRestorePrivilege      --> write anywhere
# SeBackupPrivilege       --> read anywhere

# 3. Token privileges check
$id = [System.Security.Principal.WindowsIdentity]::GetCurrent()
$privs = & whoami /priv /fo csv | ConvertFrom-Csv
$privs | Where-Object { $_.'State' -eq 'Enabled' } | Format-Table 'Privilege Name', 'State'
```

---

## 2. Service Misconfiguration

```powershell
# หา services ที่มี weak permissions
function Find-WeakServicePermissions {
    $services = Get-WmiObject Win32_Service | Where-Object PathName
    
    foreach ($svc in $services) {
        # แยกประเภท path
        $path = $svc.PathName -replace '"', '' -split '\.exe' | Select-Object -First 1
        $path = ($path + '.exe').Trim()
        
        if (Test-Path $path -ErrorAction SilentlyContinue) {
            try {
                $acl = Get-Acl $path
                $weak = $acl.Access | Where-Object {
                    $_.IdentityReference -match 'Users|Everyone|Authenticated' -and
                    $_.FileSystemRights -match 'Write|Modify|FullControl'
                }
                if ($weak) {
                    [PSCustomObject]@{
                        Service  = $svc.Name
                        Path     = $path
                        Identity = $weak.IdentityReference -join '; '
                        Rights   = $weak.FileSystemRights -join '; '
                        RunAs    = $svc.StartName
                    }
                }
            } catch { }
        }
    }
}

Find-WeakServicePermissions | Format-Table -AutoSize

# Unquoted service path
$unquoted = Get-WmiObject Win32_Service | Where-Object {
    $_.PathName -notmatch '^"' -and $_.PathName -match ' ' -and $_.PathName -notmatch '^[A-Za-z]:\\Windows'
}
$unquoted | Select-Object Name, PathName, StartName | Format-Table -AutoSize
```

---

## 3. AlwaysInstallElevated

```powershell
# ตรวจสอบ AlwaysInstallElevated (ถ้า 1 = vulnnable)
$hklm = (Get-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\Installer' -ErrorAction SilentlyContinue).AlwaysInstallElevated
$hkcu = (Get-ItemProperty 'HKCU:\SOFTWARE\Policies\Microsoft\Windows\Installer' -ErrorAction SilentlyContinue).AlwaysInstallElevated

if ($hklm -eq 1 -and $hkcu -eq 1) {
    Write-Host '[VULN] AlwaysInstallElevated enabled!' -ForegroundColor Red
    Write-Host 'Create .msi payload and install as SYSTEM'
} else {
    Write-Host '[OK] AlwaysInstallElevated not enabled' -ForegroundColor Green
}
```

---

## 4. Automated Privesc Checker

```powershell
# PrivEsc check script (lab/CTF use)
function Invoke-PrivEscCheck {
    $findings = @()
    
    # Hot Potato / Sweet Potato checks
    $privs = & whoami /priv /fo csv | ConvertFrom-Csv
    if ($privs | Where-Object { $_.'Privilege Name' -eq 'SeImpersonatePrivilege' -and $_.'State' -eq 'Enabled' }) {
        $findings += '[!] SeImpersonatePrivilege: Potato attack may be possible'
    }
    
    # AlwaysInstallElevated
    $hklm = (Get-ItemProperty 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\Installer' -EA SilentlyContinue).AlwaysInstallElevated
    $hkcu = (Get-ItemProperty 'HKCU:\SOFTWARE\Policies\Microsoft\Windows\Installer'  -EA SilentlyContinue).AlwaysInstallElevated
    if ($hklm -eq 1 -and $hkcu -eq 1) { $findings += '[!] AlwaysInstallElevated enabled' }
    
    # AutoLogon credentials
    $autologon = Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon' -EA SilentlyContinue
    if ($autologon.DefaultPassword) {
        $findings += "[!] AutoLogon credentials: $($autologon.DefaultUserName):$($autologon.DefaultPassword)"
    }
    
    # Unquoted paths
    $unq = Get-WmiObject Win32_Service | Where-Object { $_.PathName -notmatch '^"' -and $_.PathName -match ' ' }
    if ($unq) { $findings += "[!] Unquoted service paths: $($unq.Name -join ', ')" }
    
    # Writable service binaries
    $weakSvcs = Find-WeakServicePermissions  # from section 2
    if ($weakSvcs) { $findings += "[!] Writable service binaries: $($weakSvcs.Service -join ', ')" }
    
    if ($findings.Count -eq 0) {
        Write-Host '[+] No obvious privesc vectors found' -ForegroundColor Green
    } else {
        $findings | ForEach-Object { Write-Host $_ -ForegroundColor Yellow }
    }
}

Invoke-PrivEscCheck
```

---

**ก่อนหน้า ← [Part 45](Part-45.md) | ต่อไป → [Part 47: Incident Response](Part-47.md)**
