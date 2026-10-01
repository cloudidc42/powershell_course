# Part 69: Advanced Pipeline Patterns

> **ระดับ**: 🔴 Advanced | **เวลา**: ~4 ชั่วโมง

---

## 1. Pipeline-Aware Functions

```powershell
# Proper pipeline input handling
function Convert-ObjectToCSVRow {
    [CmdletBinding()]
    param(
        [Parameter(ValueFromPipeline=$true, Mandatory=$true)]
        [object]$InputObject,
        [string[]]$Properties,
        [char]$Delimiter = ','
    )
    
    begin {
        $first    = $true
        $propList = $Properties  # set in process from first object if null
    }
    
    process {
        # Auto-detect properties from first object
        if (-not $propList) {
            $propList = $InputObject.PSObject.Properties.Name
        }
        
        # Print header on first object
        if ($first) {
            $first = $false
            $propList -join $Delimiter  # header row
        }
        
        # Data row
        ($propList | ForEach-Object {
            $val = $InputObject.$_
            if ($val -match "[$Delimiter`"\r\n]") { "`"$($val -replace '"','""')`"" }
            else { $val }
        }) -join $Delimiter
    }
}

Get-Process | Select-Object Name, CPU, WorkingSet |
    Convert-ObjectToCSVRow |
    Set-Content './processes.csv'
```

---

## 2. Streaming ETL Pipeline

```powershell
# Large file processing without loading all into memory
function Invoke-StreamETL {
    param(
        [string]$InputFile,
        [string]$OutputFile,
        [scriptblock]$Transform,
        [scriptblock]$Filter      = { $true },
        [int]$BatchSize            = 1000,
        [int]$ReportEvery          = 10000
    )
    
    $reader  = [System.IO.StreamReader]::new($InputFile)
    $writer  = [System.IO.StreamWriter]::new($OutputFile, $false, [System.Text.Encoding]::UTF8)
    
    $headers  = $reader.ReadLine() -split ','
    $inCount  = 0
    $outCount = 0
    $batch    = @()
    
    $writer.WriteLine($headers -join ',')
    
    try {
        while (-not $reader.EndOfStream) {
            $line   = $reader.ReadLine()
            $fields = $line -split ','
            $obj    = [ordered]@{}
            for ($i = 0; $i -lt $headers.Count; $i++) { $obj[$headers[$i]] = $fields[$i] }
            $row    = [PSCustomObject]$obj
            $inCount++
            
            if (& $Filter $row) {
                $transformed = & $Transform $row
                $writer.WriteLine(
                    ($transformed.PSObject.Properties.Value | ForEach-Object {
                        if ($_ -match ',') { "`"$_`"" } else { $_ }
                    }) -join ','
                )
                $outCount++
            }
            
            if ($inCount % $ReportEvery -eq 0) {
                Write-Progress -Activity 'ETL' -Status "Processed $inCount rows, $outCount output"
            }
        }
    } finally {
        $reader.Dispose()
        $writer.Dispose()
    }
    
    Write-Host "ETL complete: $inCount in -> $outCount out" -ForegroundColor Green
}

# Usage: filter and transform large CSV
Invoke-StreamETL `
    -InputFile   './orders-raw.csv' `
    -OutputFile  './orders-clean.csv' `
    -Filter      { param($r) $r.Amount -gt 0 -and $r.Status -ne 'CANCELLED' } `
    -Transform   {
        param($r)
        [PSCustomObject]@{
            OrderId    = $r.OrderId
            Amount     = [math]::Round([decimal]$r.Amount, 2)
            CustomerId = $r.CustomerId.Trim()
            Date       = [datetime]::ParseExact($r.Date, 'dd/MM/yyyy', $null).ToString('yyyy-MM-dd')
        }
    }
```

---

## 3. Reactive Pipeline with Observer

```powershell
# Observer pattern for pipeline monitoring
class PipelineObserver {
    hidden [System.Collections.Generic.List[scriptblock]]$_handlers = @()
    hidden [hashtable]$_stats = @{ Processed=0; Errors=0; Start=[datetime]::Now }
    
    [void] Subscribe([scriptblock]$handler) {
        $this._handlers.Add($handler)
    }
    
    [void] Emit([string]$event, [object]$data) {
        foreach ($handler in $this._handlers) {
            try { & $handler $event $data $this._stats }
            catch { Write-Warning "Observer error: $_" }
        }
    }
    
    [object] Process([object]$item, [scriptblock]$transform) {
        try {
            $result = & $transform $item
            $this._stats.Processed++
            $this.Emit('processed', @{ item=$item; result=$result })
            return $result
        } catch {
            $this._stats.Errors++
            $this.Emit('error', @{ item=$item; error=$_ })
            return $null
        }
    }
    
    [hashtable] GetStats() {
        $elapsed             = ([datetime]::Now - $this._stats.Start).TotalSeconds
        $this._stats.ElapsedS = [math]::Round($elapsed, 1)
        $this._stats.PerSecond= if ($elapsed -gt 0) { [math]::Round($this._stats.Processed/$elapsed, 0) } else { 0 }
        return $this._stats
    }
}

$observer = [PipelineObserver]::new()
$observer.Subscribe({
    param($event, $data, $stats)
    if ($event -eq 'error') { Write-Warning "Pipeline error: $($data.error)" }
    if ($stats.Processed % 1000 -eq 0) {
        Write-Host "Processed: $($stats.Processed) ($($stats.PerSecond)/s)" -ForegroundColor Cyan
    }
})

$results = 1..10000 | ForEach-Object {
    $observer.Process($_, { param($n) $n * 2 })
} | Where-Object { $_ -ne $null }

$observer.GetStats() | Format-Table
```

---

## 4. Functional Pipeline Combinators

```powershell
# Compose functions like functional programming
function Compose-Functions {
    param([scriptblock[]]$Functions)
    return {
        param($input)
        $Functions | ForEach-Object { $input = & $_ $input }
        $input
    }
}

function Pipe {
    param($Value, [scriptblock[]]$Steps)
    $Steps | ForEach-Object { $Value = & $_ $Value }
    return $Value
}

# Usage
$pipeline = Compose-Functions @(
    { param($x) $x * 2 }
    { param($x) $x + 10 }
    { param($x) "Result: $x" }
)

& $pipeline 5   # 'Result: 20'

Pipe 5 @(
    { param($x) $x * 2 }
    { param($x) $x + 10 }
    { param($x) "Result: $x" }
)  # 'Result: 20'

# Partial application
function Curry {
    param([scriptblock]$Fn, [object[]]$Args)
    return [scriptblock]::Create("
        param([object[]]`\$rest)
        & `$fn ($Args + `\$rest)
    ".Replace('`$fn',"`$fn"))
}

$double = { param($n) $n * 2 }
$add5   = { param($n) $n + 5 }

1..10 | ForEach-Object {
    Pipe $_ @($double, $add5)
}
```

---

**ก่อนหน้า ← [Part 68](Part-68.md) | ต่อไป → [Part 70: Advanced Error Handling](Part-70.md)**
