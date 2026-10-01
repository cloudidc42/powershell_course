# Part 07: Hash Tables & Dictionaries

> **ระดับ**: 🟢 Beginner | **เวลา**: ~3 ชั่วโมง

---

## 1. สร้าง Hashtable

```powershell
# พื้นฐาน
$ht = @{}
$ht = @{ Key1 = "Value1"; Key2 = "Value2" }

# Multiline (readable)
$config = @{
    Server      = "db.example.com"
    Port        = 5432
    Database    = "myapp"
    Username    = "admin"
    MaxPoolSize = 100
    Timeout     = 30
    EnableSSL   = $true
}

# Ordered hashtable (maintains insertion order)
$ordered = [ordered]@{
    First  = 1
    Second = 2
    Third  = 3
}

# Nested
$nested = @{
    Server = @{
        Host = "localhost"
        Port = 8080
    }
    Database = @{
        Name = "mydb"
        User = "admin"
    }
}
```

---

## 2. เข้าถึงข้อมูล

```powershell
$ht = @{ Name="Alice"; Age=30; City="Bangkok" }

# เข้าถึง
$ht["Name"]     # Alice
$ht.Name        # Alice (dot notation)
$ht['Age']      # 30

# Nested
$ht['Key']['SubKey']
$nested.Server.Host   # localhost
$nested.Server['Port']  # 8080

# Get with default
$ht['Missing']                # null
$ht['Missing'] ?? 'default'   # default (PS7+)

if ($ht.ContainsKey('Name')) {
    Write-Host $ht['Name']
}

# All keys, values
$ht.Keys
$ht.Values
$ht.Count

# Enumerate
foreach ($kvp in $ht.GetEnumerator()) {
    Write-Host "$($kvp.Key) = $($kvp.Value)"
}
```

---

## 3. เพิ่ม/แก้ไข/ลบ

```powershell
$ht = @{ A = 1; B = 2 }

# เพิ่ม
$ht['C'] = 3
$ht.D = 4
$ht.Add('E', 5)

# แก้ไข
$ht['A'] = 10
$ht.B = 20

# ลบ
$ht.Remove('A')

# Merge two hashtables
$defaults = @{ Color = "Blue"; Size = "M"; Type = "Shirt" }
$custom   = @{ Color = "Red"; Size = "L" }

$merged = $defaults.Clone()
foreach ($k in $custom.Keys) {
    $merged[$k] = $custom[$k]
}
# @{ Color="Red"; Size="L"; Type="Shirt" }

# Merge function
function Merge-Hashtable {
    param([hashtable]$Base, [hashtable]$Override)
    $result = $Base.Clone()
    foreach ($key in $Override.Keys) {
        $result[$key] = $Override[$key]
    }
    return $result
}
```

---

## 4. PSCustomObject

```powershell
# แปลง hashtable เป็น object
$person = [PSCustomObject]@{
    Name   = "Alice"
    Age    = 30
    Email  = "alice@example.com"
    Skills = @("PowerShell", "Python", "Git")
}

# Properties
$person.Name
$person.Age
$person.Skills[0]

# เพิ่ม property
$person | Add-Member -NotePropertyName 'Department' -NotePropertyValue 'IT'
$person.Department  # IT

# Add multiple properties
$person | Add-Member -NotePropertyMembers @{
    Manager = "Bob"
    Level   = 3
}

# Array of objects
$team = @(
    [PSCustomObject]@{ Name="Alice"; Role="Dev";  Salary=80000 }
    [PSCustomObject]@{ Name="Bob";   Role="QA";   Salary=65000 }
    [PSCustomObject]@{ Name="Carol"; Role="DevOps";Salary=90000 }
)

$team | Sort-Object Salary -Descending | Format-Table -AutoSize
($team | Measure-Object Salary -Average).Average
$team | Where-Object { $_.Role -eq "Dev" }

# Export/Import
$team | Export-Csv 'team.csv' -NoTypeInformation
$imported = Import-Csv 'team.csv'
$team | ConvertTo-Json | Out-File 'team.json'
$fromJson = Get-Content 'team.json' | ConvertFrom-Json
```

