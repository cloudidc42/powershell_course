# Part 29: Active Directory Management

> **ระดับ**: 🟠 Advanced | **เวลา**: ~4 ชั่วขนึ่ง

---

## 1. AD Module Setup

```powershell
# ติดตั้ง RSAT AD module
Install-WindowsFeature -Name RSAT-AD-PowerShell  # Windows Server
# หรือแบบ Windows 10/11:
# Settings > Apps > Optional Features > RSAT: Active Directory DS

Import-Module ActiveDirectory
Get-Module ActiveDirectory | Select-Object Version

# Test connection
Get-ADDomain
Get-ADForest
```

---

## 2. User Management

```powershell
# สร้าง user
New-ADUser `
    -Name 'John Smith' `
    -GivenName 'John' `
    -Surname 'Smith' `
    -SamAccountName 'jsmith' `
    -UserPrincipalName 'jsmith@company.local' `
    -EmailAddress 'jsmith@company.com' `
    -Department 'IT' `
    -Title 'Systems Engineer' `
    -Path 'OU=IT,OU=Users,DC=company,DC=local' `
    -AccountPassword (ConvertTo-SecureString 'P@ssw0rd123!' -AsPlainText -Force) `
    -Enabled $true `
    -PasswordNeverExpires $false `
    -ChangePasswordAtLogon $true

# แก้ไขข้อมูล user
Set-ADUser 'jsmith' -Department 'DevOps' -Title 'Senior Engineer'
Set-ADUser 'jsmith' -Add @{ProxyAddresses = 'smtp:jsmith@company.com'}

# เปิด/ปิดใช้งาน account
Enable-ADAccount  'jsmith'
Disable-ADAccount 'jsmith'

# Reset password
Set-ADAccountPassword 'jsmith' `
    -NewPassword (ConvertTo-SecureString 'NewP@ss!' -AsPlainText -Force) `
    -Reset
Set-ADUser 'jsmith' -ChangePasswordAtLogon $true

# Unlock account
Unlock-ADAccount 'jsmith'

# Delete user
Remove-ADUser 'jsmith' -Confirm:$false

# Search users
Get-ADUser -Filter {Department -eq 'IT'} -Properties * |
    Select-Object Name, SamAccountName, EmailAddress, LastLogonDate

Get-ADUser -Filter {Enabled -eq $false} -Properties LastLogonDate |
    Where-Object { $_.LastLogonDate -lt (Get-Date).AddDays(-90) } |
    Select-Object Name, LastLogonDate

# โดยใช้ LDAP filter
Get-ADUser -LDAPFilter '(&(objectClass=user)(department=IT)(!(userAccountControl:1.2.840.113549.1.1.1:=2)))'
```

---

## 3. Group Management

```powershell
# สร้าง group
New-ADGroup `
    -Name 'IT-Admins' `
    -GroupScope Global `
    -GroupCategory Security `
    -Path 'OU=Groups,DC=company,DC=local' `
    -Description 'IT Department Administrators'

# เพิ่ม members
Add-ADGroupMember 'IT-Admins' -Members 'jsmith', 'alice', 'bob'
Add-ADGroupMember 'IT-Admins' -Members (Get-ADUser -Filter {Department -eq 'IT'})

# ลบ members
Remove-ADGroupMember 'IT-Admins' -Members 'bob' -Confirm:$false

# ดู members
Get-ADGroupMember 'IT-Admins' -Recursive

# ดูว่า user อยู่ใน group อะไร 
(Get-ADUser 'jsmith' -Properties MemberOf).MemberOf |
    Get-ADGroup | Select-Object Name, GroupScope
```

---

## 4. Bulk Operations

```powershell
# Bulk create from CSV
# users.csv:
# Name,Username,Email,Department,Password
# John Smith,jsmith,jsmith@co.com,IT,P@ss123

Import-Csv 'users.csv' | ForEach-Object {
    $params = @{
        Name               = $_.Name
        SamAccountName     = $_.Username
        UserPrincipalName  = "$($_.Username)@company.local"
        EmailAddress       = $_.Email
        Department         = $_.Department
        Path               = 'OU=Users,DC=company,DC=local'
        AccountPassword    = ConvertTo-SecureString $_.Password -AsPlainText -Force
        Enabled            = $true
        ChangePasswordAtLogon = $true
    }
    try {
        New-ADUser @params
        Write-Host "Created: $($_.Username)" -ForegroundColor Green
    } catch {
        Write-Warning "Failed $($_.Username): $_"
    }
}

# Audit: เซ็น report inactive users
$inactive = Get-ADUser -Filter {Enabled -eq $true} -Properties LastLogonDate |`
    Where-Object { $_.LastLogonDate -lt (Get-Date).AddDays(-60) -or $_.LastLogonDate -eq $null } |
    Select-Object Name, SamAccountName, LastLogonDate, @{N='DaysInactive';E={
        if ($_.LastLogonDate) { ((Get-Date) - $_.LastLogonDate).Days } else { 'Never' }
    }}

$inactive | Export-Csv 'inactive_users.csv' -NoTypeInformation
Write-Host "$($inactive.Count) inactive users exported"
```

---

## 5. OU และ GPO

```powershell
# Organizational Units
New-ADOrganizationalUnit -Name 'Dev' -Path 'OU=Users,DC=company,DC=local'
Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName

# Move object to OU
Move-ADObject 'CN=jsmith,OU=Users,DC=company,DC=local' `
    -TargetPath 'OU=Dev,OU=Users,DC=company,DC=local'

# GPO (requires GroupPolicy module)
Import-Module GroupPolicy
New-GPO -Name 'Disable USB'
Get-GPO -All | Select-Object DisplayName, GpoStatus, ModificationTime
Get-GPOReport -Name 'Disable USB' -ReportType HTML -Path 'gpo-report.html'
```

---

**ก่อนหน้า ← [Part 28](Part-28.md) | ต่อไป → [Part 30: Registry](Part-30.md)**
