# Part 26: Pester Testing Framework

> **ระดับ**: 🟠 Advanced | **เวลา**: ~4 ชั่วขนึ่ง

---

## 1. Pester พื้นฐาน

```powershell
# ติดตั้ง
Install-Module Pester -Force -SkipPublisherCheck
Get-Module Pester -ListAvailable

# สร้าง test file
# Calculator.Tests.ps1

BeforeAll {
    . "$PSScriptRoot/Calculator.ps1"  # dot-source the file to test
}

Describe 'Calculator' {
    
    Context 'Add-Numbers' {
        It 'adds two positive numbers' {
            Add-Numbers 2 3 | Should -Be 5
        }
        
        It 'adds negative numbers' {
            Add-Numbers -3 -4 | Should -Be -7
        }
        
        It 'adds zero' {
            Add-Numbers 5 0 | Should -Be 5
        }
    }
    
    Context 'Divide-Numbers' {
        It 'divides correctly' {
            Divide-Numbers 10 2 | Should -Be 5
        }
        
        It 'throws on divide by zero' {
            { Divide-Numbers 10 0 } | Should -Throw
        }
        
        It 'returns float' {
            Divide-Numbers 7 2 | Should -BeGreaterThan 3
        }
    }
}
```

---

## 2. Assertions (เงื่อนไข)

```powershell
# Should assertions
$value = 42

# Equality
$value | Should -Be 42
$value | Should -Not -Be 0
$value | Should -BeExactly 42      # type-strict

# Comparison
$value | Should -BeGreaterThan 10
$value | Should -BeLessThan 100
$value | Should -BeGreaterOrEqual 42
$value | Should -BeLessOrEqual 42

# Type
$value | Should -BeOfType [int]
$value | Should -BeOfType 'System.Int32'

# Null/Empty
$null  | Should -BeNullOrEmpty
''     | Should -BeNullOrEmpty
@()    | Should -BeNullOrEmpty
$value | Should -Not -BeNullOrEmpty

# String
'Hello World' | Should -Match 'World'
'Hello World' | Should -Not -Match 'Foo'
'Hello World' | Should -BeLike 'Hello*'
'Hello World' | Should -Contain 'Hello'  # for arrays

# Array
@(1,2,3) | Should -Contain 2
@(1,2,3) | Should -HaveCount 3
@(1,2,3) | Should -Not -Contain 5

# Throw
{ throw 'error' } | Should -Throw
{ throw 'specific error' } | Should -Throw -ExpectedMessage 'specific'
{ [int]'abc' } | Should -Throw -ExceptionType ([System.Management.Automation.RuntimeException])

# File
'C:\Windows' | Should -Exist
'C:\Windows\notepad.exe' | Should -Exist
```

---

## 3. Mocking

```powershell
# Mock: แทนที่ฟังก์ชันจริง
Describe 'Send-Email' {
    
    BeforeAll {
        function Send-Email {
            param($To, $Subject, $Body)
            # จริงๆ ส่ง email
            $smtp = [System.Net.Mail.SmtpClient]::new('smtp.example.com')
            # ...
        }
        
        function Notify-User {
            param($UserId, $Message)
            $user = Get-UserById $UserId
            Send-Email -To $user.Email -Subject 'Notification' -Body $Message
            return $true
        }
    }
    
    It 'sends email to user' {
        Mock Get-UserById { return @{Email='test@example.com'; Name='Test'} }
        Mock Send-Email { }
        
        Notify-User -UserId 123 -Message 'Hello!'
        
        Should -Invoke Send-Email -Times 1 -Exactly
        Should -Invoke Send-Email -ParameterFilter {
            $To -eq 'test@example.com'
        }
    }
    
    It 'does not send when user not found' {
        Mock Get-UserById { return $null }
        Mock Send-Email { }
        
        $result = Notify-User -UserId 999 -Message 'Hello'
        
        Should -Invoke Send-Email -Times 0
    }
}

# Mock module functions
Mock -ModuleName 'MyModule' -CommandName 'Get-Data' -MockWith {
    return @{ id=1; value='mocked' }
}
```

