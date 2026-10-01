# Part 90: Real-World Case Study — IT Automation Platform

> **ระดับ**: 🟠 World-Class | **เวลา**: ~6 ชั่วข็น (+workshop)

---

## Scenario: Enterprise IT Automation Platform

**ต้องการ**: สร้างระบบอัตโนมัติสำหรับ IT Helpdesk ที่:
- สร้าง User account ใน AD→Exchange→Azure AD
- Reset password พร้อมแจ้งเตือน
- Onboard/Offboard เบ็ดเบิลถูกต้อง
- Report สุขภาพสื่อบนและ License

---

## 1. User Lifecycle Manager

```powershell
# User Lifecycle Manager
class UserLifecycleManager {
    [string]$Domain
    [string]$OUBase
    [hashtable]$Config
    
    UserLifecycleManager([hashtable]$config) {
        $this.Config = $config
        $this.Domain = $config.Domain
        $this.OUBase = $config.OUBase
    }
    
    [PSCustomObject] Onboard([hashtable]$employee) {
        $errors = @()
        $steps  = [System.Collections.Generic.List[string]]::new()
        
        try {
            # 1. Create AD account
            $username = $this.GenerateUsername($employee.FirstName, $employee.LastName)
            $upn      = "$username@$($this.Domain)"
            $ou       = "OU=$($employee.Department),$($this.OUBase)"
            
            $password = $this.GeneratePassword(16)
            $secPass  = ConvertTo-SecureString $password -AsPlainText -Force
            
            New-ADUser `
                -Name             "$($employee.FirstName) $($employee.LastName)" `
                -GivenName        $employee.FirstName `
                -Surname          $employee.LastName `
                -SamAccountName   $username `
                -UserPrincipalName $upn `
                -Department       $employee.Department `
                -Title            $employee.Title `
                -Manager          $employee.ManagerSAM `
                -Path             $ou `
                -AccountPassword  $secPass `
                -Enabled          $true
            
            $steps.Add('AD Account: OK')
            
            # 2. Add to groups
            $groups = $this.GetDepartmentGroups($employee.Department)
            foreach ($grp in $groups) {
                Add-ADGroupMember -Identity $grp -Members $username
            }
            $steps.Add("Groups: Added to $($groups.Count) groups")
            
            # 3. Create Exchange mailbox (waits for AD sync)
            Start-Sleep -Seconds 30
            Enable-Mailbox -Identity $upn -Database $this.Config.ExchangeDB
            Set-Mailbox -Identity $upn `
                -EmailAddresses @{ Add="$username@$($this.Config.EmailDomain)" } `
                -IssueWarningQuota 8GB `
                -ProhibitSendQuota 9GB
            $steps.Add('Exchange Mailbox: OK')
            
            # 4. Assign M365 license
            $this.AssignM365License($upn, $employee.LicenseSku)
            $steps.Add('M365 License: Assigned')
            
            # 5. Send welcome email
            $this.SendWelcomeEmail($employee, $username, $password)
            $steps.Add('Welcome Email: Sent')
            
            return [PSCustomObject]@{
                Success  = $true
                Username = $username
                UPN      = $upn
                Password = $password
                Steps    = $steps
            }
        } catch {
            return [PSCustomObject]@{
                Success = $false
                Error   = $_.Exception.Message
                Steps   = $steps
            }
        }
    }
    
    [void] Offboard([string]$SAM, [switch]$ImmediateDisable) {
        $user = Get-ADUser -Identity $SAM -Properties MemberOf, Manager, Department
        
        if ($ImmediateDisable) {
            # Disable immediately
            Disable-ADAccount -Identity $SAM
        }
        
        # Remove all group memberships except Domain Users
        $user.MemberOf | Where-Object { $_ -notmatch 'CN=Domain Users' } |
            ForEach-Object { Remove-ADGroupMember -Identity $_ -Members $SAM -Confirm:$false }
        
        # Clear manager
        Set-ADUser -Identity $SAM -Manager $null
        
        # Set expiry 24h from now (grace period)
        $expiry = (Get-Date).AddHours(24)
        Set-ADAccountExpiration -Identity $SAM -DateTime $expiry
        
        # Move to Disabled OU
        $disabledOU = "OU=Disabled,$($this.OUBase)"
        Move-ADObject -Identity $user.DistinguishedName -TargetPath $disabledOU
        
        # Mailbox: set forwarding to manager, then hide
        $upn = $user.UserPrincipalName
        $mgr = Get-ADUser $user.Manager -Properties UserPrincipalName
        Set-Mailbox -Identity $upn -ForwardingAddress $mgr.UserPrincipalName -DeliverToMailboxAndForward $false
        Set-Mailbox -Identity $upn -HiddenFromAddressListsEnabled $true
        
        Write-Host "Offboarded: $SAM (expires: $expiry)" -ForegroundColor Yellow
    }
    
    hidden [string] GenerateUsername([string]$first, [string]$last) {
        $base = ($first[0] + $last).ToLower() -replace '[^a-z0-9]'
        $i    = 0
        while (Get-ADUser -Filter { SamAccountName -eq $base } -ErrorAction SilentlyContinue) {
            $base = ($first[0] + $last + (++$i)).ToLower() -replace '[^a-z0-9]'
        }
        return $base.Substring(0, [Math]::Min($base.Length, 20))
    }
    
    hidden [string] GeneratePassword([int]$length) {
        $chars = 'ABCDEFGHKLMNPRSTUVWXYZabcdefghkmnprstuvwxyz23456789!@#$%^&*'
        -join (1..$length | ForEach-Object { $chars[(Get-Random -Maximum $chars.Length)] })
    }
    
    hidden [void] AssignM365License([string]$UPN, [string]$SkuId) {
        Connect-MgGraph -Scopes 'User.ReadWrite.All' -NoWelcome
        Set-MgUserLicense -UserId $UPN `
            -AddLicenses @{ SkuId=$SkuId } `
            -RemoveLicenses @()
    }
    
    hidden [string[]] GetDepartmentGroups([string]$dept) {
        return @("GRP_All_Staff", "GRP_$dept", "GRP_VPN_Users")
    }
    
    hidden [void] SendWelcomeEmail([hashtable]$emp, [string]$user, [string]$pass) {
        $body = @"
Welcome $($emp.FirstName)!  
Your corporate account has been created.

Username : $user
Email    : $user@$($this.Config.EmailDomain)
Password : $pass (Please change immediately)

For IT support, contact helpdesk@$($this.Config.Domain)
"@
        Send-MailMessage `
            -SmtpServer $this.Config.SmtpServer `
            -From       "IT Helpdesk <noreply@$($this.Config.Domain)>" `
            -To         $emp.PersonalEmail `
            -Subject    'Your Corporate Account is Ready' `
            -Body       $body
    }
}

