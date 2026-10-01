# Part 34: CI/CD และ DevOps Automation

> **ระดับ**: 🟠 Advanced | **เวลา**: ~4 ชั่วขนึ่ง

---

## 1. GitHub Actions + PowerShell

```yaml
# .github/workflows/powershell-ci.yml
name: PowerShell CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: windows-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Dependencies
        shell: pwsh
        run: |
          Set-PSRepository PSGallery -InstallationPolicy Trusted
          Install-Module Pester -Force -SkipPublisherCheck
          Install-Module PSScriptAnalyzer -Force
      
      - name: Lint with PSScriptAnalyzer
        shell: pwsh
        run: |
          $results = Invoke-ScriptAnalyzer -Path .\src -Recurse -Severity Error,Warning
          $results | Format-Table RuleName, Severity, Message, ScriptName, Line
          if ($results | Where-Object Severity -eq 'Error') {
            Write-Error 'PSScriptAnalyzer found errors'
            exit 1
          }
      
      - name: Run Pester Tests
        shell: pwsh
        run: |
          $cfg = New-PesterConfiguration
          $cfg.Run.Path      = '.\tests'
          $cfg.TestResult.Enabled    = $true
          $cfg.TestResult.OutputPath = 'test-results.xml'
          $cfg.Coverage.Enabled      = $true
          $cfg.Coverage.Path         = '.\src'
          $cfg.Coverage.OutputPath   = 'coverage.xml'
          
          $res = Invoke-Pester -Configuration $cfg -PassThru
          if ($res.FailedCount -gt 0) {
            Write-Error "$($res.FailedCount) tests failed"
            exit 1
          }
      
      - name: Publish Test Results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          files: test-results.xml
      
      - name: Upload Coverage
        uses: codecov/codecov-action@v3
        with:
          files: coverage.xml
  
  deploy:
    needs: test
    runs-on: windows-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy
        shell: pwsh
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
          SERVER:       ${{ vars.PROD_SERVER }}
        run: |
          .\scripts\deploy.ps1 -Server $env:SERVER -Token $env:DEPLOY_TOKEN
```

---

## 2. Azure DevOps Pipeline

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
      - release/*

pool:
  vmImage: 'windows-latest'

variables:
  buildNumber: $(Build.BuildNumber)
  artifactName: MyApp_$(buildNumber)

stages:

- stage: Build
  jobs:
  - job: BuildAndTest
    steps:
    - task: PowerShell@2
      displayName: 'Install Modules'
      inputs:
        targetType: inline
        script: |
          Install-Module Pester -Force -SkipPublisherCheck
    
    - task: PowerShell@2
      displayName: 'Run Tests'
      inputs:
        targetType: filePath
        filePath: scripts/run-tests.ps1
      env:
        TEST_ENV: 'ci'
    
    - task: PublishTestResults@2
      inputs:
        testResultsFormat: JUnit
        testResultsFiles: '**/test-results.xml'
    
    - task: PublishPipelineArtifact@1
      inputs:
        targetPath: dist
        artifact: $(artifactName)

- stage: Deploy
  dependsOn: Build
  condition: and(succeeded(), eq(variables['Build.SourceBranchName'], 'main'))
  jobs:
  - deployment: DeployProd
    environment: 'production'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: DownloadPipelineArtifact@2
            inputs:
              artifact: $(artifactName)
          - task: PowerShell@2
            displayName: Deploy
            inputs:
              filePath: scripts/deploy.ps1
```

---

## 3. Deployment Script

```powershell
# scripts/deploy.ps1
[CmdletBinding()]
param(
    [Parameter(Mandatory)]
    [string]$Server,
    
    [Parameter(Mandatory)]
    [string]$Token,
    
    [string]$AppPath   = 'C:\inetpub\apps\myapp',
    [string]$ArtifactPath = '.\dist'
)

$ErrorActionPreference = 'Stop'

function Write-Step([string]$msg) {
    Write-Host "\n>> $msg" -ForegroundColor Cyan
}

try {
    Write-Step "Connecting to $Server"
    $cred    = [System.Net.NetworkCredential]::new('deploy', $Token)
    $secpass = ConvertTo-SecureString $Token -AsPlainText -Force
    $pscred  = [PSCredential]::new('deploy', $secpass)
    $session = New-PSSession -ComputerName $Server -Credential $pscred
    
    Write-Step "Stopping app pool"
    Invoke-Command $session { Stop-WebAppPool 'MyApp' -ErrorAction SilentlyContinue }
    
    Write-Step "Copying files"
    Copy-Item $ArtifactPath -Destination $AppPath -ToSession $session -Recurse -Force
    
    Write-Step "Running migrations"
    Invoke-Command $session { & "$using:AppPath\migrate.ps1" }
    
    Write-Step "Starting app pool"
    Invoke-Command $session { Start-WebAppPool 'MyApp' }
    
    Write-Step "Health check"
    Start-Sleep 5
    $resp = Invoke-RestMethod "https://$Server/health"
    if ($resp.status -ne 'ok') { throw "Health check failed: $($resp.status)" }
    
    Write-Host "\nDeploy SUCCESS" -ForegroundColor Green
    Remove-PSSession $session
} catch {
    Write-Error "Deploy FAILED: $_"
    exit 1
}
```

---

## 4. GitOps Helper

```powershell
# Semantic versioning helper
function Get-NextVersion {
    param([string]$CurrentVersion, [string]$BumpType = 'patch')
    
    $parts = $CurrentVersion.TrimStart('v') -split '\.'
    $major, $minor, $patch = [int]$parts[0], [int]$parts[1], [int]$parts[2]
    
    switch ($BumpType) {
        'major' { $major++; $minor = 0; $patch = 0 }
        'minor' { $minor++; $patch = 0 }
        'patch' { $patch++ }
    }
    "v$major.$minor.$patch"
}

# Create release tag
function New-Release {
    param([string]$Version, [string]$Message = "Release $Version")
    git tag -a $Version -m $Message
    git push origin $Version
    Write-Host "Tagged release $Version"
}

$next = Get-NextVersion '1.2.3' -BumpType minor
# Returns: 1.3.0
```

---

**ก่อนหน้า ← [Part 33](Part-33.md) | ต่อไป → [Part 35: Azure Cloud](Part-35.md)**
