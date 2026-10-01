# Part 80: Capstone 1 - Cloud Platform Automation

> **ระดับ**: 🟠 World-Class | **เวลา**: ~6 ชั่วโมง

---

## โครงการ: Full-Stack Cloud Deployment System

สร้างระบบ deploy อัตโนมัติ end-to-end ที่ครอบคลุม:
- Build Docker image
- Push to registry
- Deploy to AKS (Azure Kubernetes Service)
- Run smoke tests
- Notify team
- Rollback อัตโนมัติถ้าไม่ผ่าน

---

## 1. Deploy Pipeline Entry Point

```powershell
#Requires -Version 7.2
#Requires -Modules Az.Aks, Az.ContainerRegistry

[CmdletBinding()]
param(
    [Parameter(Mandatory)]
    [string]$AppName,
    
    [Parameter(Mandatory)]
    [string]$Version,
    
    [ValidateSet('dev','staging','production')]
    [string]$Environment = 'staging',
    
    [switch]$DryRun,
    [switch]$SkipTests,
    [string]$NotifyWebhook = $env:TEAMS_WEBHOOK
)

Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

$config = @{
    Registry     = 'mycompany.azurecr.io'
    AKSCluster   = "aks-$Environment"
    AKSNamespace = $Environment
    RG           = "rg-$Environment"
    SmokTestUrl  = "https://$AppName.$Environment.mycompany.com/health"
}

function Log { param([string]$Msg, [string]$Level='INFO')
    $color = @{INFO='White'; SUCCESS='Green'; WARN='Yellow'; ERROR='Red'}[$Level]
    Write-Host "[$(Get-Date -Format HH:mm:ss)] [$Level] $Msg" -ForegroundColor $color
}

function Notify {
    param([string]$Title, [string]$Message, [string]$Color = 'good')
    if (-not $NotifyWebhook) { return }
    $payload = @{
        attachments = @(@{ color=$Color; title=$Title; text=$Message })
    } | ConvertTo-Json -Depth 5
    Invoke-RestMethod -Uri $NotifyWebhook -Method POST -Body $payload -ContentType 'application/json' | Out-Null
}

# Start deployment
$deployId = "deploy-$AppName-$Version-$(Get-Date -Format yyyyMMddHHmmss)"
Log "Starting deployment $deployId" 'INFO'
Notify "Deploy Started" "$AppName v$Version -> $Environment" 'warning'
```

---

## 2. Build และ Push Image

```powershell
function Invoke-DockerBuild {
    param([string]$AppName, [string]$Version, [string]$Registry, [switch]$DryRun)
    
    $tag      = "$Registry/${AppName}:${Version}"
    $latestTag= "$Registry/${AppName}:latest"
    
    Log "Building Docker image: $tag"
    
    if (-not $DryRun) {
        docker build -t $tag -t $latestTag . --build-arg VERSION=$Version
        if ($LASTEXITCODE -ne 0) { throw "Docker build failed" }
        
        Log "Pushing to registry..."
        docker push $tag
        docker push $latestTag
        if ($LASTEXITCODE -ne 0) { throw "Docker push failed" }
    }
    
    Log "Image ready: $tag" 'SUCCESS'
    return $tag
}

# Login to ACR
if (-not $DryRun) {
    Connect-AzAccount -ServicePrincipal `
        -Credential ([PSCredential]::new($env:AZURE_CLIENT_ID, (ConvertTo-SecureString $env:AZURE_CLIENT_SECRET -AsPlainText -Force))) `
        -TenantId $env:AZURE_TENANT_ID | Out-Null
    
    $token = Get-AzContainerRegistryCredential -ResourceGroupName $config.RG -Name ($config.Registry -split '\.')[0]
    docker login $config.Registry -u $token.Username -p $token.Password | Out-Null
}

$imageTag = Invoke-DockerBuild -AppName $AppName -Version $Version -Registry $config.Registry -DryRun:$DryRun
```

---

## 3. Deploy to AKS

```powershell
function Deploy-ToAKS {
    param(
        [string]$AppName, [string]$ImageTag,
        [string]$Cluster, [string]$Namespace,
        [string]$RG, [switch]$DryRun
    )
    
    if (-not $DryRun) {
        Import-AzAksCredential -ResourceGroupName $RG -Name $Cluster -Force
    }
    
    $oldImage = if (-not $DryRun) {
        (kubectl get deployment/$AppName -n $Namespace -o json | ConvertFrom-Json).spec.template.spec.containers[0].image
    } else { 'dry-run' }
    
    Log "Updating image from $oldImage to $ImageTag"
    
    if (-not $DryRun) {
        kubectl set image deployment/$AppName $AppName=$ImageTag -n $Namespace
        kubectl rollout status deployment/$AppName -n $Namespace --timeout=300s
        if ($LASTEXITCODE -ne 0) { throw "Rollout failed" }
    }
    
    return $oldImage
}

function Invoke-Rollback {
    param([string]$AppName, [string]$OldImage, [string]$Namespace)
    
    Log "ROLLING BACK to $OldImage" 'WARN'
    kubectl set image deployment/$AppName $AppName=$OldImage -n $Namespace
    kubectl rollout status deployment/$AppName -n $Namespace --timeout=120s
    Log "Rollback complete" 'WARN'
}

function Test-Smoke {
    param([string]$Url, [int]$Retries = 5)
    
    for ($i = 0; $i -lt $Retries; $i++) {
        try {
            $r = Invoke-RestMethod $Url -TimeoutSec 10
            if ($r.status -eq 'healthy') { return $true }
        } catch { }
        Start-Sleep 10
    }
    return $false
}

# Execute deployment
$oldImage = $null
try {
    $oldImage = Deploy-ToAKS `
        -AppName   $AppName `
        -ImageTag  $imageTag `
        -Cluster   $config.AKSCluster `
        -Namespace $config.AKSNamespace `
        -RG        $config.RG `
        -DryRun:$DryRun
    
    if (-not $SkipTests -and -not $DryRun) {
        Log "Running smoke tests..."
        if (-not (Test-Smoke $config.SmokTestUrl)) {
            throw "Smoke tests failed!"
        }
    }
    
    Log "Deployment successful: $AppName v$Version -> $Environment" 'SUCCESS'
    Notify "Deploy SUCCESS" "$AppName v$Version deployed to $Environment" 'good'
    
} catch {
    Log "Deployment FAILED: $_" 'ERROR'
    Notify "Deploy FAILED" "$AppName v$Version: $_" 'danger'
    
    if ($oldImage -and -not $DryRun) {
        Invoke-Rollback -AppName $AppName -OldImage $oldImage -Namespace $config.AKSNamespace
        Notify "Auto-Rolled Back" "$AppName reverted to $oldImage" 'warning'
    }
    exit 1
}
```

---

**ก่อนหน้า ← [Part 79](Part-79.md) | ต่อไป → [Part 81: Capstone 2 - SecOps Dashboard](Part-81.md)**
