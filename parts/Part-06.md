# Part 06: Arrays & Collections

> **ระดับ**: 🟢 Beginner | **เวลา**: ~3 ชั่วโมง

---

## 1. สร้าง Array

```powershell
# วิธีต่างๆ ในการสร้าง array
$arr1 = @(1, 2, 3, 4, 5)          # explicit
$arr2 = 1, 2, 3, 4, 5             # implicit (comma)
$arr3 = 1..10                      # range
$arr4 = @()                        # empty array
$arr5 = ,"single"                  # single element

# Mixed types
$mixed = @(1, "hello", $true, (Get-Date), 3.14)

# Strongly typed
[int[]]$ints  = 1, 2, 3
[string[]]$strs = "a", "b", "c"

# 2D array
$matrix = @(
    @(1, 2, 3),
    @(4, 5, 6),
    @(7, 8, 9)
)
$matrix[0][0]  # 1
$matrix[1][2]  # 6
$matrix[2][1]  # 8
```

---

## 2. Array Indexing

```powershell
$arr = @("zero", "one", "two", "three", "four")

# Positive index (0-based)
$arr[0]         # zero
$arr[1]         # one
$arr[4]         # four

# Negative index (from end)
$arr[-1]        # four (last)
$arr[-2]        # three
$arr[-0]        # zero! (same as [0])

# Slice / Range
$arr[1..3]      # one, two, three
$arr[0..2]      # zero, one, two
$arr[-3..-1]    # two, three, four

# Multiple indices
$arr[0, 2, 4]   # zero, two, four

# Check bounds
if ($arr.Count -gt 3) { $arr[3] }
```

---

## 3. Array Operations

```powershell
$arr = @(5, 3, 1, 4, 2)

# Count
$arr.Count     # 5
$arr.Length    # 5 (same)

# Sort
$sorted = $arr | Sort-Object          # 1,2,3,4,5
$sorted_desc = $arr | Sort-Object -Descending

# Sort objects by property
$people = @(
    [PSCustomObject]@{Name="Charlie"; Age=30}
    [PSCustomObject]@{Name="Alice";   Age=25}
    [PSCustomObject]@{Name="Bob";     Age=28}
)
$people | Sort-Object Age
$people | Sort-Object Name, Age

# Filter
$arr | Where-Object { $_ -gt 3 }    # 5, 4
$arr | Where-Object { $_ % 2 -eq 0 } # 4, 2

# Contains
$arr -contains 3      # True
3 -in $arr            # True (PS4+)

# Index of
$arr.IndexOf(4)       # 3
[array]::IndexOf($arr, 4)  # 3

# Reverse
[array]::Reverse($arr)     # in-place!
$reversed = $arr[($arr.Length-1)..0]  # new reversed

# Unique
@(1,2,2,3,3,4) | Select-Object -Unique  # 1,2,3,4

# Join to string
$arr -join ", "        # 5, 3, 1, 4, 2

# Measure
$arr | Measure-Object -Sum -Average -Maximum -Minimum
```

---

## 4. Adding & Removing Elements

```powershell
# Array ใน PowerShell เป็น fixed-size!
# การ += สร้าง array ใหม่ทุกครั้ง!

$arr = @(1, 2, 3)
$arr += 4          # creates new array @(1,2,3,4)
$arr += @(5, 6)    # add multiple

# Remove elements
$arr = $arr | Where-Object { $_ -ne 3 }  # remove value 3
$arr = $arr[0..1]                         # keep first 2

# การใช้ += ไม่มีประสิทธิภาพ! ใช้ ArrayList หรือ Listแทน
Write-Warning "Avoid += for large arrays - O(n) performance!"
```

---

## 5. ArrayList - Dynamic Array

