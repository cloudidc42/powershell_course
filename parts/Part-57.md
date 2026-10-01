# Part 57: Exchange และ Microsoft 365

> **ระดับ**: 🟡 Professional | **เวลา**: ~4 ชั่วขนึ่ง

---

## 1. Exchange Online (M365)

```powershell
# ติดตั้ง
Install-Module ExchangeOnlineManagement -Force

# Connect
Connect-ExchangeOnline -UserPrincipalName 'admin@company.onmicrosoft.com'

# Mailboxes
Get-Mailbox -ResultSize Unlimited | Select-Object DisplayName, PrimarySmtpAddress, Database
Get-Mailbox -Identity 'john.smith@company.com' | Format-List

# Mailbox statistics
Get-MailboxStatistics 'john.smith@company.com' | 
    Select-Object DisplayName, TotalItemSize, ItemCount, LastLogonTime

# All mailboxes with size
Get-Mailbox -ResultSize Unlimited | Get-MailboxStatistics |
    Select-Object DisplayName, TotalItemSize, ItemCount |
    Sort-Object {$_.TotalItemSize.Value.ToBytes()} -Descending |
    Select-Object -First 20

# Set mailbox properties
Set-Mailbox 'jsmith@company.com' -ProhibitSendQuota 10GB -IssueWarningQuota 9GB
Set-Mailbox 'jsmith@company.com' -LitigationHoldEnabled $true

# Distribution Groups
Get-DistributionGroup
Add-DistributionGroupMember 'IT-Team@company.com' -Member 'alice@company.com'
Remove-DistributionGroupMember 'IT-Team@company.com' -Member 'bob@company.com'

# Shared Mailbox
New-Mailbox -Name 'Support' -Alias 'support' -Shared
Add-MailboxPermission 'support@company.com' -User 'alice@company.com' -AccessRights FullAccess
```

---

## 2. Microsoft Graph API

```powershell
# Microsoft Graph สำหรับ M365
Install-Module Microsoft.Graph -Force

# Connect ด้วย service principal
$tenantId   = 'your-tenant-id'
$clientId   = 'your-app-id'
$clientSecret = $env:MS_CLIENT_SECRET

$body = @{
    grant_type    = 'client_credentials'
    scope         = 'https://graph.microsoft.com/.default'
    client_id     = $clientId
    client_secret = $clientSecret
}

$token = Invoke-RestMethod `
    -Uri "https://login.microsoftonline.com/$tenantId/oauth2/v2.0/token" `
    -Method POST `
    -Body $body

$headers = @{ Authorization = "Bearer $($token.access_token)" }

# Users
$users = Invoke-RestMethod `
    -Uri 'https://graph.microsoft.com/v1.0/users?$top=100' `
    -Headers $headers

$users.value | Select-Object displayName, userPrincipalName, jobTitle, department

# ส่ง email
$mailBody = @{
    message = @{
        subject      = 'Test from PowerShell'
        body         = @{ contentType='HTML'; content='<h1>Hello!</h1>' }
        toRecipients = @(@{emailAddress=@{address='recipient@company.com'}})
    }
} | ConvertTo-Json -Depth 10

Invoke-RestMethod `
    -Uri 'https://graph.microsoft.com/v1.0/users/sender@company.com/sendMail' `
    -Method POST `
    -Headers $headers `
    -Body $mailBody `
    -ContentType 'application/json'
```

---

## 3. Teams ด้วย Graph API

```powershell
$headers = @{ Authorization = "Bearer $($token.access_token)" }

# List teams
$teams = Invoke-RestMethod 'https://graph.microsoft.com/v1.0/groups?$filter=resourceProvisioningOptions/Any(x:x eq ''Team'')' -Headers $headers
$teams.value | Select-Object id, displayName

# Send Teams message
$teamId   = 'team-id-guid'
$channelId = 'channel-id'

$msg = @{
    body = @{
        contentType = 'html'
        content     = '<b>Deployment Complete!</b> v2.1.0 is live.'
    }
} | ConvertTo-Json

Invoke-RestMethod `
    -Uri "https://graph.microsoft.com/v1.0/teams/$teamId/channels/$channelId/messages" `
    -Method POST -Headers $headers -Body $msg -ContentType 'application/json'

# Teams notification function
function Send-TeamsMessage {
    param([string]$WebhookUrl, [string]$Title, [string]$Text, [string]$Color = '0078D7')
    
    $payload = @{
        '@type'      = 'MessageCard'
        '@context'   = 'http://schema.org/extensions'
        themeColor   = $Color
        summary      = $Title
        sections     = @(@{activityTitle=$Title; activityText=$Text})
    } | ConvertTo-Json -Depth 5
    
    Invoke-RestMethod -Uri $WebhookUrl -Method POST -Body $payload -ContentType 'application/json'
}

Send-TeamsMessage `
    -WebhookUrl $env:TEAMS_WEBHOOK `
    -Title 'Deploy: MyApp v2.1.0' `
    -Text 'Production deployment successful!' `
    -Color '00CC00'
```

---

**ก่อนหน้า ← [Part 56](Part-56.md) | ต่อไป → [Part 58: Advanced REST APIs](Part-58.md)**
