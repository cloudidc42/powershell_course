# Part 09: Loops

> **ระดับ**: 🟢 Beginner | **เวลา**: ~3 ชั่วโมง

---

## 1. For Loop

```powershell
# พื้นฐาน
for ($i = 0; $i -lt 10; $i++) {
    Write-Host "Item $i"
}

# กับหลายตัวแปร
for ($i = 0, $j = 10; $i -lt $j; $i++, $j--) {
    Write-Host "i=$i j=$j"
}

# Loop backwards
for ($i = 9; $i -ge 0; $i--) {
    Write-Host $i
}

# Step by 2
for ($i = 0; $i -le 20; $i += 2) {
    Write-Host $i  # 0, 2, 4, ... 20
}

# วน array ด้วย index
$arr = @("a", "b", "c", "d", "e")
for ($i = 0; $i -lt $arr.Count; $i++) {
    Write-Host "[$i] = $($arr[$i])"
}
```

---

## 2. Foreach Loop

```powershell
# foreach สำหรับ collections
$fruits = @("Apple", "Banana", "Cherry", "Date")

foreach ($fruit in $fruits) {
    Write-Host "Fruit: $fruit"
}

# foreach กับ hashtable
$config = @{ Host = "localhost"; Port = 8080; Ssl = $true }
foreach ($key in $config.Keys) {
    Write-Host "$key = $($config[$key])"
}

# foreach กับ files
foreach ($file in Get-ChildItem 'C:\Windows' -Filter '*.dll') {
    Write-Host "$($file.Name): $([math]::Round($file.Length/1KB,1))KB"
}

# Nested foreach
$matrix = @(@(1,2,3), @(4,5,6), @(7,8,9))
foreach ($row in $matrix) {
    foreach ($cell in $row) {
        Write-Host -NoNewline "$cell "
    }
    Write-Host
}
```

---

## 3. ForEach-Object (Pipeline)

```powershell
# Pipeline version
1..10 | ForEach-Object {
    Write-Host "Number: $_"
}

# Short alias: %
1..5 | % { $_ * $_ }   # 1, 4, 9, 16, 25

# Begin/Process/End blocks
1..5 | ForEach-Object -Begin {
    Write-Host "Starting..."
    $sum = 0
} -Process {
    $sum += $_
} -End {
    Write-Host "Sum: $sum"
}

# Parallel (PowerShell 7+)
1..10 | ForEach-Object -Parallel {
    Start-Sleep -Milliseconds 100
    "Processed: $_"
} -ThrottleLimit 5

# With $using: to pass variables
$multiplier = 10
1..5 | ForEach-Object -Parallel {
    $_ * $using:multiplier
} -ThrottleLimit 3

# Shorter syntax (PS3+)
$names = @("alice", "bob", "carol")
$upper = $names | ForEach-Object { $_.ToUpper() }
```

---

## 4. While Loop

```powershell
# สูตร: ทำพรี condition เป็น true
$count = 0
while ($count -lt 5) {
    Write-Host "Count: $count"
    $count++
}

# เดินไฟล์ จนกว่าจะหมด
# $reader = [System.IO.StreamReader]::new($path)
# while ($line = $reader.ReadLine()) {
#     Process-Line $line
# }
# $reader.Close()

# รอ event
while ($true) {
    $input = Read-Host "Enter command (or 'quit')"
    if ($input -eq 'quit') { break }
    Write-Host "Processing: $input"
}
```

---

## 5. Do-While / Do-Until

```powershell
# Do-While: ทำอย่างน้อยหนึ่งครั้ง
$count = 0
do {
    Write-Host "Count: $count"
    $count++
} while ($count -lt 5)

# Do-Until: ทำจนกว่าจะเป็นจริง
do {
    $input = Read-Host "Enter Y to continue"
} until ($input -eq 'Y')

# Input validation pattern
$validInput = $false
do {
    $value = Read-Host "Enter number 1-10"
    if ($value -match '^[1-9]$|^10$') {
        $validInput = $true
    } else {
        Write-Host "Invalid! Try again" -ForegroundColor Red
    }
} while (!$validInput)
Write-Host "You entered: $value"

# Retry pattern
$maxRetries = 3
$retryCount = 0
$success    = $false

do {
    try {
        # ลองเชื่อมต่อ
        # $result = Invoke-WebRequest 'https://api.example.com' -TimeoutSec 5
        $success = $true
        Write-Host "Connected!"
    } catch {
        $retryCount++
        Write-Host "Attempt $retryCount failed: $_"
        if ($retryCount -lt $maxRetries) {
            Start-Sleep -Seconds ($retryCount * 2)  # exponential backoff
        }
    }
} until ($success -or $retryCount -ge $maxRetries)
```

