# Part 86: Cross-Platform PowerShell

> **ระดับ**: 🟠 World-Class | **เวลา**: ~4 ชั่วโมง

---

## 1. Platform Detection และ Conditional Logic

```powershell
# Platform detection
function Get-Platform {
    [CmdletBinding()]
    param()
    
    [PSCustomObject]@{
        OS           = if ($IsWindows) { 'Windows' } elseif ($IsLinux) { 'Linux' } else { 'macOS' }
        IsWindows    = $IsWindows
        IsLinux      = $IsLinux
        IsMacOS      = $IsMacOS
        PSVersion    = $PSVersionTable.PSVersion.ToString()
        PSEdition    = $PSVersionTable.PSEdition      # Desktop or Core
        Architecture = [System.Runtime.InteropServices.RuntimeInformation]::OSArchitecture.ToString()
        OSVersion    = [System.Environment]::OSVersion.VersionString
    }
}

$platform = Get-Platform
Write-Host "Running on: $($platform.OS) ($($platform.Architecture))"
Write-Host "PowerShell: v$($platform.PSVersion) $($platform.PSEdition)"

# Cross-platform script that runs on all OS
function Get-SystemInfo {
    $platform = Get-Platform
    
    if ($IsWindows) {
        $os   = Get-CimInstance Win32_OperatingSystem
        $cpu  = Get-CimInstance Win32_Processor | Select-Object -First 1
        return [PSCustomObject]@{
            OS        = $os.Caption
            CPUName   = $cpu.Name
            TotalRAMGB= [math]::Round($os.TotalVisibleMemorySize / 1MB, 2)
            FreeRAMGB = [math]::Round($os.FreePhysicalMemory      / 1MB, 2)
        }
    } elseif ($IsLinux) {
        $memInfo = Get-Content '/proc/meminfo'
        $total   = [regex]::Match(($memInfo | Where-Object { $_ -match 'MemTotal' }), '\d+').Value
        $free    = [regex]::Match(($memInfo | Where-Object { $_ -match 'MemAvailable' }), '\d+').Value
        $cpuName = (Get-Content '/proc/cpuinfo' | Where-Object { $_ -match 'model name' } | Select-Object -First 1) -replace '.*: '
        return [PSCustomObject]@{
            OS        = (uname -s)
            CPUName   = $cpuName
            TotalRAMGB= [math]::Round([int64]$total / 1MB, 2)
            FreeRAMGB = [math]::Round([int64]$free  / 1MB, 2)
        }
    } else {  # macOS
        $totalBytes = [int64](sysctl -n hw.memsize)
        return [PSCustomObject]@{
            OS        = (sw_vers -productName) + ' ' + (sw_vers -productVersion)
            CPUName   = (sysctl -n machdep.cpu.brand_string)
            TotalRAMGB= [math]::Round($totalBytes / 1GB, 2)
            FreeRAMGB = $null
        }
    }
}

Get-SystemInfo | Format-List
```

---

## 2. Cross-Platform Path Handling

```powershell
# WRONG: Windows-only path separator
# $logPath = 'C:\Logs\app.log'

# RIGHT: Cross-platform paths
function Get-AppPath {
    param(
        [ValidateSet('Logs','Config','Data','Temp')]
        [string]$Type = 'Data'
    )
    
    $base = if ($IsWindows) {
        switch ($Type) {
            'Config' { $env:APPDATA }
            'Logs'   { Join-Path $env:LOCALAPPDATA 'MyApp' 'Logs' }
            'Data'   { Join-Path $env:APPDATA 'MyApp' 'Data' }
            'Temp'   { $env:TEMP }
        }
    } elseif ($IsLinux) {
        switch ($Type) {
            'Config' { Join-Path $HOME '.config' 'myapp' }
            'Logs'   { '/var/log/myapp' }
            'Data'   { Join-Path $HOME '.local' 'share' 'myapp' }
            'Temp'   { '/tmp/myapp' }
        }
    } else {  # macOS
        switch ($Type) {
            'Config' { Join-Path $HOME 'Library' 'Application Support' 'MyApp' }
            'Logs'   { Join-Path $HOME 'Library' 'Logs' 'MyApp' }
            'Data'   { Join-Path $HOME 'Library' 'Application Support' 'MyApp' 'Data' }
            'Temp'   { Join-Path ([System.IO.Path]::GetTempPath()) 'myapp' }
        }
    }
    
    # Ensure directory exists
    if (-not (Test-Path $base)) {
        New-Item -ItemType Directory -Path $base -Force | Out-Null
    }
    return $base
}

# Always use [IO.Path]::Combine for joining
$logFile = [IO.Path]::Combine((Get-AppPath -Type 'Logs'), 'app.log')

# Or use Join-Path (also cross-platform)
$configFile = Join-Path (Get-AppPath -Type 'Config') 'settings.json'

Write-Host "Log: $logFile"
Write-Host "Config: $configFile"
```

