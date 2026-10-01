# Part 82: Capstone 3 - Enterprise PowerShell Framework

> **ระดับ**: 🟠 World-Class | **เวลา**: ~8 ชั่วโมง

---

## 1. Enterprise Module Structure

```
EnterpriseFramework/
  EnterpriseFramework.psd1       # Module manifest
  EnterpriseFramework.psm1       # Root module
  Public/                        # Exported functions
    Invoke-Pipeline.ps1
    New-ServiceRequest.ps1
    Get-ComplianceReport.ps1
  Private/                       # Internal helpers
    Write-Log.ps1
    Get-Config.ps1
  Classes/
    Pipeline.ps1
    ServiceRequest.ps1
    ComplianceChecker.ps1
  Config/
    defaults.json
    environments.json
  Tests/
    EnterpriseFramework.Tests.ps1
```

---

## 2. Enterprise Config System

```powershell
# EnterpriseFramework Config Manager
class ConfigManager {
    hidden [hashtable]$_config = @{}
    hidden [string]$_configDir
    hidden [string]$_environment
    
    ConfigManager([string]$configDir, [string]$environment = 'production') {
        $this._configDir   = $configDir
        $this._environment = $environment
        $this.LoadAll()
    }
    
    hidden [void] LoadAll() {
        # Load defaults
        $defaults = Join-Path $this._configDir 'defaults.json'
        if (Test-Path $defaults) {
            $this._config = Get-Content $defaults | ConvertFrom-Json -AsHashtable
        }
        
        # Override with environment-specific
        $envFile = Join-Path $this._configDir "$($this._environment).json"
        if (Test-Path $envFile) {
            $envConfig = Get-Content $envFile | ConvertFrom-Json -AsHashtable
            foreach ($kv in $envConfig.GetEnumerator()) {
                $this._config[$kv.Key] = $kv.Value
            }
        }
        
        # Override with env vars (prefix: EF_)
        [System.Environment]::GetEnvironmentVariables().GetEnumerator() |
            Where-Object { $_.Key -like 'EF_*' } |
            ForEach-Object {
                $key = $_.Key.Substring(3).ToLower() -replace '__','.'
                $this._config[$key] = $_.Value
            }
    }
    
    [object] Get([string]$key, [object]$default = $null) {
        $parts   = $key -split '\.'
        $current = $this._config
        foreach ($part in $parts) {
            if ($current -isnot [hashtable] -or -not $current.ContainsKey($part)) {
                return $default
            }
            $current = $current[$part]
        }
        return $current
    }
    
    [bool] Require([string[]]$keys) {
        $missing = $keys | Where-Object { $this.Get($_) -eq $null }
        if ($missing) {
            throw "Missing required config keys: $($missing -join ', ')"
        }
        return $true
    }
}

$config = [ConfigManager]::new('./Config', $env:ENVIRONMENT ?? 'production')
$config.Require(@('database.host','database.name','api.key'))

$dbHost = $config.Get('database.host')
$apiKey = $config.Get('api.key', 'default-key')
```

---

## 3. Enterprise Pipeline Engine

