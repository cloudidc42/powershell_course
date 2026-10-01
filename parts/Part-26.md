# Part 26: Pester Testing Framework

> **ระดับ**: 🟠 Advanced | **เวลา**: ~4 ชั่วโมง

---

## 1. Pester Basics

```powershell
# ติดตั้ง
Install-Module Pester -Force -SkipPublisherCheck

# Describe/It/Should structure
Describe 'Get-Greeting' {
    It 'returns greeting with name' {
        $result = Get-Greeting 'Alice'
        $result | Should -Be 'Hello, Alice!'
    }
    
    It 'returns default greeting' {
        $result = Get-Greeting
        $result | Should -Be 'Hello, World!'
    }
    
    It 'should not return null' {
        Get-Greeting 'Bob' | Should -Not -BeNullOrEmpty
    }
}

# Context for grouping
Describe 'Calculator' {
    Context 'Addition' {
        It 'adds positive numbers' {
            Add-Numbers 2 3 | Should -Be 5
        }
        It 'adds negative numbers' {
            Add-Numbers -2 -3 | Should -Be -5
        }
    }
    
    Context 'Division' {
        It 'divides correctly' {
            Divide-Numbers 10 2 | Should -Be 5
        }
        It 'throws on divide by zero' {
            { Divide-Numbers 10 0 } | Should -Throw
        }
    }
}
```

---

## 2. Should Matchers

```powershell
# Equality
5 | Should -Be 5
5 | Should -Not -Be 6

# Types
5       | Should -BeOfType [int]
'hello' | Should -BeOfType [string]

# Null/Empty
$null   | Should -BeNullOrEmpty
''      | Should -BeNullOrEmpty
'text'  | Should -Not -BeNullOrEmpty

# Collections
@(1,2,3) | Should -HaveCount 3
@(1,2,3) | Should -Contain 2
@(1,2,3) | Should -Not -Contain 5

# Strings
'Hello World' | Should -Match 'World'
'Hello World' | Should -BeLike '*World'
'Hello World' | Should -Not -BeLike '*xyz'
'ABC'         | Should -BeExactly 'ABC'

# Numbers
5  | Should -BeGreaterThan 3
5  | Should -BeLessThan 10
5  | Should -BeGreaterOrEqual 5
5  | Should -BeLessOrEqual 5
5.1 | Should -BeApproximately 5.0 -Because 'within tolerance' -ExpectedDifference 0.2

# Existence
Test-Path 'C:\Windows' | Should -BeTrue
{ Get-Item 'missing' -ErrorAction Stop } | Should -Throw -ExceptionType [System.Management.Automation.ItemNotFoundException]
```

---

## 3. BeforeAll, BeforeEach, AfterAll, AfterEach

```powershell
Describe 'UserService' {
    BeforeAll {
        # ตั้งค่าสำหรับทั้ง Describe
        $script:db = [TestDatabase]::new()
        $script:service = [UserService]::new($script:db)
    }
    
    AfterAll {
        $script:db.Dispose()
    }
    
    BeforeEach {
        # reset ก่อนทุก test
        $script:db.Clear()
    }
    
    Context 'CreateUser' {
        It 'creates user successfully' {
            $user = $script:service.Create(@{ name='Alice'; email='alice@test.com' })
            $user | Should -Not -BeNullOrEmpty
            $user.id | Should -Not -BeNullOrEmpty
        }
        
        It 'fails with duplicate email' {
            $script:service.Create(@{ name='Alice'; email='same@test.com' })
            { $script:service.Create(@{ name='Bob'; email='same@test.com' }) } | Should -Throw
        }
    }
}
```

---

## 4. Mocking

```powershell
Describe 'Send-Report' {
    BeforeAll {
        Mock Send-MailMessage {}
        Mock Get-Date { return [datetime]'2024-01-15' }
    }
    
    It 'sends email report' {
        Send-Report -To 'admin@example.com'
        
        Should -Invoke Send-MailMessage -Times 1 -Exactly
        Should -Invoke Send-MailMessage -ParameterFilter {
            $To -eq 'admin@example.com'
        }
    }
    
    It 'includes date in subject' {
        Send-Report -To 'admin@example.com'
        
        Should -Invoke Send-MailMessage -ParameterFilter {
            $Subject -like '*2024-01-15*'
        }
    }
}

# Mock with return value
Describe 'Get-Weather' {
    BeforeAll {
        Mock Invoke-RestMethod {
            return @{ temperature = 25; condition = 'Sunny' }
        } -ParameterFilter { $Uri -like '*weather*' }
    }
    
    It 'returns temperature' {
        $weather = Get-Weather 'Bangkok'
        $weather.temperature | Should -Be 25
    }
}
```

---

## 5. เรียก Tests

```powershell
# Run tests
Invoke-Pester '.\tests\'                    # ทุกไฟล์
Invoke-Pester '.\tests\User.Tests.ps1'        # ไฟล์เดียว
Invoke-Pester '.\' -Recurse                  # สแกน recursive

# Configuration
$config = New-PesterConfiguration
$config.Run.Path        = '.\tests'
$config.Output.Verbosity = 'Detailed'
$config.CodeCoverage.Enabled = $true
$config.CodeCoverage.Path    = @('.\src\*.ps1')
$config.TestResult.Enabled   = $true
$config.TestResult.OutputPath = 'TestResults.xml'
$config.TestResult.OutputFormat = 'NUnitXml'

$result = Invoke-Pester -Configuration $config
Write-Host "Passed: $($result.PassedCount) / Total: $($result.TotalCount)"
Write-Host "Coverage: $($result.CodeCoverage.CoveragePercent)%"

# CI pipeline
if ($result.FailedCount -gt 0) {
    Write-Error "$($result.FailedCount) test(s) failed!"
    exit 1
}
```

---

**ก่อนหน้า ← [Part 25](Part-25.md) | ต่อไป → [Part 27: DSC](Part-27.md)**
