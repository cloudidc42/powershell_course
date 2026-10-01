# Part 75: Async Programming ด้วย PowerShell

> **ระดับ**: 🟠 World-Class | **เวลา**: ~4 ชั่วโมง

---

## 1. Runspace และ Task Parallel

```powershell
# True async with Runspaces
function Invoke-Async {
    param([scriptblock]$ScriptBlock, [object[]]$Arguments = @())
    
    $rs   = [runspacefactory]::CreateRunspace()
    $rs.Open()
    
    $ps   = [powershell]::Create()
    $ps.Runspace = $rs
    $ps.AddScript($ScriptBlock) | Out-Null
    foreach ($arg in $Arguments) { $ps.AddArgument($arg) | Out-Null }
    
    $handle = $ps.BeginInvoke()
    
    return [PSCustomObject]@{
        PS      = $ps
        Handle  = $handle
        RS      = $rs
    }
}

function Wait-Async {
    param([PSCustomObject]$Task, [int]$TimeoutMs = 30000)
    
    $done = $Task.Handle.AsyncWaitHandle.WaitOne($TimeoutMs)
    if (-not $done) {
        $Task.PS.Stop()
        throw [TimeoutException]::new('Async task timed out')
    }
    
    try {
        $result = $Task.PS.EndInvoke($Task.Handle)
        if ($Task.PS.Streams.Error.Count -gt 0) {
            throw $Task.PS.Streams.Error[0].Exception
        }
        return $result
    } finally {
        $Task.PS.Dispose()
        $Task.RS.Dispose()
    }
}

# Launch multiple tasks concurrently
$tasks = @(
    Invoke-Async { param($url) (Invoke-RestMethod $url).Count } @('https://api.example.com/users')
    Invoke-Async { param($url) (Invoke-RestMethod $url).Count } @('https://api.example.com/products')
    Invoke-Async { param($url) (Invoke-RestMethod $url).Count } @('https://api.example.com/orders')
)

$results = $tasks | ForEach-Object { Wait-Async $_ }
Write-Host "Users: $($results[0]), Products: $($results[1]), Orders: $($results[2])"
```

---

## 2. Async Queue Worker

```powershell
class AsyncWorkerPool {
    [int]$WorkerCount
    hidden [System.Collections.Concurrent.BlockingCollection[object]]$_queue
    hidden [System.Collections.Generic.List[powershell]]$_workers
    hidden [runspacepool]$_pool
    
    AsyncWorkerPool([int]$workers, [int]$queueSize = 1000) {
        $this.WorkerCount = $workers
        $this._queue      = [System.Collections.Concurrent.BlockingCollection[object]]::new($queueSize)
        $this._workers    = [System.Collections.Generic.List[powershell]]::new()
        $this._pool       = [runspacefactory]::CreateRunspacePool(1, $workers)
        $this._pool.Open()
    }
    
    [void] Start([scriptblock]$processor) {
        for ($i = 0; $i -lt $this.WorkerCount; $i++) {
            $ps = [powershell]::Create()
            $ps.RunspacePool = $this._pool
            $ps.AddScript({
                param($queue, $proc)
                while (-not $queue.IsCompleted) {
                    $item = $null
                    if ($queue.TryTake([ref]$item, 1000)) {
                        try { & $proc $item }
                        catch { Write-Warning "Worker error: $_" }
                    }
                }
            }) | Out-Null
            $ps.AddArgument($this._queue) | Out-Null
            $ps.AddArgument($processor) | Out-Null
            $ps.BeginInvoke() | Out-Null
            $this._workers.Add($ps)
        }
    }
    
    [void] Enqueue([object]$item) {
        $this._queue.Add($item)
    }
    
    [void] Stop() {
        $this._queue.CompleteAdding()
        Start-Sleep 2  # let workers drain
        $this._pool.Close()
    }
}

$pool = [AsyncWorkerPool]::new(4)
$pool.Start({
    param($item)
    Write-Host "Processing: $($item.Id)" -ForegroundColor Cyan
    Start-Sleep -Milliseconds (Get-Random -Minimum 100 -Maximum 500)
    # Process item...
})

# Enqueue work
1..50 | ForEach-Object { $pool.Enqueue(@{ Id=$_; Data="item-$_" }) }
$pool.Stop()
```

---

## 3. Async HTTP Server

```powershell
# Non-blocking HTTP server
function Start-AsyncHttpServer {
    param([string]$Prefix = 'http://localhost:8080/', [scriptblock]$Handler)
    
    $listener = [System.Net.HttpListener]::new()
    $listener.Prefixes.Add($Prefix)
    $listener.Start()
    Write-Host "Async server at $Prefix" -ForegroundColor Green
    
    # Handle requests asynchronously
    $callback = [System.AsyncCallback]{
        param($ar)
        $ctx = $listener.EndGetContext($ar)
        
        # Start listening for next request immediately
        if ($listener.IsListening) {
            $listener.BeginGetContext($callback, $null) | Out-Null
        }
        
        # Handle current request in background
        $req = $ctx.Request
        $res = $ctx.Response
        
        try {
            $body = $null
            if ($req.HasEntityBody) {
                $reader = [System.IO.StreamReader]::new($req.InputStream)
                $body   = $reader.ReadToEnd()
                $reader.Dispose()
            }
            
            $result = & $Handler $req.HttpMethod $req.Url.AbsolutePath $body
            $json   = $result | ConvertTo-Json -Depth 10
            $bytes  = [System.Text.Encoding]::UTF8.GetBytes($json)
            $res.ContentType    = 'application/json'
            $res.ContentLength64 = $bytes.Length
            $res.OutputStream.Write($bytes, 0, $bytes.Length)
        } catch {
            $res.StatusCode = 500
        } finally {
            $res.Close()
        }
    }
    
    $listener.BeginGetContext($callback, $null) | Out-Null
    
    return $listener  # caller controls lifetime
}

$server = Start-AsyncHttpServer 'http://localhost:8080/' {
    param($method, $path, $body)
    switch ("$method $path") {
        'GET /health' { @{ status='ok' } }
        'GET /time'   { @{ time=[datetime]::UtcNow.ToString('o') } }
        default       { @{ error='Not Found'; code=404 } }
    }
}

Write-Host 'Press Ctrl+C to stop...'
while ($server.IsListening) { Start-Sleep 1 }
$server.Stop()
```

---

**ก่อนหน้า ← [Part 74](Part-74.md) | ต่อไป → [Part 76: Event-Driven Architecture](Part-76.md)**
