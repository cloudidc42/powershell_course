# Part 14: CSV, JSON และ XML

> **ระดับ**: 🟡 Intermediate | **เวลา**: ~3 ชั่วโมง

---

## 1. CSV Operations

```powershell
# Export
$users = @(
    [PSCustomObject]@{Name='Alice'; Age=30; Email='alice@example.com'; Role='Admin'}
    [PSCustomObject]@{Name='Bob';   Age=25; Email='bob@example.com';   Role='User'}
    [PSCustomObject]@{Name='Carol'; Age=28; Email='carol@example.com'; Role='Manager'}
)

$users | Export-Csv 'users.csv' -NoTypeInformation -Encoding UTF8

# Export ด้วย delimiter เฟ้น
$users | Export-Csv 'users_tab.tsv' -Delimiter "`t" -NoTypeInformation

# Import
$loaded = Import-Csv 'users.csv'
$loaded | ForEach-Object { Write-Host "$($_.Name) ($($_.Role))" }

# Import ด้วย header
$noHeader = Import-Csv 'data.csv' -Header 'Name','Score','Grade'

# แปลง type หลัง Import (CSV ทุกอย่างเป็น string!)
$loaded | ForEach-Object {
    $_.Age = [int]$_.Age  # cast string to int
}

# ConvertTo/From
$csv = $users | ConvertTo-Csv -NoTypeInformation
$back = $csv | ConvertFrom-Csv

# กรอง CSV ขนาดใหญ่ด้วย StreamReader
function Import-LargeCSV {
    param([string]$Path, [scriptblock]$Filter)
    
    $reader = [System.IO.StreamReader]::new($Path)
    $header = $reader.ReadLine() -split ','
    
    try {
        while (!$reader.EndOfStream) {
            $values = $reader.ReadLine() -split ','
            $obj = [PSCustomObject]@{}
            for ($i = 0; $i -lt $header.Count; $i++) {
                $obj | Add-Member -NotePropertyName $header[$i].Trim('"') -NotePropertyValue $values[$i].Trim('"')
            }
            if (!$Filter -or (& $Filter $obj)) {
                $obj
            }
        }
    } finally {
        $reader.Dispose()
    }
}
```

---

## 2. JSON Operations

```powershell
# ConvertTo-Json
$data = [PSCustomObject]@{
    Name    = "App Config"
    Version = "2.1.0"
    Features = @("auth", "logging", "caching")
    Database = @{
        Host = "localhost"
        Port = 5432
        Name = "myapp"
    }
}

$json = $data | ConvertTo-Json -Depth 10
Write-Host $json

# Depth: ต้อง set depth ให้ครอบ nested objects!
$complex = [PSCustomObject]@{ a = @{ b = @{ c = @{ d = "deep" } } } }
$complex | ConvertTo-Json           # ตัดที่ระดับ 2
$complex | ConvertTo-Json -Depth 10 # เต็ม

# ConvertFrom-Json
$jsonString = '{"name":"Alice","age":30,"active":true}'
$obj = $jsonString | ConvertFrom-Json
$obj.name    # Alice
$obj.age     # 30 (int!)
$obj.active  # True (bool!)

# อ่าน/เขียน JSON file
$config = Get-Content 'config.json' -Raw | ConvertFrom-Json
$config | ConvertTo-Json -Depth 10 | Set-Content 'config_out.json' -Encoding UTF8

# แก้ไข JSON (PSCustomObject เป็น read-only?)
$config = Get-Content 'config.json' | ConvertFrom-Json
# PSCustomObject สามารถแก้ไข property ได้
$config.version = "3.0"

# PS7+: ConvertFrom-Json -AsHashtable
$ht = $jsonString | ConvertFrom-Json -AsHashtable
$ht['name']  # Alice
```

---

## 3. JSON API Response

```powershell
# เรียก REST API
function Invoke-ApiRequest {
    param(
        [string]$Uri,
        [hashtable]$Headers = @{},
        [string]$Method = 'GET',
        [object]$Body = $null
    )
    
    $params = @{
        Uri     = $Uri
        Method  = $Method
        Headers = $Headers + @{ 'Content-Type' = 'application/json' }
        ErrorAction = 'Stop'
    }
    
    if ($Body) {
        $params.Body = $Body | ConvertTo-Json -Depth 10
    }
    
    try {
        $response = Invoke-RestMethod @params
        return $response
    } catch [System.Net.WebException] {
        $statusCode = [int]$_.Exception.Response.StatusCode
        throw "HTTP $statusCode : $($_.Exception.Message)"
    }
}

