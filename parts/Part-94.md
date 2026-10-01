# Part 94: World-Class Capstone — Unified Platform

> **ระดับ**: 🟠 World-Class | **เวลา**: ~8 ชั่วข็น (Final Capstone)

---

## Scenario: Unified DevOps Platform

สร้าง **Enterprise PowerShell Platform** ที่รวมทุกสิ่งที่เรียนมาในคอร์สเข้าด้วยกัน:

```
UnifiedPlatform/
├── Platform.ps1          # Entry point + module loader
├── Core/
│   ├── Config.ps1           # ConfigManager
│   ├── Logger.ps1           # StructuredLogger
│   └── Pipeline.ps1         # Pipeline DSL
├── Services/
│   ├── DeployService.ps1    # Deployment automation
│   ├── MonitorService.ps1   # Health + metrics
│   ├── SecurityService.ps1  # Secrets + compliance
│   └── LifecycleService.ps1 # User lifecycle
├── Plugins/              # Extensible plugins
└── Tests/               # Pester test suite
```

---

## 1. Platform Bootstrap

```powershell
# Platform.ps1 - entry point
using namespace System.Collections.Generic

# Load all components
$components = @(
    './Core/Config.ps1'
    './Core/Logger.ps1'
    './Core/Pipeline.ps1'
    './Services/DeployService.ps1'
    './Services/MonitorService.ps1'
    './Services/SecurityService.ps1'
    './Services/LifecycleService.ps1'
)

foreach ($c in $components) {
    if (Test-Path $c) {
        . $c
    } else {
        Write-Warning "Component not found: $c"
    }
}

# Global platform instance
$global:Platform = [PlatformContext]::new()

class PlatformContext {
    [ConfigManager]$Config
    [StructuredLogger]$Log
    [PluginRegistry]$Plugins
    [HealthAggregator]$Health
    [hashtable]$Services = @{}
    
    PlatformContext() {
        $env       = $env:PLATFORM_ENV ?? 'development'
        $this.Config  = [ConfigManager]::new('./config', $env)
        $this.Log     = [StructuredLogger]::new('platform', $env)
        $this.Plugins = [PluginRegistry]::new('./plugins')
        $this.Health  = [HealthAggregator]::new()
        
        $this.Bootstrap()
    }
    
    hidden [void] Bootstrap() {
        $this.Log.Info('Platform starting', @{ env=$this.Config.Get('app.env') })
        
        # Load plugins
        $this.Plugins.DiscoverAndLoad()
        
        # Register health checks
        $dbHost   = $this.Config.Get('database.host')
        $dbPort   = $this.Config.Get('database.port') ?? 5432
        $cacheHost= $this.Config.Get('cache.host')
        
        $this.Health.Register('database', { @{ ok=(Test-NetConnection $dbHost   -Port $dbPort -InformationLevel Quiet) } })
        $this.Health.Register('cache',    { @{ ok=(Test-NetConnection $cacheHost -Port 6379   -InformationLevel Quiet) } })
        
        $this.Log.Info('Platform ready')
    }
    
    [hashtable] GetHealth() { return $this.Health.RunAll() }
    
    [void] Shutdown() {
        $this.Log.Info('Platform shutting down')
        foreach ($svc in $this.Services.Values) {
            if ($svc.PSObject.Methods['Stop']) { $svc.Stop() }
        }
    }
}
```

---

## 2. Unified Deployment Pipeline

