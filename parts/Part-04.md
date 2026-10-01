# Part 04: Operators

> **ระดับ**: 🟢 Beginner | **เวลา**: ~2 ชั่วโมง

---

## 1. Arithmetic Operators

```powershell
# พื้นฐาน
$a = 10
$b = 3

$a + $b     # 13  (addition)
$a - $b     # 7   (subtraction)
$a * $b     # 30  (multiplication)
$a / $b     # 3.333... (division)
$a % $b     # 1   (modulo/remainder)

# Power (ไม่มี ** แต่ใช้ [math]::Pow)
[math]::Pow($a, $b)   # 1000

# Unary
-$a         # -10
+$a         # 10

# String + (concatenation)
"Hello" + " " + "World"   # Hello World
"Ha" * 3                   # HaHaHa

# Array +
@(1,2) + @(3,4)   # 1,2,3,4

# Integer division
[math]::Truncate(10 / 3)   # 3
[math]::DivRem(10, 3, [ref]$r)  # quotient; $r = remainder

# Overflow behavior
[int]::MaxValue + 1   # OverflowException หรือ wrap
[int]::MinValue - 1   # OverflowException
```

---

## 2. Assignment Operators

```powershell
$x = 10

$x += 5    # $x = $x + 5  = 15
$x -= 3    # $x = $x - 3  = 12
$x *= 2    # $x = $x * 2  = 24
$x /= 4    # $x = $x / 4  = 6
$x %= 4    # $x = $x % 4  = 2

# Increment / Decrement
$x++       # post-increment: ใช้แล้วค่อยเพิ่ม
++$x       # pre-increment: เพิ่มก่อนแล้วค่อยใช้
$x--       # post-decrement
--$x       # pre-decrement

# Null coalescing assignment (PS7+)
$y = $null
$y ??= "default"   # $y = "default" (เพราะ $y เป็น null)

$z = "existing"
$z ??= "default"   # $z ยังคงเป็น "existing"

# String assignment
$str  = "Hello"
$str += " World"   # Hello World

# Array assignment
$arr  = @(1, 2)
$arr += 3          # @(1, 2, 3)
```

---

## 3. Comparison Operators

### 3.1 พื้นฐาน

```powershell
# -eq  Equal
5 -eq 5           # True
"abc" -eq "ABC"   # True  (case-insensitive by default!)
"abc" -ceq "ABC"  # False (case-sensitive)

# -ne  Not Equal
5 -ne 6           # True

# -lt  Less Than
3 -lt 5           # True

# -le  Less or Equal
5 -le 5           # True

# -gt  Greater Than
7 -gt 5           # True

# -ge  Greater or Equal
5 -ge 5           # True

# Case sensitivity prefix
# -eq   = case insensitive (default)
# -ceq  = case sensitive
# -ieq  = case insensitive (explicit)

"Hello" -eq "HELLO"   # True
"Hello" -ceq "HELLO"  # False
"Hello" -ieq "HELLO"  # True
```

### 3.2 String Comparison

```powershell
# -like  Wildcard matching
"PowerShell" -like "Power*"    # True
"PowerShell" -like "*Shell"    # True
"PowerShell" -like "*wer*"     # True
"PowerShell" -like "Power?"	  # False (? = single char)
"PowerS" -like "Power?"        # True

# -notlike
"CMD" -notlike "Power*"        # True

# -match  Regex matching
"PowerShell 7" -match "\d+"         # True
"hello@email.com" -match '@'        # True
"12345" -match '^\d{5}$'           # True

# After -match, $Matches contains captures
"John Smith" -match '(\w+) (\w+)'
$Matches[0]   # John Smith (full match)
$Matches[1]   # John
$Matches[2]   # Smith

# -notmatch
"abc" -notmatch '\d'   # True (no digits)

# -contains (array contains element)
@(1,2,3) -contains 2       # True
@(1,2,3) -notcontains 5    # True

# -in (element in array)
2 -in @(1,2,3)             # True
5 -notin @(1,2,3)          # True

# -replace
"Hello World" -replace "World", "PowerShell"  # Hello PowerShell
"abc123" -replace '\d+', 'NUM'               # abcNUM
```

---

## 4. Logical Operators

```powershell
# -and  Both must be true
$true -and $true     # True
$true -and $false    # False

# -or   At least one true
$true -or $false     # True
$false -or $false    # False

# -not  Negate
-not $true           # False
-not $false          # True
! $true              # False (same as -not)

# -xor  Exclusive or
$true -xor $false    # True
$true -xor $true     # False

# Short-circuit evaluation
$x = 5
($x -gt 3) -and ($x -lt 10)    # True (both evaluated)
($x -gt 10) -and ($x -lt 20)   # False (second NOT evaluated)
($x -gt 10) -or ($x -lt 20)    # True (second NOT evaluated)

# Practical example
$file = "C:\test.txt"
if ((Test-Path $file) -and (Get-Item $file).Length -gt 0) {
    Write-Host "File exists and has content"
}
```

---

## 5. Type Operators

```powershell
# -is   Check type
42 -is [int]           # True
42 -is [double]        # False
42 -is [System.Int32]  # True
"hello" -is [string]   # True

# -isnot
"hello" -isnot [int]   # True

# -as   Convert (returns null if fails)
"42" -as [int]          # 42
"hello" -as [int]       # null (no error!)
"2024-01-15" -as [datetime]   # DateTime object

# Practical: Safe conversion
function ConvertToInt {
    param([string]$Value)
    $result = $Value -as [int]
    if ($null -eq $result) {
        Write-Warning "'$Value' cannot be converted to int"
        return 0
    }
    return $result
}

ConvertToInt "42"      # 42
ConvertToInt "hello"   # Warning + 0
```

