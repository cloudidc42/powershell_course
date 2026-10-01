# Part 32: Networking และ TCP/IP

> **ระดับ**: 🟠 Advanced | **เวลา**: ~3 ชั่วขนึ่ง

---

## 1. Network Information

```powershell
# Network adapters
Get-NetAdapter | Where-Object Status -eq 'Up' |
    Select-Object Name, InterfaceDescription, LinkSpeed, MacAddress

# IP Configuration
Get-NetIPAddress -AddressFamily IPv4 |
    Where-Object { $_.IPAddress -notlike '169.*' } |
    Select-Object InterfaceAlias, IPAddress, PrefixLength

# Default Gateway
Get-NetRoute -DestinationPrefix '0.0.0.0/0' | Select-Object NextHop, InterfaceAlias

# DNS Servers
Get-DnsClientServerAddress -AddressFamily IPv4 | Select-Object InterfaceAlias, ServerAddresses

# Resolve DNS
Resolve-DnsName 'google.com' -Type A
Resolve-DnsName 'google.com' -Type MX

# Test connectivity
Test-NetConnection 'google.com' -Port 443
Test-NetConnection '192.168.1.1' -CommonTCPPort HTTP

# Ping แบบตั้งค่า
$result = Test-Connection 'google.com' -Count 4
$avg    = ($result.ResponseTime | Measure-Object -Average).Average
Write-Host "Average RTT: $([math]::Round($avg))ms"
```

---

## 2. Port Scanner

```powershell
function Invoke-PortScan {
    param(
        [string]   $Target,
        [int[]]    $Ports  = @(22,23,25,53,80,110,143,443,445,3389,8080),
        [int]      $Timeout = 500
    )
    
    $results = $Ports | ForEach-Object -Parallel {
        $port    = $_
        $target  = $using:Target
        $timeout = $using:Timeout
        try {
            $tcp = [System.Net.Sockets.TcpClient]::new()
            $ar  = $tcp.BeginConnect($target, $port, $null, $null)
            $ok  = $ar.AsyncWaitHandle.WaitOne($timeout)
            $tcp.Close()
            [PSCustomObject]@{ Port=$port; Status=if($ok){'Open'}else{'Closed'} }
        } catch {
            [PSCustomObject]@{ Port=$port; Status='Filtered' }
        }
    } -ThrottleLimit 20
    
    $results | Sort-Object Port
}

Invoke-PortScan -Target '192.168.1.1' | Where-Object Status -eq 'Open' | Format-Table
```

---

## 3. Network Monitoring

```powershell
# Active connections
Get-NetTCPConnection -State Established |
    Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort, OwningProcess |
    Sort-Object RemoteAddress | Format-Table -AutoSize

# Connections พร้อมชื่อ process
Get-NetTCPConnection -State Listen | ForEach-Object {
    $proc = Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue
    [PSCustomObject]@{
        Port       = $_.LocalPort
        PID        = $_.OwningProcess
        Process    = $proc.Name
        Path       = $proc.Path
    }
} | Sort-Object Port | Format-Table -AutoSize

# Firewall rules
Get-NetFirewallRule -Enabled True -Direction Inbound |
    Select-Object DisplayName, Action, Profile |
    Format-Table -AutoSize

# เพิ่ม/ลบ firewall rule
New-NetFirewallRule -DisplayName 'Allow SSH' -Protocol TCP -LocalPort 22 -Action Allow -Direction Inbound
Remove-NetFirewallRule -DisplayName 'Allow SSH'
```

---

## 4. TCP Client/Server

```powershell
# TCP Server
function Start-TcpEchoServer {
    param([int]$Port = 9999)
    $listener = [System.Net.Sockets.TcpListener]::new([System.Net.IPAddress]::Any, $Port)
    $listener.Start()
    Write-Host "Listening on port $Port... (Ctrl+C to stop)"
    try {
        while ($true) {
            $client = $listener.AcceptTcpClient()
            $remote = $client.Client.RemoteEndPoint
            Write-Host "Connection from $remote"
            $stream = $client.GetStream()
            $reader = [System.IO.StreamReader]::new($stream)
            $writer = [System.IO.StreamWriter]::new($stream)
            $writer.AutoFlush = $true
            $line = $reader.ReadLine()
            Write-Host "  Received: $line"
            $writer.WriteLine("Echo: $line")
            $client.Close()
        }
    } finally {
        $listener.Stop()
    }
}

# TCP Client
function Send-TcpMessage {
    param([string]$Host, [int]$Port, [string]$Message)
    $client = [System.Net.Sockets.TcpClient]::new($Host, $Port)
    $stream = $client.GetStream()
    $writer = [System.IO.StreamWriter]::new($stream)
    $reader = [System.IO.StreamReader]::new($stream)
    $writer.AutoFlush = $true
    $writer.WriteLine($Message)
    $response = $reader.ReadLine()
    $client.Close()
    return $response
}

$resp = Send-TcpMessage -Host 'localhost' -Port 9999 -Message 'Hello!'
Write-Host $resp
```

---

**ก่อนหน้า ← [Part 31](Part-31.md) | ต่อไป → [Part 33: Docker & Containers](Part-33.md)**
