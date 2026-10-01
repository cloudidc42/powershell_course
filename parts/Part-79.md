# Part 79: DevSecOps Automation

> **ระดับ**: 🟠 World-Class | **เวลา**: ~5 ชั่วโมง

---

## 1. Secrets Scanning Pipeline

```powershell
# Scan code for secrets before commit
function Invoke-SecretsAudit {
    param([string]$Path, [switch]$Recurse)
    
    $patterns = @{
        'AWS Access Key'      = 'AKIA[0-9A-Z]{16}'
        'AWS Secret Key'      = '[0-9a-zA-Z/+]{40}'
        'Azure Connection Str'= 'DefaultEndpointsProtocol=https.*AccountKey='
        'GitHub Token'        = 'gh[pousr]_[A-Za-z0-9_]{36}'
        'JWT Token'           = 'eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+'
        'Private Key'         = '-----BEGIN (RSA|EC|OPENSSH) PRIVATE KEY-----'
        'Password in Code'    = '(?i)password\s*=\s*["''][^"'']{8,}'
        'API Key Variable'    = '(?i)(api_?key|apikey)\s*=\s*["''][^"'']{10,}'
        'DB Connection'       = 'Server=.*;.*Password=\S+'
    }
    
    $exclude = @('.git', 'node_modules', '.venv', '*.min.js', '*.lock')
    
    $files = Get-ChildItem -Path $Path -Recurse:$Recurse -File |
        Where-Object {
            $file = $_
            -not ($exclude | Where-Object { $file.FullName -like "*$_*" -or $file.Name -like $_ })
        } |
        Where-Object { $_.Extension -in @('.ps1','.py','.js','.ts','.json','.yaml','.yml','.env','.sh','.rb','.go','.cs') }
    
    $findings = foreach ($file in $files) {
        $content = Get-Content $file.FullName -Raw -ErrorAction SilentlyContinue
        if (-not $content) { continue }
        
        foreach ($patternName in $patterns.Keys) {
            $matches = [regex]::Matches($content, $patterns[$patternName])
            if ($matches.Count -gt 0) {
                $line = ($content.Substring(0, $matches[0].Index) -split "`n").Count
                [PSCustomObject]@{
                    File     = $file.FullName.Replace($Path, '.')
                    Line     = $line
                    Type     = $patternName
                    Snippet  = $matches[0].Value.Substring(0, [Math]::Min(40, $matches[0].Value.Length)) + '...'
                    Severity = 'HIGH'
                }
            }
        }
    }
    
    if ($findings) {
        Write-Error "SECRETS FOUND: $($findings.Count) potential secret(s) detected!"
        $findings | Format-Table File, Line, Type, Snippet
        return $false
    }
    Write-Host 'Secrets scan: CLEAN' -ForegroundColor Green
    return $true
}

# Run in CI/CD pipeline
$clean = Invoke-SecretsAudit -Path '.' -Recurse
if (-not $clean) { exit 1 }  # Fail pipeline
```

---

## 2. SAST Pipeline Integration

```powershell
# PSScriptAnalyzer as SAST
Install-Module PSScriptAnalyzer -Force

