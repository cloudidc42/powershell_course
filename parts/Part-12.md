# Part 12: Pipeline และ Object Streaming

> **ระดับ**: 🟡 Intermediate | **เวลา**: ~3 ชั่วโมง

---

## 1. Pipeline Fundamentals

```powershell
# Pipeline ส่ง objects ไม่ใช่ text!
Get-Process | Where-Object CPU -gt 10 | Sort-Object CPU -Descending | Select-Object -First 5

# แต่ละขั้นตอนใช้ objects
Get-Process |          # ProcessInfo objects
Where-Object { $_.CPU -gt 10 } |  # filtered ProcessInfo
Sort-Object CPU -Descending |      # sorted ProcessInfo
Select-Object Name, CPU, WorkingSet | # PSCustomObject ผัป
 Select-Object -First 5    # first 5

# Measure performance
$sw = [System.Diagnostics.Stopwatch]::StartNew()
Get-ChildItem 'C:\Windows' -Recurse -File -ErrorAction SilentlyContinue |
    Where-Object { $_.Extension -eq '.dll' } |
    Measure-Object Length -Sum |
    ForEach-Object { "{0:N2} MB" -f ($_.Sum/1MB) }
Write-Host "Time: $($sw.ElapsedMilliseconds)ms"
```

---

## 2. Select-Object

```powershell
$processes = Get-Process | Select-Object -First 10

# เลือก properties
$processes | Select-Object Name, Id, CPU, WorkingSet

# Calculated properties
$processes | Select-Object Name, 
    @{Name='CPU_s';    Expression={[math]::Round($_.CPU, 2)}},
    @{Name='RAM_MB';   Expression={[math]::Round($_.WorkingSet/1MB, 1)}},
    @{Name='Threads';  Expression={$_.Threads.Count}}

# Short form
$processes | Select-Object Name, 
    @{N='CPU';  E={[math]::Round($_.CPU,1)}},
    @{N='RAM';  E={"{0:N0}MB" -f ($_.WorkingSet/1MB)}}

# Expand property
$services = Get-Service | Select-Object -First 5
$services | Select-Object -ExpandProperty Name  # array of strings

# Skip/Take
Get-Process | Select-Object -Skip 5 -First 10   # page 2

# Unique
@(1,2,2,3,3,4) | Select-Object -Unique

# Index
Get-Process | Select-Object -Index 0,2,4  # positions 0, 2, 4
```

---

## 3. Where-Object

```powershell
# Simple comparison (PS3+ syntax)
Get-Process | Where-Object CPU -gt 10
Get-Service | Where-Object Status -eq 'Running'
Get-ChildItem | Where-Object Name -like '*.ps1'

# Script block (more flexible)
Get-Process | Where-Object { $_.CPU -gt 10 -and $_.WorkingSet -gt 50MB }
Get-Process | Where-Object { $_.Name -match '^s' -and $_.Id -gt 1000 }

# Alias: ?
Get-Process | ? { $_.CPU -gt 5 }

# NOT filter
Get-Service | Where-Object { $_.Status -ne 'Running' }
Get-ChildItem | Where-Object { -not $_.PSIsContainer }  # files only

# Complex
$startDate = (Get-Date).AddDays(-7)
Get-EventLog -LogName Application -Newest 1000 |
    Where-Object { $_.EntryType -eq 'Error' -and $_.TimeGenerated -gt $startDate } |
    Select-Object TimeGenerated, Source, Message |
    Sort-Object TimeGenerated -Descending
```

---

## 4. ForEach-Object in Depth

```powershell
# Begin/Process/End
$total = 0
Get-Process | ForEach-Object -Begin {
    Write-Host "Processing..."
} -Process {
    $total += $_.WorkingSet
} -End {
    "{0:N0} MB total RAM" -f ($total/1MB)
}

# Chained transformations
1..100 |
    ForEach-Object { $_ * 2 } |     # double
    Where-Object { $_ % 3 -eq 0 } | # divisible by 3
    ForEach-Object { $_ * $_ } |    # square
    Select-Object -First 5          # first 5

# Parallel (PS7+)
$servers = @('server1', 'server2', 'server3', 'server4', 'server5')
$results = $servers | ForEach-Object -Parallel {
    $hostname = $_
    $ping = Test-Connection $hostname -Count 1 -Quiet -TimeoutSeconds 2
    [PSCustomObject]@{
        Server  = $hostname
        Online  = $ping
        Time    = (Get-Date).ToString('HH:mm:ss')
    }
} -ThrottleLimit 5

$results | Sort-Object Server | Format-Table
```