---

## 6. Break, Continue, Return

```powershell
# break - ออกจาก loop
for ($i = 0; $i -lt 10; $i++) {
    if ($i -eq 5) { break }
    Write-Host $i   # 0,1,2,3,4
}

# continue - ข้าม iteration
for ($i = 0; $i -lt 10; $i++) {
    if ($i % 2 -eq 0) { continue }
    Write-Host $i   # 1,3,5,7,9
}

# return - ออกจาก function
function Find-First {
    param([array]$Items, [scriptblock]$Predicate)
    foreach ($item in $Items) {
        if (& $Predicate $item) {
            return $item
        }
    }
    return $null
}

$result = Find-First @(1,3,5,6,7) { param($x) $x % 2 -eq 0 }
# 6

# Labeled break (nested loops)
:outer for ($i = 0; $i -lt 5; $i++) {
    for ($j = 0; $j -lt 5; $j++) {
        if ($i -eq 2 -and $j -eq 2) {
            break outer   # break outer loop
        }
        Write-Host "$i,$j"
    }
}
```

---

## 7. Performance Tips

```powershell
# BAD: += in loop (O(n^2))
$results = @()
for ($i = 0; $i -lt 10000; $i++) {
    $results += $i  # creates new array every time!
}

# GOOD: List
$results = [System.Collections.Generic.List[int]]::new()
for ($i = 0; $i -lt 10000; $i++) {
    $results.Add($i)
}

# ALSO GOOD: pipeline (for simple cases)
$results = 0..9999 | ForEach-Object { $_ }

# BEST for pure pipeline
$results = 1..10000

# เปรียบเทียบเวลา
$sw = [System.Diagnostics.Stopwatch]::StartNew()

# Method 1: +=
$r1 = @()
1..1000 | ForEach-Object { $r1 += $_ }
$t1 = $sw.ElapsedMilliseconds
$sw.Restart()

# Method 2: List
$r2 = [System.Collections.Generic.List[int]]::new()
1..1000 | ForEach-Object { $r2.Add($_) }
$t2 = $sw.ElapsedMilliseconds

Write-Host "Array +=: ${t1}ms | List.Add(): ${t2}ms"
```

---

## 8. Practical Examples

```powershell
# สแกน network ด้วย parallel
function Scan-Network {
    param(
        [string]$Subnet = "192.168.1",
        [int]$Start = 1,
        [int]$End = 254
    )
    
    $jobs = [System.Collections.Generic.List[PSObject]]::new()
    
    $Start..$End | ForEach-Object -Parallel {
        $ip = "$using:Subnet.$_"
        $alive = Test-Connection -ComputerName $ip -Count 1 -Quiet -TimeoutSeconds 1
        [PSCustomObject]@{ IP = $ip; Online = $alive }
    } -ThrottleLimit 50 | Where-Object Online
}

# Prime number sieve
function Get-Primes {
    param([int]$Max)
    
    $sieve = @($true) * ($Max + 1)
    $sieve[0] = $sieve[1] = $false
    
    for ($i = 2; $i * $i -le $Max; $i++) {
        if ($sieve[$i]) {
            for ($j = $i * $i; $j -le $Max; $j += $i) {
                $sieve[$j] = $false
            }
        }
    }
    
    return (0..$Max) | Where-Object { $sieve[$_] }
}

Get-Primes 100   # 2,3,5,7,11,...,97

# Progress bar
function Invoke-WithProgress {
    param([array]$Items, [scriptblock]$Action, [string]$Label = "Processing")
    
    $total = $Items.Count
    $i = 0
    
    foreach ($item in $Items) {
        $i++
        $pct = [math]::Round($i / $total * 100)
        Write-Progress -Activity $Label -Status "$i of $total" -PercentComplete $pct
        & $Action $item
    }
    Write-Progress -Activity $Label -Completed
}

Invoke-WithProgress (Get-ChildItem 'C:\Windows' -Filter '*.dll') {
    param($file)
    Start-Sleep -Milliseconds 10  # simulate work
} "Scanning DLLs"
```

---

**ก่อนหน้า ← [Part 08](Part-08.md) | ต่อไป → [Part 10: Functions](Part-10.md)**
