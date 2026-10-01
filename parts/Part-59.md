# Part 59: Microservices ด้วย PowerShell

> **ระดับ**: 🟡 Professional | **เวลา**: ~4 ชั่วโมง

---

## 1. REST API Server ด้วย HttpListener

```powershell
class RestApiServer {
    [System.Net.HttpListener]$Listener
    [hashtable]$Routes = @{}
    [bool]$Running     = $false
    
    RestApiServer([string[]]$prefixes) {
        $this.Listener = [System.Net.HttpListener]::new()
        foreach ($p in $prefixes) { $this.Listener.Prefixes.Add($p) }
    }
    
    [void] Route([string]$method, [string]$path, [scriptblock]$handler) {
        $this.Routes["$method $path"] = $handler
    }
    
    [void] Start() {
        $this.Listener.Start()
        $this.Running = $true
        Write-Host "API Server listening..."
        
        while ($this.Running) {
            $ctx = $this.Listener.GetContext()
            $req = $ctx.Request
            $res = $ctx.Response
            
            try {
                $key     = "$($req.HttpMethod) $($req.Url.AbsolutePath)"
                $handler = $this.Routes[$key]
                
                if ($handler) {
                    $body = $null
                    if ($req.HasEntityBody) {
                        $reader = [System.IO.StreamReader]::new($req.InputStream)
                        $body   = $reader.ReadToEnd() | ConvertFrom-Json
                        $reader.Dispose()
                    }
                    $result = & $handler $req $body
                    $json   = $result | ConvertTo-Json -Depth 10
                    $bytes  = [System.Text.Encoding]::UTF8.GetBytes($json)
                    $res.ContentType    = 'application/json'
                    $res.ContentLength64 = $bytes.Length
                    $res.OutputStream.Write($bytes, 0, $bytes.Length)
                } else {
                    $res.StatusCode = 404
                    $err = [System.Text.Encoding]::UTF8.GetBytes('{"error":"Not Found"}')
                    $res.OutputStream.Write($err, 0, $err.Length)
                }
            } catch {
                $res.StatusCode = 500
            } finally {
                $res.Close()
            }
        }
    }
    
    [void] Stop() {
        $this.Running = $false
        $this.Listener.Stop()
    }
}

$server = [RestApiServer]::new(@('http://localhost:8080/'))

$server.Route('GET', '/health', {
    param($req, $body)
    @{ status = 'healthy'; timestamp = [datetime]::UtcNow.ToString('o') }
})

$server.Route('GET', '/users', {
    param($req, $body)
    @{
        users = @(
            @{ id=1; name='Alice'; email='alice@example.com' }
            @{ id=2; name='Bob';   email='bob@example.com'   }
        )
        total = 2
    }
})

$server.Route('POST', '/users', {
    param($req, $body)
    @{ id=(Get-Random -Maximum 9999); name=$body.name; email=$body.email; created=$true }
})

Start-Job { $server.Start() }
```

---

## 2. Service Discovery Pattern

```powershell
class ServiceRegistry {
    hidden [hashtable]$_services = @{}
    
    [void] Register([string]$name, [string]$host, [int]$port) {
        if (-not $this._services[$name]) { $this._services[$name] = @() }
        $this._services[$name] += @{ host=$host; port=$port; healthy=$true; registered=[datetime]::Now }
        Write-Host "Registered $name at ${host}:$port"
    }
    
    [hashtable] Discover([string]$name) {
        $instances = $this._services[$name] | Where-Object { $_.healthy }
        if (-not $instances) { throw "Service '$name' not found" }
        return $instances[(Get-Random -Maximum $instances.Count)]
    }
    
    [void] HealthCheck() {
        foreach ($svc in $this._services.Keys) {
            foreach ($inst in $this._services[$svc]) {
                try {
                    $inst.healthy   = Test-NetConnection -ComputerName $inst.host -Port $inst.port -InformationLevel Quiet
                    $inst.lastCheck = [datetime]::Now
                } catch {
                    $inst.healthy = $false
                }
            }
        }
    }
}

$registry = [ServiceRegistry]::new()
$registry.Register('user-service',    'localhost', 8081)
$registry.Register('product-service', 'localhost', 8082)

$userSvc = $registry.Discover('user-service')
$users   = Invoke-RestMethod "http://$($userSvc.host):$($userSvc.port)/users"
```

---

## 3. Event-Driven Messaging

```powershell
# HTTP-based event bus
function Publish-Event {
    param([string]$EventType, [hashtable]$Data, [string[]]$Subscribers)
    
    $event = @{
        id        = [Guid]::NewGuid().ToString()
        type      = $EventType
        timestamp = [datetime]::UtcNow.ToString('o')
        data      = $Data
    } | ConvertTo-Json -Depth 10
    
    foreach ($url in $Subscribers) {
        try {
            Invoke-RestMethod -Uri $url -Method POST -Body $event -ContentType 'application/json'
            Write-Host "Published $EventType to $url" -ForegroundColor Green
        } catch {
            Write-Warning "Failed to publish to $url: $_"
        }
    }
}

Publish-Event -EventType 'order.created' `
    -Data @{ orderId=12345; amount=99.99 } `
    -Subscribers @('http://shipping:8084/events', 'http://invoice:8085/events')
```

---

## 4. API Gateway Pattern

```powershell
class ApiGateway {
    [hashtable]$Routes = @{}
    
    [void] AddRoute([string]$prefix, [string]$targetBase) {
        $this.Routes[$prefix] = $targetBase
    }
    
    [string] Proxy([string]$method, [string]$path, [string]$body = $null) {
        $route = $this.Routes.GetEnumerator() |
            Where-Object { $path.StartsWith($_.Key) } |
            Select-Object -First 1
        
        if (-not $route) { throw "No route for $path" }
        
        $targetUrl = "$($route.Value)$($path.Substring($route.Key.Length))"
        $params    = @{ Uri=$targetUrl; Method=$method; ContentType='application/json' }
        if ($body) { $params['Body'] = $body }
        
        return (Invoke-RestMethod @params) | ConvertTo-Json -Depth 10
    }
}

$gw = [ApiGateway]::new()
$gw.AddRoute('/api/users',    'http://user-service:8081')
$gw.AddRoute('/api/products', 'http://product-service:8082')
$gw.AddRoute('/api/orders',   'http://order-service:8083')

$response = $gw.Proxy('GET', '/api/users/123', $null)
```

---

**ก่อนหน้า ← [Part 58](Part-58.md) | ต่อไป → [Part 60: Kubernetes](Part-60.md)**
