# Part 05: String Manipulation

> **ระดับ**: 🟢 Beginner | **เวลา**: ~3 ชั่วโมง

---

## 1. String Creation

```powershell
# Single quotes - literal string
$s1 = 'Hello World'
$s2 = 'It''s a test'     # escape single quote with ''
$s3 = 'Path: C:\Users'   # backslash is literal

# Double quotes - interpolation
$name = "Alice"
$s4 = "Hello $name"             # Hello Alice
$s5 = "2 + 2 = $(2 + 2)"        # 2 + 2 = 4
$s6 = "Today: $(Get-Date -Format 'yyyy-MM-dd')"
$s7 = "Tab:`tNewline:`n"        # escape sequences

# Escape sequences in double-quoted strings
# `n  = newline
# `t  = tab
# `r  = carriage return
# `0  = null
# `a  = alert/bell
# `b  = backspace
# ``  = literal backtick
# `$  = literal dollar sign
# `"  = literal double quote

$path = "C:`$env:USERPROFILE"   # literal $
$msg  = "She said `"hello`""

# Here-String
$html = @"
<!DOCTYPE html>
<html>
<body>
    <h1>Hello, $name!</h1>
    <p>Date: $(Get-Date -Format 'yyyy-MM-dd')</p>
</body>
</html>
"@

# Here-String (single quote - no interpolation)
$template = @'
{
  "name": "$name",
  "version": "$version"
}
'@
```

---

## 2. String Methods

```powershell
$str = "  Hello, PowerShell World!  "

# Case
$str.ToUpper()       # "  HELLO, POWERSHELL WORLD!  "
$str.ToLower()       # "  hello, powershell world!  "

# Trim
$str.Trim()          # "Hello, PowerShell World!"
$str.TrimStart()     # "Hello, PowerShell World!  "
$str.TrimEnd()       # "  Hello, PowerShell World!"
$str.Trim('!')       # trim specific character

# Substring
$s = "Hello World"
$s.Substring(6)       # "World"  (from index 6)
$s.Substring(6, 3)    # "Wor"   (from 6, length 3)
$s[0]                 # 'H'
$s[0..4] -join ''     # "Hello"
$s[-5..-1] -join ''   # "World"

# Search
$s.IndexOf("World")      # 6
$s.IndexOf("o")          # 4 (first)
$s.LastIndexOf("o")      # 7 (last)
$s.IndexOf("xyz")        # -1 (not found)
$s.Contains("World")     # True
$s.StartsWith("Hello")   # True
$s.EndsWith("World")     # True

# Replace
$s.Replace("World", "PowerShell")    # Hello PowerShell
$s.Replace("l", "L")                 # HeLLo WorLd

# Split
"a,b,c,d" .Split(',')     # @('a','b','c','d')
"hello world" .Split(' ')  # @('hello','world')
"1::2::3" .Split('::')    # @('1','','2','','3')
"1::2::3" .Split([string[]]['::'], [StringSplitOptions]::RemoveEmptyEntries)
# @('1','2','3')

# Join
$parts = @('Hello', 'World', 'PowerShell')
$parts -join ', '          # Hello, World, PowerShell
[string]::Join('-', $parts) # Hello-World-PowerShell

# Pad
"Hi".PadLeft(10)           # "        Hi"
"Hi".PadRight(10)          # "Hi        "
"Hi".PadLeft(10, '*')      # "********Hi"
"42".PadLeft(6, '0')       # "000042"
```

---

## 3. Format Operator (-f)

```powershell
# Format: "template" -f value1, value2, ...

# Basic
"Hello, {0}!" -f "World"          # Hello, World!
"{0} + {1} = {2}" -f 3, 4, 7     # 3 + 4 = 7

# Reuse
"{0} loves {0}" -f "PowerShell"   # PowerShell loves PowerShell

# Numbers
"{0:N2}" -f 3.14159      # 3.14     (2 decimal places)
"{0:N0}" -f 1000000      # 1,000,000 (comma separator)
"{0:F4}" -f 3.14159      # 3.1416
"{0:P1}" -f 0.75         # 75.0%
"{0:C2}" -f 1234.5       # $1,234.50 (currency)
"{0:E2}" -f 12345678     # 1.23E+007
"{0:X}" -f 255           # FF (hex)
"{0:b}" -f 10            # 1010 (binary, PS7+)
"{0:08b}" -f 10          # 00001010 (padded binary)

# Width and alignment
"{0,-10}{1,10}" -f "Left", "Right"  # Left           Right
"{0,5}" -f "Hi"                      # "   Hi" (right align)
"{0,-5}" -f "Hi"                     # "Hi   " (left align)

# Date/Time
"{0:yyyy-MM-dd}" -f (Get-Date)       # 2024-01-15
"{0:HH:mm:ss}" -f (Get-Date)        # 14:30:45
"{0:dddd, MMMM d}" -f (Get-Date)    # Monday, January 15

# Table formatting
$data = @(
    [PSCustomObject]@{ Name="Alice"; Score=95.5; Grade="A" }
    [PSCustomObject]@{ Name="Bob";   Score=82.3; Grade="B" }
    [PSCustomObject]@{ Name="Carol"; Score=78.8; Grade="C" }
)

foreach ($row in $data) {
    "{0,-10} {1,6:N1} {2,5}" -f $row.Name, $row.Score, $row.Grade
}
# Alice       95.5     A
# Bob         82.3     B
# Carol       78.8     C
```

