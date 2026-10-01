# Part 27: Desired State Configuration (DSC)

> **ระดับ**: 🟠 Advanced | **เวลา**: ~4 ชั่วขนึ่ง

---

## 1. DSC คืออะไร?

DSC (Desired State Configuration) = การกำหนด "state" ที่ต้องการของเครื่อง Windows และให้ระบบจัดการให้มันตรงตามนั้นเอง

```powershell
# ตัวอย่าง DSC Configuration พื้นฐาน
configuration WebServerSetup {
    
    param (
        [string[]]$ComputerName = 'localhost'
    )
    
    Import-DscResource -ModuleName PSDesiredStateConfiguration
    
    Node $ComputerName {
        
        # ติดตั้ง IIS
        WindowsFeature IIS {
            Name   = 'Web-Server'
            Ensure = 'Present'    # ต้องมี
        }
        
        WindowsFeature ASP {
            Name      = 'Web-Asp-Net45'
            Ensure    = 'Present'
            DependsOn = '[WindowsFeature]IIS'
        }
        
        # ทำให้สะอาด
        File DefaultSite {
            DestinationPath = 'C:\inetpub\wwwroot\index.html'
            Contents        = '<h1>Hello from DSC!</h1>'
            Type            = 'File'
            Ensure          = 'Present'
            DependsOn       = '[WindowsFeature]IIS'
        }
        
        # Service running
        Service W3SVC {
            Name        = 'W3SVC'
            State       = 'Running'
            StartupType = 'Automatic'
            DependsOn   = '[WindowsFeature]IIS'
        }
        
        # Registry
        Registry DisableAutorun {
            Key       = 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\policies\Explorer'
            ValueName = 'NoDriveTypeAutoRun'
            ValueData = '255'
            ValueType = 'Dword'
            Ensure    = 'Present'
        }
    }
}

# Compile เป็น MOF file
WebServerSetup -ComputerName 'WebServer01'
# Creates: WebServerSetup\WebServer01.mof

# Apply
Start-DscConfiguration -Path '.\WebServerSetup' -Wait -Verbose

# Test current state
Test-DscConfiguration -Path '.\WebServerSetup' -Verbose

# Get current state
Get-DscConfiguration
```

---

## 2. Custom DSC Resource

```powershell
# DSC Resource module: MyApp/DSCResources/MyApp_WebApp/MyApp_WebApp.psm1

function Get-TargetResource {
    param (
        [Parameter(Mandatory)] [string]$Name,
        [string]$Port = '80',
        [string]$Path = 'C:\inetpub\apps'
    )
    
    $appPath = Join-Path $Path $Name
    $exists  = Test-Path $appPath
    
    return @{
        Name    = $Name
        Port    = $Port
        Path    = $appPath
        Ensure  = if ($exists) { 'Present' } else { 'Absent' }
    }
}

function Test-TargetResource {
    param (
        [Parameter(Mandatory)] [string]$Name,
        [string]$Port = '80',
        [string]$Path = 'C:\inetpub\apps',
        [ValidateSet('Present','Absent')]
        [string]$Ensure = 'Present'
    )
    
    $current = Get-TargetResource @PSBoundParameters
    return $current.Ensure -eq $Ensure
}

function Set-TargetResource {
    param (
        [Parameter(Mandatory)] [string]$Name,
        [string]$Port = '80',
        [string]$Path = 'C:\inetpub\apps',
        [ValidateSet('Present','Absent')]
        [string]$Ensure = 'Present'
    )
    
    $appPath = Join-Path $Path $Name
    
    if ($Ensure -eq 'Present') {
        if (!(Test-Path $appPath)) {
            New-Item $appPath -ItemType Directory -Force
            Write-Verbose "Created app directory: $appPath"
        }
        # Configure IIS application
    } else {
        if (Test-Path $appPath) {
            Remove-Item $appPath -Recurse -Force
            Write-Verbose "Removed app: $Name"
        }
    }
}

Export-ModuleMember -Function *-TargetResource
```

---

## 3. Configuration Data

```powershell
# Separate config data from configuration
$configData = @{
    AllNodes = @(
        @{
            NodeName = 'WebServer01'
            Role     = 'Web'
            Site     = 'MainSite'
        },
        @{
            NodeName = 'DbServer01'
            Role     = 'Database'
        },
        @{
            NodeName = '*'  # applies to all
            PSDscAllowPlainTextPassword = $false
        }
    )
    NonNodeData = @{
        AppVersion = '2.1.0'
        LogPath    = 'D:\Logs'
    }
}

configuration AppDeployment {
    param ([hashtable]$ConfigData)
    
    Import-DscResource -ModuleName PSDesiredStateConfiguration
    
    Node $AllNodes.Where({$_.Role -eq 'Web'}).NodeName {
        
        File AppFiles {
            SourcePath      = '\\fileserver\app'
            DestinationPath = 'C:\App'
            Recurse         = $true
            Ensure          = 'Present'
        }
        
        Script UpdateAppVersion {
            GetScript  = { return @{Result = Get-Content 'C:\App\version.txt' -ErrorAction SilentlyContinue} }
            TestScript = { 
                $ver = Get-Content 'C:\App\version.txt' -ErrorAction SilentlyContinue
                $ver -eq $using:ConfigData.NonNodeData.AppVersion
            }
            SetScript  = {
                $using:ConfigData.NonNodeData.AppVersion | Set-Content 'C:\App\version.txt'
            }
        }
    }
}

AppDeployment -ConfigData $configData
```

---

## 4. LCM (Local Configuration Manager)

```powershell
# ตั้งค่า LCM
[DSCLocalConfigurationManager()]
configuration LCMConfig {
    
    Node localhost {
        Settings {
            RefreshMode          = 'Pull'     # Pull หรือ Push
            ConfigurationMode    = 'ApplyAndAutoCorrect'
            RebootNodeIfNeeded   = $true
            RefreshFrequencyMins = 30
            AllowModuleOverwrite = $true
        }
        
        ConfigurationRepositoryWeb PullServer {
            ServerURL          = 'https://pullserver.example.com/PSDSCPullServer.svc'
            RegistrationKey    = '7c43e9f2-e7a4-4b6d-a235-c8a6b8c7d9f1'
            ConfigurationNames = @('WebServerSetup')
        }
    }
}

LCMConfig
Set-DscLocalConfigurationManager -Path '.\LCMConfig' -Verbose

# Check LCM status
Get-DscLocalConfigurationManager | Select-Object RefreshMode, ConfigurationMode, LCMState

# Force apply
Update-DscConfiguration -Wait -Verbose
```

---

**ก่อนหน้า ← [Part 26](Part-26.md) | ต่อไป → [Part 28: Scheduled Tasks](Part-28.md)**
