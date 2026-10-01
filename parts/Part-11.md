# Part 11: Error Handling

> **ระดับ**: 🟡 Intermediate | **เวลา**: ~3 ชั่วโมง

---

## 1. Try/Catch/Finally

```powershell
# พื้นฐาน
try {
    $content = Get-Content 'C:\missing.txt' -ErrorAction Stop
    Write-Host "Content: $content"
} catch {
    Write-Host "Error: $_" -ForegroundColor Red
} finally {
    Write-Host "Always runs"
}

# ดัก exception type เฉพาะ
try {
    $result = 1/0
} catch [System.DivideByZeroException] {
    Write-Host "Division by zero!"
} catch [System.IO.FileNotFoundException] {
    Write-Host "File not found!"
} catch [System.UnauthorizedAccessException] {
    Write-Host "Access denied!"
} catch {
    Write-Host "General error: $($_.Exception.Message)"
} finally {
    Write-Host "Cleanup code"
}

# ErrorRecord properties
try {
    Get-Item 'X:\nonexistent' -ErrorAction Stop
} catch {
    Write-Host "Message : $($_.Exception.Message)"
    Write-Host "Type    : $($_.Exception.GetType().FullName)"
    Write-Host "Source  : $($_.InvocationInfo.ScriptName)"
    Write-Host "Line    : $($_.InvocationInfo.ScriptLineNumber)"
    Write-Host "Command : $($_.InvocationInfo.MyCommand)"
    Write-Host "Stack   :"
    Write-Host $_.ScriptStackTrace
}
```

---

## 2. ErrorAction และ $ErrorActionPreference

```powershell
# ErrorAction levels:
# Continue     - แสดง error และทำต่อ (default)
# SilentlyContinue - ซ่อน error และทำต่อ
# Stop         - หยุดและเข้า catch block
# Inquire      - ถาม user
# Ignore       - เหมือน SilentlyContinue

# Per-command
Get-Item 'missing.txt' -ErrorAction SilentlyContinue
Get-Item 'missing.txt' -ErrorAction Stop        # ทำให้ try/catch ใช้ได้
Get-Item 'missing.txt' -ErrorAction Continue    # default

# Global preference
$ErrorActionPreference = 'Stop'   # ทุก cmdlet throw exception
$ErrorActionPreference = 'Continue'  # restore default

# บันทึก error output
Get-Item 'missing.txt' -ErrorVariable myError -ErrorAction SilentlyContinue
if ($myError) {
    Write-Host "Error: $($myError[0].Exception.Message)"
}

# $Error automatic variable - stores all errors
Get-Item 'missing1.txt' -ErrorAction SilentlyContinue
Get-Item 'missing2.txt' -ErrorAction SilentlyContinue

$Error.Count   # number of errors
$Error[0]      # most recent error
$Error.Clear() # clear all
```

---

## 3. Throw และ Custom Exceptions

```powershell
# Throw string
function Validate-Age {
    param([int]$Age)
    if ($Age -lt 0)   { throw "Age cannot be negative" }
    if ($Age -gt 150) { throw "Age exceeds maximum" }
    return $Age
}

try { Validate-Age -100 } catch { Write-Host $_ }

# Throw Exception object
throw [System.ArgumentException]::new("Invalid argument: must be positive")

# Custom exception class
class AppException : System.Exception {
    [string]$ErrorCode
    [hashtable]$Details
    
    AppException([string]$message, [string]$code) : base($message) {
        $this.ErrorCode = $code
        $this.Details = @{}
    }
    
    AppException([string]$message, [string]$code, [hashtable]$details) : base($message) {
        $this.ErrorCode = $code
        $this.Details = $details
    }
}

class ValidationException : AppException {
    [string]$FieldName
    
    ValidationException([string]$field, [string]$message) : base(
        "Validation failed for '$field': $message", 
        "VALIDATION_ERROR"
    ) {
        $this.FieldName = $field
    }
}

# ใช้ custom exception
try {
    throw [ValidationException]::new('Email', 'Invalid format')
} catch [ValidationException] {
    Write-Host "Field: $($_.Exception.FieldName)"
    Write-Host "Code: $($_.Exception.ErrorCode)"
    Write-Host "Message: $($_.Exception.Message)"
} catch [AppException] {
    Write-Host "App error: $($_.Exception.ErrorCode)"
}
```