---

## 5. Sort-Object และ Group-Object

```powershell
# Sort
$procs = Get-Process | Select-Object Name, CPU, WorkingSet

$procs | Sort-Object Name
$procs | Sort-Object CPU -Descending
$procs | Sort-Object Name, CPU -Descending   # multi-column

# Custom sort
$procs | Sort-Object { $_.WorkingSet / 1MB } -Descending

# Stable sort (PS7+)
$procs | Sort-Object Name -Stable

# Group
$services = Get-Service | Group-Object Status
$services | ForEach-Object {
    Write-Host "$($_.Name): $($_.Count) services"
    $_.Group | Select-Object Name | Format-Wide -Column 5
}

# Group with hashtable output
$grouped = Get-Process | 
    Group-Object { [math]::Truncate($_.CPU) } |
    Select-Object @{N='CPU_Range';E={"$($_.Name)s"}}, Count |
    Sort-Object Count -Descending
```

---

## 6. Measure-Object

```powershell
# Numbers
1..100 | Measure-Object -Sum -Average -Maximum -Minimum -StandardDeviation

# Properties
Get-Process | Measure-Object WorkingSet -Sum -Average -Maximum |
Select-Object @{N='Total_MB'; E={"{0:N0}" -f ($_.Sum/1MB)}},
              @{N='Avg_MB';   E={"{0:N1}" -f ($_.Average/1MB)}},
              @{N='Max_MB';   E={"{0:N0}" -f ($_.Maximum/1MB)}}

# Text (count lines/words/chars)
Get-Content 'script.ps1' | Measure-Object -Line -Word -Character
```

---

## 7. Tee-Object, Out-*, Format-*

```powershell
# Tee - ส่ง output ไปหลายทาง
Get-Process | Tee-Object -FilePath 'procs.txt' | Where-Object CPU -gt 10

# Out-File
Get-Process | Out-File 'procs.txt'

# Out-String (convert to string)
$str = Get-Process | Out-String

# Out-Null (discard)
Get-Process | Out-Null

# Format-Table / Format-List / Format-Wide / Format-Custom
Get-Process | Format-Table Name, CPU, WorkingSet -AutoSize
Get-Process | Where-Object CPU -gt 10 | Format-List *
Get-Process | Format-Wide Name -Column 5

# ConvertTo-Html
Get-Process | Select-Object Name, CPU, WorkingSet | ConvertTo-Html | Out-File 'procs.html'

# ConvertTo-Csv
Get-Process | Select-Object Name, CPU | ConvertTo-Csv -NoTypeInformation
```

---

## 8. Pipeline Tricks

```powershell
# สร้าง custom pipeline cmdlet
function Invoke-Transform {
    [CmdletBinding()]
    param(
        [Parameter(ValueFromPipeline)]
        [PSObject]$InputObject,
        
        [Parameter(Mandatory)]
        [scriptblock]$ScriptBlock
    )
    
    process {
        & $ScriptBlock $InputObject
    }
}

Get-Process |
    Invoke-Transform { param($p) [PSCustomObject]@{
        Name = $p.Name.ToUpper()
        Mem  = "{0:N0}KB" -f ($p.WorkingSet/1KB)
    }} |
    Sort-Object Name |
    Select-Object -First 10

# Filter + transform in one
function Select-Where {
    [CmdletBinding()]
    param(
        [Parameter(ValueFromPipeline)]
        $InputObject,
        
        [Parameter(Position=0)]
        [scriptblock]$Filter,
        
        [Parameter(Position=1)]
        [scriptblock]$Transform
    )
    
    process {
        if (!$Filter -or (& $Filter $InputObject)) {
            if ($Transform) { & $Transform $InputObject }
            else { $InputObject }
        }
    }
}

# ใช้ PowerShell v3+ null-conditional pipeline
$null | ForEach-Object { "Not null: $_" }  # nothing output
$null | Where-Object { $_ }               # nothing output
```

---

**ก่อนหน้า ← [Part 11](Part-11.md) | ต่อไป → [Part 13: File System](Part-13.md)**
