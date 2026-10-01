# Part 58: Advanced REST APIs

> **ระดับ**: 🟡 Professional | **เวลา**: ~4 ชั่วขนึ่ง

---

## 1. API Client Pattern

```powershell
# Generic API client class
class ApiClient {
    [string]$BaseUrl
    [hashtable]$DefaultHeaders
    [int]$Timeout
    
    ApiClient([string]$baseUrl) {
        $this.BaseUrl        = $baseUrl.TrimEnd('/')
        $this.DefaultHeaders = @{ 'Content-Type' = 'application/json'; 'Accept' = 'application/json' }
        $this.Timeout        = 30
    }
    
    [void] SetBearerToken([string]$token) {
        $this.DefaultHeaders['Authorization'] = "Bearer $token"
    }
    
    [void] SetApiKey([string]$header, [string]$key) {
        $this.DefaultHeaders[$header] = $key
    }
    
    [object] Request([string]$method, [string]$path, [hashtable]$body = $null, [hashtable]$query = $null) {
        $url = "$($this.BaseUrl)$path"
        if ($query) {
            $qs  = ($query.GetEnumerator() | ForEach-Object { "$($_.Key)=$([Uri]::EscapeDataString($_.Value))" }) -join '&'
            $url = "$url?$qs"
        }
        
        $params = @{
            Uri     = $url
            Method  = $method
            Headers = $this.DefaultHeaders
            TimeoutSec = $this.Timeout
        }
        if ($body) { $params['Body'] = $body | ConvertTo-Json -Depth 10 }
        
        return Invoke-RestMethod @params
    }
    
    [object] Get([string]$path, [hashtable]$query = $null)  { return $this.Request('GET', $path, $null, $query) }
    [object] Post([string]$path, [hashtable]$body)           { return $this.Request('POST', $path, $body) }
    [object] Put([string]$path, [hashtable]$body)            { return $this.Request('PUT', $path, $body) }
    [object] Patch([string]$path, [hashtable]$body)          { return $this.Request('PATCH', $path, $body) }
    [object] Delete([string]$path)                           { return $this.Request('DELETE', $path) }
}

# Usage
$api = [ApiClient]::new('https://api.example.com/v1')
$api.SetBearerToken($env:API_TOKEN)

$users = $api.Get('/users', @{page='1'; limit='50'})
$user  = $api.Post('/users', @{name='Alice'; email='alice@example.com'})
$api.Put("/users/$($user.id)", @{name='Alice Smith'})
$api.Delete("/users/$($user.id)")
```

---

## 2. Pagination Helper

```powershell
# Fetch all pages automatically
function Get-AllPages {
    param(
        [ApiClient]$Client,
        [string]$Path,
        [string]$PageParam  = 'page',
        [string]$LimitParam = 'limit',
        [int]$Limit         = 100,
        [string]$DataKey    = 'data'
    )
    
    $page    = 1
    $allData = @()
    
    do {
        $result = $Client.Get($Path, @{$PageParam=[string]$page; $LimitParam=[string]$Limit})
        $data   = if ($DataKey) { $result.$DataKey } else { $result }
        $allData += $data
        $page++
        $hasMore = $data.Count -eq $Limit
    } while ($hasMore)
    
    return $allData
}

# Link header pagination
function Get-LinkHeaderPages {
    param([string]$FirstUrl, [hashtable]$Headers = @{})
    
    $url  = $FirstUrl
    $all  = @()
    
    while ($url) {
        $resp = Invoke-WebRequest -Uri $url -Headers $Headers
        $all += ($resp.Content | ConvertFrom-Json)
        
        # Parse Link header: <url>; rel="next"
        $link = $resp.Headers['Link']
        $url  = if ($link) {
            ($link -split ',') | Where-Object { $_ -match 'rel="next"' } |
                ForEach-Object { if ($_ -match '<([^>]+)>') { $Matches[1] } }
        } else { $null }
    }
    
    return $all
}
```

---

## 3. Retry และ Circuit Breaker

```powershell
function Invoke-ApiWithRetry {
    param(
        [scriptblock]$ApiCall,
        [int]$MaxRetries   = 3,
        [int]$DelaySeconds = 2,
        [int[]]$RetryStatus = @(429, 500, 502, 503, 504)
    )
    
    for ($i = 0; $i -le $MaxRetries; $i++) {
        try {
            return & $ApiCall
        } catch {
            $status = [int]$_.Exception.Response.StatusCode
            
            if ($i -eq $MaxRetries -or $status -notin $RetryStatus) {
                throw
            }
            
            # Rate limit: check Retry-After header
            $retryAfter = [int]($_.Exception.Response.Headers['Retry-After'] ?? $DelaySeconds)
            Write-Warning "Attempt $($i+1)/$MaxRetries failed ($status). Retry in ${retryAfter}s..."
            Start-Sleep $retryAfter * [Math]::Pow(2, $i)  # exponential backoff
        }
    }
}

# Circuit breaker
class CircuitBreaker {
    [string]$Name
    [int]$FailureThreshold = 5
    [int]$ResetTimeout     = 60
    hidden [int]$_failures = 0
    hidden [datetime]$_openedAt
    hidden [string]$_state = 'Closed'  # Closed / Open / HalfOpen
    
    CircuitBreaker([string]$name) { $this.Name = $name }
    
    [object] Execute([scriptblock]$call) {
        if ($this._state -eq 'Open') {
            if (([datetime]::Now - $this._openedAt).TotalSeconds -gt $this.ResetTimeout) {
                $this._state = 'HalfOpen'
            } else {
                throw [Exception]::new("Circuit $($this.Name) is OPEN")
            }
        }
        try {
            $result = & $call
            $this._failures = 0
            $this._state    = 'Closed'
            return $result
        } catch {
            $this._failures++
            if ($this._failures -ge $this.FailureThreshold) {
                $this._state    = 'Open'
                $this._openedAt = [datetime]::Now
            }
            throw
        }
    }
}

$cb = [CircuitBreaker]::new('PaymentAPI')
$cb.Execute({ Invoke-RestMethod 'https://payment.api/charge' -Method POST })
```

---

**ก่อนหน้า ← [Part 57](Part-57.md) | ต่อไป → [Part 59: Microservices](Part-59.md)**
