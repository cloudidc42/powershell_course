# Part 62: Monitoring & Alerting Automation

> **ระดับ**: 🔴 Advanced | **เวลา**: ~4 ชั่วโมง

---

## 1. Prometheus Metrics Collection

```powershell
# Query Prometheus
function Get-PrometheusMetric {
    param(
        [string]$BaseUrl,
        [string]$Query,
        [datetime]$Start = ([datetime]::Now.AddHours(-1)),
        [datetime]$End   = [datetime]::Now,
        [string]$Step    = '60s'
    )
    
    $params = @{
        query = $Query
        start = $Start.ToUniversalTime().ToString('yyyy-MM-ddTHH:mm:ssZ')
        end   = $End.ToUniversalTime().ToString('yyyy-MM-ddTHH:mm:ssZ')
        step  = $Step
    }
    $qs  = ($params.GetEnumerator() | ForEach-Object { "$($_.Key)=$([Uri]::EscapeDataString($_.Value))" }) -join '&'
    $res = Invoke-RestMethod "$BaseUrl/api/v1/query_range?$qs"
    
    if ($res.status -ne 'success') { throw "Prometheus error: $($res.error)" }
    return $res.data.result
}

# Get current value
function Get-PrometheusInstant {
    param([string]$BaseUrl, [string]$Query)
    $qs  = "query=$([Uri]::EscapeDataString($Query))"
    $res = Invoke-RestMethod "$BaseUrl/api/v1/query?$qs"
    return $res.data.result | ForEach-Object {
        [PSCustomObject]@{
            Metric = $_.metric
            Value  = [double]$_.value[1]
            Time   = [System.DateTimeOffset]::FromUnixTimeSeconds([long]$_.value[0]).LocalDateTime
        }
    }
}

$prometheus = 'http://prometheus:9090'

# CPU usage per pod
$cpuMetrics = Get-PrometheusInstant -BaseUrl $prometheus `
    -Query 'sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)'
$cpuMetrics | Sort-Object Value -Descending | Select-Object -First 10 | Format-Table

# Memory usage
$memMetrics = Get-PrometheusInstant -BaseUrl $prometheus `
    -Query 'container_memory_working_set_bytes{container!=""} / 1024 / 1024'
$memMetrics | Sort-Object Value -Descending | Select-Object -First 10 | Format-Table

# HTTP error rate
$errorRate = Get-PrometheusInstant -BaseUrl $prometheus `
    -Query 'sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)'
```

---

## 2. Alert Manager

```powershell
class AlertManager {
    [hashtable]$Rules    = @{}
    [hashtable]$Channels = @{}
    
    [void] AddRule([string]$name, [scriptblock]$condition, [string]$severity, [string]$message) {
        $this.Rules[$name] = @{
            Condition = $condition
            Severity  = $severity
            Message   = $message
            LastFired = $null
            CooldownMin = 15
        }
    }
    
    [void] AddSlackChannel([string]$name, [string]$webhook) {
        $this.Channels[$name] = @{ type='slack'; webhook=$webhook }
    }
    
    [void] AddEmailChannel([string]$name, [string]$smtp, [string]$from, [string[]]$to) {
        $this.Channels[$name] = @{ type='email'; smtp=$smtp; from=$from; to=$to }
    }
    
    [void] Evaluate([hashtable]$metrics) {
        foreach ($ruleName in $this.Rules.Keys) {
            $rule  = $this.Rules[$ruleName]
            $fired = & $rule.Condition $metrics
            
            if ($fired) {
                $cooldown = $rule.LastFired -and `
                    (([datetime]::Now - $rule.LastFired).TotalMinutes -lt $rule.CooldownMin)
                if (-not $cooldown) {
                    $this.SendAlert($ruleName, $rule)
                    $this.Rules[$ruleName].LastFired = [datetime]::Now
                }
            }
        }
    }
    
    hidden [void] SendAlert([string]$name, [hashtable]$rule) {
        Write-Warning "ALERT [$($rule.Severity.ToUpper())] $name`: $($rule.Message)"
        
        foreach ($ch in $this.Channels.Values) {
            switch ($ch.type) {
                'slack' {
                    $color = switch ($rule.Severity) {
                        'critical' { 'danger' }
                        'warning'  { 'warning' }
                        default    { 'good' }
                    }
                    $payload = @{
                        attachments = @(@{
                            color   = $color
                            title   = "[$($rule.Severity.ToUpper())] $name"
                            text    = $rule.Message
                            ts      = [System.DateTimeOffset]::Now.ToUnixTimeSeconds()
                        })
                    } | ConvertTo-Json -Depth 5
                    Invoke-RestMethod -Uri $ch.webhook -Method POST -Body $payload -ContentType 'application/json'
                }
                'email' {
                    Send-MailMessage -SmtpServer $ch.smtp -From $ch.from -To $ch.to `
                        -Subject "[ALERT-$($rule.Severity.ToUpper())] $name" `
                        -Body $rule.Message
                }
            }
        }
    }
}

$alerts = [AlertManager]::new()
$alerts.AddSlackChannel('ops', $env:SLACK_WEBHOOK)

$alerts.AddRule('HighCPU',
    { param($m) $m.CpuPercent -gt 90 },
    'warning', 'CPU usage exceeds 90%')

$alerts.AddRule('DiskFull',
    { param($m) $m.DiskFreeGB -lt 5 },
    'critical', 'Disk space below 5GB!')

$alerts.AddRule('ServiceDown',
    { param($m) $m.ServiceHealthy -eq $false },
    'critical', 'Service health check failed!')

# Evaluate loop
while ($true) {
    $metrics = @{
        CpuPercent     = (Get-Counter '\Processor(_Total)\% Processor Time').CounterSamples.CookedValue
        DiskFreeGB     = [math]::Round((Get-PSDrive C).Free / 1GB, 1)
        ServiceHealthy = (Invoke-WebRequest 'http://myapp/health' -UseBasicParsing).StatusCode -eq 200
    }
    $alerts.Evaluate($metrics)
    Start-Sleep 60
}
```

---

## 3. Grafana Dashboard via API

```powershell
$grafanaUrl = 'http://grafana:3000'
$headers    = @{ Authorization = "Bearer $env:GRAFANA_TOKEN" }

# List dashboards
$dashboards = Invoke-RestMethod "$grafanaUrl/api/search" -Headers $headers
$dashboards | Select-Object title, uid, url | Format-Table

# Get dashboard JSON
$dash = Invoke-RestMethod "$grafanaUrl/api/dashboards/uid/myapp-overview" -Headers $headers

# Create annotation (deploy marker)
function Add-GrafanaAnnotation {
    param([string]$Text, [string[]]$Tags)
    
    $annotation = @{
        text  = $Text
        tags  = $Tags
        time  = [System.DateTimeOffset]::Now.ToUnixTimeMilliseconds()
    } | ConvertTo-Json
    
    Invoke-RestMethod "$grafanaUrl/api/annotations" `
        -Method POST -Headers $headers -Body $annotation -ContentType 'application/json'
}

Add-GrafanaAnnotation -Text 'Deployed v2.1.0' -Tags @('deploy','production')

# Create alert notification channel
$channel = @{
    name    = 'Slack-Ops'
    type    = 'slack'
    settings = @{ url=$env:SLACK_WEBHOOK; recipient='#alerts' }
} | ConvertTo-Json

Invoke-RestMethod "$grafanaUrl/api/alert-notifications" `
    -Method POST -Headers $headers -Body $channel -ContentType 'application/json'
```

---

**ก่อนหน้า ← [Part 61](Part-61.md) | ต่อไป → [Part 63: Network Security](Part-63.md)**
