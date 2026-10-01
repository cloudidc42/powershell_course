# Part 22: Authentication และ Security ใน Web Apps

> **ระดับ**: 🟠 Advanced | **เวลา**: ~5 ชั่วโมง

---

## 1. Basic Authentication

```powershell
# Middleware: Basic Auth
function Get-BasicAuthMiddleware {
    param([hashtable]$Users)  # @{ username = hashedPassword }
    
    return {
        param($ctx)
        
        $authHeader = $ctx.Request.Headers['Authorization']
        if (!$authHeader -or !$authHeader.StartsWith('Basic ')) {
            $ctx.Response.StatusCode = 401
            $ctx.Response.AddHeader('WWW-Authenticate', 'Basic realm="API"')
            $ctx.Json(@{ error='Unauthorized' }, 401)
            return
        }
        
        $credentials = [System.Text.Encoding]::UTF8.GetString(
            [System.Convert]::FromBase64String($authHeader.Substring(6))
        )
        $username, $password = $credentials -split ':', 2
        
        $expectedHash = $using:Users[$username]
        if (!$expectedHash) {
            $ctx.Json(@{ error='Invalid credentials' }, 401)
            return
        }
        
        $inputHash = Get-FileHash -InputStream ([System.IO.MemoryStream]::new(
            [System.Text.Encoding]::UTF8.GetBytes($password)
        )) -Algorithm SHA256
        
        if ($inputHash.Hash -ne $expectedHash) {
            $ctx.Json(@{ error='Invalid credentials' }, 401)
            return
        }
        
        $ctx.State['User'] = $username
    }.GetNewClosure()
}

# SHA256 password hash
function Get-PasswordHash {
    param([string]$Password)
    $bytes = [System.Text.Encoding]::UTF8.GetBytes($Password)
    $hash  = [System.Security.Cryptography.SHA256]::Create().ComputeHash($bytes)
    return [System.Convert]::ToHexString($hash)
}

$userDb = @{
    'admin' = Get-PasswordHash 'admin123'
    'user1' = Get-PasswordHash 'pass456'
}

$auth = Get-BasicAuthMiddleware $userDb
```

---

## 2. JWT Authentication

```powershell
# สร้าง JWT เอง (HMAC SHA256)
function New-JwtToken {
    param(
        [hashtable]$Payload,
        [string]$Secret,
        [int]$ExpiryMinutes = 60
    )
    
    $header = @{ alg='HS256'; typ='JWT' }
    
    $now = [System.DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
    $Payload['iat'] = $now
    $Payload['exp'] = $now + ($ExpiryMinutes * 60)
    
    $encode = {
        param($obj)
        $json   = $obj | ConvertTo-Json -Compress
        $bytes  = [System.Text.Encoding]::UTF8.GetBytes($json)
        [System.Convert]::ToBase64String($bytes).TrimEnd('=').Replace('+','-').Replace('/','_')
    }
    
    $headerB64  = & $encode $header
    $payloadB64 = & $encode $Payload
    $sigInput   = "$headerB64.$payloadB64"
    
    $keyBytes = [System.Text.Encoding]::UTF8.GetBytes($Secret)
    $msgBytes = [System.Text.Encoding]::UTF8.GetBytes($sigInput)
    $hmac = [System.Security.Cryptography.HMACSHA256]::new($keyBytes)
    $sig  = $hmac.ComputeHash($msgBytes)
    $sigB64 = [System.Convert]::ToBase64String($sig).TrimEnd('=').Replace('+','-').Replace('/','_')
    
    return "$sigInput.$sigB64"
}

# Verify JWT
function Test-JwtToken {
    param([string]$Token, [string]$Secret)
    
    $parts = $Token -split '\.'
    if ($parts.Count -ne 3) { return $null }
    
    $headerB64, $payloadB64, $sigB64 = $parts
    $sigInput = "$headerB64.$payloadB64"
    
    $keyBytes = [System.Text.Encoding]::UTF8.GetBytes($Secret)
    $msgBytes = [System.Text.Encoding]::UTF8.GetBytes($sigInput)
    $hmac = [System.Security.Cryptography.HMACSHA256]::new($keyBytes)
    $sig  = $hmac.ComputeHash($msgBytes)
    $expectedSig = [System.Convert]::ToBase64String($sig).TrimEnd('=').Replace('+','-').Replace('/','_')
    
    if ($sigB64 -ne $expectedSig) { return $null }  # invalid signature
    
    $pad  = 4 - ($payloadB64.Length % 4)
    if ($pad -ne 4) { $payloadB64 += '=' * $pad }
    $payloadJson = [System.Text.Encoding]::UTF8.GetString(
        [System.Convert]::FromBase64String($payloadB64.Replace('-','+').Replace('_','/'))
    )
    $payload = $payloadJson | ConvertFrom-Json
    
    $now = [System.DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
    if ($payload.exp -lt $now) { return $null }  # expired
    
    return $payload
}

# Usage
$secret = 'my-super-secret-key-32chars-long!!'
$token = New-JwtToken -Payload @{ sub='user123'; name='Alice'; role='admin' } -Secret $secret
Write-Host "Token: $token"

$claims = Test-JwtToken -Token $token -Secret $secret
Write-Host "Subject: $($claims.sub), Role: $($claims.role)"
```