---

## 3. Cross-Platform External Commands

```powershell
# Wrapping OS commands cross-platform
function Invoke-OpenFile {
    param([string]$Path)
    
    $resolved = Resolve-Path $Path
    if ($IsWindows) {
        Start-Process $resolved
    } elseif ($IsLinux) {
        & xdg-open $resolved
    } else {  # macOS
        & open $resolved
    }
}

function Get-ProcessInfo {
    param([string]$Name)
    
    if ($IsWindows) {
        Get-Process -Name $Name -ErrorAction SilentlyContinue |
            Select-Object Name, Id, CPU, @{n='MemMB'; e={[math]::Round($_.WorkingSet/1MB,1)}}
    } else {
        $pids = (pgrep -f $Name 2>/dev/null)
        if (-not $pids) { return }
        $pids | ForEach-Object {
            $pid  = $_
            $cmd  = (ps -p $pid -o comm= 2>/dev/null)
            $mem  = (ps -p $pid -o rss=  2>/dev/null)
            [PSCustomObject]@{
                Name  = $cmd.Trim()
                Id    = [int]$pid
                CPU   = $null
                MemMB = [math]::Round([int]$mem / 1024, 1)
            }
        }
    }
}

function Get-ServiceStatus {
    param([string]$Name)
    
    if ($IsWindows) {
        $svc = Get-Service -Name $Name -ErrorAction SilentlyContinue
        return [PSCustomObject]@{ Name=$Name; Status=$svc?.Status; Running=$svc?.Status -eq 'Running' }
    } elseif ($IsLinux) {
        $status = systemctl is-active $Name 2>/dev/null
        return [PSCustomObject]@{ Name=$Name; Status=$status.Trim(); Running=($status.Trim() -eq 'active') }
    } else {
        $status = launchctl list | Where-Object { $_ -match $Name } | Select-Object -First 1
        return [PSCustomObject]@{ Name=$Name; Status='unknown'; Running=$null -ne $status }
    }
}

# Cross-platform script block
$services = @('nginx','postgresql','redis') | ForEach-Object { Get-ServiceStatus $_ }
$services | Format-Table Name, Status, Running
```

---

## 4. Cross-Platform Module เต็ม Stack

```powershell
# Module that works on all platforms
class CrossPlatformHelper {
    static [string] GetPathSeparator() {
        return [System.IO.Path]::DirectorySeparatorChar.ToString()
    }
    
    static [string] GetNewLine() {
        return [System.Environment]::NewLine
    }
    
    static [string] GetUserHome() {
        if ($IsWindows) { return $env:USERPROFILE }
        return $HOME
    }
    
    static [string] GetTempDir() {
        return [System.IO.Path]::GetTempPath()
    }
    
    static [bool] CommandExists([string]$Command) {
        return $null -ne (Get-Command $Command -ErrorAction SilentlyContinue)
    }
    
    static [string] Which([string]$Command) {
        $cmd = Get-Command $Command -ErrorAction SilentlyContinue
        return $cmd?.Source
    }
    
    static [void] EnsureDir([string]$Path) {
        if (-not (Test-Path $Path)) {
            New-Item -ItemType Directory -Path $Path -Force | Out-Null
        }
    }
    
    static [string] ToAbsolutePath([string]$Path) {
        if ([System.IO.Path]::IsPathRooted($Path)) { return $Path }
        return [System.IO.Path]::GetFullPath(
            [System.IO.Path]::Combine((Get-Location).Path, $Path)
        )
    }
}

# Test cross-platform compatibility
$isAdmin = if ($IsWindows) {
    ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole(
        [Security.Principal.WindowsBuiltInRole]::Administrator
    )
} else {
    (id -u) -eq '0'
}

Write-Host "Home:    $([CrossPlatformHelper]::GetUserHome())"
Write-Host "Temp:    $([CrossPlatformHelper]::GetTempDir())"
Write-Host "IsAdmin: $isAdmin"
Write-Host "Git:     $([CrossPlatformHelper]::Which('git'))"
```

---

**ก่อนหน้า ← [Part 85](Part-85.md) | ต่อไป → [Part 87: PowerShell DSL Patterns](Part-87.md)**
