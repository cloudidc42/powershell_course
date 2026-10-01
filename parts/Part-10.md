# Part 10: Functions & Parameters

> **ระดับ**: 🟡 Intermediate | **เวลร**: ~4 ชั่วโมง

---

## 1. ฟังก์ชันพื้นฐาน

```powershell
# Simple function
function Say-Hello {
    Write-Host "Hello, World!"
}
Say-Hello

# With parameter
function Say-Hello {
    param([string]$Name = "World")
    Write-Host "Hello, $Name!"
}
Say-Hello                  # Hello, World!
Say-Hello -Name "Alice"    # Hello, Alice!
Say-Hello "Alice"          # positional

# Return value
function Add-Numbers {
    param([int]$A, [int]$B)
    return $A + $B
    # หรือแค่
    # $A + $B
}

$result = Add-Numbers 5 3   # 8
$result = Add-Numbers -A 5 -B 3
```

---

## 2. Parameters แบบ Advanced

```powershell
function Invoke-DatabaseQuery {
    param(
        # บังคับให้ใส่ค่า
        [Parameter(Mandatory = $true, HelpMessage = "Database server name")]
        [string]$Server,
        
        # กำหนดค่าเริ่มต้น
        [Parameter(Mandatory = $false)]
        [string]$Database = "master",
        
        # จำกัด valid values
        [Parameter()]
        [ValidateSet('SELECT', 'UPDATE', 'DELETE', 'INSERT')]
        [string]$QueryType = 'SELECT',
        
        # จำกัด range
        [Parameter()]
        [ValidateRange(1, 10000)]
        [int]$Timeout = 30,
        
        # จำกัด pattern
        [Parameter()]
        [ValidatePattern('^[a-zA-Z0-9_-]+$')]
        [string]$Schema = 'dbo',
        
        # จำกัดความยาว
        [Parameter()]
        [ValidateLength(1, 500)]
        [string]$Query,
        
        # Switch (flag)
        [Parameter()]
        [switch]$DryRun,
        
        # SecureString
        [Parameter()]
        [securestring]$Password
    )
    
    Write-Host "Server: $Server, DB: $Database"
    Write-Host "Type: $QueryType, Timeout: ${Timeout}s"
    
    if ($DryRun) {
        Write-Host "[DRY RUN] Query: $Query"
        return
    }
    
    # actual query...
}

# การเรียกใช้
Invoke-DatabaseQuery -Server 'db01' -Query 'SELECT * FROM users' -DryRun
Invoke-DatabaseQuery -Server 'db01' -Database 'myapp' -QueryType 'SELECT' -Query 'SELECT 1'
```

---

## 3. CmdletBinding - Advanced Functions

```powershell
function Invoke-Deployment {
    [CmdletBinding(
        SupportsShouldProcess = $true,   # -WhatIf / -Confirm support
        ConfirmImpact = 'High'
    )]
    param(
        [Parameter(Mandatory, ValueFromPipeline)]
        [string]$Environment,
        
        [Parameter()]
        [string]$Version = 'latest',
        
        [Parameter()]
        [switch]$Force
    )
    
    begin {
        Write-Verbose "Starting deployment process"
        $deployed = 0
    }
    
    process {
        # ShouldProcess สนับสนุน -WhatIf และ -Confirm
        if ($PSCmdlet.ShouldProcess($Environment, "Deploy version $Version")) {
            Write-Verbose "Deploying $Version to $Environment"
            # actual deployment code
            Write-Host "Deployed $Version to $Environment" -ForegroundColor Green
            $deployed++
        }
    }
    
    end {
        Write-Verbose "Deployment complete. Total: $deployed environment(s)"
    }
}

# การใช้
"staging", "production" | Invoke-Deployment -Version "2.1.0" -WhatIf
# What if: Performing the operation "Deploy version 2.1.0" on target "staging".

"staging" | Invoke-Deployment -Version "2.1.0" -Verbose
# VERBOSE: Starting deployment process
# Deployed 2.1.0 to staging
# VERBOSE: Deployment complete. Total: 1 environment(s)
```

---

## 4. Pipeline Support

```powershell
function ConvertTo-UpperCase {
    [CmdletBinding()]
    param(
        [Parameter(ValueFromPipeline, ValueFromPipelineByPropertyName)]
        [string]$InputString
    )
    
    process {
        $InputString.ToUpper()
    }
}

# Pipeline input
"hello", "world" | ConvertTo-UpperCase
# HELLO\nWORLD

# ValueFromPipelineByPropertyName
function Format-User {
    [CmdletBinding()]
    param(
        [Parameter(ValueFromPipelineByPropertyName)]
        [string]$Name,
        
        [Parameter(ValueFromPipelineByPropertyName)]
        [int]$Age,
        
        [Parameter(ValueFromPipelineByPropertyName)]
        [string]$Email
    )
    process {
        "[$Name] Age: $Age | Email: $Email"
    }
}

@(
    [PSCustomObject]@{Name="Alice"; Age=30; Email="alice@example.com"}
    [PSCustomObject]@{Name="Bob";   Age=25; Email="bob@example.com"}
) | Format-User
```

