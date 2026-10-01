# Part 21: สร้าง HTTP Server เต็มรูปแบบ

> **ระดับ**: 🟠 Advanced | **เวลา**: ~5 ชั่วโมง

---

## 1. Full Web Framework Structure

```powershell
# WebApp.psm1 - กรอบแบบ web framework

class HttpContext {
    [System.Net.HttpListenerRequest]  $Request
    [System.Net.HttpListenerResponse] $Response
    [hashtable] $RouteParams
    [hashtable] $QueryParams
    [PSObject]  $Body
    [hashtable] $State   # shared state bag
    
    HttpContext($req, $res) {
        $this.Request     = $req
        $this.Response    = $res
        $this.RouteParams = @{}
        $this.QueryParams = @{}
        $this.State       = @{}
        $this._ParseQuery()
    }
    
    hidden [void] _ParseQuery() {
        foreach ($key in $this.Request.QueryString.AllKeys) {
            if ($key) { $this.QueryParams[$key] = $this.Request.QueryString[$key] }
        }
    }
    
    [string] ReadBody() {
        $reader = [System.IO.StreamReader]::new($this.Request.InputStream)
        return $reader.ReadToEnd()
    }
    
    [PSObject] ReadJson() {
        return $this.ReadBody() | ConvertFrom-Json
    }
    
    [void] Json([object]$data, [int]$status = 200) {
        $json  = $data | ConvertTo-Json -Depth 10 -Compress
        $bytes = [System.Text.Encoding]::UTF8.GetBytes($json)
        $this.Response.StatusCode     = $status
        $this.Response.ContentType    = 'application/json; charset=utf-8'
        $this.Response.ContentLength64 = $bytes.Length
        $this.Response.AddHeader('X-Powered-By', 'PowerShell')
        $this.Response.OutputStream.Write($bytes, 0, $bytes.Length)
        $this.Response.OutputStream.Close()
    }
    
    [void] Html([string]$html, [int]$status = 200) {
        $bytes = [System.Text.Encoding]::UTF8.GetBytes($html)
        $this.Response.StatusCode     = $status
        $this.Response.ContentType    = 'text/html; charset=utf-8'
        $this.Response.ContentLength64 = $bytes.Length
        $this.Response.OutputStream.Write($bytes, 0, $bytes.Length)
        $this.Response.OutputStream.Close()
    }
    
    [void] Text([string]$text, [int]$status = 200) {
        $bytes = [System.Text.Encoding]::UTF8.GetBytes($text)
        $this.Response.StatusCode     = $status
        $this.Response.ContentType    = 'text/plain; charset=utf-8'
        $this.Response.ContentLength64 = $bytes.Length
        $this.Response.OutputStream.Write($bytes, 0, $bytes.Length)
        $this.Response.OutputStream.Close()
    }
    
    [void] Redirect([string]$url, [int]$status = 302) {
        $this.Response.StatusCode   = $status
        $this.Response.RedirectLocation = $url
        $this.Response.OutputStream.Close()
    }
    
    [void] NotFound([string]$message = 'Not Found') {
        $this.Json(@{ error=$message; code=404 }, 404)
    }
    
    [void] BadRequest([string]$message = 'Bad Request') {
        $this.Json(@{ error=$message; code=400 }, 400)
    }
    
    [void] InternalError([string]$message = 'Internal Server Error') {
        $this.Json(@{ error=$message; code=500 }, 500)
    }
}
```

---

## 2. Middleware System

