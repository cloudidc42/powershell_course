# Part 03: Variables & Data Types

> **ระดับ**: 🟢 Beginner | **เวลา**: ~3 ชั่วโมง

---

## 1. การประกาศตัวแปร

ใน PowerShell ตัวแปรขึ้นต้นด้วย `$` เสมอ

```powershell
# การประกาศตัวแปรพื้นฐาน
$name    = "PowerShell"
$version = 7
$pi      = 3.14159
$enabled = $true

# Multiple assignment
$a = $b = $c = 0

# Swap values
$x = 10; $y = 20
$x, $y = $y, $x
Write-Host "x=$x, y=$y"  # x=20, y=10

# ตัวแปรหลายตัวพร้อมกัน
$first, $second, $rest = 1, 2, 3, 4, 5
# $first=1, $second=2, $rest=@(3,4,5)
```

---

## 2. Data Types หลัก

### 2.1 String

```powershell
# Single quotes - literal (ไม่แทนค่าตัวแปร)
$a = 'Hello $name'       # Hello $name

# Double quotes - interpolation
$name = 'World'
$b = "Hello $name"       # Hello World
$c = "2 + 2 = $(2+2)"    # 2 + 2 = 4

# Here-String (multiline)
$text = @"
บรรทัดที่ 1
บรรทัดที่ 2: $name
บรรทัดที่ 3
"@

# Here-String single-quote (no interpolation)
$raw = @'
This is $literal
No interpolation here
'@

# String type
[string]$s = "Hello"
$s.GetType().Name  # String
```

### 2.2 Integer Types

```powershell
[int]$i       = 42              # Int32 (-2B to 2B)
[long]$l      = 9999999999L     # Int64
[byte]$b      = 255             # 0-255
[sbyte]$sb    = -128            # -128 to 127
[int16]$s16   = 32767
[int64]$s64   = [long]::MaxValue
[uint32]$u    = 4294967295      # unsigned

# Numeric literals
$hex  = 0xFF        # 255 (hex)
$oct  = 0o17        # 15 (octal, PS7+)
$bin  = 0b1010      # 10 (binary, PS7+)
$big  = 1_000_000   # 1000000 (readable)

# KB, MB, GB shortcuts
$size1 = 1KB    # 1024
$size2 = 1MB    # 1048576
$size3 = 1GB    # 1073741824
$size4 = 1TB    # 1099511627776
$size5 = 1PB    # 1125899906842624

Write-Host "1 GB = $([math]::Round(1GB/1MB, 0)) MB"
```

### 2.3 Floating Point

```powershell
[double]$d   = 3.14159265358979  # Double precision
[float]$f    = 3.14f              # Single precision
[decimal]$m  = 9.99m             # Decimal (precise)

# เปรียบเทียบ precision
[double]$x  = 0.1 + 0.2
[decimal]$y = 0.1m + 0.2m

Write-Host "double:  $x"   # 0.30000000000000004
Write-Host "decimal: $y"   # 0.3  (ถูกต้อง!)

# Math operations
[math]::Round(3.14159, 2)    # 3.14
[math]::Floor(3.7)           # 3
[math]::Ceiling(3.2)         # 4
[math]::Abs(-5)              # 5
[math]::Sqrt(16)             # 4
[math]::Pow(2, 10)           # 1024
[math]::Log(100, 10)         # 2
```

### 2.4 Boolean

```powershell
[bool]$true_val  = $true
[bool]$false_val = $false

# Truthy / Falsy
# Truthy: non-zero numbers, non-empty strings, non-null objects
# Falsy: 0, "", $null, empty array

if (1)          { "1 is truthy" }
if ("hello")    { "non-empty string is truthy" }
if ($null)      { } else { "null is falsy" }
if (0)          { } else { "0 is falsy" }
if (@())        { } else { "empty array is falsy" }

# Conversion
[bool]0          # False
[bool]1          # True
[bool]""         # False
[bool]"hello"    # True
[bool]$null      # False
```

### 2.5 DateTime

```powershell
# สร้าง DateTime
$now     = Get-Date
$today   = [datetime]::Today
$specific = [datetime]"2024-01-15 09:30:00"
$parsed  = Get-Date "Jan 15, 2024"

# Properties
$now.Year
$now.Month
$now.Day
$now.Hour
$now.Minute
$now.Second
$now.DayOfWeek      # Monday, Tuesday...
$now.DayOfYear      # 1-366

# Formatting
$now.ToString('yyyy-MM-dd')         # 2024-01-15
$now.ToString('dd/MM/yyyy HH:mm')   # 15/01/2024 09:30
$now.ToString('dddd, MMMM d, yyyy') # Monday, January 15, 2024
Get-Date -Format 'yyyy-MM-dd'

# DateTime math
$future = $now.AddDays(30)
$past   = $now.AddMonths(-6)
$diff   = $future - $now
$diff.Days       # 30

# Compare
if ($now -gt [datetime]"2020-01-01") { "After 2020" }

# Unix timestamp
$epoch = [DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
```