```powershell
# ArrayList สามารถ Add/Remove ได้เร็ว
$list = [System.Collections.ArrayList]::new()

# Add
$null = $list.Add("Apple")
$null = $list.Add("Banana")
$null = $list.Add("Cherry")
# ใช้ $null = ... เพื่อหน้า index return

# AddRange
$null = $list.AddRange(@("Date", "Elderberry"))

# Remove
$list.Remove("Banana")     # remove by value
$list.RemoveAt(0)          # remove by index

# Insert
$list.Insert(1, "Blueberry")

# Count
$list.Count

# Access
$list[0]
$list[-1]

# Sort
$list.Sort()

# Contains
$list.Contains("Apple")

# Clear
# $list.Clear()

# Convert to array
$array = $list.ToArray()
[string[]]$stringArray = $list
```

---

## 6. Generic List (Best Practice)

```powershell
# Generic List<T> - แนะนำที่สุด
$list = [System.Collections.Generic.List[string]]::new()

# Add
$list.Add("PowerShell")
$list.Add("Python")
$list.Add("Go")

# AddRange
$list.AddRange([string[]]@("Rust", "Java"))

# Remove
$list.Remove("Python")   # True/False
$list.RemoveAt(0)

# Contains
$list.Contains("Go")

# Find
$found = $list.Find({ param($x) $x.StartsWith("G") })

# Sort
$list.Sort()

# Typed - no casting needed!
$intList = [System.Collections.Generic.List[int]]::new()
$intList.Add(42)
$intList.Add(100)
$intList.Sum()  # error - use Measure-Object
($intList | Measure-Object -Sum).Sum

# Object List
$objList = [System.Collections.Generic.List[PSObject]]::new()
$objList.Add([PSCustomObject]@{Name="Alice"; Score=95})
$objList.Add([PSCustomObject]@{Name="Bob";   Score=87})
$objList | Sort-Object Score -Descending
```

---

## 7. Array Methods

```powershell
# ใช้ static methods ของ [array] class

$arr = @(3, 1, 4, 1, 5, 9, 2, 6)

# Sort (in-place)
[array]::Sort($arr)
# $arr = 1,1,2,3,4,5,6,9

# BinarySearch (ต้อง sort ก่อน!)
[array]::BinarySearch($arr, 4)    # index 3

# Reverse (in-place)
[array]::Reverse($arr)

# IndexOf
[array]::IndexOf($arr, 5)         # first occurrence

# LastIndexOf
$dup = @(1,2,3,2,1)
[array]::LastIndexOf($dup, 2)     # 3

# Copy
$dest = New-Object int[] 5
[array]::Copy($arr, $dest, 5)    # copy first 5

# Resize
[array]::Resize([ref]$arr, 5)    # resize to 5
```

---

## 8. LINQ-style Operations (Pipeline)

```powershell
$numbers = 1..20

# Select (map)
$squares = $numbers | ForEach-Object { $_ * $_ }

# Where (filter)
$evens = $numbers | Where-Object { $_ % 2 -eq 0 }
$gt10  = $numbers | Where-Object { $_ -gt 10 }

# Aggregate
($numbers | Measure-Object -Sum).Sum       # 210
($numbers | Measure-Object -Average).Average  # 10.5

# First/Last/Skip/Take
$numbers | Select-Object -First 5   # 1-5
$numbers | Select-Object -Last 5    # 16-20
$numbers | Select-Object -Skip 10   # 11-20
$numbers | Select-Object -First 5 -Skip 5  # 6-10

# Any/All equivalent
$hasNeg = ($numbers | Where-Object { $_ -lt 0 }).Count -gt 0
$allPos = ($numbers | Where-Object { $_ -gt 0 }).Count -eq $numbers.Count

# Distinct
@(1,1,2,2,3) | Select-Object -Unique

# Zip (combine two arrays)
$a = 1..3
$b = 'a','b','c'
$zipped = for ($i=0; $i -lt [math]::Min($a.Count,$b.Count); $i++) {
    [PSCustomObject]@{ Num=$a[$i]; Letter=$b[$i] }
}
# {1,a}, {2,b}, {3,c}

# Flatten nested
$nested = @(@(1,2), @(3,4), @(5,6))
$flat = $nested | ForEach-Object { $_ }  # 1,2,3,4,5,6
```