---

## 5. Validation Attributes

```powershell
function Set-Config {
    param(
        # ต้องตรงกับ pattern
        [ValidatePattern('^[A-Z0-9_]+$')]
        [string]$Key,
        
        # เป็น set ที่กำหนด
        [ValidateSet('development', 'staging', 'production')]
        [string]$Environment,
        
        # จำนวนในช่วง
        [ValidateRange(1024, 65535)]
        [int]$Port,
        
        # ความยาว
        [ValidateLength(8, 128)]
        [string]$Password,
        
        # ไม่เป็น null/empty
        [ValidateNotNullOrEmpty()]
        [string]$ServerName,
        
        # Custom script validation
        [ValidateScript({
            if (Test-Path $_) { $true }
            else { throw "Path '$_' does not exist" }
        })]
        [string]$ConfigFile,
        
        # จำนวนตัวเลข
        [ValidateCount(1, 10)]
        [string[]]$Tags
    )
    
    Write-Host "Config set: $Key=$Environment:$Port"
}

# ทดสอบ validation
try {
    Set-Config -Key "DB_HOST" -Environment "invalid"  # Error!
} catch {
    Write-Host "Error: $_" -ForegroundColor Red
}
```

---

## 6. Default Values แบบ Dynamic

```powershell
function New-BackupJob {
    param(
        [string]$Source,
        
        # Default คำนวณเมื่อ run
        [string]$Destination = "$env:USERPROFILE\Backups\$(Get-Date -Format 'yyyy-MM-dd')",
        
        [string]$LogFile = "$env:TEMP\backup_$(Get-Date -Format 'HHmmss').log",
        
        [int]$MaxBackups = 7,
        
        [string]$CompressionLevel = 'Optimal'
    )
    
    Write-Host "Source: $Source"
    Write-Host "Dest: $Destination"
    Write-Host "Log: $LogFile"
}

New-BackupJob -Source 'C:\Projects'
```

---

## 7. Helper Patterns

```powershell
# Confirm pattern
function Remove-AllData {
    [CmdletBinding(SupportsShouldProcess, ConfirmImpact='High')]
    param([string]$Path)
    
    if ($PSCmdlet.ShouldProcess($Path, "Remove all data")) {
        Remove-Item $Path -Recurse -Force
        Write-Host "Removed: $Path"
    }
}

# Verbose logging pattern
function Invoke-Task {
    [CmdletBinding()]
    param([string]$Name)
    
    Write-Verbose "[START] $Name"
    $sw = [System.Diagnostics.Stopwatch]::StartNew()
    
    try {
        # task logic here
        Start-Sleep -Seconds 1
        Write-Verbose "[DONE] $Name in $($sw.ElapsedMilliseconds)ms"
    } catch {
        Write-Error "[FAIL] $Name: $_"
        throw
    }
}

Invoke-Task -Name "Deploy" -Verbose

# Recursive function
function Get-DirectorySize {
    param([string]$Path)
    
    $size = 0
    foreach ($item in Get-ChildItem $Path -Force) {
        if ($item.PSIsContainer) {
            $size += Get-DirectorySize $item.FullName
        } else {
            $size += $item.Length
        }
    }
    return $size
}

$sizeBytes = Get-DirectorySize 'C:\Windows\System32'
$sizeMB = [math]::Round($sizeBytes / 1MB, 2)
Write-Host "System32 size: $sizeMB MB"
```

---

## 8. Practical: Tool Functions

```powershell
# Test-Administrator
function Test-Administrator {
    $identity  = [System.Security.Principal.WindowsIdentity]::GetCurrent()
    $principal = [System.Security.Principal.WindowsPrincipal]$identity
    return $principal.IsInRole([System.Security.Principal.WindowsBuiltinRole]::Administrator)
}

if (!(Test-Administrator)) {
    Write-Warning "This script requires administrator privileges!"
    exit 1
}

# Get-ElapsedTime
function Measure-CommandTime {
    param(
        [Parameter(Mandatory)]
        [scriptblock]$Command,
        [int]$Iterations = 1
    )
    
    $times = [System.Collections.Generic.List[double]]::new()
    
    for ($i = 0; $i -lt $Iterations; $i++) {
        $sw = [System.Diagnostics.Stopwatch]::StartNew()
        & $Command | Out-Null
        $times.Add($sw.Elapsed.TotalMilliseconds)
    }
    
    [PSCustomObject]@{
        Iterations = $Iterations
        Min   = [math]::Round(($times | Measure-Object -Minimum).Minimum, 2)
        Max   = [math]::Round(($times | Measure-Object -Maximum).Maximum, 2)
        Avg   = [math]::Round(($times | Measure-Object -Average).Average, 2)
        Total = [math]::Round(($times | Measure-Object -Sum).Sum, 2)
    }
}

Measure-CommandTime { Get-Process } -Iterations 10
Measure-CommandTime { 1..1000 | ForEach-Object { $_ * $_ } } -Iterations 5
```

---

**ก่อนหน้า ← [Part 09](Part-09.md) | ต่อไป → [Part 11: Error Handling](Part-11.md)**
