# Part 87: PowerShell DSL Patterns

> **ระดับ**: 🟠 World-Class | **เวลา**: ~5 ชั่วโมง

---

## 1. Configuration DSL

```powershell
# DSL: Human-readable configuration syntax
class AppConfig {
    [string]$AppName
    [string]$Environment
    [hashtable]$Database  = @{}
    [hashtable]$Cache     = @{}
    [hashtable]$Auth      = @{}
    [hashtable]$Logging   = @{}
    [hashtable[]]$Services = @()
}

$script:_currentConfig = $null

function app {
    param([string]$Name, [scriptblock]$Config)
    $script:_currentConfig = [AppConfig]::new()
    $script:_currentConfig.AppName = $Name
    & $Config
    return $script:_currentConfig
}

function env { param([string]$Name) $script:_currentConfig.Environment = $Name }

function database {
    param(
        [string]$host    = 'localhost',
        [int]   $port    = 5432,
        [string]$name    = '',
        [string]$user    = '',
        [int]   $poolSize = 20
    )
    $script:_currentConfig.Database = @{
        Host=$host; Port=$port; Name=$name; User=$user; PoolSize=$poolSize
    }
}

function cache {
    param([string]$provider='redis', [string]$host='localhost', [int]$port=6379, [int]$ttl=3600)
    $script:_currentConfig.Cache = @{ Provider=$provider; Host=$host; Port=$port; TTL=$ttl }
}

function auth {
    param([string]$provider='jwt', [int]$tokenExpiry=3600, [string[]]$allowedOrigins=@('*'))
    $script:_currentConfig.Auth = @{ Provider=$provider; TokenExpiry=$tokenExpiry; AllowedOrigins=$allowedOrigins }
}

function logging {
    param([ValidateSet('Debug','Info','Warning','Error')][string]$level='Info', [string]$format='json', [string]$output='stdout')
    $script:_currentConfig.Logging = @{ Level=$level; Format=$format; Output=$output }
}

function service {
    param([string]$name, [string]$url, [int]$timeout=30, [bool]$required=$true)
    $script:_currentConfig.Services += @{ Name=$name; Url=$url; Timeout=$timeout; Required=$required }
}

# DSL usage - reads like config, not code
$config = app 'MyWebAPI' {
    env 'production'
    
    database `
        -host     $env:DB_HOST `
        -port     5432 `
        -name     'myapp' `
        -user     'appuser' `
        -poolSize 50
    
    cache `
        -provider 'redis' `
        -host     $env:REDIS_HOST `
        -ttl      1800
    
    auth `
        -provider       'jwt' `
        -tokenExpiry    7200 `
        -allowedOrigins @('https://app.example.com', 'https://admin.example.com')
    
    logging `
        -level  'Info' `
        -format 'json' `
        -output '/var/log/myapp/app.log'
    
    service -name 'payments'    -url 'https://payments.internal:8443' -timeout 15
    service -name 'users'       -url 'https://users.internal:8080'    -timeout 10
    service -name 'analytics'   -url 'https://analytics.internal'     -required $false
}

$config | ConvertTo-Json -Depth 5
```

---

## 2. Infrastructure DSL

```powershell
# DSL: Describe infrastructure resources
$script:_resources = @()

function resource {
    param([string]$type, [string]$name, [scriptblock]$body)
    $r = @{ type=$type; name=$name; properties=@{} }
    $script:_currentResource = $r
    & $body
    $script:_resources += $r
}

function set {
    param([string]$key, $value)
    $script:_currentResource.properties[$key] = $value
}

function tag   { param([string]$k, [string]$v)    $script:_currentResource.properties.tags[$k] = $v }
function depends_on { param([string[]]$deps)       $script:_currentResource.dependsOn = $deps }

function infrastructure {
    param([scriptblock]$body)
    $script:_resources = @()
    & $body
    return $script:_resources
}

# DSL usage
$infra = infrastructure {
    resource 'azurerm_resource_group' 'main' {
        set 'location' 'eastus'
        set 'name'     'myapp-prod-rg'
        set 'tags'     @{ Environment='production'; ManagedBy='PowerShell' }
    }
    
    resource 'azurerm_virtual_network' 'main' {
        set 'name'                'myapp-vnet'
        set 'address_space'       @('10.0.0.0/16')
        set 'resource_group_name' 'myapp-prod-rg'
        depends_on @('azurerm_resource_group.main')
    }
    
    resource 'azurerm_kubernetes_cluster' 'main' {
        set 'name'                'myapp-aks'
        set 'resource_group_name' 'myapp-prod-rg'
        set 'kubernetes_version'  '1.28'
        set 'node_count'          3
        set 'vm_size'             'Standard_D4s_v5'
        depends_on @('azurerm_virtual_network.main')
    }
}

$infra | Format-Table type, name
```

