# Part 20: Web Development ด้วย PowerShell

> **ระดับ**: 🟠 Advanced | **เวลา**: ~4 ชั่วโมง

---

## 1. HTTP Client - Invoke-WebRequest

```powershell
# GET request
$response = Invoke-WebRequest -Uri 'https://httpbin.org/get'
$response.StatusCode          # 200
$response.Headers             # hashtable
$response.Content             # body as string
$response.RawContentLength

# Parse JSON
$data = $response.Content | ConvertFrom-Json

# POST with body
$body = @{ name='Alice'; role='admin' } | ConvertTo-Json
$response = Invoke-WebRequest 'https://httpbin.org/post' \
    -Method POST \
    -Body $body \
    -ContentType 'application/json'

# Headers
$response = Invoke-WebRequest 'https://api.github.com/repos/PowerShell/PowerShell' \
    -Headers @{
        'Authorization' = "Bearer $token"
        'Accept' = 'application/vnd.github.v3+json'
        'User-Agent' = 'PowerShellCourse/1.0'
    }

# File download
Invoke-WebRequest 'https://example.com/file.zip' -OutFile 'file.zip'

# ติดตามความคืบหน้า
Invoke-WebRequest 'https://example.com/large.zip' \
    -OutFile 'large.zip' \
    -UserAgent 'PowerShell' \
    -DisableKeepAlive
```

---

## 2. Invoke-RestMethod

```powershell
# Invoke-RestMethod: auto-parses JSON/XML
$users = Invoke-RestMethod 'https://jsonplaceholder.typicode.com/users'
$users | Select-Object id, name, email

# GitHub API
$repo = Invoke-RestMethod 'https://api.github.com/repos/PowerShell/PowerShell' \
    -Headers @{ 'User-Agent' = 'PowerShellCourse' }

Write-Host "Stars: $($repo.stargazers_count)"
Write-Host "Forks: $($repo.forks_count)"
Write-Host "Lang: $($repo.language)"

# POST
$newTodo = Invoke-RestMethod 'https://jsonplaceholder.typicode.com/todos' \
    -Method POST \
    -Body (@{userId=1; title='Learn PowerShell'; completed=$false} | ConvertTo-Json) \
    -ContentType 'application/json'

Write-Host "Created ID: $($newTodo.id)"

# Pagination
function Get-AllPages {
    param([string]$Url, [int]$MaxPages = 10)
    
    $page = 1
    $all  = [System.Collections.Generic.List[PSObject]]::new()
    
    do {
        $pagedUrl = "$Url&page=$page&per_page=100"
        $items = Invoke-RestMethod $pagedUrl -ErrorAction Stop
        if (!$items) { break }
        $items | ForEach-Object { $all.Add($_) }
        $page++
    } while ($items.Count -eq 100 -and $page -le $MaxPages)
    
    return $all
}
```

---

## 3. HTTP Listener (Simple Server)

```powershell
# PowerShell เป็น HTTP Server!
function Start-SimpleHttpServer {
    param(
        [int]$Port = 8080,
        [string]$RootPath = '.'
    )
    
    $listener = [System.Net.HttpListener]::new()
    $listener.Prefixes.Add("http://+:$Port/")
    $listener.Start()
    
    Write-Host "Server started on port $Port (Ctrl+C to stop)" -ForegroundColor Green
    
    try {
        while ($listener.IsListening) {
            $context = $listener.GetContext()  # blocks
            $request  = $context.Request
            $response = $context.Response
            
            $method = $request.HttpMethod
            $url    = $request.Url.AbsolutePath
            
            Write-Host "$method $url"
            
            # Route
            $body = switch ($url) {
                '/'         { '<h1>Hello from PowerShell!</h1>' }
                '/health'   { '{"status":"ok","time":"' + (Get-Date -Format 'o') + '"}' }
                '/process'  { Get-Process | Select-Object Name,CPU | ConvertTo-Json }
                default     { "404 Not Found: $url" }
            }
            
            $statusCode = if ($url -match '^/(|health|process)$') { 200 } else { 404 }
            
            $bytes = [System.Text.Encoding]::UTF8.GetBytes($body)
            $response.StatusCode     = $statusCode
            $response.ContentLength64 = $bytes.Length
            
            if ($url -match '\.json$|/health|/process') {
                $response.ContentType = 'application/json'
            } else {
                $response.ContentType = 'text/html; charset=utf-8'
            }
            
            $response.OutputStream.Write($bytes, 0, $bytes.Length)
            $response.OutputStream.Close()
        }
    } finally {
        $listener.Stop()
        $listener.Close()
        Write-Host "Server stopped"
    }
}

# Start-SimpleHttpServer -Port 8080
```