---

## 9. Practical Examples

```powershell
# สตาก (Stack) ด้วย List
function New-Stack {
    $stack = [System.Collections.Generic.Stack[object]]::new()
    $stack
}

$stack = [System.Collections.Generic.Stack[int]]::new()
$stack.Push(1)
$stack.Push(2)
$stack.Push(3)
$stack.Pop()     # 3 (LIFO)
$stack.Peek()    # 2 (don't remove)
$stack.Count     # 2

# คิว (Queue)
$queue = [System.Collections.Generic.Queue[string]]::new()
$queue.Enqueue("First")
$queue.Enqueue("Second")
$queue.Enqueue("Third")
$queue.Dequeue()  # First (FIFO)
$queue.Peek()     # Second

# กรองไฟล์ logs และสวอบ
function Get-LogErrors {
    param([string]$LogFile, [int]$MaxErrors = 100)
    
    $errors = [System.Collections.Generic.List[PSObject]]::new()
    
    Get-Content $LogFile | ForEach-Object {
        if ($_ -match '^(?<date>[\d-]+ [\d:]+) ERROR: (?<msg>.+)') {
            $errors.Add([PSCustomObject]@{
                Time    = $Matches['date']
                Message = $Matches['msg']
            })
        }
    }
    
    return $errors | Select-Object -Last $MaxErrors
}

# เก็บผลลัพธ์ loop อย่างมีประสิทธิภาพ
$results = [System.Collections.Generic.List[PSObject]]::new()

1..100 | ForEach-Object {
    $results.Add([PSCustomObject]@{
        Number  = $_
        Square  = $_ * $_
        IsEven  = $_ % 2 -eq 0
    })
}

$results | Where-Object IsEven | Select-Object -First 10
```

---

## 10. Exercises

```powershell
# Ex1: Merge sorted arrays
function Merge-SortedArrays {
    param([int[]]$A, [int[]]$B)
    $result = [System.Collections.Generic.List[int]]::new()
    $i = $j = 0
    while ($i -lt $A.Count -and $j -lt $B.Count) {
        if ($A[$i] -le $B[$j]) { $result.Add($A[$i++]) }
        else                   { $result.Add($B[$j++]) }
    }
    while ($i -lt $A.Count) { $result.Add($A[$i++]) }
    while ($j -lt $B.Count) { $result.Add($B[$j++]) }
    return $result.ToArray()
}

Merge-SortedArrays @(1,3,5,7) @(2,4,6,8)  # 1,2,3,4,5,6,7,8

# Ex2: Rotate array
function Rotate-Array {
    param([array]$Arr, [int]$Steps = 1)
    $n = $Arr.Count
    $k = (($Steps % $n) + $n) % $n  # handle negative/large
    return $Arr[$k..($n-1)] + $Arr[0..($k-1)]
}

Rotate-Array @(1,2,3,4,5) 2    # 3,4,5,1,2
Rotate-Array @(1,2,3,4,5) -1   # 5,1,2,3,4

# Ex3: Matrix transpose
function Transpose-Matrix {
    param([array[]]$Matrix)
    $rows = $Matrix.Count
    $cols = $Matrix[0].Count
    $result = @()
    for ($j = 0; $j -lt $cols; $j++) {
        $row = @()
        for ($i = 0; $i -lt $rows; $i++) {
            $row += $Matrix[$i][$j]
        }
        $result += ,$row
    }
    return $result
}

$m = @(@(1,2,3), @(4,5,6))
$t = Transpose-Matrix $m
# @(@(1,4), @(2,5), @(3,6))
```

---

**ก่อนหน้า ← [Part 05](Part-05.md) | ต่อไป → [Part 07: Hash Tables](Part-07.md)**
