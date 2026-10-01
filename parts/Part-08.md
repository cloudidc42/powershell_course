# Part 08: Control Flow - If/Else/Switch

> **ระดับ**: 🟢 Beginner | **เวลา**: ~2 ชั่วโมง

---

## 1. If / ElseIf / Else

```powershell
# พื้นฐาน
$x = 15

if ($x -gt 10) {
    Write-Host "Greater than 10"
}

if ($x -gt 20) {
    Write-Host "Greater than 20"
} elseif ($x -gt 10) {
    Write-Host "Greater than 10"
} else {
    Write-Host "10 or less"
}

# One-liner (short form)
if ($x -gt 10) { Write-Host "Yes" }

# As expression (returns value)
$result = if ($x -gt 10) { "big" } else { "small" }

# Nested
if ($x -gt 0) {
    if ($x -lt 100) {
        Write-Host "Between 1 and 99"
    }
}

# Compound conditions
$age = 20; $hasID = $true
if ($age -ge 18 -and $hasID) {
    Write-Host "Can enter"
}

# Multiple OR
$role = "Admin"
if ($role -eq "Admin" -or $role -eq "SuperAdmin" -or $role -eq "Owner") {
    Write-Host "Has admin access"
}

# Better with -in
if ($role -in @("Admin", "SuperAdmin", "Owner")) {
    Write-Host "Has admin access"
}
```

---

## 2. Switch Statement

```powershell
# พื้นฐาน
$day = "Monday"

switch ($day) {
    "Monday"    { Write-Host "Start of work week" }
    "Friday"    { Write-Host "End of work week" }
    "Saturday"  { Write-Host "Weekend!" }
    "Sunday"    { Write-Host "Weekend!" }
    default     { Write-Host "Regular workday" }
}

# Switch กับ numbers
$score = 85
switch ($score) {
    { $_ -ge 90 } { Write-Host "A"; break }
    { $_ -ge 80 } { Write-Host "B"; break }
    { $_ -ge 70 } { Write-Host "C"; break }
    { $_ -ge 60 } { Write-Host "D"; break }
    default        { Write-Host "F" }
}

# Multiple matches (without break)
$x = 2
switch ($x) {
    1 { Write-Host "One" }
    2 { Write-Host "Two" }
    2 { Write-Host "Also Two" }  # Both execute!
    3 { Write-Host "Three" }
}
# Output: Two\nAlso Two

# Switch array input
switch (1, 2, 3, "abc") {
    1       { Write-Host "Got 1" }
    2       { Write-Host "Got 2" }
    "abc"   { Write-Host "Got abc" }
}
```

---

## 3. Switch แบบ Advanced

```powershell
# -Wildcard
$filename = "report_2024.xlsx"
switch -Wildcard ($filename) {
    "*.xlsx"  { Write-Host "Excel file" }
    "*.csv"   { Write-Host "CSV file" }
    "*.txt"   { Write-Host "Text file" }
    "report*" { Write-Host "Report file" }
    default   { Write-Host "Unknown" }
}
# Both "*.xlsx" and "report*" match!

# -Regex
$text = "Error: Connection timeout after 30s"
switch -Regex ($text) {
    '^Error'   { Write-Host "ERROR detected: $_"; break }
    '^Warning' { Write-Host "Warning detected"; break }
    '^Info'    { Write-Host "Info: $_"; break }
    default    { Write-Host "Unknown log level" }
}

# -CaseSensitive
$cmd = "Exit"
switch -CaseSensitive ($cmd) {
    "exit" { Write-Host "lowercase exit" }
    "EXIT" { Write-Host "uppercase EXIT" }
    "Exit" { Write-Host "Title Case Exit" }  # This matches
}

# -File (read from file, switch each line)
# switch -File 'data.txt' {
#     'error'   { Write-Host "Found error" }
#     default   { }
# }

# Return value from switch
$category = switch ($score) {
    { $_ -ge 90 } { "Excellent" }
    { $_ -ge 75 } { "Good" }
    { $_ -ge 60 } { "Pass" }
    default        { "Fail" }
}
Write-Host "Category: $category"
```

---

## 4. Ternary & Null Coalescing

```powershell
# Ternary (PS7+)
$age = 20
$status = $age -ge 18 ? "Adult" : "Minor"

# PowerShell 5.1 way
$status = if ($age -ge 18) { "Adult" } else { "Minor" }

# Null coalescing (PS7+)
$value = $null
$result = $value ?? "default"  # "default"

$value = "exists"
$result = $value ?? "default"  # "exists"

# Null coalescing assignment
$x = $null
$x ??= "assigned"  # now $x = "assigned"

# Chained null coalescing
$a = $null; $b = $null; $c = "found"
$result = $a ?? $b ?? $c  # "found"
```

---

## 5. Guard Clauses Pattern

```powershell
# แทนที่จะใช้ nested if (ยากอ่าน)
function Process-File-Bad {
    param([string]$Path)
    
    if (Test-Path $Path) {
        $content = Get-Content $Path
        if ($content.Count -gt 0) {
            if ($content[0] -match '^#!') {
                # do stuff
            } else {
                Write-Host "Invalid file"
            }
        } else {
            Write-Host "Empty file"
        }
    } else {
        Write-Host "File not found"
    }
}

# ดีกว่า: Guard clauses
function Process-File-Good {
    param([string]$Path)
    
    if (!(Test-Path $Path)) {
        Write-Host "File not found"
        return
    }
    
    $content = Get-Content $Path
    if ($content.Count -eq 0) {
        Write-Host "Empty file"
        return
    }
    
    if ($content[0] -notmatch '^#!') {
        Write-Host "Invalid file"
        return
    }
    
    # ทำงานหลัก (happy path)
    Write-Host "Processing $Path..."
}
```

---

## 6. Practical Examples

```powershell
# Status code handler
function Get-HttpStatusMessage {
    param([int]$Code)
    
    switch -Regex ($Code.ToString()) {
        '^2'   { return "Success ($Code)" }
        '^3'   { return "Redirect ($Code)" }
        '^4'   { 
            switch ($Code) {
                400 { return "Bad Request" }
                401 { return "Unauthorized" }
                403 { return "Forbidden" }
                404 { return "Not Found" }
                429 { return "Too Many Requests" }
                default { return "Client Error ($Code)" }
            }
        }
        '^5'   {
            switch ($Code) {
                500 { return "Internal Server Error" }
                502 { return "Bad Gateway" }
                503 { return "Service Unavailable" }
                default { return "Server Error ($Code)" }
            }
        }
        default { return "Unknown ($Code)" }
    }
}

@(200, 301, 401, 404, 500, 503) | ForEach-Object {
    Write-Host "$_: $(Get-HttpStatusMessage $_)"
}

# File processor
function Process-FileByType {
    param([string]$Path)
    
    if (!(Test-Path $Path)) { 
        throw "File not found: $Path" 
    }
    
    $ext = [System.IO.Path]::GetExtension($Path).ToLower()
    
    switch ($ext) {
        '.csv'  { Import-Csv $Path }
        '.json' { Get-Content $Path | ConvertFrom-Json }
        '.xml'  { [xml](Get-Content $Path) }
        '.txt'  { Get-Content $Path }
        '.log'  { Get-Content $Path | Select-String 'ERROR|WARN' }
        default { Write-Warning "Unsupported: $ext" }
    }
}
```

---

**ก่อนหน้า ← [Part 07](Part-07.md) | ต่อไป → [Part 09: Loops](Part-09.md)**