```powershell
class WebApp {
    hidden [System.Collections.Generic.List[hashtable]] $Routes
    hidden [System.Collections.Generic.List[scriptblock]] $Middleware
    hidden [System.Net.HttpListener] $Listener
    [bool] $Running = $false
    
    WebApp() {
        $this.Routes     = [System.Collections.Generic.List[hashtable]]::new()
        $this.Middleware = [System.Collections.Generic.List[scriptblock]]::new()
    }
    
    # Middleware
    [void] Use([scriptblock]$mw) {
        $this.Middleware.Add($mw)
    }
    
    # Routes
    [void] Get   ([string]$p, [scriptblock]$h) { $this._Route('GET',    $p, $h) }
    [void] Post  ([string]$p, [scriptblock]$h) { $this._Route('POST',   $p, $h) }
    [void] Put   ([string]$p, [scriptblock]$h) { $this._Route('PUT',    $p, $h) }
    [void] Delete([string]$p, [scriptblock]$h) { $this._Route('DELETE', $p, $h) }
    [void] Patch ([string]$p, [scriptblock]$h) { $this._Route('PATCH',  $p, $h) }
    
    hidden [void] _Route([string]$method, [string]$pattern, [scriptblock]$handler) {
        $regex = '^' + ($pattern -replace ':([a-zA-Z_]+)', '(?<$1>[^/]+)') + '/?$'
        $this.Routes.Add(@{ Method=$method; Pattern=$pattern; Regex=$regex; Handler=$handler })
    }
    
    hidden [hashtable] _Match([string]$method, [string]$path) {
        foreach ($r in $this.Routes) {
            if ($r.Method -ne $method) { continue }
            if ($path -match $r.Regex) {
                $params = @{}
                $Matches.GetEnumerator() | Where-Object { $_.Key -ne '0' } | ForEach-Object {
                    $params[$_.Key] = $_.Value
                }
                return @{ Route=$r; Params=$params; Found=$true }
            }
        }
        return @{ Found=$false }
    }
    
    [void] Listen([int]$Port) {
        $this.Listener = [System.Net.HttpListener]::new()
        $this.Listener.Prefixes.Add("http://+:$Port/")
        $this.Listener.Start()
        $this.Running = $true
        
        Write-Host "[WebApp] Listening on http://localhost:$Port" -ForegroundColor Cyan
        
        while ($this.Running) {
            try {
                $ctx = $this.Listener.GetContext()
                $this._HandleRequest($ctx)
            } catch [System.Net.HttpListenerException] {
                break
            } catch {
                Write-Warning "Error: $_"
            }
        }
    }
    
    hidden [void] _HandleRequest($rawCtx) {
        $ctx = [HttpContext]::new($rawCtx.Request, $rawCtx.Response)
        $method = $ctx.Request.HttpMethod
        $path   = $ctx.Request.Url.AbsolutePath
        
        # Run middleware
        foreach ($mw in $this.Middleware) {
            & $mw $ctx
            if (!$ctx.Response.OutputStream.CanWrite) { return }
        }
        
        # Match route
        $match = $this._Match($method, $path)
        
        if ($match.Found) {
            $ctx.RouteParams = $match.Params
            try {
                & $match.Route.Handler $ctx
            } catch {
                $ctx.InternalError("$_")
            }
        } else {
            $ctx.NotFound("No route: $method $path")
        }
    }
    
    [void] Stop() {
        $this.Running = $false
        $this.Listener.Stop()
    }
}
```

---

## 3. ตัวอย่าง: Todo API

```powershell
# server.ps1
$app   = [WebApp]::new()
$todos = @{}
$nextId = 1

# Middleware: logging
$app.Use({
    param($ctx)
    $method = $ctx.Request.HttpMethod
    $path   = $ctx.Request.Url.AbsolutePath
    $time   = Get-Date -Format 'HH:mm:ss'
    Write-Host "[$time] $method $path" -ForegroundColor Gray
})

# Middleware: CORS
$app.Use({
    param($ctx)
    $ctx.Response.AddHeader('Access-Control-Allow-Origin', '*')
    $ctx.Response.AddHeader('Access-Control-Allow-Methods', 'GET,POST,PUT,DELETE,OPTIONS')
    $ctx.Response.AddHeader('Access-Control-Allow-Headers', 'Content-Type,Authorization')
    
    if ($ctx.Request.HttpMethod -eq 'OPTIONS') {
        $ctx.Response.StatusCode = 204
        $ctx.Response.OutputStream.Close()
    }
})

# Routes
$app.Get('/health', {
    param($ctx)
    $ctx.Json(@{ status='healthy'; time=(Get-Date -Format 'o'); version='1.0' })
})

$app.Get('/todos', {
    param($ctx)
    $list = $todos.Values | Sort-Object id
    $filter = $ctx.QueryParams['completed']
    if ($null -ne $filter) {
        $list = $list | Where-Object { $_.completed.ToString() -eq $filter }
    }
    $ctx.Json($list)
})

$app.Get('/todos/:id', {
    param($ctx)
    $id = $ctx.RouteParams['id']
    if ($todos.ContainsKey($id)) {
        $ctx.Json($todos[$id])
    } else {
        $ctx.NotFound("Todo $id not found")
    }
})

$app.Post('/todos', {
    param($ctx)
    $data = $ctx.ReadJson()
    if (!$data.title) { $ctx.BadRequest('title is required'); return }
    
    $id = ($script:nextId++).ToString()
    $todo = @{
        id        = $id
        title     = $data.title
        completed = $false
        createdAt = Get-Date -Format 'o'
    }
    $script:todos[$id] = $todo
    $ctx.Json($todo, 201)
})

$app.Put('/todos/:id', {
    param($ctx)
    $id = $ctx.RouteParams['id']
    if (!$todos.ContainsKey($id)) { $ctx.NotFound("Todo $id not found"); return }
    
    $data = $ctx.ReadJson()
    $todo = $script:todos[$id]
    if ($data.title)     { $todo.title = $data.title }
    if ($null -ne $data.completed) { $todo.completed = [bool]$data.completed }
    $todo.updatedAt = Get-Date -Format 'o'
    $ctx.Json($todo)
})

$app.Delete('/todos/:id', {
    param($ctx)
    $id = $ctx.RouteParams['id']
    if (!$todos.ContainsKey($id)) { $ctx.NotFound("Todo $id not found"); return }
    
    $todo = $todos[$id]
    $todos.Remove($id)
    $ctx.Json($todo)
})

# เริ่มเซิร์ฟเวอร์
Write-Host 'Todo API running at http://localhost:8080' -ForegroundColor Green
Write-Host 'Press Ctrl+C to stop'
$app.Listen(8080)
```

---

**ก่อนหน้า ← [Part 20](Part-20.md) | ต่อไป → [Part 22: Authentication & Security](Part-22.md)**
