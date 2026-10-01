# Part 70: Advanced Error Handling & Resilience

> **ระดับ**: 🔴 Advanced | **เวลา**: ~4 ชั่วโมง

---

## 1. Structured Error Handling

```powershell
# Custom exception types via .NET
Add-Type @"
using System;

public class AppException : Exception {
    public string Code  { get; }
    public object Data2 { get; }
    public AppException(string code, string message, object data = null) : base(message) {
        Code  = code;
        Data2 = data;
    }
}

public class ValidationException  : AppException {
    public ValidationException(string field, string msg)
        : base("VALIDATION_ERROR", $"{field}: {msg}") {}
}

public class NotFoundException : AppException {
    public NotFoundException(string resource, object id)
        : base("NOT_FOUND", $"{resource} with id '{id}' not found") {}
}

public class ServiceUnavailableException : AppException {
    public ServiceUnavailableException(string service)
        : base("SERVICE_UNAVAILABLE", $"{service} is currently unavailable") {}
}
"@

# Usage
function Get-UserById {
    param([int]$Id)
    
    if ($Id -le 0) { throw [ValidationException]::new('Id', 'must be positive') }
    
    $user = Invoke-RestMethod "https://api.example.com/users/$Id" -ErrorAction SilentlyContinue
    if (-not $user)  { throw [NotFoundException]::new('User', $Id) }
    
    return $user
}

try {
    $user = Get-UserById -Id 0
} catch [ValidationException] {
    Write-Warning "Validation: $($_.Exception.Message) [Code: $($_.Exception.Code)]"
} catch [NotFoundException] {
    Write-Warning "Not found: $($_.Exception.Message)"
} catch {
    Write-Error "Unexpected: $_"
}
```

---

## 2. Result Type Pattern

```powershell
# Result<T> monad pattern
class Result {
    [bool]$Success
    [object]$Value
    [string]$Error
    [string]$Code
    
    static [Result] Ok([object]$value) {
        $r         = [Result]::new()
        $r.Success = $true
        $r.Value   = $value
        return $r
    }
    
    static [Result] Fail([string]$error, [string]$code = 'ERROR') {
        $r         = [Result]::new()
        $r.Success = $false
        $r.Error   = $error
        $r.Code    = $code
        return $r
    }
    
    [Result] Map([scriptblock]$fn) {
        if ($this.Success) {
            try   { return [Result]::Ok(& $fn $this.Value) }
            catch { return [Result]::Fail($_.Exception.Message) }
        }
        return $this
    }
    
    [Result] Bind([scriptblock]$fn) {
        if ($this.Success) { return & $fn $this.Value }
        return $this
    }
    
    [object] GetOrDefault([object]$default) {
        return if ($this.Success) { $this.Value } else { $default }
    }
}

# Functions returning Result
function Divide {
    param([double]$a, [double]$b)
    if ($b -eq 0) { return [Result]::Fail('Division by zero', 'MATH_ERROR') }
    return [Result]::Ok($a / $b)
}

function Sqrt {
    param([double]$n)
    if ($n -lt 0) { return [Result]::Fail('Cannot take sqrt of negative', 'MATH_ERROR') }
    return [Result]::Ok([Math]::Sqrt($n))
}

# Chained operations
$result = (Divide 10 2).Bind({ param($v) Sqrt $v }).Map({ param($v) [Math]::Round($v, 4) })

if ($result.Success) {
    Write-Host "Result: $($result.Value)" -ForegroundColor Green
} else {
    Write-Warning "Error [$($result.Code)]: $($result.Error)"
}

# Default value
$safe = (Divide 10 0).GetOrDefault(0)
```

---

## 3. Retry & Resilience Patterns

