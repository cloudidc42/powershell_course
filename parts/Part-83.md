# Part 83: Advanced Module Development

> **ระดับ**: 🟠 World-Class | **เวลา**: ~5 ชั่วโมง

---

## 1. Module Manifest Best Practices

```powershell
# Create production-grade module manifest
New-ModuleManifest `
    -Path           './MyModule/MyModule.psd1' `
    -RootModule     'MyModule.psm1' `
    -ModuleVersion  '2.1.0' `
    -Author         'Your Name' `
    -CompanyName    'Your Company' `
    -Copyright      "Copyright (c) $(Get-Date -Format yyyy) Your Company" `
    -Description    'Enterprise PowerShell automation module' `
    -PowerShellVersion '5.1' `
    -RequiredModules @('PSReadLine') `
    -FunctionsToExport @(
        'Invoke-DeployPipeline',
        'Get-ServiceHealth',
        'Send-Notification'
    ) `
    -CmdletsToExport @() `
    -VariablesToExport @() `
    -AliasesToExport @() `
    -Tags @('DevOps','Automation','Enterprise') `
    -ProjectUri     'https://github.com/company/MyModule' `
    -LicenseUri     'https://github.com/company/MyModule/blob/main/LICENSE' `
    -ReleaseNotes   'v2.1.0: Added parallel deployment support'

# Root module (MyModule.psm1)
$Public  = @(Get-ChildItem -Path "$PSScriptRoot\Public" -Filter '*.ps1' -Recurse)
$Private = @(Get-ChildItem -Path "$PSScriptRoot\Private" -Filter '*.ps1' -Recurse)
$Classes = @(Get-ChildItem -Path "$PSScriptRoot\Classes" -Filter '*.ps1' -Recurse)

# Load classes first (they may be needed by functions)
foreach ($file in $Classes) {
    . $file.FullName
}

# Load private helpers
foreach ($file in $Private) {
    . $file.FullName
}

# Load and export public functions
foreach ($file in $Public) {
    . $file.FullName
    Export-ModuleMember -Function $file.BaseName
}

# Module-level variables
$script:Config = $null
$script:Logger = $null

# Initialize
function Initialize-Module {
    param([hashtable]$Options = @{})
    $script:Config = [ConfigManager]::new('./Config', $Options.Environment ?? 'production')
    $script:Logger = [CorrelatedLogger]::new('MyModule')
}
```

---

## 2. Argument Completers และ ValidateScript

```powershell
# Dynamic argument completer
$envCompleter = {
    param($commandName, $paramName, $wordToComplete, $commandAst, $fakeBoundParameters)
    @('dev','staging','production','dr') | Where-Object { $_ -like "$wordToComplete*" } |
        ForEach-Object { [System.Management.Automation.CompletionResult]::new($_, $_, 'ParameterValue', $_) }
}
Register-ArgumentCompleter -CommandName 'Deploy-App' -ParameterName 'Environment' -ScriptBlock $envCompleter

# Services completer (dynamic from API)
$serviceCompleter = {
    param($commandName, $paramName, $wordToComplete, $commandAst, $fakeBoundParameters)
    try {
        $services = Invoke-RestMethod 'https://api.example.com/services'
        $services.name | Where-Object { $_ -like "$wordToComplete*" } |
            ForEach-Object { [System.Management.Automation.CompletionResult]::new($_, $_, 'ParameterValue', $_) }
    } catch { @() }
}
Register-ArgumentCompleter -CommandName 'Get-ServiceHealth' -ParameterName 'ServiceName' -ScriptBlock $serviceCompleter