---

## 4. Router Pattern

```powershell
class HttpRouter {
    hidden [System.Collections.Generic.List[hashtable]]$Routes
    
    HttpRouter() {
        $this.Routes = [System.Collections.Generic.List[hashtable]]::new()
    }
    
    [void] Get([string]$Pattern, [scriptblock]$Handler) {
        $this.Routes.Add(@{Method='GET'; Pattern=$Pattern; Handler=$Handler})
    }
    
    [void] Post([string]$Pattern, [scriptblock]$Handler) {
        $this.Routes.Add(@{Method='POST'; Pattern=$Pattern; Handler=$Handler})
    }
    
    [void] Delete([string]$Pattern, [scriptblock]$Handler) {
        $this.Routes.Add(@{Method='DELETE'; Pattern=$Pattern; Handler=$Handler})
    }
    
    [hashtable] Match([string]$Method, [string]$Path) {
        foreach ($route in $this.Routes) {
            if ($route.Method -ne $Method) { continue }
            
            # Convert pattern to regex: /users/:id -> /users/([^/]+)
            $pattern = '^' + ($route.Pattern -replace ':([a-zA-Z]+)', '(?<$1>[^/]+)') + '$'
            
            if ($Path -match $pattern) {
                $params = @{}
                $Matches.Keys | Where-Object { $_ -ne '0' } | ForEach-Object {
                    $params[$_] = $Matches[$_]
                }
                return @{ Handler=$route.Handler; Params=$params; Found=$true }
            }
        }
        return @{ Found=$false }
    }
}

# สร้าง app ด้วย router
$router = [HttpRouter]::new()
$users = @{}

$router.Get('/users', {
    $users.Values | ConvertTo-Json
})

$router.Get('/users/:id', {
    param($params)
    $id = $params['id']
    if ($users.ContainsKey($id)) { $users[$id] | ConvertTo-Json }
    else { '{}' }
})

$router.Post('/users', {
    param($params, $body)
    $id = [guid]::NewGuid().ToString('N').Substring(0,8)
    $data = $body | ConvertFrom-Json
    $users[$id] = @{ id=$id; name=$data.name; email=$data.email }
    $users[$id] | ConvertTo-Json
})
```

---

## 5. WebSocket Client

```powershell
# WebSocket client
$uri = [System.Uri]::new('wss://echo.websocket.org')
$ws  = [System.Net.WebSockets.ClientWebSocket]::new()

$cts = [System.Threading.CancellationTokenSource]::new()
$ws.ConnectAsync($uri, $cts.Token).Wait()

# Send
$msg   = [System.Text.Encoding]::UTF8.GetBytes('Hello WebSocket!')
$seg   = [System.ArraySegment[byte]]::new($msg)
$ws.SendAsync($seg, [System.Net.WebSockets.WebSocketMessageType]::Text, $true, $cts.Token).Wait()

# Receive
$buffer = New-Object byte[] 4096
$recv   = [System.ArraySegment[byte]]::new($buffer)
$result = $ws.ReceiveAsync($recv, $cts.Token).Result
$text   = [System.Text.Encoding]::UTF8.GetString($buffer, 0, $result.Count)
Write-Host "Echo: $text"

$ws.CloseAsync(
    [System.Net.WebSockets.WebSocketCloseStatus]::NormalClosure,
    'Bye',
    $cts.Token
).Wait()
$ws.Dispose()
```

---

**ก่อนหน้า ← [Part 19](Part-19.md) | ต่อไป → [Part 21: Full HTTP Server](Part-21.md)**
