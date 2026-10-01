# Part 92: Service Mesh และ Observability

> **ระดับ**: 🟠 World-Class | **เวลา**: ~5 ชั่วโมง

---

## 1. Distributed Tracing

```powershell
# OpenTelemetry-style tracing in PowerShell
class TraceContext {
    [string]$TraceId
    [string]$SpanId
    [string]$ParentSpanId
    [string]$ServiceName
    [hashtable]$Attributes = @{}
    [datetime]$StartTime
    hidden [System.Collections.Generic.List[hashtable]]$_spans = [System.Collections.Generic.List[hashtable]]::new()
    
    TraceContext([string]$serviceName) {
        $this.TraceId     = [Guid]::NewGuid().ToString('N')
        $this.SpanId      = [Guid]::NewGuid().ToString('N').Substring(0,16)
        $this.ServiceName = $serviceName
        $this.StartTime   = [datetime]::UtcNow
    }
    
    [SpanContext] StartSpan([string]$operationName, [hashtable]$attrs = @{}) {
        $span = [SpanContext]::new($this, $operationName, $this.SpanId)
        foreach ($kv in $attrs.GetEnumerator()) { $span.Attributes[$kv.Key] = $kv.Value }
        return $span
    }
    
    [void] AddSpan([hashtable]$spanData) { $this._spans.Add($spanData) }
    
    [hashtable] Export() {
        return @{
            traceId     = $this.TraceId
            serviceName = $this.ServiceName
            startTime   = $this.StartTime.ToString('o')
            spans       = @($this._spans)
        }
    }
}

class SpanContext {
    [string]$SpanId
    [string]$OperationName
    [hashtable]$Attributes = @{}
    hidden [TraceContext]$_trace
    hidden [string]$_parentSpanId
    hidden [datetime]$_start
    
    SpanContext([TraceContext]$trace, [string]$name, [string]$parentId) {
        $this.SpanId         = [Guid]::NewGuid().ToString('N').Substring(0,16)
        $this.OperationName  = $name
        $this._trace         = $trace
        $this._parentSpanId  = $parentId
        $this._start         = [datetime]::UtcNow
    }
    
    [void] SetAttribute([string]$key, [object]$value) { $this.Attributes[$key] = $value }
    
    [void] End([bool]$success = $true) {
        $duration = ([datetime]::UtcNow - $this._start).TotalMilliseconds
        $this._trace.AddSpan(@{
            spanId        = $this.SpanId
            parentSpanId  = $this._parentSpanId
            operationName = $this.OperationName
            durationMs    = [math]::Round($duration, 2)
            success       = $success
            attributes    = $this.Attributes
            startTime     = $this._start.ToString('o')
        })
    }
}

# Usage
$trace = [TraceContext]::new('payment-service')
$trace.Attributes['env'] = 'production'

$requestSpan = $trace.StartSpan('process-payment', @{ 'payment.amount'=99.99; 'payment.currency'='USD' })
try {
    # Simulate work
    $dbSpan = $trace.StartSpan('db.query')
    Start-Sleep -Milliseconds 50
    $dbSpan.SetAttribute('db.statement', 'INSERT INTO payments ...')
    $dbSpan.End()
    
    $apiSpan = $trace.StartSpan('api.call')
    Start-Sleep -Milliseconds 120
    $apiSpan.SetAttribute('http.url', 'https://stripe.com/v1/charges')
    $apiSpan.End()
    
    $requestSpan.End($true)
} catch {
    $requestSpan.SetAttribute('error', $_.Exception.Message)
    $requestSpan.End($false)
}

# Export to Jaeger / Zipkin
$traceData = $trace.Export()
$traceData | ConvertTo-Json -Depth 5
```

---

## 2. Structured Logging ด้วย Correlation