### 2.6 Null

```powershell
# $null คือ absence of value
$x = $null

# ตรวจสอบ null
if ($x -eq $null) { "x is null" }
if ($null -eq $x) { "safer: put null on left" }  # best practice

# Null coalescing (PowerShell 7+)
$result = $x ?? "default value"   # "default value"

# Null conditional (PowerShell 7+)
$length = $x?.Length              # null (ไม่ error)
$str = "Hello"
$len = $str?.Length               # 5

# Null vs empty string
$null -eq ""    # False
$null -eq $null # True
[string]::IsNullOrEmpty($null)    # True
[string]::IsNullOrEmpty("")       # True
[string]::IsNullOrWhiteSpace(" ") # True
```

---

## 3. Type System & Conversion

### 3.1 ตรวจสอบ Type

```powershell
# GetType()
"Hello".GetType()             # String
42.GetType()                  # Int32
$true.GetType()               # Boolean
(Get-Date).GetType()          # DateTime

# FullName
"Hello".GetType().FullName    # System.String
42.GetType().FullName         # System.Int32

# -is operator
"Hello" -is [string]          # True
42 -is [int]                  # True
42 -is [double]               # False
(Get-Date) -is [datetime]     # True

# -isnot operator
"Hello" -isnot [int]          # True
```

### 3.2 Type Conversion

```powershell
# Explicit cast
[int]"42"              # 42
[int]3.7               # 3 (truncate, not round)
[string]42             # "42"
[double]"3.14"         # 3.14
[bool]1                # True
[datetime]"2024-01-15" # DateTime

# Parse methods
[int]::Parse("42")
[double]::Parse("3.14")
[datetime]::Parse("2024-01-15")
[bool]::Parse("true")

# TryParse (ปลอดภัยกว่า)
$result = 0
if ([int]::TryParse("42", [ref]$result)) {
    Write-Host "Parsed: $result"
} else {
    Write-Host "Parse failed"
}

# Convert class
[System.Convert]::ToInt32("42")
[System.Convert]::ToDouble("3.14")
[System.Convert]::ToString(42)
[System.Convert]::ToBoolean(1)

# -as operator (ไม่ error ถ้าแปลงไม่ได้)
"42" -as [int]          # 42
"hello" -as [int]       # null (ไม่ error)
```

---

## 4. Special Variables

```powershell
# Automatic variables สำคัญ
$_          # Current pipeline object (= $PSItem)
$PSItem     # Same as $_
$args       # Arguments ที่ไม่ได้ declare ใน function
$error      # Array ของ errors ที่เกิดขึ้น
$?          # True ถ้า last command สำเร็จ
$LASTEXITCODE  # Exit code ของ last external command

# Environment
$PSVersionTable      # Version info
$PSCommandPath       # Path ของ script ปัจจุบัน
$PSScriptRoot        # Directory ของ script
$MyInvocation        # Info เกี่ยวกับ current command
$Host                # PowerShell host info
$HOME                # Home directory
$env:USERPROFILE     # User profile path

# Console
$PSStyle             # ANSI styling (PS7.2+)
$Error               # Error collection
$null                # Null value
$true / $false       # Boolean values

# ตัวอย่างการใช้
Get-Process | ForEach-Object {
    # $_ คือ current process
    if ($_.CPU -gt 10) {
        Write-Host "$($_.Name): CPU=$([math]::Round($_.CPU,1))"
    }
}

# $? และ $LASTEXITCODE
git status
if ($?) { "git succeeded" } else { "git failed" }

ping localhost -n 1 | Out-Null
if ($LASTEXITCODE -eq 0) { "ping OK" }
```

---

## 5. Variable Scope

```powershell
# Scope levels: Global > Script > Local > Private

# Global - accessible everywhere
$Global:counter = 0

function Increment {
    $Global:counter++
    Write-Host "Counter: $Global:counter"
}
Increment  # Counter: 1
Increment  # Counter: 2
Write-Host "Final: $Global:counter"  # Final: 2

# Script scope - accessible in script, not child functions by default
$Script:config = "production"

function GetConfig {
    Write-Host "Config: $Script:config"
}

# Local (default) - only in current scope
function TestScope {
    $local_var = "I'm local"
    Write-Host $local_var  # works
}
TestScope
# Write-Host $local_var  # error - undefined

# Private - cannot be seen by child scopes
$Private:secret = "hidden"

function TryAccess {
    Write-Host $secret  # empty - cannot see private
}

# Scope modifier in function
function SetGlobal {
    param([string]$Value)
    Set-Variable -Name 'SharedValue' -Value $Value -Scope Global
}

SetGlobal "Hello from function"
Write-Host $SharedValue  # Hello from function
```