---

## 4. Error Handling Patterns

```powershell
# Retry pattern
function Invoke-WithRetry {
    param(
        [Parameter(Mandatory)]
        [scriptblock]$Action,
        [int]$MaxAttempts = 3,
        [int]$DelaySeconds = 2,
        [type[]]$RetryOn = @([System.Exception])
    )
    
    $attempt = 0
    while ($true) {
        try {
            $attempt++
            return & $Action
        } catch {
            $shouldRetry = $false
            foreach ($type in $RetryOn) {
                if ($_.Exception -is $type) { $shouldRetry = $true; break }
            }
            
            if (!$shouldRetry -or $attempt -ge $MaxAttempts) {
                throw
            }
            
            $wait = $DelaySeconds * [math]::Pow(2, $attempt - 1)  # exponential
            Write-Warning "Attempt $attempt failed. Retrying in ${wait}s..."
            Start-Sleep -Seconds $wait
        }
    }
}

# Usage
$result = Invoke-WithRetry {
    Invoke-RestMethod 'https://api.example.com/data' -TimeoutSec 10
} -MaxAttempts 3 -DelaySeconds 1

# Result/Either pattern
function Get-SafeResult {
    param([scriptblock]$Action)
    
    try {
        $value = & $Action
        return [PSCustomObject]@{
            Success = $true
            Value   = $value
            Error   = $null
        }
    } catch {
        return [PSCustomObject]@{
            Success = $false
            Value   = $null
            Error   = $_
        }
    }
}

$result = Get-SafeResult { Get-Content 'missing.txt' -ErrorAction Stop }
if ($result.Success) {
    Write-Host $result.Value
} else {
    Write-Host "Error: $($result.Error.Exception.Message)"
}

# Structured error logging
function Write-ErrorLog {
    param(
        [Parameter(Mandatory, ValueFromPipeline)]
        [System.Management.Automation.ErrorRecord]$ErrorRecord,
        [string]$LogFile = "$env:TEMP\errors.log",
        [string]$Component = 'App'
    )
    
    $entry = [PSCustomObject]@{
        Timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
        Component = $Component
        Type      = $ErrorRecord.Exception.GetType().Name
        Message   = $ErrorRecord.Exception.Message
        ScriptLine = $ErrorRecord.InvocationInfo.ScriptLineNumber
        StackTrace = $ErrorRecord.ScriptStackTrace
    }
    
    $entry | ConvertTo-Json -Compress | Add-Content $LogFile
    Write-Verbose "Error logged to $LogFile"
}

try {
    1/0
} catch {
    $_ | Write-ErrorLog -Component 'Calculator'
}
```

---

## 5. $PSItem / $Error in Depth

```powershell
# ใน catch block: $_ = ErrorRecord
try {
    Get-ChildItem 'Z:\' -ErrorAction Stop
} catch {
    # ErrorRecord
    $_ | Get-Member -MemberType Property
    
    # Exception chain
    $ex = $_.Exception
    while ($ex) {
        Write-Host " -> $($ex.GetType().Name): $($ex.Message)"
        $ex = $ex.InnerException
    }
}

# Accessing $Error array
Get-Item 'nope1' -ErrorAction SilentlyContinue
Get-Item 'nope2' -ErrorAction SilentlyContinue

$Error | ForEach-Object {
    [PSCustomObject]@{
        Type    = $_.Exception.GetType().Name
        Message = $_.Exception.Message
        Time    = $_.InvocationInfo.HistoryId
    }
}
```

---

**ก่อนหน้า ← [Part 10](Part-10.md) | ต่อไป → [Part 12: Pipeline](Part-12.md)**
