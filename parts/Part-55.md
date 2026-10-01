# Part 55: IIS Web Server Management

> **ระดับ**: 🟡 Professional | **เวลา**: ~4 ชั่วขนึ่ง

---

## 1. IIS Module Setup

```powershell
# ติดตั้ง IIS และ WebAdministration module
Install-WindowsFeature -Name Web-Server -IncludeManagementTools
Install-WindowsFeature -Name Web-Asp-Net45
Install-WindowsFeature -Name Web-Basic-Auth

Import-Module WebAdministration

# ดูโครงสร้าง IIS
Get-Website | Format-Table Name, State, PhysicalPath, Bindings
Get-WebApplication | Format-Table Name, Site, PhysicalPath
Get-WebAppPool | Format-Table Name, State, ManagedRuntimeVersion
```

---

## 2. Site และ App Pool

```powershell
# สร้าง Application Pool
New-WebAppPool -Name 'MyAppPool'
Set-ItemProperty 'IIS:\AppPools\MyAppPool' managedRuntimeVersion 'v4.0'
Set-ItemProperty 'IIS:\AppPools\MyAppPool' enable32BitAppOnWin64 $false
Set-ItemProperty 'IIS:\AppPools\MyAppPool' processModel.identityType 'ApplicationPoolIdentity'

# ควบคุม recycling
Set-ItemProperty 'IIS:\AppPools\MyAppPool' recycling.periodicRestart.time '1.05:00:00'

# สร้าง Website
New-Website `
    -Name 'MyApp' `
    -PhysicalPath 'C:\inetpub\apps\myapp' `
    -ApplicationPool 'MyAppPool' `
    -Port 80

# HTTPS binding
New-WebBinding -Name 'MyApp' -Protocol https -Port 443 -IPAddress '*' -SslFlags 0

# Assign certificate
$cert = Get-ChildItem Cert:\LocalMachine\My | Where-Object { $_.Subject -match 'myapp.local' }
$binding = Get-WebBinding 'MyApp' -Protocol https
$binding.AddSslCertificate($cert.Thumbprint, 'My')

# Web Application inside site
New-WebApplication -Name 'api' -Site 'MyApp' -PhysicalPath 'C:\inetpub\apps\api' -ApplicationPool 'ApiPool'

# Control sites
Start-Website  'MyApp'
Stop-Website   'MyApp'
Restart-WebItem 'IIS:\Sites\MyApp'

# App pool control
Start-WebAppPool  'MyAppPool'
Stop-WebAppPool   'MyAppPool'
Restart-WebAppPool 'MyAppPool'
```

---

## 3. IIS Configuration

```powershell
# Default documents
Add-WebConfigurationProperty `
    -pspath 'MACHINE/WEBROOT/APPHOST/MyApp' `
    -filter 'system.webServer/defaultDocument/files' `
    -name '.' `
    -value @{value='index.html'}

# Custom headers
Add-WebConfigurationProperty `
    -pspath 'MACHINE/WEBROOT/APPHOST/MyApp' `
    -filter 'system.webServer/httpProtocol/customHeaders' `
    -name '.' `
    -value @{name='X-Content-Type-Options'; value='nosniff'}

# Error pages
Set-WebConfigurationProperty `
    -pspath 'MACHINE/WEBROOT/APPHOST/MyApp' `
    -filter 'system.webServer/httpErrors' `
    -name 'existingResponse' `
    -value 'Replace'

# URL Rewrite (require URL Rewrite module)
Add-WebConfigurationProperty `
    -pspath 'MACHINE/WEBROOT/APPHOST/MyApp' `
    -filter 'system.webServer/rewrite/rules' `
    -name '.' `
    -value @{name='HTTP to HTTPS'; stopProcessing='True'}

# gzip compression
Set-WebConfigurationProperty -Filter 'system.webServer/urlCompression' -Name doStaticCompression -Value $true
Set-WebConfigurationProperty -Filter 'system.webServer/urlCompression' -Name doDynamicCompression -Value $true
```

---

## 4. IIS Logs และ Monitoring

```powershell
# วิเคราะห์ IIS logs
function Get-IISLogStats {
    param(
        [string]$LogPath = 'C:\inetpub\logs\LogFiles\W3SVC1',
        [int]$LastDays    = 1
    )
    
    $since  = (Get-Date).AddDays(-$LastDays)
    $logs   = Get-ChildItem $LogPath *.log | Where-Object { $_.LastWriteTime -ge $since }
    
    $entries = $logs | ForEach-Object {
        Get-Content $_.FullName | Where-Object { $_ -notlike '#*' } |
            ForEach-Object {
                $f = $_ -split ' '
                if ($f.Count -ge 10) {
                    [PSCustomObject]@{
                        Date    = $f[0]
                        Time    = $f[1]
                        IP      = $f[2]
                        Method  = $f[3]
                        URI     = $f[4]
                        Status  = $f[8]
                        Bytes   = $f[9]
                    }
                }
            }
    }
    
    $stats = [PSCustomObject]@{
        TotalRequests   = $entries.Count
        Errors5xx       = ($entries | Where-Object { $_.Status -like '5*' }).Count
        Errors4xx       = ($entries | Where-Object { $_.Status -like '4*' }).Count
        TopURIs         = $entries | Group-Object URI | Sort-Object Count -Descending | Select-Object -First 10
        TopIPs          = $entries | Group-Object IP  | Sort-Object Count -Descending | Select-Object -First 10
    }
    
    return $stats
}

$stats = Get-IISLogStats
Write-Host "Requests: $($stats.TotalRequests), 4xx: $($stats.Errors4xx), 5xx: $($stats.Errors5xx)"
$stats.TopURIs | Format-Table Name, Count
```

---

**ก่อนหน้า ← [Part 54](Part-54.md) | ต่อไป → [Part 56: SQL Server DBA](Part-56.md)**