# Usage
$users = Invoke-ApiRequest 'https://jsonplaceholder.typicode.com/users'
$users | Select-Object id, name, email | Format-Table

# เรียก POST
$newPost = Invoke-ApiRequest 'https://jsonplaceholder.typicode.com/posts' \
    -Method 'POST' \
    -Body @{ title='Test'; body='Content'; userId=1 }
```

---

## 4. XML Operations

```powershell
# สร้าง XML
$xml = [xml]@'
<?xml version="1.0" encoding="UTF-8"?>
<catalog>
  <book id="bk101">
    <author>Corets, Eva</author>
    <title>Maeve Ascendant</title>
    <genre>Fantasy</genre>
    <price>5.95</price>
    <publish_date>2000-11-17</publish_date>
  </book>
  <book id="bk102">
    <author>Ralls, Kim</author>
    <title>Midnight Rain</title>
    <genre>Fantasy</genre>
    <price>5.95</price>
    <publish_date>2000-12-16</publish_date>
  </book>
</catalog>
'@

# เข้าถึง
$xml.catalog.book[0].title       # Maeve Ascendant
$xml.catalog.book[0].price       # 5.95
$xml.catalog.book[0].id          # bk101 (attribute)

# วนทุก node
foreach ($book in $xml.catalog.book) {
    Write-Host "$($book.id): $($book.title) - $$($book.price)"
}

# SelectNodes / XPath
$fantasies = $xml.SelectNodes('//book[genre="Fantasy"]')
$fantasies.Count  # 2

$prices = $xml.SelectNodes('//price')
$total = ($prices | ForEach-Object { [double]$_.InnerText } | Measure-Object -Sum).Sum

# แก้ไข
$xml.catalog.book[0].price = '9.99'

# เพิ่ม node
$newBook = $xml.CreateElement('book')
$newBook.SetAttribute('id', 'bk103')
$title = $xml.CreateElement('title')
$title.InnerText = 'New Book'
$newBook.AppendChild($title) | Out-Null
$xml.catalog.AppendChild($newBook) | Out-Null

# บันทึก
$xml.Save('catalog.xml')

# อ่านจาก file
[xml]$cfg = Get-Content 'config.xml'
```

---

## 5. TOML และ YAML (PS7 + modules)

```powershell
# Install parser modules
# Install-Module powershell-yaml
# Install-Module PSToml (if available)

# YAML
Import-Module powershell-yaml

$yaml = @'
name: MyApp
version: 1.0
database:
  host: localhost
  port: 5432
features:
  - auth
  - logging
'@

$config = ConvertFrom-Yaml $yaml
$config.database.host    # localhost
$config.features         # array

# ConvertTo
$data = @{ name='Test'; items = @(1,2,3) }
ConvertTo-Yaml $data

# JSON -> hashtable -> YAML round-trip
$json = '{"a":1,"b":{"c":2}}'
$ht = $json | ConvertFrom-Json -AsHashtable
$ht | ConvertTo-Yaml
```

---

## 6. Practical: Config Manager

```powershell
class ConfigManager {
    [string]$FilePath
    [hashtable]$Data
    
    ConfigManager([string]$path) {
        $this.FilePath = $path
        $this.Data = @{}
        $this.Load()
    }
    
    [void] Load() {
        if (Test-Path $this.FilePath) {
            $ext = [System.IO.Path]::GetExtension($this.FilePath).ToLower()
            switch ($ext) {
                '.json' {
                    $this.Data = Get-Content $this.FilePath -Raw |
                        ConvertFrom-Json -AsHashtable
                }
                '.xml' {
                    # load XML config
                    $xml = [xml](Get-Content $this.FilePath)
                    # convert to hashtable...
                }
            }
        }
    }
    
    [void] Save() {
        $ext = [System.IO.Path]::GetExtension($this.FilePath).ToLower()
        switch ($ext) {
            '.json' {
                $this.Data | ConvertTo-Json -Depth 10 | 
                    Set-Content $this.FilePath -Encoding UTF8
            }
        }
    }
    
    [object] Get([string]$Key, [object]$Default = $null) {
        if ($this.Data.ContainsKey($Key)) { return $this.Data[$Key] }
        return $Default
    }
    
    [void] Set([string]$Key, [object]$Value) {
        $this.Data[$Key] = $Value
        $this.Save()
    }
}
```

---

**ก่อนหน้า ← [Part 13](Part-13.md) | ต่อไป → [Part 15: Modules](Part-15.md)**
