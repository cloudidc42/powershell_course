# Part 40: Debugging และ Troubleshooting

> **ระดับ**: 🟡 Professional | **เวลา**: ~3 ชั่วขนึ่ง

---

## 1. Debugger พื้นฐาน

```powershell
# Set breakpoints
Set-PSBreakpoint -Script 'script.ps1' -Line 25
Set-PSBreakpoint -Script 'script.ps1' -Variable 'count' -Mode ReadWrite
Set-PSBreakpoint -Script 'script.ps1' -Command 'Get-ChildItem'

# Conditional breakpoint
Set-PSBreakpoint -Script 'script.ps1' -Line 50 -Action {
    if ($count -gt 100) { break }
}

# List/remove breakpoints
Get-PSBreakpoint
Remove-PSBreakpoint -Id 0
Remove-PSBreakpoint -Script 'script.ps1'

# Debugger commands (ใน PS debugger prompt):
# s  = step into
# v  = step over
# o  = step out
# c  = continue
# l  = list context
# q  = quit debugger
# h  = help
```

---

## 2. Write-Debug และ $DebugPreference

```powershell
$DebugPreference = 'Continue'  # Show debug messages
$VerbosePreference = 'Continue'
$InformationPreference = 'Continue'

function Process-Data {
    [CmdletBinding()]
    param([int[]]$Items)
    
    Write-Debug "Starting with $($Items.Count) items"
    
    $results = foreach ($item in $Items) {
        Write-Verbose "  Processing item: $item"
        if ($item -lt 0) {
            Write-Warning "Skipping negative: $item"
            continue
        }
        $item * 2
    }
    
    Write-Information "Completed: $($results.Count) results" -Tags 'Progress'
    Write-Debug "Final results: $results"
    $results
}

Process-Data -Items @(1, -2, 3, 4) -Debug -Verbose

# สั่งแค่พารามิเตอร์
Process-Data -Items @(1, 2, 3) -Debug
Process-Data -Items @(1, 2, 3) -Verbose
```

---

## 3. โปรแกรม Transcript และ Logging

```powershell
# บันทึก session (ทุกอย่างตั้งแต่ output)
Start-Transcript -Path 'C:\Logs\session.log' -Append
# ... run commands ...
Stop-Transcript

# Structured logging
class Logger {
    [string]$LogPath
    [string]$Level
    hidden [System.IO.StreamWriter]$Writer
    
    Logger([string]$logPath, [string]$level = 'INFO') {
        $this.LogPath = $logPath
        $this.Level   = $level
        $dir = Split-Path $logPath
        if ($dir -and -not (Test-Path $dir)) { New-Item $dir -ItemType Directory -Force }
        $this.Writer = [System.IO.StreamWriter]::new($logPath, $true)  # append
        $this.Writer.AutoFlush = $true
    }
    
    [void] Log([string]$level, [string]$message) {
        $ts   = [datetime]::UtcNow.ToString('o')
        $line = "{`"ts`":`"$ts`",`"level`":`"$level`",`"msg`":`"$($message -replace '"','\\"')`"}"
        $this.Writer.WriteLine($line)
        if ($level -in 'WARN','ERROR') { Write-Warning $message }
    }
    
    [void] Info([string]$msg)  { $this.Log('INFO',  $msg) }
    [void] Warn([string]$msg)  { $this.Log('WARN',  $msg) }
    [void] Error([string]$msg) { $this.Log('ERROR', $msg) }
    [void] Debug([string]$msg) {
        if ($this.Level -eq 'DEBUG') { $this.Log('DEBUG', $msg) }
    }
    [void] Dispose() { $this.Writer.Close() }
}

$log = [Logger]::new('C:\Logs\app.log', 'DEBUG')
$log.Info('Application started')
$log.Debug('Processing 1000 records')
$log.Warn('Low disk space')
$log.Error('Connection failed')
$log.Dispose()
```

---

## 4. Error Analysis

```powershell
# วิเคราะห์ error object
try {
    1/0
} catch {
    $err = $_
    $err.Exception.GetType().FullName   # System.DivideByZeroException
    $err.Exception.Message
    $err.InvocationInfo.ScriptLineNumber
    $err.InvocationInfo.Line
    $err.ScriptStackTrace
    $err.CategoryInfo
    $err | ConvertTo-Json -Depth 3
}

# $Error เก็บ error ล่าสุด (default 256)
$Error[0]    # ล่าสุด
$Error.Clear()

# Trace script execution
Set-PSDebug -Trace 1  # trace lines
Set-PSDebug -Trace 2  # trace + variables
Set-PSDebug -Off

# Enable strict mode
Set-StrictMode -Version Latest
# ทำให้เกิด error เมื่อใช้ undefined variable
```

---

## 5. VS Code Debugger Config

```json
// .vscode/launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "PowerShell: Debug Script",
            "type": "PowerShell",
            "request": "launch",
            "script": "${workspaceFolder}/src/main.ps1",
            "args": ["-Verbose", "-InputPath", "C:\\Data"]
        },
        {
            "name": "PowerShell: Attach to Process",
            "type": "PowerShell",
            "request": "attach",
            "processId": "${command:PickProcess}"
        },
        {
            "name": "PowerShell: Interactive Session",
            "type": "PowerShell",
            "request": "launch",
            "cwd": "${workspaceFolder}"
        }
    ]
}
```

---

**ก่อนหน้า ← [Part 39](Part-39.md) | ต่อไป → [Part 41: Security Architecture](Part-41.md)**