function Invoke-SASTScan {
    param(
        [string]$Path,
        [string]$Severity    = 'Warning',  # Information/Warning/Error
        [string]$ReportPath  = './sast-report.json',
        [switch]$FailOnError
    )
    
    $results = Invoke-ScriptAnalyzer -Path $Path -Recurse -Severity $Severity
    
    if ($results) {
        Write-Warning "SAST found $($results.Count) issue(s)"
        
        $grouped = $results | Group-Object Severity
        foreach ($g in $grouped | Sort-Object { @('Error','Warning','Information').IndexOf($_.Name) }) {
            $color = @{ Error='Red'; Warning='Yellow'; Information='Cyan' }[$g.Name]
            Write-Host "$($g.Name): $($g.Count)" -ForegroundColor $color
        }
        
        # Detailed findings
        $results | Select-Object `
            @{n='File'; e={ Split-Path $_.ScriptPath -Leaf }},
            Line, Severity, RuleName, Message |
            Format-Table -AutoSize
        
        # Export report
        $results | Select-Object ScriptPath, Line, Severity, RuleName, Message |
            ConvertTo-Json | Set-Content $ReportPath
        
        if ($FailOnError -and ($results | Where-Object { $_.Severity -eq 'Error' })) {
            throw "SAST scan found errors in $Path"
        }
    } else {
        Write-Host 'SAST scan: CLEAN' -ForegroundColor Green
    }
    
    return $results
}

# SBOM generation
function New-SBOM {
    param([string]$ProjectPath, [string]$OutputPath = './sbom.json')
    
    $modules  = Get-Module -ListAvailable | Where-Object { $_.ModuleBase -like "$ProjectPath*" }
    $packages = @()
    
    # PowerShell modules
    $packages += $modules | ForEach-Object {
        @{
            type    = 'powershell-module'
            name    = $_.Name
            version = $_.Version.ToString()
            path    = $_.ModuleBase
        }
    }
    
    # NuGet packages from .csproj
    Get-ChildItem $ProjectPath -Filter '*.csproj' -Recurse | ForEach-Object {
        [xml]$csproj = Get-Content $_
        $csproj.Project.ItemGroup.PackageReference | ForEach-Object {
            $packages += @{
                type    = 'nuget'
                name    = $_.Include
                version = $_.Version
            }
        }
    }
    
    $sbom = @{
        bomFormat     = 'CycloneDX'
        specVersion   = '1.4'
        version       = 1
        serialNumber  = [Guid]::NewGuid().ToString()
        metadata      = @{
            timestamp   = [datetime]::UtcNow.ToString('o')
            component   = @{ name=Split-Path $ProjectPath -Leaf; type='application' }
        }
        components    = $packages
    }
    
    $sbom | ConvertTo-Json -Depth 10 | Set-Content $OutputPath
    Write-Host "SBOM written: $OutputPath ($($packages.Count) components)" -ForegroundColor Green
    return $sbom
}

Invoke-SASTScan -Path './scripts' -Severity 'Warning' -FailOnError
New-SBOM -ProjectPath '.' -OutputPath './sbom.json'
```

---

## 3. Security Gate in CI/CD

```powershell
function Invoke-SecurityGate {
    param(
        [string]$ProjectPath,
        [hashtable]$Thresholds = @{
            MaxCriticalVulns  = 0
            MaxHighVulns      = 5
            MaxSASTErrors     = 0
            SecretsAllowed    = $false
        }
    )
    
    $violations = @()
    
    Write-Host '=== Security Gate ===' -ForegroundColor Cyan
    
    # 1. Secrets scan
    Write-Host '[1/3] Secrets Scan...' -ForegroundColor Yellow
    $secretsClean = Invoke-SecretsAudit -Path $ProjectPath -Recurse
    if (-not $secretsClean -and -not $Thresholds.SecretsAllowed) {
        $violations += 'Secrets detected in code'
    }
    
    # 2. SAST
    Write-Host '[2/3] SAST Scan...' -ForegroundColor Yellow
    $sast = Invoke-SASTScan -Path $ProjectPath -Severity 'Error'
    $errorCount = @($sast | Where-Object { $_.Severity -eq 'Error' }).Count
    if ($errorCount -gt $Thresholds.MaxSASTErrors) {
        $violations += "SAST errors: $errorCount (max $($Thresholds.MaxSASTErrors))"
    }
    
    # 3. Dependency audit
    Write-Host '[3/3] Dependency Audit...' -ForegroundColor Yellow
    try {
        $deps = Invoke-DependencyAudit -ProjectPath $ProjectPath
        $critical = @($deps | Where-Object { $_.Severity -in @('critical','Critical') }).Count
        $high     = @($deps | Where-Object { $_.Severity -in @('high','High') }).Count
        
        if ($critical -gt $Thresholds.MaxCriticalVulns) {
            $violations += "Critical vulns: $critical (max $($Thresholds.MaxCriticalVulns))"
        }
        if ($high -gt $Thresholds.MaxHighVulns) {
            $violations += "High vulns: $high (max $($Thresholds.MaxHighVulns))"
        }
    } catch { Write-Warning "Dep audit skipped: $_" }
    
    # Result
    if ($violations) {
        Write-Host "`nSecurity Gate: FAILED" -ForegroundColor Red
        $violations | ForEach-Object { Write-Host "  X $_" -ForegroundColor Red }
        return $false
    }
    
    Write-Host "`nSecurity Gate: PASSED" -ForegroundColor Green
    return $true
}

$passed = Invoke-SecurityGate -ProjectPath '.'
if (-not $passed) { exit 1 }  # Fail CI/CD pipeline
```

---

**ก่อนหน้า ← [Part 78](Part-78.md) | ต่อไป → [Part 80: Capstone Project 1](Part-80.md)**