---

## 3. Rate Limiting Middleware

```powershell
function Get-RateLimitMiddleware {
    param(
        [int]$MaxRequests = 100,
        [int]$WindowSeconds = 60
    )
    
    $store = [System.Collections.Generic.Dictionary[string, object]]::new()
    
    return {
        param($ctx)
        
        $ip  = $ctx.Request.RemoteEndPoint.Address.ToString()
        $now = [System.DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
        $windowStart = $now - $using:WindowSeconds
        
        $lock = [System.Threading.Monitor]
        [System.Threading.Monitor]::Enter($using:store)
        
        try {
            if (!$using:store.ContainsKey($ip)) {
                $using:store[$ip] = [System.Collections.Generic.List[long]]::new()
            }
            
            $requests = $using:store[$ip]
            # Remove old requests outside window
            $toRemove = $requests | Where-Object { $_ -lt $windowStart }
            $toRemove | ForEach-Object { $requests.Remove($_) | Out-Null }
            
            if ($requests.Count -ge $using:MaxRequests) {
                $ctx.Response.AddHeader('Retry-After', $using:WindowSeconds.ToString())
                $ctx.Json(@{ error='Rate limit exceeded'; limit=$using:MaxRequests }, 429)
                return
            }
            
            $requests.Add($now)
            $ctx.Response.AddHeader('X-RateLimit-Limit',     $using:MaxRequests.ToString())
            $ctx.Response.AddHeader('X-RateLimit-Remaining', ($using:MaxRequests - $requests.Count).ToString())
        } finally {
            [System.Threading.Monitor]::Exit($using:store)
        }
    }.GetNewClosure()
}
```

---

## 4. Input Validation และ Sanitization

```powershell
function Test-SqlInjection {
    param([string]$Input)
    $patterns = @(
        "['";]--",
        "\bOR\b.*=",
        "\bDROP\b",
        "\bUNION\b",
        "xp_",
        "EXEC\s*\("
    )
    foreach ($p in $patterns) {
        if ($Input -imatch $p) { return $true }
    }
    return $false
}

function ConvertTo-SafeHtml {
    param([string]$Input)
    $Input `
        -replace '&', '&amp;' `
        -replace '<', '&lt;'  `
        -replace '>', '&gt;'  `
        -replace '"', '&quot;' `
        -replace "'", '&#x27;'
}

# Validate request body
function Assert-ValidUser {
    param([PSObject]$Data)
    
    $errors = [System.Collections.Generic.List[string]]::new()
    
    if ([string]::IsNullOrWhiteSpace($Data.name)) {
        $errors.Add('name is required')
    } elseif ($Data.name.Length -gt 100) {
        $errors.Add('name must be 100 chars or less')
    } elseif ($Data.name -match '[<>&"'']') {
        $errors.Add('name contains invalid characters')
    }
    
    if ([string]::IsNullOrWhiteSpace($Data.email)) {
        $errors.Add('email is required')
    } elseif ($Data.email -notmatch '^[^@]+@[^@]+\.[^@]+$') {
        $errors.Add('email is invalid')
    }
    
    if ($Data.age -and ($Data.age -lt 0 -or $Data.age -gt 150)) {
        $errors.Add('age must be 0-150')
    }
    
    if ($errors.Count -gt 0) {
        throw [PSCustomObject]@{ Errors=$errors; StatusCode=400 }
    }
}
```

---

## 5. HTTPS Setup

```powershell
# HTTPS listener ต้องมี certificate
$listener = [System.Net.HttpListener]::new()
$listener.Prefixes.Add("https://+:443/")

# สร้าง self-signed cert (dev only)
# New-SelfSignedCertificate -DnsName 'localhost' -CertStoreLocation 'cert:\LocalMachine\My'

# ผูก cert กับ port (Windows, run as admin)
# netsh http add sslcert ipport=0.0.0.0:443 certhash=<THUMBPRINT> appid='{<GUID>}'

# Development: ใช้ Kestrel ดีกว่า
# Install-Package -Name 'Microsoft.AspNetCore.App' (เป็นวิธีที่แนะนำ)

# Redirect HTTP->HTTPS
$httpListener = [System.Net.HttpListener]::new()
$httpListener.Prefixes.Add('http://+:80/')
$httpListener.Start()

while ($httpListener.IsListening) {
    $ctx = $httpListener.GetContext()
    $httpsUrl = $ctx.Request.Url.ToString() -replace '^http://', 'https://'
    $ctx.Response.Redirect($httpsUrl)
    $ctx.Response.OutputStream.Close()
}
```

---

**ก่อนหน้า ← [Part 21](Part-21.md) | ต่อไป → [Part 23: Database Integration](Part-23.md)**
