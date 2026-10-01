# Part 45: Active Directory Security (Lab/CTF)

> **ระดับ**: 🔴 Professional/Security | **เวลา**: ~5 ชั่วขนึ่ง

> ⚠️ **สำหรับ**: CTF, AD Lab (GOAD/VULNLAB), Authorized Penetration Testing, Blue Team Defense **เท่านั้น**

---

## 1. AD Enumeration (PowerView-style)

```powershell
# ต้องมี ActiveDirectory module หรือ PowerView
Import-Module ActiveDirectory

# Domain info
$domain = Get-ADDomain
Write-Host "Domain: $($domain.DNSRoot)"
Write-Host "DC: $($domain.PDCEmulator)"
Write-Host "Forest: $($domain.Forest)"

# Domain Controllers
Get-ADDomainController -Filter * | Select-Object Name, IPv4Address, IsGlobalCatalog, OperatingSystem

# All users with admin-sensitive attributes
Get-ADUser -Filter * -Properties AdminCount, ServicePrincipalName, TrustedForDelegation, PasswordNeverExpires |
    Where-Object { $_.AdminCount -eq 1 } |
    Select-Object SamAccountName, AdminCount, PasswordNeverExpires, Enabled

# Kerberoastable accounts (SPN set on user)
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName |
    Select-Object SamAccountName, ServicePrincipalName

# AS-REP Roastable (no pre-auth)
Get-ADUser -Filter {DoesNotRequirePreAuth -eq $true} -Properties DoesNotRequirePreAuth |
    Select-Object SamAccountName

# Unconstrained delegation
Get-ADComputer -Filter {TrustedForDelegation -eq $true} -Properties TrustedForDelegation |
    Select-Object Name, DNSHostName

Get-ADUser -Filter {TrustedForDelegation -eq $true} -Properties TrustedForDelegation |
    Select-Object SamAccountName

# Constrained delegation
Get-ADObject -Filter {msDS-AllowedToDelegateTo -ne "$null"} `
    -Properties msDS-AllowedToDelegateTo, SamAccountName |
    Select-Object SamAccountName, 'msDS-AllowedToDelegateTo'
```

---

## 2. ACL Abuse Paths

```powershell
# ตรวจหา weak ACLs บน AD objects
function Find-ADACLWeakness {
    param([string]$SearchBase = (Get-ADDomain).DistinguishedName)
    
    $lowPrivGroups = @('Domain Users', 'Authenticated Users', 'Everyone')
    
    Get-ADObject -SearchBase $SearchBase -Filter * -Properties nTSecurityDescriptor | ForEach-Object {
        $obj = $_
        $obj.nTSecurityDescriptor.Access | Where-Object {
            $ace = $_
            ($lowPrivGroups -contains $ace.IdentityReference.Value.Split('\')[1]) -and
            $ace.ActiveDirectoryRights -match 'GenericWrite|GenericAll|WriteDACL|WriteOwner|WriteProperty'
        } | ForEach-Object {
            [PSCustomObject]@{
                Object  = $obj.DistinguishedName
                Identity= $_.IdentityReference
                Rights  = $_.ActiveDirectoryRights
                Type    = $_.AccessControlType
            }
        }
    }
}

# AD user property abuse
# GenericWrite on user --> write ServicePrincipalName --> Kerberoast
# GenericAll on group  --> add member
# WriteDACL            --> give yourself DCSync rights

# Check who has DCSync rights (Replicating Directory Changes)
function Get-DCsyncAccounts {
    $domain  = Get-ADDomain
    $domainDN = $domain.DistinguishedName
    $domainSD = (Get-ADObject $domainDN -Properties nTSecurityDescriptor).nTSecurityDescriptor
    
    $replicateGuid = [Guid]'1131f6aa-9c07-11d1-f79f-00c04fc2dcd2'  # Replicating Directory Changes
    
    $domainSD.Access | Where-Object {
        $_.ObjectType -eq $replicateGuid -and $_.AccessControlType -eq 'Allow'
    } | Select-Object IdentityReference, ActiveDirectoryRights
}

Get-DCsyncAccounts
```

---

## 3. Bloodhound Data Collection

```powershell
# BloodHound เป็น tool สำหรับ AD path analysis (CTF/authorized engagements)

# ใช้ SharpHound (official collector)
# https://github.com/BloodHoundAD/SharpHound

# PowerShell wrapper
function Invoke-BloodHoundCollection {
    param(
        [ValidateSet('All','DCOnly','Group','LocalAdmin','Session','Trusts','Default')]
        [string]$CollectionMethod = 'Default',
        [string]$OutputDir = '.'
    )
    
    if (!(Test-Path 'SharpHound.exe')) {
        Write-Error 'SharpHound.exe not found. Download from: https://github.com/BloodHoundAD/SharpHound'
        return
    }
    
    & .\SharpHound.exe `
        --collectionmethods $CollectionMethod `
        --outputdirectory $OutputDir `
        --zipfilename 'bloodhound_data.zip' `
        --randomfilenames
    
    Write-Host "Data collected to $OutputDir"
}
```

---

## 4. Pass-the-Hash Defense (Blue Team)

```powershell
# ตรวจสอบและป้องกัน Pass-the-Hash

# 1. Detect - ค้นหาใน Security log
# Event ID 4624 with Logon Type 3 + NTLM Auth from unexpected source
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -MaxEvents 1000 |
    ForEach-Object {
        $xml = [xml]$_.ToXml()
        $d   = $xml.Event.EventData.Data
        [PSCustomObject]@{
            Time       = $_.TimeCreated
            LogonType  = ($d | Where-Object Name -eq 'LogonType').'#text'
            AuthPackage= ($d | Where-Object Name -eq 'AuthenticationPackageName').'#text'
            TargetUser = ($d | Where-Object Name -eq 'TargetUserName').'#text'
            SourceIP   = ($d | Where-Object Name -eq 'IpAddress').'#text'
        }
    } | Where-Object { $_.LogonType -eq '3' -and $_.AuthPackage -eq 'NTLM' } |
    Group-Object TargetUser | Sort-Object Count -Descending

# 2. Mitigations:
# - Enable Credential Guard (requires VBS/UEFI)
# - Set LocalAccountTokenFilterPolicy = 0
# - Use Protected Users Security Group
# - Disable NTLM or restrict to domain controllers only

# Check Credential Guard
(Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root/Microsoft/Windows/DeviceGuard).SecurityServicesRunning
# 2 = Credential Guard running

# Protected Users group: ผู้ใช้ใน group นี้ จะไม่สามารถ cache credentials
Add-ADGroupMember -Identity 'Protected Users' -Members 'AdminUser'
```

---

**ก่อนหน้า ← [Part 44](Part-44.md) | ต่อไป → [Part 46: Privilege Escalation Lab](Part-46.md)**
