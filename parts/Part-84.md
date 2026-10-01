# Part 84: Advanced OOP Patterns ใน PowerShell

> **ระดับ**: 🟠 World-Class | **เวลา**: ~5 ชั่วโมง

---

## 1. Inheritance และ Abstract Classes

```powershell
# Abstract base class pattern
class StorageProvider {
    [string]$Name
    [string]$Region
    
    StorageProvider([string]$name, [string]$region) {
        $this.Name   = $name
        $this.Region = $region
        # Cannot instantiate abstract class
        if ($this.GetType().Name -eq 'StorageProvider') {
            throw [InvalidOperationException]::new('StorageProvider is abstract')
        }
    }
    
    # Abstract methods - must be overridden
    [void] Upload([string]$key, [byte[]]$data) { throw [NotImplementedException]::new() }
    [byte[]] Download([string]$key)             { throw [NotImplementedException]::new() }
    [bool] Exists([string]$key)                 { throw [NotImplementedException]::new() }
    [void] Delete([string]$key)                 { throw [NotImplementedException]::new() }
    
    # Concrete methods (inherited)
    [void] UploadText([string]$key, [string]$text) {
        $this.Upload($key, [System.Text.Encoding]::UTF8.GetBytes($text))
    }
    
    [string] DownloadText([string]$key) {
        return [System.Text.Encoding]::UTF8.GetString($this.Download($key))
    }
    
    [string] ToString() { return "$($this.GetType().Name)($($this.Name)/$($this.Region))" }
}

# Azure Blob implementation
class AzureBlobStorage : StorageProvider {
    hidden [string]$_container
    hidden [object]$_client
    
    AzureBlobStorage([string]$connStr, [string]$container) : base('AzureBlob', 'eastus') {
        $this._container = $container
        # $this._client = Initialize Azure SDK client...
    }
    
    [void] Upload([string]$key, [byte[]]$data) {
        Write-Verbose "[Azure] Uploading $key ($($data.Length) bytes) to $($this._container)"
        # $this._client.UploadBlob($key, $data)
    }
    
    [byte[]] Download([string]$key) {
        Write-Verbose "[Azure] Downloading $key from $($this._container)"
        return @()
    }
    
    [bool] Exists([string]$key) {
        return $false  # Check Azure
    }
    
    [void] Delete([string]$key) {
        Write-Verbose "[Azure] Deleting $key"
    }
}

# S3 implementation
class S3Storage : StorageProvider {
    hidden [string]$_bucket
    
    S3Storage([string]$bucket, [string]$region) : base('S3', $region) {
        $this._bucket = $bucket
    }
    
    [void] Upload([string]$key, [byte[]]$data) {
        Write-Verbose "[S3] Uploading $key to s3://$($this._bucket)/$key"
        # Write-S3Object
    }
    
    [byte[]] Download([string]$key) {
        Write-Verbose "[S3] Downloading s3://$($this._bucket)/$key"
        return @()
    }
    
    [bool] Exists([string]$key) { return $false }
    [void] Delete([string]$key) { Write-Verbose "[S3] Deleting $key" }
}

# Polymorphic usage
[StorageProvider[]]$providers = @(
    [AzureBlobStorage]::new($env:AZURE_CONN, 'assets')
    [S3Storage]::new('my-bucket', 'us-east-1')
)

foreach ($provider in $providers) {
    Write-Host "Using: $provider"
    $provider.UploadText('hello.txt', 'Hello from PowerShell!')
}
```

---

## 2. Interface-like Patterns

