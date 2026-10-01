# Part 68: PowerShell Remoting Advanced

> **ระดับ**: 🔴 Advanced | **เวลา**: ~4 ชั่วโมง

---

## 1. PSSession Management

```powershell
# Create persistent sessions
$cred     = Get-Credential
$servers  = @('WEB01','WEB02','APP01','APP02','DB01')

$sessions = New-PSSession -ComputerName $servers -Credential $cred

# Run command across all servers
Invoke-Command -Session $sessions -ScriptBlock {
    [PSCustomObject]@{
        Server   = $env:COMPUTERNAME
        CPU      = [math]::Round((Get-Counter '\Processor(_Total)\% Processor Time').CounterSamples.CookedValue, 1)
        MemFreeGB= [math]::Round((Get-WmiObject Win32_OS).FreePhysicalMemory / 1MB, 1)
        Uptime   = (Get-Date) - (gcim Win32_OS).LastBootUpTime
    }
} | Format-Table Server, CPU, MemFreeGB, Uptime

# Deploy file to all servers
foreach ($session in $sessions) {
    Copy-Item -Path './deploy/app.zip' -ToSession $session -Destination 'C:\Deploy\app.zip'
    Invoke-Command -Session $session -ScriptBlock {
        Expand-Archive 'C:\Deploy\app.zip' -DestinationPath 'C:\inetpub\myapp' -Force
        Write-Host "Deployed on $env:COMPUTERNAME" -ForegroundColor Green
    }
}

# Clean up sessions
$sessions | Remove-PSSession
```

---

## 2. SSH Remoting (Cross-Platform)

```powershell
# PowerShell 7+ SSH remoting (Linux/Mac targets)
# Install OpenSSH on target: apt install openssh-server

# Connect via SSH
$linuxSession = New-PSSession -HostName 'linux-server.domain.com' -UserName 'admin' -SSHTransport

Invoke-Command -Session $linuxSession -ScriptBlock {
    Get-Process | Sort-Object CPU -Descending | Select-Object -First 5 |
        Select-Object Name, @{n='CPU';e={[math]::Round($_.CPU,1)}}, Id
}

# SSH key-based authentication
$sshKey  = 'C:\Users\admin\.ssh\id_rsa'
$session = New-PSSession `
    -HostName 'deploy-target.domain.com' `
    -UserName 'deployer' `
    -SSHTransport `
    -KeyFilePath $sshKey

# Run deployment script remotely
Invoke-Command -Session $session -FilePath './scripts/deploy-linux.ps1' `
    -ArgumentList @{ Version='2.1.0'; Env='production' }

$session | Remove-PSSession
```

---

## 3. JEA (Just Enough Administration)

```powershell
# Create JEA Role Capability file
$roleCapabilities = @"
@{
    RoleCapabilityVersion = '2.0'
    Author                = 'Security Team'
    CompanyName           = 'MyCompany'
    
    VisibleCmdlets = @(
        'Get-Service',
        'Get-Process',
        'Get-EventLog',
        @{ Name='Restart-Service'; Parameters=@(@{Name='Name'; ValidateSet='IIS','W3SVC'}) }
    )
    
    VisibleFunctions = @('Get-SystemInfo', 'Restart-WebService')
    
    FunctionDefinitions = @(
        @{
            Name        = 'Get-SystemInfo'
            ScriptBlock = { 
                [PSCustomObject]@{
                    ComputerName = \$env:COMPUTERNAME
                    CPU          = (Get-Counter '\\Processor(_Total)\\% Processor Time').CounterSamples.CookedValue
                    MemGB        = [math]::Round((gcim Win32_OS).FreePhysicalMemory/1MB, 1)
                }
            }
        },
        @{
            Name        = 'Restart-WebService'
            ScriptBlock = { Restart-Service W3SVC -Force }
        }
    )
    
    VisibleExternalCommands = @()
    VisibleProviders         = @('FileSystem')
}
"@

# Save role capability
$moduleDir = 'C:\Program Files\WindowsPowerShell\Modules\JEAWebOps'
New-Item -ItemType Directory -Path "$moduleDir\RoleCapabilities" -Force | Out-Null
$roleCapabilities | Set-Content "$moduleDir\RoleCapabilities\WebOperator.psrc"

# Create session configuration
$jeaConfig = @{
    SessionType          = 'RestrictedRemoteServer'
    RunAsVirtualAccount  = $true
    TranscriptDirectory  = 'C:\JEATranscripts'
    RoleDefinitions      = @{
        'DOMAIN\WebOperators' = @{ RoleCapabilities = 'WebOperator' }
    }
}

$configPath = 'C:\JEA\WebOps.pssc'
New-PSSessionConfigurationFile @jeaConfig -Path $configPath -Verbose

# Register JEA endpoint
Register-PSSessionConfiguration -Name 'JEA.WebOps' -Path $configPath -Force

# Test JEA as user
$jeaSession = New-PSSession -ComputerName localhost -ConfigurationName 'JEA.WebOps' -Credential $limitedCred
Get-PSSessionCapability -ConfigurationName 'JEA.WebOps' -Username 'DOMAIN\WebOperator1'
```

---

## 4. Remote Script Deployment Framework

```powershell
function Invoke-RemoteDeployment {
    param(
        [string[]]$Servers,
        [string]$ScriptPath,
        [hashtable]$Parameters  = @{},
        [PSCredential]$Credential,
        [int]$MaxParallel        = 5,
        [int]$TimeoutSeconds     = 300
    )
    
    $results = [System.Collections.Concurrent.ConcurrentBag[psobject]]::new()
    
    $Servers | ForEach-Object -Parallel {
        $server  = $_
        $results = $using:results
        $cred    = $using:Credential
        $script  = $using:ScriptPath
        $params  = $using:Parameters
        
        $start  = [datetime]::Now
        $status = 'Unknown'
        $error  = $null
        
        try {
            $sessionOpts = New-PSSessionOption -OperationTimeout ($using:TimeoutSeconds * 1000)
            $session     = New-PSSession -ComputerName $server -Credential $cred `
                -SessionOption $sessionOpts -ErrorAction Stop
            
            $result  = Invoke-Command -Session $session -FilePath $script `
                -ArgumentList $params
            
            $session | Remove-PSSession
            $status = 'Success'
        } catch {
            $error  = $_.Exception.Message
            $status = 'Failed'
        }
        
        $results.Add([PSCustomObject]@{
            Server   = $server
            Status   = $status
            Duration = [math]::Round(([datetime]::Now - $start).TotalSeconds, 1)
            Error    = $error
        })
    } -ThrottleLimit $MaxParallel
    
    $all      = $results | Sort-Object Server
    $success  = @($all | Where-Object { $_.Status -eq 'Success' }).Count
    $failed   = @($all | Where-Object { $_.Status -eq 'Failed'  }).Count
    
    Write-Host "Deployment complete: $success OK, $failed failed" `
        -ForegroundColor $(if ($failed -eq 0) { 'Green' } else { 'Red' })
    $all | Format-Table
    return $all
}

Invoke-RemoteDeployment `
    -Servers  @('WEB01','WEB02','WEB03','WEB04') `
    -ScriptPath './deploy.ps1' `
    -Parameters @{ Version='2.1.0'; Rollback=$false } `
    -Credential $cred `
    -MaxParallel 4
```

---

**ก่อนหน้า ← [Part 67](Part-67.md) | ต่อไป → [Part 69: Advanced Pipeline Patterns](Part-69.md)**