---

## 5. Dictionary<K,V>

```powershell
# Generic Dictionary - faster than hashtable for large data
$dict = [System.Collections.Generic.Dictionary[string,int]]::new()

$dict['one']   = 1
$dict['two']   = 2
$dict['three'] = 3

# ContainsKey/Value
$dict.ContainsKey('one')    # True
$dict.ContainsValue(2)      # True

# TryGetValue
if ($dict.TryGetValue('two', [ref]$val)) {
    Write-Host "Found: $val"
}

# Iterate
foreach ($kv in $dict) {
    "$($kv.Key) = $($kv.Value)"
}

$dict.Keys
$dict.Values
$dict.Count

# Remove
$dict.Remove('one')

# Case-insensitive
$caseInsensitive = [System.Collections.Generic.Dictionary[string,string]]::new(
    [System.StringComparer]::OrdinalIgnoreCase
)
$caseInsensitive['Hello'] = 'World'
$caseInsensitive['HELLO']   # World
```

---

## 6. Practical Examples

```powershell
# Word frequency counter
function Get-WordFrequency {
    param([string]$Text)
    
    $freq = @{}
    $words = $Text.ToLower() -split '[^a-z0-9]+' | Where-Object { $_ }
    
    foreach ($word in $words) {
        if ($freq.ContainsKey($word)) {
            $freq[$word]++
        } else {
            $freq[$word] = 1
        }
    }
    
    $freq.GetEnumerator() | 
        Sort-Object Value -Descending | 
        Select-Object @{N='Word';E={$_.Key}}, @{N='Count';E={$_.Value}}
}

$text = "the quick brown fox jumps over the lazy dog the fox"
Get-WordFrequency $text
# Word   Count
# ----   -----
# the    3
# fox    2
# ...

# In-memory cache
$cache = @{}
$cacheExpiry = @{}

function Get-Cached {
    param([string]$Key, [scriptblock]$Fetch, [int]$TtlSeconds = 60)
    
    $now = Get-Date
    if ($cache.ContainsKey($Key) -and 
        $cacheExpiry[$Key] -gt $now) {
        Write-Verbose "Cache hit: $Key"
        return $cache[$Key]
    }
    
    Write-Verbose "Cache miss: $Key"
    $value = & $Fetch
    $cache[$Key] = $value
    $cacheExpiry[$Key] = $now.AddSeconds($TtlSeconds)
    return $value
}

# Usage
$data = Get-Cached 'processes' { Get-Process } 30

# Configuration manager
class Config {
    hidden [hashtable] $data = @{}
    hidden [hashtable] $defaults = @{}
    
    Config([hashtable]$Defaults) {
        $this.defaults = $Defaults
    }
    
    [object] Get([string]$Key) {
        if ($this.data.ContainsKey($Key)) { return $this.data[$Key] }
        if ($this.defaults.ContainsKey($Key)) { return $this.defaults[$Key] }
        return $null
    }
    
    [void] Set([string]$Key, [object]$Value) {
        $this.data[$Key] = $Value
    }
    
    [void] LoadFromFile([string]$Path) {
        if (Test-Path $Path) {
            $content = Get-Content $Path | ConvertFrom-Json
            $content.PSObject.Properties | ForEach-Object {
                $this.data[$_.Name] = $_.Value
            }
        }
    }
}

$cfg = [Config]::new(@{ Timeout = 30; MaxRetries = 3 })
$cfg.Get('Timeout')    # 30 (from defaults)
$cfg.Set('Timeout', 60)
$cfg.Get('Timeout')    # 60 (from data)
```

---

**ก่อนหน้า ← [Part 06](Part-06.md) | ต่อไป → [Part 08: Control Flow](Part-08.md)**