---

## 6. Bitwise Operators

```powershell
# -band  Bitwise AND
0b1100 -band 0b1010   # 0b1000 = 8
12 -band 10           # 8

# -bor   Bitwise OR
0b1100 -bor 0b1010    # 0b1110 = 14
12 -bor 10            # 14

# -bxor  Bitwise XOR
0b1100 -bxor 0b1010   # 0b0110 = 6
12 -bxor 10           # 6

# -bnot  Bitwise NOT
-bnot 12              # -13

# -shl  Shift left
1 -shl 3              # 8 (1 * 2^3)
5 -shl 2              # 20

# -shr  Shift right
8 -shr 1              # 4
20 -shr 2             # 5

# Practical: Flags
$READ    = 0b001  # 1
$WRITE   = 0b010  # 2
$EXECUTE = 0b100  # 4

$perms = $READ -bor $WRITE   # 3 (read + write)

# Check permissions
($perms -band $READ) -ne 0    # True (has read)
($perms -band $EXECUTE) -ne 0 # False (no execute)
```

---

## 7. Range Operator

```powershell
# .. Range operator
1..5              # 1, 2, 3, 4, 5
5..1              # 5, 4, 3, 2, 1
'a'..'e'          # a, b, c, d, e
'A'..'Z'          # A-Z

# ใช้ใน loop
foreach ($i in 1..10) {
    Write-Host "Item $i"
}

# ใน array
$array = 1..100
$array[0]    # 1
$array[-1]   # 100
$array[0..4] # 1,2,3,4,5

# String to array
[char[]]('a'..'z') -join ''   # abcdefghijklmnopqrstuvwxyz
```

---

## 8. Pipeline Chain Operators (PS7+)

```powershell
# && - run next only if previous succeeded
git add . && git commit -m "update" && git push

# || - run next only if previous failed
Get-Process notepad || Start-Process notepad

# ตัวอย่าง
Test-Path 'C:\temp' || New-Item 'C:\temp' -ItemType Directory
# ถ้า path ไม่มี ให้สร้าง

# Combined
Connect-VpnServer && Deploy-App || Write-Host "Deploy failed"
```

---

## 9. Redirection Operators

```powershell
# > Redirect to file (overwrite)
Get-Process > processes.txt

# >> Append to file
Get-Process >> processes.txt

# 2> Redirect errors
Get-Process badprocess 2> errors.txt

# 2>&1 Redirect errors to stdout
Get-Process * 2>&1 > all-output.txt

# *> Redirect all streams
Get-Process *> everything.txt

# Streams:
# 1 = stdout (Success)
# 2 = Error
# 3 = Warning
# 4 = Verbose
# 5 = Debug
# 6 = Information

# Verbose redirect
Get-ChildItem -Verbose 4>&1

# Suppress errors
Get-Process nonexistent 2>$null
```

---

## 10. Ternary Operator (PS7+)

```powershell
# Syntax: condition ? value_if_true : value_if_false
$age = 20
$status = $age -ge 18 ? "Adult" : "Minor"
Write-Host $status  # Adult

# Nested
$score = 75
$grade = $score -ge 90 ? "A" :
         $score -ge 80 ? "B" :
         $score -ge 70 ? "C" :
         $score -ge 60 ? "D" : "F"
Write-Host "Grade: $grade"  # C

# PowerShell 5.1 equivalent
$status = if ($age -ge 18) { "Adult" } else { "Minor" }
```

---

## 11. Practical Examples

```powershell
# ตรวจสอบ disk space
$drives = Get-PSDrive -PSProvider FileSystem
foreach ($drive in $drives) {
    if ($drive.Used -gt 0) {
        $pct = [math]::Round($drive.Used / ($drive.Used + $drive.Free) * 100, 1)
        $status = $pct -gt 90 ? "CRITICAL" : $pct -gt 75 ? "WARNING" : "OK"
        Write-Host "$($drive.Name): $pct% [$status]" -ForegroundColor (
            $pct -gt 90 ? "Red" : $pct -gt 75 ? "Yellow" : "Green"
        )
    }
}

# Password validator
function Test-PasswordStrength {
    param([string]$Password)
    
    $checks = [ordered]@{
        'Length >= 8'     = $Password.Length -ge 8
        'Has uppercase'   = $Password -cmatch '[A-Z]'
        'Has lowercase'   = $Password -cmatch '[a-z]'
        'Has digit'       = $Password -match '\d'
        'Has special'     = $Password -match '[!@#$%^&*]'
    }
    
    $score = ($checks.Values | Where-Object { $_ } | Measure-Object).Count
    
    foreach ($check in $checks.GetEnumerator()) {
        $icon = $check.Value ? "✅" : "❌"
        Write-Host "$icon $($check.Key)"
    }
    
    Write-Host "Score: $score/5 - $("Weak","Fair","Good","Strong","Very Strong","Excellent"[$score])"
}

Test-PasswordStrength "P@ssw0rd!"
```

---

**ก่อนหน้า ← [Part 03](Part-03.md) | ต่อไป → [Part 05: String Manipulation](Part-05.md)**