```powershell
class PipelineStep {
    [string]$Name
    [scriptblock]$Action
    [scriptblock]$Condition   = { $true }  # Run if true
    [bool]$ContinueOnError    = $false
    [int]$TimeoutSeconds      = 300
    
    PipelineStep([string]$name, [scriptblock]$action) {
        $this.Name   = $name
        $this.Action = $action
    }
}

class Pipeline {
    [string]$Name
    [PipelineStep[]]$Steps = @()
    [hashtable]$Context    = @{}
    hidden [System.Collections.Generic.List[hashtable]]$_results
    
    Pipeline([string]$name) {
        $this.Name     = $name
        $this._results = [System.Collections.Generic.List[hashtable]]::new()
    }
    
    [Pipeline] AddStep([string]$name, [scriptblock]$action) {
        $this.Steps += [PipelineStep]::new($name, $action)
        return $this  # Fluent API
    }
    
    [Pipeline] AddConditionalStep([string]$name, [scriptblock]$condition, [scriptblock]$action) {
        $step           = [PipelineStep]::new($name, $action)
        $step.Condition = $condition
        $this.Steps    += $step
        return $this
    }
    
    [hashtable] Execute() {
        $startTime = [datetime]::Now
        $success   = $true
        
        Write-Host "=== Pipeline: $($this.Name) ==" -ForegroundColor Cyan
        
        foreach ($step in $this.Steps) {
            # Check condition
            if (-not (& $step.Condition $this.Context)) {
                Write-Host "[SKIP] $($step.Name)" -ForegroundColor Gray
                continue
            }
            
            $stepStart = [datetime]::Now
            Write-Host "[RUN]  $($step.Name)..." -ForegroundColor Yellow -NoNewline
            
            try {
                $result = Invoke-WithTimeout {
                    param($action, $ctx)
                    & $action $ctx
                } -TimeoutSeconds $step.TimeoutSeconds
                
                $duration = [math]::Round(([datetime]::Now - $stepStart).TotalSeconds, 1)
                Write-Host " OK (${duration}s)" -ForegroundColor Green
                
                $this._results.Add(@{ Step=$step.Name; Status='OK'; DurationS=$duration; Output=$result })
                
                # Pass output to context
                if ($result -is [hashtable]) {
                    foreach ($kv in $result.GetEnumerator()) { $this.Context[$kv.Key] = $kv.Value }
                }
                
            } catch {
                $duration = [math]::Round(([datetime]::Now - $stepStart).TotalSeconds, 1)
                Write-Host " FAILED (${duration}s): $_" -ForegroundColor Red
                $this._results.Add(@{ Step=$step.Name; Status='FAILED'; Error=$_.Exception.Message })
                
                if (-not $step.ContinueOnError) {
                    $success = $false
                    break
                }
            }
        }
        
        $totalDuration = [math]::Round(([datetime]::Now - $startTime).TotalSeconds, 1)
        $status = if ($success) { 'SUCCESS' } else { 'FAILED' }
        Write-Host "=== Pipeline $status in ${totalDuration}s ===" `
            -ForegroundColor $(if ($success) {'Green'} else {'Red'})
        
        return @{
            Name     = $this.Name
            Status   = $status
            Duration = $totalDuration
            Steps    = $this._results
        }
    }
}

# Usage - full deployment pipeline
$deployPipeline = [Pipeline]::new('Production Deploy')
$deployPipeline.Context = @{ AppName='myapp'; Version='2.1.0'; Env='prod' }

$deployPipeline.
    AddStep('Secrets Scan', { param($ctx) Invoke-SecretsAudit -Path '.' -Recurse }).
    AddStep('SAST',         { param($ctx) Invoke-SASTScan -Path './scripts' -FailOnError }).
    AddStep('Tests',        { param($ctx) Invoke-Pester -Path './tests' -CI }).
    AddStep('Build Image',  { param($ctx) Invoke-DockerBuild -AppName $ctx.AppName -Version $ctx.Version -Registry 'mycompany.azurecr.io' }).
    AddConditionalStep('Deploy Staging',
        { param($ctx) $ctx.Env -ne 'prod' },
        { param($ctx) Deploy-ToAKS -AppName $ctx.AppName -ImageTag $ctx.imageTag -Cluster 'aks-staging' -Namespace 'staging' -RG 'rg-staging' }
    ).
    AddConditionalStep('Deploy Production',
        { param($ctx) $ctx.Env -eq 'prod' },
        { param($ctx) Deploy-ToAKS -AppName $ctx.AppName -ImageTag $ctx.imageTag -Cluster 'aks-prod' -Namespace 'production' -RG 'rg-prod' }
    ).
    AddStep('Smoke Tests',  { param($ctx) Test-Smoke "https://$($ctx.AppName).prod.company.com/health" })

$result = $deployPipeline.Execute()
$result | ConvertTo-Json -Depth 5 | Set-Content './deploy-result.json'
if ($result.Status -ne 'SUCCESS') { exit 1 }
```

---

**ก่อนหน้า ← [Part 81](Part-81.md) | ต่อไป → [Part 83: Advanced Module Development](Part-83.md)**
