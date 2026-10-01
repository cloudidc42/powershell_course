# Part 54: Centralized Logging และ SIEM Integration

> **ระดับ**: 🟡 Professional | **เวลา**: ~4 ชั่วขนึ่ง

---

## 1. Structured Logging

```powershell
# JSON structured log (Serilog-style)
function Write-StructuredLog {
    param(
        [ValidateSet('DEBUG','INFO','WARN','ERROR','FATAL')]
        [string]$Level,
        [string]$Message,
        [hashtable]$Properties = @{},
        [string]$LogFile
    )
    
    $entry = @{
        '@timestamp' = (Get-Date).ToUniversalTime().ToString('o')
        level        = $Level
        message      = $Message
        host         = $env:COMPUTERNAME
        pid          = $PID
    } + $Properties
    
    $json = $entry | ConvertTo-Json -Compress
    
    if ($LogFile) {
        $json | Add-Content $LogFile
    } else {
        Write-Host $json
    }
}

Write-StructuredLog INFO 'User login' @{user='alice'; ip='192.168.1.5'; result='success'}
Write-StructuredLog ERROR 'DB connection failed' @{db='mydb'; error='timeout'; retry=3}
```

---

## 2. ELK Stack Integration

```powershell
# ส่ง logs ไป Elasticsearch
function Send-ToElasticsearch {
    param(
        [string]$ElasticsearchUrl = 'http://elasticsearch:9200',
        [string]$Index,
        [hashtable]$Document
    )
    
    $body = $Document | ConvertTo-Json -Depth 5 -Compress
    Invoke-RestMethod `
        -Uri "$ElasticsearchUrl/$Index/_doc" `
        -Method POST `
        -Body $body `
        -ContentType 'application/json'
}

# Batch bulk index
function Send-ElasticsearchBulk {
    param([string]$Url, [string]$Index, [array]$Documents)
    
    $bulk = ($Documents | ForEach-Object {
        '{"index":{"_index":"' + $Index + '"}}'
        $_ | ConvertTo-Json -Compress
    }) -join "`n"
    $bulk += "`n"
    
    Invoke-RestMethod `
        -Uri "$Url/_bulk" `
        -Method POST `
        -Body $bulk `
        -ContentType 'application/x-ndjson'
}

# การใช้งาน
$events = Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -MaxEvents 100 -EA SilentlyContinue |
    ForEach-Object {
        @{
            '@timestamp' = $_.TimeCreated.ToString('o')
            event_id     = $_.Id
            message      = $_.Message.Substring(0, [Math]::Min(500, $_.Message.Length))
            computer     = $_.MachineName
        }
    }

Send-ElasticsearchBulk -Url 'http://elk:9200' -Index 'winlogs-2024' -Documents $events
```

---

## 3. Splunk Integration

```powershell
# Splunk HTTP Event Collector (HEC)
function Send-ToSplunk {
    param(
        [string]$HecUrl = 'https://splunk:8088/services/collector',
        [string]$HecToken,
        [hashtable]$Event,
        [string]$Index    = 'main',
        [string]$Source   = 'powershell',
        [string]$Sourcetype = 'ps_structured'
    )
    
    $payload = @{
        time       = [DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
        host       = $env:COMPUTERNAME
        source     = $Source
        sourcetype = $Sourcetype
        index      = $Index
        event      = $Event
    } | ConvertTo-Json -Depth 10
    
    Invoke-RestMethod `
        -Uri $HecUrl `
        -Method POST `
        -Headers @{'Authorization' = "Splunk $HecToken"} `
        -Body $payload `
        -ContentType 'application/json' `
        -SkipCertificateCheck  # only for internal/test
}

$splunkToken = $env:SPLUNK_HEC_TOKEN
Send-ToSplunk -HecToken $splunkToken -Event @{
    action   = 'deploy'
    app      = 'myapp'
    version  = '2.1.0'
    user     = $env:USERNAME
    success  = $true
}
```

---

## 4. Syslog และ Fluentd

```powershell
# Syslog ผ่าน UDP
function Send-Syslog {
    param(
        [string]$Server = 'syslog.example.com',
        [int]   $Port   = 514,
        [string]$Message,
        [int]   $Facility = 1,   # user
        [int]   $Severity = 6    # info
    )
    
    $priority = $Facility * 8 + $Severity
    $hostname = $env:COMPUTERNAME
    $timestamp = Get-Date -Format 'MMM dd HH:mm:ss'
    $syslogMsg = "<$priority>$timestamp $hostname PowerShell: $Message"
    
    $udp    = [System.Net.Sockets.UdpClient]::new()
    $bytes  = [Text.Encoding]::ASCII.GetBytes($syslogMsg)
    $udp.Send($bytes, $bytes.Length, $Server, $Port)
    $udp.Close()
}

Send-Syslog -Server 'syslog.company.com' -Message "User $env:USERNAME ran deployment script"

# Fluentd ผ่าน HTTP
function Send-ToFluentd {
    param([string]$Url, [string]$Tag, [hashtable]$Record)
    Invoke-RestMethod `
        -Uri "$Url/$Tag" `
        -Method POST `
        -Body ($Record | ConvertTo-Json) `
        -ContentType 'application/json'
}

Send-ToFluentd -Url 'http://fluentd:24224' -Tag 'app.deploy' -Record @{
    level = 'info'
    msg   = 'Deployment complete'
    v     = '2.1.0'
}
```

---

**ก่อนหน้า ← [Part 53](Part-53.md) | ต่อไป → [Part 55: IIS Management](Part-55.md)**