---

## 6. Strongly Typed Variables

```powershell
# กำหนด type ให้ตัวแปร
[int]$age = 25
$age = 30        # OK
$age = "thirty"  # Error! Cannot convert

[string]$name = "Alice"
$name = 42       # 42 converts to "42"

[datetime]$birthday = "1995-05-15"
$birthday.Year   # 1995

[uri]$url = "https://example.com"
$url.Host        # example.com
$url.Scheme      # https

[ipaddress]$ip = "192.168.1.1"
$ip.AddressFamily  # InterNetwork

[version]$ver = "7.4.0"
$ver.Major    # 7
$ver.Minor    # 4

# Array typing
[int[]]$numbers = 1, 2, 3
[string[]]$names = "Alice", "Bob", "Carol"
```

---

## 7. Working with Variables

```powershell
# Get/Set Variable cmdlets
Set-Variable -Name 'myVar' -Value 42
Get-Variable -Name 'myVar'
Get-Variable  # all variables

# ดู variables ที่มีอยู่
Get-Variable | Where-Object { $_.Name -notlike '_*' } | 
    Select-Object Name, Value | Format-Table

# Remove variable
Remove-Variable -Name 'myVar'
Remove-Variable -Name 'myVar' -ErrorAction SilentlyContinue

# Variable existence
if (Test-Path Variable:myVar) {
    Write-Host "myVar exists"
}

# Get-Variable with scope
Get-Variable -Scope Global
Get-Variable -Scope Script

# Read-only variable
New-Variable -Name 'PI' -Value 3.14159 -Option ReadOnly
# $PI = 3.14  # Error!

# Constant variable
New-Variable -Name 'MAX_SIZE' -Value 100 -Option Constant
# Cannot modify or remove
```

---

## 8. Practical Examples

### Config Management

```powershell
# ใช้ตัวแปรเพื่อ config
$config = [PSCustomObject]@{
    Environment = 'Production'
    ServerName  = 'web01'
    Port        = 443
    Timeout     = [timespan]::FromSeconds(30)
    MaxRetries  = 3
    EnableSSL   = $true
    StartDate   = [datetime]'2024-01-01'
}

Write-Host "Server: $($config.ServerName):$($config.Port)"
Write-Host "Environment: $($config.Environment)"
if ($config.EnableSSL) { Write-Host "SSL enabled" }
```

### User Input

```powershell
# รับ input จากผู้ใช้
$username = Read-Host "Enter username"
$password = Read-Host "Enter password" -AsSecureString
$age      = [int](Read-Host "Enter age")

Write-Host "Welcome, $username!"
Write-Host "Age: $age"

# Validate input
do {
    $num = Read-Host "Enter a number (1-10)"
    $valid = [int]::TryParse($num, [ref]$null) -and [int]$num -ge 1 -and [int]$num -le 10
    if (!$valid) { Write-Host "Invalid! Try again" -ForegroundColor Red }
} until ($valid)

Write-Host "You entered: $num"
```

---

## 9. Exercises

```powershell
# Exercise 1: Type exploration
$values = @(42, 3.14, "hello", $true, $null, (Get-Date))
foreach ($v in $values) {
    $type = if ($null -eq $v) { "null" } else { $v.GetType().Name }
    Write-Host "Value: '$v' -> Type: $type"
}

# Exercise 2: Type conversion
$inputs = @("42", "3.14", "true", "2024-01-15", "100")
foreach ($inp in $inputs) {
    Write-Host "'$inp' -> int: $($inp -as [int]) | double: $($inp -as [double])"
}

# Exercise 3: Calculator
function Calculate {
    param([double]$A, [string]$Op, [double]$B)
    switch ($Op) {
        '+' { $A + $B }
        '-' { $A - $B }
        '*' { $A * $B }
        '/' { if ($B -ne 0) { $A / $B } else { "Error: div by zero" } }
        '%' { $A % $B }
    }
}

Calculate 10 '+' 5   # 15
Calculate 10 '/' 3   # 3.333...
Calculate 10 '/' 0   # Error

# Exercise 4: Profile variables
$profile_data = [PSCustomObject]@{
    Name      = Read-Host "Your name"
    BirthYear = [int](Read-Host "Birth year")
    Language  = "PowerShell"
    JoinDate  = Get-Date
}

$age = (Get-Date).Year - $profile_data.BirthYear
Write-Host "Name: $($profile_data.Name), Age: $age"
Write-Host "Using $($profile_data.Language) since $($profile_data.JoinDate.Year)"
```

---

**ก่อนหน้า ← [Part 02](Part-02.md) | ต่อไป → [Part 04: Operators](Part-04.md)**
