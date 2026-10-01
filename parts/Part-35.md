# Part 35: Azure Cloud ด้วย PowerShell

> **ระดับ**: 🟠 Advanced | **เวลา**: ~4 ชั่วขนึ่ง

---

## 1. Setup Az Module

```powershell
# ติดตั้ง
Install-Module Az -Force -AllowClobber
Install-Module Az.Accounts, Az.Compute, Az.Storage, Az.Network -Force

# Login
Connect-AzAccount
# หรือใช้ service principal ใน CI
$cred = [PSCredential]::new(
    $env:ARM_CLIENT_ID,
    (ConvertTo-SecureString $env:ARM_CLIENT_SECRET -AsPlainText -Force)
)
Connect-AzAccount -ServicePrincipal -Credential $cred -TenantId $env:ARM_TENANT_ID

# เลือก subscription
Get-AzSubscription | Select-Object Name, Id, State
Set-AzContext -SubscriptionId 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
```

---

## 2. Resource Groups และ VMs

```powershell
# Resource Groups
New-AzResourceGroup -Name 'rg-myapp-prod' -Location 'southeastasia'
Get-AzResourceGroup | Select-Object ResourceGroupName, Location
Remove-AzResourceGroup -Name 'rg-myapp-test' -Force

# Virtual Machines
Get-AzVM -ResourceGroupName 'rg-myapp-prod'
Get-AzVM | Select-Object Name, ResourceGroupName, Location, @{N='Size';E={$_.HardwareProfile.VmSize}}

# Start/Stop VMs
Start-AzVM  -ResourceGroupName 'rg-myapp-prod' -Name 'vm-web01'
Stop-AzVM   -ResourceGroupName 'rg-myapp-prod' -Name 'vm-web01' -Force
Restart-AzVM -ResourceGroupName 'rg-myapp-prod' -Name 'vm-web01'

# Create VM
$cred = Get-Credential -UserName 'azureuser'
New-AzVM `
    -ResourceGroupName 'rg-myapp-prod' `
    -Name 'vm-web02' `
    -Location 'southeastasia' `
    -Image 'Win2022AzureEditionCore' `
    -Size 'Standard_B2s' `
    -Credential $cred `
    -OpenPorts 80,443,3389

# Run script on VM
Invoke-AzVMRunCommand `
    -ResourceGroupName 'rg-myapp-prod' `
    -Name 'vm-web01' `
    -CommandId 'RunPowerShellScript' `
    -ScriptPath '.\setup.ps1'
```

---

## 3. Azure Storage

```powershell
# Storage Account
$sa = Get-AzStorageAccount -ResourceGroupName 'rg-myapp-prod' -Name 'mystorageacct'
$ctx = $sa.Context

# Blob operations
Get-AzStorageContainer -Context $ctx
New-AzStorageContainer -Name 'uploads' -Context $ctx -Permission Off

Set-AzStorageBlobContent -Container 'uploads' -File 'C:\file.pdf' -Blob 'docs/file.pdf' -Context $ctx
Get-AzStorageBlob -Container 'uploads' -Context $ctx | Select-Object Name, Length, LastModified
Get-AzStorageBlobContent -Container 'uploads' -Blob 'docs/file.pdf' -Destination 'C:\Download' -Context $ctx
Remove-AzStorageBlob -Container 'uploads' -Blob 'docs/file.pdf' -Context $ctx

# SAS URL
$sas = New-AzStorageBlobSASToken `
    -Container 'uploads' `
    -Blob 'docs/file.pdf' `
    -Context $ctx `
    -Permission 'r' `
    -ExpiryTime (Get-Date).AddHours(1) `
    -FullUri
Write-Host "Download URL: $sas"

# Azure File Share
New-AzStorageShare -Name 'fileshare' -Context $ctx
Set-AzStorageFileContent -ShareName 'fileshare' -Source 'report.pdf' -Context $ctx
```

---

## 4. Azure Functions + App Service

```powershell
# App Service Plan
New-AzAppServicePlan `
    -ResourceGroupName 'rg-myapp-prod' `
    -Name 'asp-myapp' `
    -Location 'southeastasia' `
    -Tier 'Standard' `
    -NumberofWorkers 2

# Web App
New-AzWebApp `
    -ResourceGroupName 'rg-myapp-prod' `
    -Name 'myapp-api' `
    -AppServicePlan 'asp-myapp'

# App Settings
$settings = @{
    DB_CONNECTION = 'Server=...'
    API_KEY       = 'secret'
    ASPNETCORE_ENVIRONMENT = 'Production'
}
Set-AzWebApp -ResourceGroupName 'rg-myapp-prod' -Name 'myapp-api' -AppSettings $settings

# Deploy ZIP
Publish-AzWebApp `
    -ResourceGroupName 'rg-myapp-prod' `
    -Name 'myapp-api' `
    -ArchivePath '.\dist\app.zip' `
    -Force

# Monitoring
Get-AzMetric `
    -ResourceId (Get-AzWebApp -Name 'myapp-api').Id `
    -MetricName 'Requests', 'Http5xx', 'AverageResponseTime' `
    -StartTime (Get-Date).AddHours(-1) `
    -EndTime (Get-Date) `
    -TimeGrain '00:05:00'
```

---

**ก่อนหน้า ← [Part 34](Part-34.md) | ต่อไป → [Part 36: AWS](Part-36.md)**