```powershell
class StructuredLogger {
    [string]$ServiceName
    [string]$Environment
    [ValidateSet('Debug','Info','Warning','Error','Critical')]
    [string]$MinLevel = 'Info'
    [string]$Output   = 'stdout'
    
    hidden static [hashtable]$_levelOrder = @{ Debug=0; Info=1; Warning=2; Error=3; Critical=4 }
    hidden [string]$_correlationId
    
    StructuredLogger([string]$service, [string]$env) {
        $this.ServiceName  = $service
        $this.Environment  = $env
        $this._correlationId = [Guid]::NewGuid().ToString('N').Substring(0,12)
    }
    
    [void] SetCorrelationId([string]$id) { $this._correlationId = $id }
    
    hidden [void] Write([string]$level, [string]$message, [hashtable]$fields = @{}) {
        if ([StructuredLogger]::_levelOrder[$level] -lt [StructuredLogger]::_levelOrder[$this.MinLevel]) { return }
        
        $entry = @{
            timestamp     = [datetime]::UtcNow.ToString('o')
            level         = $level
            service       = $this.ServiceName
            env           = $this.Environment
            correlationId = $this._correlationId
            message       = $message
        }
        foreach ($kv in $fields.GetEnumerator()) { $entry[$kv.Key] = $kv.Value }
        
        $json = $entry | ConvertTo-Json -Compress
        
        if ($this.Output -eq 'stdout') {
            $color = switch ($level) {
                'Debug'    { 'DarkGray' }
                'Info'     { 'White' }
                'Warning'  { 'Yellow' }
                'Error'    { 'Red' }
                'Critical' { 'Magenta' }
            }
            Write-Host $json -ForegroundColor $color
        } else {
            Add-Content -Path $this.Output -Value $json
        }
    }
    
    [void] Debug([string]$msg, [hashtable]$f=@{})    { $this.Write('Debug',    $msg, $f) }
    [void] Info([string]$msg, [hashtable]$f=@{})     { $this.Write('Info',     $msg, $f) }
    [void] Warning([string]$msg, [hashtable]$f=@{})  { $this.Write('Warning',  $msg, $f) }
    [void] Error([string]$msg, [hashtable]$f=@{})    { $this.Write('Error',    $msg, $f) }
    [void] Critical([string]$msg, [hashtable]$f=@{}) { $this.Write('Critical', $msg, $f) }
}

$log = [StructuredLogger]::new('order-service', 'production')
$log.MinLevel = 'Info'

$log.Info('Order received',  @{ orderId='ORD-123'; customerId='CUST-456'; amount=99.99 })
$log.Info('Payment processed', @{ orderId='ORD-123'; provider='stripe'; chargeId='ch_abc' })
$log.Warning('Shipping delay', @{ orderId='ORD-123'; delayDays=2; reason='carrier-issue' })
```

---

## 3. Health Check Aggregator

```powershell
class HealthAggregator {
    hidden [hashtable]$_checks = @{}
    
    [void] Register([string]$name, [scriptblock]$checkFn, [int]$timeoutSec = 5) {
        $this._checks[$name] = @{ fn=$checkFn; timeout=$timeoutSec }
    }
    
    [hashtable] RunAll() {
        $results  = @{}
        $overall  = 'healthy'
        
        foreach ($kv in $this._checks.GetEnumerator()) {
            $name = $kv.Key
            $sw   = [System.Diagnostics.Stopwatch]::StartNew()
            
            try {
                $job = Start-Job $kv.Value.fn
                $done = Wait-Job $job -Timeout $kv.Value.timeout
                
                if ($done) {
                    $result = Receive-Job $job
                    $status = if ($result.ok) { 'healthy' } else { 'unhealthy' }
                } else {
                    Stop-Job $job
                    $status = 'timeout'
                }
                Remove-Job $job -Force
            } catch {
                $status = 'error'
            }
            
            $sw.Stop()
            $results[$name] = @{ status=$status; latencyMs=[math]::Round($sw.Elapsed.TotalMilliseconds,1) }
            if ($status -ne 'healthy') { $overall = 'degraded' }
        }
        
        return @{ status=$overall; checks=$results; timestamp=[datetime]::UtcNow.ToString('o') }
    }
}

$health = [HealthAggregator]::new()
$health.Register('database',     { @{ ok=(Test-NetConnection localhost -Port 5432).TcpTestSucceeded } }, 5)
$health.Register('redis',        { @{ ok=(Test-NetConnection localhost -Port 6379).TcpTestSucceeded } }, 3)
$health.Register('payments-api', { @{ ok=(Invoke-RestMethod 'https://api.stripe.com/v1/health' -ErrorAction SilentlyContinue) } }, 5)

$result = $health.RunAll()
$result | ConvertTo-Json -Depth 3
```

---

**ก่อนหน้า ← [Part 91](Part-91.md) | ต่อไป → [Part 93: Configuration Management](Part-93.md)**