---

## 3. Pipeline Builder DSL

```powershell
# Fluent DSL for build/CI pipelines
class PipelineDSL {
    hidden [System.Collections.Generic.List[hashtable]]$_stages = [System.Collections.Generic.List[hashtable]]::new()
    [string]$Name
    [hashtable]$Env = @{}
    
    PipelineDSL([string]$name) { $this.Name = $name }
    
    [PipelineDSL] WithEnv([hashtable]$env) {
        foreach ($kv in $env.GetEnumerator()) { $this.Env[$kv.Key] = $kv.Value }
        return $this
    }
    
    [PipelineDSL] Stage([string]$name, [scriptblock]$steps) {
        $stage = @{ name=$name; steps=@(); when=$null }
        $script:_currentStage = $stage
        & $steps
        $this._stages.Add($stage)
        return $this
    }
    
    [PipelineDSL] When([scriptblock]$condition) {
        if ($script:_currentStage) { $script:_currentStage.when = $condition }
        return $this
    }
    
    [void] Run([bool]$dryRun = $false) {
        Write-Host "`n=== Pipeline: $($this.Name) ===" -ForegroundColor Cyan
        
        foreach ($stage in $this._stages) {
            # Evaluate condition
            if ($stage.when) {
                $cond = & $stage.when
                if (-not $cond) {
                    Write-Host "[SKIP] $($stage.name)" -ForegroundColor DarkGray
                    continue
                }
            }
            
            Write-Host "`n[STAGE] $($stage.name)" -ForegroundColor Yellow
            
            foreach ($step in $stage.steps) {
                Write-Host "  > $($step.name)" -ForegroundColor White
                if (-not $dryRun) {
                    try {
                        & $step.action
                        Write-Host "    OK" -ForegroundColor Green
                    } catch {
                        Write-Host "    FAILED: $_" -ForegroundColor Red
                        if ($step.continueOnError) { continue } else { throw }
                    }
                }
            }
        }
    }
}

function step {
    param([string]$name, [scriptblock]$action, [switch]$continueOnError)
    if ($script:_currentStage) {
        $script:_currentStage.steps += @{ name=$name; action=$action; continueOnError=$continueOnError.IsPresent }
    }
}

$script:_currentStage = $null

$pipeline = [PipelineDSL]::new('Production Deploy')
$pipeline
    .WithEnv(@{ APP='myapp'; REGISTRY='myregistry.azurecr.io'; ENV='production' })
    .Stage('Build', {
        step 'Run tests'       { Invoke-Pester './tests' -PassThru | Should -InvokeVerifiable }
        step 'Build image'     { docker build -t "$env:REGISTRY/$env:APP:latest" . }
        step 'Push image'      { docker push "$env:REGISTRY/$env:APP:latest" }
    })
    .Stage('Deploy', {
        step 'Update manifest' { kubectl set image deployment/$env:APP app="$env:REGISTRY/$env:APP:latest" -n $env:ENV }
        step 'Wait for rollout' { kubectl rollout status deployment/$env:APP -n $env:ENV --timeout=300s }
    })
    .Stage('Smoke Test', {
        step 'Health check'    { Invoke-RestMethod "https://$env:APP.example.com/health" }
        step 'Notify Slack'    { Write-Host 'Deploy complete' } -continueOnError
    })

$pipeline.Run(-dryRun:$true)
```

---

**ก่อนหน้า ← [Part 86](Part-86.md) | ต่อไป → [Part 88: Performance Profiling](Part-88.md)**
