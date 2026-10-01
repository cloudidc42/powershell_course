# Part 01: Introduction to PowerShell

> **ระดับ**: 🟢 Beginner  
> **เวลาที่ใช้**: ~3 ชั่วโมง  
> **เป้าหมาย**: เข้าใจ PowerShell คืออะไร ทำไมต้องใช้ และเริ่มต้นใช้งานได้

---

## 📌 สารบัญ

1. [PowerShell คืออะไร?](#1-powershell-คืออะไร)
2. [ประวัติและวิวัฒนาการ](#2-ประวัติและวิวัฒนาการ)
3. [PowerShell vs CMD vs Bash](#3-powershell-vs-cmd-vs-bash)
4. [Use Cases ของ PowerShell](#4-use-cases-ของ-powershell)
5. [Cmdlet คืออะไร](#5-cmdlet-คืออะไร)
6. [คำสั่งพื้นฐาน](#6-คำสั่งพื้นฐาน)
7. [การใช้ Help System](#7-การใช้-help-system)
8. [Pipeline คืออะไร](#8-pipeline-คืออะไร)
9. [Object-Based Shell](#9-object-based-shell)
10. [Exercises](#10-exercises)

---

## 1. PowerShell คืออะไร?

**PowerShell** คือ task automation และ configuration management framework จาก Microsoft ประกอบด้วย:

- **Command-line shell** - ใช้สั่งงานผ่าน command line
- **Scripting language** - เขียนสคริปต์อัตโนมัติได้
- **Configuration management platform** - จัดการการตั้งค่าระบบ

สิ่งที่ทำให้ PowerShell พิเศษคือมันทำงานกับ **.NET objects** ไม่ใช่แค่ text ธรรมดา ทำให้การจัดการข้อมูลทำได้ง่ายและมีประสิทธิภาพมากกว่า shell แบบดั้งเดิม

```powershell
# ตัวอย่างง่ายๆ - แสดง processes ที่กินหน่วยความจำมากที่สุด
Get-Process | Sort-Object WorkingSet -Descending | Select-Object -First 5 Name, WorkingSet
```

ผลลัพธ์:
```
Name         WorkingSet
----         ----------
chrome      524288000
code        312457216
Explorer    156327936
svchost      98304000
powershell   67108864
```

---

## 2. ประวัติและวิวัฒนาการ

### Timeline

```
ปี 2002  → เริ่มพัฒนา (ชื่อเดิม "Monad")
ปี 2006  → PowerShell 1.0 - Windows XP SP2, Server 2003
ปี 2008  → PowerShell 2.0 - Remoting, Background Jobs
ปี 2012  → PowerShell 3.0 - Workflows, Scheduled Jobs  
ปี 2013  → PowerShell 4.0 - DSC (Desired State Configuration)
ปี 2016  → PowerShell 5.0/5.1 - Classes, Package Management
ปี 2016  → PowerShell Core 6.0 - Cross-platform (Open Source!)
ปี 2018  → PowerShell Core 6.1/6.2 - Linux/macOS improvements
ปี 2020  → PowerShell 7.0 - .NET 5, ไม่ใช่ "Core" แล้ว
ปี 2021  → PowerShell 7.2 - LTS release
ปี 2022  → PowerShell 7.3
ปี 2023  → PowerShell 7.4 - .NET 8
ปี 2024  → PowerShell 7.5
```

### สองสาขาหลัก

| | Windows PowerShell | PowerShell (Core/7+) |
|---|---|---|
| Version | 5.1 (สุดท้าย) | 7.x (active) |
| Platform | Windows เท่านั้น | Windows, Linux, macOS |
| .NET | .NET Framework 4.x | .NET 6/7/8 |
| Open Source | ไม่ | ✅ GitHub |
| Status | Maintenance | Active development |
| Path | `C:\Windows\System32\WindowsPowerShell\v1.0\` | `C:\Program Files\PowerShell\7\` |

```powershell
# ตรวจสอบ version ที่ใช้อยู่
$PSVersionTable

# Output:
# Name                           Value
# ----                           -----
# PSVersion                      7.4.0
# PSEdition                      Core
# GitCommitId                    7.4.0
# OS                             Microsoft Windows 10.0.22621
# Platform                       Win32NT
# PSCompatibleVersions           {1.0, 2.0, 3.0, 4.0, 5.0, 5.1, 6.0, 6.1, 6.2, 7.0, 7.1, 7.2, 7.3, 7.4}
# PSRemotingProtocolVersion      2.3
# SerializationVersion           1.1.0.1
# WSManStackVersion              3.0
```

---

## 3. PowerShell vs CMD vs Bash

### เปรียบเทียบพื้นฐาน

| คุณสมบัติ | CMD | Bash | PowerShell |
|----------|-----|------|------------|
| Output type | Text | Text | **Objects** |
| Cross-platform | ❌ | ✅ | ✅ (7+) |
| OOP Support | ❌ | ❌ | ✅ |
| .NET Integration | ❌ | ❌ | ✅ |
| Piping | Text | Text | **Objects** |
| Error handling | Limited | Basic | **Advanced** |
| Module system | ❌ | Basic | **Rich** |
| Tab completion | Basic | Good | **Excellent** |

### ตัวอย่างเปรียบเทียบ: หา Process ที่ใช้ CPU มาก

**CMD (ทำไม่ได้โดยตรง):**
```cmd
tasklist /FO CSV | sort
# ได้แค่ text, ไม่สามารถ sort ตาม CPU usage ได้ง่ายๆ
```

**Bash:**
```bash
ps aux --sort=-%cpu | head -10
# ได้ text แต่ต้อง parse เองถ้าต้องการใช้ข้อมูล
```

**PowerShell:**
```powershell
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10 |
Select-Object Name, CPU, WorkingSet, @{N='MB';E={[math]::Round($_.WorkingSet/1MB,2)}}
# ได้ Objects พร้อมใช้ คำนวณได้ทันที
```

### Object vs Text Pipeline

```powershell
# Bash-style (text): ต้อง parse text
# Get processes -> text -> grep -> awk -> text

# PowerShell-style (objects): ใช้ properties โดยตรง
Get-Service |
    Where-Object { $_.Status -eq 'Running' } |  # filter ด้วย property
    Sort-Object DisplayName |                    # sort ด้วย property  
    Select-Object Name, DisplayName, Status      # เลือก properties ที่ต้องการ
```

---

## 4. Use Cases ของ PowerShell

### 4.1 System Administration

```powershell
# สร้าง user accounts จำนวนมากจาก CSV
$users = Import-Csv 'C:\users.csv'
foreach ($user in $users) {
    New-LocalUser -Name $user.Username -FullName $user.FullName -NoPassword
    Write-Host "Created: $($user.Username)" -ForegroundColor Green
}
```

### 4.2 Automation & Scripting

```powershell
# Backup ไฟล์อัตโนมัติทุกวัน
$source = 'C:\Projects'
$dest   = "D:\Backup\$(Get-Date -Format 'yyyy-MM-dd')"
Copy-Item -Path $source -Destination $dest -Recurse
Write-Host "Backup completed to $dest"
```

### 4.3 Network Administration

```powershell
# ตรวจสอบ server ว่า online หรือไม่
$servers = @('server01','server02','server03','db01','web01')
$results = $servers | ForEach-Object {
    [PSCustomObject]@{
        Server = $_
        Online = (Test-Connection $_ -Count 1 -Quiet)
        Time   = Get-Date
    }
}
$results | Format-Table -AutoSize
```

### 4.4 Security & Penetration Testing (Authorized)

```powershell
# สแกน open ports บน target ที่ได้รับอนุญาต
function Test-Port {
    param([string]$Computer, [int[]]$Ports)
    foreach ($port in $Ports) {
        $result = Test-NetConnection -ComputerName $Computer -Port $port -WarningAction SilentlyContinue
        [PSCustomObject]@{
            Port   = $port
            Open   = $result.TcpTestSucceeded
            Status = if ($result.TcpTestSucceeded) { 'OPEN' } else { 'CLOSED' }
        }
    }
}
# ใช้เฉพาะบน authorized systems!
Test-Port -Computer '192.168.1.100' -Ports 22,80,443,3389,8080
```

### 4.5 Cloud Management

```powershell
# Azure - สร้าง VM (ต้องติดตั้ง Az module)
Connect-AzAccount
New-AzVM -ResourceGroupName 'MyRG' -Name 'MyVM' -Location 'EastUS' `
         -Image 'Win2022Datacenter' -Size 'Standard_D2s_v3'
```

### 4.6 Web Development

```powershell
# สร้าง Simple HTTP Server ด้วย PowerShell
$listener = [System.Net.HttpListener]::new()
$listener.Prefixes.Add('http://localhost:8080/')
$listener.Start()
Write-Host 'Server running at http://localhost:8080/'

while ($listener.IsListening) {
    $context  = $listener.GetContext()
    $response = $context.Response
    $html     = "<h1>Hello from PowerShell!</h1><p>Time: $(Get-Date)</p>"
    $buffer   = [System.Text.Encoding]::UTF8.GetBytes($html)
    $response.ContentLength64 = $buffer.Length
    $response.OutputStream.Write($buffer, 0, $buffer.Length)
    $response.OutputStream.Close()
}
```

---

## 5. Cmdlet คืออะไร

**Cmdlet** (อ่านว่า "command-let") คือคำสั่งใน PowerShell ที่มีรูปแบบเป็น **Verb-Noun**

### Naming Convention

```
Verb  -  Noun
----     ----
Get  -  Process     → Get-Process
Set  -  Location    → Set-Location  
New  -  Item        → New-Item
Remove-Item         → Remove-Item
Start-Service       → Start-Service
Stop-Service        → Stop-Service
Test-Connection     → Test-Connection
Invoke-WebRequest   → Invoke-WebRequest
```

### Approved Verbs

```powershell
# ดู Verbs ที่ approved ทั้งหมด
Get-Verb

# Verbs หลักที่ใช้บ่อย:
# Get    - ดึงข้อมูล
# Set    - ตั้งค่า
# New    - สร้างใหม่
# Remove - ลบ
# Add    - เพิ่ม
# Start  - เริ่ม
# Stop   - หยุด
# Invoke - เรียกใช้งาน
# Test   - ทดสอบ
# Import - นำเข้า
# Export - ส่งออก
# Write  - เขียน/แสดงผล
# Read   - อ่าน
```

### Aliases

```powershell
# PowerShell มี aliases สำหรับ Unix/DOS compatibility
Get-Alias

# ตัวอย่าง aliases:
# ls    → Get-ChildItem
# cd    → Set-Location
# cp    → Copy-Item
# mv    → Move-Item
# rm    → Remove-Item
# cat   → Get-Content
# echo  → Write-Output
# clear → Clear-Host
# pwd   → Get-Location
# man   → Get-Help
# ps    → Get-Process

# ดู alias ทั้งหมด
Get-Alias | Sort-Object Name | Format-Table Name, Definition -AutoSize
```

---

## 6. คำสั่งพื้นฐาน

### 6.1 Navigation

```powershell
# ดู directory ปัจจุบัน
Get-Location      # เต็ม
pwd               # alias

# เปลี่ยน directory
Set-Location C:\Windows
cd C:\Windows      # alias

# กลับ directory ก่อนหน้า
pop
cd -              # PowerShell 6+

# แสดงไฟล์ใน directory
Get-ChildItem
ls                # alias
dir               # alias

# แสดงแบบละเอียด
Get-ChildItem -Force          # รวม hidden files
Get-ChildItem *.txt           # filter
Get-ChildItem -Recurse        # recursive
Get-ChildItem -Recurse *.log  # recursive + filter
```

### 6.2 Working with Files

```powershell
# อ่านไฟล์
Get-Content 'C:\Windows\System32\drivers\etc\hosts'
cat hosts.txt     # alias

# เขียนไฟล์
"Hello, World!" | Out-File 'test.txt'
Set-Content 'test.txt' -Value "New content"
Add-Content 'test.txt' -Value "Additional line"

# สร้างไฟล์/folder
New-Item -Path 'C:\Test\newfile.txt' -ItemType File
New-Item -Path 'C:\Test\newfolder'   -ItemType Directory
mkdir 'C:\Test\newfolder'            # alias

# คัดลอก/ย้าย/ลบ
Copy-Item 'source.txt' 'dest.txt'
Move-Item 'old.txt' 'new.txt'
Remove-Item 'file.txt'
Remove-Item 'folder' -Recurse         # ลบ folder
```

### 6.3 Displaying Output

```powershell
# แสดงข้อความ
Write-Host "Hello, World!"                           # to console
Write-Host "Error!" -ForegroundColor Red             # สีแดง
Write-Host "Success!" -ForegroundColor Green         # สีเขียว
Write-Host "Warning!" -ForegroundColor Yellow        # สีเหลือง

Write-Output "This goes to pipeline"                 # to pipeline
Write-Verbose "Debug message" -Verbose               # verbose message
Write-Warning "Something might be wrong"             # warning
Write-Error "Something went wrong"                   # error stream
Write-Debug "Debug info" -Debug                      # debug stream

# Format output
Get-Process | Format-Table    # table
Get-Process | Format-List     # list
Get-Process | Format-Wide     # wide
Get-Service | Out-GridView    # GUI grid (Windows)
```

### 6.4 Variables ขั้นต้น

```powershell
# กำหนดค่าตัวแปร
$name = "PowerShell"
$version = 7.4
$isEnabled = $true

# ใช้งานตัวแปร
Write-Host "Welcome to $name version $version"
Write-Host "Is enabled: $isEnabled"

# ดู variables ทั้งหมด
Get-Variable

# ลบ variable
Remove-Variable name
```

---

## 7. การใช้ Help System

Help system ของ PowerShell ดีมาก ควรเรียนรู้การใช้งาน

### 7.1 Get-Help

```powershell
# ดู help ของ cmdlet
Get-Help Get-Process
Get-Help Get-Process -Detailed    # รายละเอียด
Get-Help Get-Process -Full        # เต็ม
Get-Help Get-Process -Examples    # ตัวอย่าง เท่านั้น
Get-Help Get-Process -Online      # เปิด browser
Get-Help Get-Process -Parameter * # ดู parameters ทั้งหมด

# ค้นหา help
Get-Help *process*               # ค้นหาทุกอย่างที่มีคำว่า process
Get-Help about_*                 # conceptual help topics
Get-Help about_Pipeline          # pipeline concepts
Get-Help about_Variables         # variables
Get-Help about_Functions         # functions

# อัพเดท help files (ต้อง Run as Admin)
Update-Help
Update-Help -Force -ErrorAction SilentlyContinue
```

### 7.2 Get-Command

```powershell
# ค้นหา commands
Get-Command *process*           # ค้นหาตาม name pattern
Get-Command -Verb Get           # cmdlets ที่ใช้ Get
Get-Command -Noun Service       # cmdlets ที่จัดการ Service
Get-Command -Module ActiveDirectory  # cmdlets จาก module
Get-Command -CommandType Function    # functions เท่านั้น
Get-Command -CommandType Alias       # aliases เท่านั้น

# ดู syntax ของ command
Get-Command Get-Process -Syntax

# ดูว่า command อยู่ที่ไหน
Get-Command notepad
(Get-Command notepad).Source
```

### 7.3 Get-Member

```powershell
# ดู properties และ methods ของ objects
Get-Process | Get-Member
Get-Process | Get-Member -MemberType Property  # properties
Get-Process | Get-Member -MemberType Method    # methods

# ตัวอย่าง
"Hello" | Get-Member         # String methods
(Get-Date) | Get-Member      # DateTime methods
42 | Get-Member              # Int32 methods

# เทียบเท่า .GetType()
$x = "Hello"
$x.GetType()
$x | Get-Member
```

---

## 8. Pipeline คืออะไร

**Pipeline** (`|`) คือการส่งผลลัพธ์จาก cmdlet หนึ่งไปยังอีก cmdlet หนึ่ง

### 8.1 พื้นฐาน Pipeline

```powershell
# โครงสร้าง
# Command1 | Command2 | Command3 | Command4

# ตัวอย่างง่าย
Get-Process | Where-Object { $_.CPU -gt 10 } | Sort-Object CPU -Descending
#     ↑                    ↑                          ↑
# ดึง processes    กรองเฉพาะที่ CPU > 10    เรียงตาม CPU มากไปน้อย

# อีกตัวอย่าง
Get-Service |
    Where-Object Status -eq 'Running' |
    Select-Object Name, DisplayName |
    Sort-Object Name |
    Export-Csv 'running-services.csv' -NoTypeInformation
```

### 8.2 $_ และ $PSItem

```powershell
# $_ หรือ $PSItem คือ current object ใน pipeline
Get-Process | ForEach-Object {
    Write-Host "Process: $($_.Name) - PID: $($_.Id)" -ForegroundColor Cyan
}

# เหมือนกัน (PowerShell 3.0+)
Get-Process | ForEach-Object {
    Write-Host "Process: $($PSItem.Name)" -ForegroundColor Cyan
}

# Short syntax (PowerShell 7+)
Get-Process | ForEach-Object { "$($_.Name): CPU=$($_.CPU)" }
```

### 8.3 Pipeline ที่ทรงพลัง

```powershell
# นับจำนวน
Get-Process | Measure-Object               # count
Get-Process | Measure-Object CPU -Sum      # sum of CPU
Get-Process | Measure-Object WorkingSet -Average -Maximum -Minimum

# Group by
Get-Process | Group-Object Company | Sort-Object Count -Descending

# Select properties + calculated
Get-Process | Select-Object Name, 
    @{Name='CPU(s)'; Expression={[math]::Round($_.CPU, 2)}},
    @{Name='Memory(MB)'; Expression={[math]::Round($_.WorkingSet/1MB, 2)}}

# Filter ด้วย Where-Object
Get-EventLog -LogName System -Newest 100 | 
    Where-Object { $_.EntryType -eq 'Error' } |
    Select-Object TimeGenerated, Source, Message |
    Export-Csv 'system-errors.csv' -NoTypeInformation
```

---

## 9. Object-Based Shell

นี่คือความแตกต่างที่สำคัญที่สุดของ PowerShell

### 9.1 ทุกอย่างคือ Object

```powershell
# Get-Process ไม่คืน text - คืน System.Diagnostics.Process objects
$process = Get-Process -Name 'powershell'

# Properties ที่ใช้ได้
$process.Name
$process.Id
$process.CPU
$process.WorkingSet
$process.StartTime
$process.MainWindowTitle

# Methods ที่ใช้ได้
# $process.Kill()     # kill process
# $process.Refresh()  # refresh data

# String ก็เป็น object
$str = "Hello, PowerShell!"
$str.ToUpper()       # HELLO, POWERSHELL!
$str.Split(',')      # ['Hello', ' PowerShell!']
$str.Length          # 18
$str.Replace('Hello', 'Hi')  # Hi, PowerShell!
```

### 9.2 สร้าง Custom Objects

```powershell
# สร้าง custom object
$person = [PSCustomObject]@{
    Name    = "สมชาย"
    Age     = 30
    Email   = "somchai@example.com"
    IsAdmin = $false
}

# ใช้งาน
Write-Host "Name: $($person.Name)"
Write-Host "Age: $($person.Age)"

# สร้าง array of objects
$team = @(
    [PSCustomObject]@{ Name = "Alice"; Role = "Dev";  Exp = 5 }
    [PSCustomObject]@{ Name = "Bob";   Role = "QA";   Exp = 3 }
    [PSCustomObject]@{ Name = "Carol"; Role = "DevOps";Exp = 7 }
)

# ทำงานเหมือน table
$team | Format-Table -AutoSize
$team | Where-Object { $_.Exp -gt 4 }
$team | Sort-Object Exp -Descending
$team | Measure-Object Exp -Average
```

### 9.3 Type System

```powershell
# PowerShell มี Type system จาก .NET

# ดู type
"Hello".GetType().FullName          # System.String
(42).GetType().FullName             # System.Int32
(3.14).GetType().FullName           # System.Double
$true.GetType().FullName            # System.Boolean
(Get-Date).GetType().FullName       # System.DateTime
(Get-Process)[0].GetType().FullName # System.Diagnostics.Process

# Cast types
[int]"42"                # 42 (Int32)
[string]42               # "42"
[double]"3.14"           # 3.14
[datetime]"2024-01-15"   # DateTime object
[array]"single item"     # Array ที่มี 1 element
```

---

## 10. Exercises

### Exercise 1: First Commands

```powershell
# ทำตามขั้นตอน:
# 1. เปิด PowerShell terminal
# 2. ตรวจสอบ version
$PSVersionTable.PSVersion

# 3. ดู location ปัจจุบัน
Get-Location

# 4. ไปที่ Desktop (ปรับ path ตาม OS)
Set-Location "$env:USERPROFILE\Desktop"

# 5. สร้าง folder
New-Item -Path 'PSLearning' -ItemType Directory

# 6. ไปที่ folder ใหม่
Set-Location 'PSLearning'

# 7. สร้างไฟล์
"# My PowerShell Notes" | Out-File 'notes.md'

# 8. ดูไฟล์
Get-Content 'notes.md'

# 9. กลับ Desktop
Set-Location ..
```

### Exercise 2: Explore Cmdlets

```powershell
# 1. ค้นหา cmdlets เกี่ยวกับ Network
Get-Command *network*
Get-Command *net*

# 2. ดู help ของ Test-Connection
Get-Help Test-Connection -Examples

# 3. ทดสอบ ping
Test-Connection google.com -Count 3

# 4. ค้นหา cmdlets เกี่ยวกับ Service
Get-Command *service*

# 5. ดู services ที่กำลัง run
Get-Service | Where-Object Status -eq 'Running'

# 6. นับ services
Get-Service | Where-Object Status -eq 'Running' | Measure-Object
```

### Exercise 3: Pipeline Practice

```powershell
# 1. แสดง top 5 processes ตาม CPU
Get-Process | Sort-Object CPU -Descending | Select-Object -First 5

# 2. แสดง processes ที่ใช้ RAM > 100MB
Get-Process | Where-Object { $_.WorkingSet -gt 100MB } |
Select-Object Name, @{N='RAM(MB)'; E={[math]::Round($_.WorkingSet/1MB,2)}} |
Sort-Object 'RAM(MB)' -Descending

# 3. นับ processes ตาม company
Get-Process | Where-Object Company | Group-Object Company |
Sort-Object Count -Descending | Select-Object -First 5 Count, Name

# 4. ส่งออก process list ไป CSV
Get-Process | Select-Object Name, Id, CPU, WorkingSet |
Export-Csv 'processes.csv' -NoTypeInformation

# 5. Import กลับและแสดงผล
Import-Csv 'processes.csv' | Select-Object -First 5
```

### Exercise 4: Help System

```powershell
# 1. ดู help topics ทั้งหมด
Get-Help about_* | Select-Object Name

# 2. อ่าน about_Pipeline
Get-Help about_Pipeline

# 3. ค้นหา cmdlets เกี่ยวกับ file
Get-Command -Noun Item

# 4. ดู examples ของ Get-ChildItem
Get-Help Get-ChildItem -Examples

# 5. ดู members ของ string
"PowerShell" | Get-Member

# 6. ลองใช้ string methods
$text = "hello powershell world"
$text.ToUpper()
$text.Split(' ')
$text.Replace('world', 'universe')
$text.Contains('powershell')
```

### Exercise 5: Mini Project - System Info

```powershell
# สร้างสคริปต์แสดงข้อมูลระบบ

function Show-SystemInfo {
    Write-Host "=== System Information ==="     -ForegroundColor Cyan
    Write-Host ""
    
    # Computer info
    $os  = Get-CimInstance Win32_OperatingSystem
    $cpu = Get-CimInstance Win32_Processor
    $ram = Get-CimInstance Win32_PhysicalMemory
    
    Write-Host "Computer: $env:COMPUTERNAME"    -ForegroundColor Yellow
    Write-Host "OS: $($os.Caption)"              -ForegroundColor Yellow
    Write-Host "CPU: $($cpu.Name)"               -ForegroundColor Yellow
    Write-Host "RAM: $([math]::Round(($ram | Measure-Object Capacity -Sum).Sum / 1GB, 2)) GB" -ForegroundColor Yellow
    Write-Host "PowerShell: $($PSVersionTable.PSVersion)" -ForegroundColor Yellow
    Write-Host ""
    
    # Disk info
    Write-Host "=== Disk Usage ===" -ForegroundColor Cyan
    Get-PSDrive -PSProvider FileSystem | 
        Where-Object { $_.Used -gt 0 } |
        Select-Object Name,
            @{N='Total(GB)'; E={[math]::Round($_.Used/1GB + $_.Free/1GB, 2)}},
            @{N='Used(GB)';  E={[math]::Round($_.Used/1GB, 2)}},
            @{N='Free(GB)';  E={[math]::Round($_.Free/1GB, 2)}},
            @{N='Used%';     E={[math]::Round($_.Used/($_.Used+$_.Free)*100, 1)}} |
        Format-Table -AutoSize
    
    # Running services count
    $svcCount = (Get-Service | Where-Object Status -eq 'Running').Count
    Write-Host "Running Services: $svcCount" -ForegroundColor Yellow
    
    # Process count
    Write-Host "Running Processes: $((Get-Process).Count)" -ForegroundColor Yellow
}

# เรียกใช้
Show-SystemInfo
```

### Exercise 6: Object Exploration

```powershell
# ฝึก explore objects

# 1. DateTime object
$now = Get-Date
$now | Get-Member -MemberType Property
# ลอง:
$now.Year
$now.Month
$now.DayOfWeek
$now.ToString('yyyy-MM-dd')
$now.AddDays(30)
$now.AddMonths(-1)

# 2. File object
$file = Get-Item $PROFILE  # หรือไฟล์ใดก็ได้
$file | Get-Member -MemberType Property
# ลอง:
$file.Name
$file.Length
$file.LastWriteTime
$file.Extension
$file.DirectoryName

# 3. Process object
$ps = Get-Process -Name 'powershell' | Select-Object -First 1
$ps | Get-Member
# ลอง:
$ps.Name
$ps.Id
$ps.StartTime
$ps.HasExited
```

---

## 📝 สรุป Part 01

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| PowerShell คืออะไร | Shell + Scripting language + .NET objects |
| ประวัติ | v1.0 (2006) → Core 6.0 → PowerShell 7.x |
| vs CMD/Bash | Object-based, cross-platform, .NET integration |
| Cmdlet | Verb-Noun convention |
| Help System | Get-Help, Get-Command, Get-Member |
| Pipeline | ส่ง objects ระหว่าง cmdlets |
| Object | ทุกอย่างคือ .NET objects |

---

## 🔗 อ่านเพิ่มเติม

- [PowerShell Documentation](https://docs.microsoft.com/en-us/powershell/)
- [PowerShell GitHub](https://github.com/PowerShell/PowerShell)
- `Get-Help about_Pipeline`
- `Get-Help about_Objects`
- `Get-Help about_Comparison_Operators`

---

**ต่อไป → [Part 02: Installation & Environment Setup](Part-02.md)**