# Usage
$mgr = [UserLifecycleManager]::new(@{
    Domain      = 'corp.example.com'
    OUBase      = 'OU=Users,DC=corp,DC=example,DC=com'
    EmailDomain = 'example.com'
    ExchangeDB  = 'MAILDB01'
    SmtpServer  = 'smtp.corp.example.com'
    LicenseSku  = '6fd2c87f-b296-42f0-b197-1e91e994b900'  # E3
})

$result = $mgr.Onboard(@{
    FirstName    = 'Somchai'
    LastName     = 'Jaidee'
    Department   = 'Engineering'
    Title        = 'Software Engineer'
    ManagerSAM   = 'thana.wong'
    LicenseSku   = '6fd2c87f-b296-42f0-b197-1e91e994b900'
    PersonalEmail= 'somchai.j@gmail.com'
})

if ($result.Success) {
    Write-Host "Onboarded: $($result.Username)" -ForegroundColor Green
    $result.Steps | ForEach-Object { Write-Host "  - $_" }
} else {
    Write-Error "Failed: $($result.Error)"
}
```

---

## 2. License และ Asset Report

```powershell
function Get-ITAssetReport {
    param([string]$OutputDir = './reports')
    
    New-Item -ItemType Directory $OutputDir -Force | Out-Null
    
    # AD users summary
    $users = Get-ADUser -Filter * -Properties Enabled, Department, LastLogonDate, PasswordLastSet |
        Select-Object Name, SamAccountName, Department, Enabled,
            @{n='DaysSinceLogin'; e={
                if ($_.LastLogonDate) { [int]([datetime]::Now - $_.LastLogonDate).TotalDays } else { 999 }
            }},
            @{n='PasswordAgeD'; e={
                if ($_.PasswordLastSet) { [int]([datetime]::Now - $_.PasswordLastSet).TotalDays } else { 999 }
            }}
    
    # Stats
    $report = @{
        GeneratedAt       = [datetime]::Now.ToString('o')
        TotalUsers        = $users.Count
        ActiveUsers       = @($users | Where-Object Enabled).Count
        DisabledUsers     = @($users | Where-Object { -not $_.Enabled }).Count
        StaleAccounts     = @($users | Where-Object { $_.DaysSinceLogin -gt 90 -and $_.Enabled }).Count
        OldPasswords      = @($users | Where-Object { $_.PasswordAgeD -gt 90 }).Count
        ByDepartment      = ($users | Group-Object Department | Select-Object Name, Count)
    }
    
    # HTML report
    $html = @"
<!DOCTYPE html>
<html><head><style>
body{font-family:Arial;margin:20px;background:#f5f5f5}
table{width:100%;border-collapse:collapse;background:#fff}
th{background:#0078d4;color:#fff;padding:8px}
td{border:1px solid #ddd;padding:6px}
.warn{color:orange} .danger{color:red}
</style></head><body>
<h1>IT Asset Report - $($report.GeneratedAt)</h1>
<h2>Summary</h2>
<ul>
<li>Total Users: <b>$($report.TotalUsers)</b></li>
<li>Active: <b>$($report.ActiveUsers)</b></li>
<li>Stale (&gt;90d no login): <b class='warn'>$($report.StaleAccounts)</b></li>
<li>Old Passwords (&gt;90d): <b class='warn'>$($report.OldPasswords)</b></li>
</ul>
<h2>By Department</h2>
<table><tr><th>Department</th><th>Count</th></tr>
$($report.ByDepartment | ForEach-Object { "<tr><td>$($_.Name)</td><td>$($_.Count)</td></tr>" } | Join-String)
</table></body></html>
"@
    
    $html | Set-Content "$OutputDir/asset-report-$(Get-Date -Format yyyyMMdd).html"
    $report | ConvertTo-Json -Depth 5 | Set-Content "$OutputDir/asset-report-$(Get-Date -Format yyyyMMdd).json"
    
    Write-Host "Report saved to $OutputDir" -ForegroundColor Green
    return $report
}

Get-ITAssetReport
```

---

**ก่อนหน้า ← [Part 89](Part-89.md) | ต่อไป → [Part 91: Case Study — Cloud Migration](Part-91.md)**