```powershell
function Invoke-WithResilience {
    param(
        [scriptblock]$Action,
        [int]$MaxRetries       = 3,
        [int]$InitialDelayMs   = 500,
        [double]$BackoffFactor = 2.0,
        [int[]]$RetryOnStatus  = @(429,500,502,503,504),
        [scriptblock]$OnRetry  = { param($attempt,$error) Write-Verbose "Retry $attempt`: $error" },
        [scriptblock]$Fallback = $null
    )
    
    $delay = $InitialDelayMs
    
    for ($attempt = 1; $attempt -le ($MaxRetries + 1); $attempt++) {
        try {
            return & $Action
        } catch {
            $ex     = $_.Exception
            $status = [int]($ex.Response?.StatusCode ?? 0)
            
            $isRetryable = $attempt -le $MaxRetries -and
                ($status -in $RetryOnStatus -or $ex -is [System.Net.WebException])
            
            if ($isRetryable) {
                & $OnRetry $attempt $ex.Message
                Start-Sleep -Milliseconds $delay
                $delay = [int]($delay * $BackoffFactor)
            } elseif ($Fallback) {
                Write-Verbose "All retries exhausted, using fallback"
                return & $Fallback $ex
            } else {
                throw
            }
        }
    }
}

# Usage with fallback to cache
$data = Invoke-WithResilience `
    -Action    { Invoke-RestMethod 'https://api.example.com/data' } `
    -MaxRetries 3 `
    -OnRetry   { param($n,$e) Write-Warning "Attempt $n failed: $e" } `
    -Fallback  { param($e) Get-Content './cache/data.json' | ConvertFrom-Json }

# Timeout wrapper
function Invoke-WithTimeout {
    param([scriptblock]$Action, [int]$TimeoutSeconds = 30)
    
    $job    = Start-Job -ScriptBlock $Action
    $result = Wait-Job $job -Timeout $TimeoutSeconds
    
    if ($result) {
        $output = Receive-Job $job
        Remove-Job $job
        return $output
    } else {
        Remove-Job $job -Force
        throw [TimeoutException]::new("Operation timed out after ${TimeoutSeconds}s")
    }
}

Invoke-WithTimeout { Invoke-RestMethod 'https://slow-api.example.com/data' } -TimeoutSeconds 10
```

---

## 4. Structured Logging with Context

```powershell
class CorrelatedLogger {
    [string]$CorrelationId
    [string]$Service
    [hashtable]$Context = @{}
    
    CorrelatedLogger([string]$service) {
        $this.Service       = $service
        $this.CorrelationId = [Guid]::NewGuid().ToString('N').Substring(0,12)
    }
    
    hidden [void] Write([string]$level, [string]$message, [hashtable]$data = @{}) {
        $entry = @{
            ts            = [datetime]::UtcNow.ToString('o')
            level         = $level
            service       = $this.Service
            correlationId = $this.CorrelationId
            message       = $message
        }
        foreach ($kv in ($this.Context + $data).GetEnumerator()) {
            $entry[$kv.Key] = $kv.Value
        }
        $json = $entry | ConvertTo-Json -Compress
        
        switch ($level) {
            'ERROR' { Write-Host $json -ForegroundColor Red    }
            'WARN'  { Write-Host $json -ForegroundColor Yellow }
            'DEBUG' { Write-Host $json -ForegroundColor Gray   }
            default { Write-Host $json -ForegroundColor White  }
        }
    }
    
    [void] Info ([string]$msg, [hashtable]$d = @{}) { $this.Write('INFO',  $msg, $d) }
    [void] Warn ([string]$msg, [hashtable]$d = @{}) { $this.Write('WARN',  $msg, $d) }
    [void] Error([string]$msg, [hashtable]$d = @{}) { $this.Write('ERROR', $msg, $d) }
    [void] Debug([string]$msg, [hashtable]$d = @{}) { $this.Write('DEBUG', $msg, $d) }
    
    [void] WithContext([hashtable]$ctx) {
        foreach ($kv in $ctx.GetEnumerator()) { $this.Context[$kv.Key] = $kv.Value }
    }
}

$log = [CorrelatedLogger]::new('order-service')
$log.WithContext(@{ userId=42; env='production' })

$log.Info('Processing order', @{ orderId=12345; amount=99.99 })
try {
    # Do work...
    $log.Info('Order complete', @{ orderId=12345; duration_ms=142 })
} catch {
    $log.Error('Order failed', @{ orderId=12345; error=$_.Exception.Message })
}
```

---

**ก่อนหน้า ← [Part 69](Part-69.md) | ต่อไป → [Part 71: Data Processing & Analytics](Part-71.md)**
