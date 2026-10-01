# Part 88: Performance Profiling และ Optimization

> **ระดับ**: 🟠 World-Class | **เวลา**: ~5 ชั่วโมง

---

## 1. Profiling Tools

```powershell
# Measure-Command with detailed output
function Measure-ScriptBlock {
    param(
        [scriptblock]$ScriptBlock,
        [string]$Label = 'Benchmark',
        [int]$Iterations = 5,
        [switch]$Warmup
    )
    
    # Warmup run (JIT)
    if ($Warmup) { & $ScriptBlock | Out-Null }
    
    $times = 1..$Iterations | ForEach-Object {
        (Measure-Command { & $ScriptBlock }).TotalMilliseconds
    }
    
    $stats = [PSCustomObject]@{
        Label      = $Label
        Iterations = $Iterations
        MinMs      = [math]::Round(($times | Measure-Object -Minimum).Minimum, 3)
        MaxMs      = [math]::Round(($times | Measure-Object -Maximum).Maximum, 3)
        AvgMs      = [math]::Round(($times | Measure-Object -Average).Average, 3)
        MedianMs   = [math]::Round(($times | Sort-Object)[[int]($Iterations/2)], 3)
        StdDevMs   = [math]::Round(
            [math]::Sqrt((($times | ForEach-Object { ($_ - ($times | Measure-Object -Average).Average) ** 2 }) | Measure-Object -Average).Average),
            3
        )
    }
    
    Write-Host "[$($stats.Label)] Avg=$($stats.AvgMs)ms Min=$($stats.MinMs)ms Max=$($stats.MaxMs)ms" -ForegroundColor Cyan
    return $stats
}

# Compare two implementations
$resultA = Measure-ScriptBlock {
    # Method A: string concatenation (slow)
    $str = ''
    1..1000 | ForEach-Object { $str += "item$_," }
    $str
} -Label 'StringConcat' -Iterations 5 -Warmup

$resultB = Measure-ScriptBlock {
    # Method B: StringBuilder (fast)
    $sb = [System.Text.StringBuilder]::new()
    1..1000 | ForEach-Object { $sb.Append("item$_,") | Out-Null }
    $sb.ToString()
} -Label 'StringBuilder' -Iterations 5 -Warmup

$speedup = [math]::Round($resultA.AvgMs / $resultB.AvgMs, 1)
Write-Host "StringBuilder is ${speedup}x faster" -ForegroundColor Green
```

---

## 2. Memory Profiling

```powershell
# Memory profiler
function Measure-MemoryUsage {
    param([scriptblock]$ScriptBlock, [string]$Label = '')
    
    [GC]::Collect()
    [GC]::WaitForPendingFinalizers()
    [GC]::Collect()
    
    $before = [GC]::GetTotalMemory($true)
    $result = & $ScriptBlock
    $after  = [GC]::GetTotalMemory($false)
    
    $allocatedMB = [math]::Round(($after - $before) / 1MB, 3)
    Write-Host "[$Label] Memory delta: ${allocatedMB}MB" -ForegroundColor Yellow
    return $result
}

# Track memory allocations
Measure-MemoryUsage {
    # Large array allocation
    $arr = 1..100000 | ForEach-Object { [PSCustomObject]@{ Id=$_; Name="item$_"; Value=(Get-Random) } }
    Write-Host "Created $($arr.Count) objects"
} -Label 'ObjectArray'

Measure-MemoryUsage {
    # More efficient: typed array
    $arr = [System.Collections.Generic.List[hashtable]]::new(100000)
    1..100000 | ForEach-Object { $arr.Add(@{ Id=$_; Name="item$_"; Value=(Get-Random) }) }
    Write-Host "Created $($arr.Count) objects (List)"
} -Label 'ListHashtable'

# Memory leak detection
function Test-MemoryLeak {
    param([scriptblock]$ScriptBlock, [int]$Cycles = 10)
    
    $readings = @()
    for ($i = 0; $i -lt $Cycles; $i++) {
        & $ScriptBlock | Out-Null
        [GC]::Collect()
        $readings += [GC]::GetTotalMemory($true)
    }
    
    # Check for upward trend
    $deltas = for ($i = 1; $i -lt $readings.Count; $i++) {
        $readings[$i] - $readings[$i-1]
    }
    $avgDelta = ($deltas | Measure-Object -Average).Average
    
    if ($avgDelta -gt 100KB) {
        Write-Warning "Possible memory leak! Avg growth: $([math]::Round($avgDelta/1KB,1))KB per cycle"
    } else {
        Write-Host "No leak detected (avg delta: $([math]::Round($avgDelta/1KB,1))KB)" -ForegroundColor Green
    }
    
    return $readings
}
```

---

## 3. Pipeline Optimization เทคนิค