---

## 4. Regular Expressions

```powershell
# -match
"hello123" -match '\d+'      # True; $Matches[0] = "123"
"test@email.com" -match '@'  # True

# Named captures
"John Smith, 30" -match '(?<name>[\w ]+), (?<age>\d+)'
$Matches['name']   # John Smith
$Matches['age']    # 30

# -replace with regex
"Hello World" -replace '\s+', '_'    # Hello_World
"abc123def" -replace '\d+', '[NUM]'  # abc[NUM]def
"HELLO" -replace '[A-Z]', { $_.Value.ToLower() }  # hello

# Select-String (grep-like)
"error: file not found" | Select-String 'error: (.+)'
$match = $Matches[1]   # "file not found"

# Regex object
$regex = [regex]'\b\w{5}\b'   # words with exactly 5 chars
$text = "Hello World PowerShell rocks today"
$regex.Matches($text) | ForEach-Object { $_.Value }
# Hello, World, rocks, today

# Test match
[regex]::IsMatch("test@email.com", '^[\w.]+@[\w.]+\.[a-z]{2,}$')

# Replace with callback
$result = [regex]::Replace("hello world", '\b\w', { $args[0].Value.ToUpper() })
# Hello World

# Split by regex
"one1two2three3four" -split '\d'  # one, two, three, four
```

---

## 5. String Operators (-split, -join, -replace)

```powershell
# -split
"a,b,c" -split ','           # a, b, c
"hello world" -split ' '     # hello, world
"a::b::c" -split '::'        # a, b, c
"one1two2three" -split '\d'  # one, two, three (regex)

# -split with limit
"a,b,c,d" -split ',', 2      # a, "b,c,d" (max 2 parts)

# -join
$arr = 'a','b','c'
$arr -join '-'               # a-b-c
$arr -join ''                # abc
@(1,2,3) -join '+'           # 1+2+3

# -replace
"Hello World" -replace 'World', 'PS'   # Hello PS
"aAbBcC" -replace '[a-z]', '_'         # _A_B_C (regex)
"2024-01-15" -replace '-', '/'         # 2024/01/15

# -ireplace (case insensitive, default)
"HELLO" -ireplace 'hello', 'hi'   # hi

# -creplace (case sensitive)
"HELLO hello" -creplace 'hello', 'HI'  # HELLO HI (only lowercase replaced)
```

---

## 6. String Comparison

```powershell
# Case-insensitive (default)
"Hello" -eq "HELLO"    # True
"Hello" -eq "hello"    # True

# Case-sensitive (prefix c)
"Hello" -ceq "HELLO"   # False
"Hello" -ceq "Hello"   # True

# Comparison
"apple" -lt "banana"   # True (alphabetical)
"b" -gt "a"            # True

# .NET Compare
[string]::Compare("hello", "HELLO", $true)   # 0 (equal, ignore case)
[string]::Compare("hello", "HELLO", $false)  # non-zero (case-sensitive)

# .Equals
"hello".Equals("HELLO")  # False (case-sensitive by default)
"hello".Equals("HELLO", [System.StringComparison]::OrdinalIgnoreCase)  # True
```

---

## 7. String Building

```powershell
# StringBuilder - เร็วกว่าสำหรับการต่อ string หลายครั้ง
$sb = [System.Text.StringBuilder]::new()

for ($i = 1; $i -le 10; $i++) {
    $null = $sb.AppendLine("Line $i")
}
$result = $sb.ToString()
Write-Host $result

# เปรียบเทียบ performance
# BAD - สร้าง string ใหม่ทุก iteration
$bad = ""
for ($i = 0; $i -lt 10000; $i++) {
    $bad += "x"  # O(n^2) memory!
}

# GOOD - StringBuilder
$sb = [System.Text.StringBuilder]::new(10000)
for ($i = 0; $i -lt 10000; $i++) {
    $null = $sb.Append("x")
}
$good = $sb.ToString()

# ALSO GOOD for PS - collect then join
$parts = 1..10000 | ForEach-Object { "x" }
$result = $parts -join ''
```

---

## 8. Encoding & Conversion