---

## 4. BeforeAll/BeforeEach/AfterEach/AfterAll

```powershell
Describe 'File Operations' {
    
    BeforeAll {
        $testDir = Join-Path $TestDrive 'TestFiles'
        New-Item $testDir -ItemType Directory -Force
    }
    
    BeforeEach {
        # ทำก่อนทุกเทสต์
        $testFile = Join-Path $testDir 'test.txt'
        'Initial content' | Set-Content $testFile
    }
    
    AfterEach {
        # ทำเดิมหลังเทสต์
        Remove-Item $testFile -Force -ErrorAction SilentlyContinue
    }
    
    AfterAll {
        Remove-Item $testDir -Recurse -Force -ErrorAction SilentlyContinue
    }
    
    It 'reads file content' {
        $content = Get-Content $testFile
        $content | Should -Be 'Initial content'
    }
    
    It 'modifies file content' {
        'New content' | Set-Content $testFile
        Get-Content $testFile | Should -Be 'New content'
    }
    
    It 'appends to file' {
        Add-Content $testFile 'Appended'
        $lines = Get-Content $testFile
        $lines | Should -HaveCount 2
        $lines[-1] | Should -Be 'Appended'
    }
}
```

---

## 5. Test Configuration และ CI

```powershell
# Pester configuration
$config = New-PesterConfiguration
$config.Run.Path = '.\Tests'
$config.Run.Recurse = $true
$config.Output.Verbosity = 'Detailed'
$config.TestResult.Enabled = $true
$config.TestResult.OutputPath = 'test-results.xml'
$config.TestResult.OutputFormat = 'JUnitXml'
$config.Coverage.Enabled = $true
$config.Coverage.Path = '.\src'
$config.Coverage.OutputFormat = 'JaCoCo'
$config.Coverage.OutputPath = 'coverage.xml'

Invoke-Pester -Configuration $config

# เรียกใน CI pipeline
# GitHub Actions workflow:
# - name: Run Pester Tests
#   shell: pwsh
#   run: |
#     Install-Module Pester -Force
#     Invoke-Pester -Configuration @{
#       Run = @{ Path = './Tests' }
#       TestResult = @{ Enabled = $true; OutputPath = 'results.xml' }
#     }

# Code coverage check
$result = Invoke-Pester -Configuration $config -PassThru
if ($result.CodeCoverage.CoveragePercent -lt 80) {
    Write-Error "Coverage $($result.CodeCoverage.CoveragePercent)% < 80% threshold"
    exit 1
}
```

---

## 6. Integration Test Pattern

```powershell
Describe 'API Integration' -Tag 'Integration' {
    
    BeforeAll {
        $baseUrl = 'http://localhost:8080'
        # Start server in background
        $server = Start-Job { & 'C:\app\server.ps1' }
        Start-Sleep 2  # wait for startup
    }
    
    AfterAll {
        $server | Stop-Job
        $server | Remove-Job
    }
    
    It 'health check returns 200' {
        $response = Invoke-WebRequest "$baseUrl/health" -ErrorAction Stop
        $response.StatusCode | Should -Be 200
    }
    
    It 'GET /todos returns array' {
        $data = Invoke-RestMethod "$baseUrl/todos"
        $data | Should -BeOfType [array]
    }
    
    It 'POST /todos creates item' {
        $body = @{title='Test Todo'} | ConvertTo-Json
        $item = Invoke-RestMethod "$baseUrl/todos" -Method POST \
            -Body $body -ContentType 'application/json'
        $item.id | Should -Not -BeNullOrEmpty
        $item.title | Should -Be 'Test Todo'
    }
}

# Run only unit tests (skip integration)
Invoke-Pester -Tag 'Unit' -ExcludeTag 'Integration'
```

---

**ก่อนหน้า ← [Part 25](Part-25.md) | ต่อไป → [Part 27: DSC](Part-27.md)**