```powershell
# PowerShell doesn't have interfaces, but we simulate them

# IHealthCheck interface simulation
class IHealthCheck {
    [hashtable] Check() { throw [NotImplementedException]::new('Check() must be implemented') }
    [bool] IsHealthy()   {
        try { return $this.Check().status -eq 'healthy' }
        catch { return $false }
    }
}

class DatabaseHealth : IHealthCheck {
    [string]$ConnectionString
    DatabaseHealth([string]$conn) { $this.ConnectionString = $conn }
    
    [hashtable] Check() {
        try {
            $conn = [System.Data.SqlClient.SqlConnection]::new($this.ConnectionString)
            $conn.Open()
            $conn.Close()
            return @{ status='healthy'; latencyMs=([datetime]::Now - [datetime]::Now).TotalMilliseconds }
        } catch {
            return @{ status='unhealthy'; error=$_.Exception.Message }
        }
    }
}

class ApiHealth : IHealthCheck {
    [string]$Url
    ApiHealth([string]$url) { $this.Url = $url }
    
    [hashtable] Check() {
        $start = [datetime]::Now
        try {
            $r = Invoke-RestMethod $this.Url -TimeoutSec 5
            return @{ status='healthy'; latencyMs=([datetime]::Now - $start).TotalMilliseconds }
        } catch {
            return @{ status='unhealthy'; error=$_.Exception.Message }
        }
    }
}

# Composite health check
$checks = @(
    [DatabaseHealth]::new($env:DB_CONN)
    [ApiHealth]::new('https://api.example.com/health')
    [ApiHealth]::new('https://payments.example.com/health')
)

$results = $checks | ForEach-Object {
    $r = $_.Check()
    [PSCustomObject]@{
        Type    = $_.GetType().Name
        Healthy = $_.IsHealthy()
        Details = $r
    }
}

$allHealthy = ($results | Where-Object { -not $_.Healthy }).Count -eq 0
Write-Host "Overall health: $(if ($allHealthy) {'HEALTHY'} else {'DEGRADED'})" -ForegroundColor $(if ($allHealthy) {'Green'} else {'Red'})
```

---

## 3. Strategy และ Decorator Patterns

```powershell
# Strategy pattern - interchangeable algorithms
class NotificationStrategy {
    [void] Send([string]$title, [string]$body) { throw [NotImplementedException]::new() }
}

class SlackStrategy : NotificationStrategy {
    [string]$Webhook
    SlackStrategy([string]$webhook) { $this.Webhook = $webhook }
    [void] Send([string]$title, [string]$body) {
        $payload = @{ text="*$title*`n$body" } | ConvertTo-Json
        Invoke-RestMethod -Uri $this.Webhook -Method POST -Body $payload -ContentType 'application/json'
    }
}

class EmailStrategy : NotificationStrategy {
    [string]$SmtpServer; [string]$From; [string[]]$To
    EmailStrategy([string]$smtp, [string]$from, [string[]]$to) {
        $this.SmtpServer=$smtp; $this.From=$from; $this.To=$to
    }
    [void] Send([string]$title, [string]$body) {
        Send-MailMessage -SmtpServer $this.SmtpServer -From $this.From -To $this.To -Subject $title -Body $body
    }
}

class Notifier {
    [NotificationStrategy[]]$Strategies = @()
    
    [Notifier] Add([NotificationStrategy]$strategy) {
        $this.Strategies += $strategy
        return $this
    }
    
    [void] Notify([string]$title, [string]$body) {
        foreach ($s in $this.Strategies) {
            try { $s.Send($title, $body) }
            catch { Write-Warning "Notification via $($s.GetType().Name) failed: $_" }
        }
    }
}

# Decorator: Throttled Notifier
class ThrottledNotifier {
    hidden [Notifier]$_inner
    hidden [hashtable]$_lastSent = @{}
    [int]$CooldownMinutes = 15
    
    ThrottledNotifier([Notifier]$inner) { $this._inner = $inner }
    
    [void] Notify([string]$key, [string]$title, [string]$body) {
        $last = $this._lastSent[$key]
        if ($last -and (([datetime]::Now - $last).TotalMinutes -lt $this.CooldownMinutes)) {
            Write-Verbose "Throttled: $key"
            return
        }
        $this._lastSent[$key] = [datetime]::Now
        $this._inner.Notify($title, $body)
    }
}

$notifier = [Notifier]::new()
$notifier.Add([SlackStrategy]::new($env:SLACK_WEBHOOK))
$notifier.Add([EmailStrategy]::new('smtp.company.com', 'alerts@company.com', @('ops@company.com')))

$throttled = [ThrottledNotifier]::new($notifier)
$throttled.Notify('cpu-alert', 'High CPU Alert', 'CPU usage exceeded 90% on PROD01')
```

---

**ก่อนหน้า ← [Part 83](Part-83.md) | ต่อไป → [Part 85: Plugin Architecture](Part-85.md)**