# Custom ValidateScript with helpful errors
function Invoke-DeployPipeline {
    [CmdletBinding(SupportsShouldProcess)]
    param(
        [Parameter(Mandatory)]
        [ValidateScript({
            if ($_ -match '^\d+\.\d+\.\d+$') { return $true }
            throw "Version must be semver format (e.g., 1.2.3). Got: $_"
        })]
        [string]$Version,
        
        [ValidateSet('blue-green','rolling','canary')]
        [string]$Strategy = 'rolling',
        
        [ValidateRange(1, 100)]
        [int]$CanaryPercent = 10
    )
    
    if ($PSCmdlet.ShouldProcess("Deploy v$Version using $Strategy strategy")) {
        # Execute deployment
        Write-Host "Deploying v$Version via $Strategy" -ForegroundColor Green
    }
}
```

---

## 3. Module Versioning และ Publishing

```powershell
# Semantic versioning automation
function Get-NextVersion {
    param(
        [string]$CurrentVersion,
        [ValidateSet('Major','Minor','Patch')]
        [string]$BumpType = 'Patch'
    )
    
    $parts   = $CurrentVersion -split '\.'
    $major   = [int]$parts[0]
    $minor   = [int]$parts[1]
    $patch   = [int]$parts[2]
    
    switch ($BumpType) {
        'Major' { $major++; $minor=0; $patch=0 }
        'Minor' { $minor++;           $patch=0 }
        'Patch' { $patch++ }
    }
    
    return "$major.$minor.$patch"
}

function Publish-ModuleToGallery {
    param(
        [string]$ModulePath,
        [string]$ApiKey     = $env:PSGALLERY_KEY,
        [string]$Repository = 'PSGallery',
        [switch]$WhatIf
    )
    
    # Run tests first
    $testResult = Invoke-Pester -Path "$ModulePath\Tests" -PassThru
    if ($testResult.FailedCount -gt 0) {
        throw "Tests failed: $($testResult.FailedCount) failures"
    }
    
    # Script analyzer
    $issues = Invoke-ScriptAnalyzer -Path $ModulePath -Severity 'Error'
    if ($issues) { throw "Script Analyzer found $($issues.Count) errors" }
    
    if ($WhatIf) {
        Write-Host "[WhatIf] Would publish $ModulePath to $Repository" -ForegroundColor Yellow
        return
    }
    
    Publish-Module -Path $ModulePath -NuGetApiKey $ApiKey -Repository $Repository
    Write-Host "Published successfully!" -ForegroundColor Green
}

# Auto-bump and publish
$psd1Path = './MyModule/MyModule.psd1'
$manifest  = Import-PowerShellDataFile $psd1Path
$newVersion= Get-NextVersion $manifest.ModuleVersion -BumpType 'Patch'

Update-ModuleManifest -Path $psd1Path -ModuleVersion $newVersion
Publish-ModuleToGallery -ModulePath './MyModule' -ApiKey $env:PSGALLERY_KEY
```

---

## 4. Module Testing เต็ม Stack

```powershell
# Integration + Unit test combination
Describe 'MyModule' {
    BeforeAll {
        Import-Module './MyModule/MyModule.psd1' -Force
        Initialize-Module -Options @{ Environment='test' }
        
        # Mock external dependencies
        Mock Invoke-RestMethod { @{ status='healthy' } } -ModuleName MyModule
        Mock Send-MailMessage {} -ModuleName MyModule
    }
    
    Context 'Get-ServiceHealth' {
        It 'returns healthy for running service' {
            $result = Get-ServiceHealth -ServiceName 'TestService'
            $result.Status | Should -Be 'Healthy'
        }
        
        It 'returns cached result within TTL' {
            Get-ServiceHealth -ServiceName 'TestService'
            Get-ServiceHealth -ServiceName 'TestService'  # second call
            # Should have called API only once
            Should -Invoke Invoke-RestMethod -Times 1 -ModuleName MyModule
        }
        
        It 'handles API failure gracefully' {
            Mock Invoke-RestMethod { throw 'Connection refused' } -ModuleName MyModule
            { Get-ServiceHealth -ServiceName 'DeadService' } | Should -Not -Throw
            (Get-ServiceHealth -ServiceName 'DeadService').Status | Should -Be 'Unknown'
        }
    }
    
    Context 'Invoke-DeployPipeline' {
        It 'validates semver version format' {
            { Invoke-DeployPipeline -Version 'invalid' -WhatIf } | Should -Throw
            { Invoke-DeployPipeline -Version '1.2.3'   -WhatIf } | Should -Not -Throw
        }
    }
    
    AfterAll { Remove-Module MyModule -Force }
}
```

---

**ก่อนหน้า ← [Part 82](Part-82.md) | ต่อไป → [Part 84: Advanced OOP Patterns](Part-84.md)**