```powershell
# Full-featured deploy with all integrations
function Invoke-UnifiedDeploy {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory)][string]$App,
        [Parameter(Mandatory)][string]$Version,
        [Parameter(Mandatory)][ValidateSet('dev','staging','production')][string]$Env,
        [switch]$SkipTests,
        [switch]$BlueGreen
    )
    
    $log     = $global:Platform.Log
    $config  = $global:Platform.Config
    $trace   = [TraceContext]::new("deploy-$App")
    
    $log.Info('Deploy initiated', @{ app=$App; version=$Version; env=$Env })
    
    $pipeline = [PipelineDSL]::new("Deploy $App v$Version to $Env")
    $pipeline
        .WithEnv(@{ APP=$App; VERSION=$Version; ENV=$Env })
        .Stage('Pre-Deploy Checks', {
            step 'Security scan' {
                $issues = Invoke-SecretsAudit -Path "./src" -ErrorAction SilentlyContinue
                if ($issues) { throw "Security gate: $($issues.Count) secrets found" }
            }
            step 'SAST scan' {
                $issues = Invoke-ScriptAnalyzer -Path './src' -Severity 'Error' -Recurse
                if ($issues) { throw "SAST: $($issues.Count) errors" }
            }
            step 'Run tests' -continueOnError:$SkipTests {
                $r = Invoke-Pester './tests' -PassThru
                if ($r.FailedCount -gt 0) { throw "Tests: $($r.FailedCount) failed" }
            }
        })
        .Stage('Build & Push', {
            step 'Docker build' {
                $registry = $config.Get('registry.url')
                docker build -t "$registry/$env:APP:$env:VERSION" "./src"
                docker push "$registry/$env:APP:$env:VERSION"
            }
        })
        .Stage('Deploy', {
            step 'Update K8s deployment' {
                $registry  = $config.Get('registry.url')
                $namespace = $env:ENV
                kubectl set image "deployment/$env:APP" app="$registry/$env:APP:$env:VERSION" -n $namespace
                kubectl rollout status "deployment/$env:APP" -n $namespace --timeout=300s
            }
        })
        .Stage('Smoke Tests', {
            step 'Health check' {
                $url = $config.Get("services.$env:APP.url")
                $r   = Invoke-RestMethod "$url/health" -TimeoutSec 30
                if ($r.status -ne 'healthy') { throw "App unhealthy after deploy" }
            }
        })
        .Stage('Notify', {
            step 'Slack notification' -continueOnError {
                $webhook = $config.Get('notifications.slack.webhook')
                Invoke-RestMethod $webhook -Method POST -Body (@{
                    text="Deployed $env:APP v$env:VERSION to $env:ENV :rocket:"
                } | ConvertTo-Json) -ContentType 'application/json'
            }
        })
    
    if ($PSCmdlet.ShouldProcess("$App v$Version to $Env")) {
        $pipeline.Run()
    }
    
    $log.Info('Deploy complete', @{ app=$App; version=$Version; env=$Env })
}

# Run a deploy
Invoke-UnifiedDeploy -App 'api-gateway' -Version '3.2.1' -Env 'staging' -WhatIf
```

---

## 3. Automated Ops Runbook

```powershell
# Self-healing operations runbook
class OpsRunbook {
    hidden [hashtable]$_playbooks = @{}
    
    [void] Register([string]$condition, [scriptblock]$action, [string]$description) {
        $this._playbooks[$condition] = @{ action=$action; description=$description; lastRun=$null }
    }
    
    [void] Run([string]$incident) {
        Write-Host "=== Incident: $incident ==" -ForegroundColor Red
        
        foreach ($kv in $this._playbooks.GetEnumerator()) {
            $pattern = $kv.Key
            if ($incident -match $pattern) {
                $pb = $kv.Value
                Write-Host "  Applying playbook: $($pb.description)" -ForegroundColor Yellow
                try {
                    & $pb.action $incident
                    $pb.lastRun = [datetime]::Now
                    Write-Host "  Playbook executed OK" -ForegroundColor Green
                } catch {
                    Write-Warning "  Playbook failed: $_"
                }
            }
        }
    }
}

$runbook = [OpsRunbook]::new()

$runbook.Register('HighCPU', {
    param($incident)
    $node = ($incident | Select-String 'node:(\S+)').Matches.Groups[1].Value
    Write-Host "Restarting high-CPU pods on $node"
    kubectl get pods --field-selector="spec.nodeName=$node" -o name | ForEach-Object {
        kubectl delete $_ --grace-period=30
    }
}, 'Restart pods on high-CPU node')

$runbook.Register('DiskFull', {
    param($incident)
    Write-Host 'Cleaning Docker images and logs'
    docker system prune -f
    Get-ChildItem '/var/log' -Filter '*.gz' | Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-7) } | Remove-Item
}, 'Clean disk space')

$runbook.Register('ServiceDown', {
    param($incident)
    $svc = ($incident | Select-String 'service:(\S+)').Matches.Groups[1].Value
    Write-Host "Restarting service: $svc"
    kubectl rollout restart "deployment/$svc" -n production
}, 'Restart downed service')

# Simulate incident
$runbook.Run('HighCPU on node:k8s-worker-3 (95%)')
$runbook.Run('ServiceDown service:payment-api timeout=30s')
```

---

**ก่อนหน้า ← [Part 93](Part-93.md) | ต่อไป → [Part 95: Course Completion และ Next Steps](Part-95.md)**
