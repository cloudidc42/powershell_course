# Part 52: PowerShell Internals และ Runtime

> **ระดับ**: 🟡 Professional | **เวลา**: ~5 ชั่วขนึ่ง

---

## 1. Runspace และ Pipeline

```powershell
# Runspace = isolated PowerShell execution environment

# สร้าง runspace ด้วยตัวเอง
$rs = [RunspaceFactory]::CreateRunspace()
$rs.Open()

$ps = [PowerShell]::Create()
$ps.Runspace = $rs
[void]$ps.AddScript('$x = 42; $x * 2')
$result = $ps.Invoke()
Write-Host "Result: $result"  # 84

$ps.Dispose()
$rs.Close()

# Runspace pool สำหรับ parallel work
$pool = [RunspaceFactory]::CreateRunspacePool(1, 10)
$pool.Open()

$jobs = 1..20 | ForEach-Object {
    $num = $_
    $ps  = [PowerShell]::Create()
    $ps.RunspacePool = $pool
    [void]$ps.AddScript({ param($n) [math]::Sqrt($n) }).AddParameter('n', $num)
    @{ PS = $ps; Handle = $ps.BeginInvoke(); Input = $num }
}

$results = $jobs | ForEach-Object {
    [PSCustomObject]@{
        Input  = $_.Input
        Result = [math]::Round($_.PS.EndInvoke($_.Handle)[0], 4)
    }
    $_.PS.Dispose()
}

$pool.Close()
$results | Format-Table
```

---

## 2. CommandInfo และ AST

```powershell
# CommandInfo
Get-Command Get-Process | Select-Object Name, CommandType, Module, Definition

# รายละเอียด cmdlet
$cmd = Get-Command Invoke-WebRequest
$cmd | Get-Member
$cmd.Parameters.Keys | Sort-Object
$cmd.ParameterSets | Select-Object Name, Parameters | ForEach-Object {
    Write-Host "-- $($_.Name) --"
    $_.Parameters | Select-Object Name, IsMandatory | Format-Table -AutoSize
}

# Abstract Syntax Tree (AST)
$code = '1..10 | Where-Object { $_ % 2 -eq 0 } | ForEach-Object { $_ * 2 }'
$tokens = $null; $errors = $null
$ast = [System.Management.Automation.Language.Parser]::ParseInput($code, [ref]$tokens, [ref]$errors)

# แสดงโครงสร้าง AST
$ast.FindAll({$args[0] -is [System.Management.Automation.Language.CommandAst]}, $true) |
    ForEach-Object { Write-Host "Command: $($_.CommandElements[0])" }

# Find variables in script
$script = Get-Content 'script.ps1' -Raw
$ast2, $tokens2, $errors2 = {}
$ast2 = [System.Management.Automation.Language.Parser]::ParseInput($script, [ref]$tokens2, [ref]$errors2)
$vars = $ast2.FindAll({$args[0] -is [System.Management.Automation.Language.VariableExpressionAst]}, $true)
$vars | Group-Object { $_.VariablePath.UserPath } | Sort-Object Count -Descending
```

---

## 3. Type Accelerators

```powershell
# Type accelerators = shortcuts สำหรับ .NET types
[string]      # = [System.String]
[int]         # = [System.Int32]
[bool]        # = [System.Boolean]
[datetime]    # = [System.DateTime]
[regex]       # = [System.Text.RegularExpressions.Regex]
[xml]         # = [System.Xml.XmlDocument]
[pscustomobject] # = [System.Management.Automation.PSObject]
[scriptblock] # = [System.Management.Automation.ScriptBlock]
[hashtable]   # = [System.Collections.Hashtable]

# ดู type accelerators ทั้งหมด
[psobject].Assembly.GetType('System.Management.Automation.TypeAccelerators')::Get |
    Sort-Object Key | Format-Table Key, Value

# เพิ่ม custom type accelerator
$ta = [psobject].Assembly.GetType('System.Management.Automation.TypeAccelerators')
$ta::Add('mylist', [System.Collections.Generic.List[object]])
$l = [mylist]::new()
$l.Add(1); $l.Add(2)

# ลบ
$ta::Remove('mylist')
```

---

## 4. Session State และ Scope

```powershell
# Session state
$host.Runspace.SessionStateProxy.GetVariable('PSVersionTable')
$host.Runspace.SessionStateProxy.PSVariable.GetValue('executionContext')

# Scope isolation
function Outer {
    $x = 'outer'
    function Inner { "x = $x" }  # lexical scope: sees outer's x
    Inner
}
Outer  # x = outer

# Module scope
$m = New-Module {
    $private = 'secret'
    function Get-Public { 'public data' }
    Export-ModuleMember -Function Get-Public
}

Import-Module $m
Get-Public          # OK
$m.NewBoundScriptBlock({$private}).Invoke()  # เข้าถึง module scope

# GlobalScope
$global:sharedVar = 'hello'
function ReadGlobal { $global:sharedVar }
ReadGlobal

# Thread-safe session
[System.Threading.ThreadLocal[string]]$tls = [System.Threading.ThreadLocal[string]]::new({ 'default' })
$tls.Value = 'thread-1'
Write-Host $tls.Value
```

---

**ก่อนหน้า ← [Part 51](Part-51.md) | ต่อไป → [Part 53: SecureString & Secrets Management](Part-53.md)**