```powershell
# Base64
$text = "Hello, PowerShell!"
$bytes = [System.Text.Encoding]::UTF8.GetBytes($text)
$b64 = [System.Convert]::ToBase64String($bytes)
Write-Host "Base64: $b64"  # SGVsbG8sIFBvd2VyU2hlbGwh

# Decode
$decoded = [System.Text.Encoding]::UTF8.GetString(
    [System.Convert]::FromBase64String($b64)
)
Write-Host "Decoded: $decoded"  # Hello, PowerShell!

# URL Encode/Decode
Add-Type -AssemblyName System.Web
$url = "https://example.com/search?q=hello world&lang=th"
$encoded = [System.Web.HttpUtility]::UrlEncode($url)
$decoded = [System.Web.HttpUtility]::UrlDecode($encoded)

# HTML Encode/Decode
$html = [System.Web.HttpUtility]::HtmlEncode("<script>alert('xss')</script>")
# &lt;script&gt;alert(&#39;xss&#39;)&lt;/script&gt;

# String to bytes
$bytes = [System.Text.Encoding]::UTF8.GetBytes("Hello")
$bytes  # 72 101 108 108 111

# Bytes to hex
$hex = ($bytes | ForEach-Object { $_.ToString("X2") }) -join ' '
# 48 65 6C 6C 6F
```

---

## 9. Practical String Functions

```powershell
# Truncate with ellipsis
function Truncate-String {
    param([string]$Text, [int]$MaxLength = 50, [string]$Suffix = "...")
    if ($Text.Length -le $MaxLength) { return $Text }
    return $Text.Substring(0, $MaxLength - $Suffix.Length) + $Suffix
}

Truncate-String "This is a very long string that should be truncated" 20
# This is a very l...

# Title Case
function ConvertTo-TitleCase {
    param([string]$Text)
    $culture = [System.Globalization.CultureInfo]::CurrentCulture
    return $culture.TextInfo.ToTitleCase($Text.ToLower())
}

ConvertTo-TitleCase "hello world"   # Hello World

# Slug (URL-friendly)
function ConvertTo-Slug {
    param([string]$Text)
    $slug = $Text.ToLower()
    $slug = $slug -replace '[^a-z0-9\s-]', ''
    $slug = $slug -replace '\s+', '-'
    $slug = $slug.Trim('-')
    return $slug
}

ConvertTo-Slug "Hello World! (2024)"   # hello-world-2024

# Word count
function Get-WordCount {
    param([string]$Text)
    ($Text.Trim() -split '\s+').Count
}

Get-WordCount "The quick brown fox jumps"   # 5

# Repeat string
function Repeat-String {
    param([string]$Str, [int]$Times)
    $Str * $Times
}

Repeat-String "=-" 20   # =-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-

# Mask sensitive data
function Mask-String {
    param([string]$Value, [int]$ShowLast = 4, [char]$MaskChar = '*')
    if ($Value.Length -le $ShowLast) { return $MaskChar * $Value.Length }
    $masked = $MaskChar * ($Value.Length - $ShowLast)
    return $masked + $Value.Substring($Value.Length - $ShowLast)
}

Mask-String "4532015112830366"  # ************0366
Mask-String "secretpassword" 0  # **************
```

---

## 10. Exercises

```powershell
# Ex1: Email validator
function Test-EmailAddress {
    param([string]$Email)
    $pattern = '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    [PSCustomObject]@{
        Email   = $Email
        IsValid = $Email -match $pattern
    }
}

@('user@example.com', 'bad-email', 'test@.com', 'hello@world.co.th') |
    ForEach-Object { Test-EmailAddress $_ }

# Ex2: Log parser
$logs = @"
2024-01-15 10:23:45 INFO Application started
2024-01-15 10:24:12 ERROR Connection failed: timeout
2024-01-15 10:24:15 WARN Retrying connection (1/3)
2024-01-15 10:24:18 INFO Connection restored
"@

$pattern = '^(?<date>[\d-]+) (?<time>[\d:]+) (?<level>\w+) (?<message>.+)$'
$logs -split '`n' | Where-Object { $_ } | ForEach-Object {
    if ($_ -match $pattern) {
        [PSCustomObject]@{
            DateTime = "$($Matches['date']) $($Matches['time'])"
            Level    = $Matches['level']
            Message  = $Matches['message']
        }
    }
} | Format-Table -AutoSize

# Ex3: Template engine
function Expand-Template {
    param([string]$Template, [hashtable]$Data)
    $result = $Template
    foreach ($key in $Data.Keys) {
        $result = $result -replace "\{\{$key\}\}", $Data[$key]
    }
    return $result
}

$template = "Dear {{name}},\nYour order #{{order}} is ready.\nTotal: {{total}}"
$data = @{ name = "Alice"; order = "12345"; total = "\$99.00" }
Write-Host (Expand-Template $template $data)

# Ex4: CSV to Markdown table
function ConvertTo-MarkdownTable {
    param([string]$CsvContent)
    $rows = $CsvContent -split '`n' | Where-Object { $_ }
    $headers = $rows[0] -split ','
    $separator = $headers | ForEach-Object { '---' }
    
    "| $($headers -join ' | ') |"
    "| $($separator -join ' | ') |"
    
    $rows[1..($rows.Count-1)] | ForEach-Object {
        $cols = $_ -split ','
        "| $($cols -join ' | ') |"
    }
}

$csv = "Name,Age,Role\nAlice,30,Dev\nBob,25,QA\nCarol,35,DevOps"
ConvertTo-MarkdownTable $csv
```

---

**ก่อนหน้า ← [Part 04](Part-04.md) | ต่อไป → [Part 06: Arrays & Collections](Part-06.md)**