```powershell
# Optimization 1: Filter early, project late
# SLOW: process all, then filter
$slow = Measure-ScriptBlock {
    $items = 1..10000 | ForEach-Object { [PSCustomObject]@{ Id=$_; Value=Get-Random } }
    $items | Select-Object * | Where-Object { $_.Value -gt 500000 } | Select-Object Id
} -Label 'FilterLate'

# FAST: filter first with Where-Object, project last
$fast = Measure-ScriptBlock {
    $items = 1..10000 | ForEach-Object { [PSCustomObject]@{ Id=$_; Value=Get-Random } }
    $items | Where-Object { $_.Value -gt 500000 } | Select-Object Id
} -Label 'FilterEarly'

# Optimization 2: Avoid repeated property access
function Optimize-PropertyAccess {
    param([object[]]$Objects)
    
    # SLOW: access same property repeatedly in loop
    $sum1 = 0
    foreach ($obj in $Objects) { $sum1 += $obj.Value }
    
    # FAST: use Measure-Object for numeric operations
    $sum2 = ($Objects | Measure-Object -Property Value -Sum).Sum
    
    # FASTER: use .NET array methods
    $values = [double[]]($Objects.Value)
    $sum3   = [System.Linq.Enumerable]::Sum([IEnumerable[double]]$values)
    
    return $sum3
}

# Optimization 3: Runspace parallelism
function Invoke-ParallelOptimized {
    param(
        [array]$InputObjects,
        [scriptblock]$ScriptBlock,
        [int]$ThrottleLimit = 10
    )
    
    if ($PSVersionTable.PSVersion.Major -ge 7) {
        # PS7+: ForEach-Object -Parallel is cleanest
        return $InputObjects | ForEach-Object -Parallel $ScriptBlock -ThrottleLimit $ThrottleLimit
    }
    
    # PS5.1: Manual runspace pool
    $pool     = [runspacefactory]::CreateRunspacePool(1, $ThrottleLimit)
    $pool.Open()
    
    $tasks = $InputObjects | ForEach-Object {
        $ps = [powershell]::Create().AddScript($ScriptBlock).AddArgument($_)
        $ps.RunspacePool = $pool
        @{ PS=$ps; Result=$ps.BeginInvoke() }
    }
    
    $results = $tasks | ForEach-Object {
        $_.PS.EndInvoke($_.Result)
        $_.PS.Dispose()
    }
    
    $pool.Close()
    $pool.Dispose()
    return $results
}

# Benchmark serial vs parallel
$workItems = 1..20

$serial = Measure-ScriptBlock {
    $workItems | ForEach-Object { Start-Sleep -Milliseconds 100; $_ * 2 }
} -Label 'Serial'

$parallel = Measure-ScriptBlock {
    Invoke-ParallelOptimized $workItems { param($x) Start-Sleep -Milliseconds 100; $x * 2 } -ThrottleLimit 10
} -Label 'Parallel'

Write-Host "Parallel speedup: $([math]::Round($serial.AvgMs / $parallel.AvgMs, 1))x" -ForegroundColor Green
```

---

## 4. Profiling Pipeline Bottlenecks

```powershell
# Trace pipeline stages
class ProfiledPipeline {
    hidden [System.Collections.Generic.List[hashtable]]$_timings = [System.Collections.Generic.List[hashtable]]::new()
    
    [object[]] Run([object[]]$input, [hashtable[]]$stages) {
        $data = $input
        foreach ($stage in $stages) {
            $sw = [System.Diagnostics.Stopwatch]::StartNew()
            $data = $data | & $stage.transform
            $sw.Stop()
            $this._timings.Add(@{
                Stage   = $stage.name
                InputCount  = $input.Count
                OutputCount = @($data).Count
                ElapsedMs   = $sw.Elapsed.TotalMilliseconds
            })
        }
        return $data
    }
    
    [void] PrintReport() {
        Write-Host "`n=== Pipeline Profile ==" -ForegroundColor Cyan
        $total = ($this._timings | Measure-Object ElapsedMs -Sum).Sum
        
        $this._timings | ForEach-Object {
            $pct = [math]::Round($_.ElapsedMs / $total * 100, 1)
            $bar = '#' * [int]($pct / 5)
            Write-Host ("  {0,-20} {1,8:F1}ms  {2,5}%  {3}" -f $_.Stage, $_.ElapsedMs, $pct, $bar)
        }
        Write-Host ("  {0,-20} {1,8:F1}ms  100%" -f 'TOTAL', $total) -ForegroundColor Yellow
    }
}

$pp     = [ProfiledPipeline]::new()
$result = $pp.Run(1..10000, @(
    @{ name='Parse';       transform={ $_ | ForEach-Object { @{ id=$_; val=Get-Random } } } }
    @{ name='Filter';      transform={ $_ | Where-Object   { $_.val -gt 500000 }       } }
    @{ name='Transform';   transform={ $_ | ForEach-Object { $_.val = $_.val * 1.1; $_ } } }
    @{ name='Sort';        transform={ $_ | Sort-Object -Property val -Descending       } }
    @{ name='Top100';      transform={ $_ | Select-Object -First 100                   } }
))

$pp.PrintReport()
Write-Host "Result: $($result.Count) items"
```

---

**ก่อนหน้า ← [Part 87](Part-87.md) | ต่อไป → [Part 89: Module Publishing](Part-89.md)**
