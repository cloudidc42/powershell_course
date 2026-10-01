# Part 76: Event-Driven Architecture

> **ระดับ**: 🟠 World-Class | **เวลา**: ~4 ชั่วโมง

---

## 1. Event Bus Implementation

```powershell
class EventBus {
    hidden [hashtable]$_handlers = @{}
    hidden [System.Collections.Concurrent.ConcurrentQueue[hashtable]]$_queue
    hidden [bool]$_running = $false
    
    EventBus() {
        $this._queue = [System.Collections.Concurrent.ConcurrentQueue[hashtable]]::new()
    }
    
    [void] Subscribe([string]$eventType, [scriptblock]$handler) {
        if (-not $this._handlers[$eventType]) { $this._handlers[$eventType] = @() }
        $this._handlers[$eventType] += $handler
    }
    
    [void] Publish([string]$eventType, [hashtable]$data = @{}) {
        $event = @{
            Type      = $eventType
            Data      = $data
            Timestamp = [datetime]::UtcNow
            Id        = [Guid]::NewGuid().ToString('N').Substring(0,8)
        }
        $this._queue.Enqueue($event)
    }
    
    [void] Process([int]$MaxEvents = 100) {
        $count = 0
        while ($count -lt $MaxEvents) {
            $event = $null
            if (-not $this._queue.TryDequeue([ref]$event)) { break }
            
            $handlers = $this._handlers[$event.Type]
            if ($handlers) {
                foreach ($h in $handlers) {
                    try { & $h $event.Data $event }
                    catch { Write-Warning "Handler error for $($event.Type): $_" }
                }
            }
            
            # Wildcard handlers
            foreach ($h in $this._handlers['*']) {
                try { & $h $event.Data $event }
                catch { }
            }
            
            $count++
        }
    }
    
    [void] StartProcessingLoop([int]$intervalMs = 100) {
        $this._running = $true
        $bus = $this
        Register-EngineEvent -SourceIdentifier 'EventBus.Process' -Action {
            $bus.Process(50)
        } | Out-Null
        
        $timer = [System.Timers.Timer]::new($intervalMs)
        $timer.Add_Elapsed({
            New-Event -SourceIdentifier 'EventBus.Process'
        })
        $timer.Start()
    }
}

# Setup event bus
$bus = [EventBus]::new()

# Subscribe to events
$bus.Subscribe('user.created', {
    param($data, $event)
    Write-Host "New user: $($data.name) ($($event.Id))" -ForegroundColor Green
    # Send welcome email, create profile, etc.
})

$bus.Subscribe('order.placed', {
    param($data, $event)
    Write-Host "Order placed: #$($data.orderId) - $$($data.amount)" -ForegroundColor Cyan
})

$bus.Subscribe('order.placed', {
    param($data, $event)
    # Second handler for same event - trigger inventory check
    Write-Host "Checking inventory for order $($data.orderId)" -ForegroundColor Yellow
})

# Audit all events
$bus.Subscribe('*', {
    param($data, $event)
    "[$($event.Timestamp.ToString('HH:mm:ss'))] $($event.Type) - $($event.Id)" | 
        Add-Content './events.log'
})

# Publish events
$bus.Publish('user.created', @{ name='Alice'; email='alice@example.com'; id=42 })
$bus.Publish('order.placed', @{ orderId=12345; customerId=42; amount=99.99 })

# Process
$bus.Process()
```

---

## 2. FileSystem Watcher

```powershell
function Start-FolderWatcher {
    param(
        [string]$Path,
        [string]$Filter  = '*.*',
        [hashtable]$Handlers = @{}
    )
    
    $watcher = [System.IO.FileSystemWatcher]::new($Path, $Filter)
    $watcher.IncludeSubdirectories = $true
    $watcher.NotifyFilter = [System.IO.NotifyFilters]'FileName,LastWrite,DirectoryName'
    
    foreach ($eventType in @('Created','Changed','Deleted','Renamed')) {
        $handler = $Handlers[$eventType]
        if ($handler) {
            $watcher."Add_$eventType"({
                param($src, $e)
                & $handler $e
            })
        }
    }
    
    $watcher.EnableRaisingEvents = $true
    Write-Host "Watching: $Path" -ForegroundColor Green
    return $watcher
}

$watcher = Start-FolderWatcher -Path 'C:\Deploy\Incoming' -Handlers @{
    Created = {
        param($e)
        Write-Host "New file: $($e.FullPath)" -ForegroundColor Green
        
        # Process new deployment package
        if ($e.FullPath -match '\.zip$') {
            Write-Host "Deploying $($e.Name)..." -ForegroundColor Yellow
            Expand-Archive $e.FullPath -DestinationPath 'C:\Deploy\Active' -Force
            Move-Item $e.FullPath 'C:\Deploy\Archive\'
        }
    }
    Deleted = {
        param($e)
        Write-Host "Deleted: $($e.Name)" -ForegroundColor Red
    }
}

# Keep watching
Write-Host 'Watching for files... Press Ctrl+C to stop'
try {
    while ($true) { Start-Sleep 1 }
} finally {
    $watcher.Dispose()
}
```

---

## 3. Redis Pub/Sub

```powershell
# Redis messaging (StackExchange.Redis)
Add-Type -Path 'C:\tools\StackExchange.Redis.dll'

$redis = [StackExchange.Redis.ConnectionMultiplexer]::Connect($env:REDIS_CONN)
$pub   = $redis.GetSubscriber()
$db    = $redis.GetDatabase()

# Subscribe to channel
$pub.Subscribe('notifications', {
    param($channel, $message)
    $event = $message.ToString() | ConvertFrom-Json
    Write-Host "[$($event.type)] $($event.message)" -ForegroundColor Yellow
})

# Publish to channel
function Send-RedisEvent {
    param([string]$Channel, [string]$Type, [string]$Message, [hashtable]$Data = @{})
    
    $payload = @{
        type      = $Type
        message   = $Message
        data      = $Data
        timestamp = [datetime]::UtcNow.ToString('o')
    } | ConvertTo-Json -Compress
    
    $pub.Publish($Channel, $payload) | Out-Null
}

Send-RedisEvent -Channel 'notifications' -Type 'deploy' `
    -Message 'Production deploy complete' `
    -Data @{ version='2.1.0'; deployedBy='CI/CD' }

# Stream-based events (Redis Streams)
function Add-RedisStreamEvent {
    param([string]$Stream, [hashtable]$Fields)
    $entries = $Fields.GetEnumerator() | ForEach-Object {
        [StackExchange.Redis.NameValueEntry]::new($_.Key, $_.Value)
    }
    return $db.StreamAdd($Stream, $entries)
}

function Get-RedisStreamEvents {
    param([string]$Stream, [string]$From = '0-0', [int]$Count = 100)
    return $db.StreamRange($Stream, $From, '+', $Count)
}

Add-RedisStreamEvent 'app-events' @{
    type    = 'order.placed'
    orderId = '12345'
    amount  = '99.99'
}
```

---

**ก่อนหน้า ← [Part 75](Part-75.md) | ต่อไป → [Part 77: Machine Learning Integration](Part-77.md)**
