# Part 89: Module Publishing และ PSGallery

> **ระดับ**: 🟠 World-Class | **เวลา**: ~4 ชั่วโมง

---

## 1. Module Structure มาตรฐาน

```
MyModule/
├── MyModule.psd1           # Manifest
├── MyModule.psm1           # Root module
├── Public/                 # Exported functions
│   ├── Get-Something.ps1
│   └── Invoke-Action.ps1
├── Private/                # Internal helpers
│   └── ConvertTo-Internal.ps1
├── Classes/                # PS classes
│   └── MyClass.ps1
├── en-US/                  # Help files
│   └── MyModule-help.xml
├── Tests/                  # Pester tests
│   └── MyModule.Tests.ps1
└── CHANGELOG.md
```

```powershell
# Build script: build.ps1
[CmdletBinding()]
param(
    [string]$Version,
    [switch]$Publish,
    [string]$Repository = 'PSGallery'
)

$ModuleName = 'MyModule'
$BuildDir   = './dist'
$SrcDir     = './src'

# 1. Run tests
Write-Host '=== Running Tests ===' -ForegroundColor Cyan
$testResult = Invoke-Pester -Path "$SrcDir/Tests" -PassThru -OutputFormat NUnitXml -OutputFile 'test-results.xml'
if ($testResult.FailedCount -gt 0) { throw "$($testResult.FailedCount) tests failed" }
Write-Host "All $($testResult.PassedCount) tests passed" -ForegroundColor Green

# 2. Run SAST
Write-Host '=== Running Script Analyzer ===' -ForegroundColor Cyan
$issues = Invoke-ScriptAnalyzer -Path $SrcDir -Severity Error -Recurse
if ($issues) {
    $issues | Format-Table
    throw "Script Analyzer found $($issues.Count) errors"
}

# 3. Build
Write-Host '=== Building ===' -ForegroundColor Cyan
if (Test-Path $BuildDir) { Remove-Item $BuildDir -Recurse -Force }
New-Item -ItemType Directory $BuildDir | Out-Null
Copy-Item -Path "$SrcDir/*" -Destination "$BuildDir/$ModuleName" -Recurse -Force

# Update version
if ($Version) {
    $psd1 = "$BuildDir/$ModuleName/$ModuleName.psd1"
    Update-ModuleManifest -Path $psd1 -ModuleVersion $Version
    Write-Host "Version set to $Version"
}

# 4. Test module loads
Import-Module "$BuildDir/$ModuleName/$ModuleName.psd1" -Force
$cmdCount = (Get-Command -Module $ModuleName).Count
Write-Host "Module exports $cmdCount commands"
Remove-Module $ModuleName -Force

# 5. Publish
if ($Publish) {
    Write-Host '=== Publishing ===' -ForegroundColor Cyan
    Publish-Module -Path "$BuildDir/$ModuleName" `
        -NuGetApiKey $env:PSGALLERY_KEY `
        -Repository  $Repository
    Write-Host "Published $ModuleName v$Version to $Repository" -ForegroundColor Green
}
```

---

## 2. Private Repository (Nexus / GitHub Packages)

```powershell
# Register private NuGet repository
function Register-PrivateRepo {
    param(
        [string]$Name,
        [string]$SourceUrl,
        [string]$PublishUrl,
        [PSCredential]$Credential
    )
    
    $repoParams = @{
        Name               = $Name
        SourceLocation     = $SourceUrl
        PublishLocation    = $PublishUrl
        InstallationPolicy = 'Trusted'
    }
    
    if ($Credential) {
        $repoParams['Credential'] = $Credential
    }
    
    Register-PSRepository @repoParams
    Write-Host "Registered repository: $Name" -ForegroundColor Green
}

# Register GitHub Packages
$cred = [PSCredential]::new(
    $env:GITHUB_ACTOR,
    (ConvertTo-SecureString $env:GITHUB_TOKEN -AsPlainText -Force)
)

Register-PrivateRepo `
    -Name       'GitHubPackages' `
    -SourceUrl  'https://nuget.pkg.github.com/myorg/index.json' `
    -PublishUrl 'https://nuget.pkg.github.com/myorg' `
    -Credential $cred

# Publish to private repo
Publish-Module -Path './dist/MyModule' -Repository 'GitHubPackages' -Credential $cred

# Install from private repo
Install-Module 'MyModule' -Repository 'GitHubPackages' -Credential $cred -Force
```

---

## 3. Automatic Help Generation

```powershell
# Generate MAML help from comment-based help
function Export-ModuleHelp {
    param(
        [string]$ModulePath,
        [string]$OutputDir = './en-US'
    )
    
    # Requires PlatyPS module
    Install-Module PlatyPS -Force -Scope CurrentUser
    Import-Module PlatyPS
    
    # Import the module
    Import-Module $ModulePath -Force
    $moduleName = (Get-Item $ModulePath).BaseName
    
    # Generate markdown docs
    New-MarkdownHelp `
        -Module      $moduleName `
        -OutputFolder './docs' `
        -Force
    
    # Build MAML help file
    New-ExternalHelp `
        -Path          './docs' `
        -OutputPath    $OutputDir `
        -Force
    
    Write-Host "Help generated in $OutputDir" -ForegroundColor Green
}

# Example function with complete comment-based help
function Invoke-DataProcessing {
    <#
    .SYNOPSIS
        Processes data with optional transformation pipeline.
    
    .DESCRIPTION
        A long description of the function's purpose, usage, and behavior.
        Supports pipeline input and produces structured output.
    
    .PARAMETER InputObject
        The data object to process. Accepts pipeline input.
    
    .PARAMETER TransformScript
        Optional scriptblock to transform each item.
    
    .PARAMETER PassThru
        Returns the processed object rather than printing.
    
    .EXAMPLE
        1..10 | Invoke-DataProcessing
        
        Process integers 1-10 with default settings.
    
    .EXAMPLE
        Get-Item *.csv | Invoke-DataProcessing -TransformScript { $_.Name.ToUpper() }
        
        Process file items, transforming names to uppercase.
    
    .OUTPUTS
        PSCustomObject with properties: Input, Output, ProcessedAt.
    
    .LINK
        https://github.com/company/MyModule
    #>
    [CmdletBinding()]
    [OutputType([PSCustomObject])]
    param(
        [Parameter(Mandatory, ValueFromPipeline)]
        [object]$InputObject,
        
        [scriptblock]$TransformScript = { $_ },
        
        [switch]$PassThru
    )
    
    process {
        $output = & $TransformScript $InputObject
        $result = [PSCustomObject]@{
            Input       = $InputObject
            Output      = $output
            ProcessedAt = [datetime]::UtcNow
        }
        if ($PassThru) { return $result }
        else { Write-Output $result }
    }
}
```

---

**ก่อนหน้า ← [Part 88](Part-88.md) | ต่อไป → [Part 90: Real-World Case Study — IT Automation](Part-90.md)**
