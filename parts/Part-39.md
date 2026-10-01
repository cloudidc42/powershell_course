# Part 39: Performance Optimization

> **ระดับ**: 🟡 Professional | **เวลา**: ~4 ชั่วขนึ่ง

---

## 1. วัดประสิทธิภาพ

```powershell
# Measure-Object สำหรับวัดเวลา
function Measure-Code {
    param([scriptblock]$ScriptBlock, [int]$Iterations = 1)
    $sw = [System.Diagnostics.Stopwatch]::StartNew()
    for ($i = 0; $i -lt $Iterations; $i++) { & $ScriptBlock }
    $sw.Stop()
    [PSCustomObject]@{
        TotalMs  = [math]::Round($sw.ElapsedMilliseconds, 2)
        AvgMs    = [math]::Round($sw.ElapsedMilliseconds / $Iterations, 4)
        Iterations = $Iterations
    }
}

# Comparison
$n = 10000
$slow = Measure-Code { $arr = @(); 1..$n | ForEach-Object { $arr += $_ } } -Iterations 5
$fast = Measure-Code { $arr = [System.Collections.Generic.List[int]]::new(); 1..$n | ForEach-Object { $arr.Add($_) } } -Iterations 5

Write-Host "Slow (+= array):    $($slow.AvgMs)ms"
Write-Host "Fast (List.Add()):  $($fast.AvgMs)ms"
Write-Host "Speedup: $([math]::Round($slow.AvgMs / $fast.AvgMs, 1))x"
```

---

## 2. เทคนิคที่เร็วขึ้น

```powershell
# 1. ใช้ List<T> แทน array +=
$list = [System.Collections.Generic.List[PSCustomObject]]::new()
foreach ($i in 1..100000) { $list.Add([PSCustomObject]@{N=$i;V=$i*2}) }
# (เร็วกว่า $arr += 100x)

# 2. เลียง $null แทน Out-Null
$null = Some-Function    # เร็วกว่า
Some-Function | Out-Null # ช้ากว่า

# 3. Where-Object vs .Where()
$items = 1..100000
$a = $items | Where-Object { $_ -gt 50000 }  # pipeline overhead
$b = $items.Where({ $_ -gt 50000 })          # faster for in-memory collections

# 4. ForEach-Object vs foreach statement
$a = $items | ForEach-Object { $_ * 2 }  # pipeline
$b = foreach ($i in $items) { $i * 2 }   # faster statement

# 5. StringBuilder for string concat
$sb = [System.Text.StringBuilder]::new()
foreach ($i in 1..10000) { [void]$sb.Append("Line $i`n") }
$result = $sb.ToString()
# (เร็วกว่า += string 1000x)

# 6. [hashtable] vs [PSCustomObject] for large lookups
$ht = @{}
1..100000 | ForEach-Object { $ht[$_] = "val$_" }
$ht[50000]  # O(1) lookup

# 7. เลือก Get-ChildItem -File vs where
Get-ChildItem -File -Recurse                         # เร็ว
$arr = Get-ChildItem -Recurse | Where-Object -Not PSIsContainer  # ช้า
```

---

## 3. Parallel Processing

```powershell
# ForEach-Object -Parallel (PS 7+)
$results = 1..20 | ForEach-Object -Parallel {
    $n = $_
    Start-Sleep -Milliseconds (Get-Random -Max 500)
    [PSCustomObject]@{ N=$n; Result=$n*$n; Thread=[System.Threading.Thread]::CurrentThread.ManagedThreadId }
} -ThrottleLimit 5

$results | Sort-Object N | Format-Table -AutoSize

# Thread-safe collection
$bag = [System.Collections.Concurrent.ConcurrentBag[PSCustomObject]]::new()
1..100 | ForEach-Object -Parallel {
    $b = $using:bag
    $b.Add([PSCustomObject]@{N=$_; V=$_*$_})
} -ThrottleLimit 10
$bag.Count  # all 100

# Runspace pool (low-level, more control)
function Invoke-Parallel {
    param([array]$Items, [scriptblock]$ScriptBlock, [int]$MaxThreads = 5)
    
    $pool = [RunspaceFactory]::CreateRunspacePool(1, $MaxThreads)
    $pool.Open()
    
    $jobs = $Items | ForEach-Object {
        $ps = [PowerShell]::Create().AddScript($ScriptBlock).AddArgument($_)
        $ps.RunspacePool = $pool
        @{ PS=$ps; Handle=$ps.BeginInvoke(); Input=$_ }
    }
    
    $results = $jobs | ForEach-Object {
        $_.PS.EndInvoke($_.Handle)
        $_.PS.Dispose()
    }
    
    $pool.Close()
    return $results
}

$out = Invoke-Parallel -Items @('a','b','c','d','e') -ScriptBlock {
    param($item)
    "Processed: $item"
} -MaxThreads 3
$out
```

---

## 4. Memory Profiling

```powershell
# วัดหน่วยความจำ
function Measure-Memory {
    param([scriptblock]$Before, [scriptblock]$After)
    [GC]::Collect()
    $before = [System.GC]::GetTotalMemory($true)
    & $Before
    [GC]::Collect()
    & $After
    $after = [System.GC]::GetTotalMemory($true)
    [PSCustomObject]@{
        BeforeMB = [math]::Round($before/1MB, 2)
        AfterMB  = [math]::Round($after/1MB, 2)
        DeltaMB  = [math]::Round(($after-$before)/1MB, 2)
    }
}

$result = Measure-Memory -Before {} -After {
    $global:big = 1..1000000 | ForEach-Object { [PSCustomObject]@{N=$_; Data="item$_"} }
}
$result

# Force GC
$global:big = $null
[System.GC]::Collect()
[System.GC]::WaitForPendingFinalizers()
[System.GC]::Collect()
```

---

**ก่อนหน้า ← [Part 38](Part-38.md) | ต่อไป → [Part 40: Debugging](Part-40.md)**
